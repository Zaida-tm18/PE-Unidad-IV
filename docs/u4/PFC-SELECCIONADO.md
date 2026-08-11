# PFC Seleccionado: BIOPET

**Unidad IV — Actividad GA**
**Repositorio base:** `PFC-VET-ENTR3-v0.9.0-rc` (rama `main`)
**Versión analizada:** `v0.9.0-rc` — Tercera Entrega

---

## 1. ¿Qué es BIOPET?

BIOPET es un sistema web de gestión veterinaria desarrollado como Proyecto
Fin de Curso (PFC) de la asignatura Aplicaciones Web. Permite el registro y
autenticación de usuarios con distintos roles, y la administración
(CRUD) de mascotas, con autorización tanto por rol como por propietario de
la mascota.

El sistema no es una extensión de un producto existente: nace para
reemplazar procesos manuales (hojas de cálculo y mensajería) que la clínica
veterinaria usaba antes, según el problema levantado en la Entrega 1A del
proyecto (entrevistas a tres veterinarios y dos auxiliares, y encuestas a
quince dueños de mascotas).

## 2. Problema que resuelve

Antes de BIOPET, la gestión de la información clínica y administrativa de
una clínica veterinaria pequeña se hacía de forma manual y dispersa:

- Los datos de mascotas y dueños se llevaban en hojas de cálculo sin
  control de acceso ni historial de cambios.
- La comunicación entre veterinarios, auxiliares y dueños dependía de
  mensajería informal, sin trazabilidad.
- No existía un mecanismo centralizado para saber quién podía ver o
  modificar qué información (un dueño podía, en teoría, acceder a datos de
  mascotas que no eran suyas).

BIOPET centraliza esta información en una plataforma web con roles
diferenciados (`ADMIN`, `VETERINARIO`, `AUXILIAR`, `DUENO`), control de
acceso por rol y por propietario, y una API documentada que permite, a
futuro, integrar otros módulos (historial clínico, citas, facturación).

> Nota de trazabilidad del propio proyecto: el SRS original (Entrega 1A)
> planteaba ASP.NET Core/C# y Bootstrap como stack. El equipo migró la
> implementación real a Java 21 + Spring Boot y Angular, decisión
> documentada en `ADR-002-pila-tecnologica.md`. Los requisitos de negocio no
> cambiaron; solo la forma de implementarlos.

## 3. Alcance funcional actual (v0.9.0-rc)

Implementado:

- Registro, login, refresh y logout de usuarios (JWT en cookie).
- Gestión de usuarios con roles.
- CRUD de mascotas con autorización por rol y por dueño.
- Resumen agregado de mascotas activas por especie (vía función SQL).
- Gestión de vacunas, citas y consultas asociadas a una mascota.
- Consumo de una API externa de especies (`ExternalApiController`).

Pendiente (declarado explícitamente por el propio equipo en el SRS):
historial clínico completo, telemetría IoT, recomendaciones asistidas,
facturación y reportes avanzados.

## 4. Stack tecnológico

| Componente | Detalle |
|---|---|
| Backend | Java 21, Spring Boot 3.2.12 |
| Seguridad | Spring Security 6, JWT (jjwt 0.12.6, HMAC-SHA256), cookies `HttpOnly`/`Secure`/`SameSite=Strict` |
| Persistencia | Spring Data JPA + Hibernate, PostgreSQL 16, Flyway (migraciones versionadas) |
| Caché / revocación de tokens | Redis 7 (Spring Data Redis + Spring Cache) |
| Documentación de API | springdoc-openapi 2.5.0 (Swagger UI) |
| Cobertura de pruebas | JaCoCo 0.8.12 |
| Frontend | Angular 17.3, TypeScript 5.4, componentes standalone |
| Orquestación | Docker Compose + `Makefile` |

## 5. Arquitectura general

BIOPET sigue una arquitectura cliente-servidor de tres capas:

```
Angular (SPA, puerto 4200)
        │  HTTPS/HTTP + cookie JWT
        ▼
Spring Boot API REST (puerto 8080 / 8443 con TLS)
        │  Spring Data JPA / Hibernate
        ▼
PostgreSQL 16  ── Redis 7 (caché y revocación de tokens)
```

Todo el stack se orquesta con Docker Compose (`docker-compose.yml` para el
entorno HTTP y `docker-compose.tls.yml` como overlay para HTTPS/TLS 1.3).
El detalle a nivel de contenedores y de componentes internos del backend
está modelado con diagramas C4 en el propio repositorio
(`docs/diagrams/c4-contenedores/` y
`docs/diagrams/c4-componentes-backend/C4-L3-backend.md`).

