---
name: serilog-logging
description: >
  Structured logging with Serilog in .NET 10, including ELK stack integration, enrichers, 
  correlation IDs, and observability patterns. Use this skill whenever the user asks about 
  logging, structured logging, Serilog configuration, log enrichment, correlation tracking, 
  ELK integration, Elasticsearch sink, Kibana dashboards, log levels, sensitive data filtering, 
  request/response logging, or any observability concern. Also trigger for "log ekle", "loglama", 
  "Serilog", "structured logging", "correlation ID", "request logging", "ELK", "Elasticsearch", 
  "Kibana", "Grafana", "log level", "enricher", "sensitive data", "PII filtering", or any .NET 
  logging and monitoring question.
---

# Serilog Structured Logging

Production patterns for Serilog with ELK in .NET 10.

## Reference Files — Read on Demand

| When the user asks about... | Read this file |
|----------------------------|---------------|
| Enrichers, PII filtering, correlation ID | `./enrichers-and-filtering.md` |
| ELK setup, Elasticsearch sink, ILM, Docker Compose | `./elk-setup.md` |

Read only the relevant file.

## Core Setup (Program.cs)

```csharp
builder.Host.UseSerilog((context, services, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithEnvironmentName()
        .Enrich.WithMachineName()
        .Enrich.WithProperty("Application", "ProjectName.API")
        .WriteTo.Console(outputTemplate:
            "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext}\n  {Message:lj}\n{Exception}")
        .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(
            new Uri(context.Configuration["Elasticsearch:Url"]!))
        {
            IndexFormat = $"app-{context.HostingEnvironment.EnvironmentName.ToLower()}-{{0:yyyy.MM.dd}}",
            AutoRegisterTemplate = true,
            AutoRegisterTemplateVersion = AutoRegisterTemplateVersion.ESv7,
        });
});

app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";
    options.GetLevel = (ctx, elapsed, ex) =>
        ex is not null || ctx.Response.StatusCode >= 500 ? LogEventLevel.Error :
        elapsed > 1000 || ctx.Response.StatusCode >= 400 ? LogEventLevel.Warning :
        LogEventLevel.Information;
});
```

## Structured Logging Rules

```csharp
// GOOD — structured, searchable
logger.LogInformation("Appointment {AppointmentId} created for {MentorId}", id, mentorId);

// BAD — string interpolation destroys structure
logger.LogInformation($"Appointment {id} created for {mentorId}");
```

- Use meaningful property names (`PaymentId`, not `Id`)
- Never log passwords, tokens, emails, phone numbers
- Use `{@Object}` for destructuring, `{Property}` for scalar values

## Log Levels

| Level | Use For |
|-------|---------|
| Debug | Dev diagnostics, cache hit/miss — off in production |
| Information | Business events: "Appointment created", "Payment processed" |
| Warning | Slow query, retry attempt, rate limit hit |
| Error | Unhandled exception, external service failure |
| Fatal | App crash, startup failure |

Production default: `Information`. EF Core / ASP.NET internals: `Warning`.

## Correlation ID

```csharp
// Middleware: extract or generate, push to LogContext
var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault()
    ?? Guid.NewGuid().ToString();
using (LogContext.PushProperty("CorrelationId", correlationId))
    await next(context);
```
