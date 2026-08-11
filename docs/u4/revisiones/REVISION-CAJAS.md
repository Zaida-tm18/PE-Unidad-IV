# REVISION-CAJAS — Revisión crítica del PFC BIOPET

- **Proyecto:** BIOPET — Sistema web de gestión veterinaria (Backend: Java 21, Spring Boot 3.2.12; Frontend: Angular 17; PostgreSQL 16 + Redis 7; Docker Compose).
- **Alcance de la revisión:** solo backend y soporte de despliegue/pruebas/documentación; el frontend se menciona únicamente cuando aporta evidencia (flujo de autenticación).
- **Método:** inspección directa de código, configuración, migraciones, tests y evidencias de `docs/` (no es revisión de ejecución). Cada afirmación referencia un archivo o fragmento concreto.
- **Commit de referencia de la revisión:** `eabf704` (main).

---

## 1. Arquitectura MVC

### ✅ Qué está bien
- Separación de capas real y consistente en todo el backend: `controller/` (orquestación HTTP), `service/` (lógica de negocio), `repository/` (acceso a datos), `dto/` (contratos), `entity/` (modelo JPA). Ejemplo típico: `MascotaController.java` solo recibe `@Valid @RequestBody`, aplica `@PreAuthorize` y delega en `MascotaService`; el servicio contiene las reglas de negocio (`resolverDuenio` valida que el asignado sea `ROLE_DUENO`, `MascotaService.java:122-130`) y el repositorio aísla las consultas (`MascotaRepository.java`).
- Los controladores no contienen lógica de negocio: en los 7 controladores revisados (`AuthController`, `MascotaController`, `CitaController`, `ConsultaController`, `VacunaController`, `UsuarioController`, `ExternalApiController`) no hay SQL, reglas de permisos de datos ni transformaciones de dominio dentro de la capa HTTP; solo `AuthController` manipula cookies, lo cual es responsabilidad de presentación aceptable.
- Transacciones delimitadas en la capa de servicio con `@Transactional` / `@Transactional(readOnly = true)` (p. ej. `CitaService.java:46,64,80`) y `open-in-view: false` en `application.yml:13`, evitando el anti-patrón Open Session in View.

### ⚠️ Qué limitación o problema existen
- La frontera de permisos está **partida en dos capas distintas y de forma distinta por módulo**: el control por rol vive en `@PreAuthorize` del controlador y el control por datos vive en el servicio — y `CitaService.java:20-32` lo documenta como decisión. El resultado es que **un módulo olvidó aplicar la regla de datos**: `ConsultaService.listar()` (`ConsultaService.java:39-42`) para `ROLE_DUENO` ejecuta `consultaRepository.findAllByActivoTrue(pageable)` — el **mismo listado global sin filtrar que usan staff/ADMIN** — y el propio comentario lo admite ("filtrado real de propiedad se aplica en buscar()"). Consecuencia: un dueño con token válido ve las consultas clínicas de **todas** las mascotas de la clínica (fuga de datos clínicos entre propietarios; escalación horizontal de privilegios). Contrasta con `MascotaService.listar` (`MascotaService.java:38-40`) y `VacunaService.listar` (`VacunaService.java:36-38`), que sí filtran por `findAllByMascota_Duenio_IdAndActivoTrue`.
- Falta el método de repositorio que el servicio debería usar: `ConsultaRepository.java` no tiene `findAllByMascota_Duenio_IdAndActivoTrue` (sí lo tienen `CitaRepository.java:12` y `VacunaRepository.java:16`).

### 🔧 Qué mejora recomendarías
1. Corregir la fuga: agregar a `ConsultaRepository` el método `Page<Consulta> findAllByMascota_Duenio_IdAndActivoTrue(Long duenioId, Pageable pageable)` y usarlo en la rama `ROLE_DUENO` de `ConsultaService.listar()`. Prioridad alta: es un fallo de autorización por datos.
2. Unificar la estrategia de control de acceso en una única política: o bien siempre en el servicio (recomendado, porque los filtros por datos dependen del usuario autenticado), dejando `@PreAuthorize` solo como primera barrera por rol; o bien un helper compartido tipo `AccessPolicyService.verificarAccesoGlobalORPropietario(usuario, mascota)` para eliminar la duplicación de `verificarAcceso`/`verificarPropiedad`/`tieneAccesoGlobal` entre `MascotaService`, `CitaService`, `ConsultaService` y `VacunaService`.

---

## 2. Organización del backend

### ✅ Qué está bien
- Estructura por capas clara y uniforme bajo `com.biopet`: `config/`, `controller/`, `dto/`, `entity/`, `exception/`, `integration/`, `repository/`, `security/`, `service/`. Convenciones de nombres consistentes: `XxxController`, `XxxService`, `XxxRepository`, `XxxRequest`/`XxxResponse` (records), `XxxException`.
- Separación entre DTOs de entrada y de salida (no se exponen entidades JPA directamente en la API), y uso de records inmutables (`MascotaRequest.java`, `MascotaResponse.java`).
- Integración con terceros aislada en su propio paquete `integration/` (`ExternalApiClient`, `ExternalApiService`, `RestTemplateConfig`), lo que reduce el acoplamiento del resto del dominio.
- Migraciones Flyway versionadas y **reservadas por integrante** con convención documentada en los propios archivos (`V2__citas.sql:2-3`), lo que evitó conflictos de esquema en trabajo paralelo.

### ⚠️ Qué limitación o problema existen
- **Metadatos del proyecto desactualizados**: `Backend/pom.xml:17` describe el artefacto como "Entrega 1B - JWT + Spring Data JPA", cuando el proyecto ya tiene citas, consultas, vacunas e integración externa; el `README.md:1` se presenta como "Tercera Entrega v0.9.0-rc" y referencia otro repositorio (`JirachinG19Stdio/PFC-VET-ENTR3-v0.9.0-rc`, `README.md:66`). Para un trabajo académico es aceptable, pero dificulta la trazabilidad y la reutilización del repositorio.
- `dto/` es un paquete plano con 20+ clases; al crecer el dominio convendrá agrupar por feature. Es cosmético hoy, pero la tendencia ya se ve en nombres genéricos como `ExternalApiResponse.java`.
- Sin archivos de convención compartidos (estilo/checkstyle) ni reglas de lint en el build; la consistencia depende de la disciplina del equipo.

