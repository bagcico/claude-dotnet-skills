---
name: dotnet-testing
description: >
  Testing patterns for .NET 10 with xUnit, NSubstitute, FluentAssertions, and Testcontainers. 
  Use this skill whenever the user asks about unit testing, integration testing, writing tests, 
  test patterns, mocking, test fixtures, or testing any component (MediatR handlers, validators, 
  endpoints, consumers, EF Core queries). Also trigger for "test yaz", "unit test", "integration 
  test", "xUnit", "NSubstitute", "Moq", "FluentAssertions", "Testcontainers", "test coverage", 
  "TDD", "test pattern", "mock", "fake", "stub", "handler test", "validator test", "endpoint 
  test", "consumer test", or any .NET testing concern.
---

# .NET 10 Testing Patterns

Production testing patterns using xUnit + NSubstitute + FluentAssertions. Covers unit testing 
for handlers, validators, and services, plus integration testing with Testcontainers.

Before writing test code, read the relevant reference:
- For all test patterns and examples: `references/test-patterns.md`

## Testing Stack

| Library | Purpose |
|---------|---------|
| **xUnit** | Test framework (parallel by default, constructor DI) |
| **NSubstitute** | Mocking (cleaner syntax than Moq, no `.Object` needed) |
| **FluentAssertions** | Readable assertions (`.Should().Be()`, `.Should().ContainSingle()`) |
| **Testcontainers** | Real PostgreSQL + Redis in Docker for integration tests |
| **Microsoft.AspNetCore.Mvc.Testing** | `WebApplicationFactory` for API integration tests |
| **MassTransit.Testing** | In-memory harness for consumer tests |
| **Bogus** | Fake data generation for test factories |

## Test Project Structure

```
tests/
├── ProjectName.UnitTests/
│   ├── Modules/
│   │   └── Appointment/
│   │       ├── Commands/
│   │       │   └── CreateAppointmentCommandHandlerTests.cs
│   │       ├── Queries/
│   │       │   └── GetAppointmentQueryHandlerTests.cs
│   │       └── Validators/
│   │           └── CreateAppointmentCommandValidatorTests.cs
│   ├── Common/
│   │   └── Behaviors/
│   │       └── ValidationBehaviorTests.cs
│   └── Fakes/
│       ├── FakeDbContext.cs
│       └── TestDataFactory.cs
│
├── ProjectName.IntegrationTests/
│   ├── Fixtures/
│   │   ├── IntegrationTestFixture.cs
│   │   └── TestWebApplicationFactory.cs
│   ├── Modules/
│   │   └── Appointment/
│   │       └── AppointmentEndpointTests.cs
│   └── appsettings.Testing.json
│
└── ProjectName.ArchitectureTests/
    └── ArchitectureTests.cs
```

## Naming Convention

Test classes: `{ClassUnderTest}Tests`
Test methods: `{Method}_{Scenario}_{ExpectedResult}`

```csharp
public class CreateAppointmentCommandHandlerTests
{
    [Fact]
    public async Task Handle_ValidCommand_ReturnsSuccessWithNewId()
    
    [Fact]
    public async Task Handle_MentorNotFound_ReturnsFailure()
    
    [Fact]
    public async Task Handle_ConflictingAppointment_ReturnsFailure()
    
    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(180)]
    public async Task Handle_InvalidDuration_ReturnsValidationFailure(int duration)
}
```

## Test Organization Rules

1. **Mirror the source structure** — test files parallel the production code layout
2. **One test class per production class** — keep focused
3. **Arrange-Act-Assert** — clearly separate the three phases
4. **No logic in tests** — no if/else, no loops, no try/catch in tests
5. **Test behavior, not implementation** — don't assert internal method calls
6. **Use factories** — `TestDataFactory` for creating test entities consistently

## Quick Reference: What to Test

| Component | Test Type | What to Assert |
|-----------|-----------|---------------|
| Command Handler | Unit | Correct entity created/updated, correct Result returned |
| Query Handler | Unit | Correct data returned, proper filtering applied |
| Validator | Unit | Valid input passes, each rule rejects invalid input |
| Mapping Extension | Unit | All properties mapped correctly |
| Pipeline Behavior | Unit | Correct flow for success/failure paths |
| Endpoint | Integration | HTTP status codes, response body shape |
| Consumer | Unit + Integration | Message processed, side effects triggered |
| Full Flow | Integration | End-to-end with real DB and message bus |
