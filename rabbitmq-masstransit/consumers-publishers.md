# Messaging Patterns

## Table of Contents
1. [Consumer Patterns](#consumer-patterns)
2. [Publisher Patterns](#publisher-patterns)
3. [Request/Response](#requestresponse)
4. [Retry & Error Handling](#retry--error-handling)
5. [Consumer Configuration](#consumer-configuration)
6. [Testing Consumers](#testing-consumers)
7. [Monitoring & Observability](#monitoring--observability)

---

## Consumer Patterns

### Basic Event Consumer

```csharp
using MassTransit;
using ProjectName.Contracts.Events;

namespace ProjectName.Infrastructure.Consumers;

public sealed class AppointmentCreatedEventConsumer(
    IApplicationDbContext dbContext,
    ILogger<AppointmentCreatedEventConsumer> logger
) : IConsumer<AppointmentCreatedEvent>
{
    public async Task Consume(ConsumeContext<AppointmentCreatedEvent> context)
    {
        var message = context.Message;

        logger.LogInformation(
            "Processing AppointmentCreatedEvent for {AppointmentId}",
            message.AppointmentId);

        // Create notification for mentor
        var notification = new Notification
        {
            Id = Guid.NewGuid(),
            UserId = message.MentorId,
            Title = "New Appointment",
            Body = $"You have a new appointment at {message.ScheduledAt:g}",
            Type = NotificationType.AppointmentReminder,
            ReferenceId = message.AppointmentId,
            CreatedAt = DateTime.UtcNow
        };

        dbContext.Notifications.Add(notification);
        await dbContext.SaveChangesAsync(context.CancellationToken);
    }
}
```

### Command Consumer

```csharp
public sealed class SendNotificationCommandConsumer(
    INotificationService notificationService,
    ILogger<SendNotificationCommandConsumer> logger
) : IConsumer<SendNotificationCommand>
{
    public async Task Consume(ConsumeContext<SendNotificationCommand> context)
    {
        var command = context.Message;

        logger.LogInformation(
            "Sending notification to {UserId}: {Title}",
            command.UserId, command.Title);

        await notificationService.SendAsync(
            command.UserId,
            command.Title,
            command.Body,
            command.Type,
            context.CancellationToken);
    }
}
```

### Consumer with Database + Publishing (Chain Pattern)

A consumer that processes a message, updates the database, and publishes a follow-up event:

```csharp
public sealed class ProcessPaymentCommandConsumer(
    IApplicationDbContext dbContext,
    IPaymentGateway paymentGateway,
    ILogger<ProcessPaymentCommandConsumer> logger
) : IConsumer<ProcessPaymentCommand>
{
    public async Task Consume(ConsumeContext<ProcessPaymentCommand> context)
    {
        var command = context.Message;

        var subscription = await dbContext.Subscriptions
            .FirstOrDefaultAsync(s => s.Id == command.SubscriptionId,
                context.CancellationToken);

        if (subscription is null)
        {
            logger.LogWarning("Subscription {Id} not found, skipping", command.SubscriptionId);
            return; // Don't retry — permanent failure
        }

        // Call external payment gateway
        var paymentResult = await paymentGateway.ChargeAsync(
            subscription.PaymentMethodId,
            command.Amount,
            context.CancellationToken);

        if (!paymentResult.IsSuccess)
        {
            // Publish failure event and return (don't throw — this is a business outcome)
            await context.Publish(new PaymentFailedEvent(
                SubscriptionId: command.SubscriptionId,
                Reason: paymentResult.Error!,
                OccurredAt: DateTime.UtcNow
            ));
            return;
        }

        // Update subscription
        subscription.LastPaymentAt = DateTime.UtcNow;
        subscription.Status = SubscriptionStatus.Active;
        await dbContext.SaveChangesAsync(context.CancellationToken);

        // Publish success event
        await context.Publish(new PaymentCompletedEvent(
            SubscriptionId: command.SubscriptionId,
            Amount: command.Amount,
            TransactionId: paymentResult.TransactionId,
            OccurredAt: DateTime.UtcNow
        ));
    }
}
```

### Idempotent Consumer

For operations that must not be duplicated (payments, notifications):

```csharp
public sealed class ProcessPaymentCommandConsumer(
    IApplicationDbContext dbContext,
    IPaymentGateway paymentGateway
) : IConsumer<ProcessPaymentCommand>
{
    public async Task Consume(ConsumeContext<ProcessPaymentCommand> context)
    {
        var command = context.Message;

        // Idempotency check: has this message already been processed?
        var alreadyProcessed = await dbContext.ProcessedMessages
            .AnyAsync(m => m.MessageId == context.MessageId,
                context.CancellationToken);

        if (alreadyProcessed)
            return;

        // ... process payment ...

        // Record message as processed
        dbContext.ProcessedMessages.Add(new ProcessedMessage
        {
            MessageId = context.MessageId ?? Guid.NewGuid(),
            ConsumerType = nameof(ProcessPaymentCommandConsumer),
            ProcessedAt = DateTime.UtcNow
        });

        await dbContext.SaveChangesAsync(context.CancellationToken);
    }
}
```

### Batch Consumer

For high-throughput scenarios where processing messages one-by-one is inefficient:

```csharp
public sealed class AuditLogEventBatchConsumer(
    IApplicationDbContext dbContext
) : IConsumer<Batch<AuditLogEvent>>
{
    public async Task Consume(ConsumeContext<Batch<AuditLogEvent>> context)
    {
        var logs = context.Message
            .Select(msg => new AuditLog
            {
                Id = Guid.NewGuid(),
                Action = msg.Message.Action,
                UserId = msg.Message.UserId,
                Details = msg.Message.Details,
                CreatedAt = msg.Message.OccurredAt
            })
            .ToList();

        dbContext.AuditLogs.AddRange(logs);
        await dbContext.SaveChangesAsync(context.CancellationToken);
    }
}

// Configuration
cfg.AddConsumer<AuditLogEventBatchConsumer>(c =>
    c.Options<BatchOptions>(o => o
        .SetMessageLimit(100)
        .SetTimeLimit(TimeSpan.FromSeconds(5))));
```

---

## Publisher Patterns

### Publish Event (Broadcast)

```csharp
// From MediatR handler — inject IPublishEndpoint
await publishEndpoint.Publish(new AppointmentCreatedEvent(...), cancellationToken);

// From consumer — use ConsumeContext
await context.Publish(new PaymentCompletedEvent(...));
```

### Send Command (Direct)

```csharp
// From handler — inject ISendEndpointProvider
var endpoint = await sendEndpointProvider
    .GetSendEndpoint(new Uri("queue:notification-send"));

await endpoint.Send(new SendNotificationCommand(
    UserId: userId,
    Title: "Your appointment is confirmed",
    Body: message,
    Type: NotificationType.Push,
    ReferenceId: appointmentId
), cancellationToken);
```

### Scheduled Messages

```csharp
// Send a reminder 1 hour before appointment
var scheduledTime = appointment.ScheduledAt.AddHours(-1);

await context.SchedulePublish(
    scheduledTime,
    new AppointmentReminderEvent(
        AppointmentId: appointment.Id,
        MentorId: appointment.MentorId,
        StudentId: appointment.StudentId,
        ScheduledAt: appointment.ScheduledAt
    ));
```

Requires a scheduler (Hangfire, Quartz, or RabbitMQ delayed message plugin).

---

## Request/Response

For synchronous-style queries over messaging (use sparingly — prefer direct API calls):

### Requester

```csharp
public sealed class CheckMentorAvailabilityHandler(
    IRequestClient<CheckMentorAvailabilityRequest> client
) : IRequestHandler<CheckMentorAvailabilityQuery, Result<bool>>
{
    public async Task<Result<bool>> Handle(
        CheckMentorAvailabilityQuery request,
        CancellationToken cancellationToken)
    {
        var response = await client.GetResponse<MentorAvailabilityResponse>(
            new CheckMentorAvailabilityRequest(
                request.MentorId,
                request.ScheduledAt,
                request.DurationMinutes),
            cancellationToken);

        return Result<bool>.Success(response.Message.IsAvailable);
    }
}
```

### Responder

```csharp
public sealed class CheckMentorAvailabilityConsumer(
    IApplicationDbContext dbContext
) : IConsumer<CheckMentorAvailabilityRequest>
{
    public async Task Consume(ConsumeContext<CheckMentorAvailabilityRequest> context)
    {
        var req = context.Message;

        var hasConflict = await dbContext.Appointments
            .AnyAsync(a =>
                a.MentorId == req.MentorId &&
                a.Status != AppointmentStatus.Cancelled &&
                a.ScheduledAt < req.ScheduledAt.AddMinutes(req.DurationMinutes) &&
                a.ScheduledAt.AddMinutes(a.DurationMinutes) > req.ScheduledAt,
                context.CancellationToken);

        await context.RespondAsync(new MentorAvailabilityResponse(!hasConflict));
    }
}
```

---

## Retry & Error Handling

### Differentiating Transient vs Permanent Failures

```csharp
e.UseMessageRetry(r => r
    .Exponential(5, TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(1), TimeSpan.FromSeconds(5))
    // Only retry transient failures
    .Handle<DbUpdateException>()
    .Handle<TimeoutException>()
    .Handle<HttpRequestException>()
    // Don't retry these — they won't fix themselves
    .Ignore<ValidationException>()
    .Ignore<ArgumentException>()
    .Ignore<InvalidOperationException>());
```

### Circuit Breaker

```csharp
e.UseCircuitBreaker(cb =>
{
    cb.TrackingPeriod = TimeSpan.FromMinutes(1);
    cb.TripThreshold = 15;    // Trip after 15 failures
    cb.ActiveThreshold = 10;  // In the tracking period
    cb.ResetInterval = TimeSpan.FromMinutes(5);
});
```

### Rate Limiting

```csharp
e.UseRateLimit(100, TimeSpan.FromSeconds(1)); // 100 messages per second
```

### Dead Letter Queue Reprocessing

When messages end up in `{queue}_error`, investigate and reprocess:

```csharp
// In a separate admin endpoint or job
public sealed class ReprocessDeadLettersJob(IBusControl bus)
{
    public async Task ExecuteAsync(string sourceQueue, CancellationToken cancellationToken)
    {
        var errorQueueUri = new Uri($"queue:{sourceQueue}_error");
        var receiveEndpoint = bus.ConnectReceiveEndpoint(errorQueueUri, cfg =>
        {
            cfg.Consumer<DeadLetterReprocessConsumer>();
        });

        await receiveEndpoint.Ready;
        // Process messages from the error queue...
    }
}
```

---

## Consumer Configuration

### Per-Consumer Concurrency

```csharp
cfg.AddConsumer<SendNotificationCommandConsumer>(c =>
{
    c.ConcurrentMessageLimit = 10; // Max 10 parallel message handlers
});
```

### Endpoint-Specific Configuration

```csharp
rabbit.ReceiveEndpoint("payment-process", e =>
{
    e.PrefetchCount = 16;        // Messages pre-fetched from RabbitMQ
    e.ConcurrentMessageLimit = 8; // Concurrent handlers

    e.UseMessageRetry(r => r.Intervals(
        TimeSpan.FromSeconds(5),
        TimeSpan.FromSeconds(15),
        TimeSpan.FromSeconds(30)));

    e.ConfigureConsumer<ProcessPaymentCommandConsumer>(context);
});
```

### Separate Queues for Priority

```csharp
// High-priority: payment processing
rabbit.ReceiveEndpoint("payment-process-high", e =>
{
    e.PrefetchCount = 32;
    e.ConcurrentMessageLimit = 16;
    e.ConfigureConsumer<ProcessPaymentCommandConsumer>(context);
});

// Low-priority: analytics, logging
rabbit.ReceiveEndpoint("analytics-process-low", e =>
{
    e.PrefetchCount = 8;
    e.ConcurrentMessageLimit = 4;
    e.ConfigureConsumer<AnalyticsEventConsumer>(context);
});
```

---

## Testing Consumers

### Unit Testing with InMemoryTestHarness

```csharp
[Fact]
public async Task AppointmentCreatedEvent_CreatesNotification()
{
    await using var provider = new ServiceCollection()
        .AddMassTransitTestHarness(cfg =>
        {
            cfg.AddConsumer<AppointmentCreatedEventConsumer>();
        })
        .AddScoped<IApplicationDbContext>(_ => CreateTestDbContext())
        .BuildServiceProvider(true);

    var harness = provider.GetRequiredService<ITestHarness>();
    await harness.Start();

    // Publish event
    await harness.Bus.Publish(new AppointmentCreatedEvent(
        AppointmentId: Guid.NewGuid(),
        MentorId: Guid.NewGuid(),
        StudentId: Guid.NewGuid(),
        ScheduledAt: DateTime.UtcNow.AddDays(1),
        OccurredAt: DateTime.UtcNow));

    // Assert consumer processed the message
    var consumerHarness = harness.GetConsumerHarness<AppointmentCreatedEventConsumer>();
    Assert.True(await consumerHarness.Consumed.Any<AppointmentCreatedEvent>());

    // Assert notification was created in DB
    var dbContext = provider.GetRequiredService<IApplicationDbContext>();
    Assert.True(await dbContext.Notifications.AnyAsync());
}
```

---

## Monitoring & Observability

### Structured Logging in Consumers

```csharp
public async Task Consume(ConsumeContext<AppointmentCreatedEvent> context)
{
    using var _ = LogContext.Push(
        new PropertyEnricher("MessageId", context.MessageId),
        new PropertyEnricher("ConversationId", context.ConversationId),
        new PropertyEnricher("CorrelationId", context.CorrelationId));

    logger.LogInformation(
        "Consuming {MessageType} for AppointmentId={AppointmentId}",
        nameof(AppointmentCreatedEvent),
        context.Message.AppointmentId);

    // ... consumer logic ...
}
```

### Health Checks

```csharp
// In Program.cs
builder.Services.AddHealthChecks()
    .AddRabbitMQ(
        rabbitConnectionString: builder.Configuration["RabbitMQ:ConnectionString"]!,
        name: "rabbitmq",
        tags: ["messaging", "rabbitmq"]);
```