Dentro del backend, el código se organiza siguiendo el patrón MVC de Spring,
con una separación adicional de capas típica de aplicaciones Spring Boot
"enterprise":

```
com.biopet
 ├─ controller/   → capa web: recibe la petición HTTP, valida entrada, delega
 ├─ service/      → lógica de negocio, transacciones
 ├─ repository/   → acceso a datos (Spring Data JPA)
 ├─ entity/       → entidades JPA mapeadas a las tablas de PostgreSQL
 ├─ dto/          → objetos de entrada/salida (Request/Response)
 ├─ security/     → JWT, filtros, blacklist de tokens, rate limiting de login
 ├─ config/       → configuración de Spring (seguridad, CORS, OpenAPI, etc.)
 ├─ exception/    → manejo centralizado de errores (`GlobalExceptionHandler`)
 └─ integration/  → cliente de la API externa de especies
```

Controladores identificados en el código: `AuthController`,
`UsuarioController`, `MascotaController`, `VacunaController`,
`CitaController`, `ConsultaController` y `ExternalApiController`.

## 6. Flujo MVC real de una petición autenticada

A diferencia de un diagrama MVC genérico, este flujo está descrito con los
nombres de clase y método reales del repositorio, usando como ejemplo el
listado paginado de mascotas (`GET /api/mascotas`):

1. **Angular → HTTP request.** `MascotaApiService.listar()` envía
   `GET /api/mascotas` incluyendo la cookie de autenticación (el JWT no se
   maneja manualmente en el frontend: viaja en una cookie `HttpOnly`).
2. **Filtro de seguridad.** `JwtAuthenticationFilter.doFilterInternal()`
   intercepta la petición antes de que llegue al controlador y extrae el
   token con `resolveToken(request)`.
3. **Contexto de seguridad.** Si el token es válido, Spring registra la
   autenticación del usuario en el `SecurityContextHolder`, disponible luego
   vía `@AuthenticationPrincipal` en el controlador.
4. **Controller.** `MascotaController.listar(Pageable, UserDetails)` recibe
   la petición ya autenticada, junto con los parámetros de paginación
   (`page`, `size`, `sort`), protegida además por
   `@PreAuthorize("hasAnyRole('ADMIN','VETERINARIO','AUXILIAR','DUENO')")`.
5. **Service.** `MascotaService.listar(Pageable, String email)` ejecuta la
   lógica de negocio dentro de una transacción de solo lectura, incluyendo
   la regla de negocio de que un `DUENO` solo puede ver sus propias
   mascotas.
6. **Repository (usuario).** `UsuarioRepository.findByEmailAndActivoTrue(email)`
   resuelve el usuario autenticado a partir del correo del JWT.
7. **Repository (mascotas).** Según el rol resuelto, el servicio invoca
   `MascotaRepository.findAllByActivoTrue(pageable)` (roles de clínica) o
   `MascotaRepository.findAllByDuenioIdAndActivoTrue(...)` (rol `DUENO`).
8. **PostgreSQL.** Hibernate traduce la operación del repositorio a SQL y la
   ejecuta contra la tabla `mascotas`.
9. **Mapeo a DTO.** Las entidades obtenidas se transforman a `MascotaResponse`
   mediante `map(this::toResponse)`, evitando exponer la entidad JPA
   directamente en la API.
10. **Serialización.** `MappingJackson2HttpMessageConverter` (Jackson)
    convierte el `Page<MascotaResponse>` a JSON.
11. **Respuesta.** Angular recibe `200 OK` con el contenido paginado en
    formato JSON y lo consume en el componente correspondiente.

Este flujo es el mismo patrón que siguen el resto de los endpoints CRUD del
sistema (vacunas, citas, consultas, usuarios), cambiando únicamente el
controller/service/repository involucrados.

## 7. Fuentes internas verificadas

- `README.md` (raíz del repositorio) — stack, arquitectura y flujo MVC.
- `Backend/src/main/java/com/biopet/controller/*.java` — controladores reales.
- `Backend/src/main/resources/application.yml` — configuración de springdoc/Swagger.
- `docs/requisitos/SRS.md` — problema de negocio y alcance funcional.

---

*Elaborado para la Unidad IV — Actividad GA, grupo BIOPET. Este documento
cubre exclusivamente la parte que me corresponde (descripción del PFC,
stack, arquitectura y flujo MVC); no incluye las revisiones críticas ni las
investigaciones de SOAP/REST o Jamstack/PWA/IA, que aportan Cajas y Panamá
(ver sección de espacio reservado en `RETROALIMENTACION-PFC.md`).*
