# Validaciones en Vertical Slice Architecture con .NET Core 8

## Introducción

En aplicaciones que utilizan Vertical Slice Architecture, surge una pregunta fundamental: **¿dónde colocar las validaciones?** ¿En los comandos de los requests (duplicando código) o centralizadas en las entidades del dominio?

Esta guía explora ambos enfoques, sus ventajas y desventajas, y proporciona una implementación práctica usando FluentValidation.

---

## FluentValidation: Instalación y Configuración

### 1. Instalación del Paquete NuGet

```bash
dotnet add package FluentValidation.AspNetCore
```

### 2. Configuración en Program.cs

```csharp
using FluentValidation;

var builder = WebApplication.CreateBuilder(args);

// Registrar FluentValidation
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Si usas MediatR (común en Vertical Slice)
builder.Services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssemblyContaining<Program>();
});

var app = builder.Build();
```

### 3. Pipeline de Validación con MediatR

```csharp
using FluentValidation;
using MediatR;

public class ValidationBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any())
        {
            return await next();
        }

        var context = new ValidationContext<TRequest>(request);

        var validationResults = await Task.WhenAll(
            _validators.Select(v => v.ValidateAsync(context, cancellationToken))
        );

        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Any())
        {
            throw new ValidationException(failures);
        }

        return await next();
    }
}
```

Registrar el behavior en Program.cs:

```csharp
builder.Services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssemblyContaining<Program>();
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
});
```

---

## Enfoque 1: Validación en los Request (Comandos)

### Ejemplo Completo

```csharp
// Features/Products/CreateProduct.cs
public static class CreateProduct
{
    public record Command(
        string Name,
        decimal Price,
        string Description
    ) : IRequest<Result>;

    public class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(x => x.Name)
                .NotEmpty().WithMessage("El nombre es obligatorio")
                .MaximumLength(100).WithMessage("El nombre no puede exceder 100 caracteres");

            RuleFor(x => x.Price)
                .GreaterThan(0).WithMessage("El precio debe ser mayor que cero");

            RuleFor(x => x.Description)
                .MaximumLength(500).WithMessage("La descripción no puede exceder 500 caracteres");
        }
    }

    public class Handler : IRequestHandler<Command, Result>
    {
        private readonly AppDbContext _context;

        public Handler(AppDbContext context)
        {
            _context = context;
        }

        public async Task<Result> Handle(Command request, CancellationToken ct)
        {
            var product = new Product
            {
                Name = request.Name,
                Price = request.Price,
                Description = request.Description
            };

            _context.Products.Add(product);
            await _context.SaveChangesAsync(ct);

            return Result.Success();
        }
    }
}
```

### Ventajas

✅ **Implementación rápida** - Todo el código está en un solo lugar  
✅ **Feedback inmediato** - Las validaciones se ejecutan antes de llegar al dominio  
✅ **Fácil de testear** - Tests unitarios directos  
✅ **Validaciones de contexto naturales** - Consultas a BD y permisos son sencillas  
✅ **Perfecto para Vertical Slice** - Toda la funcionalidad cohesionada

### Desventajas

❌ **Duplicación de código** - Si múltiples comandos usan las mismas validaciones  
❌ **Dominio anémico** - El modelo de dominio puede perder su riqueza  
❌ **Riesgo de inconsistencia** - Diferentes comandos pueden validar de forma distinta

---

## Enfoque 2: Validación Centralizada en el Dominio

### Ejemplo Completo

