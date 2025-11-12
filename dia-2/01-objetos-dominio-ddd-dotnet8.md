# Objetos del Dominio en DDD con .NET 8

## Introducción

En Domain-Driven Design (DDD), el dominio de una aplicación se modela utilizando diferentes tipos de objetos, cada uno con responsabilidades específicas. Este documento describe los principales tipos de objetos y proporciona ejemplos prácticos en .NET 8.

---

## 1. Entity (Entidad)

### Definición

Objetos con identidad única que persiste en el tiempo, independientemente de sus atributos. Dos entidades son diferentes aunque tengan los mismos valores si tienen diferente identidad.

### Características

- Tienen un identificador único (ID)
- Su identidad permanece constante durante su ciclo de vida
- Dos entidades con el mismo ID son la misma entidad

### Patrón de Constructores Protected y Factory Methods

Las entidades utilizan **constructores protected** combinados con **factory methods estáticos públicos** por las siguientes razones:

1. **Evitar generación de eventos en recuperación de datos**: Cuando Entity Framework recupera entidades de la base de datos, usa el constructor sin parámetros protected. Esto evita que se generen eventos de dominio durante la hidratación.

2. **Control de creación**: Los factory methods estáticos controlan cómo se crean las entidades nuevas, asegurando que todas las validaciones se ejecuten y los eventos correspondientes se generen.

3. **Herencia**: Los setters protected permiten que clases derivadas puedan modificar las propiedades si es necesario.

**Flujo de trabajo:**
- **Nueva entidad**: Usuario → Factory Method → Constructor protected (con validaciones y eventos)
- **Recuperar de BD**: Entity Framework → Constructor protected sin parámetros (sin eventos)

### Clase Base Entity

```csharp
public abstract class Entity<TId> : IEquatable<Entity<TId>>
    where TId : notnull
{
    public TId Id { get; protected set; }

    // Constructor protegido - solo accesible desde clases derivadas y EF Core
    protected Entity(TId id)
    {
        Id = id;
    }

    // Constructor sin parámetros para EF Core
    protected Entity()
    {
        Id = default!;
    }

    public bool Equals(Entity<TId>? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        if (GetType() != other.GetType()) return false;
        
        return EqualityComparer<TId>.Default.Equals(Id, other.Id);
    }

    public override bool Equals(object? obj)
    {
        return obj is Entity<TId> entity && Equals(entity);
    }

    public override int GetHashCode()
    {
        return Id.GetHashCode();
    }

    public static bool operator ==(Entity<TId>? left, Entity<TId>? right)
    {
        return Equals(left, right);
    }

    public static bool operator !=(Entity<TId>? left, Entity<TId>? right)
    {
        return !Equals(left, right);
    }
}
```

### Ejemplo de Entidad

```csharp
// Entidad simple que forma parte de un agregado
public class LineaPedido : Entity<Guid>
{
    public Guid ProductoId { get; protected set; }
    public int Cantidad { get; protected set; }
    public decimal PrecioUnitario { get; protected set; }
    public decimal Subtotal => Cantidad * PrecioUnitario;

    // Constructor protegido - solo para EF Core y clases derivadas
    protected LineaPedido(Guid id, Guid productoId, int cantidad, decimal precioUnitario) 
        : base(id)
    {
        ProductoId = productoId;
        Cantidad = cantidad;
        PrecioUnitario = precioUnitario;
    }

    // Constructor sin parámetros para EF Core
    protected LineaPedido() : base() { }

    // Factory Method estático - punto de entrada público para crear la entidad
    public static LineaPedido Crear(Guid productoId, int cantidad, decimal precioUnitario)
    {
        if (cantidad <= 0)
            throw new ArgumentException("La cantidad debe ser mayor a cero");
        
        if (precioUnitario <= 0)
            throw new ArgumentException("El precio unitario debe ser mayor a cero");

        return new LineaPedido(Guid.NewGuid(), productoId, cantidad, precioUnitario);
    }

    // Métodos de negocio internos al agregado
    internal void ActualizarCantidad(int nuevaCantidad)
    {
        if (nuevaCantidad <= 0)
            throw new ArgumentException("La cantidad debe ser mayor a cero");
        
        Cantidad = nuevaCantidad;
    }
}
```

---

## 2. Value Object (Objeto de Valor)

### Definición