### 🔧 Qué mejora recomendarías
- Actualizar `<description>` en `pom.xml`, el `README.md` (nombre de repositorio, versión) y el `version` del OpenAPI para que reflejen el estado real.
- Agregar checkstyle/spotless con una regla mínima al build (`verify`) para fijar las convenciones de formato e imports.
- Documentar en `docs/adr/` una convención de agrupación de DTOs si el dominio sigue creciendo.

---

## 3. API REST

### ✅ Qué está bien
- Diseño orientado a recursos y consistente en lo esencial: `/api/mascotas`, `/api/citas`, `/api/consultas`, `/api/vacunas`, `/api/usuarios`, con sustantivos en plural y verbos HTTP correctos (GET para lectura, POST para creación, PUT para actualización completa, DELETE para borrado lógico).
- Uso correcto de códigos de estado: `201 Created` en creación (`MascotaController.java:44`), `204 No Content` en eliminación (`MascotaController.java:58`), `404` para recurso inexistente, `409` para email duplicado (`GlobalExceptionHandler.java:24-27`), `422 Unprocessable Entity` para errores de validación (`GlobalExceptionHandler.java:39-59`), `429` con cabecera `Retry-After` para rate limiting (`GlobalExceptionHandler.java:72-85`).
- Errores estandarizados con **RFC 7807 Problem Details** (`ProblemDetailFactory.java`, `ProblemType.java`) — incluye `type`, `title`, `status`, `detail`, `instance`; los tests lo verifican (`MascotaControllerTest.java:100-105`).
- Paginación uniforme con `Pageable`/`Page<T>` de Spring Data en todos los listados.
- Validación de rutas con regex en varios controladores (`MascotaController.java:35` `@PathVariable Long id` bajo `{id:\d+}`) que evita disparar errores de conversión.

### ⚠️ Qué limitación o problema existen
- **Inconsistencia en los patrones de ruta entre módulos**: `MascotaController`, `CitaController` y `UsuarioController` usan `{id:\d+}`; `ConsultaController.java:31,43,50` y `VacunaController.java:38,50,57` usan `{id}` sin regex. La API responde de forma distinta según el módulo ante el mismo input inválido (p. ej. `/api/consultas/abc` llega al `MethodArgumentTypeMismatchException` → 400, mientras `/api/mascotas/abc` ni siquiera matchea → otro código). Es un detalle, pero rompe la uniformidad del contrato.
- **Matriz de permisos por módulo sin criterio documentado**: DELETE de citas solo `ADMIN` (`CitaController.java:51`), mientras DELETE de consultas y vacunas permite `ADMIN/VETERINARIO/AUXILIAR` (`ConsultaController.java:51`, `VacunaController.java:58`); un veterinario puede borrar una consulta clínica pero no una cita. Puede ser intencional, pero no hay ADR ni test que lo justifique, lo que invita a errores futuros.
- El endpoint externo rompe la convención: `GET /api/externa/especies?especie=dog` (`ExternalApiController.java:23`) es un servicio de acción con parámetro de consulta libre, no un recurso; además `@RequestParam String especie` sin `@NotBlank`/`@Size` admite peticiones vacías que se traducen en llamadas inútiles a la API externa.
- No hay versionado de API (`/api/v1/...`): correcto para un PFC, pero conviene decidirlo antes de que existan consumidores externos.
- El `RegistroRequest.rol` aparece como campo obligatorio del contrato público (ver tema 5) — la API documenta algo que no ejecuta.

### 🔧 Qué mejora recomendaría
- Unificar `{id:\d+}` (o un `@Pattern` en `@PathVariable`) en todos los controladores y añadir un test de contrato que recorra todos los endpoints.
- Documentar en `docs/adr/` la matriz de permisos por recurso y verbo, y alinearla entre módulos (definir quién puede borrar qué y por qué).
- Mover la consulta externa a un recurso con semántica propia (p. ej. `GET /api/especies/{especie}/informacion`) y validar el parámetro con `@NotBlank @Size(max=50)`.
- Agregar `@Operation`/`@Tag` de springdoc en cada endpoint (ver tema 4) para fijar el contrato visible.

---

## 4. Swagger / OpenAPI

### ✅ Qué está bien
- springdoc-openapi 2.5.0 configurado con rutas propias y públicas: `/api/openapi` (JSON) y `/api/docs` (Swagger UI) (`application.yml:52-57`), y permitidas sin autenticación en `SecurityConfig.java:73-86`.
- `OpenApiConfig.java` declara el esquema de seguridad `bearerAuth` (HTTP bearer JWT) y lo aplica globalmente con `addSecurityItem`.
- Existe prueba automatizada que verifica que Swagger UI se sirve y que el documento OpenAPI describe rutas reales de la app (`SwaggerUiTest.java:31-55`).

### ⚠️ Qué limitación o problema existen
- **No hay ninguna anotación de documentación en el código**: `grep` de `@Operation|@Schema|@Tag` en `Backend/src/main/java` → 0 resultados. Todos los títulos, descripciones y ejemplos son generados automáticamente por springdoc; ningún endpoint tiene `summary`, `description`, ejemplo de payload ni códigos de error documentados.
- **La descripción de la API está desactualizada respecto al código real**: `OpenApiConfig.java:22` dice "…autenticación JWT y CRUD de mascotas", pero la API ya tiene citas, consultas, vacunas, usuarios, refresh/logout e integración con API externa.
- La doc de seguridad solo muestra `bearer`; la autenticación real también funciona con cookies HttpOnly (`JwtAuthenticationFilter.java:78-90` resuelve primero cookie y luego header), así que el consumidor de Swagger no ve el mecanismo que el frontend de Angular realmente usa.
- Campos muertos del contrato aparecen en la documentación como obligatorios (p. ej. `rol` en `RegistroRequest`), lo que confunde a quien consume la API desde Postman/Swagger (los `docs/postman/*.json` lo arrastran).

### 🔧 Qué mejora recomendable
- Añadir `@Tag(name = "Mascotas")`, `@Operation(summary, description)` y `@Schema(example = ...)` en al menos un endpoint representativo por controlador, y documentar los códigos de error (400/401/403/404/409/422/429) de forma uniforme.
- Actualizar `Info.description` de `OpenApiConfig.java` y la versión; documentar la autenticación por cookies (mencionar que el navegador envía la cookie automáticamente con `withCredentials`) además del bearer para integraciones script.
- Eliminar o corregir el campo `rol` de `RegistroRequest` (ver tema 5) para que el contrato documentado coincida con el comportamiento.

