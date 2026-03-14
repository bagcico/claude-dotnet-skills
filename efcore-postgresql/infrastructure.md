# Infrastructure & Performance

## Concurrency

### Optimistic (xmin)
```csharp
// Entity
public uint RowVersion { get; set; }

// Configuration
builder.Property(a => a.RowVersion).HasColumnName("xmin").HasColumnType("xid").IsRowVersion();
```

### Pessimistic (SELECT FOR UPDATE)
```csharp
var entity = await dbContext.Appointments
    .FromSqlInterpolated($"SELECT * FROM appointments WHERE id = {id} FOR UPDATE")
    .FirstOrDefaultAsync(cancellationToken);
```
Use sparingly: payments, seat reservation, counter increments.

### Advisory Locks
```csharp
var lockKey = HashCode.Combine("renewal", subscriptionId);
var acquired = await dbContext.Database
    .SqlQueryRaw<bool>($"SELECT pg_try_advisory_lock({lockKey})")
    .FirstAsync(cancellationToken);
// ... critical section ...
await dbContext.Database.ExecuteSqlRawAsync($"SELECT pg_advisory_unlock({lockKey})", cancellationToken);
```

## Connection Pool

```json
"Host=localhost;Database=appdb;Maximum Pool Size=20;Minimum Pool Size=5;Connection Idle Lifetime=300;"
```

- `Maximum Pool Size=20` — not 100+, each connection ~10MB RAM in PG
- `Minimum Pool Size=5` — pre-warm
- Pool math: `20 pool × 5 ECS tasks = 100 connections` → RDS `max_connections=200`

## RDS Configuration

### GP3 over GP2 (always)
GP3: baseline 3,000 IOPS free. GP2: IOPS tied to storage size.

### Key Parameters
```
max_connections = 200        shared_buffers = 25% RAM
effective_cache_size = 75% RAM   work_mem = 256MB
random_page_cost = 1.1       effective_io_concurrency = 200
```

## Performance Diagnostics

### Enable Query Logging (dev only)
```csharp
optionsBuilder.UseNpgsql(conn).LogTo(Console.WriteLine, LogLevel.Information)
    .EnableSensitiveDataLogging().EnableDetailedErrors();
```

### Slow Query Interceptor (production)
```csharp
public class SlowQueryInterceptor : DbCommandInterceptor
{
    public override async ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command, CommandExecutedEventData eventData,
        DbDataReader result, CancellationToken cancellationToken = default)
    {
        if (eventData.Duration.TotalMilliseconds > 200)
            _logger.LogWarning("Slow query ({DurationMs}ms): {CommandText}",
                eventData.Duration.TotalMilliseconds, command.CommandText);
        return result;
    }
}
```

### EXPLAIN ANALYZE
```csharp
var plan = await dbContext.Database
    .SqlQueryRaw<string>("EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) " + sql)
    .ToListAsync(cancellationToken);
```
Look for: `Seq Scan` on large tables, `Nested Loop` with high rows, `Sort` with `Disk`, `Hash Join` on large sets.
