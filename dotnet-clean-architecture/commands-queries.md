# Commands & Queries

Templates for Command/Query records, handlers, and validators.

## Command Template

### Command Record

```csharp
using MediatR;
using ProjectName.Application.Common.Results;

namespace ProjectName.Application.Modules.Appointment.Commands.CreateAppointment;

public sealed record CreateAppointmentCommand(
    Guid MentorId,
    Guid StudentId,
    DateTime ScheduledAt,
    int DurationMinutes,
    string? Notes
) : IRequest<Result<Guid>>;
```

### Command Handler

Primary constructor for DI. Direct DbContext access, business rules in handler.

```csharp
using MediatR;
using Microsoft.EntityFrameworkCore;
using ProjectName.Application.Common.Interfaces;
using ProjectName.Application.Common.Results;

namespace ProjectName.Application.Modules.Appointment.Commands.CreateAppointment;

public sealed class CreateAppointmentCommandHandler(
    IApplicationDbContext dbContext
) : IRequestHandler<CreateAppointmentCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(
        CreateAppointmentCommand request, CancellationToken cancellationToken)
    {
        // Business rule validation (not input validation — that's in the Validator)
        var mentorExists = await dbContext.Mentors
            .AnyAsync(m => m.Id == request.MentorId, cancellationToken);

        if (!mentorExists)
            return Result<Guid>.Failure("Mentor not found.");

        var hasConflict = await dbContext.Appointments
            .AnyAsync(a =>
                a.MentorId == request.MentorId &&
                a.Status != AppointmentStatus.Cancelled &&
                a.ScheduledAt < request.ScheduledAt.AddMinutes(request.DurationMinutes) &&
                a.ScheduledAt.AddMinutes(a.DurationMinutes) > request.ScheduledAt,
                cancellationToken);

        if (hasConflict)
            return Result<Guid>.Failure("Mentor has a conflicting appointment.");

        var appointment = new Appointment
        {
            Id = Guid.NewGuid(),
            MentorId = request.MentorId,
            StudentId = request.StudentId,
            ScheduledAt = request.ScheduledAt,
            DurationMinutes = request.DurationMinutes,
            Notes = request.Notes,
            Status = AppointmentStatus.Pending
        };

        dbContext.Appointments.Add(appointment);
        await dbContext.SaveChangesAsync(cancellationToken);

        return Result<Guid>.Success(appointment.Id);
    }
}
```

### Command Validator

Input validation only (format, length, required). Business rules belong in handler.

```csharp
using FluentValidation;

namespace ProjectName.Application.Modules.Appointment.Commands.CreateAppointment;

public sealed class CreateAppointmentCommandValidator
    : AbstractValidator<CreateAppointmentCommand>
{
    public CreateAppointmentCommandValidator()
    {
        RuleFor(x => x.MentorId).NotEmpty().WithMessage("Mentor ID is required.");
        RuleFor(x => x.StudentId).NotEmpty().WithMessage("Student ID is required.");
        RuleFor(x => x.ScheduledAt).GreaterThan(DateTime.UtcNow)
            .WithMessage("Appointment must be scheduled in the future.");
        RuleFor(x => x.DurationMinutes).InclusiveBetween(15, 120)
            .WithMessage("Duration must be between 15 and 120 minutes.");
    }
}
```

### Update Command Pattern

```csharp
public sealed record UpdateAppointmentCommand(
    Guid Id, DateTime ScheduledAt, int DurationMinutes, string? Notes
) : IRequest<Result<AppointmentDto>>;

// Handler: fetch → modify → save → return ToDto()
public async Task<Result<AppointmentDto>> Handle(...)
{
    var entity = await dbContext.Appointments
        .FirstOrDefaultAsync(a => a.Id == request.Id, cancellationToken);
    if (entity is null)
        return Result<AppointmentDto>.Failure("Appointment not found.");

    entity.ScheduledAt = request.ScheduledAt;
    entity.DurationMinutes = request.DurationMinutes;
    entity.Notes = request.Notes;
    await dbContext.SaveChangesAsync(cancellationToken);

    return Result<AppointmentDto>.Success(entity.ToDto());
}
```

### Delete Command Pattern

```csharp
public sealed record DeleteAppointmentCommand(Guid Id) : IRequest<Result>;
// Handler returns Result (non-generic) for void operations
```

## Query Templates

### Single Entity Query

```csharp
public sealed record GetAppointmentQuery(Guid Id)
    : IRequest<Result<AppointmentDto>>;

public sealed class GetAppointmentQueryHandler(
    IApplicationDbContext dbContext
) : IRequestHandler<GetAppointmentQuery, Result<AppointmentDto>>
{
    public async Task<Result<AppointmentDto>> Handle(
        GetAppointmentQuery request, CancellationToken cancellationToken)
    {
        var appointment = await dbContext.Appointments
            .AsNoTracking()
            .Include(a => a.Mentor)
            .Include(a => a.Student)
            .FirstOrDefaultAsync(a => a.Id == request.Id, cancellationToken);

        if (appointment is null)
            return Result<AppointmentDto>.Failure("Appointment not found.");

        return Result<AppointmentDto>.Success(appointment.ToDto());
    }
}
```

### List Query with Pagination

```csharp
public sealed record GetAppointmentListQuery(
    Guid? MentorId, Guid? StudentId, AppointmentStatus? Status,
    DateTime? FromDate, DateTime? ToDate,
    int PageNumber = 1, int PageSize = 20
) : IRequest<Result<PaginatedList<AppointmentDto>>>;

// Handler: build IQueryable step by step with conditional Where clauses
// Always: AsNoTracking(), OrderByDescending, then PaginatedList.CreateAsync()
```