---

## 5. CRUD

### ✅ Qué está bien
- CRUD completo y uniforme para los 4 dominios (mascotas, citas, consultas, vacunas) más usuarios admin; eliminación **lógica** (`activo=false`) en todos los casos (`MascotaService.java:94-95`, `CitaService.java:102-103`, `ConsultaService.java:105-106`, `VacunaService.java:98-99`), verificada además por test (`MascotaControllerTest.java:744-751`).
- Validaciones de entrada con Bean Validation (`@Valid` + anotaciones en records DTO): `MascotaRequest.java` (`@NotBlank`, `@Size`, `@PastOrPresent`), `ConsultaRequest.java`, `VacunaRequest.java`, `LoginRequest.java`. Errores agregados por campo en `GlobalExceptionHandler.java:40-58` (422 + `errors`).
- Manejo de errores centralizado y sin fugas de detalles internos: las excepciones de dominio (`RecursoNoEncontradoException`, `EmailDuplicadoException`) se traducen a Problem Details con mensajes controlados.
- Reglas de integridad de negocio verificadas: `resolverDuenio`/`resolverVeterinario` validan que el usuario asignado tenga el rol correcto (`MascotaService.java:122-130`, `CitaService.java:116-125`), con tests para los casos negativos (`MascotaControllerTest.java:489-583`).

### ⚠️ Qué limitación o problema existen
- **Contratos con campos obligatorios que el servidor ignora**:
  - `RegistroRequest.java:13` exige `@NotNull Rol rol` en el registro público, pero `AuthService.registrar()` lo ignora y fuerza `ROLE_DUENO` (`AuthService.java:56-62`). El propio proyecto lo documenta en un javadoc (`CitaRequest.java:9-14`): es un campo muerto que además parece una vía de escalada a `ROLE_ADMIN` para quien no lea el código — hoy inocuo por el hardcode, pero peligroso si alguien "arregla" el servicio sin revisar.
  - `CitaRequest.java:19` exige `estado` en el POST, pero `CitaService.crear()` lo ignora y fuerza `PROGRAMADA` (`CitaService.java:73`).
- **Validaciones incompletas en dominio de negocio**: `CitaRequest.fechaHora` no tiene `@Future` (se puede agendar una cita en el pasado); no hay control de solapamiento/duplicado de citas (misma mascota en el mismo horario se agenda dos veces); `fechaNacimiento` de mascota se valida con `@PastOrPresent` pero no hay rango mínimo.
- El CRUD de usuarios no permite reactivar cuentas eliminadas (borrado lógico sin reversa por la API), y `UsuarioService.eliminar` no impide que un admin se auto-desactive (`UsuarioService.java:92-98`), dejando la app sin administradores activos.

### 🔧 Qué mejora recomendable
- Eliminar `rol` de `RegistroRequest` (o documentarlo con `@Schema(accessMode = READ_ONLY)`) y eliminar `estado` de `CitaRequest` en POST (crear `CitaCrearRequest` sin estado o volverlo opcional y exigir `@NotNull` solo en `PUT`).
- Añadir `@Future` a `CitaRequest.fechaHora` y una restricción de unicidad `UNIQUE (mascota_id, fecha_hora)` (ver tema 8) más su validación en servicio.
- Proteger la auto-desactivación del admin y considerar un endpoint de reactivación (`POST /api/usuarios/{id}/reactivar`).

---

## 6. Autenticación y roles

### ✅ Qué está bien
- **JWT robusto**: jjwt 0.12.6 con HMAC-SHA256, validación de largo mínimo de clave de 256 bits en `JwtService.java:31-33`, claims `iss`/`aud` verificados en `extractClaims` (`JwtService.java:67-75`), `sub`, `email`, `rol` y `typ` (distingue access vs refresh).
- **Doble token**: access (1 h) + refresh (7 días), con blacklist de `jti` en Redis para revocar tras logout (`TokenBlacklistService.java`), y verificación de revocación en el filtro (`JwtAuthenticationFilter.java:54`). El diseño sigue el ADR-003 (`docs/adr/ADR-003-jwt-redis.md`).
- **Cookies HttpOnly seguras**: `JwtCookieService.java:71-78` emite cookies `httpOnly`, `secure`, `SameSite=Strict`, con path restringido para el refresh (`/api/auth`) — mitigación real de XSS/CSRF; el frontend nunca toca el token (`frontend/src/app/core/auth.service.ts:22-28`).
- Sesiones stateless (`SessionCreationPolicy.STATELESS`, `SecurityConfig.java:57`), `BCryptPasswordEncoder(12)` (`SecurityConfig.java:122`), `@EnableMethodSecurity` y `@PreAuthorize` en todos los endpoints protegidos con matriz de roles por recurso.
- Control de acceso por datos en servicio (propiedad del recurso) más allá del rol: `verificarPropiedad` (`MascotaService.java:116-120`), `verificarAccesoLectura/Escritura` (`CitaService.java:131-141`), `verificarAcceso` (`VacunaService.java:135-139`).
- Auditoría de eventos de autenticación con formato estructurado y sanitizado (`AuthenticationAuditService.java:57-76`), y rate limiting de login por IP con bloqueo temporal y `Retry-After` (`LoginRateLimiterService.java`).

### ⚠️ Qué limitación o problema existen
- **El rate limiter de login es en memoria** (`ConcurrentHashMap`, `LoginRateLimiterService.java:23`): no funciona en despliegues multi-instancia, no usa Redis a pesar de estar disponible, y no respeta `X-Forwarded-For` (tras un proxy/reverse proxy todos los clientes comparten la IP → el límite se dispara globalmente o se evade fácilmente). Además solo protege `/login`: `/registro` y `/refresh` no tienen límite (abuso de creación de cuentas).
- **El refresh token no se rota**: `AuthService.refresh()` reutiliza el mismo refresh token (`AuthService.java:124`). Si un token se filtra, es válido durante los 7 días completos sin detección de reuso; la práctica recomendada (OWASP) es rotar y emitir uno nuevo en cada refresh.
- **Credencial de administrador conocida en el código y en el seed**: `DataInitializer.java:21` crea `admin@biopet.ec` con contraseña `Admin123*` (además duplicada en `db/seed.sql:7-8`). En un despliegue descuidado es un backdoor conocido.
- `RegistroRequest.rol` como campo expuesto (ver tema 5): riesgo latente de escalada si alguien cambia el hardcode.
- La política de contraseña es solo `@Size(min = 8)` (`RegistroRequest.java:12`); sin exigencia de complejidad.