```csharp
// Domain/Entities/Product.cs
public class Product
{
    public int Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public string Description { get; private set; }

    private Product() { } // Para EF Core

    public static Result<Product> Create(string name, decimal price, string description)
    {
        var validator = new ProductValidator();
        var validationResult = validator.Validate(
            new ProductValidationModel(name, price, description)
        );

        if (!validationResult.IsValid)
        {
            return Result<Product>.Failure(
                validationResult.Errors.Select(e => e.ErrorMessage)
            );
        }

        return Result<Product>.Success(new Product
        {
            Name = name,
            Price = price,
            Description = description
        });
    }

    public Result UpdatePrice(decimal newPrice)
    {
        if (newPrice <= 0)
            return Result.Failure("El precio debe ser mayor que cero");

        Price = newPrice;
        return Result.Success();
    }

    private record ProductValidationModel(string Name, decimal Price, string Description);

    private class ProductValidator : AbstractValidator<ProductValidationModel>
    {
        public ProductValidator()
        {
            RuleFor(x => x.Name)
                .NotEmpty().WithMessage("El nombre es obligatorio")
                .MaximumLength(100);

            RuleFor(x => x.Price)
                .GreaterThan(0).WithMessage("El precio debe ser mayor que cero");

            RuleFor(x => x.Description)
                .MaximumLength(500);
        }
    }
}

// Features/Products/CreateProduct.cs
public class Handler : IRequestHandler<Command, Result>
{
    private readonly AppDbContext _context;

    public Handler(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Result> Handle(Command request, CancellationToken ct)
    {
        var productResult = Product.Create(
            request.Name, 
            request.Price, 
            request.Description
        );

        if (productResult.IsFailure)
            return Result.Failure(productResult.Errors);

        _context.Products.Add(productResult.Value);
        await _context.SaveChangesAsync(ct);

        return Result.Success();
    }
}
```

### Ventajas

✅ **Una sola fuente de verdad** - Las validaciones están centralizadas  
✅ **Garantiza invariantes de negocio** - El dominio se autoprotege  
✅ **Reutilizable** - Todos los casos de uso usan las mismas reglas  
✅ **Dominio rico** - El modelo tiene comportamiento y lógica  
✅ **Protege la integridad** - Imposible crear entidades inválidas

### Desventajas

❌ **Más complejo de implementar** - Requiere más infraestructura  
❌ **Validaciones de contexto difíciles** - Consultas a BD son complicadas en el dominio  
❌ **Feedback más tardío** - Los errores se detectan después  
❌ **Puede ser excesivo** - Para aplicaciones simples puede ser "overkill"

---

## El Problema de los Tests

### Escenario Real

Si las validaciones están **solo en los comandos**, los tests unitarios del dominio no son reales:

```csharp
// Test del dominio - FALSA SEGURIDAD
[Fact]
public void Product_Create_WithValidData_ShouldSucceed()
{
    // Esto pasa porque no hay validación en el dominio
    var product = new Product 
    { 
        Name = "", // ❌ Inválido pero funciona
        Price = -100 // ❌ Inválido pero funciona
    };
    
    // ✅ El test pasa, pero no debería
    Assert.NotNull(product);
}
```

Además, es necesario testear **cada comando** por separado:

```csharp
[Fact]
public async Task CreateProduct_WithInvalidName_ShouldFail() { }

[Fact]
public async Task UpdateProduct_WithInvalidName_ShouldFail() { }

[Fact]
public async Task ImportProduct_WithInvalidName_ShouldFail() { }
// ... repites lo mismo N veces
```

### Solución con Dominio Protegido

```csharp
// Domain/Entities/Product.cs
public class Product
{
    public int Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    
    // Constructor privado - NADIE puede crear un Product inválido
    private Product() { }
    
    public static Result<Product> Create(string name, decimal price, string description = "")
    {
        var errors = new List<string>();
        
        if (string.IsNullOrWhiteSpace(name))
            errors.Add("El nombre es obligatorio");
        else if (name.Length > 100)
            errors.Add("El nombre no puede exceder 100 caracteres");
            
        if (price < 0)
            errors.Add("El precio no puede ser negativo");
            
        if (price == 0 && string.IsNullOrWhiteSpace(description))
            errors.Add("Los productos gratuitos requieren descripción");
            
        if (errors.Any())
            return Result<Product>.Failure(errors);
            
        return Result<Product>.Success(new Product
        {
            Name = name.Trim(),
            Price = price,
            Description = description?.Trim() ?? ""
        });
    }
}
```

---

## Enfoque Híbrido (Recomendado)

