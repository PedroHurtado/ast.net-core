# Introducción a ASP.NET Core

## 1. Visión General

### ¿Qué es ASP.NET Core?

ASP.NET Core es un framework **open-source, multiplataforma y de alto rendimiento** para construir aplicaciones web modernas, APIs RESTful, microservicios y aplicaciones en tiempo real. Desarrollado por Microsoft, representa una reescritura completa de ASP.NET Framework, diseñado desde cero para ser modular, ligero y compatible con Windows, Linux y macOS.

### Características Principales

#### 1. Multiplataforma
ASP.NET Core se ejecuta en múltiples sistemas operativos:
- **Windows**: IIS, Kestrel
- **Linux**: Nginx, Apache (como reverse proxy)
- **macOS**: Para desarrollo y hosting

```bash
# Mismo código, múltiples plataformas
dotnet run  # Funciona en Windows, Linux y macOS
```

#### 2. Alto Rendimiento
Es uno de los frameworks web más rápidos según TechEmpower Benchmarks, superando a Node.js, Spring Boot y muchos otros.

**Ejemplo de rendimiento comparativo (requests/segundo):**
- ASP.NET Core: ~7,000,000
- Node.js: ~900,000
- Spring Boot: ~400,000

#### 3. Modular y Ligero
Solo incluyes lo que necesitas mediante paquetes NuGet:

```csharp
// Program.cs - Configuración mínima
var builder = WebApplication.CreateBuilder(args);

// Solo añades servicios que necesitas
builder.Services.AddControllers();      // Solo si necesitas Controllers
builder.Services.AddRazorPages();       // Solo si usas Razor Pages
builder.Services.AddSignalR();          // Solo si necesitas real-time

var app = builder.Build();
app.MapControllers();
app.Run();
```

#### 4. Inyección de Dependencias Integrada
DI es un ciudadano de primera clase:

```csharp
// Registrar servicios
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddTransient<IEmailService, EmailService>();
builder.Services.AddSingleton<ICacheService, CacheService>();

// Usar en controllers
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;
    
    public ProductsController(IProductRepository repository)
    {
        _repository = repository; // Inyección automática
    }
}
```

#### 5. Configuración Flexible
Múltiples fuentes de configuración:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Lee configuración de múltiples fuentes
// 1. appsettings.json
// 2. appsettings.{Environment}.json
// 3. Variables de entorno
// 4. Argumentos de línea de comandos
// 5. Azure Key Vault, etc.

var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var apiKey = builder.Configuration["ExternalApi:ApiKey"];
```

#### 6. Middleware Pipeline
Procesamiento de requests modular y configurable:

```csharp
var app = builder.Build();

// Pipeline de middleware - el orden importa
app.UseHttpsRedirection();      // 1. Redirige HTTP -> HTTPS
app.UseStaticFiles();           // 2. Sirve archivos estáticos
app.UseRouting();               // 3. Configura routing
app.UseAuthentication();        // 4. Autenticación
app.UseAuthorization();         // 5. Autorización
app.MapControllers();           // 6. Endpoints de controllers

app.Run();
```

### Arquitectura de ASP.NET Core

```
┌─────────────────────────────────────────┐
│         HTTP Request (Cliente)          │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│         Kestrel Web Server              │
│  (Servidor web multiplataforma)         │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│      Middleware Pipeline                │
│  ┌─────────────────────────────┐       │
│  │ Exception Handler            │       │
│  ├─────────────────────────────┤       │
│  │ HTTPS Redirection           │       │
│  ├─────────────────────────────┤       │
│  │ Static Files                │       │
│  ├─────────────────────────────┤       │
│  │ Routing                     │       │
│  ├─────────────────────────────┤       │
│  │ Authentication              │       │
│  ├─────────────────────────────┤       │
│  │ Authorization               │       │
│  ├─────────────────────────────┤       │
│  │ Custom Middleware           │       │
│  └─────────────────────────────┘       │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│      Endpoint / Controller              │
│  - MVC Controllers                      │
│  - Razor Pages                          │
│  - Minimal APIs                         │
│  - SignalR Hubs                         │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│      Business Logic / Services          │
│  (Inyección de Dependencias)            │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│      Data Access Layer                  │
│  - Entity Framework Core                │
│  - Dapper                               │
│  - ADO.NET                              │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│         Base de Datos                   │
└─────────────────────────────────────────┘
```

### Tipos de Aplicaciones

#### 1. Web APIs (RESTful)
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public ActionResult<IEnumerable<Product>> GetAll()
    {
        return Ok(products);
    }
    
    [HttpPost]
    public ActionResult<Product> Create(Product product)
    {
        // Lógica de creación
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }
}
```

