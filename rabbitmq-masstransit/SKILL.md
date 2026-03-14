---
name: rabbitmq-masstransit
description: >
  RabbitMQ messaging patterns with MassTransit in .NET 10. Use this skill whenever the user 
  asks about message queues, event-driven architecture, pub/sub, consumers, producers, sagas, 
  state machines, retry policies, dead letter queues, or async communication between services. 
  Also trigger for "mesaj kuyruğu", "event yayınla", "consumer yaz", "saga oluştur", "retry 
  policy", "dead letter", "RabbitMQ", "MassTransit", "message broker", "publish", "send", 
  "consume", "event bus", "integration event", "outbox pattern", or any asynchronous messaging 
  concern in .NET.
---

# RabbitMQ + MassTransit Patterns

Production patterns for MassTransit with RabbitMQ in .NET 10.

## Reference Files — Read on Demand

| When the user asks about... | Read this file |
|----------------------------|---------------|
| Consumers, publishers, configuration, retry | `./consumers-publishers.md` |
| Sagas, outbox, multi-service architecture | `./sagas-and-advanced.md` |

Read only the relevant file.

## Message Types

| Type | Purpose | Method | Exchange |
|------|---------|--------|----------|
| Event | Broadcast (past tense) | `Publish<T>()` | Fanout |
| Command | Direct instruction | `Send<T>(endpoint)` | Direct |

**Naming**: Events = past tense (`AppointmentCreatedEvent`), Commands = imperative (`SendNotificationCommand`).

## Message Contracts

Records in a shared contracts project. No behavior, no dependencies:

```csharp
public record AppointmentCreatedEvent(
    Guid AppointmentId, Guid MentorId, Guid StudentId,
    DateTime ScheduledAt, DateTime OccurredAt);

public record SendNotificationCommand(
    Guid UserId, string Title, string Body,
    NotificationType Type, Guid? ReferenceId);
```

## Basic Setup

```csharp
builder.Services.AddMassTransit(cfg =>
{
    cfg.AddConsumers(typeof(Program).Assembly);
    cfg.UsingRabbitMq((context, rabbit) =>
    {
        rabbit.Host(builder.Configuration["RabbitMQ:Host"], h =>
        {
            h.Username(builder.Configuration["RabbitMQ:Username"]!);
            h.Password(builder.Configuration["RabbitMQ:Password"]!);
        });
        rabbit.UseMessageRetry(r => r.Exponential(5,
            TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(1), TimeSpan.FromSeconds(5)));
        rabbit.ConfigureEndpoints(context);
    });
});
```

## Publishing from MediatR Handlers

Publish events AFTER `SaveChangesAsync`, not before:
```csharp
await dbContext.SaveChangesAsync(cancellationToken);
await publishEndpoint.Publish(new AppointmentCreatedEvent(...), cancellationToken);
```
For guaranteed delivery, use Outbox pattern → see `./sagas-and-advanced.md`.

## Error Handling: 3 Levels
1. **Retry** — transient failures, exponential backoff
2. **Redelivery** — delayed retry after minutes/hours
3. **Error Queue** — `{queue}_error` for investigation
