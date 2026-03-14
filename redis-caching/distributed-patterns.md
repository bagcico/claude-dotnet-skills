# Distributed Patterns

## Table of Contents
1. [Distributed Locking](#distributed-locking)
2. [Rate Limiting](#rate-limiting)
3. [Pub/Sub for Cache Invalidation](#pubsub-for-cache-invalidation)
4. [Distributed Counter](#distributed-counter)
5. [Leaderboard / Sorted Sets](#leaderboard--sorted-sets)
6. [Health Checks](#health-checks)

---

## Distributed Locking

For coordinating operations across multiple service instances (ECS tasks). Prevents 
duplicate processing of payments, renewals, scheduled jobs.

### RedLock-Style Distributed Lock

```csharp
using StackExchange.Redis;

namespace ProjectName.Infrastructure.Caching;

public sealed class RedisDistributedLock(IConnectionMultiplexer redis)
    : IDistributedLock
{
    public async Task<IAsyncDisposable?> TryAcquireAsync(
        string resource,
        TimeSpan expiry,
        CancellationToken cancellationToken = default)
    {
        var db = redis.GetDatabase();
        var lockId = Guid.NewGuid().ToString();
        var lockKey = $"lock:{resource}";

        var acquired = await db.StringSetAsync(
            lockKey, lockId, expiry, When.NotExists);

        if (!acquired)
            return null;

        return new LockHandle(db, lockKey, lockId);
    }

    public async Task<IAsyncDisposable> AcquireAsync(
        string resource,
        TimeSpan expiry,
        TimeSpan? timeout = null,
        CancellationToken cancellationToken = default)
    {
        var deadline = DateTime.UtcNow.Add(timeout ?? TimeSpan.FromSeconds(30));

        while (DateTime.UtcNow < deadline)
        {
            var handle = await TryAcquireAsync(resource, expiry, cancellationToken);
            if (handle is not null)
                return handle;

            await Task.Delay(TimeSpan.FromMilliseconds(100), cancellationToken);
        }

        throw new TimeoutException(
            $"Could not acquire lock on '{resource}' within timeout.");
    }

    private sealed class LockHandle(IDatabase db, string key, string value)
        : IAsyncDisposable
    {
        private const string ReleaseScript = """
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
            """;

        public async ValueTask DisposeAsync()
        {
            await db.ScriptEvaluateAsync(ReleaseScript,
                new RedisKey[] { key },
                new RedisValue[] { value });
        }
    }
}

// Interface in Application layer
public interface IDistributedLock
{
    Task<IAsyncDisposable?> TryAcquireAsync(string resource, TimeSpan expiry,
        CancellationToken cancellationToken = default);
    Task<IAsyncDisposable> AcquireAsync(string resource, TimeSpan expiry,
        TimeSpan? timeout = null, CancellationToken cancellationToken = default);
}
```

### Usage in Command Handlers

```csharp
public sealed class ProcessRenewalCommandHandler(
    IApplicationDbContext dbContext,
    IDistributedLock distributedLock,
    ILogger<ProcessRenewalCommandHandler> logger
) : IRequestHandler<ProcessRenewalCommand, Result>
{
    public async Task<Result> Handle(
        ProcessRenewalCommand request, CancellationToken cancellationToken)
    {
        // Acquire lock to prevent duplicate processing
        var lockHandle = await distributedLock.TryAcquireAsync(
            $"renewal:{request.SubscriptionId}",
            TimeSpan.FromMinutes(2),
            cancellationToken);

        if (lockHandle is null)
            return Result.Failure("Renewal already being processed.");

        await using (lockHandle)
        {
            var subscription = await dbContext.Subscriptions
                .FirstOrDefaultAsync(s => s.Id == request.SubscriptionId,
                    cancellationToken);

            if (subscription is null)
                return Result.Failure("Subscription not found.");

            // ... renewal logic ...

            await dbContext.SaveChangesAsync(cancellationToken);
            return Result.Success();
        }
    }
}
```

### Idempotency Key Pattern

For operations that must execute exactly once, combine distributed lock with 
idempotency tracking:

```csharp
public async Task<Result<Guid>> Handle(
    ProcessPaymentCommand request, CancellationToken cancellationToken)
{
    var idempotencyKey = $"payment:idempotency:{request.IdempotencyKey}";
    var db = redis.GetDatabase();

    // Check if already processed
    var existingResult = await db.StringGetAsync(idempotencyKey);
    if (existingResult.HasValue)
        return Result<Guid>.Success(Guid.Parse(existingResult!));

    // Acquire lock for this specific payment
    await using var lockHandle = await distributedLock.AcquireAsync(
        $"payment:{request.IdempotencyKey}", TimeSpan.FromMinutes(5),
        cancellationToken: cancellationToken);

    // Double-check after lock acquisition
    existingResult = await db.StringGetAsync(idempotencyKey);
    if (existingResult.HasValue)
        return Result<Guid>.Success(Guid.Parse(existingResult!));

    // Process payment...
    var transactionId = Guid.NewGuid();

    // Store result for idempotency (24h TTL)
    await db.StringSetAsync(idempotencyKey, transactionId.ToString(),
        TimeSpan.FromHours(24));

    return Result<Guid>.Success(transactionId);
}
```

---

## Rate Limiting

### Sliding Window Rate Limiter (Redis-backed)

For API rate limiting across multiple instances. Uses Redis sorted sets:

```csharp
public sealed class RedisSlidingWindowRateLimiter(IConnectionMultiplexer redis)
    : IRateLimiter
{
    public async Task<RateLimitResult> CheckAsync(
        string clientKey,
        int maxRequests,
        TimeSpan window,
        CancellationToken cancellationToken = default)
    {
        var db = redis.GetDatabase();
        var key = $"ratelimit:{clientKey}";
        var now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        var windowStart = now - (long)window.TotalMilliseconds;

        // Lua script for atomic sliding window check
        var script = """
            -- Remove expired entries
            redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', ARGV[1])
            -- Count current entries
            local count = redis.call('ZCARD', KEYS[1])
            if count < tonumber(ARGV[3]) then
                -- Under limit — add this request
                redis.call('ZADD', KEYS[1], ARGV[2], ARGV[2] .. ':' .. math.random())
                redis.call('PEXPIRE', KEYS[1], ARGV[4])
                return {1, tonumber(ARGV[3]) - count - 1}
            else
                -- Over limit
                return {0, 0}
            end
            """;

        var result = (RedisResult[]?)await db.ScriptEvaluateAsync(
            script,
            new RedisKey[] { key },
            new RedisValue[]
            {
                windowStart,
                now,
                maxRequests,
                (long)window.TotalMilliseconds
            });

        var allowed = (int)result![0] == 1;
        var remaining = (int)result[1];

        return new RateLimitResult(allowed, remaining, maxRequests);
    }
}

public record RateLimitResult(bool IsAllowed, int Remaining, int Limit);
```

### Rate Limiting Middleware

```csharp
public sealed class RateLimitingMiddleware(
    RequestDelegate next,
    IRateLimiter rateLimiter
)
{
    public async Task InvokeAsync(HttpContext context)
    {
        // Use CF-Connecting-IP for real client IP behind Cloudflare
        var clientIp = context.Request.Headers["CF-Connecting-IP"].FirstOrDefault()
            ?? context.Connection.RemoteIpAddress?.ToString()
            ?? "unknown";

        var result = await rateLimiter.CheckAsync(
            clientIp,
            maxRequests: 100,
            window: TimeSpan.FromMinutes(1));

        context.Response.Headers["X-RateLimit-Limit"] = result.Limit.ToString();
        context.Response.Headers["X-RateLimit-Remaining"] = result.Remaining.ToString();

        if (!result.IsAllowed)
        {
            context.Response.StatusCode = StatusCodes.Status429TooManyRequests;
            context.Response.Headers["Retry-After"] = "60";
            await context.Response.WriteAsJsonAsync(new { error = "Rate limit exceeded." });
            return;
        }

        await next(context);
    }
}
```

### Per-User Rate Limiting

```csharp
// In endpoint definition
group.MapPost("/", CreateAppointment)
    .RequireRateLimiting("authenticated-user");

// Rate limiting policy based on user identity
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("authenticated-user", context =>
    {
        var userId = context.User.FindFirst("sub")?.Value ?? "anonymous";
        return RateLimitPartition.GetSlidingWindowLimiter(userId, _ =>
            new SlidingWindowRateLimiterOptions
            {
                PermitLimit = 30,
                Window = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 6
            });
    });
});
```

---

## Pub/Sub for Cache Invalidation

When running multiple service instances (ECS tasks), use Redis pub/sub to invalidate 
L1 (in-memory) caches across all instances:

```csharp
public sealed class RedisCacheInvalidationService(
    IConnectionMultiplexer redis,
    IMemoryCache memoryCache,
    ILogger<RedisCacheInvalidationService> logger
) : BackgroundService
{
    private const string Channel = "cache:invalidation";

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var subscriber = redis.GetSubscriber();

        await subscriber.SubscribeAsync(
            RedisChannel.Pattern(Channel),
            (_, message) =>
            {
                var key = message.ToString();
                logger.LogDebug("Cache invalidation received for key: {Key}", key);

                if (key.EndsWith("*"))
                {
                    // Prefix invalidation — can't selectively clear IMemoryCache
                    // In production, use a tagged memory cache or just clear all
                    if (memoryCache is MemoryCache mc)
                        mc.Compact(1.0); // Clear all
                }
                else
                {
                    memoryCache.Remove(key);
                }
            });

        // Keep running until stopped
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    public static async Task PublishInvalidation(
        IConnectionMultiplexer redis, string key)
    {
        var subscriber = redis.GetSubscriber();
        await subscriber.PublishAsync(
            RedisChannel.Literal(Channel), key);
    }
}
```

---

## Distributed Counter

Atomic counter operations — useful for view counts, rate tracking, inventory:

```csharp
public sealed class RedisCounter(IConnectionMultiplexer redis)
{
    public async Task<long> IncrementAsync(string key,
        TimeSpan? expiry = null)
    {
        var db = redis.GetDatabase();
        var value = await db.StringIncrementAsync($"counter:{key}");

        if (expiry.HasValue && value == 1)
            await db.KeyExpireAsync($"counter:{key}", expiry);

        return value;
    }

    public async Task<long> GetAsync(string key)
    {
        var db = redis.GetDatabase();
        var value = await db.StringGetAsync($"counter:{key}");
        return value.HasValue ? (long)value : 0;
    }

    public async Task<long> DecrementAsync(string key)
    {
        var db = redis.GetDatabase();
        return await db.StringDecrementAsync($"counter:{key}");
    }
}

// Usage: daily active user counter
await counter.IncrementAsync($"dau:{DateTime.UtcNow:yyyy-MM-dd}", TimeSpan.FromDays(7));
```

---

## Leaderboard / Sorted Sets

For ranking systems (mentor ratings, student scores):

```csharp
public sealed class RedisLeaderboard(IConnectionMultiplexer redis)
{
    public async Task UpdateScoreAsync(string leaderboard, string memberId, double score)
    {
        var db = redis.GetDatabase();
        await db.SortedSetAddAsync($"leaderboard:{leaderboard}", memberId, score);
    }

    public async Task<List<LeaderboardEntry>> GetTopAsync(
        string leaderboard, int count = 10)
    {
        var db = redis.GetDatabase();
        var entries = await db.SortedSetRangeByRankWithScoresAsync(
            $"leaderboard:{leaderboard}",
            start: 0,
            stop: count - 1,
            order: Order.Descending);

        return entries.Select((e, i) => new LeaderboardEntry(
            Rank: i + 1,
            MemberId: e.Element.ToString(),
            Score: e.Score
        )).ToList();
    }

    public async Task<long?> GetRankAsync(string leaderboard, string memberId)
    {
        var db = redis.GetDatabase();
        var rank = await db.SortedSetRankAsync(
            $"leaderboard:{leaderboard}", memberId, Order.Descending);
        return rank.HasValue ? rank.Value + 1 : null;
    }
}

public record LeaderboardEntry(int Rank, string MemberId, double Score);
```

---

## Health Checks

```csharp
// In Program.cs
builder.Services.AddHealthChecks()
    .AddRedis(
        builder.Configuration["Redis:ConnectionString"]!,
        name: "redis",
        tags: ["cache", "redis"]);
```

### Custom Health Check with Latency

```csharp
public sealed class RedisLatencyHealthCheck(IConnectionMultiplexer redis)
    : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var db = redis.GetDatabase();
        var stopwatch = Stopwatch.StartNew();

        try
        {
            await db.PingAsync();
            stopwatch.Stop();

            var latency = stopwatch.ElapsedMilliseconds;

            if (latency > 100)
                return HealthCheckResult.Degraded(
                    $"Redis latency is {latency}ms (threshold: 100ms)");

            return HealthCheckResult.Healthy(
                $"Redis is healthy (latency: {latency}ms)");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Redis is unreachable", ex);
        }
    }
}
```