#### 2. Minimal APIs (Nuevo en .NET 6+)
```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/products", () => 
{
    return Results.Ok(products);
});

app.MapPost("/api/products", (Product product) => 
{
    // Lógica de creación
    return Results.Created($"/api/products/{product.Id}", product);
});

app.Run();
```

#### 3. Razor Pages
```csharp
// Pages/Products/Index.cshtml.cs
public class IndexModel : PageModel
{
    private readonly IProductRepository _repository;
    
    public IndexModel(IProductRepository repository)
    {
        _repository = repository;
    }
    
    public List<Product> Products { get; set; }
    
    public async Task OnGetAsync()
    {
        Products = await _repository.GetAllAsync();
    }
}
```

#### 4. MVC (Model-View-Controller)
```csharp
public class HomeController : Controller
{
    public IActionResult Index()
    {
        return View();
    }
    
    public IActionResult About()
    {
        ViewData["Message"] = "Your application description page.";
        return View();
    }
}
```

#### 5. Blazor (WebAssembly o Server)
```razor
@page "/products"
@inject IProductService ProductService

<h3>Products</h3>

@if (products == null)
{
    <p><em>Cargando productos...</em></p>
}
else
{
    <div class="products-grid">
        @foreach (var product in products)
        {
            <div class="product-card">
                <h4>@product.Name</h4>
                <p class="price">@product.Price.ToString("C")</p>
                <button @onclick="() => AddToCart(product.Id)">
                    Agregar al carrito
                </button>
            </div>
        }
    </div>
}

@code {
    private List<Product>? products;
    
    protected override async Task OnInitializedAsync()
    {
        products = await ProductService.GetAllAsync();
    }
    
    private void AddToCart(int productId)
    {
        // Lógica para agregar al carrito
        Console.WriteLine($"Producto {productId} agregado al carrito");
    }
}
```

### Versiones de ASP.NET Core

| Versión | Lanzamiento | Características Principales | Soporte |
|---------|-------------|----------------------------|---------|
| .NET 9 | Nov 2024 | OpenAPI nativo, mejoras HybridCache | STS (18 meses) |
| .NET 8 | Nov 2023 | Native AOT, Identity API endpoints, Blazor mejoras | **LTS (3 años)** |
| .NET 7 | Nov 2022 | Rate limiting, Output caching | Finalizado |
| .NET 6 | Nov 2021 | Minimal APIs, Hot Reload | **LTS** (finaliza Nov 2024) |
| .NET 5 | Nov 2020 | Unificación .NET Core y .NET Framework | Finalizado |

**Recomendación actual:** Usar **.NET 8** (LTS) para producción o **.NET 9** para proyectos nuevos.

---

## 2. Configuración del Entorno de Desarrollo

### Requisitos Previos

#### Opción 1: .NET SDK (Esencial)

**Descarga e Instalación:**
1. Visita: https://dotnet.microsoft.com/download
2. Descarga .NET 8 SDK (o .NET 9 para última versión)
3. Ejecuta el instalador

**Verificación:**
```bash
dotnet --version
# Salida esperada: 8.0.xxx o 9.0.xxx

dotnet --info
# Muestra información completa del SDK instalado
```

**Comandos útiles:**
```bash
# Listar SDKs instalados
dotnet --list-sdks

# Listar runtimes instalados
dotnet --list-runtimes

# Ver plantillas disponibles
dotnet new list

# Ayuda general
dotnet --help
```

#### Opción 2: IDEs y Editores

##### Visual Studio 2022 (Windows/Mac)
**Características:**
- IntelliSense avanzado
- Debugging visual
- Diseñador de UI
- Profiler de rendimiento
- Integración con Azure

**Descarga:** https://visualstudio.microsoft.com/

**Workloads a instalar:**
- ASP.NET and web development
- .NET desktop development (opcional)
- Azure development (opcional)

**Extensiones recomendadas:**
- Web Essentials
- CodeMaid
- ReSharper (de pago, pero muy potente)

