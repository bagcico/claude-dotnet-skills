# Module Scaffold

Complete file list and templates for scaffolding a new module.

## Pre-Scaffold Checklist

Ask the user (if not clear from context):
1. **Module name** — e.g., "Appointment", "Payment"
2. **Main entity fields** — properties of the primary entity
3. **Relationships** — FK references to existing entities?
4. **Status enum?** — does the entity have a lifecycle?
5. **Query filters** — what filters for the list endpoint?

## Files to Generate

For a module `{Module}` with main entity `{Entity}`:

```
 1. Domain/Modules/{Module}/Entities/{Entity}.cs
 2. Domain/Modules/{Module}/Enums/{Entity}Status.cs          (if applicable)
 3. Application/Modules/{Module}/DTOs/{Entity}Dto.cs
 4. Application/Modules/{Module}/Mappings/{Module}MappingExtensions.cs
 5. Application/Modules/{Module}/Commands/Create{Entity}/Create{Entity}Command.cs
 6. Application/Modules/{Module}/Commands/Create{Entity}/Create{Entity}CommandHandler.cs
 7. Application/Modules/{Module}/Commands/Create{Entity}/Create{Entity}CommandValidator.cs
 8. Application/Modules/{Module}/Commands/Update{Entity}/ (Command + Handler + Validator)
 9. Application/Modules/{Module}/Commands/Delete{Entity}/ (Command + Handler)
10. Application/Modules/{Module}/Queries/Get{Entity}/ (Query + Handler)
11. Application/Modules/{Module}/Queries/Get{Entity}List/ (Query + Handler + Validator)
12. Infrastructure/Persistence/Configurations/{Module}/{Entity}Configuration.cs
13. API/Endpoints/{Module}/{Module}Endpoints.cs
```

Also update:
- `IApplicationDbContext.cs` — add `DbSet<{Entity}>`
- `ApplicationDbContext.cs` — add `DbSet<{Entity}>`
- `Program.cs` — add `app.Map{Module}Endpoints();`

## Domain Entity Template

```csharp
namespace ProjectName.Domain.Modules.Appointment.Entities;

public class Appointment : AuditableEntity<Guid>
{
    public Guid MentorId { get; set; }
    public Guid StudentId { get; set; }
    public DateTime ScheduledAt { get; set; }
    public int DurationMinutes { get; set; }
    public AppointmentStatus Status { get; set; }
    public string? Notes { get; set; }

    // Navigation properties
    public Mentor Mentor { get; set; } = null!;
    public Student Student { get; set; } = null!;
}
```

## EF Configuration Template

```csharp
public class AppointmentConfiguration : IEntityTypeConfiguration<Appointment>
{
    public void Configure(EntityTypeBuilder<Appointment> builder)
    {
        builder.ToTable("appointments");
        builder.HasKey(a => a.Id);

        builder.Property(a => a.Id).HasColumnName("id").ValueGeneratedNever();
        builder.Property(a => a.MentorId).HasColumnName("mentor_id").IsRequired();
        builder.Property(a => a.StudentId).HasColumnName("student_id").IsRequired();
        builder.Property(a => a.ScheduledAt).HasColumnName("scheduled_at").IsRequired();
        builder.Property(a => a.DurationMinutes).HasColumnName("duration_minutes").IsRequired();
        builder.Property(a => a.Status).HasColumnName("status").HasConversion<string>().HasMaxLength(20).IsRequired();
        builder.Property(a => a.Notes).HasColumnName("notes").HasMaxLength(1000);
        builder.Property(a => a.CreatedAt).HasColumnName("created_at").IsRequired();
        builder.Property(a => a.CreatedBy).HasColumnName("created_by").HasMaxLength(100);
        builder.Property(a => a.UpdatedAt).HasColumnName("updated_at");
        builder.Property(a => a.UpdatedBy).HasColumnName("updated_by").HasMaxLength(100);

        // Indexes
        builder.HasIndex(a => a.MentorId).HasDatabaseName("ix_appointments_mentor_id");
        builder.HasIndex(a => a.StudentId).HasDatabaseName("ix_appointments_student_id");
        builder.HasIndex(a => new { a.MentorId, a.ScheduledAt })
            .HasDatabaseName("ix_appointments_mentor_scheduled");

        // Relationships
        builder.HasOne(a => a.Mentor).WithMany(m => m.Appointments)
            .HasForeignKey(a => a.MentorId).OnDelete(DeleteBehavior.Restrict);
        builder.HasOne(a => a.Student).WithMany(s => s.Appointments)
            .HasForeignKey(a => a.StudentId).OnDelete(DeleteBehavior.Restrict);
    }
}
```

### PostgreSQL Naming Convention
- Tables: `snake_case` plural → `appointments`
- Columns: `snake_case` → `mentor_id`
- Indexes: `ix_{table}_{columns}` → `ix_appointments_mentor_id`
- Unique: `uix_{table}_{columns}`
- Enums: `HasConversion<string>()` preferred for readability

## IApplicationDbContext Update

```csharp
public interface IApplicationDbContext
{
    DbSet<Appointment> Appointments { get; }
    // ... other DbSets
    DatabaseFacade Database { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

## Post-Scaffold: Create Migration

```bash
dotnet ef migrations add Add{Module}Tables \
    --project src/ProjectName.Infrastructure \
    --startup-project src/ProjectName.API \
    --output-dir Persistence/Migrations
```