### 🔧 Qué mejora recomendable
- Mover el rate limiting a Redis con `INCR`+`EXPIRE` (o Lua), basarse en la IP real respetando `X-Forwarded-For` solo si el proxy es confiable, y ampliarlo a `/registro` y `/refresh`.
- Rotar el refresh token en cada `/refresh` y registrar el `jti` anterior como usado (detección de reuso → invalidar sesión).
- Convertir la credencial admin en configurable vía variable de entorno (`ADMIN_INITIAL_PASSWORD`) y/o forzar cambio de contraseña en el primer ingreso; nunca versionar credenciales de trabajo (ni el hash) en el repo.
- Reforzar la política de contraseñas (longitud mínima + complejidad) en `RegistroRequest` y `UsuarioRequest`.

---

## 7. Seguridad

### ✅ Qué está bien
- **Cabeceras de seguridad** completas y forzadas: HSTS con `preload`, CSP (`default-src 'self'`), `X-Content-Type-Options`, `X-Frame-Options DENY`, `Referrer-Policy no-referrer` (`SecurityConfig.java:58-67`), verificadas por `SecurityHeadersTest` y evidenciadas en `docs/mediciones/sec/A05-security-headers.md`.
- **CORS restringido**: orígenes desde variable de entorno, nunca `*` con `allowCredentials(true)`, métodos y cabeceras explícitos (`SecurityConfig.java:95-105`).
- **Inyección SQL**: acceso a datos vía JPA/derived queries y la única query nativa usa parámetros (`MascotaRepository.java:18`), con prueba dedicada `SqlInjectionSecurityTest.java` (evidencia en `docs/mediciones/sec/A03-injection.md`).
- **TLS**: perfil `tls` con TLS 1.3 y conector dual HTTP/HTTPS (`application-tls.yml`, `TomcatDualConnectorConfig.java`), keystore no versionado, evidencias de headers HTTPS en `docs/mediciones/sec/`.
- Secretos externalizados por variables de entorno con valores por defecto de desarrollo documentados como tales (`application.yml:6-8,37`), y **cuenta de BD de privilegios mínimos** para la app (`db/roles.sql`: solo `SELECT/INSERT/UPDATE/DELETE` + `USAGE` en secuencias, sin DDL).
- CSRF deshabilitado con justificación válida para JWT stateless en cookies `SameSite=Strict` (`SecurityConfig.java:55`); `/actuator/health` público pero el resto de endpoints de actuator exigen autenticación (`SecurityConfig.java:85`).

### ⚠️ Qué limitación o problema existen
- **Secreto real versionado en el repositorio**: `.env.example:49` contiene `APP_EXTERNAL_API_KEY=gsLqb9Xduk7fzthXAsym4HsHq0UeGn50032ypYbo` — una clave de API de API Ninjas con formato real, commitada en el historial (commit `b355659`), con el comentario "NINGÚN valor aquí es un secreto real" que no se cumple para este caso. Debe considerarse comprometida y rotarse.
- **Y además está mal cableada**: la propiedad `app.external-api.key` no existe en `application.yml` (se verificó por grep: solo aparece en `ExternalApiClient.java:27` como `@Value` con default vacío). Como `@Value` no aplica *relaxed binding* desde variables de entorno, `APP_EXTERNAL_API_KEY` nunca llega a la app → el header `X-Api-Key` se envía vacío y la integración falla con 401/502 en cualquier despliegue que no defina la propiedad por otra vía. Doble problema: clave filtrada **e** inefectiva.
- JWT secret con valor por defecto versionado (`application.yml:37`) — aceptable para desarrollo si se documenta, pero el default debería ser *fail-fast* en producción (negar arranque si no hay `JWT_SECRET` definido).
- Sin límites globales de abuso: cualquier usuario autenticado puede golpear `GET /api/externa/especies` y agotar la cuota de la API externa (solo existe el rate limit de login).
- El filtro JWT degrada en silencio: ante un token inválido no autentica pero continúa la cadena (`JwtAuthenticationFilter.java:71-73`) — correcto para no romper endpoints públicos, pero la respuesta final para rutas protegidas depende del entry point; no es un fallo, pero conviene un test que lo fije.

### 🔧 Qué mejora recomendable
- Rotar la clave de API Ninjas, eliminarla del historial (reescribir el commit con `git filter-repo` o, mínimo, purgarla del HEAD) y dejar SOLO el placeholder en `.env.example`.
- Declarar en `application.yml` el bloque `app.external-api: key: ${APP_EXTERNAL_API_KEY:}` (y `base-url`, `cache-ttl-seconds`) para cerrar el cableado.
- Hacer fail-fast: validar al arranque que `JWT_SECRET`, `DB_APP_PASSWORD` y `APP_EXTERNAL_API_KEY` no sean los valores por defecto cuando `SPRING_PROFILES_ACTIVE` incluya `prod`.
- Rate limit de uso de la API externa por usuario (Redis), y un test de seguridad de contrato que verifique el comportamiento 401/502 del endpoint externo.

---

## 8. Base de datos

### ✅ Qué está bien
- PostgreSQL 16 con **Flyway** y 4 migraciones ordenadas (`V1__schema_inicial.sql` → `V4__vacunas.sql`) con integridad referencial real: FKs (`fk_mascotas_duenio`, `fk_citas_mascota`, `fk_citas_veterinario`, etc.), CHECK constraints (`chk_usuarios_rol`, `chk_citas_estado`), índices en columnas de FK y en `activo`, y triggers `set_actualizado_en()` para auditoría de timestamps.
- `ddl-auto: validate` (`application.yml:12`): Hibernate valida el esquema contra las entidades en cada arranque, detectando desalineaciones.
- Modelado de entidades coherente con las migraciones (`@Column(nullable = false, length=…)`, `@ManyToOne(fetch = LAZY)`, timestamps con `@PrePersist`/`@PreUpdate` en `Usuario.java:44-56`, `Mascota.java:46-56`).
- **Privilegios mínimos por rol de BD**: la cuenta de aplicación `biopet_app` solo tiene CRUD sobre tablas del dominio y `ALTER DEFAULT PRIVILEGES` para que las migraciones futuras hereden los mismos permisos (`db/roles.sql:18-34`). Esto es una buena práctica poco común en PFCs.
- Pruebas de integración contra PostgreSQL real con Testcontainers (no solo H2): `ResumenEspeciesIntegrationTest.java:31` y `TriggerActualizadoEnIntegrationTest.java` arrancan un contenedor `postgres:16-alpine`, corren Flyway y aplican el procedimiento real `fn_resumen_mascotas_por_especie.sql`.

