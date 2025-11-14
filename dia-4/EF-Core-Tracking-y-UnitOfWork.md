# Manejo de Estados y Unit of Work en Entity Framework Core 8

## 📋 Índice
1. [Introducción](#introducción)
2. [Fundamentos del Change Tracking](#fundamentos-del-change-tracking)
3. [Operaciones CRUD con Estados](#operaciones-crud-con-estados)
4. [Optimización con Tracking](#optimización-con-tracking)
5. [Patrón Unit of Work](#patrón-unit-of-work)
6. [Implementación Final](#implementación-final)
7. [Conclusiones](#conclusiones)

---

## Introducción

En Entity Framework Core, el **Change Tracking** es el mecanismo que permite al contexto detectar automáticamente los cambios en las entidades. Sin embargo, cuando trabajamos con entidades que vienen de fuera del contexto (DTOs, APIs, etc.), necesitamos manejar manualmente los estados de las entidades.

Este documento explora las mejores prácticas para:
- Manipular estados de entidades (`Added`, `Modified`, `Deleted`)
- Optimizar operaciones UPDATE aprovechando el tracking automático
- Implementar el patrón Unit of Work de forma elegante

---

## Fundamentos del Change Tracking

### Estados de una Entidad en EF Core

```
┌─────────────────────────────────────────────┐
│          Estados de EntityState             │
├─────────────────────────────────────────────┤
│ • Detached  → No está siendo rastreada      │
│ • Unchanged → Está rastreada, sin cambios   │
│ • Added     → Nueva, será insertada         │
│ • Modified  → Modificada, será actualizada  │
│ • Deleted   → Marcada para eliminación      │
└─────────────────────────────────────────────┘
```

### Dos Escenarios Fundamentales

#### Escenario 1: Entidad Tracked (leída con tracking)
```csharp
// La entidad está siendo rastreada por el contexto
var producto = await context.Productos.FindAsync(1);
// Estado: Unchanged

producto.Nombre = "Nuevo Nombre";
// Estado: Modified (automático)

await context.SaveChangesAsync();
// SQL: UPDATE Productos SET Nombre = 'Nuevo Nombre' WHERE Id = 1
// ✅ Solo actualiza la columna que cambió
```

#### Escenario 2: Entidad Detached (viene de fuera)
```csharp
// La entidad NO está siendo rastreada
var producto = new Producto 
{ 
    Id = 1, 
    Nombre = "Nuevo Nombre",
    Precio = 100
};

context.Entry(producto).State = EntityState.Modified;
await context.SaveChangesAsync();
// SQL: UPDATE Productos SET Nombre='...', Precio='...', 
//      FechaCreacion='...', etc WHERE Id = 1
// ❌ Actualiza TODAS las columnas
```

---

## Operaciones CRUD con Estados

### 1️⃣ ADD - Agregar Nueva Entidad

```csharp
var nuevaEntidad = new Producto 
{ 
    Nombre = "Producto Nuevo",
    Precio = 50
    // No establecer Id si es auto-generado
};

// Cambiar el estado a Added
context.Entry(nuevaEntidad).State = EntityState.Added;

await context.SaveChangesAsync();
```

**📌 Nota:** `Entry()` hace attach automático, no necesitas llamar a `Attach()` explícitamente.

### 2️⃣ UPDATE - Actualizar Entidad

#### Opción A: Con Tracking (Recomendado)
```csharp
// 1. Leer la entidad con tracking
var producto = await context.Productos.FindAsync(1);

// 2. Modificar las propiedades necesarias
producto.Nombre = "Nombre Modificado";
producto.Precio = 150;

// 3. Solo SaveChanges - EF detecta los cambios automáticamente
await context.SaveChangesAsync();

// ✅ SQL genera UPDATE solo para Nombre y Precio
```

#### Opción B: Sin Tracking (Actualiza TODO)
```csharp
var producto = new Producto 
{ 
    Id = 1,
    Nombre = "Modificado",
    Precio = 100
    // Todas las propiedades...
};

context.Entry(producto).State = EntityState.Modified;
await context.SaveChangesAsync();

// ❌ SQL actualiza TODAS las columnas
```

#### Opción C: Actualización Parcial
```csharp
var producto = new Producto { Id = 1, Nombre = "Solo esto" };

context.Attach(producto);
context.Entry(producto).Property(p => p.Nombre).IsModified = true;

await context.SaveChangesAsync();
// ✅ SQL actualiza solo Nombre
```

### 3️⃣ REMOVE - Eliminar Entidad

```csharp
// Solo necesitas el Id
var producto = new Producto { Id = 1 };

// Cambiar el estado a Deleted
context.Entry(producto).State = EntityState.Deleted;

await context.SaveChangesAsync();
```

**💡 Ventaja:** No necesitas cargar toda la entidad desde la base de datos para eliminarla.

---

## Optimización con Tracking

### La Regla de Oro

> **Para UPDATE**: Si vas a leer la entidad antes de modificarla, aprovecha el tracking automático.
> Solo usa `SaveChanges()`, EF Core detectará qué cambió.

### Comparación de Eficiencia

| Enfoque | SQL Generado | Eficiencia |
|---------|-------------|------------|
| **Leer + Modificar + Save** | UPDATE solo columnas modificadas | ✅ Óptimo |
| **Entry + State = Modified** | UPDATE todas las columnas | ⚠️ Subóptimo |
| **Property.IsModified** | UPDATE columnas específicas | ✅ Bueno |

### Ejemplo Práctico

```csharp
public class ProductoService
{
    private readonly DbContext _context;

    // ❌ MENOS EFICIENTE
    public async Task ActualizarIneficiente(ProductoDto dto)
    {
        var producto = new Producto 
        { 
            Id = dto.Id,
            Nombre = dto.Nombre,
            Precio = dto.Precio,
            Descripcion = dto.Descripcion,
            Stock = dto.Stock
        };
        
        _context.Entry(producto).State = EntityState.Modified;
        await _context.SaveChangesAsync();
        // Actualiza TODO, incluso lo que no cambió
    }

    // ✅ MÁS EFICIENTE
    public async Task ActualizarEficiente(ProductoDto dto)
    {
        var producto = await _context.Productos.FindAsync(dto.Id);
        
        if (producto == null) 
            throw new NotFoundException();
        
        producto.Nombre = dto.Nombre;
        producto.Precio = dto.Precio;
        // Solo asignas lo que cambió
        
        await _context.SaveChangesAsync();
        // EF Core hace diff automático
    }
}
```

---

## Patrón Unit of Work

### ¿Por qué Unit of Work?

El patrón Unit of Work encapsula las operaciones de persistencia y asegura que todas las operaciones se realicen en una única transacción. En EF Core, el `DbContext` ya implementa este patrón, pero podemos crear una abstracción para:

- Desacoplar nuestro código del `DbContext` concreto
- Facilitar testing con mocks
- Exponer solo los métodos necesarios

### Implementación Elegante

#### 1. Definir la Interface

```csharp
public interface IUnitOfWork
{
    EntityEntry<TEntity> Entry<TEntity>(TEntity entity) where TEntity : class;
    
    int SaveChanges();
    
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

#### 2. Implementar en el DbContext

```csharp
public class MiDbContext : DbContext, IUnitOfWork
{
    public MiDbContext(DbContextOptions<MiDbContext> options) 
        : base(options)
    {
    }

    // DbSets
    public DbSet<Producto> Productos { get; set; }
    public DbSet<Cliente> Clientes { get; set; }
    public DbSet<Pedido> Pedidos { get; set; }

    // ✨ No necesitas implementar nada más
    // Los métodos Entry, SaveChanges y SaveChangesAsync
    // ya existen en DbContext con las firmas correctas
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(MiDbContext).Assembly);
    }
}
```

**🎯 Clave:** `DbContext` ya tiene los métodos que declaramos en `IUnitOfWork`. Al heredar la interface, el contrato se cumple automáticamente sin necesidad de implementación explícita.

#### 3. Repositorio Genérico con Unit of Work

```csharp
public class RepositorioGenerico<T> where T : class
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly DbSet<T> _dbSet;

    public RepositorioGenerico(IUnitOfWork unitOfWork, DbContext context)
    {
        _unitOfWork = unitOfWork;
        _dbSet = context.Set<T>();
    }

    // CREATE
    public async Task AgregarAsync(T entidad)
    {
        _unitOfWork.Entry(entidad).State = EntityState.Added;
        await _unitOfWork.SaveChangesAsync();
    }

    // READ
    public async Task<T?> ObtenerPorIdAsync(object id)
    {
        return await _dbSet.FindAsync(id);
    }

    public async Task<List<T>> ObtenerTodosAsync()
    {
        return await _dbSet.ToListAsync();
    }

    // UPDATE
    public async Task ActualizarAsync(T entidad)
    {
        // La entidad ya fue leída con tracking
        // Solo guardamos los cambios detectados automáticamente
        await _unitOfWork.SaveChangesAsync();
    }

    // DELETE
    public async Task EliminarAsync(T entidad)
    {
        _unitOfWork.Entry(entidad).State = EntityState.Deleted;
        await _unitOfWork.SaveChangesAsync();
    }
}
```

#### 4. Configuración de Dependency Injection

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Registrar DbContext
builder.Services.AddDbContext<MiDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")
    )
);

// Registrar como IUnitOfWork
builder.Services.AddScoped<IUnitOfWork>(provider => 
    provider.GetRequiredService<MiDbContext>()
);

// Registrar repositorios
builder.Services.AddScoped(typeof(RepositorioGenerico<>));
```

---

## Implementación Final

### Estructura de Carpetas Sugerida

```
📁 Proyecto
├── 📁 Data
│   ├── MiDbContext.cs
│   └── Configurations/
│       ├── ProductoConfiguration.cs
│       └── ClienteConfiguration.cs
├── 📁 Repositories
│   ├── IUnitOfWork.cs
│   └── RepositorioGenerico.cs
├── 📁 Services
│   └── ProductoService.cs
└── 📁 Models
    ├── Producto.cs
    └── Cliente.cs
```

### Ejemplo de Servicio Completo

```csharp
public class ProductoService
{
    private readonly RepositorioGenerico<Producto> _repositorio;
    private readonly IUnitOfWork _unitOfWork;

    public ProductoService(
        RepositorioGenerico<Producto> repositorio,
        IUnitOfWork unitOfWork)
    {
        _repositorio = repositorio;
        _unitOfWork = unitOfWork;
    }

    public async Task<Producto> CrearProductoAsync(CrearProductoDto dto)
    {
        var producto = new Producto
        {
            Nombre = dto.Nombre,
            Precio = dto.Precio,
            Stock = dto.Stock
        };

        await _repositorio.AgregarAsync(producto);
        return producto;
    }

    public async Task ActualizarProductoAsync(int id, ActualizarProductoDto dto)
    {
        var producto = await _repositorio.ObtenerPorIdAsync(id);
        
        if (producto == null)
            throw new NotFoundException($"Producto con Id {id} no encontrado");

        // Modificar propiedades (tracking automático)
        producto.Nombre = dto.Nombre;
        producto.Precio = dto.Precio;
        producto.Stock = dto.Stock;

        // SaveChanges detecta qué cambió
        await _repositorio.ActualizarAsync(producto);
    }

    public async Task EliminarProductoAsync(int id)
    {
        var producto = new Producto { Id = id };
        await _repositorio.EliminarAsync(producto);
    }

    // Operación con múltiples cambios en una transacción
    public async Task ActualizarInventarioAsync(
        List<ActualizacionInventarioDto> actualizaciones)
    {
        foreach (var act in actualizaciones)
        {
            var producto = await _repositorio.ObtenerPorIdAsync(act.ProductoId);
            if (producto != null)
            {
                producto.Stock += act.Cantidad;
            }
        }

        // Un solo SaveChanges para todas las operaciones
        await _unitOfWork.SaveChangesAsync();
    }
}
```

---

## Transacciones y SaveChanges

### El Desafío de Múltiples Operaciones

Cuando trabajamos con casos de uso que requieren modificar múltiples entidades, surge una pregunta fundamental: **¿Cómo garantizamos que todas las operaciones se ejecuten atómicamente?**

### Comportamiento de SaveChanges

#### Sin Transacción Explícita

Cada llamada a `SaveChanges()` crea su propia transacción implícita:

```csharp
public async Task ActualizarProductoYCliente()
{
    // Operación 1
    var producto = await _productoRepo.ObtenerPorIdAsync(1);
    producto.Precio = 150;
    await _productoRepo.ActualizarAsync(); 
    // ✅ Commit automático - cambio persistido
    
    // Operación 2
    var cliente = await _clienteRepo.ObtenerPorIdAsync(1);
    cliente.Credito -= 150;
    await _clienteRepo.ActualizarAsync();
    // ⚠️ Si falla aquí, el producto YA fue actualizado
    
    // ❌ PROBLEMA: No hay atomicidad entre operaciones
}
```

**Resultado:** Dos transacciones independientes. Si la segunda falla, la primera ya se persistió.

#### Con Transacción Explícita

```csharp
[Transactional] // Atributo personalizado
public async Task ActualizarProductoYCliente()
{
    // Operación 1
    var producto = await _productoRepo.ObtenerPorIdAsync(1);
    producto.Precio = 150;
    await _productoRepo.ActualizarAsync(); 
    // ⏸️ NO hace commit, solo ejecuta SQL
    
    // Operación 2
    var cliente = await _clienteRepo.ObtenerPorIdAsync(1);
    cliente.Credito -= 150;
    await _clienteRepo.ActualizarAsync();
    // ⏸️ NO hace commit, solo ejecuta SQL
    
    // ✅ Al finalizar el método, commit de TODA la transacción
    // Si cualquier operación falla, TODO hace rollback
}
```

**Resultado:** Una única transacción que engloba ambas operaciones.

### Cómo Funciona Internamente

```csharp
// Pseudocódigo de SaveChanges en EF Core
public int SaveChanges()
{
    // ¿Existe una transacción activa?
    if (Database.CurrentTransaction != null)
    {
        // ✅ Usar la transacción existente
        // NO crear transacción nueva
        // NO hacer commit automático
        return ExecuteChanges();
    }
    else
    {
        // ⚠️ No hay transacción
        // Crear transacción implícita
        using var transaction = Database.BeginTransaction();
        try
        {
            var result = ExecuteChanges();
            transaction.Commit(); // Commit automático
            return result;
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }
}
```

**Conclusión clave:** EF Core detecta automáticamente si hay una transacción activa y se integra con ella.

### Implementación del Atributo [Transactional]

```csharp
// Atributo personalizado
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class TransactionalAttribute : Attribute
{
}

// Implementación con MediatR Pipeline Behavior
public class TransactionalBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly DbContext _context;

    public TransactionalBehavior(DbContext context)
    {
        _context = context;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // Verificar si tiene el atributo [Transactional]
        var handlerType = request.GetType();
        var hasAttribute = handlerType
            .GetCustomAttributes(typeof(TransactionalAttribute), true)
            .Any();

        if (!hasAttribute)
        {
            // Sin transacción, ejecutar normalmente
            return await next();
        }

        // Crear estrategia de ejecución (maneja reintentos)
        var strategy = _context.Database.CreateExecutionStrategy();
        
        return await strategy.ExecuteAsync(async () =>
        {
            // Iniciar transacción
            await using var transaction = 
                await _context.Database.BeginTransactionAsync(cancellationToken);
            
            try
            {
                var response = await next();
                
                // Commit si todo salió bien
                await transaction.CommitAsync(cancellationToken);
                return response;
            }
            catch
            {
                // Rollback en caso de error
                await transaction.RollbackAsync(cancellationToken);
                throw;
            }
        });
    }
}
```

### El Problema del Rendimiento

Llamar a `SaveChanges()` múltiples veces dentro de una transacción funciona, pero **no es óptimo**:

```csharp
[Transactional]
public async Task ActualizarMuchosProductos(List<int> productosIds)
{
    foreach (var id in productosIds)
    {
        var producto = await _productoRepo.ObtenerPorIdAsync(id);
        producto.Precio += 10;
        await _productoRepo.ActualizarAsync(); // ❌ SaveChanges en cada iteración
    }
    
    // ✅ Todo en una transacción, PERO...
    // ❌ Ejecutaste N comandos SQL individuales
}
```

**Impacto:**
- ✅ Atomicidad garantizada
- ❌ Rendimiento subóptimo (muchos round-trips a la BD)
- ❌ Mayor tiempo de bloqueo de la transacción

### Soluciones Recomendadas

#### Opción 1: SaveChanges Solo al Final (Recomendado)

```csharp
public class RepositorioGenerico<T> where T : class
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly DbSet<T> _dbSet;

    // NO llama a SaveChanges automáticamente
    public void Agregar(T entidad)
    {
        _unitOfWork.Entry(entidad).State = EntityState.Added;
    }

    public void Eliminar(T entidad)
    {
        _unitOfWork.Entry(entidad).State = EntityState.Deleted;
    }

    // Para UPDATE no necesitas método especial
    // Solo modificas la entidad que ya está en tracking
    
    public async Task<T?> ObtenerPorIdAsync(object id)
    {
        return await _dbSet.FindAsync(id);
    }
}
```

**Uso en el Use Case:**

```csharp
[Transactional]
public async Task ActualizarMultiplesEntidades(ActualizarDto dto)
{
    // Operación 1: Modificar producto
    var producto = await _productoRepo.ObtenerPorIdAsync(dto.ProductoId);
    producto.Precio = dto.NuevoPrecio;
    producto.Stock -= dto.Cantidad;
    
    // Operación 2: Modificar cliente
    var cliente = await _clienteRepo.ObtenerPorIdAsync(dto.ClienteId);
    cliente.Credito -= dto.Total;
    
    // Operación 3: Agregar pedido
    var pedido = new Pedido
    {
        ClienteId = dto.ClienteId,
        ProductoId = dto.ProductoId,
        Cantidad = dto.Cantidad,
        Total = dto.Total
    };
    _pedidoRepo.Agregar(pedido);
    
    // ✅ UN SOLO SaveChanges para todas las operaciones
    await _unitOfWork.SaveChangesAsync();
    
    // Commit automático de la transacción al finalizar
}
```

**Ventajas:**
- ✅ Máximo rendimiento (un solo batch SQL)
- ✅ Control explícito sobre la persistencia
- ✅ Código más limpio y comprensible

#### Opción 2: Parámetro Opcional para AutoSave

```csharp
public class RepositorioGenerico<T> where T : class
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly DbSet<T> _dbSet;

    public async Task AgregarAsync(T entidad, bool autoSave = true)
    {
        _unitOfWork.Entry(entidad).State = EntityState.Added;
        
        if (autoSave)
        {
            await _unitOfWork.SaveChangesAsync();
        }
    }

    public async Task EliminarAsync(T entidad, bool autoSave = true)
    {
        _unitOfWork.Entry(entidad).State = EntityState.Deleted;
        
        if (autoSave)
        {
            await _unitOfWork.SaveChangesAsync();
        }
    }

    public async Task ActualizarAsync(bool autoSave = true)
    {
        if (autoSave)
        {
            await _unitOfWork.SaveChangesAsync();
        }
    }
}
```

**Uso:**

```csharp
[Transactional]
public async Task ActualizarMultiplesEntidades(ActualizarDto dto)
{
    // Deshabilitar autoSave en cada operación
    await _productoRepo.AgregarAsync(producto, autoSave: false);
    await _clienteRepo.EliminarAsync(cliente, autoSave: false);
    await _pedidoRepo.ActualizarAsync(autoSave: false);
    
    // Guardar todo de una vez
    await _unitOfWork.SaveChangesAsync();
}
```

### Tabla Comparativa de Comportamientos

| Escenario | Transacciones | Commits | Rendimiento | Atomicidad |
|-----------|---------------|---------|-------------|------------|
| Sin `[Transactional]` | Una por cada `SaveChanges()` | Múltiples independientes | ⚠️ Medio | ❌ No garantizada |
| Con `[Transactional]` + múltiples `SaveChanges()` | Una única transacción | Un commit al final | ⚠️ Subóptimo | ✅ Garantizada |
| Con `[Transactional]` + un `SaveChanges()` final | Una única transacción | Un commit al final | ✅ Óptimo | ✅ Garantizada |
| Sin transacción + un `SaveChanges()` final | Una transacción implícita | Un commit | ✅ Óptimo | ✅ Para esa operación |

### Ejemplo Completo: Caso de Uso Real

```csharp
// Use Case: Procesar una venta
[Transactional]
public class ProcesarVentaUseCase
{
    private readonly RepositorioGenerico<Producto> _productoRepo;
    private readonly RepositorioGenerico<Cliente> _clienteRepo;
    private readonly RepositorioGenerico<Venta> _ventaRepo;
    private readonly IUnitOfWork _unitOfWork;

    public async Task<VentaDto> EjecutarAsync(ProcesarVentaCommand command)
    {
        // 1. Validar y obtener producto
        var producto = await _productoRepo.ObtenerPorIdAsync(command.ProductoId);
        if (producto == null)
            throw new NotFoundException("Producto no encontrado");
        
        if (producto.Stock < command.Cantidad)
            throw new BusinessException("Stock insuficiente");

        // 2. Validar y obtener cliente
        var cliente = await _clienteRepo.ObtenerPorIdAsync(command.ClienteId);
        if (cliente == null)
            throw new NotFoundException("Cliente no encontrado");
        
        var total = producto.Precio * command.Cantidad;
        if (cliente.Credito < total)
            throw new BusinessException("Crédito insuficiente");

        // 3. Actualizar producto (ya está en tracking)
        producto.Stock -= command.Cantidad;
        producto.VentasTotales += command.Cantidad;

        // 4. Actualizar cliente (ya está en tracking)
        cliente.Credito -= total;
        cliente.TotalCompras += total;

        // 5. Crear venta
        var venta = new Venta
        {
            ClienteId = command.ClienteId,
            ProductoId = command.ProductoId,
            Cantidad = command.Cantidad,
            PrecioUnitario = producto.Precio,
            Total = total,
            Fecha = DateTime.UtcNow
        };
        _ventaRepo.Agregar(venta);

        // 6. UN SOLO SaveChanges para todas las operaciones
        await _unitOfWork.SaveChangesAsync();
        
        // Si llegamos aquí, la transacción hace commit automático
        // Si cualquier paso falló, rollback automático
        
        return MapToDto(venta);
    }
}
```

### Configuración en Dependency Injection

```csharp
// Program.cs
builder.Services.AddDbContext<MiDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")
    )
);

builder.Services.AddScoped<IUnitOfWork>(provider => 
    provider.GetRequiredService<MiDbContext>()
);

// Si usas MediatR
builder.Services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
    // Registrar el behavior transaccional
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(TransactionalBehavior<,>));
});
```

### Mejores Prácticas

#### ✅ Hacer

1. **Usar `[Transactional]` para operaciones multi-entidad**
   ```csharp
   [Transactional]
   public async Task OperacionCompleja() { }
   ```

2. **Llamar a `SaveChanges()` UNA SOLA VEZ al final**
   ```csharp
   // Todas las modificaciones...
   await _unitOfWork.SaveChangesAsync(); // Al final
   ```

3. **Mantener transacciones cortas y enfocadas**
   ```csharp
   // Solo la lógica de persistencia dentro de la transacción
   ```

4. **Usar repositorios sin AutoSave para transacciones**
   ```csharp
   public void Agregar(T entidad) 
   { 
       _unitOfWork.Entry(entidad).State = EntityState.Added;
       // Sin SaveChanges
   }
   ```

#### ❌ Evitar

1. **Múltiples `SaveChanges()` en bucles**
   ```csharp
   foreach (var item in items)
   {
       // modificar item
       await repo.ActualizarAsync(); // ❌ Ineficiente
   }
   ```

2. **Transacciones muy largas**
   ```csharp
   [Transactional]
   public async Task ProcesoLargo()
   {
       // Llamadas HTTP, esperas largas, etc. ❌
       await _unitOfWork.SaveChangesAsync();
   }
   ```

3. **Lógica de negocio fuera de transacciones cuando se requiere atomicidad**
   ```csharp
   // Sin [Transactional]
   public async Task ActualizarVariasEntidades() // ❌ Riesgo
   {
       await repo1.ActualizarAsync();
       await repo2.ActualizarAsync(); // No atómico
   }
   ```

### Resumen Visual

```
┌─────────────────────────────────────────────────────────────┐
│              Flujo de Transacción con [Transactional]        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Transactional] ────► BEGIN TRANSACTION                    │
│       │                                                      │
│       ├─► Operación 1 (tracked)                            │
│       │                                                      │
│       ├─► Operación 2 (tracked)                            │
│       │                                                      │
│       ├─► Operación 3 (Entry.State = Added)                │
│       │                                                      │
│       └─► SaveChanges() ────► Ejecuta SQL en BD            │
│                              (sin commit aún)                │
│                                                              │
│  Fin del método ────────────► COMMIT TRANSACTION            │
│                               ✅ Todo persistido             │
│                                                              │
│  Si hay excepción ──────────► ROLLBACK TRANSACTION          │
│                               ❌ Nada persistido             │
└─────────────────────────────────────────────────────────────┘
```

---

## Conclusiones

### 🎯 Puntos Clave

1. **Para ADD y DELETE**: Usa `Entry().State = ...` directamente
   - No necesitas `Attach()` explícito
   - Solo necesitas el Id para DELETE

2. **Para UPDATE**: Lee con tracking y modifica
   - EF Core detecta automáticamente los cambios
   - SQL solo actualiza lo que cambió
   - Es más eficiente que `State = Modified`

3. **Unit of Work**: Simple y elegante
   - El `DbContext` ya implementa el patrón
   - Solo crea una interface para abstraer
   - No necesitas implementación explícita

### ✅ Ventajas de Este Enfoque

- **Código limpio**: Sin complejidad innecesaria
- **Alto rendimiento**: Aprovecha el change tracking automático
- **Testeable**: Fácil de mockear con interfaces
- **Mantenible**: Separación clara de responsabilidades
- **Transaccional**: Control total sobre cuándo persistir cambios

### 📚 Recursos Adicionales

- [Documentación oficial de EF Core](https://docs.microsoft.com/ef/core/)
- [Change Tracking in EF Core](https://docs.microsoft.com/ef/core/change-tracking/)
- [Unit of Work Pattern](https://martinfowler.com/eaaCatalog/unitOfWork.html)

---