Objetos inmutables definidos únicamente por sus atributos, sin identidad conceptual. Dos objetos de valor con los mismos atributos son intercambiables.

### Características

- Inmutables
- No tienen identidad propia
- Se comparan por sus valores, no por referencia
- Suelen ser pequeños y cohesivos

### Clase Base ValueObject

```csharp
public abstract class ValueObject : IEquatable<ValueObject>
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public bool Equals(ValueObject? other)
    {
        if (other is null || GetType() != other.GetType())
            return false;

        return GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }

    public override bool Equals(object? obj)
    {
        return obj is ValueObject valueObject && Equals(valueObject);
    }

    public override int GetHashCode()
    {
        return GetEqualityComponents()
            .Aggregate(1, (current, obj) =>
            {
                unchecked
                {
                    return current * 23 + (obj?.GetHashCode() ?? 0);
                }
            });
    }

    public static bool operator ==(ValueObject? left, ValueObject? right)
    {
        return Equals(left, right);
    }

    public static bool operator !=(ValueObject? left, ValueObject? right)
    {
        return !Equals(left, right);
    }
}
```

### Ejemplo con Clase

```csharp
public class Dinero : ValueObject
{
    public decimal Cantidad { get; }
    public string Moneda { get; }

    public Dinero(decimal cantidad, string moneda)
    {
        if (cantidad < 0)
            throw new ArgumentException("La cantidad no puede ser negativa");
        
        if (string.IsNullOrWhiteSpace(moneda))
            throw new ArgumentException("La moneda es requerida");

        Cantidad = cantidad;
        Moneda = moneda.ToUpperInvariant();
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Cantidad;
        yield return Moneda;
    }

    public Dinero Sumar(Dinero otro)
    {
        if (Moneda != otro.Moneda)
            throw new DomainException($"No se pueden sumar monedas diferentes");

        return new Dinero(Cantidad + otro.Cantidad, Moneda);
    }

    public static Dinero operator +(Dinero a, Dinero b) => a.Sumar(b);

    public override string ToString() => $"{Cantidad:N2} {Moneda}";
}
```

### Ejemplo con Record

```csharp
public record Email
{
    public string Valor { get; init; }

    public Email(string valor)
    {
        if (string.IsNullOrWhiteSpace(valor))
            throw new ArgumentException("El email no puede estar vacío");

        if (!EsEmailValido(valor))
            throw new ArgumentException($"El email '{valor}' no es válido");

        Valor = valor.ToLowerInvariant();
    }

    private static bool EsEmailValido(string email)
    {
        return email.Contains('@') && email.Contains('.');
    }

    public override string ToString() => Valor;
}
```

---

## 3. Aggregate Root (Raíz de Agregado)

### Definición

Entidad principal que actúa como punto de entrada a un agregado. Define los límites de consistencia y encapsula un conjunto de entidades y objetos de valor relacionados.

### Características

- Es una entidad
- Define límites transaccionales
- Es el único punto de acceso desde fuera del agregado
- Garantiza la consistencia del conjunto
- Puede o no generar eventos de dominio

### Clase Base AggregateRoot

```csharp
// Un Aggregate Root es simplemente una entidad que marca el límite de consistencia
public abstract class AggregateRoot<TId> : Entity<TId>
    where TId : notnull
{
    // Constructor protegido - solo accesible desde clases derivadas y EF Core
    protected AggregateRoot(TId id) : base(id)
    {
    }

    // Constructor sin parámetros para EF Core
    protected AggregateRoot() : base()
    {
    }
}
```

### Clase Base para AggregateRoot con Eventos (Opcional)

```csharp
// Solo si el Aggregate Root necesita generar eventos de dominio
public abstract class AggregateRootWithEvents<TId> : AggregateRoot<TId>
    where TId : notnull
{
    private readonly List<DomainEvent> _domainEvents = new();
    public IReadOnlyCollection<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    // Constructor protegido - solo accesible desde clases derivadas y EF Core
    protected AggregateRootWithEvents(TId id) : base(id)
    {
    }

    // Constructor sin parámetros para EF Core
    protected AggregateRootWithEvents() : base()
    {
    }

    protected void RaiseDomainEvent(DomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}
```

### Ejemplo: Aggregate Root sin Eventos

