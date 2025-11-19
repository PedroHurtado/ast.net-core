# Laboratorio: Autenticación JWT en .NET Core 8

## Índice
1. [Introducción](#introducción)
2. [Teoría: JWT y Autenticación](#teoría-jwt-y-autenticación)
3. [Configuración del Proyecto](#configuración-del-proyecto)
4. [Ejercicio 1: Configuración Básica de JWT](#ejercicio-1-configuración-básica-de-jwt)
5. [Ejercicio 2: Generar y Validar Tokens](#ejercicio-2-generar-y-validar-tokens)
6. [Ejercicio 3: Protección Global de Endpoints](#ejercicio-3-protección-global-de-endpoints)
7. [Ejercicio 4: Refresh Tokens](#ejercicio-4-refresh-tokens)
8. [Ejercicio 5: Roles y Claims](#ejercicio-5-roles-y-claims)
9. [Conclusiones y Mejores Prácticas](#conclusiones-y-mejores-prácticas)

---

## Introducción

Este laboratorio te enseñará a implementar autenticación JWT (JSON Web Tokens) en .NET Core 8 con un enfoque similar a Spring Security, donde **todas las rutas están protegidas por defecto** excepto aquellas que explícitamente excluyas (como Swagger).

**Duración estimada:** 2-3 horas  
**Nivel:** Intermedio  
**Prerequisitos:**
- .NET Core 8 SDK instalado
- Visual Studio 2022 / VS Code / Rider
- Conocimientos básicos de ASP.NET Core
- Postman o similar para probar APIs

---

## Teoría: JWT y Autenticación

### ¿Qué es JWT?

JWT (JSON Web Token) es un estándar abierto (RFC 7519) que define un formato compacto y autónomo para transmitir información de forma segura entre partes como un objeto JSON.

### Estructura de un JWT

Un JWT consta de tres partes separadas por puntos (.):

```
header.payload.signature
```

**Ejemplo:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

#### 1. Header (Encabezado)
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
Especifica el algoritmo de firma (HS256, RS256, etc.) y el tipo de token.

#### 2. Payload (Carga útil)
```json
{
  "sub": "user@example.com",
  "name": "Juan Pérez",
  "role": "Admin",
  "exp": 1735689600,
  "iat": 1735603200
}
```
Contiene los claims (declaraciones) sobre el usuario y metadata adicional.

#### 3. Signature (Firma)
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```
Garantiza que el token no ha sido alterado.

### Claims Estándar

- **sub** (subject): Identificador del usuario
- **iss** (issuer): Emisor del token
- **aud** (audience): Destinatario del token
- **exp** (expiration): Fecha de expiración
- **iat** (issued at): Fecha de emisión
- **nbf** (not before): Fecha antes de la cual el token no es válido
- **jti** (JWT ID): Identificador único del token

### ¿Por qué JWT?

**Ventajas:**
- **Stateless**: No requiere almacenamiento en servidor
- **Escalable**: Ideal para microservicios y aplicaciones distribuidas
- **Portable**: Funciona en diferentes dominios y plataformas
- **Autocontenido**: Toda la información necesaria está en el token

**Desventajas:**
- **No revocable**: Una vez emitido, es válido hasta su expiración (solución: refresh tokens)
- **Tamaño**: Más grande que un ID de sesión
- **Seguridad**: Debe transmitirse siempre por HTTPS

### Flujo de Autenticación JWT

```
1. Cliente → POST /api/auth/login (usuario + contraseña)
2. Servidor → Valida credenciales → Genera JWT
3. Servidor → Responde con JWT
4. Cliente → Almacena JWT (localStorage, sessionStorage, memoria)
5. Cliente → GET /api/productos (Header: Authorization: Bearer <token>)
6. Servidor → Valida JWT → Responde con datos
```

### Comportamiento Estilo Spring Security

En Spring Security, la configuración por defecto protege **todos los endpoints** y debes especificar explícitamente cuáles son públicos:

```java
http
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/auth/**", "/swagger-ui/**").permitAll()
        .anyRequest().authenticated()
    )
```

En este laboratorio replicaremos este comportamiento en .NET Core 8.

---

## Configuración del Proyecto

### Paso 1: Crear el Proyecto

```bash
dotnet new webapi -n JwtAuthDemo
cd JwtAuthDemo
```

### Paso 2: Instalar Paquetes NuGet

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
dotnet add package BCrypt.Net-Next
```

### Paso 3: Estructura del Proyecto

```
JwtAuthDemo/
├── Controllers/
│   ├── AuthController.cs
│   ├── ProductsController.cs
│   └── UsersController.cs
├── Models/
│   ├── LoginRequest.cs
│   ├── LoginResponse.cs
│   ├── RegisterRequest.cs
│   └── User.cs
├── Services/
│   ├── IAuthService.cs
│   ├── AuthService.cs
│   └── JwtService.cs
├── Configuration/
│   └── JwtSettings.cs
├── Program.cs
└── appsettings.json
```

---

## Ejercicio 1: Configuración Básica de JWT

### Objetivo
Configurar la infraestructura básica para JWT, incluyendo settings y servicios.

### Paso 1: Configurar appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "JwtSettings": {
    "SecretKey": "MiClaveSecretaSuperSeguraYLargaDeAlMenos32Caracteres123456",
    "Issuer": "JwtAuthDemo",
    "Audience": "JwtAuthDemoUsers",
    "ExpirationInMinutes": 60,
    "RefreshTokenExpirationInDays": 7
  }
}
```

> ⚠️ **IMPORTANTE**: En producción, la SecretKey debe:
> - Tener al menos 32 caracteres
> - Almacenarse en Azure Key Vault, AWS Secrets Manager, o variables de entorno
> - Nunca estar en el código fuente

### Paso 2: Crear Clase de Configuración

Crea `Configuration/JwtSettings.cs`:

```csharp
namespace JwtAuthDemo.Configuration;

public class JwtSettings
{
    public string SecretKey { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int ExpirationInMinutes { get; set; }
    public int RefreshTokenExpirationInDays { get; set; }
}
```

### Paso 3: Crear Modelos

Crea `Models/User.cs`:

```csharp
namespace JwtAuthDemo.Models;

public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string Role { get; set; } = "User"; // User, Admin, Manager
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool IsActive { get; set; } = true;
}
```

Crea `Models/LoginRequest.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace JwtAuthDemo.Models;

public class LoginRequest
{
    [Required(ErrorMessage = "El email es obligatorio")]
    [EmailAddress(ErrorMessage = "El email no es válido")]
    public string Email { get; set; } = string.Empty;

    [Required(ErrorMessage = "La contraseña es obligatoria")]
    [MinLength(6, ErrorMessage = "La contraseña debe tener al menos 6 caracteres")]
    public string Password { get; set; } = string.Empty;
}
```

Crea `Models/LoginResponse.cs`:

```csharp
namespace JwtAuthDemo.Models;

public class LoginResponse
{
    public string Token { get; set; } = string.Empty;
    public string RefreshToken { get; set; } = string.Empty;
    public DateTime ExpiresAt { get; set; }
    public UserInfo User { get; set; } = new();
}

public class UserInfo
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string Role { get; set; } = string.Empty;
}
```

Crea `Models/RegisterRequest.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace JwtAuthDemo.Models;

public class RegisterRequest
{
    [Required(ErrorMessage = "El nombre es obligatorio")]
    [MinLength(2, ErrorMessage = "El nombre debe tener al menos 2 caracteres")]
    public string Name { get; set; } = string.Empty;

    [Required(ErrorMessage = "El email es obligatorio")]
    [EmailAddress(ErrorMessage = "El email no es válido")]
    public string Email { get; set; } = string.Empty;

    [Required(ErrorMessage = "La contraseña es obligatoria")]
    [MinLength(6, ErrorMessage = "La contraseña debe tener al menos 6 caracteres")]
    public string Password { get; set; } = string.Empty;

    [Required(ErrorMessage = "La confirmación de contraseña es obligatoria")]
    [Compare("Password", ErrorMessage = "Las contraseñas no coinciden")]
    public string ConfirmPassword { get; set; } = string.Empty;
}
```

### Paso 4: Configurar Program.cs (Parte 1)

Modifica `Program.cs`:

```csharp
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using Microsoft.OpenApi.Models;
using JwtAuthDemo.Configuration;
using JwtAuthDemo.Services;

var builder = WebApplication.CreateBuilder(args);

// Configurar JwtSettings desde appsettings.json
var jwtSettings = builder.Configuration
    .GetSection("JwtSettings")
    .Get<JwtSettings>() ?? throw new InvalidOperationException("JwtSettings no configurado");

builder.Services.AddSingleton(jwtSettings);

// Agregar servicios
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

// Configurar Swagger con soporte para JWT
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "JWT Auth Demo API",
        Version = "v1",
        Description = "API de demostración con autenticación JWT"
    });

    // Definir el esquema de seguridad JWT
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "Bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "Ingresa el token JWT en el formato: Bearer {tu token}"
    });

    // Requerir el esquema de seguridad globalmente
    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});

// Configurar autenticación JWT
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.RequireHttpsMetadata = false; // En producción debe ser true
    options.SaveToken = true;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwtSettings.SecretKey)),
        ValidateIssuer = true,
        ValidIssuer = jwtSettings.Issuer,
        ValidateAudience = true,
        ValidAudience = jwtSettings.Audience,
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero // Eliminar tolerancia de tiempo
    };
});

// Configurar autorización
builder.Services.AddAuthorization();

// Registrar servicios personalizados
builder.Services.AddScoped<IJwtService, JwtService>();
builder.Services.AddScoped<IAuthService, AuthService>();

var app = builder.Build();

// Configurar pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// IMPORTANTE: El orden es crucial
app.UseAuthentication(); // Primero autenticación
app.UseAuthorization();  // Luego autorización

app.MapControllers();

app.Run();
```

---

## Ejercicio 2: Generar y Validar Tokens

### Objetivo
Crear el servicio para generar y validar tokens JWT.

### Paso 1: Crear Interface del Servicio JWT

Crea `Services/IJwtService.cs`:

```csharp
using System.Security.Claims;
using JwtAuthDemo.Models;

namespace JwtAuthDemo.Services;

public interface IJwtService
{
    string GenerateToken(User user);
    string GenerateRefreshToken();
    ClaimsPrincipal? ValidateToken(string token);
}
```

### Paso 2: Implementar Servicio JWT

Crea `Services/JwtService.cs`:

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using Microsoft.IdentityModel.Tokens;
using JwtAuthDemo.Configuration;
using JwtAuthDemo.Models;

namespace JwtAuthDemo.Services;

public class JwtService : IJwtService
{
    private readonly JwtSettings _jwtSettings;
    private readonly ILogger<JwtService> _logger;

    public JwtService(JwtSettings jwtSettings, ILogger<JwtService> logger)
    {
        _jwtSettings = jwtSettings;
        _logger = logger;
    }

    public string GenerateToken(User user)
    {
        var securityKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_jwtSettings.SecretKey));
        
        var credentials = new SigningCredentials(
            securityKey, 
            SecurityAlgorithms.HmacSha256);

        // Crear claims (información del usuario en el token)
        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Email),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.Name),
            new Claim(ClaimTypes.Role, user.Role),
            new Claim("userId", user.Id.ToString())
        };

        // Crear el token
        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationInMinutes),
            signingCredentials: credentials
        );

        var tokenString = new JwtSecurityTokenHandler().WriteToken(token);

        _logger.LogInformation(
            "Token generado para usuario {UserId} - {Email}", 
            user.Id, 
            user.Email);

        return tokenString;
    }

    public string GenerateRefreshToken()
    {
        var randomNumber = new byte[32];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomNumber);
        return Convert.ToBase64String(randomNumber);
    }

    public ClaimsPrincipal? ValidateToken(string token)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.UTF8.GetBytes(_jwtSettings.SecretKey);

        try
        {
            var principal = tokenHandler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ValidateIssuer = true,
                ValidIssuer = _jwtSettings.Issuer,
                ValidateAudience = true,
                ValidAudience = _jwtSettings.Audience,
                ValidateLifetime = true,
                ClockSkew = TimeSpan.Zero
            }, out SecurityToken validatedToken);

            return principal;
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Error validando token");
            return null;
        }
    }
}
```

### Paso 3: Crear Servicio de Autenticación

Crea `Services/IAuthService.cs`:

```csharp
using JwtAuthDemo.Models;

namespace JwtAuthDemo.Services;

public interface IAuthService
{
    Task<LoginResponse?> LoginAsync(LoginRequest request);
    Task<LoginResponse?> RegisterAsync(RegisterRequest request);
    Task<User?> GetUserByEmailAsync(string email);
}
```

Crea `Services/AuthService.cs`:

```csharp
using BCrypt.Net;
using JwtAuthDemo.Models;

namespace JwtAuthDemo.Services;

public class AuthService : IAuthService
{
    private readonly IJwtService _jwtService;
    private readonly ILogger<AuthService> _logger;
    
    // Simulación de base de datos en memoria
    private static readonly List<User> _users = new()
    {
        new User
        {
            Id = 1,
            Email = "admin@example.com",
            Name = "Administrador",
            PasswordHash = BCrypt.Net.BCrypt.HashPassword("admin123"),
            Role = "Admin",
            IsActive = true
        },
        new User
        {
            Id = 2,
            Email = "user@example.com",
            Name = "Usuario Normal",
            PasswordHash = BCrypt.Net.BCrypt.HashPassword("user123"),
            Role = "User",
            IsActive = true
        }
    };

    public AuthService(IJwtService jwtService, ILogger<AuthService> logger)
    {
        _jwtService = jwtService;
        _logger = logger;
    }

    public async Task<LoginResponse?> LoginAsync(LoginRequest request)
    {
        await Task.Delay(100); // Simular latencia de DB

        var user = _users.FirstOrDefault(u => 
            u.Email.Equals(request.Email, StringComparison.OrdinalIgnoreCase));

        if (user == null)
        {
            _logger.LogWarning("Intento de login fallido: usuario no encontrado {Email}", request.Email);
            return null;
        }

        if (!user.IsActive)
        {
            _logger.LogWarning("Intento de login con cuenta inactiva {Email}", request.Email);
            return null;
        }

        if (!BCrypt.Net.BCrypt.Verify(request.Password, user.PasswordHash))
        {
            _logger.LogWarning("Intento de login fallido: contraseña incorrecta {Email}", request.Email);
            return null;
        }

        var token = _jwtService.GenerateToken(user);
        var refreshToken = _jwtService.GenerateRefreshToken();
        var expiresAt = DateTime.UtcNow.AddMinutes(60);

        _logger.LogInformation("Login exitoso para {Email}", user.Email);

        return new LoginResponse
        {
            Token = token,
            RefreshToken = refreshToken,
            ExpiresAt = expiresAt,
            User = new UserInfo
            {
                Id = user.Id,
                Email = user.Email,
                Name = user.Name,
                Role = user.Role
            }
        };
    }

    public async Task<LoginResponse?> RegisterAsync(RegisterRequest request)
    {
        await Task.Delay(100); // Simular latencia de DB

        // Verificar si el usuario ya existe
        if (_users.Any(u => u.Email.Equals(request.Email, StringComparison.OrdinalIgnoreCase)))
        {
            _logger.LogWarning("Intento de registro con email existente {Email}", request.Email);
            return null;
        }

        // Crear nuevo usuario
        var newUser = new User
        {
            Id = _users.Max(u => u.Id) + 1,
            Email = request.Email,
            Name = request.Name,
            PasswordHash = BCrypt.Net.BCrypt.HashPassword(request.Password),
            Role = "User",
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };

        _users.Add(newUser);

        _logger.LogInformation("Usuario registrado exitosamente {Email}", newUser.Email);

        // Generar token automáticamente después del registro
        var token = _jwtService.GenerateToken(newUser);
        var refreshToken = _jwtService.GenerateRefreshToken();
        var expiresAt = DateTime.UtcNow.AddMinutes(60);

        return new LoginResponse
        {
            Token = token,
            RefreshToken = refreshToken,
            ExpiresAt = expiresAt,
            User = new UserInfo
            {
                Id = newUser.Id,
                Email = newUser.Email,
                Name = newUser.Name,
                Role = newUser.Role
            }
        };
    }

    public async Task<User?> GetUserByEmailAsync(string email)
    {
        await Task.Delay(50); // Simular latencia de DB
        return _users.FirstOrDefault(u => 
            u.Email.Equals(email, StringComparison.OrdinalIgnoreCase));
    }
}
```

---

## Ejercicio 3: Protección Global de Endpoints

### Objetivo
Implementar protección global de todos los endpoints, similar a Spring Security, permitiendo solo acceso público a endpoints específicos.

### Paso 1: Crear Atributo Personalizado para Endpoints Públicos

Crea `Attributes/AllowAnonymousAttribute.cs` (si no existe ya):

```csharp
// .NET Core ya incluye este atributo, solo documentamos su uso
// using Microsoft.AspNetCore.Authorization;
```

### Paso 2: Configurar Política de Autorización Global

Modifica `Program.cs` para agregar política global:

```csharp
// Reemplazar la línea: builder.Services.AddAuthorization();
// Por:

builder.Services.AddAuthorization(options =>
{
    // Política por defecto: requiere autenticación
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

### Paso 3: Crear Controlador de Autenticación

Crea `Controllers/AuthController.cs`:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using JwtAuthDemo.Models;
using JwtAuthDemo.Services;

namespace JwtAuthDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;
    private readonly ILogger<AuthController> _logger;

    public AuthController(IAuthService authService, ILogger<AuthController> logger)
    {
        _authService = authService;
        _logger = logger;
    }

    /// <summary>
    /// Endpoint público para login
    /// </summary>
    [AllowAnonymous]
    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginRequest request)
    {
        if (!ModelState.IsValid)
        {
            return BadRequest(new
            {
                success = false,
                message = "Datos inválidos",
                errors = ModelState.Values.SelectMany(v => v.Errors.Select(e => e.ErrorMessage))
            });
        }

        var result = await _authService.LoginAsync(request);

        if (result == null)
        {
            return Unauthorized(new
            {
                success = false,
                message = "Email o contraseña incorrectos"
            });
        }

        return Ok(new
        {
            success = true,
            message = "Login exitoso",
            data = result
        });
    }

    /// <summary>
    /// Endpoint público para registro
    /// </summary>
    [AllowAnonymous]
    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterRequest request)
    {
        if (!ModelState.IsValid)
        {
            return BadRequest(new
            {
                success = false,
                message = "Datos inválidos",
                errors = ModelState.Values.SelectMany(v => v.Errors.Select(e => e.ErrorMessage))
            });
        }

        var result = await _authService.RegisterAsync(request);

        if (result == null)
        {
            return BadRequest(new
            {
                success = false,
                message = "El email ya está registrado"
            });
        }

        return Ok(new
        {
            success = true,
            message = "Registro exitoso",
            data = result
        });
    }

    /// <summary>
    /// Endpoint protegido para obtener información del usuario actual
    /// </summary>
    [HttpGet("me")]
    public async Task<IActionResult> GetCurrentUser()
    {
        var email = User.FindFirst(System.Security.Claims.ClaimTypes.Email)?.Value;
        
        if (string.IsNullOrEmpty(email))
        {
            return Unauthorized(new { success = false, message = "Token inválido" });
        }

        var user = await _authService.GetUserByEmailAsync(email);

        if (user == null)
        {
            return NotFound(new { success = false, message = "Usuario no encontrado" });
        }

        return Ok(new
        {
            success = true,
            data = new
            {
                user.Id,
                user.Email,
                user.Name,
                user.Role,
                user.CreatedAt
            }
        });
    }

    /// <summary>
    /// Endpoint público para verificar el estado de la API
    /// </summary>
    [AllowAnonymous]
    [HttpGet("health")]
    public IActionResult Health()
    {
        return Ok(new
        {
            success = true,
            message = "API funcionando correctamente",
            timestamp = DateTime.UtcNow
        });
    }
}
```

### Paso 4: Crear Controladores Protegidos

Crea `Controllers/ProductsController.cs`:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace JwtAuthDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize] // Este atributo es redundante con FallbackPolicy, pero es explícito
public class ProductsController : ControllerBase
{
    private static readonly List<Product> Products = new()
    {
        new Product { Id = 1, Name = "Laptop", Price = 999.99m },
        new Product { Id = 2, Name = "Mouse", Price = 29.99m },
        new Product { Id = 3, Name = "Teclado", Price = 79.99m },
    };

    /// <summary>
    /// Obtener todos los productos - Requiere autenticación
    /// </summary>
    [HttpGet]
    public IActionResult GetAll()
    {
        var userName = User.Identity?.Name;
        var userRole = User.FindFirst(System.Security.Claims.ClaimTypes.Role)?.Value;

        return Ok(new
        {
            success = true,
            message = $"Productos obtenidos por {userName} (Rol: {userRole})",
            data = Products
        });
    }

    /// <summary>
    /// Obtener producto por ID - Requiere autenticación
    /// </summary>
    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        var product = Products.FirstOrDefault(p => p.Id == id);
        
        if (product == null)
        {
            return NotFound(new { success = false, message = "Producto no encontrado" });
        }

        return Ok(new { success = true, data = product });
    }

    /// <summary>
    /// Crear producto - Solo Admin
    /// </summary>
    [HttpPost]
    [Authorize(Roles = "Admin")]
    public IActionResult Create([FromBody] Product product)
    {
        product.Id = Products.Max(p => p.Id) + 1;
        Products.Add(product);

        return CreatedAtAction(
            nameof(GetById),
            new { id = product.Id },
            new { success = true, message = "Producto creado", data = product }
        );
    }

    /// <summary>
    /// Eliminar producto - Solo Admin
    /// </summary>
    [HttpDelete("{id}")]
    [Authorize(Roles = "Admin")]
    public IActionResult Delete(int id)
    {
        var product = Products.FirstOrDefault(p => p.Id == id);
        
        if (product == null)
        {
            return NotFound(new { success = false, message = "Producto no encontrado" });
        }

        Products.Remove(product);
        return Ok(new { success = true, message = "Producto eliminado" });
    }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}
```

Crea `Controllers/UsersController.cs`:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace JwtAuthDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize(Roles = "Admin")] // Todo el controlador requiere rol Admin
public class UsersController : ControllerBase
{
    private readonly ILogger<UsersController> _logger;

    public UsersController(ILogger<UsersController> logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// Listar todos los usuarios - Solo Admin
    /// </summary>
    [HttpGet]
    public IActionResult GetAll()
    {
        // En una aplicación real, esto vendría de la base de datos
        var users = new[]
        {
            new { Id = 1, Email = "admin@example.com", Name = "Admin", Role = "Admin" },
            new { Id = 2, Email = "user@example.com", Name = "Usuario", Role = "User" }
        };

        return Ok(new { success = true, data = users });
    }

    /// <summary>
    /// Ver estadísticas - Solo Admin
    /// </summary>
    [HttpGet("stats")]
    public IActionResult GetStats()
    {
        return Ok(new
        {
            success = true,
            data = new
            {
                totalUsers = 2,
                activeUsers = 2,
                adminUsers = 1,
                regularUsers = 1
            }
        });
    }
}
```

### Paso 5: Excluir Swagger de la Autenticación

El código en `Program.cs` ya está configurado correctamente. Swagger está excluido automáticamente porque:

1. Los endpoints de Swagger no están decorados con `[Authorize]`
2. Los endpoints de Swagger son parte del middleware, no de los controladores
3. La `FallbackPolicy` solo se aplica a los endpoints de los controladores

### Paso 6: Probar la Configuración

```bash
dotnet run
```

**Pruebas a realizar:**

1. **Acceso a Swagger** (debe funcionar sin autenticación):
   - `https://localhost:7xxx/swagger`

2. **Endpoints públicos** (sin token):
   ```bash
   # Health check
   curl https://localhost:7xxx/api/auth/health
   
   # Login
   curl -X POST https://localhost:7xxx/api/auth/login \
     -H "Content-Type: application/json" \
     -d '{"email":"user@example.com","password":"user123"}'
   ```

3. **Endpoints protegidos** (sin token - debe fallar):
   ```bash
   curl https://localhost:7xxx/api/products
   # Respuesta: 401 Unauthorized
   ```

4. **Endpoints protegidos** (con token - debe funcionar):
   ```bash
   # Primero hacer login y copiar el token
   TOKEN="tu_token_aqui"
   
   curl https://localhost:7xxx/api/products \
     -H "Authorization: Bearer $TOKEN"
   ```

---

## Ejercicio 4: Refresh Tokens

### Objetivo
Implementar refresh tokens para renovar tokens expirados sin requerir credenciales.

### Paso 1: Crear Modelos para Refresh Token

Crea `Models/RefreshTokenRequest.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace JwtAuthDemo.Models;

public class RefreshTokenRequest
{
    [Required]
    public string Token { get; set; } = string.Empty;

    [Required]
    public string RefreshToken { get; set; } = string.Empty;
}
```

Crea `Models/RefreshToken.cs`:

```csharp
namespace JwtAuthDemo.Models;

public class RefreshToken
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public string Token { get; set; } = string.Empty;
    public DateTime ExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool IsRevoked { get; set; } = false;
    public bool IsUsed { get; set; } = false;
}
```

### Paso 2: Extender AuthService

Modifica `Services/IAuthService.cs`:

```csharp
public interface IAuthService
{
    Task<LoginResponse?> LoginAsync(LoginRequest request);
    Task<LoginResponse?> RegisterAsync(RegisterRequest request);
    Task<User?> GetUserByEmailAsync(string email);
    Task<LoginResponse?> RefreshTokenAsync(RefreshTokenRequest request);
}
```

Modifica `Services/AuthService.cs` para agregar gestión de refresh tokens:

```csharp
// Agregar al inicio de la clase, después de _users:
private static readonly List<RefreshToken> _refreshTokens = new();

// Agregar método auxiliar
private void StoreRefreshToken(int userId, string refreshToken, int expirationDays)
{
    var token = new RefreshToken
    {
        Id = _refreshTokens.Count + 1,
        UserId = userId,
        Token = refreshToken,
        ExpiresAt = DateTime.UtcNow.AddDays(expirationDays),
        CreatedAt = DateTime.UtcNow
    };

    _refreshTokens.Add(token);
}

// Modificar LoginAsync para almacenar refresh token
public async Task<LoginResponse?> LoginAsync(LoginRequest request)
{
    // ... código existente ...

    var token = _jwtService.GenerateToken(user);
    var refreshToken = _jwtService.GenerateRefreshToken();
    var expiresAt = DateTime.UtcNow.AddMinutes(60);

    // Almacenar refresh token
    StoreRefreshToken(user.Id, refreshToken, 7);

    _logger.LogInformation("Login exitoso para {Email}", user.Email);

    return new LoginResponse
    {
        Token = token,
        RefreshToken = refreshToken,
        ExpiresAt = expiresAt,
        User = new UserInfo
        {
            Id = user.Id,
            Email = user.Email,
            Name = user.Name,
            Role = user.Role
        }
    };
}

// Modificar RegisterAsync similar a LoginAsync
public async Task<LoginResponse?> RegisterAsync(RegisterRequest request)
{
    // ... código existente hasta antes del return ...

    var token = _jwtService.GenerateToken(newUser);
    var refreshToken = _jwtService.GenerateRefreshToken();
    var expiresAt = DateTime.UtcNow.AddMinutes(60);

    // Almacenar refresh token
    StoreRefreshToken(newUser.Id, refreshToken, 7);

    return new LoginResponse
    {
        Token = token,
        RefreshToken = refreshToken,
        ExpiresAt = expiresAt,
        User = new UserInfo
        {
            Id = newUser.Id,
            Email = newUser.Email,
            Name = newUser.Name,
            Role = newUser.Role
        }
    };
}

// Agregar nuevo método
public async Task<LoginResponse?> RefreshTokenAsync(RefreshTokenRequest request)
{
    await Task.Delay(50);

    // Validar el token actual (aunque esté expirado)
    var principal = _jwtService.ValidateToken(request.Token);
    
    if (principal == null)
    {
        // Intentar validar sin verificar expiración
        var tokenHandler = new System.IdentityModel.Tokens.Jwt.JwtSecurityTokenHandler();
        var jwtToken = tokenHandler.ReadJwtToken(request.Token);
        
        var email = jwtToken.Claims.FirstOrDefault(c => c.Type == "email")?.Value;
        
        if (string.IsNullOrEmpty(email))
        {
            _logger.LogWarning("Token inválido para refresh");
            return null;
        }

        var user = await GetUserByEmailAsync(email);
        if (user == null)
        {
            return null;
        }

        // Validar refresh token
        var storedRefreshToken = _refreshTokens.FirstOrDefault(rt =>
            rt.Token == request.RefreshToken &&
            rt.UserId == user.Id &&
            !rt.IsRevoked &&
            !rt.IsUsed &&
            rt.ExpiresAt > DateTime.UtcNow);

        if (storedRefreshToken == null)
        {
            _logger.LogWarning("Refresh token inválido o expirado para usuario {UserId}", user.Id);
            return null;
        }

        // Marcar el refresh token como usado
        storedRefreshToken.IsUsed = true;

        // Generar nuevos tokens
        var newToken = _jwtService.GenerateToken(user);
        var newRefreshToken = _jwtService.GenerateRefreshToken();
        var expiresAt = DateTime.UtcNow.AddMinutes(60);

        // Almacenar nuevo refresh token
        StoreRefreshToken(user.Id, newRefreshToken, 7);

        _logger.LogInformation("Token renovado para usuario {Email}", user.Email);

        return new LoginResponse
        {
            Token = newToken,
            RefreshToken = newRefreshToken,
            ExpiresAt = expiresAt,
            User = new UserInfo
            {
                Id = user.Id,
                Email = user.Email,
                Name = user.Name,
                Role = user.Role
            }
        };
    }

    return null;
}
```

### Paso 3: Agregar Endpoint de Refresh

Modifica `Controllers/AuthController.cs` para agregar:

```csharp
/// <summary>
/// Endpoint público para renovar token
/// </summary>
[AllowAnonymous]
[HttpPost("refresh")]
public async Task<IActionResult> RefreshToken([FromBody] RefreshTokenRequest request)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(new
        {
            success = false,
            message = "Datos inválidos"
        });
    }

    var result = await _authService.RefreshTokenAsync(request);

    if (result == null)
    {
        return Unauthorized(new
        {
            success = false,
            message = "Token de actualización inválido o expirado"
        });
    }

    return Ok(new
    {
        success = true,
        message = "Token renovado exitosamente",
        data = result
    });
}

/// <summary>
/// Endpoint protegido para cerrar sesión (revocar tokens)
/// </summary>
[HttpPost("logout")]
public IActionResult Logout()
{
    // En una implementación real, aquí revocarías el refresh token
    var email = User.FindFirst(System.Security.Claims.ClaimTypes.Email)?.Value;
    
    _logger.LogInformation("Logout para usuario {Email}", email);

    return Ok(new
    {
        success = true,
        message = "Sesión cerrada exitosamente"
    });
}
```

### Paso 4: Probar Refresh Token

```bash
# 1. Login
curl -X POST https://localhost:7xxx/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"user123"}'

# Copiar el token y refreshToken de la respuesta

# 2. Usar el API normalmente
curl https://localhost:7xxx/api/products \
  -H "Authorization: Bearer TU_TOKEN"

# 3. Cuando el token expire (o para probar), usar refresh
curl -X POST https://localhost:7xxx/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{
    "token": "TU_TOKEN_EXPIRADO",
    "refreshToken": "TU_REFRESH_TOKEN"
  }'

# 4. Usar el nuevo token
curl https://localhost:7xxx/api/products \
  -H "Authorization: Bearer NUEVO_TOKEN"
```

---

## Ejercicio 5: Roles y Claims

### Objetivo
Implementar autorización basada en roles y claims personalizados.

### Paso 1: Crear Políticas de Autorización

Modifica `Program.cs` para agregar políticas personalizadas:

```csharp
builder.Services.AddAuthorization(options =>
{
    // Política por defecto: requiere autenticación
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    // Política para Admin
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    // Política para Admin o Manager
    options.AddPolicy("AdminOrManager", policy =>
        policy.RequireRole("Admin", "Manager"));

    // Política personalizada con claims
    options.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription", "premium"));

    // Política con múltiples requisitos
    options.AddPolicy("SeniorAdmin", policy =>
    {
        policy.RequireRole("Admin");
        policy.RequireClaim("seniority", "senior");
    });
});
```

### Paso 2: Agregar Claims Personalizados

Modifica `Services/JwtService.cs` en el método `GenerateToken`:

```csharp
public string GenerateToken(User user)
{
    var securityKey = new SymmetricSecurityKey(
        Encoding.UTF8.GetBytes(_jwtSettings.SecretKey));
    
    var credentials = new SigningCredentials(
        securityKey, 
        SecurityAlgorithms.HmacSha256);

    // Crear lista de claims
    var claims = new List<Claim>
    {
        new Claim(JwtRegisteredClaimNames.Sub, user.Email),
        new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        new Claim(JwtRegisteredClaimNames.Email, user.Email),
        new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new Claim(ClaimTypes.Name, user.Name),
        new Claim(ClaimTypes.Role, user.Role),
        new Claim("userId", user.Id.ToString()),
        new Claim("email", user.Email)
    };

    // Agregar claims personalizados según el rol o usuario
    if (user.Role == "Admin")
    {
        claims.Add(new Claim("canDelete", "true"));
        claims.Add(new Claim("canManageUsers", "true"));
    }

    if (user.Email == "admin@example.com")
    {
        claims.Add(new Claim("seniority", "senior"));
    }

    // Ejemplo de claim de suscripción
    claims.Add(new Claim("subscription", user.Id == 1 ? "premium" : "free"));

    var token = new JwtSecurityToken(
        issuer: _jwtSettings.Issuer,
        audience: _jwtSettings.Audience,
        claims: claims,
        expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationInMinutes),
        signingCredentials: credentials
    );

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

### Paso 3: Crear Controlador con Políticas

Crea `Controllers/AdminController.cs`:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace JwtAuthDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AdminController : ControllerBase
{
    /// <summary>
    /// Solo para administradores
    /// </summary>
    [HttpGet("dashboard")]
    [Authorize(Policy = "AdminOnly")]
    public IActionResult GetDashboard()
    {
        return Ok(new
        {
            success = true,
            message = "Dashboard de administrador",
            data = new
            {
                totalUsers = 150,
                totalOrders = 1234,
                revenue = 98765.43m
            }
        });
    }

    /// <summary>
    /// Para Admin o Manager
    /// </summary>
    [HttpGet("reports")]
    [Authorize(Policy = "AdminOrManager")]
    public IActionResult GetReports()
    {
        var userRole = User.FindFirst(System.Security.Claims.ClaimTypes.Role)?.Value;
        
        return Ok(new
        {
            success = true,
            message = $"Reportes accedidos por {userRole}",
            data = new[] { "Reporte 1", "Reporte 2", "Reporte 3" }
        });
    }

    /// <summary>
    /// Solo para Admin Senior
    /// </summary>
    [HttpDelete("critical-data")]
    [Authorize(Policy = "SeniorAdmin")]
    public IActionResult DeleteCriticalData()
    {
        return Ok(new
        {
            success = true,
            message = "Operación crítica ejecutada"
        });
    }

    /// <summary>
    /// Verificar claims del usuario actual
    /// </summary>
    [HttpGet("my-claims")]
    [Authorize]
    public IActionResult GetMyClaims()
    {
        var claims = User.Claims.Select(c => new
        {
            type = c.Type,
            value = c.Value
        });

        return Ok(new
        {
            success = true,
            data = claims
        });
    }
}
```

### Paso 4: Crear Handler Personalizado de Autorización

Crea `Authorization/MinimumAgeRequirement.cs`:

```csharp
using Microsoft.AspNetCore.Authorization;

namespace JwtAuthDemo.Authorization;

public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var ageClai = context.User.FindFirst("age");

        if (ageClaim != null && int.TryParse(ageClaim.Value, out var age))
        {
            if (age >= requirement.MinimumAge)
            {
                context.Succeed(requirement);
            }
        }

        return Task.CompletedTask;
    }
}
```

Registrar en `Program.cs`:

```csharp
using JwtAuthDemo.Authorization;

// Después de AddAuthorization
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();

// Agregar política con requirement personalizado
builder.Services.AddAuthorization(options =>
{
    // ... políticas existentes ...

    options.AddPolicy("Over18", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
});
```

### Paso 5: Probar Autorización por Roles

```bash
# Login como Admin
curl -X POST https://localhost:7xxx/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"admin123"}'

# Acceder a dashboard (debe funcionar)
curl https://localhost:7xxx/api/admin/dashboard \
  -H "Authorization: Bearer ADMIN_TOKEN"

# Login como User
curl -X POST https://localhost:7xxx/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"user123"}'

# Intentar acceder a dashboard (debe fallar con 403 Forbidden)
curl https://localhost:7xxx/api/admin/dashboard \
  -H "Authorization: Bearer USER_TOKEN"
```

---

## Conclusiones y Mejores Prácticas

### Resumen del Laboratorio

Has aprendido a:

1. ✅ Configurar autenticación JWT en .NET Core 8
2. ✅ Generar y validar tokens JWT
3. ✅ Proteger todos los endpoints por defecto (estilo Spring Security)
4. ✅ Excluir endpoints específicos con `[AllowAnonymous]`
5. ✅ Implementar refresh tokens para renovar sesiones
6. ✅ Usar roles y claims para autorización
7. ✅ Crear políticas de autorización personalizadas

### Diferencias con Spring Security

| Aspecto | Spring Security | .NET Core 8 |
|---------|----------------|-------------|
| Protección por defecto | `anyRequest().authenticated()` | `FallbackPolicy.RequireAuthenticatedUser()` |
| Endpoints públicos | `.requestMatchers().permitAll()` | `[AllowAnonymous]` |
| Roles | `@PreAuthorize("hasRole('ADMIN')")` | `[Authorize(Roles = "Admin")]` |
| Políticas | Security Expressions | Authorization Policies |
| Configuración | Java Config / Annotations | Program.cs / Attributes |

### Mejores Prácticas de Seguridad

#### 1. Almacenamiento de Secretos
```csharp
// ❌ MAL - Nunca en código
var secretKey = "mi-clave-secreta";

// ✅ BIEN - Variables de entorno
var secretKey = Environment.GetEnvironmentVariable("JWT_SECRET_KEY");

// ✅ MEJOR - Azure Key Vault / AWS Secrets Manager
var secretKey = await keyVaultClient.GetSecretAsync("jwt-secret-key");
```

#### 2. Configuración del Token
```csharp
// ✅ Configuración recomendada
new TokenValidationParameters
{
    ValidateIssuerSigningKey = true,  // Validar firma
    ValidateIssuer = true,            // Validar emisor
    ValidateAudience = true,          // Validar audiencia
    ValidateLifetime = true,          // Validar expiración
    ClockSkew = TimeSpan.Zero,        // Sin tolerancia de tiempo
    RequireExpirationTime = true      // Exigir fecha de expiración
};
```

#### 3. HTTPS en Producción
```csharp
// ✅ En producción
options.RequireHttpsMetadata = true;

// ❌ Solo en desarrollo
options.RequireHttpsMetadata = false;
```

#### 4. Tiempo de Expiración
```json
{
  "JwtSettings": {
    "ExpirationInMinutes": 15,          // Access token corto
    "RefreshTokenExpirationInDays": 7   // Refresh token más largo
  }
}
```

#### 5. Almacenamiento en Cliente
```javascript
// ❌ MAL - localStorage (vulnerable a XSS)
localStorage.setItem('token', token);

// ✅ MEJOR - memoria (se pierde al cerrar)
let authToken = null;

// ✅ MEJOR - httpOnly cookie (si es misma origen)
// Configurar en servidor con httpOnly, secure, sameSite
```

#### 6. Rotación de Refresh Tokens
```csharp
// ✅ Siempre generar nuevo refresh token
public async Task<LoginResponse?> RefreshTokenAsync(RefreshTokenRequest request)
{
    // Marcar el viejo como usado
    oldRefreshToken.IsUsed = true;
    
    // Generar nuevos tokens
    var newToken = _jwtService.GenerateToken(user);
    var newRefreshToken = _jwtService.GenerateRefreshToken();
    
    return new LoginResponse { Token = newToken, RefreshToken = newRefreshToken };
}
```

#### 7. Validación de Entrada
```csharp
// ✅ Siempre validar con Data Annotations
public class LoginRequest
{
    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    [MinLength(6)]
    public string Password { get; set; }
}
```

### Checklist de Producción

Antes de desplegar a producción, verifica:

- [ ] SecretKey almacenada en servicio de secretos (Key Vault, etc.)
- [ ] `RequireHttpsMetadata = true`
- [ ] HTTPS habilitado (certificado SSL válido)
- [ ] Tokens de acceso con expiración corta (15-30 min)
- [ ] Refresh tokens con expiración razonable (7-30 días)
- [ ] Logging de eventos de seguridad
- [ ] Rate limiting en endpoints de autenticación
- [ ] Validación de entrada robusta
- [ ] Manejo de errores sin exponer información sensible
- [ ] CORS configurado correctamente
- [ ] Base de datos para usuarios real (no lista en memoria)
- [ ] Hash de contraseñas con BCrypt, Argon2, o PBKDF2
- [ ] Implementar políticas de contraseñas fuertes
- [ ] 2FA para operaciones sensibles
- [ ] Auditoría de accesos

### Integración con Entity Framework Core

Para usar con base de datos real:

```csharp
// ApplicationDbContext.cs
public class ApplicationDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<RefreshToken> RefreshTokens { get; set; }
}

// Program.cs
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// AuthService.cs
public class AuthService : IAuthService
{
    private readonly ApplicationDbContext _context;
    
    public async Task<User?> GetUserByEmailAsync(string email)
    {
        return await _context.Users
            .FirstOrDefaultAsync(u => u.Email == email);
    }
}
```

### Ejemplo de Cliente (Frontend)

#### JavaScript/TypeScript
```javascript
class AuthService {
    async login(email, password) {
        const response = await fetch('/api/auth/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email, password })
        });
        
        const data = await response.json();
        
        if (data.success) {
            this.token = data.data.token;
            this.refreshToken = data.data.refreshToken;
        }
        
        return data;
    }
    
    async callApi(endpoint) {
        const response = await fetch(endpoint, {
            headers: {
                'Authorization': `Bearer ${this.token}`
            }
        });
        
        if (response.status === 401) {
            // Token expirado, intentar refresh
            await this.refreshAccessToken();
            return this.callApi(endpoint); // Reintentar
        }
        
        return response.json();
    }
    
    async refreshAccessToken() {
        const response = await fetch('/api/auth/refresh', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                token: this.token,
                refreshToken: this.refreshToken
            })
        });
        
        const data = await response.json();
        
        if (data.success) {
            this.token = data.data.token;
            this.refreshToken = data.data.refreshToken;
        }
    }
}
```

### Recursos Adicionales

- [Microsoft Docs - JWT Bearer Authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/)
- [RFC 7519 - JSON Web Token](https://tools.ietf.org/html/rfc7519)
- [JWT.io - Debugger](https://jwt.io/)
- [OWASP - Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

### Próximos Pasos

Para profundizar:

1. Implementar **ASP.NET Core Identity** para gestión completa de usuarios
2. Agregar **2FA (Two-Factor Authentication)**
3. Implementar **OAuth 2.0 / OpenID Connect**
4. Integrar con **IdentityServer** o **Azure AD B2C**
5. Agregar **Rate Limiting** con AspNetCoreRateLimit
6. Implementar **API Keys** para clientes M2M
7. Configurar **CORS** adecuadamente
8. Agregar **Audit Logging** de eventos de seguridad

---

## Ejercicio Extra: Cliente de Prueba Completo

Crea un controlador para probar todo el flujo:

```csharp
[ApiController]
[Route("api/[controller]")]
public class TestController : ControllerBase
{
    [AllowAnonymous]
    [HttpGet("public")]
    public IActionResult PublicEndpoint()
    {
        return Ok(new { message = "Este endpoint es público" });
    }

    [HttpGet("protected")]
    public IActionResult ProtectedEndpoint()
    {
        var userName = User.Identity?.Name;
        return Ok(new { message = $"Hola {userName}, estás autenticado" });
    }

    [Authorize(Roles = "Admin")]
    [HttpGet("admin-only")]
    public IActionResult AdminOnlyEndpoint()
    {
        return Ok(new { message = "Solo administradores pueden ver esto" });
    }

    [Authorize(Policy = "PremiumUser")]
    [HttpGet("premium")]
    public IActionResult PremiumEndpoint()
    {
        return Ok(new { message = "Contenido premium" });
    }
}
```

---