El enfoque más práctico combina lo mejor de ambos mundos:

### Reglas de Distribución

| Tipo de Validación | Ubicación | Ejemplo |
|---------------------|-----------|---------|
| **Invariantes** | Dominio | "El precio no puede ser negativo" |
| **Reglas de negocio** | Dominio | "Productos gratis requieren descripción" |
| **Formato básico** | Dominio | "Nombre máximo 100 caracteres" |
| **Unicidad** | Comando | "No existe otro producto con ese nombre" |
| **Permisos** | Comando | "El usuario puede crear productos" |
| **Referencias** | Comando | "La categoría existe" |

### Implementación del Enfoque Híbrido

```csharp
// Domain/Entities/Product.cs
public class Product
{
    public int Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public string Description { get; private set; }
    public int? CategoryId { get; private set; }

    private Product() { }

    // Validaciones de DOMINIO: Invariantes y reglas de negocio críticas
    public static Result<Product> Create(string name, decimal price, string description)
    {
        if (string.IsNullOrWhiteSpace(name))
            return Result<Product>.Failure("El producto debe tener un nombre");

        if (price < 0)
            return Result<Product>.Failure("El precio no puede ser negativo");

        // Regla de negocio específica del dominio
        if (price == 0 && string.IsNullOrWhiteSpace(description))
            return Result<Product>.Failure("Los productos gratuitos deben tener descripción");

        return Result<Product>.Success(new Product
        {
            Name = name,
            Price = price,
            Description = description
        });
    }

    public Result AssignCategory(int categoryId)
    {
        if (categoryId <= 0)
            return Result.Failure("ID de categoría inválido");

        CategoryId = categoryId;
        return Result.Success();
    }
    
    public Result UpdatePrice(decimal newPrice)
    {
        if (newPrice < 0)
            return Result.Failure("El precio no puede ser negativo");
            
        Price = newPrice;
        return Result.Success();
    }
}

// Features/Products/CreateProduct.cs
public static class CreateProduct
{
    public record Command(
        string Name,
        decimal Price,
        string Description,
        int CategoryId
    ) : IRequest<Result<int>>;

    // Validaciones de COMANDO: Formato y contexto de aplicación
    public class Validator : AbstractValidator<Command>
    {
        private readonly AppDbContext _context;

        public Validator(AppDbContext context)
        {
            _context = context;

            // Validaciones de formato (opcionales, el dominio ya valida lo crítico)
            RuleFor(x => x.Name)
                .NotEmpty().WithMessage("El nombre es obligatorio")
                .MaximumLength(100).WithMessage("Máximo 100 caracteres");

            RuleFor(x => x.Price)
                .GreaterThanOrEqualTo(0).WithMessage("El precio debe ser positivo");

            // Validaciones de CONTEXTO (esto no puede estar en el dominio)
            RuleFor(x => x.Name)
                .MustAsync(async (name, ct) => 
                    !await _context.Products.AnyAsync(p => p.Name == name, ct))
                .WithMessage("Ya existe un producto con ese nombre");
                
            RuleFor(x => x.CategoryId)
                .MustAsync(CategoryExists)
                .WithMessage("La categoría no existe");
        }

        private async Task<bool> CategoryExists(int categoryId, CancellationToken ct)
        {
            return await _context.Categories.AnyAsync(c => c.Id == categoryId, ct);
        }
    }

    public class Handler : IRequestHandler<Command, Result<int>>
    {
        private readonly AppDbContext _context;

        public Handler(AppDbContext context)
        {
            _context = context;
        }

        public async Task<Result<int>> Handle(Command request, CancellationToken ct)
        {
            // El dominio valida sus invariantes
            var productResult = Product.Create(
                request.Name,
                request.Price,
                request.Description
            );

            if (productResult.IsFailure)
                return Result<int>.Failure(productResult.Errors);

            var product = productResult.Value;
            var categoryResult = product.AssignCategory(request.CategoryId);
            
            if (categoryResult.IsFailure)
                return Result<int>.Failure(categoryResult.Errors);

            _context.Products.Add(product);
            await _context.SaveChangesAsync(ct);

            return Result<int>.Success(product.Id);
        }
    }
}
```

