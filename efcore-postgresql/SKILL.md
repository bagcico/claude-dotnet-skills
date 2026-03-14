---
name: efcore-postgresql
description: >
  EF Core + PostgreSQL patterns for .NET 10 including query optimization, indexing strategies, 
  migrations, and performance tuning. Use this skill whenever the user asks about database 
  queries, EF Core configuration, PostgreSQL-specific features, N+1 problems, slow queries, 
  index optimization, migrations, bulk operations, complex SQL with EF, GIN indexes, full-text 
  search, JSON columns, recursive CTEs, connection pooling, or any database performance issue. 
  Also trigger for "query yavaş", "N+1", "index ekle", "migration oluştur", "bulk insert", 
  "raw SQL", "database optimization", "sorgu optimizasyonu", "PostgreSQL", "EF Core", 
  "DbContext", "OUTER APPLY", "tsvector", "JSONB", or any .NET data access concern.
---

# EF Core + PostgreSQL Patterns

Production patterns for EF Core with Npgsql in .NET 10.

## Reference Files — Read on Demand

| When the user asks about... | Read this file |
|----------------------------|---------------|
| Query optimization, N+1, pagination, projections | `./query-optimization.md` |
| Bulk operations, raw SQL, OUTER APPLY | `./write-and-raw-sql.md` |
| GIN indexes, full-text search, JSONB, arrays, CTEs | `./postgresql-features.md` |
| Concurrency, locking, connection pooling, RDS | `./infrastructure.md` |

Read only the relevant file. Do not read all files at once.

## Core Principles

**Direct DbContext** — no repository abstraction. `IApplicationDbContext` injected into handlers.

**PostgreSQL naming**: tables=`snake_case` plural, columns=`snake_case`, indexes=`ix_{table}_{cols}`, unique=`uix_{table}_{cols}`.

**Every entity** gets its own `IEntityTypeConfiguration<T>`. Never use Data Annotations.

## Query Checklist (memorize this)

Before writing any query:
1. Read-only? → `AsNoTracking()`
2. Multiple collection includes? → `AsSplitQuery()`
3. Need all columns? → Project with `Select()` to DTO
4. List query? → Add pagination
5. Conditional filters? → Build `IQueryable` step by step
6. Runs frequently? → Consider caching
7. N+1 risk? → Use `Include()` or projection

## Index Rules

**Create indexes for**: every FK column, frequent WHERE columns, ORDER BY in paginated queries, composite for multi-column filters (most selective first).

**Skip indexes for**: small tables (<1K rows), low cardinality columns (booleans), write-heavy/read-rare tables, columns already covered by composite prefix.

## Migration Commands

```bash
dotnet ef migrations add {Name} \
    --project src/ProjectName.Infrastructure \
    --startup-project src/ProjectName.API \
    --output-dir Persistence/Migrations
```

Naming: `AddAppointmentTable`, `AddMentorIdIndexToAppointments`, `AlterAppointmentAddDurationColumn`

### Zero-Downtime Rules
1. Never drop columns directly — stop reading first, deploy, then drop
2. Add columns as nullable → backfill → add NOT NULL
3. Create indexes with `CONCURRENTLY` via raw SQL
4. Large data migrations — batch in chunks
