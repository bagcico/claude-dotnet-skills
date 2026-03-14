# Minimal API Endpoints

## Endpoint Class Template

One static class per module. All endpoints in a single `Map{Module}Endpoints` extension.

```csharp
using MediatR;
using Microsoft.AspNetCore.Mvc;

namespace ProjectName.API.Endpoints.Appointment;

public static class AppointmentEndpoints
{
    public static void MapAppointmentEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/appointments")
            .WithTags("Appointments")
            .RequireAuthorization();

        group.MapPost("/", CreateAppointment)
            .WithName(nameof(CreateAppointment))
            .Produces<Guid>(StatusCodes.Status201Created)
            .ProducesValidationProblem()
            .Produces(StatusCodes.Status400BadRequest);

        group.MapGet("/{id:guid}", GetAppointment)
            .WithName(nameof(GetAppointment))
            .Produces<AppointmentDto>()
            .Produces(StatusCodes.Status404NotFound);

        group.MapGet("/", GetAppointmentList)
            .WithName(nameof(GetAppointmentList))
            .Produces<PaginatedList<AppointmentDto>>();

        group.MapPut("/{id:guid}", UpdateAppointment)
            .WithName(nameof(UpdateAppointment));

        group.MapDelete("/{id:guid}", DeleteAppointment)
            .WithName(nameof(DeleteAppointment))
            .Produces(StatusCodes.Status204NoContent);
    }

    private static async Task<IResult> CreateAppointment(
        [FromBody] CreateAppointmentCommand command,
        ISender sender, CancellationToken cancellationToken)
    {
        var result = await sender.Send(command, cancellationToken);
        return result.IsSuccess
            ? Results.Created($"/api/appointments/{result.Value}", result.Value)
            : ToErrorResult(result);
    }

    private static async Task<IResult> GetAppointment(
        Guid id, ISender sender, CancellationToken cancellationToken)
    {
        var result = await sender.Send(new GetAppointmentQuery(id), cancellationToken);
        return result.IsSuccess ? Results.Ok(result.Value) : Results.NotFound(result.Error);
    }

    private static async Task<IResult> GetAppointmentList(
        [AsParameters] GetAppointmentListQuery query,
        ISender sender, CancellationToken cancellationToken)
    {
        var result = await sender.Send(query, cancellationToken);
        return result.IsSuccess ? Results.Ok(result.Value) : Results.BadRequest(result.Error);
    }

    private static async Task<IResult> UpdateAppointment(
        Guid id, [FromBody] UpdateAppointmentCommand command,
        ISender sender, CancellationToken cancellationToken)
    {
        if (id != command.Id) return Results.BadRequest("ID mismatch.");
        var result = await sender.Send(command, cancellationToken);
        return result.IsSuccess ? Results.Ok(result.Value) : Results.NotFound(result.Error);
    }

    private static async Task<IResult> DeleteAppointment(
        Guid id, ISender sender, CancellationToken cancellationToken)
    {
        var result = await sender.Send(new DeleteAppointmentCommand(id), cancellationToken);
        return result.IsSuccess ? Results.NoContent() : Results.NotFound(result.Error);
    }

    // Shared helper: Result<T> → IResult with validation error support
    private static IResult ToErrorResult<T>(Result<T> result)
    {
        if (result.ValidationErrors is { Length: > 0 })
            return Results.ValidationProblem(
                new Dictionary<string, string[]> { { "Errors", result.ValidationErrors } });
        return Results.BadRequest(new { error = result.Error });
    }
}
```

## Registration in Program.cs

```csharp
app.MapAppointmentEndpoints();
app.MapPaymentEndpoints();
// ... other modules
```
