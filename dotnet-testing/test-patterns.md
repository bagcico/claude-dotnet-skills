# Test Patterns

## Table of Contents
1. [Command Handler Tests](#command-handler-tests)
2. [Query Handler Tests](#query-handler-tests)
3. [Validator Tests](#validator-tests)
4. [Mapping Tests](#mapping-tests)
5. [Pipeline Behavior Tests](#pipeline-behavior-tests)
6. [Integration Tests (WebApplicationFactory)](#integration-tests)
7. [Consumer Tests (MassTransit)](#consumer-tests)
8. [Test Utilities & Factories](#test-utilities--factories)
9. [Architecture Tests](#architecture-tests)

---

## Command Handler Tests

### FakeDbContext Setup

Use an in-memory SQLite database for unit tests — it supports relational features 
that the EF InMemory provider doesn't (foreign keys, transactions):

```csharp
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;

namespace ProjectName.UnitTests.Fakes;

public static class FakeDbContext
{
    public static ApplicationDbContext Create()
    {
        var connection = new SqliteConnection("DataSource=:memory:");
        connection.Open();

        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlite(connection)
            .Options;

        var context = new ApplicationDbContext(options);
        context.Database.EnsureCreated();
        return context;
    }
}
```

### Testing a Create Command Handler

```csharp
using FluentAssertions;
using ProjectName.Application.Modules.Appointment.Commands.CreateAppointment;
using ProjectName.Domain.Modules.Appointment.Entities;
using ProjectName.Domain.Modules.Appointment.Enums;
using ProjectName.UnitTests.Fakes;

namespace ProjectName.UnitTests.Modules.Appointment.Commands;

public class CreateAppointmentCommandHandlerTests : IDisposable
{
    private readonly ApplicationDbContext _dbContext;
    private readonly CreateAppointmentCommandHandler _handler;

    public CreateAppointmentCommandHandlerTests()
    {
        _dbContext = FakeDbContext.Create();
        _handler = new CreateAppointmentCommandHandler(_dbContext);
    }

    [Fact]
    public async Task Handle_ValidCommand_ReturnsSuccessWithNewId()
    {
        // Arrange
        var mentor = TestDataFactory.CreateMentor();
        var student = TestDataFactory.CreateStudent();
        _dbContext.Mentors.Add(mentor);
        _dbContext.Students.Add(student);
        await _dbContext.SaveChangesAsync();

        var command = new CreateAppointmentCommand(
            MentorId: mentor.Id,
            StudentId: student.Id,
            ScheduledAt: DateTime.UtcNow.AddDays(1),
            DurationMinutes: 60,
            Notes: "First session");

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeEmpty();

        var appointment = await _dbContext.Appointments
            .FirstOrDefaultAsync(a => a.Id == result.Value);
        appointment.Should().NotBeNull();
        appointment!.MentorId.Should().Be(mentor.Id);
        appointment.Status.Should().Be(AppointmentStatus.Pending);
    }

    [Fact]
    public async Task Handle_MentorNotFound_ReturnsFailure()
    {
        // Arrange
        var command = new CreateAppointmentCommand(
            MentorId: Guid.NewGuid(),
            StudentId: Guid.NewGuid(),
            ScheduledAt: DateTime.UtcNow.AddDays(1),
            DurationMinutes: 60,
            Notes: null);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("Mentor not found");
    }

    [Fact]
    public async Task Handle_ConflictingAppointment_ReturnsFailure()
    {
        // Arrange
        var mentor = TestDataFactory.CreateMentor();
        _dbContext.Mentors.Add(mentor);

        var existingAppointment = TestDataFactory.CreateAppointment(
            mentorId: mentor.Id,
            scheduledAt: DateTime.UtcNow.AddDays(1).Date.AddHours(10),
            durationMinutes: 60);
        _dbContext.Appointments.Add(existingAppointment);
        await _dbContext.SaveChangesAsync();

        // Same mentor, overlapping time
        var command = new CreateAppointmentCommand(
            MentorId: mentor.Id,
            StudentId: Guid.NewGuid(),
            ScheduledAt: DateTime.UtcNow.AddDays(1).Date.AddHours(10).AddMinutes(30),
            DurationMinutes: 60,
            Notes: null);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("conflicting");
    }

    public void Dispose() => _dbContext.Dispose();
}
```

### Testing an Update Command Handler

```csharp
[Fact]
public async Task Handle_ExistingAppointment_UpdatesAndReturnsDto()
{
    // Arrange
    var appointment = TestDataFactory.CreateAppointment();
    _dbContext.Appointments.Add(appointment);
    await _dbContext.SaveChangesAsync();

    var command = new UpdateAppointmentCommand(
        Id: appointment.Id,
        ScheduledAt: DateTime.UtcNow.AddDays(3),
        DurationMinutes: 90,
        Notes: "Updated notes");

    // Act
    var result = await _handler.Handle(command, CancellationToken.None);

    // Assert
    result.IsSuccess.Should().BeTrue();
    result.Value!.DurationMinutes.Should().Be(90);
    result.Value.Notes.Should().Be("Updated notes");
}

[Fact]
public async Task Handle_NonexistentAppointment_ReturnsNotFound()
{
    // Arrange
    var command = new UpdateAppointmentCommand(
        Id: Guid.NewGuid(),
        ScheduledAt: DateTime.UtcNow.AddDays(3),
        DurationMinutes: 60,
        Notes: null);

    // Act
    var result = await _handler.Handle(command, CancellationToken.None);

    // Assert
    result.IsFailure.Should().BeTrue();
    result.Error.Should().Contain("not found");
}
```

---

## Query Handler Tests

```csharp
public class GetAppointmentListQueryHandlerTests : IDisposable
{
    private readonly ApplicationDbContext _dbContext;
    private readonly GetAppointmentListQueryHandler _handler;

    public GetAppointmentListQueryHandlerTests()
    {
        _dbContext = FakeDbContext.Create();
        _handler = new GetAppointmentListQueryHandler(_dbContext);
    }

    [Fact]
    public async Task Handle_WithMentorFilter_ReturnsOnlyMentorAppointments()
    {
        // Arrange
        var mentor1 = TestDataFactory.CreateMentor();
        var mentor2 = TestDataFactory.CreateMentor();
        _dbContext.Mentors.AddRange(mentor1, mentor2);

        _dbContext.Appointments.AddRange(
            TestDataFactory.CreateAppointment(mentorId: mentor1.Id),
            TestDataFactory.CreateAppointment(mentorId: mentor1.Id),
            TestDataFactory.CreateAppointment(mentorId: mentor2.Id));
        await _dbContext.SaveChangesAsync();

        var query = new GetAppointmentListQuery(
            MentorId: mentor1.Id,
            StudentId: null,
            Status: null,
            FromDate: null,
            ToDate: null);

        // Act
        var result = await _handler.Handle(query, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value!.Items.Should().HaveCount(2);
        result.Value.Items.Should().AllSatisfy(a =>
            a.MentorId.Should().Be(mentor1.Id));
    }

    [Fact]
    public async Task Handle_WithPagination_ReturnsCorrectPage()
    {
        // Arrange
        var mentor = TestDataFactory.CreateMentor();
        _dbContext.Mentors.Add(mentor);

        for (int i = 0; i < 25; i++)
            _dbContext.Appointments.Add(
                TestDataFactory.CreateAppointment(mentorId: mentor.Id));
        await _dbContext.SaveChangesAsync();

        var query = new GetAppointmentListQuery(
            MentorId: null, StudentId: null, Status: null,
            FromDate: null, ToDate: null,
            PageNumber: 2, PageSize: 10);

        // Act
        var result = await _handler.Handle(query, CancellationToken.None);

        // Assert
        result.Value!.Items.Should().HaveCount(10);
        result.Value.TotalCount.Should().Be(25);
        result.Value.PageNumber.Should().Be(2);
        result.Value.HasPreviousPage.Should().BeTrue();
        result.Value.HasNextPage.Should().BeTrue();
    }

    public void Dispose() => _dbContext.Dispose();
}
```

---

## Validator Tests

Test each validation rule independently:

```csharp
using FluentAssertions;
using FluentValidation.TestHelper;

namespace ProjectName.UnitTests.Modules.Appointment.Validators;

public class CreateAppointmentCommandValidatorTests
{
    private readonly CreateAppointmentCommandValidator _validator = new();

    [Fact]
    public void Validate_ValidCommand_PassesAllRules()
    {
        // Arrange
        var command = new CreateAppointmentCommand(
            MentorId: Guid.NewGuid(),
            StudentId: Guid.NewGuid(),
            ScheduledAt: DateTime.UtcNow.AddDays(1),
            DurationMinutes: 60,
            Notes: "Test notes");

        // Act
        var result = _validator.TestValidate(command);

        // Assert
        result.ShouldNotHaveAnyValidationErrors();
    }

    [Fact]
    public void Validate_EmptyMentorId_FailsValidation()
    {
        var command = new CreateAppointmentCommand(
            MentorId: Guid.Empty,
            StudentId: Guid.NewGuid(),
            ScheduledAt: DateTime.UtcNow.AddDays(1),
            DurationMinutes: 60,
            Notes: null);

        var result = _validator.TestValidate(command);

        result.ShouldHaveValidationErrorFor(x => x.MentorId)
            .WithErrorMessage("Mentor ID is required.");
    }

    [Fact]
    public void Validate_PastScheduledAt_FailsValidation()
    {
        var command = new CreateAppointmentCommand(
            MentorId: Guid.NewGuid(),
            StudentId: Guid.NewGuid(),
            ScheduledAt: DateTime.UtcNow.AddHours(-1),
            DurationMinutes: 60,
            Notes: null);

        var result = _validator.TestValidate(command);

        result.ShouldHaveValidationErrorFor(x => x.ScheduledAt)
            .WithErrorMessage("Appointment must be scheduled in the future.");
    }

    [Theory]
    [InlineData(0)]
    [InlineData(10)]
    [InlineData(150)]
    [InlineData(-1)]
    public void Validate_InvalidDuration_FailsValidation(int duration)
    {
        var command = new CreateAppointmentCommand(
            MentorId: Guid.NewGuid(),
            StudentId: Guid.NewGuid(),
            ScheduledAt: DateTime.UtcNow.AddDays(1),
            DurationMinutes: duration,
            Notes: null);

        var result = _validator.TestValidate(command);

        result.ShouldHaveValidationErrorFor(x => x.DurationMinutes);
    }

    [Theory]
    [InlineData(15)]
    [InlineData(60)]
    [InlineData(120)]
    public void Validate_ValidDuration_PassesValidation(int duration)
    {
        var command = new CreateAppointmentCommand(
            MentorId: Guid.NewGuid(),
            StudentId: Guid.NewGuid(),
            ScheduledAt: DateTime.UtcNow.AddDays(1),
            DurationMinutes: duration,
            Notes: null);

        var result = _validator.TestValidate(command);

        result.ShouldNotHaveValidationErrorFor(x => x.DurationMinutes);
    }
}
```

---

## Mapping Tests

```csharp
public class AppointmentMappingExtensionsTests
{
    [Fact]
    public void ToDto_ValidEntity_MapsAllProperties()
    {
        // Arrange
        var mentor = TestDataFactory.CreateMentor(fullName: "Ahmet Yılmaz");
        var student = TestDataFactory.CreateStudent(fullName: "Ayşe Demir");
        var entity = TestDataFactory.CreateAppointment(
            mentorId: mentor.Id,
            scheduledAt: new DateTime(2026, 4, 1, 10, 0, 0, DateTimeKind.Utc));
        entity.Mentor = mentor;
        entity.Student = student;

        // Act
        var dto = entity.ToDto();

        // Assert
        dto.Id.Should().Be(entity.Id);
        dto.MentorId.Should().Be(mentor.Id);
        dto.MentorName.Should().Be("Ahmet Yılmaz");
        dto.StudentId.Should().Be(student.Id);
        dto.StudentName.Should().Be("Ayşe Demir");
        dto.ScheduledAt.Should().Be(entity.ScheduledAt);
        dto.DurationMinutes.Should().Be(entity.DurationMinutes);
    }

    [Fact]
    public void ToDto_NullNavigations_ReturnsEmptyStrings()
    {
        var entity = TestDataFactory.CreateAppointment();
        entity.Mentor = null!;
        entity.Student = null!;

        var dto = entity.ToDto();

        dto.MentorName.Should().BeEmpty();
        dto.StudentName.Should().BeEmpty();
    }
}
```

---

## Pipeline Behavior Tests

### ValidationBehavior Tests

```csharp
public class ValidationBehaviorTests
{
    [Fact]
    public async Task Handle_NoValidators_CallsNext()
    {
        // Arrange
        var validators = Enumerable.Empty<IValidator<TestCommand>>();
        var behavior = new ValidationBehavior<TestCommand, Result<string>>(validators);
        var nextCalled = false;

        // Act
        var result = await behavior.Handle(
            new TestCommand("valid"),
            () => { nextCalled = true; return Task.FromResult(Result<string>.Success("ok")); },
            CancellationToken.None);

        // Assert
        nextCalled.Should().BeTrue();
        result.IsSuccess.Should().BeTrue();
    }

    [Fact]
    public async Task Handle_ValidationFails_ReturnsResultFailureWithoutCallingNext()
    {
        // Arrange
        var validator = new TestCommandValidator(); // Requires non-empty Name
        var behavior = new ValidationBehavior<TestCommand, Result<string>>(
            new[] { validator });
        var nextCalled = false;

        // Act
        var result = await behavior.Handle(
            new TestCommand(""), // Invalid
            () => { nextCalled = true; return Task.FromResult(Result<string>.Success("ok")); },
            CancellationToken.None);

        // Assert
        nextCalled.Should().BeFalse();
        result.IsFailure.Should().BeTrue();
        result.ValidationErrors.Should().NotBeEmpty();
    }

    // Test helpers
    private record TestCommand(string Name) : IRequest<Result<string>>;

    private class TestCommandValidator : AbstractValidator<TestCommand>
    {
        public TestCommandValidator()
        {
            RuleFor(x => x.Name).NotEmpty().WithMessage("Name is required.");
        }
    }
}
```

---

## Integration Tests

### WebApplicationFactory Setup

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Testcontainers.PostgreSql;

namespace ProjectName.IntegrationTests.Fixtures;

public class IntegrationTestFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("testdb")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public WebApplicationFactory<Program> Factory { get; private set; } = null!;
    public HttpClient Client { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();

        Factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.UseEnvironment("Testing");
                builder.ConfigureServices(services =>
                {
                    // Replace real DB with Testcontainer
                    services.RemoveAll<DbContextOptions<ApplicationDbContext>>();
                    services.AddDbContext<ApplicationDbContext>(options =>
                        options.UseNpgsql(_postgres.GetConnectionString()));

                    // Replace Redis with in-memory
                    services.RemoveAll<ICacheService>();
                    services.AddSingleton<ICacheService, InMemoryCacheService>();

                    // Use MassTransit test harness
                    services.AddMassTransitTestHarness();
                });
            });

        Client = Factory.CreateClient();

        // Run migrations
        using var scope = Factory.Services.CreateScope();
        var dbContext = scope.ServiceProvider
            .GetRequiredService<ApplicationDbContext>();
        await dbContext.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        Client.Dispose();
        await Factory.DisposeAsync();
        await _postgres.DisposeAsync();
    }
}
```

### Endpoint Integration Tests

```csharp
[Collection("Integration")]
public class AppointmentEndpointTests(IntegrationTestFixture fixture)
    : IClassFixture<IntegrationTestFixture>
{
    [Fact]
    public async Task CreateAppointment_ValidRequest_Returns201()
    {
        // Arrange
        var mentor = await SeedMentor(fixture);
        var command = new
        {
            MentorId = mentor.Id,
            StudentId = Guid.NewGuid(),
            ScheduledAt = DateTime.UtcNow.AddDays(1),
            DurationMinutes = 60,
            Notes = "Integration test"
        };

        // Act
        var response = await fixture.Client.PostAsJsonAsync("/api/appointments", command);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var id = await response.Content.ReadFromJsonAsync<Guid>();
        id.Should().NotBeEmpty();
        response.Headers.Location.Should().NotBeNull();
    }

    [Fact]
    public async Task CreateAppointment_InvalidCommand_Returns400()
    {
        var command = new
        {
            MentorId = Guid.Empty,
            StudentId = Guid.Empty,
            ScheduledAt = DateTime.UtcNow.AddDays(-1),
            DurationMinutes = 0
        };

        var response = await fixture.Client.PostAsJsonAsync("/api/appointments", command);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task GetAppointment_ExistingId_Returns200WithDto()
    {
        // Arrange — create via API
        var mentor = await SeedMentor(fixture);
        var createResponse = await fixture.Client.PostAsJsonAsync("/api/appointments",
            new { MentorId = mentor.Id, StudentId = Guid.NewGuid(),
                  ScheduledAt = DateTime.UtcNow.AddDays(1), DurationMinutes = 60 });
        var id = await createResponse.Content.ReadFromJsonAsync<Guid>();

        // Act
        var response = await fixture.Client.GetAsync($"/api/appointments/{id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var dto = await response.Content.ReadFromJsonAsync<AppointmentDto>();
        dto.Should().NotBeNull();
        dto!.Id.Should().Be(id);
    }

    private static async Task<Mentor> SeedMentor(IntegrationTestFixture fixture)
    {
        using var scope = fixture.Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        var mentor = TestDataFactory.CreateMentor();
        db.Mentors.Add(mentor);
        await db.SaveChangesAsync();
        return mentor;
    }
}
```

---

## Consumer Tests

```csharp
public class AppointmentCreatedEventConsumerTests : IDisposable
{
    private readonly ApplicationDbContext _dbContext;

    public AppointmentCreatedEventConsumerTests()
    {
        _dbContext = FakeDbContext.Create();
    }

    [Fact]
    public async Task Consume_ValidEvent_CreatesNotification()
    {
        // Arrange
        await using var provider = new ServiceCollection()
            .AddMassTransitTestHarness(cfg =>
            {
                cfg.AddConsumer<AppointmentCreatedEventConsumer>();
            })
            .AddScoped<IApplicationDbContext>(_ => _dbContext)
            .AddLogging()
            .BuildServiceProvider(true);

        var harness = provider.GetRequiredService<ITestHarness>();
        await harness.Start();

        var @event = new AppointmentCreatedEvent(
            AppointmentId: Guid.NewGuid(),
            MentorId: Guid.NewGuid(),
            StudentId: Guid.NewGuid(),
            ScheduledAt: DateTime.UtcNow.AddDays(1),
            OccurredAt: DateTime.UtcNow);

        // Act
        await harness.Bus.Publish(@event);

        // Assert
        var consumerHarness = harness
            .GetConsumerHarness<AppointmentCreatedEventConsumer>();
        (await consumerHarness.Consumed.Any<AppointmentCreatedEvent>())
            .Should().BeTrue();

        var notification = await _dbContext.Notifications
            .FirstOrDefaultAsync(n => n.ReferenceId == @event.AppointmentId);
        notification.Should().NotBeNull();
        notification!.UserId.Should().Be(@event.MentorId);
    }

    public void Dispose() => _dbContext.Dispose();
}
```

---

## Test Utilities & Factories

### TestDataFactory with Bogus

```csharp
using Bogus;

namespace ProjectName.UnitTests.Fakes;

public static class TestDataFactory
{
    private static readonly Faker Faker = new("tr");

    public static Mentor CreateMentor(
        Guid? id = null,
        string? fullName = null,
        bool isActive = true)
    {
        return new Mentor
        {
            Id = id ?? Guid.NewGuid(),
            FullName = fullName ?? Faker.Name.FullName(),
            Email = Faker.Internet.Email(),
            IsActive = isActive,
            CreatedAt = DateTime.UtcNow
        };
    }

    public static Student CreateStudent(
        Guid? id = null,
        string? fullName = null)
    {
        return new Student
        {
            Id = id ?? Guid.NewGuid(),
            FullName = fullName ?? Faker.Name.FullName(),
            Email = Faker.Internet.Email(),
            CreatedAt = DateTime.UtcNow
        };
    }

    public static Appointment CreateAppointment(
        Guid? id = null,
        Guid? mentorId = null,
        Guid? studentId = null,
        DateTime? scheduledAt = null,
        int durationMinutes = 60,
        AppointmentStatus status = AppointmentStatus.Pending)
    {
        return new Appointment
        {
            Id = id ?? Guid.NewGuid(),
            MentorId = mentorId ?? Guid.NewGuid(),
            StudentId = studentId ?? Guid.NewGuid(),
            ScheduledAt = scheduledAt ?? DateTime.UtcNow.AddDays(Faker.Random.Int(1, 30)),
            DurationMinutes = durationMinutes,
            Status = status,
            CreatedAt = DateTime.UtcNow
        };
    }
}
```

### InMemoryCacheService for Tests

```csharp
public sealed class InMemoryCacheService : ICacheService
{
    private readonly ConcurrentDictionary<string, (object Value, DateTime Expiry)> _store = new();

    public Task<T?> GetAsync<T>(string key, CancellationToken cancellationToken = default)
    {
        if (_store.TryGetValue(key, out var entry) && entry.Expiry > DateTime.UtcNow)
            return Task.FromResult((T?)entry.Value);

        _store.TryRemove(key, out _);
        return Task.FromResult(default(T));
    }

    public Task SetAsync<T>(string key, T value, TimeSpan? expiration = null,
        CancellationToken cancellationToken = default)
    {
        var expiry = DateTime.UtcNow.Add(expiration ?? TimeSpan.FromMinutes(5));
        _store[key] = (value!, expiry);
        return Task.CompletedTask;
    }

    public Task RemoveAsync(string key, CancellationToken cancellationToken = default)
    {
        _store.TryRemove(key, out _);
        return Task.CompletedTask;
    }

    public Task RemoveByPrefixAsync(string prefix, CancellationToken cancellationToken = default)
    {
        var keys = _store.Keys.Where(k => k.StartsWith(prefix)).ToList();
        foreach (var key in keys)
            _store.TryRemove(key, out _);
        return Task.CompletedTask;
    }

    public async Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory,
        TimeSpan? expiration = null, CancellationToken cancellationToken = default)
    {
        var cached = await GetAsync<T>(key, cancellationToken);
        if (cached is not null) return cached;
        var value = await factory();
        await SetAsync(key, value, expiration, cancellationToken);
        return value;
    }
}
```

---

## Architecture Tests

Enforce architectural rules with NetArchTest:

```csharp
using NetArchTest.Rules;

namespace ProjectName.ArchitectureTests;

public class ArchitectureTests
{
    private const string DomainNamespace = "ProjectName.Domain";
    private const string ApplicationNamespace = "ProjectName.Application";
    private const string InfrastructureNamespace = "ProjectName.Infrastructure";
    private const string ApiNamespace = "ProjectName.API";

    [Fact]
    public void Domain_ShouldNotDependOn_Application()
    {
        var result = Types.InAssembly(typeof(BaseEntity<>).Assembly)
            .ShouldNot()
            .HaveDependencyOn(ApplicationNamespace)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Domain_ShouldNotDependOn_Infrastructure()
    {
        var result = Types.InAssembly(typeof(BaseEntity<>).Assembly)
            .ShouldNot()
            .HaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Application_ShouldNotDependOn_Infrastructure()
    {
        var result = Types.InAssembly(typeof(Result<>).Assembly)
            .ShouldNot()
            .HaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Handlers_ShouldEndWith_Handler()
    {
        var result = Types.InAssembly(typeof(Result<>).Assembly)
            .That()
            .ImplementInterface(typeof(IRequestHandler<,>))
            .Should()
            .HaveNameEndingWith("Handler")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Validators_ShouldEndWith_Validator()
    {
        var result = Types.InAssembly(typeof(Result<>).Assembly)
            .That()
            .Inherit(typeof(AbstractValidator<>))
            .Should()
            .HaveNameEndingWith("Validator")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }
}
```
