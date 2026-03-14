# Pipeline Behaviors

Four MediatR pipeline behaviors registered in order: Validation → Logging → Transaction → Caching.

## ValidationBehavior (FluentValidation → Result\<T\> hybrid)

Catches validation failures and converts to `Result<T>.ValidationFailure()` — no exceptions.

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators
) : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!validators.Any())
            return await next(cancellationToken);

        var context = new ValidationContext<TRequest>(request);
        var validationResults = await Task.WhenAll(
            validators.Select(v => v.ValidateAsync(context, cancellationToken)));

        var failures = validationResults.SelectMany(r => r.Errors)
            .Where(f => f is not null).ToList();

        if (failures.Count == 0)
            return await next(cancellationToken);

        var responseType = typeof(TResponse);

        if (responseType.IsGenericType &&
            responseType.GetGenericTypeDefinition() == typeof(Result<>))
        {
            var errors = failures.Select(f => f.ErrorMessage).ToArray();
            var failureMethod = responseType.GetMethod(nameof(Result<object>.ValidationFailure))!;
            return (TResponse)failureMethod.Invoke(null, [errors])!;
        }

        if (responseType == typeof(Result))
        {
            var errors = failures.Select(f => f.ErrorMessage).ToArray();
            return (TResponse)(object)Result.ValidationFailure(errors);
        }

        throw new ValidationException(failures);
    }
}
```

## LoggingBehavior

Logs request name, module, duration. Warns on slow requests (>500ms).

```csharp
public sealed class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger
) : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {RequestName} {@Request}", requestName, request);

        var stopwatch = Stopwatch.StartNew();
        var response = await next(cancellationToken);
        stopwatch.Stop();

        if (stopwatch.ElapsedMilliseconds > 500)
            logger.LogWarning("Long running: {RequestName} ({ElapsedMs}ms)",
                requestName, stopwatch.ElapsedMilliseconds);

        logger.LogInformation("Handled {RequestName} in {ElapsedMs}ms",
            requestName, stopwatch.ElapsedMilliseconds);
        return response;
    }
}
```

## TransactionBehavior

Wraps command handlers in a DB transaction. Skips queries (by naming convention).

```csharp
public sealed class TransactionBehavior<TRequest, TResponse>(
    IApplicationDbContext dbContext
) : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (typeof(TRequest).Name.EndsWith("Query"))
            return await next(cancellationToken);

        await using var transaction = await dbContext.Database
            .BeginTransactionAsync(cancellationToken);
        try
        {
            var response = await next(cancellationToken);
            await transaction.CommitAsync(cancellationToken);
            return response;
        }
        catch
        {
            await transaction.RollbackAsync(cancellationToken);
            throw;
        }
    }
}
```

## CachingBehavior

Caches query results via `ICacheable` interface. Commands bypass automatically.

```csharp
public interface ICacheable
{
    string CacheKey { get; }
    TimeSpan? Expiration { get; }
}

public sealed class CachingBehavior<TRequest, TResponse>(
    ICacheService cacheService
) : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (request is not ICacheable cacheable)
            return await next(cancellationToken);

        var cached = await cacheService.GetAsync<TResponse>(
            cacheable.CacheKey, cancellationToken);
        if (cached is not null)
            return cached;

        var response = await next(cancellationToken);
        await cacheService.SetAsync(cacheable.CacheKey, response,
            cacheable.Expiration ?? TimeSpan.FromMinutes(5), cancellationToken);
        return response;
    }
}
```
