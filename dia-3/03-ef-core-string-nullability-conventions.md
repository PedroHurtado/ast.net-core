# Convenciones de Nullability para Strings en Entity Framework Core 8.0

## Introducción

Esta guía explica cómo controlar si las propiedades `string` son nullable o no nullable en Entity Framework Core 8.0 mediante convenciones personalizadas.

---

## 1. Habilitar Nullable Reference Types (Recomendado)

Primero, asegúrate de tener habilitado en tu archivo `.csproj`:

```xml
<PropertyGroup>
    <Nullable>enable</Nullable>
</PropertyGroup>
```

Con esto habilitado, EF Core respetará automáticamente las anotaciones de nullability en tus entidades:

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = null!; // NOT NULL en BD
    public string? Description { get; set; }   // NULL en BD
}
```

---

## 2. Convención Global con ConfigureConventions

Si quieres que **TODOS** los strings sean NOT NULL por defecto:

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    // Todos los strings serán NOT NULL por defecto
    configurationBuilder.Properties<string>()
        .HaveMaxLength(500)
        .AreRequired(); // ← Clave aquí
}
```

### Ventajas
- Simple y directo
- Se aplica globalmente
- Fácil de mantener

---

## 3. Convención Personalizada Avanzada

Para tener control total basado en nullable annotations:

```csharp
public class StringNullabilityConvention : IModelFinalizingConvention
{
    public void ProcessModelFinalizing(
        IConventionModelBuilder modelBuilder,
        IConventionContext<IConventionModelBuilder> context)
    {
        foreach (var entityType in modelBuilder.Metadata.GetEntityTypes())
        {
            foreach (var property in entityType.GetProperties())
            {
                if (property.ClrType == typeof(string))
                {
                    // Si no tiene configuración explícita de nullability
                    if (property.IsNullable == true && !property.IsPrimaryKey())
                    {
                        // Opción A: Hacer NOT NULL por defecto
                        property.Builder.IsRequired();
                        
                        // Opción B: Respetar nullable annotations del código
                        // (esto ya lo hace EF Core por defecto si tienes <Nullable>enable</Nullable>)
                    }
                }
            }
        }
    }
}
```

### Registrar la convención:

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.Conventions.Add(_ => new StringNullabilityConvention());
}
```

---

## 4. Configuración Híbrida (Recomendado)

Esta es la **mejor práctica** para la mayoría de proyectos:

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    // Por defecto, strings NOT NULL con longitud máxima
    configurationBuilder.Properties<string>()
        .HaveMaxLength(500)
        .AreRequired();
}
```

En tus entidades, marcas explícitamente los campos nullable:

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;  // NOT NULL (por convención)
    public string? Description { get; set; }          // NULL (explícito con ?)
    public string? Notes { get; set; }                // NULL (explícito con ?)
}
```

### Ventajas de este enfoque:
- ✅ Seguridad en tiempo de compilación
- ✅ Base de datos consistente
- ✅ Código claro e intencional
- ✅ Menos configuración manual

---

## 5. Sin Nullable Reference Types

Si **NO** usas Nullable Reference Types, puedes crear convenciones basadas en nombres o atributos:

```csharp
public class StringNullabilityByNameConvention : IModelFinalizingConvention
{
    public void ProcessModelFinalizing(
        IConventionModelBuilder modelBuilder,
        IConventionContext<IConventionModelBuilder> context)
    {
        foreach (var entityType in modelBuilder.Metadata.GetEntityTypes())
        {
            foreach (var property in entityType.GetProperties())
            {
                if (property.ClrType == typeof(string))
                {
                    // NOT NULL por defecto, excepto si termina en "Optional"
                    var isOptional = property.Name.EndsWith("Optional") || 
                                   property.Name.StartsWith("Optional");
                    
                    property.Builder.IsRequired(!isOptional);
                }
            }
        }
    }
}
```

### Ejemplo de uso:

```csharp
public class Product
{
    public string Name { get; set; }          // NOT NULL
    public string DescriptionOptional { get; set; }  // NULL (por convención de nombre)
}
```

---

## 6. Ejemplo Completo de DbContext

```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer("your-connection-string");
    }

    protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
    {
        // Convención: Todos los strings NOT NULL por defecto
        configurationBuilder.Properties<string>()
            .AreRequired()
            .HaveMaxLength(500);
        
        // Convención: Decimales con precisión estándar
        configurationBuilder.Properties<decimal>()
            .HavePrecision(18, 2);
        
        // Convención: DateTime como datetime2
        configurationBuilder.Properties<DateTime>()
            .HaveColumnType("datetime2");
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Aquí puedes agregar configuraciones específicas que sobrescriban las convenciones
        modelBuilder.Entity<Product>(entity =>
        {
            entity.Property(p => p.Description)
                .IsRequired(false) // Sobrescribe la convención para esta propiedad
                .HasMaxLength(2000);
        });
    }
}
```

---

## 7. Comparación de Enfoques

| Enfoque | Ventajas | Desventajas |
|---------|----------|-------------|
| **Nullable Reference Types** | Type-safe, warnings en compilación | Requiere .NET 6+ |
| **ConfigureConventions** | Simple, centralizado | Menos flexible |
| **IModelFinalizingConvention** | Máxima flexibilidad | Más código |
| **Convenciones por nombre** | No requiere NRT | Menos explícito, propenso a errores |

---

## Recomendación Final

Para proyectos modernos con EF Core 8.0:

1. **Habilita** `<Nullable>enable</Nullable>` en tu proyecto
2. **Configura** una convención que haga strings NOT NULL por defecto:

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.Properties<string>()
        .AreRequired()
        .HaveMaxLength(500);
}
```

3. **Marca explícitamente** con `?` las propiedades que sí quieres nullable:

```csharp
public class MyEntity
{
    public string RequiredField { get; set; } = string.Empty;
    public string? OptionalField { get; set; }
}
```

Este enfoque te proporciona:
- ✅ Código limpio y mantenible
- ✅ Seguridad de tipos
- ✅ Base de datos consistente
- ✅ Menos errores en runtime

---

## Recursos Adicionales

- [Entity Framework Core Documentation](https://docs.microsoft.com/ef/core/)
- [Model Configuration in EF Core](https://docs.microsoft.com/ef/core/modeling/)
- [Nullable Reference Types](https://docs.microsoft.com/dotnet/csharp/nullable-references)

---