##### Visual Studio Code (Multiplataforma)
**Extensiones esenciales:**
```json
{
  "recommendations": [
    "ms-dotnettools.csharp",              // C# oficial
    "ms-dotnettools.csdevkit",            // C# Dev Kit
    "ms-dotnettools.vscode-dotnet-runtime",
    "kreativ-software.csharpextensions",  // Extensiones C#
    "jongrant.csharpsortusings",          // Organizar usings
    "patcx.vscode-nuget-gallery",         // Gestor NuGet visual
    "formulahendry.dotnet-test-explorer", // Test Explorer
    "ms-azuretools.vscode-docker"         // Docker support
  ]
}
```

**Configuración .vscode/launch.json:**
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": ".NET Core Launch (web)",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build",
      "program": "${workspaceFolder}/bin/Debug/net8.0/MyApp.dll",
      "args": [],
      "cwd": "${workspaceFolder}",
      "stopAtEntry": false,
      "serverReadyAction": {
        "action": "openExternally",
        "pattern": "\\bNow listening on:\\s+(https?://\\S+)"
      },
      "env": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      },
      "sourceFileMap": {
        "/Views": "${workspaceFolder}/Views"
      }
    }
  ]
}
```

**Configuración .vscode/tasks.json:**
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "build",
      "command": "dotnet",
      "type": "process",
      "args": [
        "build",
        "${workspaceFolder}/MyApp.csproj",
        "/property:GenerateFullPaths=true",
        "/consoleloggerparameters:NoSummary"
      ],
      "problemMatcher": "$msCompile"
    },
    {
      "label": "watch",
      "command": "dotnet",
      "type": "process",
      "args": [
        "watch",
        "run",
        "--project",
        "${workspaceFolder}/MyApp.csproj"
      ],
      "problemMatcher": "$msCompile"
    }
  ]
}
```

##### JetBrains Rider (Multiplataforma, de pago)
**Ventajas:**
- Refactoring avanzado
- Análisis de código en tiempo real
- Excelente para proyectos grandes
- Integración con bases de datos

**Descarga:** https://www.jetbrains.com/rider/

#### Herramientas Adicionales

##### Git
```bash
# Instalación Windows
winget install Git.Git

# Instalación macOS
brew install git

# Instalación Linux (Ubuntu/Debian)
sudo apt-get install git

# Verificación
git --version
```

##### Docker (Opcional pero recomendado)
**Para desarrollo con contenedores:**
```bash
# Instalación Windows/Mac
# Descargar Docker Desktop: https://www.docker.com/products/docker-desktop

# Instalación Linux
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Verificación
docker --version
docker-compose --version
```

##### SQL Server / PostgreSQL / MySQL
**Opciones para base de datos:**

**SQL Server (Windows/Linux/Docker):**
```bash
# Usando Docker
docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=YourStrong@Passw0rd" \
   -p 1433:1433 --name sqlserver \
   -d mcr.microsoft.com/mssql/server:2022-latest
```

**PostgreSQL:**
```bash
# Usando Docker
docker run --name postgres \
   -e POSTGRES_PASSWORD=mypassword \
   -p 5432:5432 \
   -d postgres:16
```

##### Postman / Insomnia
**Para testing de APIs:**
- Postman: https://www.postman.com/downloads/
- Insomnia: https://insomnia.rest/download

##### Azure CLI (Opcional)
```bash
# Instalación Windows
winget install Microsoft.AzureCLI

# Instalación macOS
brew install azure-cli

# Instalación Linux
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verificación
az --version
```

### Configuración del Entorno

#### Variables de Entorno

**Windows (PowerShell):**
```powershell
# Temporales (sesión actual)
$env:ASPNETCORE_ENVIRONMENT = "Development"
$env:ConnectionStrings__DefaultConnection = "Server=localhost;Database=MyDb;..."

# Permanentes (sistema)
[System.Environment]::SetEnvironmentVariable(
    "ASPNETCORE_ENVIRONMENT", 
    "Development", 
    [System.EnvironmentVariableTarget]::User
)
```

**Linux/macOS (Bash):**
```bash
# Temporales
export ASPNETCORE_ENVIRONMENT=Development
export ConnectionStrings__DefaultConnection="Server=localhost;..."

# Permanentes (añadir a ~/.bashrc o ~/.zshrc)
echo 'export ASPNETCORE_ENVIRONMENT=Development' >> ~/.bashrc
source ~/.bashrc
```

#### Configuración de HTTPS en Desarrollo

```bash
# Confiar en el certificado de desarrollo
dotnet dev-certs https --trust

# Limpiar y regenerar certificados si hay problemas
dotnet dev-certs https --clean
dotnet dev-certs https --trust
```

#### Configuración Global de .NET