---

## Tests con Enfoque Híbrido

### Tests del Dominio (Reales y Completos)

```csharp
public class ProductTests
{
    [Fact]
    public void Create_WithValidData_ShouldSucceed()
    {
        var result = Product.Create("iPhone 15", 999.99m, "Latest model");
        
        Assert.True(result.IsSuccess);
        Assert.Equal("iPhone 15", result.Value.Name);
        Assert.Equal(999.99m, result.Value.Price);
    }
    
    [Fact]
    public void Create_WithEmptyName_ShouldFail()
    {
        var result = Product.Create("", 999.99m);
        
        Assert.True(result.IsFailure);
        Assert.Contains("nombre", result.Errors.First().ToLower());
    }
    
    [Fact]
    public void Create_WithNegativePrice_ShouldFail()
    {
        var result = Product.Create("iPhone", -100m);
        
        Assert.True(result.IsFailure);
        Assert.Contains("precio", result.Errors.First().ToLower());
    }
    
    [Fact]
    public void Create_FreeProductWithoutDescription_ShouldFail()
    {
        var result = Product.Create("Gratis", 0m, "");
        
        Assert.True(result.IsFailure);
        Assert.Contains("descripción", result.Errors.First().ToLower());
    }
    
    [Fact]
    public void UpdatePrice_WithNegativeValue_ShouldFail()
    {
        var product = Product.Create("Test", 100m).Value;
        
        var result = product.UpdatePrice(-50m);
        
        Assert.True(result.IsFailure);
        Assert.Equal(100m, product.Price); // No cambió
    }
}
```

### Tests de Comandos (Solo Contexto)

```csharp
public class CreateProductTests
{
    [Fact]
    public async Task Handle_WithDuplicateName_ShouldFail()
    {
        // Arrange
        var context = CreateInMemoryContext();
        var existingProduct = Product.Create("iPhone", 999m, "Existing").Value;
        context.Products.Add(existingProduct);
        await context.SaveChangesAsync();
        
        var validator = new CreateProduct.Validator(context);
        var command = new CreateProduct.Command("iPhone", 799m, "Duplicate", 1);
        
        // Act
        var validationResult = await validator.ValidateAsync(command);
        
        // Assert
        Assert.False(validationResult.IsValid);
        Assert.Contains("ya existe", validationResult.Errors[0].ErrorMessage.ToLower());
    }
    
    [Fact]
    public async Task Handle_WithNonExistentCategory_ShouldFail()
    {
        // Arrange
        var context = CreateInMemoryContext();
        var validator = new CreateProduct.Validator(context);
        var command = new CreateProduct.Command("New Product", 99m, "Description", 999);
        
        // Act
        var validationResult = await validator.ValidateAsync(command);
        
        // Assert
        Assert.False(validationResult.IsValid);
        Assert.Contains("categoría", validationResult.Errors[0].ErrorMessage.ToLower());
    }
}
```

---

## Implementación del Patrón Result

La clase `Result` no viene incluida en .NET, por lo que debe implementarse:

```csharp
// Common/Result.cs
public class Result
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public string Error { get; }
    public IEnumerable<string> Errors { get; }

    protected Result(bool isSuccess, string error, IEnumerable<string> errors = null)
    {
        IsSuccess = isSuccess;
        Error = error;
        Errors = errors ?? (error != null ? new[] { error } : Array.Empty<string>());
    }

    public static Result Success() => new Result(true, null);
    public static Result Failure(string error) => new Result(false, error);
    public static Result Failure(IEnumerable<string> errors) 
        => new Result(false, string.Join(", ", errors), errors);
}

// Common/Result{T}.cs
public class Result<T> : Result
{
    public T Value { get; }

    protected Result(T value, bool isSuccess, string error, IEnumerable<string> errors = null) 
        : base(isSuccess, error, errors)
    {
        Value = value;
    }

    public static Result<T> Success(T value) => new Result<T>(value, true, null);
    public new static Result<T> Failure(string error) => new Result<T>(default, false, error);
    public new static Result<T> Failure(IEnumerable<string> errors) 
        => new Result<T>(default, false, string.Join(", ", errors), errors);
}
```