### ⚠️ Qué limitación o problema existen
- **Doble fuente de verdad del esquema**: `db/schema.sql` duplica manualmente la V1 para que Docker pueda inicializar sin esperar al backend, y su propio encabezado advierte que debe mantenerse a mano ("este archivo debe actualizarse manualmente… o el arranque vía Docker quedará desfasado"). **Ya está desfasado**: no contiene `citas`, `consultas` ni `vacunas` (solo `usuarios`/`mascotas`). Hoy "funciona" porque Flyway completa el esquema al arrancar, pero es un accidente a la espera de un problema (p. ej. si alguien desactiva Flyway o usa otro despliegue).
- **Sin unicidad para citas**: no hay `UNIQUE (mascota_id, fecha_hora)` ni `exclusion` de horarios → doble reserva posible a nivel de BD (ver tema 5).
- **Inconsistencias entre migraciones**: `V3__consultas.sql` no crea el trigger `trg_consultas_actualizado_en` (sí lo crean V2 y V4) ni índice sobre `activo`, a diferencia de las otras tablas. La entidad `Consulta` actualiza `actualizadoEn` por JPA (`@PreUpdate`), así que hoy no rompe, pero el esquema es desigual.
- Las FKs hacia `usuarios` (`veterinario_id`, `duenio_id`) no restringen rol en BD: la validación de "debe ser VETERINARIO/DUENO" vive solo en la capa de servicio (`CitaService.java:120-123`); un INSERT directo o un bug de servicio puede asignar cualquier usuario.
- `db/schema.sql` + `db/seed.sql` + `DataInitializer` dejan credenciales y datos de desarrollo en la ruta de inicialización de cualquier despliegue que copie el docker-compose (ver tema 7).

### 🔧 Qué mejora recomendable
- Eliminar la duplicación: o bien usar únicamente Flyway (`spring.flyway.locations` + `default-schema`) y hacer que `db/schema.sql` desaparezca del `docker-entrypoint-initdb.d`, o bien generar el init script desde las migraciones (p. ej. `flyway-maven-plugin` con goal `migrate` a un esquema "plantilla" y export). Prioridad media-alta por el riesgo de drift.
- Añadir `UNIQUE (mascota_id, fecha_hora)` en `citas` (migración V5) + validación amigable en servicio.
- Alinear V3 con las demás (trigger de `actualizado_en` e índice `idx_consultas_activo`) en la misma V5.
- Evaluar CHECKs de rol en BD (FKs compuestas o triggers) si se quiere defensa en profundidad; al menos documentar en ADR que la validación de rol es responsabilidad de la capa de servicio.

---

## 9. Redis

### ✅ Qué está bien
- **Tres usos reales y diferenciados**:
  1. **Caché de respuestas** vía Spring Cache con TTL de 5 min (`application.yml:29-33`): `MascotaService.listar` (`@Cacheable` con clave por usuario+página, `MascotaService.java:32`) e invalidación con `@CacheEvict(allEntries=true)` en escrituras (`MascotaService.java:54,69,86`); mismo patrón en `ConsultaService`.
  2. **Blacklist de tokens JWT** con TTL = vida restante del token (`TokenBlacklistService.java:18-23`).
  3. **Cache-aside de la API externa** (TTL 10 min) en `ExternalApiService.java:34-59`, con evidencia de rendimiento real: 1363 ms → 21 ms (~64x) en `docs/u4/evidencias/fred/redis-cache-comparacion.md`.
- Mediciones honestas y reproducibles: hit ratio del caché de listado verificado con `DBSIZE`/`KEYS`/TTL durante una corrida k6 completa (`docs/mediciones/perf/REPORT.md:23-29`).
- Escrituras de caché no fatales: `guardarEnCache` captura excepciones y degrada a sin-caché (`ExternalApiService.java:62-69`).

### ⚠️ Qué limitación o problema existen
- **Subutilización evidente**: el rate limiter de login es la funcionalidad ideal para Redis y está en memoria (`LoginRateLimiterService.java:23`), mientras Redis ya está en la infraestructura y se usa para blacklist (ver tema 6).
- **Lecturas de Redis sin protección**: en `ExternalApiService.obtenerInfoEspecie` el `redisTemplate.opsForValue().get(cacheKey)` (`ExternalApiService.java:38`) no está en try/catch (a diferencia de la escritura): si Redis cae, el endpoint externo responde 500 en lugar de degradar a llamada directa. Igual ocurre en `TokenBlacklistService.isRevoked` desde el filtro: `RedisConnectionFailureException` propagaría como 500 a toda la API autenticada — el ADR-003 lo acepta como "error de infraestructura", pero no hay un mecanismo de degradación ni mensaje amigable.
- Solo se cachean los listados de mascotas y consultas; `CitaService.listar` y `VacunaService.listar` (que tienen el mismo perfil de consulta frecuente) no usan caché. El `CacheEvict(allEntries=true)` global es simple pero invalida entradas de todos los usuarios ante cualquier escritura (aceptable a esta escala, no escala bien).
- Redis en docker-compose sin `requirepass` ni config de persistencia (`docker-compose.yml:22-31`) — correcto para dev, debe documentarse como límite para producción. No se observan credenciales ni ACL para entornos no locales.

### 🔧 Qué mejora recomendable
- Mover rate limiting y contadores a Redis (operaciones atómicas `INCR`+`EXPIRE`), aprovechando la infraestructura ya existente.
- Envolver las lecturas de caché/blacklist en try/catch con degradación definida (p. ej. "si Redis no responde en X ms, seguir sin caché y loguear warning"), y configurar timeouts de conexión a Redis en `application.yml`.
- Extender `@Cacheable` a los listados de citas y vacunas con el mismo patrón, y documentar la política de evicción.
- En producción: `requirepass`/ACLs y política de evicción explícita (`maxmemory-policy` documentada en `docs/mediciones/redis/config-maxmemory.txt` para dev).

