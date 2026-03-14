# Redis Setup

## Service Registration

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
    options.InstanceName = "app:";
});

builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration["Redis:ConnectionString"]!));

builder.Services.AddScoped<ICacheService, RedisCacheService>();
```

## ICacheService Interface

```csharp
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken cancellationToken = default);
    Task SetAsync<T>(string key, T value, TimeSpan? expiration = null, CancellationToken cancellationToken = default);
    Task RemoveAsync(string key, CancellationToken cancellationToken = default);
    Task RemoveByPrefixAsync(string prefix, CancellationToken cancellationToken = default);
    Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiration = null, CancellationToken cancellationToken = default);
}
```

## RedisCacheService

```csharp
public sealed class RedisCacheService(IDistributedCache cache, IConnectionMultiplexer redis) : ICacheService
{
    private static readonly JsonSerializerOptions JsonOptions = new()
    { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };

    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        var data = await cache.GetStringAsync(key, ct);
        return data is null ? default : JsonSerializer.Deserialize<T>(data, JsonOptions);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiration = null, CancellationToken ct = default)
    {
        var data = JsonSerializer.Serialize(value, JsonOptions);
        await cache.SetStringAsync(key, data, new DistributedCacheEntryOptions
        { AbsoluteExpirationRelativeToNow = expiration ?? TimeSpan.FromMinutes(5) }, ct);
    }

    public async Task RemoveAsync(string key, CancellationToken ct = default)
        => await cache.RemoveAsync(key, ct);

    public async Task RemoveByPrefixAsync(string prefix, CancellationToken ct = default)
    {
        var server = redis.GetServer(redis.GetEndPoints().First());
        var keys = server.Keys(pattern: $"app:{prefix}*").ToArray();
        if (keys.Length > 0) await redis.GetDatabase().KeyDeleteAsync(keys);
    }

    public async Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiration = null, CancellationToken ct = default)
    {
        var cached = await GetAsync<T>(key, ct);
        if (cached is not null) return cached;
        var value = await factory();
        await SetAsync(key, value, expiration, ct);
        return value;
    }
}
```

## CacheKey Builder

```csharp
public static class CacheKeys
{
    public static string Entity<T>(Guid id) => $"{typeof(T).Name.ToLower()}:detail:{id}";
    public static string List<T>(string filterHash) => $"{typeof(T).Name.ToLower()}:list:{filterHash}";
    public static string User(Guid userId, string scope) => $"user:{userId}:{scope}";
    public static string ComputeFilterHash(object filters) =>
        Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(JsonSerializer.Serialize(filters))))[..8].ToLower();
}
```

## CachingBehavior Integration

Queries implement `ICacheable` to opt-in to caching:
```csharp
public sealed record GetMentorListQuery(...) : IRequest<Result<PaginatedList<MentorDto>>>, ICacheable
{
    public string CacheKey => $"mentor:list:{CacheKeys.ComputeFilterHash(this)}";
    public TimeSpan? Expiration => TimeSpan.FromMinutes(10);
}
```
