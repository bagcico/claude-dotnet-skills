# PostgreSQL-Specific Features

## GIN Indexes (Arrays & JSONB)

```csharp
// Array column
builder.Property(m => m.Tags).HasColumnName("tags").HasColumnType("text[]");
builder.HasIndex(m => m.Tags).HasDatabaseName("ix_mentors_tags_gin").HasMethod("gin");

// JSONB column — use raw SQL in migration
migrationBuilder.Sql("CREATE INDEX ix_mentors_metadata_gin ON mentors USING gin (metadata jsonb_path_ops);");
```

### Array Queries
```csharp
.Where(m => m.Tags.Contains("math"))                           // ANY
.Where(m => m.Tags.Any(t => searchTags.Contains(t)))           // Overlap
.Where(m => requestedTags.All(t => m.Tags.Contains(t)))        // Contains all
```

## Full-Text Search (tsvector)

### Generated Column
```csharp
builder.Property(p => p.SearchVector).HasColumnName("search_vector").HasColumnType("tsvector")
    .HasComputedColumnSql("to_tsvector('turkish', coalesce(name, '') || ' ' || coalesce(description, ''))", stored: true);
builder.HasIndex(p => p.SearchVector).HasDatabaseName("ix_products_search_vector_gin").HasMethod("gin");
```

### Querying
```csharp
var results = await dbContext.Products.AsNoTracking()
    .Where(p => p.SearchVector.Matches(EF.Functions.PlainToTsQuery("turkish", searchTerm)))
    .OrderByDescending(p => p.SearchVector.Rank(EF.Functions.PlainToTsQuery("turkish", searchTerm)))
    .Take(20).ToListAsync(cancellationToken);

// Prefix matching (autocomplete)
var tsQuery = EF.Functions.ToTsQuery("turkish", searchTerm.Trim() + ":*");
```

## JSONB Columns

```csharp
builder.Property(p => p.Settings).HasColumnName("settings").HasColumnType("jsonb")
    .HasConversion(
        v => JsonSerializer.Serialize(v, JsonSerializerOptions.Default),
        v => JsonSerializer.Deserialize<MentorSettings>(v, JsonSerializerOptions.Default)!);

// Query via EF
.Where(p => EF.Functions.JsonContains(p.Settings, JsonSerializer.Serialize(new { AcceptsNewStudents = true })))

// Complex JSONB — use raw SQL
"WHERE (mp.settings->>'AcceptsNewStudents')::boolean = true AND (mp.settings->>'MaxStudents')::int > @threshold"
```

## Recursive CTEs

### Subscription Chain
```csharp
var chain = await dbContext.Database.SqlQueryRaw<SubscriptionChainDto>(@"
    WITH RECURSIVE chain AS (
        SELECT id, parent_id, status, started_at, 0 AS depth FROM subscriptions WHERE id = @rootId
        UNION ALL
        SELECT s.id, s.parent_id, s.status, s.started_at, c.depth + 1
        FROM subscriptions s INNER JOIN chain c ON s.id = c.parent_id
    )
    SELECT id AS ""Id"", parent_id AS ""ParentId"", status AS ""Status"", depth AS ""Depth""
    FROM chain ORDER BY depth",
    new NpgsqlParameter("rootId", subscriptionId)).ToListAsync(cancellationToken);
```

### Category Tree
```csharp
// Same CTE pattern with ARRAY[id] for path tracking, ordered by path
```

## Enum Mapping

**String (preferred)**: `builder.Property(a => a.Status).HasConversion<string>().HasMaxLength(30);`

**Native PostgreSQL**: `modelBuilder.HasPostgresEnum<AppointmentStatus>("appointment_status");`

String preferred because: simpler migrations, human-readable, no ALTER TYPE for new values.

## Generated Columns
```csharp
builder.Property(m => m.FullName).HasColumnName("full_name")
    .HasComputedColumnSql("first_name || ' ' || last_name", stored: true);
```

## Partitioning (10M+ rows)
Range partition by date — use raw SQL in migration. EF Core works transparently.

## pg_repack (zero-downtime maintenance)
```bash
pg_repack -h localhost -d appdb -t appointments --no-superuser-check
```
Use when table bloat > 20-30%. Schedule monthly. Prefer over VACUUM FULL (no exclusive locks).