---

## 10. Integración con API externa

### ✅ Qué está bien
- **Cliente desacoplado y resiliente en lo básico**: `ExternalApiClient` (componente) + `ExternalApiService` (servicio) + `RestTemplateConfig` (bean) separados, con URL y clave externalizadas (`ExternalApiClient.java:24-28`).
- **Timeouts explícitos**: connect 3 s / read 5 s (`RestTemplateConfig.java:16-18`).
- **Taxonomía de errores completa**: distingue 429, 4xx, 5xx y timeout de conexión, mapeándolos a `ExternalApiException` con contexto (`ExternalApiClient.java:48-56`), y el `GlobalExceptionHandler.java:94-100` los convierte en **502 con detalle genérico** (no filtra internos al cliente).
- **Cache-aside con TTL** para reducir llamadas y cumplir la cuota (evidencia ~64x en `docs/u4/evidencias/fred/redis-cache-comparacion.md`), con mapeo DTO defensivo (`@JsonIgnoreProperties(ignoreUnknown = true)`, `ExternalApiClient.java:59-71`).
- Endpoint protegido por autenticación y rol (`ExternalApiController.java:22`), y lectura de caché que devuelve origen "cache" vs "api-ninjas" (útil para depurar y para el informe).

### ⚠️ Qué limitación o problema existen
- **Bug de cableado de la clave** (ya descrito en el tema 7): sin `app.external-api.key` en `application.yml`, el header `X-Api-Key` sale vacío; el flujo feliz de la evidencia (`redis-cache-comparacion.md`) no es reproducible desde una copia limpia del repo sin inyectar la propiedad por fuera del código. Es la debilidad más grave de este módulo.
- **Sin reintentos ni circuit breaker**: un 5xx o timeout de la API externa se convierte directamente en 502 sin reintento con backoff, y no se honra `Retry-After` en el 429 (que sí se detecta). Con la cuota gratuita de API Ninjas, el 429 será frecuente.
- **Cache con huecos**: solo se cachea el primer resultado; resultados vacíos (`resultados.isEmpty()`) lanzan excepción sin cachear (lo correcto), pero la lectura de caché sin try/catch convierte una caída de Redis en 500 (ver tema 9). Además, el TTL de 10 min está comentado como decisión ad-hoc en código (`ExternalApiService.java:23`), no documentada en ADR.
- Acoplamiento al proveedor: el DTO record `AnimalApiNinjasDto` vive dentro de `ExternalApiClient`; si mañana hay otro proveedor, habría que refactorizar. Menor.
- Sin límite de uso por usuario (cuota compartida entre todos los clientes autenticados — ver tema 7).

### 🔧 Qué mejora recomendable
- Corregir el cableado en `application.yml` y rotar la clave; añadir un test con mock de la API externa (p. ej. WireMock/MockWebServer) que cubra 200/404/429/5xx/timeout y cache hit/miss.
- Añadir reintento con backoff exponencial en 5xx/timeouts y, si la API provee `Retry-After`, respetarlo para 429; evaluar Resilience4j para circuit breaker cuando haya más de un consumidor.
- Envolver la lectura de caché en try/catch y degradar a llamada directa; documentar el TTL y la estrategia de caché en un ADR.
- Mover el DTO del proveedor a `integration/` con un modelo de dominio neutral (p. ej. `EspecieInfo`) para desacoplar del proveedor concreto.

---

## 11. Pruebas

### ✅ Qué está bien
- **109 tests, 0 fallos**, con umbral de cobertura automatizado: `mvn verify` falla si LINE/BRANCH/COMPLEXITY < 60 % (regla `BUNDLE` de jacoco en `Backend/pom.xml:169-199`); medición real registrada en `docs/mediciones/sec/jacoco-summary.md` (95.87 % lineas, 76.87 % ramas, 80 % complejidad).
- **Tipos de tests variados y de calidad**:
  - Integración HTTP completa con `@SpringBootTest` + MockMvc (login real, cookies, `@PreAuthorize`, Problem Details): `MascotaControllerTest.java`, `AuthControllerTest`, `CitaControllerTest`, `ConsultaControllerTest`, `VacunaControllerTest`, `UsuarioControllerTest`.
  - Tests de seguridad: `SqlInjectionSecurityTest`, `SecurityHeadersTest`, `JwtCookieAuthenticationTest`, `LoginRateLimiterServiceTest`, `JwtServiceTest`, `JwtCookieServiceTest`, `AuthenticationAuditServiceTest`, `TokenBlacklistService` (mock).
  - **Integración contra PostgreSQL real con Testcontainers** para el procedimiento almacenado y los triggers: `ResumenEspeciesIntegrationTest.java:30-48` (arranca contenedor, corre Flyway, aplica el `.sql` real) y `TriggerActualizadoEnIntegrationTest`.
  - Tests de configuración (perfil TLS): `TomcatDualConnectorConfigTest`.
- Casos negativos bien cubiertos: 403 por rol insuficiente, 404 por recurso ajeno/inexistente, 422 por validación, borrado lógico verificado (`MascotaControllerTest.java:744-751`).

### ⚠️ Qué limitación o problema existen
- **La fuga de datos de `ConsultaService.listar()` no tiene test**: en `ConsultaControllerTest` no existe ningún test de `GET /api/consultas` con rol `ROLE_DUENO` que verifique el filtrado por propiedad (solo se prueba `buscar` por id con 403). La regresión existe precisamente donde no hay cobertura de autorización de listados.
- **La integración externa no tiene tests**: no hay ningún test de `ExternalApiClient`/`ExternalApiService` (ni mock server); las excepciones 429/5xx/timeout y el cache hit/miss no están verificados (solo hay evidencia manual en `docs/u4/evidencias/`).
- La mayoría de los tests usan **H2 en modo PostgreSQL** (`application-test.yml:3`) y no ejercitan las migraciones Flyway ni los triggers/procs (los tests Testcontainers cubren solo 2 piezas). Riesgo de falsos positivos: SQL compatible H2 ≠ SQL PostgreSQL (el bug de `V3` sin trigger no se detectaría, y el cableado de la API externa tampoco porque se mockea).
- Sin tests de frontend (no hay `.spec.ts` en `frontend/src/app`) ni e2e; el flujo de refresh automático del interceptor (`http-error.interceptor.ts`) no tiene prueba.
- Exclusiones de jacoco a discutir: `dto/**` excluido íntegro (`pom.xml:149`) — las anotaciones de validación de los DTOs son parte del contrato y convendría al menos contar las anotaciones críticas; además el umbral es global (BUNDLE), por lo que un paquete entero (p. ej. `integration/`) puede quedar sin cubrir sin romper el build.

