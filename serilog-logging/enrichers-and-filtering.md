# Logging Patterns

## Table of Contents
1. [Custom Enrichers](#custom-enrichers)
2. [Sensitive Data Filtering](#sensitive-data-filtering)
3. [MediatR Logging Integration](#mediatr-logging-integration)
4. [MassTransit Logging Integration](#masstransit-logging-integration)
5. [EF Core Query Logging](#ef-core-query-logging)
6. [ELK Stack Configuration](#elk-stack-configuration)
7. [Performance Logging](#performance-logging)
8. [Health Check Logging](#health-check-logging)

---

## Custom Enrichers

### User Context Enricher

Automatically adds the current user's ID and role to every log entry:

```csharp
using Serilog.Core;
using Serilog.Events;

public sealed class UserContextEnricher(IHttpContextAccessor httpContextAccessor)
    : ILogEventEnricher
{
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        var httpContext = httpContextAccessor.HttpContext;
        if (httpContext?.User.Identity?.IsAuthenticated != true)
            return;

        var userId = httpContext.User.FindFirst("sub")?.Value;
        var userRole = httpContext.User.FindFirst("role")?.Value;

        if (userId is not null)
            logEvent.AddPropertyIfAbsent(
                propertyFactory.CreateProperty("UserId", userId));

        if (userRole is not null)
            logEvent.AddPropertyIfAbsent(
                propertyFactory.CreateProperty("UserRole", userRole));
    }
}

// Registration
builder.Host.UseSerilog((context, services, configuration) =>
{
    configuration
        .Enrich.With(new UserContextEnricher(
            services.GetRequiredService<IHttpContextAccessor>()))
        // ... rest of config
});
```

### Service Version Enricher

For tracking which deployment version generated each log:

```csharp
public sealed class ServiceVersionEnricher : ILogEventEnricher
{
    private readonly string _version;

    public ServiceVersionEnricher()
    {
        _version = typeof(ServiceVersionEnricher).Assembly
            .GetCustomAttribute<AssemblyInformationalVersionAttribute>()?
            .InformationalVersion ?? "unknown";
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        logEvent.AddPropertyIfAbsent(
            propertyFactory.CreateProperty("ServiceVersion", _version));
    }
}
```

### ECS Task Enricher (AWS)

```csharp
public sealed class EcsTaskEnricher : ILogEventEnricher
{
    private readonly string? _taskId;
    private readonly string? _taskArn;

    public EcsTaskEnricher()
    {
        _taskId = Environment.GetEnvironmentVariable("ECS_TASK_ID");
        _taskArn = Environment.GetEnvironmentVariable("ECS_TASK_ARN");
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        if (_taskId is not null)
            logEvent.AddPropertyIfAbsent(
                propertyFactory.CreateProperty("EcsTaskId", _taskId));
        if (_taskArn is not null)
            logEvent.AddPropertyIfAbsent(
                propertyFactory.CreateProperty("EcsTaskArn", _taskArn));
    }
}
```

---

## Sensitive Data Filtering

### Destructuring Policy for PII

Prevents accidental logging of sensitive data by intercepting object destructuring:

```csharp
public sealed class SensitiveDataDestructuringPolicy : IDestructuringPolicy
{
    private static readonly HashSet<string> SensitiveProperties = new(StringComparer.OrdinalIgnoreCase)
    {
        "Password", "PasswordHash", "Secret", "Token", "AccessToken", "RefreshToken",
        "ApiKey", "CreditCard", "CardNumber", "Cvv", "Ssn", "TcKimlik",
        "Email", "PhoneNumber", "Phone"
    };

    public bool TryDestructure(object value, ILogEventPropertyValueFactory propertyValueFactory,
        out LogEventPropertyValue? result)
    {
        result = null;
        var type = value.GetType();

        if (type.IsPrimitive || type == typeof(string) || type == typeof(decimal))
            return false;

        var properties = type.GetProperties()
            .Select(p =>
            {
                var propValue = p.GetValue(value);
                var logValue = SensitiveProperties.Contains(p.Name)
                    ? "[REDACTED]"
                    : propValue;

                return new LogEventProperty(
                    p.Name,
                    propertyValueFactory.CreatePropertyValue(logValue, destructureObjects: true));
            })
            .ToList();

        result = new StructureValue(properties);
        return true;
    }
}

// Registration
configuration.Destructure.With<SensitiveDataDestructuringPolicy>();
```

### Log Filtering by Expression

Filter out noisy or sensitive logs using Serilog.Expressions:

```csharp
configuration.Filter.ByExcluding(
    "RequestPath like '/health%' or RequestPath like '/metrics%'");
```

In appsettings.json:
```json
{
  "Serilog": {
    "Filter": [
      {
        "Name": "ByExcluding",
        "Args": {
          "expression": "RequestPath like '/health%' or RequestPath like '/favicon%'"
        }
      }
    ]
  }
}
```

---

## MediatR Logging Integration

### Enhanced LoggingBehavior

The `LoggingBehavior` from the Clean Architecture skill already logs request/response timing.
Enhance it with structured properties:

```csharp
public sealed class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger
) : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        var moduleName = GetModuleName(typeof(TRequest));

        using (LogContext.PushProperty("MediatRRequest", requestName))
        using (LogContext.PushProperty("Module", moduleName))
        {
            logger.LogInformation(
                "Handling {RequestName} in module {Module}",
                requestName, moduleName);

            var stopwatch = Stopwatch.StartNew();

            try
            {
                var response = await next(cancellationToken);
                stopwatch.Stop();

                // Log result status if it's a Result<T>
                if (response is Result result)
                {
                    if (result.IsFailure)
                        logger.LogWarning(
                            "Request {RequestName} failed: {Error} ({ElapsedMs}ms)",
                            requestName, result.Error, stopwatch.ElapsedMilliseconds);
                    else
                        logger.LogInformation(
                            "Handled {RequestName} successfully ({ElapsedMs}ms)",
                            requestName, stopwatch.ElapsedMilliseconds);
                }

                if (stopwatch.ElapsedMilliseconds > 500)
                    logger.LogWarning(
                        "Long running request: {RequestName} ({ElapsedMs}ms)",
                        requestName, stopwatch.ElapsedMilliseconds);

                return response;
            }
            catch (Exception ex)
            {
                stopwatch.Stop();
                logger.LogError(ex,
                    "Request {RequestName} threw exception after {ElapsedMs}ms",
                    requestName, stopwatch.ElapsedMilliseconds);
                throw;
            }
        }
    }

    private static string GetModuleName(Type type)
    {
        var ns = type.Namespace ?? string.Empty;
        var parts = ns.Split('.');
        var moduleIndex = Array.IndexOf(parts, "Modules");
        return moduleIndex >= 0 && moduleIndex + 1 < parts.Length
            ? parts[moduleIndex + 1]
            : "Unknown";
    }
}
```

---

## MassTransit Logging Integration

### Consumer Logging with Correlation

```csharp
public async Task Consume(ConsumeContext<AppointmentCreatedEvent> context)
{
    using (LogContext.PushProperty("MessageId", context.MessageId))
    using (LogContext.PushProperty("ConversationId", context.ConversationId))
    using (LogContext.PushProperty("CorrelationId", context.CorrelationId))
    using (LogContext.PushProperty("MessageType", nameof(AppointmentCreatedEvent)))
    using (LogContext.PushProperty("ConsumerType", nameof(AppointmentCreatedEventConsumer)))
    {
        logger.LogInformation(
            "Consuming {MessageType} — AppointmentId={AppointmentId}",
            nameof(AppointmentCreatedEvent),
            context.Message.AppointmentId);

        try
        {
            // ... consumer logic ...

            logger.LogInformation(
                "Successfully consumed {MessageType} — AppointmentId={AppointmentId}",
                nameof(AppointmentCreatedEvent),
                context.Message.AppointmentId);
        }
        catch (Exception ex)
        {
            logger.LogError(ex,
                "Failed to consume {MessageType} — AppointmentId={AppointmentId}",
                nameof(AppointmentCreatedEvent),
                context.Message.AppointmentId);
            throw; // Let MassTransit retry
        }
    }
}
```

---

## EF Core Query Logging

### Selective Query Logging

In development, log all SQL. In production, only log slow queries:

```csharp
// Development
optionsBuilder
    .UseNpgsql(connectionString)
    .LogTo(
        message => Log.Logger.Debug(message),
        new[] { DbLoggerCategory.Database.Command.Name },
        LogLevel.Information)
    .EnableSensitiveDataLogging();

// Production — use SlowQueryInterceptor (from EF Core skill)
optionsBuilder
    .UseNpgsql(connectionString)
    .AddInterceptors(new SlowQueryInterceptor(
        serviceProvider.GetRequiredService<ILogger<SlowQueryInterceptor>>()));
```

---

