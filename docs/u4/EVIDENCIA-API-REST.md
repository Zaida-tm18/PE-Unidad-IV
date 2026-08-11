# Evidencia de API REST — BIOPET

**Unidad IV — Actividad GA**

Este documento presenta el acceso a Swagger/OpenAPI y la evidencia de
ejecución de al menos tres endpoints reales del backend de BIOPET
(`AuthController`, `MascotaController`).
---

## 1. Acceso a Swagger

### 1.1 Ruta actual del proyecto (verificada en `application.yml`)

La versión anterior del `README.md` documentaba Swagger en
`http://localhost:8080/api/swagger-ui.html`, que es la ruta **por defecto**
de springdoc-openapi. Sin embargo, en `v0.9.0-rc` el proyecto **sobrescribe
esa ruta por defecto** en `Backend/src/main/resources/application.yml`:

```yaml
springdoc:
  api-docs:
    path: /api/openapi
  swagger-ui:
    path: /api/docs
    operationsSorter: method
```

Por lo tanto, la ruta real y vigente para acceder a Swagger UI en esta
versión es:

| Recurso | URL actual |
|---|---|
| **Swagger UI** | `http://localhost:8080/api/docs` |
| **OpenAPI JSON** (spec cruda) | `http://localhost:8080/api/openapi` |

Con el perfil TLS activo (`docker-compose.tls.yml`), ambas rutas también
están disponibles en `https://localhost:8443/api/docs` y
`https://localhost:8443/api/openapi`.

**Nota de corrección para el informe final:** al citar el acceso a Swagger
en el documento consolidado, debe usarse `/api/docs` (ruta real del
proyecto) y no `/api/swagger-ui.html` (ruta genérica desactualizada que aún
aparece en el inicio rápido del `README.md`).

### 1.2 Captura — pantalla principal de Swagger UI