```bash
# Ver configuración global
dotnet nuget list source

# Añadir fuente NuGet personalizada
dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org

# Configurar proxy (si es necesario)
dotnet nuget config -set http_proxy=http://proxy.company.com:8080
```

---

## 3. Crear un Proyecto de ASP.NET Core

### Método 1: CLI de .NET (Recomendado)

#### Web API (RESTful)
```bash
# Crear directorio del proyecto
mkdir MiApiProductos
cd MiApiProductos

# Crear proyecto Web API
dotnet new webapi -n MiApiProductos -f net8.0

# Estructura creada:
# MiApiProductos/
# ├── Controllers/
# │   └── WeatherForecastController.cs
# ├── Properties/
# │   └── launchSettings.json
# ├── appsettings.json
# ├── appsettings.Development.json
# ├── Program.cs
# ├── MiApiProductos.csproj
# └── WeatherForecast.cs

# Navegar al proyecto
cd MiApiProductos

# Restaurar dependencias
dotnet restore

# Compilar
dotnet build

# Ejecutar
dotnet run

# La aplicación estará disponible en:
# https://localhost:7xxx
# http://localhost:5xxx
```

#### Minimal API
```bash
# Crear proyecto minimalista
dotnet new web -n MiMinimalApi -f net8.0

# Estructura muy simple:
# MiMinimalApi/
# ├── Properties/
# │   └── launchSettings.json
# ├── appsettings.json
# ├── Program.cs
# └── MiMinimalApi.csproj
```

**Ejemplo Program.cs de Minimal API:**
```csharp
var builder = WebApplication.CreateBuilder(args);

// Añadir servicios
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configurar Swagger en Development
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// Definir endpoints
var productos = new List<Producto>
{
    new(1, "Laptop", 1200m),
    new(2, "Mouse", 25m),
    new(3, "Teclado", 75m)
};

app.MapGet("/api/productos", () => Results.Ok(productos))
    .WithName("GetProductos")
    .WithOpenApi();

app.MapGet("/api/productos/{id}", (int id) =>
{
    var producto = productos.FirstOrDefault(p => p.Id == id);
    return producto is not null ? Results.Ok(producto) : Results.NotFound();
})
    .WithName("GetProductoPorId")
    .WithOpenApi();

app.MapPost("/api/productos", (Producto producto) =>
{
    productos.Add(producto);
    return Results.Created($"/api/productos/{producto.Id}", producto);
})
    .WithName("CrearProducto")
    .WithOpenApi();

app.Run();

record Producto(int Id, string Nombre, decimal Precio);
```

#### Razor Pages
```bash
# Crear proyecto Razor Pages
dotnet new razor -n MiWebRazor -f net8.0

# Estructura:
# MiWebRazor/
# ├── Pages/
# │   ├── Shared/
# │   │   ├── _Layout.cshtml
# │   │   └── _ValidationScriptsPartial.cshtml
# │   ├── _ViewImports.cshtml
# │   ├── _ViewStart.cshtml
# │   ├── Error.cshtml
# │   ├── Error.cshtml.cs
# │   ├── Index.cshtml
# │   ├── Index.cshtml.cs
# │   ├── Privacy.cshtml
# │   └── Privacy.cshtml.cs
# ├── wwwroot/
# │   ├── css/
# │   ├── js/
# │   └── lib/
# ├── appsettings.json
# ├── Program.cs
# └── MiWebRazor.csproj
```

#### MVC
```bash
# Crear proyecto MVC
dotnet new mvc -n MiWebMvc -f net8.0

# Estructura:
# MiWebMvc/
# ├── Controllers/
# │   └── HomeController.cs
# ├── Models/
# │   └── ErrorViewModel.cs
# ├── Views/
# │   ├── Home/
# │   │   ├── Index.cshtml
# │   │   └── Privacy.cshtml
# │   ├── Shared/
# │   │   ├── _Layout.cshtml
# │   │   └── _ValidationScriptsPartial.cshtml
# │   ├── _ViewImports.cshtml
# │   └── _ViewStart.cshtml
# ├── wwwroot/
# ├── appsettings.json
# ├── Program.cs
# └── MiWebMvc.csproj
```

### Método 2: Visual Studio

1. **Abrir Visual Studio 2022**
2. **Crear nuevo proyecto:**
   - File → New → Project
   - Buscar "ASP.NET Core Web API"
   - Seleccionar plantilla
   - Click "Next"

