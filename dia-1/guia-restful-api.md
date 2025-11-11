# Guía de Buenas Prácticas para APIs RESTful

## 📋 Índice

1. [Modelo de Datos](#modelo-de-datos)
2. [Buenas Prácticas en URLs](#buenas-prácticas-en-urls)
3. [Operaciones CRUD](#operaciones-crud)
4. [Códigos de Estado HTTP](#códigos-de-estado-http)
5. [Recursos Adicionales](#recursos-adicionales)

---

## 🍕 Modelo de Datos

### Ejemplo: Catálogo de Pizzas

```json
{
  "id": "uuid",
  "name": "string",
  "description": "string",
  "url": "string",
  "price": "number",
  "ingredients": [
    {
      "id": "uuid",
      "name": "string",
      "cost": "number"
    }
  ]
}
```

**Cálculo del precio:**
```
precio = suma(costo_ingredientes) + 20% beneficio
```

---

## 🎯 Buenas Prácticas en URLs

### Servidor Base
```
http://localhost:8080
```

### 1. Pluralización del Recurso
✅ **Correcto:**
```
/pizzas
```

❌ **Incorrecto:**
```
/pizza
```

### 2. No Exponer Acciones en la URL

Las acciones se definen mediante los verbos HTTP, no en la URL.

❌ **Incorrecto:**
```
/pizzas/create
/pizzas/update
/pizzas/delete
```

✅ **Correcto:**
```
POST   /pizzas      → Crear
PUT    /pizzas/{id} → Actualizar completo
PATCH  /pizzas/{id} → Actualizar parcial
DELETE /pizzas/{id} → Eliminar
```

### 3. No Exponer Formatos en la URL

❌ **Incorrecto:**
```
/pizzas.json
/pizzas.xml
```

✅ **Correcto:**

Usar headers HTTP para negociación de contenido:

```http
Accept: application/json
Content-Type: application/json
```

**Tipos MIME comunes:**
- `application/json`
- `application/xml`
- `text/html`
- `text/plain`

📚 [Lista completa de tipos MIME](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/MIME_types/Common_types)

### 4. Versionado de la API

```
/v1/pizzas
/v2/pizzas
```

**Beneficios:**
- Permite evolución sin romper clientes existentes
- Facilita la migración gradual
- Mejora la mantenibilidad

---

## 🔧 Operaciones CRUD

### 1️⃣ CREATE - Crear un Recurso

**Endpoint:**
```
POST /v1/pizzas
```

**Request:**
```http
POST /v1/pizzas HTTP/1.1
Content-Type: application/json

{
  "name": "Margarita",
  "description": "Pizza clásica italiana",
  "url": "https://example.com/images/margarita.jpg",
  "ingredients": [
    {"id": "ing-001"},
    {"id": "ing-002"},
    {"id": "ing-003"}
  ]
}
```

**Response exitosa:**
```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /v1/pizzas/123

{
  "id": "123",
  "name": "Margarita",
  "price": 12.50,
  "url": "https://example.com/images/margarita.jpg",
  "description": "Pizza clásica italiana",
  "ingredients": [
    {"id": "ing-001", "name": "Tomate"},
    {"id": "ing-002", "name": "Mozzarella"},
    {"id": "ing-003", "name": "Albahaca"}
  ]
}
```

**Códigos de estado:**
- ✅ `201 Created` - Recurso creado exitosamente
- ❌ `400 Bad Request` - Sintaxis incorrecta
- ❌ `422 Unprocessable Entity` - Error de validación
- ❌ `409 Conflict` - El recurso ya existe
- ❌ `401 Unauthorized` - No autenticado
- ❌ `403 Forbidden` - Sin permisos
- ❌ `500 Internal Server Error` - Error del servidor

---

### 2️⃣ UPDATE - Actualizar un Recurso

**Endpoint:**
```
PUT   /v1/pizzas/{id}  → Actualización completa
PATCH /v1/pizzas/{id}  → Actualización parcial
```

**Diferencias PUT vs PATCH:**
- **PUT:** Reemplaza el recurso completo
- **PATCH:** Modifica solo los campos enviados

**Request:**
```http
PUT /v1/pizzas/123 HTTP/1.1
Content-Type: application/json

{
  "name": "Margarita Premium",
  "description": "Pizza clásica con ingredientes premium",
  "url": "https://example.com/images/margarita-premium.jpg",
  "ingredients": [
    {"id": "ing-001"},
    {"id": "ing-002"},
    {"id": "ing-003"},
    {"id": "ing-004"}
  ]
}
```

**Response exitosa:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "123",
  "name": "Margarita Premium",
  "price": 15.00,
  "url": "https://example.com/images/margarita-premium.jpg",
  "description": "Pizza clásica con ingredientes premium",
  "ingredients": [...]
}
```

**Códigos de estado:**
- ✅ `200 OK` - Actualizado con contenido en respuesta
- ✅ `204 No Content` - Actualizado sin contenido en respuesta
- ❌ `400 Bad Request` - Sintaxis incorrecta
- ❌ `404 Not Found` - Recurso no existe
- ❌ `422 Unprocessable Entity` - Error de validación
- ❌ `409 Conflict` - Conflicto de versión
- ❌ `401 Unauthorized` - No autenticado
- ❌ `403 Forbidden` - Sin permisos
- ❌ `500 Internal Server Error` - Error del servidor

---

### 3️⃣ DELETE - Eliminar un Recurso

**Endpoint:**
```
DELETE /v1/pizzas/{id}
```

**Request:**
```http
DELETE /v1/pizzas/123 HTTP/1.1
```

**Response exitosa:**
```http
HTTP/1.1 204 No Content
```

**Códigos de estado:**
- ✅ `204 No Content` - Eliminado exitosamente
- ❌ `400 Bad Request` - ID inválido
- ❌ `404 Not Found` - Recurso no existe
- ❌ `401 Unauthorized` - No autenticado
- ❌ `403 Forbidden` - Sin permisos
- ❌ `500 Internal Server Error` - Error del servidor

---

### 4️⃣ GET - Obtener un Recurso por ID

**Endpoint:**
```
GET /v1/pizzas/{id}
```

**Request:**
```http
GET /v1/pizzas/123 HTTP/1.1
Accept: application/json
```

**Response exitosa:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "123",
  "name": "Margarita",
  "price": 12.50,
  "url": "https://example.com/images/margarita.jpg",
  "description": "Pizza clásica italiana",
  "ingredients": [...]
}
```

**Códigos de estado:**
- ✅ `200 OK` - Recurso encontrado
- ❌ `400 Bad Request` - ID inválido
- ❌ `404 Not Found` - Recurso no existe
- ❌ `401 Unauthorized` - No autenticado
- ❌ `403 Forbidden` - Sin permisos
- ❌ `500 Internal Server Error` - Error del servidor

---

### 5️⃣ GET - Obtener Colección de Recursos

**Endpoint:**
```
GET /v1/pizzas
```

**Request:**
```http
GET /v1/pizzas HTTP/1.1
Accept: application/json
```

**Response exitosa:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "id": "123",
    "name": "Margarita",
    "price": 12.50,
    "ingredients": [...]
  },
  {
    "id": "124",
    "name": "Pepperoni",
    "price": 14.00,
    "ingredients": [...]
  }
]
```

**Response vacía:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

[]
```

**Códigos de estado:**
- ✅ `200 OK` - Colección obtenida (puede estar vacía)
- ❌ `400 Bad Request` - Parámetros inválidos
- ❌ `401 Unauthorized` - No autenticado
- ❌ `403 Forbidden` - Sin permisos
- ❌ `500 Internal Server Error` - Error del servidor

---

## 🔍 Query Strings y Filtrado

### Ejemplos de uso:

```
GET /v1/pizzas?name=carb&size=25&page=1&attributes=id,name,price
```

### Parámetros comunes:

| Parámetro | Descripción | Ejemplo |
|-----------|-------------|---------|
| `name` | Filtro por nombre | `name=margarita` |
| `page` | Número de página | `page=1` |
| `size` | Elementos por página | `size=25` |
| `sort` | Ordenamiento | `sort=price,asc` |
| `attributes` | Proyección de campos | `attributes=id,name,price` |

### Proyección de Campos

Similar a SQL:

```sql
-- Sin proyección
SELECT * FROM pizzas

-- Con proyección
SELECT id, name, price FROM pizzas
```

En REST:
```
GET /v1/pizzas?attributes=id,name,price
```

**Beneficios:**
- Reduce el tamaño de la respuesta
- Mejora el rendimiento
- Optimiza el uso de ancho de banda

---

## 📊 Códigos de Estado HTTP

### Respuestas Exitosas (2xx)

| Código | Descripción | Uso |
|--------|-------------|-----|
| `200 OK` | Petición exitosa | GET, PUT, PATCH con body |
| `201 Created` | Recurso creado | POST |
| `204 No Content` | Exitoso sin contenido | DELETE, PUT sin body |

### Errores del Cliente (4xx)

| Código | Descripción | Cuándo usar |
|--------|-------------|-------------|
| `400 Bad Request` | Sintaxis incorrecta | Formato JSON inválido |
| `401 Unauthorized` | No autenticado | Sin credenciales válidas |
| `403 Forbidden` | Sin permisos | Usuario autenticado pero sin acceso |
| `404 Not Found` | Recurso no existe | ID no encontrado |
| `409 Conflict` | Conflicto de estado | Recurso duplicado |
| `422 Unprocessable Entity` | Error de validación | Datos inválidos |

### Errores del Servidor (5xx)

| Código | Descripción | Acción |
|--------|-------------|--------|
| `500 Internal Server Error` | Error no controlado | Reintentar (retry) |
| `503 Service Unavailable` | Servicio no disponible | Reintentar más tarde |

📚 [Referencia completa de códigos HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)

---

## 🚨 Formato de Respuestas de Error

Estructura estándar:

```json
{
  "path": "/v1/pizzas/123",
  "message": "Pizza no encontrada",
  "status": 404,
  "timestamp": "2025-11-11T10:30:00Z"
}
```

### Ejemplo con validación:

```json
{
  "path": "/v1/pizzas",
  "message": "Error de validación",
  "status": 422,
  "timestamp": "2025-11-11T10:30:00Z",
  "errors": [
    {
      "field": "name",
      "message": "El nombre es obligatorio"
    },
    {
      "field": "ingredients",
      "message": "Debe incluir al menos un ingrediente"
    }
  ]
}
```

---

## 📚 Recursos Adicionales

### Documentación Oficial

- [MDN - Tipos MIME](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/MIME_types/Common_types)
- [MDN - Códigos de Estado HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- [OData Protocol](https://www.odata.org/) - Estándar para APIs RESTful

### Mejores Prácticas

1. **Versionado**: Siempre versiona tu API desde el inicio
2. **Documentación**: Usa OpenAPI/Swagger para documentar
3. **HATEOAS**: Considera incluir hipervínculos en respuestas
4. **Rate Limiting**: Implementa límites de peticiones
5. **CORS**: Configura correctamente el Cross-Origin Resource Sharing
6. **Autenticación**: Usa JWT, OAuth 2.0, o API Keys
7. **HTTPS**: Siempre en producción
8. **Logging**: Registra todas las peticiones para debugging

### Herramientas Recomendadas

- **Postman** / **Insomnia** - Testing de APIs
- **Swagger UI** - Documentación interactiva
- **Bruno** - Cliente REST open source
- **HTTPie** - Cliente HTTP por línea de comandos

---

## 🎓 Conceptos Clave para Recordar

1. **REST es un estilo arquitectónico**, no un protocolo
2. **Los recursos son sustantivos**, las acciones son verbos HTTP
3. **Stateless**: cada petición debe contener toda la información necesaria
4. **Cacheable**: las respuestas deben indicar si pueden ser cacheadas
5. **Idempotencia**: 
   - GET, PUT, DELETE → Idempotentes
   - POST → No idempotente
6. **Separación cliente-servidor**: permite evolución independiente

---

**Última actualización:** Noviembre 2025

**Autor:** Pedro - Consultor React & Arquitectura Web

---

> 💡 **Tip**: Usa esta guía como referencia rápida durante el desarrollo. Para implementaciones específicas, consulta la documentación de tu framework o biblioteca.