```csharp
public class Categoria : AggregateRoot<Guid>
{
    public string Nombre { get; protected set; }
    public string? Descripcion { get; protected set; }
    public bool Activa { get; protected set; }

    // Constructor protegido - solo para EF Core y clases derivadas
    protected Categoria(Guid id, string nombre) : base(id)
    {
        Nombre = nombre;
        Activa = true;
    }

    // Constructor sin parámetros para EF Core
    protected Categoria() : base() { }

    // Factory Method estático - punto de entrada público
    public static Categoria Crear(string nombre)
    {
        if (string.IsNullOrWhiteSpace(nombre))
            throw new ArgumentException("El nombre es requerido");

        return new Categoria(Guid.NewGuid(), nombre);
    }

    public void Actualizar(string nombre, string? descripcion)
    {
        if (string.IsNullOrWhiteSpace(nombre))
            throw new ArgumentException("El nombre es requerido");

        Nombre = nombre;
        Descripcion = descripcion;
    }

    public void Desactivar() => Activa = false;
    public void Activar() => Activa = true;
}
```

### Ejemplo: Aggregate Root con Eventos

```csharp
public class Pedido : AggregateRootWithEvents<Guid>
{
    private readonly List<LineaPedido> _lineas = new();

    public Guid ClienteId { get; protected set; }
    public DateTime FechaCreacion { get; protected set; }
    public EstadoPedido Estado { get; protected set; }
    public IReadOnlyCollection<LineaPedido> Lineas => _lineas.AsReadOnly();

    // Constructor protegido - solo para EF Core y clases derivadas
    protected Pedido(Guid id, Guid clienteId) : base(id)
    {
        ClienteId = clienteId;
        FechaCreacion = DateTime.UtcNow;
        Estado = EstadoPedido.Borrador;
    }

    // Constructor sin parámetros para EF Core
    protected Pedido() : base() { }

    // Factory Method estático - punto de entrada público
    public static Pedido Crear(Guid clienteId)
    {
        if (clienteId == Guid.Empty)
            throw new ArgumentException("El ID del cliente no puede estar vacío");

        return new Pedido(Guid.NewGuid(), clienteId);
    }

    public void AgregarLinea(Guid productoId, int cantidad, decimal precioUnitario)
    {
        if (Estado != EstadoPedido.Borrador)
            throw new PedidoYaCerradoException(Id);

        var linea = LineaPedido.Crear(productoId, cantidad, precioUnitario);
        _lineas.Add(linea);
    }

    public void Confirmar()
    {
        if (Estado != EstadoPedido.Borrador)
            throw new DomainException("Solo se pueden confirmar pedidos en borrador");

        if (!_lineas.Any())
            throw new DomainException("No se puede confirmar un pedido sin líneas");

        Estado = EstadoPedido.Confirmado;
        RaiseDomainEvent(new PedidoConfirmadoEvent(Id, ClienteId, CalcularTotal().Cantidad));
    }

    public Dinero CalcularTotal()
    {
        var total = _lineas.Sum(l => l.Subtotal);
        return new Dinero(total, "EUR");
    }
}

public enum EstadoPedido
{
    Borrador,
    Confirmado,
    Enviado,
    Entregado,
    Cancelado
}
```

---

## 4. Domain Service (Servicio de Dominio)

### Definición

Operaciones o lógica de negocio que no pertenecen naturalmente a ninguna entidad u objeto de valor. Representan comportamientos del dominio que involucran múltiples objetos.

### Características

- Sin estado (stateless)
- Operan sobre entidades y objetos de valor
- Contienen lógica de negocio pura
- Nombrados con verbos del lenguaje ubicuo

### Ejemplo

```csharp
public interface IServicioCalculoDescuentos
{
    Dinero CalcularDescuento(Pedido pedido, Cliente cliente);
}

public class ServicioCalculoDescuentos : IServicioCalculoDescuentos
{
    public Dinero CalcularDescuento(Pedido pedido, Cliente cliente)
    {
        var total = pedido.CalcularTotal();
        decimal porcentajeDescuento = 0;

        // Lógica de negocio compleja que involucra múltiples entidades
        if (cliente.TipoCliente == TipoCliente.Premium)
            porcentajeDescuento += 0.10m; // 10%

        if (total.Cantidad > 1000)
            porcentajeDescuento += 0.05m; // 5% adicional

        if (pedido.Lineas.Count >= 10)
            porcentajeDescuento += 0.03m; // 3% adicional

        var descuento = total.Cantidad * porcentajeDescuento;
        return new Dinero(descuento, total.Moneda);
    }
}
```