3. **Configurar proyecto:**
   ```
   Project name: MiApiProductos
   Location: C:\Projects\
   Solution name: MiApiProductos
   ☑ Place solution and project in the same directory
   ```

4. **Información adicional:**
   ```
   Framework: .NET 8.0 (Long Term Support)
   Authentication type: None
   ☑ Configure for HTTPS
   ☑ Enable OpenAPI support (Swagger)
   ☐ Use controllers
   ☐ Enable Docker
   ☐ Do not use top-level statements
   ```

5. **Click "Create"**

### Método 3: Visual Studio Code

1. **Abrir VS Code**
2. **Abrir terminal integrada** (Ctrl + `)
3. **Ejecutar comandos:**
```bash
mkdir MiApiProductos
cd MiApiProductos
dotnet new webapi -n MiApiProductos
code .
```
4. **Aceptar instalación de assets requeridos** (C# extension)

### Estructura de un Proyecto Web API

```
MiApiProductos/
├── Controllers/                    # Controladores de la API
│   └── WeatherForecastController.cs
│
├── Properties/                     # Configuración de ejecución
│   └── launchSettings.json        # Perfiles de ejecución
│
├── appsettings.json               # Configuración general
├── appsettings.Development.json   # Configuración de desarrollo
│
├── Program.cs                      # Punto de entrada de la aplicación
│
├── MiApiProductos.csproj          # Archivo de proyecto
│
└── WeatherForecast.cs             # Modelo de ejemplo
```

### Análisis del Archivo Program.cs

```csharp
// Program.cs - Configuración moderna (Minimal Hosting Model)

// 1. Crear el builder de la aplicación
var builder = WebApplication.CreateBuilder(args);

// 2. Configurar servicios (Dependency Injection Container)
builder.Services.AddControllers(); // Añadir soporte para controllers

// Configurar Swagger para documentación de API
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Ejemplos de otros servicios comunes:
// builder.Services.AddDbContext<ApplicationDbContext>(options =>
//     options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
// 
// builder.Services.AddScoped<IProductRepository, ProductRepository>();
// builder.Services.AddTransient<IEmailService, EmailService>();
// builder.Services.AddSingleton<ICacheService, CacheService>();

// 3. Construir la aplicación
var app = builder.Build();

// 4. Configurar el pipeline de middleware (HTTP request pipeline)

// Swagger solo en desarrollo
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();       // Genera especificación OpenAPI
    app.UseSwaggerUI();     // UI interactiva de Swagger
}

app.UseHttpsRedirection();  // Redirigir HTTP a HTTPS

app.UseAuthorization();     // Middleware de autorización

app.MapControllers();       // Mapear endpoints de controllers

// 5. Ejecutar la aplicación
app.Run();
```

### Ejemplo Completo: API de Productos

#### 1. Crear el proyecto
```bash
dotnet new webapi -n ProductosApi -f net8.0
cd ProductosApi
```

#### 2. Instalar paquetes necesarios
```bash
# Entity Framework Core para SQL Server
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools

# FluentValidation para validaciones
dotnet add package FluentValidation.AspNetCore
```

#### 3. Crear el modelo
```csharp
// Models/Producto.cs
namespace ProductosApi.Models;

public class Producto
{
    public int Id { get; set; }
    public string Nombre { get; set; } = string.Empty;
    public string Descripcion { get; set; } = string.Empty;
    public decimal Precio { get; set; }
    public int Stock { get; set; }
    public DateTime FechaCreacion { get; set; } = DateTime.UtcNow;
    public bool Activo { get; set; } = true;
}
```

#### 4. Crear el DbContext
```csharp
// Data/ApplicationDbContext.cs
using Microsoft.EntityFrameworkCore;
using ProductosApi.Models;

namespace ProductosApi.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Producto> Productos => Set<Producto>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Producto>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Nombre).IsRequired().HasMaxLength(100);
            entity.Property(e => e.Precio).HasColumnType("decimal(18,2)");
            
            // Datos de prueba
            entity.HasData(
                new Producto { Id = 1, Nombre = "Laptop HP", Descripcion = "Laptop 15 pulgadas", Precio = 899.99m, Stock = 10 },
                new Producto { Id = 2, Nombre = "Mouse Logitech", Descripcion = "Mouse inalámbrico", Precio = 29.99m, Stock = 50 },
                new Producto { Id = 3, Nombre = "Teclado Mecánico", Descripcion = "Teclado RGB", Precio = 79.99m, Stock = 25 }
            );
        });
    }
}
```

#### 5. Crear el repositorio
```csharp
// Repositories/IProductoRepository.cs
using ProductosApi.Models;

