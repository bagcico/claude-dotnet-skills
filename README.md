# .NET 10 Claude Code Skills

Production-ready [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills) for **.NET 10 backend development**. Six skills covering the full stack: architecture scaffolding, database optimization, messaging, caching, logging, and testing.

Built by a software architect with 20+ years of enterprise .NET experience across retail, finance, telecom, and EdTech sectors.

## Why Use These Skills?

**Claude Code generates code.** But without context, it generates *generic* code — boilerplate that doesn't match your architecture, naming conventions, or patterns.

These skills solve that problem. They teach Claude your exact stack, so when you say *"add a CreateOrder command"*, it produces:

- The correct folder structure (`Modules/Order/Commands/CreateOrder/`)
- A `record` command implementing `IRequest<Result<Guid>>`
- A handler with primary constructor DI and direct `DbContext` access
- A FluentValidation validator with proper `.WithMessage()` rules
- A Minimal API endpoint with `Result<T>` → `IResult` mapping
- An EF Core configuration with `snake_case` PostgreSQL naming

All consistent with every other file in your codebase — **zero manual fixup**.

### Token Efficiency: Lazy-Load Architecture

Skills use a **two-tier loading** strategy:

1. **SKILL.md** (~70–95 lines) — always loaded. Contains conventions, rules, and a reference table.
2. **Sub-files** (~100–300 lines each) — loaded on demand. Claude reads only what it needs.

This means a typical interaction loads **~200 lines** instead of **~1,000+**, saving ~60–70% in token cost while keeping full depth available.

## Skills Overview

| # | Skill | Files | What It Covers |
|---|-------|-------|----------------|
| 1 | `dotnet-clean-architecture` | 6 | Module-first CQRS scaffold, MediatR handlers, FluentValidation, Result\<T\>, Minimal API endpoints, pipeline behaviors |
| 2 | `efcore-postgresql` | 5 | Query optimization, N+1 prevention, keyset pagination, bulk operations, GIN indexes, full-text search, JSONB, recursive CTEs, connection pooling |
| 3 | `rabbitmq-masstransit` | 3 | Consumers, publishers, saga state machines, outbox pattern, retry/circuit breaker, batch consumers, idempotency |
| 4 | `redis-caching` | 4 | Cache-aside, write-through, stampede prevention, distributed locking, sliding window rate limiting, pub/sub invalidation |
| 5 | `serilog-logging` | 3 | Structured logging, ELK stack setup, custom enrichers, correlation IDs, PII filtering, index lifecycle management |
| 6 | `dotnet-testing` | 2 | xUnit + NSubstitute + FluentAssertions, handler/validator/endpoint/consumer tests, Testcontainers, architecture tests |

## Tech Stack

These skills are opinionated around a specific, battle-tested stack:

- **.NET 10** with C# 13 features (primary constructors, records, file-scoped namespaces)
- **Clean Architecture** with module-first organization
- **CQRS + MediatR** with 4 pipeline behaviors (Validation, Logging, Transaction, Caching)
- **EF Core** with direct `DbContext` access (no repository abstraction)
- **PostgreSQL** via Npgsql with `snake_case` naming conventions
- **Redis** via StackExchange.Redis for caching and distributed operations
- **RabbitMQ** via MassTransit for messaging and event-driven architecture
- **Serilog** with Elasticsearch sink for structured logging
- **FluentValidation** with `Result<T>` hybrid (no exceptions for validation failures)
- **Minimal APIs** over traditional controllers
- **xUnit + NSubstitute + FluentAssertions** for testing

## Architecture Decisions

These skills encode specific architectural choices:

| Decision | Choice | Why |
|----------|--------|-----|
| Organization | Module-first | Related code stays together; modules are self-contained |
| Data access | Direct DbContext | EF Core IS the repository. Extra abstraction adds complexity without value |
| Endpoints | Minimal APIs | Less ceremony, faster, static methods grouped by module |
| Validation | FluentValidation → Result\<T\> | No exceptions for expected failures; pipeline behavior converts automatically |
| Mapping | Manual extension methods | Explicit, debuggable, zero magic — `ToDto()` per module |
| IDs | Guid (default) | `BaseEntity<TId>` allows flexibility; Guid is the sensible default |
| Error handling | Custom `Result<T>` | Lightweight, no third-party dependency, fits the hybrid validation pattern |

## Installation

### Claude Code (CLI)

Copy the skill folders into your project's `.claude/skills/` directory:

```bash
# Clone or download this repository
git clone https://github.com/bagcico/claude-dotnet-skills.git

# Copy skills to your project
cp -r dotnet-claude-skills/* /path/to/your-project/.claude/skills/
```

### Claude.ai (Web)

Upload the skill files when working with Claude on .NET projects, or reference them in your project knowledge.

## File Structure

```
.claude/skills/
├── dotnet-clean-architecture/
│   ├── SKILL.md               ← Entry point: conventions + reference table
│   ├── scaffold.md            ← Full module scaffold (entity → config → endpoint)
│   ├── commands-queries.md    ← Command/Query + Handler + Validator templates
│   ├── behaviors.md           ← 4 pipeline behaviors (Validation, Logging, Transaction, Caching)
│   ├── result-pattern.md      ← Result<T>, PaginatedList<T>, BaseEntity, DTOs, mappings
│   └── endpoints.md           ← Minimal API endpoint patterns
│
├── efcore-postgresql/
│   ├── SKILL.md               ← Query checklist, index rules, migration commands
│   ├── query-optimization.md  ← N+1, split queries, pagination, projections
│   ├── write-and-raw-sql.md   ← Bulk ops, COPY, LATERAL JOIN, raw aggregation
│   ├── postgresql-features.md ← GIN, tsvector, JSONB, arrays, recursive CTEs
│   └── infrastructure.md      ← Locking, connection pool, RDS tuning, EXPLAIN
│
├── rabbitmq-masstransit/
│   ├── SKILL.md               ← Message types, contracts, basic setup, error levels
│   ├── consumers-publishers.md ← Consumer patterns, idempotency, batching, testing
│   └── sagas-and-advanced.md  ← State machines, outbox, multi-service, graceful shutdown
│
├── redis-caching/
│   ├── SKILL.md               ← Key conventions, invalidation rules, connection config
│   ├── setup.md               ← ICacheService, RedisCacheService, CacheKey builder
│   ├── caching-patterns.md    ← Cache-aside, write-through, stampede, multi-level, warming
│   └── distributed-patterns.md ← Distributed lock, rate limiting, pub/sub, counters
│
├── serilog-logging/
│   ├── SKILL.md               ← Core setup, structured logging rules, log levels
│   ├── enrichers-and-filtering.md ← Custom enrichers, PII filtering, MediatR/MassTransit integration
│   └── elk-setup.md           ← Docker Compose, index templates, ILM policies
│
└── dotnet-testing/
    ├── SKILL.md               ← Test stack, project structure, naming conventions
    └── test-patterns.md       ← Handler, validator, endpoint, consumer, architecture tests
```

## Usage Examples

Once installed, just talk to Claude naturally:

| You say | Claude does |
|---------|------------|
| *"Create a new Payment module with CRUD"* | Scaffolds entity, EF config, DTOs, mappings, commands, queries, handlers, validators, endpoints — all in the correct folder structure |
| *"Add a GetOrderList query with date range filter and pagination"* | Generates query record with filter params, handler with conditional `IQueryable` building, `AsNoTracking()`, `PaginatedList<T>` |
| *"This query is slow, it's doing N+1"* | Analyzes the pattern, suggests `Include()` or projection, adds proper composite index in EF configuration |
| *"Add a saga for order processing"* | Generates MassTransit state machine with states, events, transitions, EF Core persistence, timeout handling |
| *"Add distributed locking to the payment handler"* | Implements Redis-based lock with Lua script release, `IAsyncDisposable` pattern, proper timeout |
| *"Write tests for CreateOrderCommandHandler"* | Generates xUnit tests with FakeDbContext, `TestDataFactory`, success/failure/validation scenarios |

## Customization

These skills are a starting point. Fork and adapt:

- **Different ID type?** Change `Guid` to `long` in `result-pattern.md` and `scaffold.md`
- **Using AutoMapper?** Replace the mapping section in `result-pattern.md`
- **Prefer Controllers?** Swap `endpoints.md` with controller templates
- **Different message broker?** Replace the MassTransit skill with your broker's patterns
- **Using Ardalis.Result?** Update `result-pattern.md` and `behaviors.md`

## Contributing

Found a bug in a template? Have a pattern that should be included? PRs are welcome.

Focus areas for contribution:
- AWS deployment patterns (ECS, Lambda, CDK)
- GraphQL / gRPC endpoint skills
- Hangfire / background job patterns
- SignalR real-time patterns
- Multi-tenancy patterns
<<<<<<< HEAD
=======

>>>>>>> 4222154 (Initialize)
## License

MIT — use freely in personal and commercial projects.