---

## 5. Repository (Repositorio)

### Definición

Abstracciones que encapsulan la lógica de acceso a datos, proporcionando una interfaz orientada a colecciones para recuperar y persistir agregados.

### Características

- Ocultan los detalles de persistencia
- Trabajan con agregados completos
- Un repositorio por cada Aggregate Root
- Métodos típicos: Add, Update, Remove, FindById

### Ejemplo

```csharp
public interface IRepositorioPedidos
{
    Task<Pedido?> ObtenerPorIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<List<Pedido>> ObtenerPorClienteAsync(Guid clienteId, CancellationToken cancellationToken = default);
    Task AgregarAsync(Pedido pedido, CancellationToken cancellationToken = default);
    Task ActualizarAsync(Pedido pedido, CancellationToken cancellationToken = default);
}

// Implementación con EF Core
public class RepositorioPedidos : IRepositorioPedidos
{
    private readonly ApplicationDbContext _context;

    public RepositorioPedidos(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Pedido?> ObtenerPorIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return await _context.Pedidos
            .Include(p => p.Lineas)
            .FirstOrDefaultAsync(p => p.Id == id, cancellationToken);
    }

    public async Task<List<Pedido>> ObtenerPorClienteAsync(Guid clienteId, CancellationToken cancellationToken = default)
    {
        return await _context.Pedidos
            .Include(p => p.Lineas)
            .Where(p => p.ClienteId == clienteId)
            .ToListAsync(cancellationToken);
    }

    public async Task AgregarAsync(Pedido pedido, CancellationToken cancellationToken = default)
    {
        await _context.Pedidos.AddAsync(pedido, cancellationToken);
        await _context.SaveChangesAsync(cancellationToken);
    }

    public async Task ActualizarAsync(Pedido pedido, CancellationToken cancellationToken = default)
    {
        _context.Pedidos.Update(pedido);
        await _context.SaveChangesAsync(cancellationToken);
    }
}
```

---

## 6. Factory (Factoría)

### Definición

Encapsulan la lógica compleja de creación de objetos del dominio, especialmente agregados con múltiples componentes.

### Características

- Centralizan la lógica de construcción compleja
- Aseguran que los objetos se crean en estado válido
- Pueden ser métodos estáticos o clases dedicadas

### Ejemplo

```csharp
public static class PedidoFactory
{
    public static Pedido CrearPedidoVacio(Guid clienteId)
    {
        // Utiliza el Factory Method de la entidad
        return Pedido.Crear(clienteId);
    }

    public static async Task<Pedido> CrearPedidoDesdeCarritoAsync(
        Carrito carrito,
        IRepositorioProductos repositorioProductos,
        CancellationToken cancellationToken = default)
    {
        // Utiliza el Factory Method de la entidad
        var pedido = Pedido.Crear(carrito.ClienteId);

        foreach (var item in carrito.Items)
        {
            var producto = await repositorioProductos.ObtenerPorIdAsync(item.ProductoId, cancellationToken)
                ?? throw new DomainException($"Producto {item.ProductoId} no encontrado");

            pedido.AgregarLinea(producto.Id, item.Cantidad, producto.Precio);
        }

        return pedido;
    }
}
```

---

## 7. Domain Event (Evento de Dominio)

### Definición

Representan algo significativo que ha ocurrido en el dominio y que otras partes del sistema necesitan conocer.

### Características

- Inmutables
- Nombrados en pasado (PedidoConfirmado, ProductoAgotado)
- Contienen datos relevantes del evento
- Permiten desacoplar partes del sistema

### Clase Base y Ejemplos

```csharp
public abstract record DomainEvent
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public DateTime OcurridoEn { get; init; } = DateTime.UtcNow;
}

// Eventos específicos
public record PedidoConfirmadoEvent(Guid PedidoId, Guid ClienteId, decimal Total) : DomainEvent;
public record PedidoEnviadoEvent(Guid PedidoId, DateTime FechaEnvio) : DomainEvent;
public record ProductoAgotadoEvent(Guid ProductoId, string Nombre) : DomainEvent;
```

### Manejo de Eventos