namespace ProductosApi.Repositories;

public interface IProductoRepository
{
    Task<IEnumerable<Producto>> GetAllAsync();
    Task<Producto?> GetByIdAsync(int id);
    Task<Producto> CreateAsync(Producto producto);
    Task<Producto?> UpdateAsync(int id, Producto producto);
    Task<bool> DeleteAsync(int id);
    Task<bool> ExistsAsync(int id);
}

// Repositories/ProductoRepository.cs
using Microsoft.EntityFrameworkCore;
using ProductosApi.Data;
using ProductosApi.Models;

namespace ProductosApi.Repositories;

public class ProductoRepository : IProductoRepository
{
    private readonly ApplicationDbContext _context;

    public ProductoRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<Producto>> GetAllAsync()
    {
        return await _context.Productos
            .Where(p => p.Activo)
            .ToListAsync();
    }

    public async Task<Producto?> GetByIdAsync(int id)
    {
        return await _context.Productos
            .FirstOrDefaultAsync(p => p.Id == id && p.Activo);
    }

    public async Task<Producto> CreateAsync(Producto producto)
    {
        _context.Productos.Add(producto);
        await _context.SaveChangesAsync();
        return producto;
    }

    public async Task<Producto?> UpdateAsync(int id, Producto producto)
    {
        var existingProducto = await _context.Productos.FindAsync(id);
        if (existingProducto is null)
            return null;

        existingProducto.Nombre = producto.Nombre;
        existingProducto.Descripcion = producto.Descripcion;
        existingProducto.Precio = producto.Precio;
        existingProducto.Stock = producto.Stock;

        await _context.SaveChangesAsync();
        return existingProducto;
    }

    public async Task<bool> DeleteAsync(int id)
    {
        var producto = await _context.Productos.FindAsync(id);
        if (producto is null)
            return false;

        producto.Activo = false; // Soft delete
        await _context.SaveChangesAsync();
        return true;
    }

    public async Task<bool> ExistsAsync(int id)
    {
        return await _context.Productos.AnyAsync(p => p.Id == id && p.Activo);
    }
}
```

#### 6. Crear DTOs
```csharp
// DTOs/ProductoDto.cs
namespace ProductosApi.DTOs;

public record ProductoDto(
    int Id,
    string Nombre,
    string Descripcion,
    decimal Precio,
    int Stock,
    DateTime FechaCreacion
);

public record CreateProductoDto(
    string Nombre,
    string Descripcion,
    decimal Precio,
    int Stock
);

public record UpdateProductoDto(
    string Nombre,
    string Descripcion,
    decimal Precio,
    int Stock
);
```

#### 7. Crear el Controller
```csharp
// Controllers/ProductosController.cs
using Microsoft.AspNetCore.Mvc;
using ProductosApi.DTOs;
using ProductosApi.Models;
using ProductosApi.Repositories;

namespace ProductosApi.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ProductosController : ControllerBase
{
    private readonly IProductoRepository _repository;
    private readonly ILogger<ProductosController> _logger;

