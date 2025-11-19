# Laboratorio: Health Checks y Monitorización en .NET Core 8

## Índice
1. [Introducción](#introducción)
2. [Teoría: Health Checks](#teoría-health-checks)
3. [Teoría: Monitorización de Aplicaciones](#teoría-monitorización-de-aplicaciones)
4. [¿Por qué en Microservicios?](#por-qué-en-microservicios)
5. [Ejercicio 1: Health Check Básico](#ejercicio-1-health-check-básico)
6. [Ejercicio 2: Health Checks Personalizados](#ejercicio-2-health-checks-personalizados)
7. [Ejercicio 3: UI de Health Checks](#ejercicio-3-ui-de-health-checks)
8. [Ejercicio 4: Integración con Prometheus y Grafana](#ejercicio-4-integración-con-prometheus-y-grafana)
9. [Ejercicio 5: Logging Estructurado con Serilog](#ejercicio-5-logging-estructurado-con-serilog)
10. [Ejercicio 6: Application Insights](#ejercicio-6-application-insights)
11. [Conclusiones](#conclusiones)

---

## Introducción

Este laboratorio cubre dos aspectos fundamentales en arquitecturas distribuidas y microservicios: **Health Checks** y **Monitorización de Aplicaciones**. Aprenderás a implementar mecanismos que permitan verificar el estado de tus servicios y obtener métricas valiosas para la operación y el diagnóstico.

**Duración estimada:** 3-4 horas  
**Nivel:** Intermedio  
**Prerequisitos:** 
- .NET Core 8 SDK instalado
- Visual Studio 2022 / VS Code / Rider
- Docker Desktop (opcional para ejercicios avanzados)
- Conocimientos básicos de ASP.NET Core

---

## Teoría: Health Checks

### ¿Qué son los Health Checks?

Los Health Checks son endpoints especiales en tu aplicación que proporcionan información sobre el estado de salud del servicio y sus dependencias. Responden preguntas como:

- ¿Está la aplicación funcionando correctamente?
- ¿Puede conectarse a la base de datos?
- ¿Están disponibles los servicios externos?
- ¿Hay suficiente memoria/CPU disponible?

### Tipos de Health Checks

1. **Liveness Probe**: Verifica si la aplicación está "viva" y puede procesar solicitudes
2. **Readiness Probe**: Verifica si la aplicación está lista para recibir tráfico
3. **Startup Probe**: Verifica si la aplicación ha terminado de inicializarse

### Estados de Health Check

- **Healthy**: El componente funciona correctamente
- **Degraded**: El componente funciona pero con problemas
- **Unhealthy**: El componente no está funcionando

### Componentes en .NET Core 8

.NET Core proporciona el paquete `Microsoft.Extensions.Diagnostics.HealthChecks` que incluye:

- `IHealthCheck`: Interface para implementar health checks personalizados
- `HealthCheckService`: Servicio para ejecutar health checks
- Middleware de health checks
- Publishers para enviar resultados a sistemas externos

---

## Teoría: Monitorización de Aplicaciones

### ¿Qué es la Monitorización?

La monitorización es el proceso continuo de recopilación, análisis y visualización de métricas y logs de una aplicación para:

- Detectar problemas antes de que afecten a usuarios
- Entender el comportamiento de la aplicación
- Optimizar el rendimiento
- Facilitar el diagnóstico de errores
- Tomar decisiones informadas

### Pilares de la Observabilidad

La observabilidad se basa en tres pilares:

1. **Logs**: Registros de eventos discretos que ocurren en el sistema
2. **Métricas**: Valores numéricos medidos a lo largo del tiempo (CPU, memoria, peticiones/segundo)
3. **Traces**: Seguimiento de solicitudes a través de múltiples servicios

### Herramientas Comunes

- **Prometheus**: Sistema de monitorización y base de datos de series temporales
- **Grafana**: Plataforma de visualización y análisis
- **ELK Stack** (Elasticsearch, Logstash, Kibana): Para gestión de logs
- **Application Insights**: Solución APM de Azure
- **Serilog**: Framework de logging estructurado

---

## ¿Por qué en Microservicios?

### Complejidad Distribuida

En una arquitectura de microservicios, una solicitud puede atravesar múltiples servicios. Esto introduce:

- **Puntos de fallo múltiples**: Cualquier servicio puede fallar
- **Latencia acumulada**: Los problemas de rendimiento se amplifican
- **Dependencias en cascada**: El fallo de un servicio puede afectar a otros

### Necesidad de Health Checks

Los health checks son críticos porque:

1. **Orquestadores (Kubernetes, Docker Swarm)** los usan para:
   - Decidir si reiniciar un contenedor
   - Determinar si enviar tráfico a una instancia
   - Gestionar el escalado automático

2. **Load Balancers** los utilizan para:
   - Remover instancias no saludables del pool
   - Distribuir tráfico solo a instancias saludables

3. **Circuit Breakers** los aprovechan para:
   - Evitar llamadas a servicios caídos
   - Implementar patrones de resiliencia

### Necesidad de Monitorización

La monitorización es esencial porque:

1. **Visibilidad**: Con decenas o cientos de servicios, necesitas ver qué está pasando
2. **Correlación**: Debes relacionar eventos en diferentes servicios
3. **Alertas proactivas**: Detectar problemas antes de que los usuarios los reporten
4. **Análisis de tendencias**: Identificar patrones y predecir problemas
5. **Debugging distribuido**: Rastrear una solicitud a través de múltiples servicios
6. **SLAs y SLOs**: Medir y garantizar niveles de servicio

### Ejemplo del Mundo Real

Imagina un e-commerce con microservicios:

```
Cliente → API Gateway → Catálogo → Inventario → Base de Datos
                     ↓
                   Pedidos → Pagos → Servicio Externo
                     ↓
                  Notificaciones
```

**Sin Health Checks y Monitorización:**
- Un problema en la base de datos puede hacer que todos los servicios fallen en cascada
- No sabrías cuál servicio está causando lentitud
- Los usuarios reportarían errores antes que tu equipo los detecte

**Con Health Checks y Monitorización:**
- Kubernetes reinicia automáticamente servicios no saludables
- Dashboards muestran qué servicio tiene alta latencia
- Alertas notifican al equipo antes de que afecte a usuarios
- Traces muestran exactamente dónde falla una transacción

---

## Ejercicio 1: Health Check Básico

### Objetivo
Crear una API básica con health checks predeterminados.

### Paso 1: Crear el Proyecto

```bash
dotnet new webapi -n HealthCheckDemo
cd HealthCheckDemo
dotnet add package Microsoft.AspNetCore.Diagnostics.HealthChecks
```

### Paso 2: Configurar Health Checks

Modifica `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Agregar Health Checks
builder.Services.AddHealthChecks();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();

// Mapear endpoints de health checks
app.MapHealthChecks("/health");

app.MapControllers();

app.Run();
```

### Paso 3: Probar

```bash
dotnet run
```

Accede a: `https://localhost:7xxx/health`

Deberías ver: `Healthy`

### Paso 4: Health Check con Detalles

Modifica el mapeo para incluir detalles:

```csharp
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using System.Text.Json;

// ...

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration.TotalMilliseconds
            }),
            totalDuration = report.TotalDuration.TotalMilliseconds
        });
        
        await context.Response.WriteAsync(result);
    }
});
```

Ahora al acceder a `/health` verás una respuesta JSON detallada.

---

## Ejercicio 2: Health Checks Personalizados

### Objetivo
Crear health checks personalizados para validar dependencias específicas.

### Paso 1: Simular Dependencias

Crea una carpeta `Services` y agrega `DatabaseService.cs`:

```csharp
namespace HealthCheckDemo.Services;

public interface IDatabaseService
{
    Task<bool> IsAvailableAsync();
}

public class DatabaseService : IDatabaseService
{
    private readonly ILogger<DatabaseService> _logger;
    private bool _isHealthy = true;

    public DatabaseService(ILogger<DatabaseService> logger)
    {
        _logger = logger;
    }

    public async Task<bool> IsAvailableAsync()
    {
        // Simular chequeo de base de datos
        await Task.Delay(100);
        return _isHealthy;
    }

    // Método para simular fallo
    public void SimulateFailure() => _isHealthy = false;
    public void RestoreHealth() => _isHealthy = true;
}
```

### Paso 2: Crear Health Check Personalizado

Crea carpeta `HealthChecks` y agrega `DatabaseHealthCheck.cs`:

```csharp
using Microsoft.Extensions.Diagnostics.HealthChecks;
using HealthCheckDemo.Services;

namespace HealthCheckDemo.HealthChecks;

public class DatabaseHealthCheck : IHealthCheck
{
    private readonly IDatabaseService _databaseService;
    private readonly ILogger<DatabaseHealthCheck> _logger;

    public DatabaseHealthCheck(
        IDatabaseService databaseService,
        ILogger<DatabaseHealthCheck> logger)
    {
        _databaseService = databaseService;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var isAvailable = await _databaseService.IsAvailableAsync();

            if (isAvailable)
            {
                return HealthCheckResult.Healthy(
                    "Database is available and responding");
            }

            return HealthCheckResult.Unhealthy(
                "Database is not responding");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error checking database health");
            return HealthCheckResult.Unhealthy(
                "Error checking database health",
                ex);
        }
    }
}
```

### Paso 3: Crear Health Check para Servicio Externo

Agrega `ExternalApiHealthCheck.cs`:

```csharp
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace HealthCheckDemo.HealthChecks;

public class ExternalApiHealthCheck : IHealthCheck
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IConfiguration _configuration;
    private readonly ILogger<ExternalApiHealthCheck> _logger;

    public ExternalApiHealthCheck(
        IHttpClientFactory httpClientFactory,
        IConfiguration configuration,
        ILogger<ExternalApiHealthCheck> logger)
    {
        _httpClientFactory = httpClientFactory;
        _configuration = configuration;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var client = _httpClientFactory.CreateClient();
            var url = _configuration["ExternalApi:HealthCheckUrl"] 
                ?? "https://jsonplaceholder.typicode.com/posts/1";

            var response = await client.GetAsync(url, cancellationToken);

            if (response.IsSuccessStatusCode)
            {
                return HealthCheckResult.Healthy(
                    $"External API is reachable. Status: {response.StatusCode}");
            }

            return HealthCheckResult.Degraded(
                $"External API returned {response.StatusCode}");
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "External API health check failed");
            return HealthCheckResult.Unhealthy(
                "External API is unreachable",
                ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error in health check");
            return HealthCheckResult.Unhealthy(
                "Health check failed with unexpected error",
                ex);
        }
    }
}
```

### Paso 4: Registrar Servicios y Health Checks

Modifica `Program.cs`:

```csharp
using HealthCheckDemo.HealthChecks;
using HealthCheckDemo.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddHttpClient();

// Registrar servicios
builder.Services.AddSingleton<IDatabaseService, DatabaseService>();

// Registrar Health Checks
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>(
        "database",
        tags: new[] { "db", "sql" })
    .AddCheck<ExternalApiHealthCheck>(
        "external_api",
        tags: new[] { "api", "external" });

var app = builder.Build();

// ... resto del código
```

### Paso 5: Crear Endpoints con Tags

```csharp
// Health check general
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration.TotalMilliseconds,
                tags = e.Value.Tags
            }),
            totalDuration = report.TotalDuration.TotalMilliseconds
        }, new JsonSerializerOptions { WriteIndented = true });
        await context.Response.WriteAsync(result);
    }
});

// Health check solo para base de datos
app.MapHealthChecks("/health/db", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("db"),
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description
            })
        });
        await context.Response.WriteAsync(result);
    }
});

// Liveness - solo verifica si la app está viva
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // No ejecuta ningún health check, solo responde
});

// Readiness - verifica si está lista para recibir tráfico
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

### Paso 6: Controlador para Simular Fallos

Crea `Controllers/HealthTestController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using HealthCheckDemo.Services;

namespace HealthCheckDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
public class HealthTestController : ControllerBase
{
    private readonly DatabaseService _databaseService;

    public HealthTestController(IDatabaseService databaseService)
    {
        _databaseService = (DatabaseService)databaseService;
    }

    [HttpPost("simulate-db-failure")]
    public IActionResult SimulateDatabaseFailure()
    {
        _databaseService.SimulateFailure();
        return Ok(new { message = "Database failure simulated" });
    }

    [HttpPost("restore-db-health")]
    public IActionResult RestoreDatabaseHealth()
    {
        _databaseService.RestoreHealth();
        return Ok(new { message = "Database health restored" });
    }
}
```

### Paso 7: Probar

```bash
dotnet run
```

Prueba los diferentes endpoints:
- `GET /health` - Todos los health checks
- `GET /health/db` - Solo base de datos
- `GET /health/live` - Liveness probe
- `POST /api/healthtest/simulate-db-failure` - Simular fallo
- `GET /health` - Ver estado degradado
- `POST /api/healthtest/restore-db-health` - Restaurar

---

## Ejercicio 3: UI de Health Checks

### Objetivo
Agregar una interfaz visual para monitorear health checks.

### Paso 1: Instalar Paquete

```bash
dotnet add package AspNetCore.HealthChecks.UI
dotnet add package AspNetCore.HealthChecks.UI.Client
dotnet add package AspNetCore.HealthChecks.UI.InMemory.Storage
```

### Paso 2: Configurar Health Checks UI

Modifica `Program.cs`:

```csharp
using HealthChecks.UI.Client;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;

// ... código anterior ...

builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database", tags: new[] { "db", "sql", "ready" })
    .AddCheck<ExternalApiHealthCheck>("external_api", tags: new[] { "api", "external", "ready" });

// Agregar Health Checks UI
builder.Services
    .AddHealthChecksUI(setup =>
    {
        setup.SetEvaluationTimeInSeconds(10); // Evaluar cada 10 segundos
        setup.MaximumHistoryEntriesPerEndpoint(50);
        setup.AddHealthCheckEndpoint("API Health", "/health");
    })
    .AddInMemoryStorage();

var app = builder.Build();

// ... código anterior ...

// Endpoint para Health Checks UI
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// UI de Health Checks
app.MapHealthChecksUI(setup =>
{
    setup.UIPath = "/health-ui";
    setup.ApiPath = "/health-ui-api";
});
```

### Paso 3: Configurar appsettings.json

Agrega configuración de Health Checks UI:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "HealthChecks": "Information"
    }
  },
  "AllowedHosts": "*",
  "HealthChecksUI": {
    "HealthChecks": [
      {
        "Name": "API Health",
        "Uri": "https://localhost:7xxx/health"
      }
    ],
    "EvaluationTimeInSeconds": 10,
    "MinimumSecondsBetweenFailureNotifications": 60
  }
}
```

### Paso 4: Probar UI

```bash
dotnet run
```

Accede a: `https://localhost:7xxx/health-ui`

Verás una interfaz gráfica mostrando el estado de todos tus health checks con:
- Estado actual (Healthy, Degraded, Unhealthy)
- Historial de estados
- Tiempo de respuesta
- Detalles de cada check

---

## Ejercicio 4: Integración con Prometheus y Grafana

### Objetivo
Exponer métricas de la aplicación para Prometheus y visualizarlas en Grafana.

### Paso 1: Instalar Paquetes

```bash
dotnet add package prometheus-net.AspNetCore
dotnet add package prometheus-net.AspNetCore.HealthChecks
```

### Paso 2: Configurar Prometheus

Modifica `Program.cs`:

```csharp
using Prometheus;

// ... código anterior ...

var app = builder.Build();

// ... código anterior ...

// Middleware de Prometheus
app.UseRouting();
app.UseHttpMetrics(); // Métricas HTTP automáticas

app.UseAuthorization();

// Endpoint de métricas
app.MapMetrics();

// Health Checks como métricas de Prometheus
app.UseHealthChecksPrometheusExporter("/metrics-healthchecks");

app.MapControllers();
```

### Paso 3: Crear Métricas Personalizadas

Crea `Services/MetricsService.cs`:

```csharp
using Prometheus;

namespace HealthCheckDemo.Services;

public class MetricsService
{
    private readonly Counter _requestCounter;
    private readonly Histogram _requestDuration;
    private readonly Gauge _activeRequests;
    private readonly Counter _errorCounter;

    public MetricsService()
    {
        _requestCounter = Metrics.CreateCounter(
            "api_requests_total",
            "Total number of API requests",
            new CounterConfiguration
            {
                LabelNames = new[] { "method", "endpoint", "status" }
            });

        _requestDuration = Metrics.CreateHistogram(
            "api_request_duration_seconds",
            "Duration of API requests in seconds",
            new HistogramConfiguration
            {
                LabelNames = new[] { "method", "endpoint" },
                Buckets = Histogram.ExponentialBuckets(0.001, 2, 10)
            });

        _activeRequests = Metrics.CreateGauge(
            "api_requests_active",
            "Number of active API requests",
            new GaugeConfiguration
            {
                LabelNames = new[] { "method", "endpoint" }
            });

        _errorCounter = Metrics.CreateCounter(
            "api_errors_total",
            "Total number of API errors",
            new CounterConfiguration
            {
                LabelNames = new[] { "method", "endpoint", "error_type" }
            });
    }

    public void RecordRequest(string method, string endpoint, int statusCode)
    {
        _requestCounter.WithLabels(method, endpoint, statusCode.ToString()).Inc();
    }

    public IDisposable TrackRequestDuration(string method, string endpoint)
    {
        _activeRequests.WithLabels(method, endpoint).Inc();
        return _requestDuration.WithLabels(method, endpoint).NewTimer();
    }

    public void RecordError(string method, string endpoint, string errorType)
    {
        _errorCounter.WithLabels(method, endpoint, errorType).Inc();
    }

    public void DecrementActiveRequests(string method, string endpoint)
    {
        _activeRequests.WithLabels(method, endpoint).Dec();
    }
}
```

### Paso 4: Middleware de Métricas

Crea `Middleware/MetricsMiddleware.cs`:

```csharp
using HealthCheckDemo.Services;

namespace HealthCheckDemo.Middleware;

public class MetricsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly MetricsService _metricsService;

    public MetricsMiddleware(RequestDelegate next, MetricsService metricsService)
    {
        _next = next;
        _metricsService = metricsService;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var method = context.Request.Method;
        var endpoint = context.Request.Path.Value ?? "unknown";

        using var timer = _metricsService.TrackRequestDuration(method, endpoint);

        try
        {
            await _next(context);
            _metricsService.RecordRequest(method, endpoint, context.Response.StatusCode);
        }
        catch (Exception ex)
        {
            _metricsService.RecordError(method, endpoint, ex.GetType().Name);
            throw;
        }
        finally
        {
            _metricsService.DecrementActiveRequests(method, endpoint);
        }
    }
}
```

### Paso 5: Registrar Servicios y Middleware

```csharp
builder.Services.AddSingleton<MetricsService>();

// ...

var app = builder.Build();

// ...

app.UseMiddleware<MetricsMiddleware>();
```

### Paso 6: Crear docker-compose para Prometheus y Grafana

Crea `docker-compose.yml` en la raíz del proyecto:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_SECURITY_ADMIN_USER=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - monitoring
    depends_on:
      - prometheus

volumes:
  prometheus-data:
  grafana-data:

networks:
  monitoring:
    driver: bridge
```

### Paso 7: Configurar Prometheus

Crea `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'healthcheck-api'
    static_configs:
      - targets: ['host.docker.internal:5000'] # Ajusta el puerto
    metrics_path: '/metrics'
    scrape_interval: 5s
```

### Paso 8: Configurar Datasource de Grafana

Crea carpeta `grafana/datasources` y archivo `prometheus.yml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

### Paso 9: Ejecutar

```bash
# Terminal 1 - API
dotnet run

# Terminal 2 - Prometheus y Grafana
docker-compose up -d
```

Accede a:
- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000` (admin/admin)
- Métricas de la API: `https://localhost:7xxx/metrics`

### Paso 10: Crear Dashboard en Grafana

1. Accede a Grafana (localhost:3000)
2. Ve a Dashboards → New Dashboard
3. Add visualization
4. Selecciona Prometheus como datasource
5. Agrega queries:
   - `rate(api_requests_total[5m])` - Requests por segundo
   - `api_requests_active` - Requests activos
   - `rate(api_errors_total[5m])` - Errores por segundo
   - `histogram_quantile(0.95, rate(api_request_duration_seconds_bucket[5m]))` - P95 latencia

---

## Ejercicio 5: Logging Estructurado con Serilog

### Objetivo
Implementar logging estructurado con Serilog para mejor análisis y búsqueda de logs.

### Paso 1: Instalar Paquetes

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Seq
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
dotnet add package Serilog.Enrichers.Process
```

### Paso 2: Configurar Serilog

Modifica `Program.cs`:

```csharp
using Serilog;
using Serilog.Events;

// Configurar Serilog antes de crear el builder
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.Hosting.Lifetime", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .Enrich.WithProcessId()
    .WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.File(
        path: "logs/app-.log",
        rollingInterval: RollingInterval.Day,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}",
        retainedFileCountLimit: 7)
    .WriteTo.Seq("http://localhost:5341") // Opcional: servidor Seq
    .CreateLogger();

try
{
    Log.Information("Starting web application");

    var builder = WebApplication.CreateBuilder(args);

    // Agregar Serilog
    builder.Host.UseSerilog();

    // ... resto del código ...

    var app = builder.Build();

    // Request logging de Serilog
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("RequestScheme", httpContext.Request.Scheme);
            diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
            diagnosticContext.Set("RemoteIP", httpContext.Connection.RemoteIpAddress?.ToString());
        };
    });

    // ... resto del código ...

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

### Paso 3: Crear Servicio con Logging

Crea `Services/OrderService.cs`:

```csharp
namespace HealthCheckDemo.Services;

public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public async Task<string> ProcessOrderAsync(int orderId, string customerName, decimal amount)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = orderId,
            ["CustomerName"] = customerName,
            ["Amount"] = amount
        }))
        {
            _logger.LogInformation("Starting order processing");

            try
            {
                // Simular procesamiento
                await Task.Delay(Random.Shared.Next(100, 500));

                if (amount > 10000)
                {
                    _logger.LogWarning("High value order detected: {Amount}", amount);
                }

                // Simular error aleatorio
                if (Random.Shared.Next(0, 10) < 2)
                {
                    throw new InvalidOperationException("Payment gateway timeout");
                }

                _logger.LogInformation(
                    "Order processed successfully. OrderId: {OrderId}, Amount: {Amount:C}",
                    orderId,
                    amount);

                return $"Order-{orderId}";
            }
            catch (Exception ex)
            {
                _logger.LogError(
                    ex,
                    "Failed to process order. OrderId: {OrderId}",
                    orderId);
                throw;
            }
        }
    }
}
```

### Paso 4: Crear Controlador

Crea `Controllers/OrdersController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using HealthCheckDemo.Services;

namespace HealthCheckDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly OrderService _orderService;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(OrderService orderService, ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        _logger.LogInformation(
            "Received order creation request for customer: {CustomerName}",
            request.CustomerName);

        try
        {
            var orderId = Random.Shared.Next(1000, 9999);
            var result = await _orderService.ProcessOrderAsync(
                orderId,
                request.CustomerName,
                request.Amount);

            return Ok(new { orderId = result, status = "processed" });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating order");
            return StatusCode(500, new { error = "Failed to process order" });
        }
    }
}

public record CreateOrderRequest(string CustomerName, decimal Amount);
```

### Paso 5: Registrar Servicio

```csharp
builder.Services.AddScoped<OrderService>();
```

### Paso 6: Configurar Seq (Opcional)

Crea `docker-compose-seq.yml`:

```yaml
version: '3.8'

services:
  seq:
    image: datalust/seq:latest
    container_name: seq
    ports:
      - "5341:80"
    environment:
      - ACCEPT_EULA=Y
    volumes:
      - seq-data:/data

volumes:
  seq-data:
```

Ejecutar:

```bash
docker-compose -f docker-compose-seq.yml up -d
```

Accede a Seq: `http://localhost:5341`

### Paso 7: Probar Logging

```bash
dotnet run
```

Hacer varias peticiones:

```bash
curl -X POST https://localhost:7xxx/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customerName":"Juan Pérez","amount":150.50}'
```

Verás logs estructurados en:
- Consola
- Archivo `logs/app-yyyyMMdd.log`
- Seq (si está configurado)

---

## Ejercicio 6: Application Insights

### Objetivo
Integrar Azure Application Insights para monitorización APM completa.

### Paso 1: Instalar Paquetes

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
dotnet add package Microsoft.ApplicationInsights.SnapshotCollector
```

### Paso 2: Configurar Application Insights

Modifica `Program.cs`:

```csharp
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
    options.EnableAdaptiveSampling = true;
    options.EnableQuickPulseMetricStream = true;
});
```

### Paso 3: Agregar Configuración

En `appsettings.json`:

```json
{
  "ApplicationInsights": {
    "ConnectionString": "InstrumentationKey=your-key-here;IngestionEndpoint=https://..."
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    },
    "ApplicationInsights": {
      "LogLevel": {
        "Default": "Information",
        "Microsoft": "Warning"
      }
    }
  }
}
```

### Paso 4: Telemetría Personalizada

Crea `Services/TelemetryService.cs`:

```csharp
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.DataContracts;

namespace HealthCheckDemo.Services;

public class TelemetryService
{
    private readonly TelemetryClient _telemetryClient;

    public TelemetryService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public void TrackOrderCreated(int orderId, decimal amount, string customerName)
    {
        var properties = new Dictionary<string, string>
        {
            { "OrderId", orderId.ToString() },
            { "CustomerName", customerName }
        };

        var metrics = new Dictionary<string, double>
        {
            { "Amount", (double)amount }
        };

        _telemetryClient.TrackEvent("OrderCreated", properties, metrics);
    }

    public void TrackOrderProcessingDuration(int orderId, TimeSpan duration)
    {
        var properties = new Dictionary<string, string>
        {
            { "OrderId", orderId.ToString() }
        };

        _telemetryClient.TrackMetric(
            "OrderProcessingDuration",
            duration.TotalMilliseconds,
            properties);
    }

    public void TrackDependency(
        string dependencyName,
        string commandName,
        DateTimeOffset startTime,
        TimeSpan duration,
        bool success)
    {
        _telemetryClient.TrackDependency(
            dependencyName,
            commandName,
            startTime,
            duration,
            success);
    }

    public IOperationHolder<RequestTelemetry> StartOperation(string operationName)
    {
        return _telemetryClient.StartOperation<RequestTelemetry>(operationName);
    }
}
```

### Paso 5: Usar Telemetría

Modifica `OrderService.cs`:

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;
    private readonly TelemetryService _telemetryService;

    public OrderService(
        ILogger<OrderService> logger,
        TelemetryService telemetryService)
    {
        _logger = logger;
        _telemetryService = telemetryService;
    }

    public async Task<string> ProcessOrderAsync(
        int orderId,
        string customerName,
        decimal amount)
    {
        var startTime = DateTimeOffset.UtcNow;

        using (var operation = _telemetryService.StartOperation($"ProcessOrder-{orderId}"))
        {
            try
            {
                _logger.LogInformation("Starting order processing");

                // Simular llamada a base de datos
                var dbStartTime = DateTimeOffset.UtcNow;
                await Task.Delay(Random.Shared.Next(50, 150));
                _telemetryService.TrackDependency(
                    "SQL Database",
                    "SaveOrder",
                    dbStartTime,
                    DateTimeOffset.UtcNow - dbStartTime,
                    true);

                // Simular llamada a servicio externo
                var apiStartTime = DateTimeOffset.UtcNow;
                await Task.Delay(Random.Shared.Next(100, 300));
                _telemetryService.TrackDependency(
                    "Payment API",
                    "ProcessPayment",
                    apiStartTime,
                    DateTimeOffset.UtcNow - apiStartTime,
                    true);

                var duration = DateTimeOffset.UtcNow - startTime;
                _telemetryService.TrackOrderProcessingDuration(orderId, duration);
                _telemetryService.TrackOrderCreated(orderId, amount, customerName);

                operation.Telemetry.Success = true;
                return $"Order-{orderId}";
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process order");
                operation.Telemetry.Success = false;
                throw;
            }
        }
    }
}
```

### Paso 6: Registrar Servicio

```csharp
builder.Services.AddSingleton<TelemetryService>();
```

### Paso 7: Simulación Local (Sin Azure)

Para desarrollo local sin Application Insights real, crea un mock:

```csharp
public class MockTelemetryService : TelemetryService
{
    private readonly ILogger<MockTelemetryService> _logger;

    public MockTelemetryService(
        TelemetryClient telemetryClient,
        ILogger<MockTelemetryService> logger)
        : base(telemetryClient)
    {
        _logger = logger;
    }

    public new void TrackOrderCreated(int orderId, decimal amount, string customerName)
    {
        _logger.LogInformation(
            "[TELEMETRY] Order created: OrderId={OrderId}, Amount={Amount}, Customer={Customer}",
            orderId, amount, customerName);
    }

    // Override otros métodos...
}
```

Registrar según el entorno:

```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddSingleton<TelemetryService, MockTelemetryService>();
}
else
{
    builder.Services.AddSingleton<TelemetryService>();
}
```

---

## Conclusiones

### Resumen de Conceptos

Has aprendido a implementar:

1. **Health Checks Básicos**: Verificación del estado de la aplicación
2. **Health Checks Personalizados**: Validación de dependencias específicas
3. **Health Checks UI**: Visualización gráfica del estado
4. **Prometheus y Grafana**: Métricas y dashboards
5. **Serilog**: Logging estructurado y búsqueda eficiente
6. **Application Insights**: APM y telemetría completa

### Mejores Prácticas

#### Health Checks
- Implementa health checks para todas las dependencias críticas
- Usa diferentes endpoints para liveness, readiness y startup
- No incluyas lógica pesada en los health checks
- Establece timeouts apropiados
- Usa tags para categorizar checks

#### Monitorización
- Implementa los tres pilares: logs, métricas y traces
- Usa logging estructurado para facilitar búsquedas
- Define métricas de negocio además de técnicas
- Establece alertas proactivas
- Crea dashboards para diferentes audiencias

#### En Producción
- Configura retención de logs apropiada
- Implementa sampling para reducir costos
- Usa correlation IDs para rastrear requests
- Protege endpoints de monitorización
- Documenta tus métricas y alertas

### Arquitectura de Microservicios

En una arquitectura de microservicios, la estrategia completa incluye:

1. **Service Mesh** (Istio, Linkerd): Telemetría a nivel de red
2. **Distributed Tracing** (Jaeger, Zipkin): Seguimiento de transacciones
3. **Centralized Logging** (ELK, Loki): Agregación de logs
4. **Metrics Aggregation** (Prometheus): Métricas centralizadas
5. **APM** (Application Insights, Datadog): Monitorización de aplicaciones
6. **Alerting** (AlertManager, PagerDuty): Sistema de alertas

### Próximos Pasos

Para profundizar:

1. Implementa **OpenTelemetry** para observabilidad estandarizada
2. Configura **Distributed Tracing** con Jaeger
3. Implementa **Circuit Breakers** con Polly
4. Explora **Service Mesh** con Istio
5. Configura **Kubernetes Probes** con tus health checks
6. Implementa **Custom Metrics** de negocio
7. Crea **Runbooks** basados en alertas

### Recursos Adicionales

- [Microsoft Docs - Health Checks](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)
- [Prometheus - Best Practices](https://prometheus.io/docs/practices/naming/)
- [Serilog Documentation](https://serilog.net/)
- [Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [12 Factor App - Logs](https://12factor.net/logs)

---

## Ejercicio Final (Opcional)

Crea una aplicación completa de e-commerce con:

1. **Servicios**: Catálogo, Carrito, Pedidos, Pagos
2. **Health Checks**: Para cada dependencia (DB, Redis, APIs externas)
3. **Métricas**: Pedidos procesados, ingresos, latencia por servicio
4. **Logging**: Trazabilidad completa de transacciones
5. **Dashboards**: Grafana con métricas de negocio y técnicas
6. **Alertas**: Configurar alertas para SLOs críticos

Este ejercicio te preparará para implementar observabilidad en sistemas de producción reales.

---