```csharp
public interface IDomainEventHandler<in TEvent> where TEvent : DomainEvent
{
    Task HandleAsync(TEvent domainEvent, CancellationToken cancellationToken = default);
}

// Handler de ejemplo
public class EnviarEmailCuandoPedidoConfirmadoHandler 
    : IDomainEventHandler<PedidoConfirmadoEvent>
{
    private readonly IServicioEmail _servicioEmail;

    public EnviarEmailCuandoPedidoConfirmadoHandler(IServicioEmail servicioEmail)
    {
        _servicioEmail = servicioEmail;
    }

    public async Task HandleAsync(
        PedidoConfirmadoEvent domainEvent, 
        CancellationToken cancellationToken = default)
    {
        await _servicioEmail.EnviarEmailConfirmacionPedido(
            domainEvent.ClienteId,
            domainEvent.PedidoId,
            cancellationToken);
    }
}
```

---

## 8. Domain Exception (Excepción de Dominio)

### Definición

Excepciones específicas del dominio que representan violaciones de reglas de negocio.

### Características

- Heredan de una clase base común
- Nombradas según la regla de negocio violada
- Pueden contener información contextual
- Se lanzan cuando algo invalida la lógica del dominio

### Clase Base y Ejemplos

```csharp
// Excepción base del dominio
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    
    protected DomainException(string message, Exception innerException) 
        : base(message, innerException) { }
}

// Excepciones específicas
public class StockInsuficienteException : DomainException
{
    public int StockDisponible { get; }
    public int CantidadSolicitada { get; }
    
    public StockInsuficienteException(int stockDisponible, int cantidadSolicitada)
        : base($"Stock insuficiente. Disponible: {stockDisponible}, Solicitado: {cantidadSolicitada}")
    {
        StockDisponible = stockDisponible;
        CantidadSolicitada = cantidadSolicitada;
    }
}

public class PedidoYaCerradoException : DomainException
{
    public Guid PedidoId { get; }
    
    public PedidoYaCerradoException(Guid pedidoId)
        : base($"El pedido {pedidoId} ya está cerrado y no puede modificarse")
    {
        PedidoId = pedidoId;
    }
}
```

---

## Resumen de Conceptos Clave

### Jerarquía de Objetos

```
Entity<TId>
    ↓
AggregateRoot<TId>
    ↓
AggregateRootWithEvents<TId> (opcional)
```

### Reglas Importantes

1. **No todas las entidades son Aggregate Roots**: Las entidades como `LineaPedido` forman parte de un agregado pero no son la raíz.

2. **No todos los Aggregate Roots generan eventos**: Un `AggregateRoot` puede existir sin necesidad de publicar eventos de dominio.

3. **Un repositorio por Aggregate Root**: Solo las raíces de agregado tienen repositorios, no las entidades internas.

4. **Value Objects son inmutables**: Una vez creados, sus valores no cambian. Para modificar, se crea una nueva instancia.

5. **Las Domain Exceptions protegen las invariantes**: Se lanzan cuando se intenta violar una regla de negocio.

6. **Constructores protected con Factory Methods**: Los constructores son protected para que solo EF Core pueda usarlos durante la recuperación de datos. Los usuarios crean entidades mediante factory methods estáticos públicos que ejecutan validaciones y generan eventos.

7. **Setters protected para herencia**: Las propiedades usan setters protected para permitir modificaciones en clases derivadas si es necesario.

### Cuándo usar cada objeto

| Objeto | Usar cuando... |
|--------|----------------|
| **Entity** | El objeto tiene identidad única y forma parte de un agregado |
| **Value Object** | El objeto se define por sus atributos y es intercambiable |
| **Aggregate Root** | Necesitas definir límites de consistencia transaccional |
| **Domain Service** | La lógica involucra múltiples agregados o no pertenece a ninguno |
| **Repository** | Necesitas persistir y recuperar un Aggregate Root |
| **Factory** | La creación del objeto es compleja o requiere validaciones |
| **Domain Event** | Algo importante ha ocurrido y otras partes deben saberlo |
| **Domain Exception** | Se ha violado una regla de negocio |

---

## Conclusión

Estos objetos trabajan juntos para crear un modelo rico que refleja el lenguaje y las reglas del negocio, manteniéndolo aislado de preocupaciones técnicas de infraestructura. La correcta identificación y uso de cada tipo de objeto es fundamental para implementar DDD efectivamente en .NET 8.
