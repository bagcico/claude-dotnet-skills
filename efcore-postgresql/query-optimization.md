# Query Optimization

## Read Patterns

### Single Entity
```csharp
// By PK only (checks change tracker first)
var entity = await dbContext.Appointments.FindAsync([id], cancellationToken);

// With includes (queries only)
var entity = await dbContext.Appointments
    .AsNoTracking()
    .Include(a => a.Mentor).Include(a => a.Student)
    .FirstOrDefaultAsync(a => a.Id == id, cancellationToken);
```

### Existence / Count
```csharp
var exists = await dbContext.Appointments.AnyAsync(a => a.Id == id, cancellationToken);
var count = await dbContext.Appointments
    .CountAsync(a => a.Status == AppointmentStatus.Confirmed, cancellationToken);
```

### Split Queries
Use `AsSplitQuery()` with 2+ collection navigation includes to avoid cartesian explosion:
```csharp
var mentor = await dbContext.Mentors.AsNoTracking()
    .Include(m => m.Appointments).Include(m => m.Reviews)
    .AsSplitQuery()
    .FirstOrDefaultAsync(m => m.Id == id, cancellationToken);
```

## N+1 Prevention

**Problem**: loop with queries inside.

**Solution 1 — Eager loading**: `Include()` with filtered collections.
**Solution 2 — Projection** (preferred for DTOs):
```csharp
var mentors = await dbContext.Mentors.AsNoTracking()
    .Select(m => new MentorWithStatsDto(
        m.Id, m.FullName,
        m.Appointments.Count(a => a.Status == AppointmentStatus.Confirmed),
        m.Appointments.Where(a => a.Status == AppointmentStatus.Confirmed)
            .OrderByDescending(a => a.ScheduledAt)
            .Select(a => new AppointmentSummaryDto(a.Id, a.ScheduledAt, a.DurationMinutes))
            .Take(5).ToList()
    )).ToListAsync(cancellationToken);
```

**Solution 3 — GroupBy** for aggregations in a single query.

## Pagination

### Offset-Based (standard)
```csharp
var totalCount = await query.CountAsync(cancellationToken);
var items = await query.Skip((pageNumber - 1) * pageSize).Take(pageSize)
    .Select(a => a.ToDto()).ToListAsync(cancellationToken);
```

### Keyset/Cursor (for 100K+ rows)
```csharp
// First page
var items = await query.OrderByDescending(a => a.ScheduledAt)
    .ThenByDescending(a => a.Id).Take(pageSize).ToListAsync(cancellationToken);

// Next pages — use last item as cursor
var lastItem = items.Last();
var nextPage = await query
    .Where(a => a.ScheduledAt < lastItem.ScheduledAt ||
               (a.ScheduledAt == lastItem.ScheduledAt && a.Id < lastItem.Id))
    .OrderByDescending(a => a.ScheduledAt).ThenByDescending(a => a.Id)
    .Take(pageSize).ToListAsync(cancellationToken);
```

## Conditional Query Building

```csharp
var query = dbContext.Appointments.AsNoTracking().AsQueryable();

if (request.MentorId.HasValue)
    query = query.Where(a => a.MentorId == request.MentorId.Value);
if (request.Status.HasValue)
    query = query.Where(a => a.Status == request.Status.Value);
if (!string.IsNullOrWhiteSpace(request.SearchTerm))
    query = query.Where(a => a.Mentor.FullName.Contains(request.SearchTerm)
        || a.Student.FullName.Contains(request.SearchTerm));
if (request.FromDate.HasValue)
    query = query.Where(a => a.ScheduledAt >= request.FromDate.Value);

query = request.SortBy switch
{
    "date" => request.SortDesc ? query.OrderByDescending(a => a.ScheduledAt) : query.OrderBy(a => a.ScheduledAt),
    _ => query.OrderByDescending(a => a.CreatedAt)
};
```

## Projection Optimization

```csharp
// BAD — loads all columns
var list = await dbContext.Appointments.AsNoTracking().ToListAsync(ct);
var dtos = list.Select(a => a.ToDto()).ToList();

// GOOD — SQL only fetches needed columns
var dtos = await dbContext.Appointments.AsNoTracking()
    .Select(a => new AppointmentDto(a.Id, a.MentorId, a.Mentor.FullName, ...))
    .ToListAsync(ct);
```

Use projection for list queries. Use `Include + ToDto()` when you need the entity for writes.