### 🔧 Qué mejora recomendable
- Añadir test de autorización de listados para `DUENO` en `ConsultaControllerTest` (dos dueños, dos mascotas, verificar que el listado solo contiene lo propio) — es el test que hoy habría capturado la fuga.
- Tests de `ExternalApiClient` con un servidor mock (WireMock o `MockRestServiceServer`) cubriendo 200/404/429/5xx/timeout y cache hit/miss con Redis (Testcontainers `redis`).
- Subir la proporción de tests de integración contra PostgreSQL real (los ya declarados Testcontainers) para validar migraciones y triggers.
- Agregar tests de componente en el frontend (al menos `auth.service` e interceptor) y considerar un umbral por paquete en jacoco (p. ej. ≥50 % en `integration/`).

---

## 12. Docker

### ✅ Qué está bien
- **Multi-stage build** del backend: imagen `maven:3.9-eclipse-temurin-21` para compilar y `eclipse-temurin:21-jre-alpine` para runtime (`Backend/Dockerfile:1-10`); el runtime final es pequeño y no incluye el toolchain Maven.
- **Digest fijados por SHA-256** para las imágenes base de Postgres 16 y Redis 7 (`docker-compose.yml:3,23`) — trazabilidad y reproducibilidad supply-chain; el Dockerfile del backend también fija digests de las dos etapas.
- **Healthchecks en los 4 servicios** (`docker-compose.yml:17-21,27-31,44-48,57-62`) y arranque ordenado con `depends_on: condition: service_healthy` (`docker-compose.yml:39-43,55-57`) — el backend espera a Postgres/Redis y el frontend al backend.
- Volumen persistente para Postgres (`postgres_data`), override TLS en archivo separado (`docker-compose.tls.yml`) que no toca el resto de servicios, y `.env` externo para configuración (`.env.example` documentado).
- Evidencias de verificación reproducible: `docs/u4/evidencias/fred/docker-verificacion.txt` (digests, servicios healthy) y `docs/mediciones/sec/raw/docker-compose-*`.

### ⚠️ Qué limitación o problema existen
- **Sin `.dockerignore`** en la raíz ni en `Backend/`/`frontend/` (verificado: no existe ninguno). El contexto de build incluye `.git/`, `docs/`, `k6/`, `node_modules/` si existiera, etc. → builds lentos e imágenes con datos del repo en capas de build (y cualquier archivo sensible no excluido entra al contexto).
- **El contenedor backend corre como root**: no hay `USER` en `Backend/Dockerfile`. Práctica recomendada: usuario no privilegiado en runtime.
- **Build del frontend no reproducible**: `frontend/Dockerfile:4` usa `npm install` copiando solo `package.json` (no `package-lock.json` ni `npm ci`) → las versiones de dependencias pueden variar entre builds; y las imágenes `node:20-alpine` / `nginx:1.25-alpine` no tienen digest fijado (a diferencia de las del backend/compose).
- Sin políticas de reinicio, límites de recursos (`mem_limit`/`cpus`) ni límites de logs en `docker-compose.yml`.
- La inicialización de BD vía `docker-entrypoint-initdb.d` (`docker-compose.yml:13-16`) acopla el esquema a Docker y duplica a Flyway (ver tema 8); en un despliegue que no use esos scripts, `biopet_app` ni siquiera existiría.

### 🔧 Qué mejora recomendable
- Crear `.dockerignore` (raíz, `Backend/`, `frontend/`) excluyendo `.git`, `docs`, `node_modules`, `target`, `dist`, `.env`, `k6`.
- Añadir `USER` no root en el Dockerfile del backend (p. ej. `eclipse-temurin` ya trae el usuario `temurin` en la JRE de Alpine) y `RUN chown` del jar.
- En el frontend: copiar `package-lock.json` y usar `npm ci`; fijar digests de `node:20-alpine` y `nginx:1.25-alpine`.
- Agregar `restart: unless-stopped` (o documentar la ausencia), `mem_limit`/`cpus` y `logging: max-size` a los servicios.

---

## 13. Mantenibilidad general

### ✅ Qué está bien
- Código legible y compacto: servicios de 150-160 líneas, nombres descriptivos, records inmutables, `@Getter/@Setter/@Builder` de Lombok (menos boilerplate). Los servicios más complejos documentan sus reglas con javadoc de calidad (p. ej. `CitaService.java:20-32`, `UsuarioService.java:16-22`, `CitaRequest.java:9-14`).
- **Documentación de decisiones de arquitectura**: ADRs (pila, JWT+Redis, PostgreSQL, despliegue, seguridad, acceso a datos) en `docs/adr/`, diagramas C4 (L1-L3) y DER generados, matriz de trazabilidad (`docs/trazabilidad/matriz.csv`) con script de validación (`scripts/validate-traceability.sh`).
- Evidencias de rendimiento/seguridad/usabilidad reproducibles y con commit asociado (`docs/mediciones/`, `docs/u4/evidencias/`), lo que facilita auditoría académica.
- Convención de migraciones reservadas por integrante y `CHANGELOG-REQ.md` para requisitos (gestión de cambios ordenada para un trabajo de equipo).

