---
name: dotnet-clean-architecture
description: >
  Scaffold and generate .NET 10 Clean Architecture code using CQRS + MediatR pattern with 
  module-first organization. Use this skill whenever the user asks to create a new module, 
  feature, command, query, entity, DTO, endpoint, or any architectural component in a .NET 
  project. Also trigger when the user mentions adding a new API endpoint, CRUD operations, 
  domain entity, MediatR handler, FluentValidation validator, or asks about project structure. 
  Trigger for phrases like "yeni modül oluştur", "command ekle", "query yaz", "endpoint ekle", 
  "entity oluştur", "scaffold", "CQRS", "feature ekle", or any .NET code generation request 
  that implies Clean Architecture patterns.
---

# .NET 10 Clean Architecture + CQRS Scaffold

Module-first Clean Architecture with CQRS/MediatR, Minimal APIs, direct DbContext access, 
FluentValidation→Result<T> hybrid, and manual mapping via extension methods.

## Reference Files — Read on Demand

| When the user asks to... | Read this file first |
|--------------------------|---------------------|
| Create a new module or full CRUD scaffold | `./scaffold.md` |
| Add a command or query + handler | `./commands-queries.md` |
| Understand or implement pipeline behaviors | `./behaviors.md` |
| Work with Result\<T\>, DTOs, or mapping | `./result-pattern.md` |
| Add a Minimal API endpoint | `./endpoints.md` |

Read only the relevant file. Do not read all files at once.

## Solution Structure

```
src/
├── ProjectName.Domain/
│   ├── Common/                      # BaseEntity<TId>, AuditableEntity, IDomainEvent
│   └── Modules/{Module}/
│       ├── Entities/
│       ├── Enums/
│       └── Interfaces/
├── ProjectName.Application/
│   ├── Common/
│   │   ├── Behaviors/               # Validation, Logging, Transaction, Caching
│   │   ├── Results/                  # Result<T>, PaginatedList<T>
│   │   ├── Interfaces/              # IApplicationDbContext, ICacheService
│   │   └── Extensions/
│   └── Modules/{Module}/
│       ├── Commands/{CommandName}/   # Command + Handler + Validator
│       ├── Queries/{QueryName}/      # Query + Handler + Validator (optional)
│       ├── DTOs/
│       └── Mappings/                 # {Module}MappingExtensions.cs
├── ProjectName.Infrastructure/
│   └── Persistence/
│       ├── ApplicationDbContext.cs
│       ├── Configurations/{Module}/  # IEntityTypeConfiguration per entity
│       └── Migrations/
└── ProjectName.API/
    ├── Endpoints/{Module}/           # {Module}Endpoints.cs (Minimal API)
    ├── Middleware/
    └── Program.cs
```

## Key Conventions

### Naming
- Commands: `{Verb}{Noun}Command` → `CreateAppointmentCommand`
- Queries: `Get{Noun}Query` / `Get{Noun}ListQuery`
- Handlers: `{CommandOrQuery}Handler`
- Validators: `{CommandOrQuery}Validator`
- DTOs: `{Noun}Dto`
- Mappings: `{Module}MappingExtensions` (static class with `ToDto()` extensions)
- Endpoints: `{Module}Endpoints` (static class, one per module)
- EF Configs: `{Entity}Configuration`

### Architecture Rules
1. **Module-first**: Both Domain and Application group by business module, not technical concern.
2. **No repository**: Handlers inject `IApplicationDbContext` directly. EF Core IS the repository.
3. **Minimal APIs**: Static methods grouped by module, registered via `Map{Module}Endpoints()`.
4. **Result\<T\> hybrid**: FluentValidation runs in `ValidationBehavior`, returns `Result<T>.ValidationFailure()` — no exceptions for validation.
5. **Manual mapping**: Extension methods `ToDto()` / `ToDtoList()` per module.
6. **Guid IDs**: Default Id type is `Guid` via `BaseEntity<TId>`.

### Commands vs Queries
- **Command** = write → returns `Result<T>` or `Result`
- **Query** = read-only → handler uses `AsNoTracking()`, returns `Result<T>`
- Never mix reads and writes in the same handler.

### CancellationToken
Always pass through: handler → EF queries → `SaveChangesAsync`.

### File Generation Order (new feature)
1. Domain entity → 2. EF Configuration → 3. DTOs → 4. Mapping extensions →
5. Command/Query → 6. Handler → 7. Validator → 8. Endpoint