### Extensiones para ASP.NET Core

```csharp
// Extensions/ResultExtensions.cs
public static class ResultExtensions
{
    public static IActionResult ToActionResult(this Result result)
    {
        if (result.IsSuccess)
            return new OkResult();
        
        return new BadRequestObjectResult(new { errors = result.Errors });
    }
    
    public static IActionResult ToActionResult<T>(this Result<T> result)
    {
        if (result.IsSuccess)
            return new OkObjectResult(result.Value);
        
        return new BadRequestObjectResult(new { errors = result.Errors });
    }
}
```

### Uso en Controllers

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProduct.Command command)
    {
        var result = await _mediator.Send(command);
        return result.ToActionResult();
    }
    
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateProduct.Command command)
    {
        command = command with { Id = id };
        var result = await _mediator.Send(command);
        return result.ToActionResult();
    }
}
```

---

## Librerías Alternativas para Result

### FluentResults

```bash
dotnet add package FluentResults
```

```csharp
using FluentResults;

public static Result<Product> Create(string name, decimal price)
{
    if (string.IsNullOrWhiteSpace(name))
        return Result.Fail<Product>("El nombre es obligatorio");
    
    var product = new Product { Name = name, Price = price };
    return Result.Ok(product);
}
```

### Ardalis.Result

```bash
dotnet add package Ardalis.Result
```

```csharp
using Ardalis.Result;

public static Result<Product> Create(string name, decimal price)
{
    if (string.IsNullOrWhiteSpace(name))
        return Result<Product>.Invalid(new ValidationError 
        { 
            ErrorMessage = "El nombre es obligatorio" 
        });
    
    return Result<Product>.Success(new Product { Name = name, Price = price });
}
```

---

## Comparación de Enfoques

### Validación Solo en Comandos

**Cuándo usar:**
- Aplicaciones simples con pocas operaciones por entidad
- Equipos pequeños con buena comunicación
- Casos donde la velocidad de desarrollo es prioritaria
- Vertical Slice puro sin complejidad de dominio

### Validación Solo en Dominio

**Cuándo usar:**
- Aplicaciones complejas con muchas operaciones por entidad
- Reglas de negocio críticas que no pueden fallar
- Necesidad de consistencia absoluta
- Domain-Driven Design completo

### Enfoque Híbrido (Recomendado)

**Cuándo usar:**
- La mayoría de aplicaciones empresariales
- Balance entre pragmatismo y calidad
- Equipos que valoran tanto la velocidad como la robustez
- Aplicaciones que evolucionarán en complejidad

---

## Conclusiones y Mejores Prácticas

### Principios Generales

1. **Invariantes críticas siempre en el dominio** - Las reglas que si fallan rompen el sistema
2. **Contexto de aplicación en comandos** - Validaciones que requieren infraestructura
3. **Formato básico duplicado está bien** - Es el trade-off de Vertical Slice
4. **Tests del dominio deben ser reales** - Si puedes crear entidades inválidas, algo está mal

### Regla de Oro

> Si una validación puede ejecutarse sin acceso a BD, servicios externos o contexto de aplicación, probablemente debe estar en el dominio.

### Recomendación Final

Para la mayoría de proyectos con Vertical Slice Architecture:

- ✅ **Usa el enfoque híbrido**
- ✅ **Dominio**: Invariantes y reglas de negocio complejas
- ✅ **Comandos**: Formato, contexto y validaciones de aplicación
- ✅ **Tests**: Dominio completo + comandos para contexto
- ✅ **Result Pattern**: Para comunicar éxito/fallo sin excepciones

Este enfoque proporciona el mejor balance entre cohesión de Vertical Slice, protección del dominio y mantenibilidad del código.