### ⚠️ Qué limitación o problema existen
- **Duplicación de lógica de autorización**: `tieneAccesoGlobal`/`verificarAcceso`/`verificarPropiedad`/`usuarioActual`/`resolverVeterinario` se repiten casi idénticos en `MascotaService`, `CitaService`, `ConsultaService` y `VacunaService` (4 copias del mismo `usuarioRepository.findByEmailAndActivoTrue` + 4 variantes de chequeo de propiedad). Es la raíz de la inconsistencia del tema 1: la regla se copió y en una copia se olvidó el filtro.
- `toResponse`/`resolverX` duplicados entre servicios (cada uno mapea a su propio DTO con campos casi iguales).
- Documentación interna desactualizada en varios puntos: `pom.xml:17` ("Entrega 1B"), `README.md` (repositorio/entrega viejos), `OpenApiConfig` (solo mascotas), `db/schema.sql` (desfasado) — el código avanza más rápido que los documentos que lo describen.
- El comentario en `ConsultaService.java:42` ("aquí listamos y filtramos abajo si se requiere endpoint dedicado") documenta una intención sin implementar: los comentarios que describen código faltante son un smell de deuda pendiente.
- Sin CI visible para el backend (los workflows de `.github/` existentes no se revisaron en esta entrega; no hay evidencia en repo de que `mvn verify` corra en cada PR).

### 🔧 Qué mejora recomendable
- Extraer un componente `AccesoPorDatos` (o un `BaseService` con `usuarioActual()` y `verificarPropiedad()`) usado por los 4 servicios, y con ello eliminar la duplicación que produjo la fuga de datos.
- Centralizar los mapeos entidad→DTO en un mapper por dominio (o `MapStruct` si el proyecto crece).
- Hacer una pasada de "doc-code sync": actualizar `pom.xml`, `README.md`, `OpenApiConfig` y eliminar/regenerar `db/schema.sql`.
- Activar CI (GitHub Actions) que ejecute `mvn verify` + `npm build` en cada PR, para fijar cobertura, tests y build en el tiempo.

---

## Cierre: valoración del PFC

### Fortalezas principales (priorizadas)
1. **Seguridad de autenticación madura para un PFC**: JWT con access/refresh, blacklist en Redis, cookies HttpOnly/SameSite, BCrypt(12), `@PreAuthorize` + control de acceso por datos y auditoría de eventos (`JwtService`, `JwtCookieService`, `TokenBlacklistService`, `AuthenticationAuditService`). Pocos trabajos de fin de curso llegan a este nivel.
2. **Buenas prácticas de BD**: Flyway versionado, `ddl-auto: validate`, FKs/CHECKs/índices, cuenta de aplicación con privilegios mínimos y `ALTER DEFAULT PRIVILEGES` (`db/roles.sql`).
3. **Estrategia de pruebas sólida y verificada**: 109 tests con umbral automático de cobertura ≥60 % en `verify`, tests de seguridad dedicados y tests de integración con PostgreSQL real vía Testcontainers.
4. **Diseño de la API cuidado**: RFC 7807 Problem Details, códigos de estado correctos (201/204/404/409/422/429+Retry-After), paginación uniforme y errores sin fuga de internos.
5. **Uso real y medido de Redis** (caché, blacklist, cache-aside externo con mejora ~64x documentada) y **resiliencia básica de la integración externa** (timeouts, taxonomía de errores 4xx/5xx/429).
6. **Disciplina de evidencia**: ADRs, C4, matriz de trazabilidad, mediciones reproducibles con commit asociado — extraordinariamente documentado para el ámbito académico.

### Debilidades identificadas (priorizadas)
1. **Fuga de datos en `ConsultaService.listar()`** para `ROLE_DUENO` (lista global sin filtrar, admitida en un comentario del propio código) — fallo real de autorización horizontal, sin test que lo cubra.
2. **Clave de API externa comprometida y mal cableada**: secreto real versionado en `.env.example` y propiedad `app.external-api.key` ausente de `application.yml` — la integración falla desde una copia limpia.
3. **Control de acceso duplicado e inconsistente entre módulos** (4 copias de la misma lógica; matriz de permisos DELETE distinta entre citas y consultas/vacunas).
4. **Rate limiting en memoria y solo en login**; refresh token sin rotación; credencial admin `Admin123*` versionada y sembrada — huecos de seguridad operacional.
5. **Doble fuente de verdad del esquema** (`db/schema.sql` desfasado vs Flyway) y migraciones desiguales (V3 sin trigger ni índice `activo`).
6. **Documentación de API y metadatos desactualizados** (OpenAPI solo "mascotas", 0 anotaciones `@Operation`, `pom.xml`/README desfasados) y build no reproducible del frontend (`npm install` sin lockfile).

### Mejoras recomendadas — roadmap corto
1. **Semanas 1-2 (crítico, seguridad)**: corregir el filtrado por dueño en `ConsultaService.listar()` + test de regresión; rotar y eliminar la clave de API Ninjas del repo; declarar `app.external-api` en `application.yml`; eliminar `rol` de `RegistroRequest` y `estado` de `CitaRequest` (POST).
2. **Semanas 2-4 (arquitectura)**: unificar la política de acceso en un helper compartido (elimina la duplicación de los 4 servicios); alinear V3 con las demás migraciones y añadir unicidad de citas (V5); desactivar `db/schema.sql` en favor de Flyway puro.
3. **Semanas 4-6 (operación)**: rotación de refresh tokens, rate limiting en Redis (login+registro+refresh), credencial admin por entorno; test de la integración externa con mock server; `.dockerignore`, `npm ci`, `USER` no root y digest en imágenes frontend.
4. **Semanas 6-8 (contrato y CI)**: `@Operation`/`@Tag` + actualización de `OpenApiConfig`; actualizar metadatos (pom/README); activar GitHub Actions con `mvn verify` y build de frontend.

### Valoración general del PFC
BIOPET es, con diferencia, un proyecto por encima de la media de los PFC de aplicación web: la arquitectura por capas es limpia, la autenticación y la seguridad de cabeceras/CORS/TLS están bien resueltas, la base de datos está correctamente versionada y modelada, y existe una disciplina de pruebas y evidencia (ADRs, mediciones, trazabilidad) que la mayoría de trabajos académicos no alcanza. No obstante, **no está listo para producción sin trabajo adicional**: tiene al menos un fallo real de autorización por datos (listado de consultas), un secreto comprometido en el historial con un cableado defectuoso que rompe la integración externa desde cero, y un control de acceso duplicado que ya produjo una regresión. Todas las debilidades son corregibles con cambios acotados (el roadmap anterior lo muestra); el proyecto está en un estado "release candidate bien auditado pero con deuda de seguridad puntual", lo que es una posición muy razonable para una entrega académica y una base sólida si se quiere llevar a producción.

---

