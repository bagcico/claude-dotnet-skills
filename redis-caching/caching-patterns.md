# Caching Patterns

## Table of Contents
1. [Cache-Aside Pattern](#cache-aside-pattern)
2. [Write-Through Pattern](#write-through-pattern)
3. [Cache Stampede Prevention](#cache-stampede-prevention)
4. [Multi-Level Caching](#multi-level-caching)
5. [Sliding vs Absolute Expiration](#sliding-vs-absolute-expiration)
6. [Cache Warming](#cache-warming)
7. [Serialization](#serialization)

---

## Cache-Aside Pattern

The default and most common pattern. Application checks cache first, falls back to database:

```csharp
// In a query handler
public async Task<Result<MentorDto>> Handle(
    GetMentorQuery request, CancellationToken cancellationToken)
{
    var cacheKey = CacheKeys.Entity<Mentor>(request.Id);

    // 1. Check cache
    var cached = await cacheService.GetAsync<MentorDto>(cacheKey, cancellationToken);
    if (cached is not null)
        return Result<MentorDto>.Success(cached);

    // 2. Cache miss — query database
    var mentor = await dbContext.Mentors
        .AsNoTracking()
        .Include(m => m.Specializations)
        .FirstOrDefaultAsync(m => m.Id == request.Id, cancellationToken);

    if (mentor is null)
        return Result<MentorDto>.Failure("Mentor not found.");

    var dto = mentor.ToDto();

    // 3. Populate cache
    await cacheService.SetAsync(cacheKey, dto, TimeSpan.FromMinutes(15), cancellationToken);

    return Result<MentorDto>.Success(dto);
}
```

Or use the `GetOrSetAsync` shorthand:

```csharp
var dto = await cacheService.GetOrSetAsync(
    CacheKeys.Entity<Mentor>(request.Id),
    async () =>
    {
        var mentor = await dbContext.Mentors
            .AsNoTracking()
            .FirstOrDefaultAsync(m => m.Id == request.Id, cancellationToken);
        return mentor?.ToDto();
    },
    TimeSpan.FromMinutes(15),
    cancellationToken);
```

### When to Use Cache-Aside
- Read-heavy data with infrequent writes
- Data that can tolerate short staleness (seconds to minutes)
- Entity lookups, reference data, configuration

### When NOT to Use Cache-Aside
- Frequently updated data (cache is always stale)
- Data that must be real-time consistent (account balances, inventory counts)
- Write-heavy workloads (cache invalidation overhead exceeds benefit)

---

## Write-Through Pattern

Update cache simultaneously with the database. Useful when stale data is unacceptable 
for frequently-read entities:

```csharp
public sealed class UpdateMentorProfileCommandHandler(
    IApplicationDbContext dbContext,
    ICacheService cacheService
) : IRequestHandler<UpdateMentorProfileCommand, Result<MentorProfileDto>>
{
    public async Task<Result<MentorProfileDto>> Handle(
        UpdateMentorProfileCommand request, CancellationToken cancellationToken)
    {
        var profile = await dbContext.MentorProfiles
            .FirstOrDefaultAsync(p => p.MentorId == request.MentorId, cancellationToken);

        if (profile is null)
            return Result<MentorProfileDto>.Failure("Profile not found.");

        // Update entity
        profile.Bio = request.Bio;
        profile.UpdatedAt = DateTime.UtcNow;

        await dbContext.SaveChangesAsync(cancellationToken);

        var dto = profile.ToDto();

        // Write-through: update cache with fresh data immediately
        await cacheService.SetAsync(
            CacheKeys.Entity<MentorProfile>(profile.MentorId),
            dto,
            TimeSpan.FromHours(1),
            cancellationToken);

        return Result<MentorProfileDto>.Success(dto);
    }
}
```

---

## Cache Stampede Prevention

When a popular cache key expires, hundreds of concurrent requests may all hit the database 
simultaneously. This is a "cache stampede" or "thundering herd".

### Solution: Distributed Lock on Cache Miss

```csharp
public async Task<T> GetOrSetWithLockAsync<T>(
    string key,
    Func<Task<T>> factory,
    TimeSpan? expiration = null,
    CancellationToken cancellationToken = default)
{
    // Check cache
    var cached = await GetAsync<T>(key, cancellationToken);
    if (cached is not null)
        return cached;

    // Try to acquire lock — only one request populates cache
    var lockKey = $"lock:{key}";
    var db = redis.GetDatabase();
    var lockId = Guid.NewGuid().ToString();
    var lockAcquired = await db.StringSetAsync(lockKey, lockId, TimeSpan.FromSeconds(10),
        When.NotExists);

    if (lockAcquired)
    {
        try
        {
            // Double-check cache (another thread may have populated while waiting)
            cached = await GetAsync<T>(key, cancellationToken);
            if (cached is not null)
                return cached;

            var value = await factory();
            await SetAsync(key, value, expiration, cancellationToken);
            return value;
        }
        finally
        {
            // Release lock only if we still own it
            var script = """
                if redis.call("get", KEYS[1]) == ARGV[1] then
                    return redis.call("del", KEYS[1])
                else
                    return 0
                end
                """;
            await db.ScriptEvaluateAsync(script,
                new RedisKey[] { lockKey },
                new RedisValue[] { lockId });
        }
    }
    else
    {
        // Another request is populating — wait briefly and retry from cache
        await Task.Delay(100, cancellationToken);
        return await GetAsync<T>(key, cancellationToken) ?? await factory();
    }
}
```

### Solution: Early Expiration (Soft TTL)

Store data with a soft TTL that's shorter than the actual Redis TTL. When a request 
sees data past its soft TTL, it refreshes in background:

```csharp
public class CacheEntry<T>
{
    public T Value { get; set; } = default!;
    public DateTime SoftExpiry { get; set; }
}

public async Task<T> GetWithSoftExpiryAsync<T>(
    string key, Func<Task<T>> factory,
    TimeSpan softTtl, TimeSpan hardTtl,
    CancellationToken cancellationToken = default)
{
    var entry = await GetAsync<CacheEntry<T>>(key, cancellationToken);

    if (entry is not null)
    {
        if (DateTime.UtcNow < entry.SoftExpiry)
            return entry.Value; // Still fresh

        // Stale but usable — refresh in background
        _ = Task.Run(async () =>
        {
            var fresh = await factory();
            await SetAsync(key, new CacheEntry<T>
            {
                Value = fresh,
                SoftExpiry = DateTime.UtcNow.Add(softTtl)
            }, hardTtl, CancellationToken.None);
        }, cancellationToken);

        return entry.Value; // Return stale data while refreshing
    }

    // Full cache miss — synchronous fetch
    var value = await factory();
    await SetAsync(key, new CacheEntry<T>
    {
        Value = value,
        SoftExpiry = DateTime.UtcNow.Add(softTtl)
    }, hardTtl, cancellationToken);

    return value;
}
```

---

## Multi-Level Caching

L1 (in-process memory) + L2 (Redis) for ultra-hot data:

```csharp
using Microsoft.Extensions.Caching.Memory;

public sealed class MultiLevelCacheService(
    IMemoryCache memoryCache,
    ICacheService redisCache
) : IMultiLevelCacheService
{
    public async Task<T?> GetAsync<T>(string key,
        CancellationToken cancellationToken = default)
    {
        // L1: in-process memory (microseconds)
        if (memoryCache.TryGetValue<T>(key, out var memoryValue))
            return memoryValue;

        // L2: Redis (milliseconds)
        var redisValue = await redisCache.GetAsync<T>(key, cancellationToken);
        if (redisValue is not null)
        {
            // Backfill L1 with shorter TTL
            memoryCache.Set(key, redisValue, TimeSpan.FromSeconds(30));
            return redisValue;
        }

        return default;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiration = null,
        CancellationToken cancellationToken = default)
    {
        // L1: short TTL
        memoryCache.Set(key, value, TimeSpan.FromSeconds(30));

        // L2: full TTL
        await redisCache.SetAsync(key, value, expiration, cancellationToken);
    }

    public async Task RemoveAsync(string key,
        CancellationToken cancellationToken = default)
    {
        memoryCache.Remove(key);
        await redisCache.RemoveAsync(key, cancellationToken);
    }
}
```

Use multi-level caching for:
- Configuration/settings data loaded on every request
- User profile/session data
- Feature flags
- Extremely hot reference data (categories, countries, etc.)

---

## Sliding vs Absolute Expiration

### Absolute Expiration
Cache entry expires at a fixed time, regardless of access:
```csharp
var options = new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15)
};
```
Use for: data that changes on a schedule, stale data that must be refreshed periodically.

### Sliding Expiration
Cache entry expires after a period of inactivity — each access resets the timer:
```csharp
var options = new DistributedCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(5),
    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1) // safety cap
};
```
Use for: session data, user-specific data, anything where active users should keep the cache warm.

Always combine sliding with an absolute cap to prevent indefinite caching.

---

## Cache Warming

Pre-populate caches on application startup for critical hot data:

```csharp
public sealed class CacheWarmupService(
    IServiceProvider serviceProvider,
    ILogger<CacheWarmupService> logger
) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Wait for app to fully start
        await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);

        using var scope = serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<IApplicationDbContext>();
        var cacheService = scope.ServiceProvider.GetRequiredService<ICacheService>();

        logger.LogInformation("Starting cache warmup...");

        // Warm up categories (reference data)
        var categories = await dbContext.Categories
            .AsNoTracking()
            .Where(c => c.IsActive)
            .Select(c => c.ToDto())
            .ToListAsync(stoppingToken);

        await cacheService.SetAsync("categories:active:all", categories,
            TimeSpan.FromHours(6), stoppingToken);

        // Warm up top mentors
        var topMentors = await dbContext.Mentors
            .AsNoTracking()
            .Where(m => m.IsActive)
            .OrderByDescending(m => m.Rating)
            .Take(50)
            .Select(m => m.ToDto())
            .ToListAsync(stoppingToken);

        await cacheService.SetAsync("mentor:list:top50", topMentors,
            TimeSpan.FromMinutes(30), stoppingToken);

        logger.LogInformation("Cache warmup completed: {CategoryCount} categories, {MentorCount} mentors",
            categories.Count, topMentors.Count);
    }
}

// Register in Program.cs
builder.Services.AddHostedService<CacheWarmupService>();
```

---

## Serialization

### System.Text.Json Configuration for Redis

```csharp
private static readonly JsonSerializerOptions JsonOptions = new()
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    Converters =
    {
        new JsonStringEnumConverter(JsonNamingPolicy.CamelCase)
    }
};
```

### Handling Polymorphic Types

If you cache DTOs that contain derived types or interfaces:

```csharp
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    TypeInfoResolver = new DefaultJsonTypeInfoResolver()
};
```

### Cache Size Awareness

Redis stores everything in memory. Be mindful of value sizes:
- Small values (< 1KB): entity DTOs, counters — great for Redis
- Medium values (1-100KB): lists, paginated results — acceptable
- Large values (> 100KB): full reports, large datasets — consider compression or avoid caching
- Very large values (> 1MB): don't cache in Redis, use file storage or CDN

For medium values, consider compression:

```csharp
public async Task SetCompressedAsync<T>(string key, T value, TimeSpan? expiration = null,
    CancellationToken cancellationToken = default)
{
    var json = JsonSerializer.SerializeToUtf8Bytes(value, JsonOptions);

    using var compressedStream = new MemoryStream();
    await using (var gzipStream = new GZipStream(compressedStream, CompressionLevel.Fastest))
    {
        await gzipStream.WriteAsync(json, cancellationToken);
    }

    var db = redis.GetDatabase();
    await db.StringSetAsync(key, compressedStream.ToArray(), expiration);
}
```
