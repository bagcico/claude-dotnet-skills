# Advanced Patterns

## Table of Contents
1. [Saga State Machines](#saga-state-machines)
2. [Outbox Pattern](#outbox-pattern)
3. [Message Topology](#message-topology)
4. [Multi-Service Architecture](#multi-service-architecture)
5. [Graceful Shutdown](#graceful-shutdown)

---

## Saga State Machines

Sagas coordinate long-running workflows across multiple services. MassTransit's 
Automatonymous state machine is the preferred approach — it's explicit, testable, 
and persisted to a database.

### When to Use Sagas

- Multi-step workflows (payment → confirmation → notification)
- Processes that span multiple services
- Workflows with timeout/compensation logic
- Anything requiring "rollback" or "undo" on failure

### Saga State (Entity)

```csharp
using MassTransit;

namespace ProjectName.Domain.Modules.Subscription.Entities;

public class SubscriptionRenewalState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = null!;

    public Guid SubscriptionId { get; set; }
    public Guid UserId { get; set; }
    public decimal Amount { get; set; }
    public string? PaymentTransactionId { get; set; }
    public string? FailureReason { get; set; }
    public int RetryCount { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
}
```

### Saga State Machine

```csharp
using MassTransit;

namespace ProjectName.Infrastructure.Sagas;

public class SubscriptionRenewalStateMachine
    : MassTransitStateMachine<SubscriptionRenewalState>
{
    // States
    public State PaymentPending { get; private set; } = null!;
    public State PaymentProcessing { get; private set; } = null!;
    public State PaymentCompleted { get; private set; } = null!;
    public State PaymentFailed { get; private set; } = null!;
    public State NotificationSent { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Faulted { get; private set; } = null!;

    // Events
    public Event<RenewalInitiatedEvent> RenewalInitiated { get; private set; } = null!;
    public Event<PaymentCompletedEvent> PaymentCompleted_ { get; private set; } = null!;
    public Event<PaymentFailedEvent> PaymentFailed_ { get; private set; } = null!;
    public Event<NotificationSentEvent> NotificationSent_ { get; private set; } = null!;

    public SubscriptionRenewalStateMachine()
    {
        InstanceState(x => x.CurrentState);

        // Event correlation — how to match events to saga instances
        Event(() => RenewalInitiated, x =>
            x.CorrelateById(ctx => ctx.Message.SubscriptionId));
        Event(() => PaymentCompleted_, x =>
            x.CorrelateById(ctx => ctx.Message.SubscriptionId));
        Event(() => PaymentFailed_, x =>
            x.CorrelateById(ctx => ctx.Message.SubscriptionId));
        Event(() => NotificationSent_, x =>
            x.CorrelateById(ctx => ctx.Message.SubscriptionId));

        // State transitions
        Initially(
            When(RenewalInitiated)
                .Then(ctx =>
                {
                    ctx.Saga.SubscriptionId = ctx.Message.SubscriptionId;
                    ctx.Saga.UserId = ctx.Message.UserId;
                    ctx.Saga.Amount = ctx.Message.Amount;
                    ctx.Saga.CreatedAt = DateTime.UtcNow;
                })
                .Publish(ctx => new ProcessPaymentCommand(
                    SubscriptionId: ctx.Saga.SubscriptionId,
                    Amount: ctx.Saga.Amount,
                    IdempotencyKey: ctx.Saga.CorrelationId.ToString()))
                .TransitionTo(PaymentProcessing)
        );

        During(PaymentProcessing,
            When(PaymentCompleted_)
                .Then(ctx =>
                {
                    ctx.Saga.PaymentTransactionId = ctx.Message.TransactionId;
                })
                .Publish(ctx => new SendNotificationCommand(
                    UserId: ctx.Saga.UserId,
                    Title: "Subscription Renewed",
                    Body: $"Your subscription has been renewed. Amount: {ctx.Saga.Amount:C}",
                    Type: NotificationType.SubscriptionRenewal,
                    ReferenceId: ctx.Saga.SubscriptionId))
                .TransitionTo(PaymentCompleted),

            When(PaymentFailed_)
                .Then(ctx =>
                {
                    ctx.Saga.FailureReason = ctx.Message.Reason;
                    ctx.Saga.RetryCount++;
                })
                .IfElse(ctx => ctx.Saga.RetryCount < 3,
                    // Retry: re-send payment command
                    retry => retry
                        .Publish(ctx => new ProcessPaymentCommand(
                            SubscriptionId: ctx.Saga.SubscriptionId,
                            Amount: ctx.Saga.Amount,
                            IdempotencyKey: $"{ctx.Saga.CorrelationId}-{ctx.Saga.RetryCount}"))
                        .TransitionTo(PaymentProcessing),
                    // Give up: mark as failed
                    giveUp => giveUp
                        .Publish(ctx => new SendNotificationCommand(
                            UserId: ctx.Saga.UserId,
                            Title: "Payment Failed",
                            Body: $"We couldn't renew your subscription: {ctx.Saga.FailureReason}",
                            Type: NotificationType.PaymentFailed,
                            ReferenceId: ctx.Saga.SubscriptionId))
                        .TransitionTo(Faulted)
                        .Finalize())
        );

        During(PaymentCompleted,
            When(NotificationSent_)
                .Then(ctx => ctx.Saga.CompletedAt = DateTime.UtcNow)
                .TransitionTo(Completed)
                .Finalize()
        );

        // Cleanup completed sagas after 30 days
        SetCompletedWhenFinalized();
    }
}
```

### Saga Persistence (EF Core)

```csharp
// DbContext configuration
public class SubscriptionRenewalStateMap :
    SagaClassMap<SubscriptionRenewalState>
{
    protected override void Configure(
        EntityTypeBuilder<SubscriptionRenewalState> builder, ModelBuilder model)
    {
        builder.ToTable("subscription_renewal_sagas");

        builder.Property(x => x.CorrelationId)
            .HasColumnName("correlation_id");

        builder.Property(x => x.CurrentState)
            .HasColumnName("current_state")
            .HasMaxLength(64);

        builder.HasIndex(x => x.SubscriptionId)
            .HasDatabaseName("ix_renewal_sagas_subscription_id");
    }
}

// Registration in Program.cs
cfg.AddSagaStateMachine<SubscriptionRenewalStateMachine, SubscriptionRenewalState>()
    .EntityFrameworkRepository(r =>
    {
        r.ExistingDbContext<ApplicationDbContext>();
        r.UsePostgres();
    });
```

### Saga with Timeouts

```csharp
// Add schedule for timeout
public Schedule<SubscriptionRenewalState, PaymentTimeoutExpired> PaymentTimeout
    { get; private set; } = null!;

// In constructor
Schedule(() => PaymentTimeout, x => x.PaymentTimeoutTokenId,
    s => s.Delay = TimeSpan.FromMinutes(30));

// In state machine
During(PaymentProcessing,
    When(PaymentTimeout.Received)
        .Then(ctx =>
        {
            ctx.Saga.FailureReason = "Payment processing timed out";
        })
        .TransitionTo(Faulted)
        .Finalize(),

    When(RenewalInitiated)
        .Schedule(PaymentTimeout, ctx => new PaymentTimeoutExpired(
            SubscriptionId: ctx.Saga.SubscriptionId))
        // ... rest of transition
);

// Cancel timeout when payment succeeds
During(PaymentProcessing,
    When(PaymentCompleted_)
        .Unschedule(PaymentTimeout)
        // ... rest of transition
);
```

---

## Outbox Pattern

The outbox ensures messages are published only when the database transaction commits.
Without it, you risk: save succeeds + publish fails = lost message, or publish succeeds 
+ save fails = phantom message.

### EF Core Outbox Configuration

```csharp
builder.Services.AddMassTransit(cfg =>
{
    cfg.AddEntityFrameworkOutbox<ApplicationDbContext>(o =>
    {
        o.UsePostgres();
        o.UseBusOutbox();            // Messages sent via bus
        o.QueryDelay = TimeSpan.FromSeconds(1);   // Poll interval
        o.DuplicateDetectionWindow = TimeSpan.FromMinutes(5);
    });

    cfg.AddConsumers(typeof(Program).Assembly);

    cfg.UsingRabbitMq((context, rabbit) =>
    {
        // ... RabbitMQ configuration ...
        rabbit.ConfigureEndpoints(context);
    });
});
```

### Using the Outbox in Handlers

The outbox is transparent — use `IPublishEndpoint` / `ISendEndpointProvider` as normal. 
Messages are written to the outbox table in the same transaction as your entity changes, 
then delivered asynchronously.

```csharp
public sealed class CreateAppointmentCommandHandler(
    IApplicationDbContext dbContext,
    IPublishEndpoint publishEndpoint
) : IRequestHandler<CreateAppointmentCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(
        CreateAppointmentCommand request,
        CancellationToken cancellationToken)
    {
        var appointment = new Appointment { /* ... */ };
        dbContext.Appointments.Add(appointment);

        // This publish is captured by the outbox — not sent until SaveChanges commits
        await publishEndpoint.Publish(new AppointmentCreatedEvent(/* ... */), cancellationToken);

        // Both the entity AND the outbox message commit atomically
        await dbContext.SaveChangesAsync(cancellationToken);

        return Result<Guid>.Success(appointment.Id);
    }
}
```

### Outbox Tables Migration

MassTransit creates outbox tables automatically. Add this migration if needed:

```csharp
// In a migration
migrationBuilder.Sql(@"
    -- MassTransit outbox tables (created by AddEntityFrameworkOutbox)
    -- inbox_state, outbox_state, outbox_message
    -- These are managed by MassTransit — do not modify manually
");
```

---

## Message Topology

### Exchange & Queue Naming

MassTransit creates exchanges and queues automatically. Default naming:

```
Exchange: ProjectName.Contracts.Events:AppointmentCreatedEvent (fanout)
Queue: appointment-created-event (consumer-specific)
```

### Custom Topology

```csharp
rabbit.MessageTopology.SetEntityNameFormatter(new CustomEntityNameFormatter());

// Or per-message configuration
rabbit.Publish<AppointmentCreatedEvent>(x =>
{
    x.ExchangeType = ExchangeType.Topic;
});

rabbit.Send<SendNotificationCommand>(x =>
{
    x.UseRoutingKeyFormatter(ctx => ctx.Message.Type.ToString());
});
```

---

## Multi-Service Architecture

### Shared Contracts Project

```
src/
├── ProjectName.Contracts/          # Shared by all services
│   ├── Events/
│   │   ├── AppointmentCreatedEvent.cs
│   │   ├── PaymentCompletedEvent.cs
│   │   └── PaymentFailedEvent.cs
│   ├── Commands/
│   │   ├── SendNotificationCommand.cs
│   │   └── ProcessPaymentCommand.cs
│   └── Responses/
│       └── MentorAvailabilityResponse.cs
├── ProjectName.API/                # HTTP API service
├── ProjectName.Worker/             # Background consumer service
└── ProjectName.Scheduler/          # Scheduled job service
```

### Worker Service (Consumer-Only)

```csharp
// ProjectName.Worker/Program.cs
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddDbContext<ApplicationDbContext>(/* ... */);

builder.Services.AddMassTransit(cfg =>
{
    cfg.AddConsumers(typeof(Program).Assembly);

    cfg.UsingRabbitMq((context, rabbit) =>
    {
        rabbit.Host(builder.Configuration["RabbitMQ:Host"]!);
        rabbit.ConfigureEndpoints(context);
    });
});

var host = builder.Build();
await host.RunAsync();
```

---

## Graceful Shutdown

```csharp
// MassTransit handles graceful shutdown automatically via IHostedService.
// When the host stops:
// 1. Stops accepting new messages
// 2. Waits for in-progress consumers to complete (up to StopTimeout)
// 3. Closes RabbitMQ connections

// Configure shutdown timeout (default 30 seconds)
builder.Services.AddOptions<MassTransitHostOptions>()
    .Configure(options =>
    {
        options.WaitUntilStarted = true;
        options.StopTimeout = TimeSpan.FromSeconds(60);
    });
```
