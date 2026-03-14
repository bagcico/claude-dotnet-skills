# Write Operations & Raw SQL

## Write Patterns

### Create
```csharp
var entity = new Appointment { Id = Guid.NewGuid(), ..., CreatedAt = DateTime.UtcNow };
dbContext.Appointments.Add(entity);
await dbContext.SaveChangesAsync(cancellationToken);
```

### Update (fetch-modify-save)
```csharp
var entity = await dbContext.Appointments
    .FirstOrDefaultAsync(a => a.Id == request.Id, cancellationToken);
if (entity is null) return Result<AppointmentDto>.Failure("Not found.");
entity.ScheduledAt = request.ScheduledAt;
entity.UpdatedAt = DateTime.UtcNow;
await dbContext.SaveChangesAsync(cancellationToken);
```

### Batch Update (no entity loading)
```csharp
var affected = await dbContext.Appointments
    .Where(a => a.Status == AppointmentStatus.Pending && a.ScheduledAt < DateTime.UtcNow.AddDays(-7))
    .ExecuteUpdateAsync(s => s
        .SetProperty(a => a.Status, AppointmentStatus.Cancelled)
        .SetProperty(a => a.UpdatedAt, DateTime.UtcNow), cancellationToken);
```

### Batch Delete
```csharp
var deleted = await dbContext.AuditLogs
    .Where(l => l.CreatedAt < DateTime.UtcNow.AddYears(-1))
    .ExecuteDeleteAsync(cancellationToken);
```

Note: `ExecuteUpdateAsync`/`ExecuteDeleteAsync` bypass change tracker and EF events.

## Bulk Insert

### AddRange (up to ~1K rows)
```csharp
dbContext.Notifications.AddRange(entities);
await dbContext.SaveChangesAsync(cancellationToken);
```

### PostgreSQL COPY (10K+ rows)
```csharp
await using var conn = new NpgsqlConnection(connectionString);
await conn.OpenAsync(cancellationToken);
await using var writer = await conn.BeginBinaryImportAsync(
    "COPY notifications (id, user_id, message, created_at) FROM STDIN (FORMAT BINARY)", cancellationToken);
foreach (var item in items)
{
    await writer.StartRowAsync(cancellationToken);
    await writer.WriteAsync(item.Id, cancellationToken);
    await writer.WriteAsync(item.UserId, cancellationToken);
    await writer.WriteAsync(item.Message, cancellationToken);
    await writer.WriteAsync(item.CreatedAt, NpgsqlDbType.TimestampTz, cancellationToken);
}
await writer.CompleteAsync(cancellationToken);
```
10-50x faster than EF AddRange for large datasets.

## Raw SQL

### Safe Parameterized
```csharp
var appointments = await dbContext.Appointments
    .FromSqlInterpolated($"SELECT * FROM appointments WHERE mentor_id = {mentorId}")
    .AsNoTracking().ToListAsync(cancellationToken);
```

### LATERAL JOIN (PostgreSQL's OUTER APPLY)
"Top N per group" — e.g., latest appointment per mentor:
```csharp
var results = await dbContext.Mentors.AsNoTracking()
    .Where(m => m.IsActive)
    .SelectMany(m => dbContext.Appointments
        .Where(a => a.MentorId == m.Id && a.Status == AppointmentStatus.Confirmed)
        .OrderByDescending(a => a.ScheduledAt).Take(1).DefaultIfEmpty(),
        (m, a) => new MentorLatestAppointmentDto(
            m.Id, m.FullName, a != null ? a.Id : (Guid?)null, a != null ? a.ScheduledAt : null))
    .ToListAsync(cancellationToken);
```

### Complex Aggregation
```csharp
var stats = await dbContext.Database.SqlQueryRaw<MentorStatsDto>(@"
    SELECT m.id AS ""MentorId"", m.full_name AS ""MentorName"",
           COUNT(a.id) AS ""TotalAppointments"",
           COUNT(a.id) FILTER (WHERE a.status = 'Completed') AS ""CompletedAppointments"",
           AVG(a.duration_minutes) FILTER (WHERE a.status = 'Completed') AS ""AvgDurationMinutes""
    FROM mentors m LEFT JOIN appointments a ON a.mentor_id = m.id
    WHERE m.is_active = true
    GROUP BY m.id, m.full_name
    HAVING COUNT(a.id) > 0
    ORDER BY COUNT(a.id) DESC").ToListAsync(cancellationToken);
```
Column aliases must match C# property names (case-sensitive, use double quotes for PascalCase).
