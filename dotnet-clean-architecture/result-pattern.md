# Result\<T\> Pattern, DTOs & Mapping

## Result\<T\> Implementation

```csharp
namespace ProjectName.Application.Common.Results;

public class Result
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public string? Error { get; }
    public string[]? ValidationErrors { get; }

    protected Result(bool isSuccess, string? error, string[]? validationErrors = null)
    {
        IsSuccess = isSuccess;
        Error = error;
        ValidationErrors = validationErrors;
    }

    public static Result Success() => new(true, null);
    public static Result Failure(string error) => new(false, error);
    public static Result ValidationFailure(string[] errors)
        => new(false, "Validation failed.", errors);
}

public class Result<T> : Result
{
    public T? Value { get; }

    private Result(bool isSuccess, T? value, string? error, string[]? validationErrors = null)
        : base(isSuccess, error, validationErrors) => Value = value;

    public static Result<T> Success(T value) => new(true, value, null);
    public new static Result<T> Failure(string error) => new(false, default, error);
    public new static Result<T> ValidationFailure(string[] errors)
        => new(false, default, "Validation failed.", errors);
}
```

## PaginatedList\<T\>

```csharp
public class PaginatedList<T>
{
    public List<T> Items { get; }
    public int PageNumber { get; }
    public int PageSize { get; }
    public int TotalCount { get; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => PageNumber > 1;
    public bool HasNextPage => PageNumber < TotalPages;

    public PaginatedList(List<T> items, int totalCount, int pageNumber, int pageSize)
    {
        Items = items; TotalCount = totalCount;
        PageNumber = pageNumber; PageSize = pageSize;
    }

    public static async Task<PaginatedList<T>> CreateAsync(
        IQueryable<T> source, int pageNumber, int pageSize,
        CancellationToken cancellationToken = default)
    {
        var totalCount = await source.CountAsync(cancellationToken);
        var items = await source.Skip((pageNumber - 1) * pageSize)
            .Take(pageSize).ToListAsync(cancellationToken);
        return new PaginatedList<T>(items, totalCount, pageNumber, pageSize);
    }
}
```

## Base Entities

```csharp
public abstract class BaseEntity<TId> where TId : notnull
{
    public TId Id { get; set; } = default!;
}

public abstract class AuditableEntity<TId> : BaseEntity<TId> where TId : notnull
{
    public DateTime CreatedAt { get; set; }
    public string? CreatedBy { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public string? UpdatedBy { get; set; }
}
```

## DTO Pattern

DTOs are immutable `record` types:

```csharp
public sealed record AppointmentDto(
    Guid Id, Guid MentorId, string MentorName, Guid StudentId, string StudentName,
    DateTime ScheduledAt, int DurationMinutes, string Status, string? Notes,
    DateTime CreatedAt
);
```

## Mapping Extensions

One static class per module with `ToDto()` and `ToDtoList()` extension methods:

```csharp
namespace ProjectName.Application.Modules.Appointment.Mappings;

public static class AppointmentMappingExtensions
{
    public static AppointmentDto ToDto(this Appointment entity) => new(
        Id: entity.Id,
        MentorId: entity.MentorId,
        MentorName: entity.Mentor?.FullName ?? string.Empty,
        StudentId: entity.StudentId,
        StudentName: entity.Student?.FullName ?? string.Empty,
        ScheduledAt: entity.ScheduledAt,
        DurationMinutes: entity.DurationMinutes,
        Status: entity.Status.ToString(),
        Notes: entity.Notes,
        CreatedAt: entity.CreatedAt
    );

    public static List<AppointmentDto> ToDtoList(
        this IEnumerable<Appointment> entities) =>
        entities.Select(e => e.ToDto()).ToList();
}
```