    public ProductosController(
        IProductoRepository repository,
        ILogger<ProductosController> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    /// <summary>
    /// Obtiene todos los productos activos
    /// </summary>
    [HttpGet]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<ProductoDto>>> GetAll()
    {
        _logger.LogInformation("Obteniendo todos los productos");
        
        var productos = await _repository.GetAllAsync();
        var productosDto = productos.Select(p => new ProductoDto(
            p.Id, p.Nombre, p.Descripcion, p.Precio, p.Stock, p.FechaCreacion
        ));
        
        return Ok(productosDto);
    }

    /// <summary>
    /// Obtiene un producto por ID
    /// </summary>
    [HttpGet("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductoDto>> GetById(int id)
    {
        var producto = await _repository.GetByIdAsync(id);
        
        if (producto is null)
        {
            _logger.LogWarning("Producto {Id} no encontrado", id);
            return NotFound();
        }

        var productoDto = new ProductoDto(
            producto.Id, producto.Nombre, producto.Descripcion, 
            producto.Precio, producto.Stock, producto.FechaCreacion
        );
        
        return Ok(productoDto);
    }

    /// <summary>
    /// Crea un nuevo producto
    /// </summary>
    [HttpPost]
    [ProducesResponseType(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ProductoDto>> Create(CreateProductoDto createDto)
    {
        var producto = new Producto
        {
            Nombre = createDto.Nombre,
            Descripcion = createDto.Descripcion,
            Precio = createDto.Precio,
            Stock = createDto.Stock
        };

        var createdProducto = await _repository.CreateAsync(producto);
        
        var productoDto = new ProductoDto(
            createdProducto.Id, createdProducto.Nombre, createdProducto.Descripcion,
            createdProducto.Precio, createdProducto.Stock, createdProducto.FechaCreacion
        );

        _logger.LogInformation("Producto {Id} creado exitosamente", createdProducto.Id);
        
        return CreatedAtAction(
            nameof(GetById), 
            new { id = createdProducto.Id }, 
            productoDto
        );
    }

    /// <summary>
    /// Actualiza un producto existente
    /// </summary>
    [HttpPut("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductoDto>> Update(int id, UpdateProductoDto updateDto)
    {
        var producto = new Producto
        {
            Nombre = updateDto.Nombre,
            Descripcion = updateDto.Descripcion,
            Precio = updateDto.Precio,
            Stock = updateDto.Stock
        };

        var updatedProducto = await _repository.UpdateAsync(id, producto);
        
        if (updatedProducto is null)
        {
            _logger.LogWarning("Producto {Id} no encontrado para actualizar", id);
            return NotFound();
        }

        var productoDto = new ProductoDto(
            updatedProducto.Id, updatedProducto.Nombre, updatedProducto.Descripcion,
            updatedProducto.Precio, updatedProducto.Stock, updatedProducto.FechaCreacion
        );

        _logger.LogInformation("Producto {Id} actualizado exitosamente", id);
        
        return Ok(productoDto);
    }

    /// <summary>
    /// Elimina un producto (soft delete)
    /// </summary>
    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _repository.DeleteAsync(id);
        
        if (!deleted)
        {
            _logger.LogWarning("Producto {Id} no encontrado para eliminar", id);
            return NotFound();
        }

        _logger.LogInformation("Producto {Id} eliminado exitosamente", id);
        
        return NoContent();
    }
}
```

#### 8. Configurar Program.cs
```csharp
// Program.cs
using Microsoft.EntityFrameworkCore;
using ProductosApi.Data;
using ProductosApi.Repositories;

var builder = WebApplication.CreateBuilder(args);

// Configurar DbContext
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseInMemoryDatabase("ProductosDb")); // Usar InMemory para ejemplo
    // Para SQL Server real:
    // options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Registrar repositorios
builder.Services.AddScoped<IProductoRepository, ProductoRepository>();

// Configurar controllers
builder.Services.AddControllers();

// Configurar Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new()
    {
        Title = "Productos API",
        Version = "v1",
        Description = "API para gestión de productos"
    });
});

// Configurar CORS (opcional)
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

var app = builder.Build();

// Configurar pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("AllowAll");
app.UseAuthorization();
app.MapControllers();

app.Run();
```

#### 9. Ejecutar el proyecto
```bash
# Compilar
dotnet build

# Ejecutar
dotnet run

# O con hot reload
dotnet watch run
```

#### 10. Probar la API

**Usando curl:**
```bash
# GET - Obtener todos los productos
curl https://localhost:7xxx/api/productos

# GET - Obtener producto por ID
curl https://localhost:7xxx/api/productos/1

# POST - Crear producto
curl -X POST https://localhost:7xxx/api/productos \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Monitor Samsung",
    "descripcion": "Monitor 27 pulgadas",
    "precio": 299.99,
    "stock": 15
  }'

# PUT - Actualizar producto
curl -X PUT https://localhost:7xxx/api/productos/1 \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Laptop HP Actualizada",
    "descripcion": "Laptop 15 pulgadas - Mejorada",
    "precio": 799.99,
    "stock": 8
  }'

# DELETE - Eliminar producto
curl -X DELETE https://localhost:7xxx/api/productos/1
```

**Usando Swagger:**
1. Navega a `https://localhost:7xxx/swagger`
2. Prueba los endpoints directamente desde la interfaz

### Comandos Útiles de .NET CLI

```bash
# Crear solución
dotnet new sln -n MiSolucion

# Añadir proyecto a la solución
dotnet sln add MiApiProductos/MiApiProductos.csproj

# Añadir paquete NuGet
dotnet add package NombreDelPaquete
dotnet add package NombreDelPaquete --version 1.2.3

# Remover paquete
dotnet remove package NombreDelPaquete

# Listar paquetes instalados
dotnet list package

# Restaurar dependencias
dotnet restore

# Limpiar compilación
dotnet clean

# Compilar en Release
dotnet build --configuration Release

# Publicar aplicación
dotnet publish --configuration Release --output ./publish

# Ejecutar tests
dotnet test

# Crear migración de EF Core
dotnet ef migrations add InitialCreate

# Aplicar migraciones
dotnet ef database update

# Formatear código
dotnet format

# Ver información del proyecto
dotnet --info
```