![Pantalla de Swagger UI en http://localhost:8080/api/docs, mostrando los controladores agrupados](evidencias/swagger-home.png)
---

## 2. Endpoints ejecutados

### 2.1 Endpoint 1 — Login (obtener sesión autenticada)

| Campo | Detalle |
|---|---|
| Método | `POST` |
| Ruta | `/api/auth/login` |
| Objetivo | Autenticar a un usuario existente y establecer la sesión (JWT) en cookies `HttpOnly` para poder consumir el resto de endpoints protegidos. |
| Controlador / método | `AuthController.login(LoginRequest, ...)` |
| Body de ejemplo | `{ "email": "admin@biopet.dev", "password": "********" }` |
| Código HTTP esperado | `200 OK` (credenciales válidas) / `401 Unauthorized` (credenciales inválidas, `BadCredentialsException`) |
| Resultado | Respuesta `AuthSessionResponse` con el tiempo de expiración del token (`expiresIn`); el `access_token` y `refresh_token` viajan como cookies `Secure`, `HttpOnly`, `SameSite=Strict`, no en el cuerpo de la respuesta. |

Ejemplo de respuesta `200 OK`:

```json
{
  "expiresIn": 3600
}
```

![Ejecución de POST /api/auth/login en Swagger, mostrando el código 200, el body de respuesta y las cookies Set-Cookie en la respuesta HTTP](evidencias/endpoint-login.png)
---

### 2.2 Endpoint 2 — Listado paginado de mascotas

| Campo | Detalle |
|---|---|
| Método | `GET` |
| Ruta | `/api/mascotas` |
| Objetivo | Obtener el listado paginado de mascotas visibles para el usuario autenticado (todas si es `ADMIN`/`VETERINARIO`/`AUXILIAR`, solo las propias si es `DUENO`). |
| Controlador / método | `MascotaController.listar(Pageable, UserDetails)` → `MascotaService.listar(...)` |
| Autorización | `@PreAuthorize("hasAnyRole('ADMIN','VETERINARIO','AUXILIAR','DUENO')")` — requiere cookie de sesión válida del paso anterior. |
| Parámetros | `page`, `size`, `sort` (paginación estándar de Spring Data) |
| Código HTTP esperado | `200 OK` (autenticado) / `401 Unauthorized` (sin sesión) / `403 Forbidden` (rol sin permiso) |
| Resultado | `Page<MascotaResponse>` con el contenido, número de página, tamaño y total de elementos. |

Ejemplo de respuesta `200 OK` (estructura real de `MascotaResponse`):

```json
{
  "content": [
    {
      "id": 1,
      "duenioId": 5,
      "duenioNombre": "María Pérez",
      "nombre": "Firulais",
      "especie": "Canino",
      "raza": "Labrador",
      "fechaNacimiento": "2021-03-14",
      "activo": true,
      "creadoEn": "2026-01-10T15:22:00Z",
      "actualizadoEn": "2026-01-10T15:22:00Z"
    }
  ],
  "totalElements": 1,
  "totalPages": 1,
  "number": 0,
  "size": 20
}
```
![Ejecución de GET /api/mascotas en Swagger, mostrando el 200 OK y el JSON paginado](evidencias/endpoint-listar-mascotas.png)
---

### 2.3 Endpoint 3 — Crear una mascota

| Campo | Detalle |
|---|---|
| Método | `POST` |
| Ruta | `/api/mascotas` |
| Objetivo | Registrar una nueva mascota asociada a un dueño (`duenioId`). |
| Controlador / método | `MascotaController.crear(MascotaRequest)` → `MascotaService.crear(...)` |
| Autorización | `@PreAuthorize("hasAnyRole('ADMIN','VETERINARIO','AUXILIAR')")` — un `DUENO` no puede crear mascotas directamente. |
| Body de ejemplo | ver abajo (validado con Bean Validation: `@NotBlank`, `@NotNull`, `@PastOrPresent`) |
| Código HTTP esperado | `201 Created` (éxito) / `400 Bad Request` (validación fallida) / `403 Forbidden` (rol `DUENO`) |
| Resultado | Objeto `MascotaResponse` de la mascota creada, incluyendo `id` autogenerado. |

Body de la petición:

```json
{
  "duenioId": 5,
  "nombre": "Firulais",
  "especie": "Canino",
  "raza": "Labrador",
  "fechaNacimiento": "2021-03-14"
}
```
![Ejecución de POST /api/mascotas en Swagger, mostrando el 201 Created y el objeto creado con su id](evidencias/endpoint-crear-mascota.png)
---

### 2.4 Endpoint adicional (opcional, recomendado para la defensa) — Eliminar mascota

| Campo | Detalle |
|---|---|
| Método | `DELETE` |
| Ruta | `/api/mascotas/{id}` |
| Objetivo | Dar de baja lógica una mascota (marcar `activo = false`, no borrado físico). |
| Controlador / método | `MascotaController.eliminar(Long, UserDetails)` |
| Código HTTP esperado | `204 No Content` (éxito) / `404 Not Found` (id inexistente) |
| Resultado | Sin cuerpo de respuesta; verificar luego con `GET /api/mascotas/{id}` que `activo=false`. |

![Ejecución de DELETE api/mascotas/{id} en Swagger](evidencias/endpoint-eliminar-mascota.png)
---

## 3. Resumen para la defensa oral

Para la defensa se debe mostrar, en vivo:

1. El sistema funcionando (`docker compose ps` con los 4 servicios
   `healthy`, y el frontend en `http://localhost:4200`).
2. Swagger UI abierto en la ruta real `http://localhost:8080/api/docs`.
3. Los 3 endpoints de la sección 2 ejecutados en vivo, mostrando código
   HTTP y respuesta.

## 4. Fuentes internas verificadas

- `Backend/src/main/resources/application.yml`
- `Backend/src/main/java/com/biopet/controller/AuthController.java`
- `Backend/src/main/java/com/biopet/controller/MascotaController.java`
- `Backend/src/main/java/com/biopet/dto/MascotaRequest.java`,
  `MascotaResponse.java`