---

## Referencias de Calidad

### Documentación Oficial

1. **Microsoft Learn - ASP.NET Core**
   - https://learn.microsoft.com/en-us/aspnet/core/
   - Documentación oficial completa y actualizada

2. **ASP.NET Core GitHub Repository**
   - https://github.com/dotnet/aspnetcore
   - Código fuente, issues, y contribuciones

3. **C# Documentation**
   - https://learn.microsoft.com/en-us/dotnet/csharp/
   - Guía completa del lenguaje C#

### Tutoriales y Guías

4. **Microsoft Learn Training Paths**
   - https://learn.microsoft.com/en-us/training/paths/aspnet-core-web-api/
   - Rutas de aprendizaje estructuradas y gratuitas

5. **ASP.NET Core Best Practices**
   - https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices
   - Mejores prácticas oficiales

6. **Entity Framework Core Documentation**
   - https://learn.microsoft.com/en-us/ef/core/
   - ORM oficial de .NET

### Blogs y Recursos Comunitarios

7. **Scott Hanselman's Blog**
   - https://www.hanselman.com/blog/
   - Contenido de calidad sobre .NET y ASP.NET Core

8. **Andrew Lock's Blog (dotnetcore tutorials)**
   - https://andrewlock.net/
   - Tutoriales profundos sobre ASP.NET Core

9. **Code Maze**
   - https://code-maze.com/
   - Tutoriales detallados de .NET y C#

10. **Rick Strahl's Web Log**
    - https://weblog.west-wind.com/
    - Experiencia real con ASP.NET Core

### Videos y Canales

11. **dotnet YouTube Channel**
    - https://www.youtube.com/@dotnet
    - Videos oficiales de Microsoft

12. **Nick Chapsas**
    - https://www.youtube.com/@nickchapsas
    - Contenido avanzado sobre .NET y C#

13. **IAmTimCorey**
    - https://www.youtube.com/@IAmTimCorey
    - Tutoriales completos y bien explicados

### Herramientas y Utilidades

14. **NuGet Package Explorer**
    - https://www.nuget.org/
    - Repositorio oficial de paquetes .NET

15. **Swashbuckle (Swagger para ASP.NET Core)**
    - https://github.com/domaindrivendev/Swashbuckle.AspNetCore
    - Documentación de APIs

16. **BenchmarkDotNet**
    - https://benchmarkdotnet.org/
    - Framework para benchmarking de código .NET

### Libros Recomendados

17. **"ASP.NET Core in Action" - Andrew Lock**
    - Manning Publications
    - Cobertura completa de ASP.NET Core

18. **"C# 12 and .NET 8 - Modern Cross-Platform Development" - Mark J. Price**
    - Packt Publishing
    - Actualizado a las últimas versiones

19. **"Pro ASP.NET Core 7" - Adam Freeman**
    - Apress
    - Referencia completa y detallada

### Comunidades

20. **Stack Overflow - ASP.NET Core Tag**
    - https://stackoverflow.com/questions/tagged/asp.net-core
    - Preguntas y respuestas de la comunidad

21. **Reddit - r/dotnet**
    - https://www.reddit.com/r/dotnet/
    - Comunidad activa de desarrolladores .NET

22. **Discord - .NET Community**
    - https://discord.gg/dotnet
    - Chat en tiempo real con otros desarrolladores

---

## Conclusión

ASP.NET Core es un framework moderno, potente y versátil que permite construir aplicaciones web de alta calidad. Su naturaleza multiplataforma, alto rendimiento y ecosistema rico lo convierten en una excelente elección para proyectos de cualquier escala.

**Próximos pasos recomendados:**
1. Practica creando pequeñas APIs siguiendo los ejemplos
2. Explora Entity Framework Core para acceso a datos
3. Aprende sobre autenticación y autorización con Identity
4. Investiga patrones de arquitectura como Clean Architecture
5. Experimenta con testing (xUnit, NUnit)
6. Despliega tu primera aplicación en la nube (Azure, AWS, o similar)

¡Bienvenido al mundo de ASP.NET Core!
