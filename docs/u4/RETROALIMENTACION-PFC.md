# Retroalimentación cruzada — PFC BIOPET

**Unidad IV — Actividad GA**

Este documento integra las dos revisiones críticas independientes del PFC
BIOPET (Cajas y Panamá) más una síntesis final. Se arma una vez que ambos
integrantes entreguen `REVISION-CAJAS.md` y `REVISION-PANAMA.md`.

> ✅ **Estado actual: completo.** Ambos insumos fueron recibidos e
> integrados íntegros a continuación. La síntesis cruzada (sección 3) ya
> está redactada a partir del contenido real de las dos revisiones.

---

## 1. Revisión crítica #1 — Cajas

**Fuente:** `docs/u4/revisiones/REVISION-CAJAS.md`

<details>
<summary><strong>Ver revisión completa de Cajas (13 aspectos técnicos + cierre)</strong></summary>

# REVISION-CAJAS — Revisión crítica del PFC BIOPET

- **Proyecto:** BIOPET — Sistema web de gestión veterinaria (Backend: Java 21, Spring Boot 3.2.12; Frontend: Angular 17; PostgreSQL 16 + Redis 7; Docker Compose).
- **Alcance de la revisión:** solo backend y soporte de despliegue/pruebas/documentación; el frontend se menciona únicamente cuando aporta evidencia (flujo de autenticación).
- **Método:** inspección directa de código, configuración, migraciones, tests y evidencias de `docs/` (no es revisión de ejecución). Cada afirmación referencia un archivo o fragmento concreto.
- **Commit de referencia de la revisión:** `eabf704` (main).

## 1. Arquitectura MVC

**✅ Qué está bien:** separación de capas real y consistente (`controller/`, `service/`, `repository/`, `dto/`, `entity/`); controladores sin lógica de negocio en los 7 revisados; transacciones delimitadas en la capa de servicio (`@Transactional`, `open-in-view: false`).

**⚠️ Problema — el hallazgo más grave de toda la revisión:** la frontera de permisos está partida en dos capas y aplicada de forma distinta por módulo. `ConsultaService.listar()` para `ROLE_DUENO` ejecuta el **mismo listado global sin filtrar** que usan staff/ADMIN (el propio comentario del código lo admite), mientras `MascotaService` y `VacunaService` sí filtran por dueño. Consecuencia: **un dueño con token válido ve las consultas clínicas de todas las mascotas de la clínica** — fuga de datos clínicos entre propietarios, escalación horizontal de privilegios. Falta además el método de repositorio (`findAllByMascota_Duenio_IdAndActivoTrue`) que `ConsultaRepository` no tiene.

**🔧 Mejora:** agregar el método de repositorio faltante y usarlo en la rama `ROLE_DUENO` de `ConsultaService.listar()` (prioridad alta, fallo de autorización); unificar la estrategia de control de acceso en un único lugar (helper compartido) para eliminar la duplicación entre `MascotaService`, `CitaService`, `ConsultaService` y `VacunaService`.

## 2. Organización del backend

**✅ Qué está bien:** estructura por capas clara y uniforme; DTOs de entrada/salida separados de entidades (records inmutables); integración externa aislada en su propio paquete; migraciones Flyway reservadas por integrante.

**⚠️ Problema:** metadatos desactualizados (`pom.xml` describe el proyecto como "Entrega 1B"; el README referencia otro repositorio); `dto/` es un paquete plano de 20+ clases sin agrupar por feature; sin checkstyle/spotless en el build.

**🔧 Mejora:** actualizar `pom.xml`/README al estado real; agregar linter al build; documentar convención de agrupación de DTOs si el dominio crece.

## 3. API REST

**✅ Qué está bien:** diseño orientado a recursos con verbos HTTP correctos; códigos de estado correctos (201/204/404/409/422/429+Retry-After); errores estandarizados con RFC 7807 Problem Details; paginación uniforme; validación de rutas con regex en varios controladores.

**⚠️ Problema:** inconsistencia en patrones de ruta entre módulos (`{id:\d+}` vs `{id}` sin regex, con comportamiento distinto ante input inválido); matriz de permisos DELETE sin criterio documentado (un veterinario puede borrar una consulta pero no una cita, sin ADR que lo justifique); el endpoint de API externa rompe la convención REST (parámetro de consulta libre sin validar); sin versionado de API; `RegistroRequest.rol` es un campo del contrato que el servidor ignora.

**🔧 Mejora:** unificar el patrón de rutas con test de contrato; documentar la matriz de permisos por recurso/verbo en un ADR; mover la consulta externa a un recurso propio y validado.

## 4. Swagger / OpenAPI

**✅ Qué está bien:** springdoc-openapi 2.5.0 con rutas propias (`/api/openapi`, `/api/docs`) públicas y sin autenticación; esquema de seguridad `bearerAuth` declarado; prueba automatizada que verifica que Swagger UI se sirve correctamente.

**⚠️ Problema:** cero anotaciones `@Operation`/`@Schema`/`@Tag` en todo el código (documentación 100% autogenerada, sin ejemplos ni descripciones); la descripción de la API en `OpenApiConfig` está desactualizada (solo menciona "autenticación JWT y CRUD de mascotas" cuando ya existen citas, consultas, vacunas, etc.); no documenta el mecanismo de autenticación por cookies que realmente usa el frontend Angular, solo bearer.

**🔧 Mejora:** anotar al menos un endpoint representativo por controlador; actualizar `Info.description`; documentar ambos mecanismos de autenticación (cookie + bearer).

## 5. CRUD

**✅ Qué está bien:** CRUD completo y uniforme para los 4 dominios con eliminación lógica verificada por test; validaciones con Bean Validation; manejo de errores centralizado sin fuga de detalles internos; reglas de integridad de negocio verificadas con tests.

**⚠️ Problema — contratos con campos que el servidor ignora:** `RegistroRequest.rol` es obligatorio en el contrato pero `AuthService.registrar()` lo ignora y fuerza `ROLE_DUENO` (hoy inocuo, pero riesgoso si alguien "arregla" el servicio sin revisar el contrato); `CitaRequest.estado` exigido en el POST pero también ignorado. Validaciones incompletas: sin `@Future` en `fechaHora` de citas (se puede agendar en el pasado), sin control de solapamiento de horarios. `UsuarioService.eliminar` no impide que un admin se auto-desactive.

**🔧 Mejora:** eliminar o marcar como solo-lectura los campos muertos del contrato; añadir `@Future` y restricción de unicidad para citas; proteger la auto-desactivación del admin.

## 6. Autenticación y roles

**✅ Qué está bien:** JWT robusto (HMAC-SHA256, validación de longitud de clave, claims `iss`/`aud`); doble token access/refresh con blacklist en Redis; cookies `HttpOnly`/`Secure`/`SameSite=Strict`; sesiones stateless; `BCryptPasswordEncoder(12)`; control de acceso por datos (propiedad del recurso) además de por rol; auditoría de eventos de autenticación y rate limiting de login con `Retry-After`.

**⚠️ Problema:** el rate limiter de login vive en memoria (`ConcurrentHashMap`) — no funciona en despliegue multi-instancia, no usa Redis pese a estar disponible, y solo protege `/login` (no `/registro` ni `/refresh`); el refresh token **no rota** (si se filtra, es válido los 7 días completos sin detección de reuso); la cuenta admin sembrada tiene contraseña fija en código y en el seed SQL; política de contraseña solo exige longitud mínima, sin complejidad.

**🔧 Mejora:** mover el rate limiting a Redis y extenderlo a registro/refresh; rotar el refresh token en cada uso; externalizar la contraseña admin vía variable de entorno; reforzar la política de contraseñas.

## 7. Seguridad

**✅ Qué está bien:** cabeceras de seguridad completas (HSTS, CSP, X-Frame-Options, etc.) verificadas por test; CORS restringido sin `*`; protección contra inyección SQL vía JPA/derived queries con prueba dedicada; perfil TLS 1.3 disponible; cuenta de BD de privilegios mínimos para la aplicación; CSRF deshabilitado con justificación válida para JWT stateless.

**⚠️ Problema — el hallazgo de seguridad más concreto:** **una clave real de API externa (API Ninjas) está versionada en `.env.example` y comprometida en el historial de commits**, pese a que el propio archivo afirma que ningún valor ahí es un secreto real. Además está mal cableada: la propiedad `app.external-api.key` no existe en `application.yml`, por lo que la variable de entorno nunca llega a la aplicación — la integración externa falla desde una copia limpia del repositorio. Doble problema: clave filtrada **e** inefectiva.

**🔧 Mejora:** rotar la clave, purgarla del historial, declarar correctamente la propiedad en `application.yml`, y validar al arranque (fail-fast) que los secretos críticos no tengan valores por defecto en producción.

## 8. Base de datos

**✅ Qué está bien:** PostgreSQL 16 con Flyway y 4 migraciones ordenadas con integridad referencial real (FKs, CHECKs, índices, triggers de auditoría); `ddl-auto: validate`; modelado de entidades coherente con las migraciones; privilegios mínimos por rol de BD; pruebas de integración contra PostgreSQL real con Testcontainers.

**⚠️ Problema:** doble fuente de verdad del esquema — `db/schema.sql` duplica manualmente la V1 y ya está desfasado (no contiene citas/consultas/vacunas); sin restricción de unicidad para evitar doble reserva de citas; `V3__consultas.sql` no crea el trigger de auditoría que sí tienen las demás migraciones; las FKs no restringen el rol del usuario asignado a nivel de BD (solo en la capa de servicio).

**🔧 Mejora:** eliminar la duplicación del esquema (usar solo Flyway); añadir restricción de unicidad de citas; alinear V3 con las demás migraciones.

## 9. Redis

**✅ Qué está bien:** tres usos reales y diferenciados (caché de respuestas con TTL, blacklist de tokens JWT, cache-aside de la API externa con mejora ~64x medida); mediciones honestas y reproducibles del hit ratio; escrituras de caché no fatales (degradan a sin-caché si fallan).

**⚠️ Problema:** subutilización evidente — el rate limiter de login sería el caso ideal para Redis y sigue en memoria; las lecturas de caché/blacklist no están protegidas con try/catch (si Redis cae, ciertos endpoints responderían 500 en vez de degradar); solo se cachean mascotas y consultas, no citas ni vacunas; Redis en Docker Compose sin `requirepass` (aceptable en dev, debe documentarse como límite para producción).

**🔧 Mejora:** mover rate limiting a Redis; envolver lecturas de caché en try/catch con degradación definida; extender el caché a citas y vacunas.

## 10. Integración con API externa

**✅ Qué está bien:** cliente desacoplado y resiliente en lo básico (timeouts explícitos, taxonomía de errores 429/4xx/5xx/timeout, cache-aside con TTL, mapeo DTO defensivo); endpoint protegido por autenticación y rol.

**⚠️ Problema — la debilidad más grave de este módulo:** el mismo bug de cableado de la clave (tema 7): el flujo "feliz" documentado en la evidencia no es reproducible desde una copia limpia del repositorio. Además, sin reintentos ni circuit breaker ante 5xx/timeout, y sin límite de uso por usuario de la cuota compartida.

**🔧 Mejora:** corregir el cableado y rotar la clave; agregar reintento con backoff exponencial; evaluar circuit breaker si hay más de un consumidor.

## 11. Pruebas

**✅ Qué está bien:** 109 tests, 0 fallos, con umbral de cobertura automatizado que falla el build si baja de 60%; cobertura real medida de 95.87% de líneas; tipos de tests variados (integración HTTP completa, seguridad, PostgreSQL real con Testcontainers); casos negativos bien cubiertos (403/404/422).

**⚠️ Problema — conecta directamente con el hallazgo del tema 1:** la fuga de datos de `ConsultaService.listar()` **no tiene test** — no existe ningún test de `GET /api/consultas` con rol `DUENO` que verifique el filtrado por propiedad. La regresión existe precisamente donde no hay cobertura de autorización de listados. Tampoco hay tests de la integración externa, ni tests de frontend (`0` archivos `.spec.ts`).

**🔧 Mejora:** agregar el test de autorización de listados que hoy habría capturado la fuga; tests de la integración externa con servidor mock; tests de componente en el frontend.

## 12. Docker

**✅ Qué está bien:** multi-stage build del backend con imagen final pequeña; digests fijados por SHA-256 para las imágenes base de Postgres y Redis; healthchecks en los 4 servicios con arranque ordenado; volumen persistente para Postgres; evidencias de verificación reproducible.

**⚠️ Problema:** sin `.dockerignore` en ningún nivel del proyecto; el contenedor backend corre como root (sin `USER` en el Dockerfile); build del frontend no reproducible (`npm install` sin lockfile, sin digests fijados en sus imágenes base).

**🔧 Mejora:** crear `.dockerignore`; usuario no root en el backend; `npm ci` con lockfile y digests fijados en el frontend.

## 13. Mantenibilidad general

**✅ Qué está bien:** código legible y compacto con javadoc de calidad en los servicios más complejos; documentación de decisiones de arquitectura vía ADRs; diagramas C4 y matriz de trazabilidad con script de validación; evidencias de rendimiento/seguridad/usabilidad reproducibles y con commit asociado.

**⚠️ Problema — la raíz de varios de los hallazgos anteriores:** la lógica de autorización (`tieneAccesoGlobal`/`verificarAcceso`/`verificarPropiedad`) se repite casi idéntica en los cuatro servicios de dominio — es precisamente la razón por la que una de las cuatro copias "olvidó" el filtro (tema 1). Documentación interna desactualizada en varios puntos (pom.xml, README, OpenApiConfig, schema.sql).

**🔧 Mejora:** extraer un componente compartido de control de acceso usado por los cuatro servicios, eliminando la duplicación que produjo la fuga de datos; hacer una pasada de sincronización documentación-código.

## Cierre: valoración del PFC (Cajas)

**Fortalezas principales:** seguridad de autenticación madura para un PFC (JWT access/refresh, blacklist Redis, cookies HttpOnly, BCrypt, control de acceso por rol y por datos); buenas prácticas de base de datos (Flyway, validación de esquema, privilegios mínimos); estrategia de pruebas sólida (109 tests, umbral de cobertura automático ≥60%); diseño de API cuidado (RFC 7807, códigos de estado correctos); uso real y medido de Redis; disciplina de evidencia (ADRs, C4, trazabilidad).

**Debilidades identificadas:** fuga de datos en `ConsultaService.listar()` para `ROLE_DUENO` (fallo real de autorización horizontal, sin test); clave de API externa comprometida y mal cableada; control de acceso duplicado e inconsistente entre módulos; rate limiting en memoria y sin rotación de refresh token; doble fuente de verdad del esquema de base de datos; documentación de API y metadatos desactualizados.

**Mejoras recomendadas (roadmap corto):** semanas 1-2, corregir el filtrado de `ConsultaService` y rotar la clave filtrada (crítico, seguridad); semanas 2-4, unificar la política de acceso y alinear migraciones (arquitectura); semanas 4-6, rotación de refresh tokens y rate limiting en Redis (operación); semanas 6-8, documentación OpenAPI y CI (contrato).

**Valoración general:** BIOPET está, con diferencia, por encima de la media de los PFC de aplicación web — arquitectura limpia, seguridad y BD bien resueltas, disciplina de pruebas y evidencia poco común en trabajos académicos. No obstante, no está listo para producción sin trabajo adicional: tiene al menos un fallo real de autorización por datos, un secreto comprometido con cableado defectuoso, y control de acceso duplicado que ya produjo una regresión real. Es un "release candidate bien auditado pero con deuda de seguridad puntual", posición razonable para una entrega académica.

</details>

## 2. Revisión crítica #2 — Panamá

**Fuente:** `docs/u4/revisiones/REVISION-PANAMA.md`

<details>
<summary><strong>Ver revisión completa de Panamá (5 aspectos + cierre)</strong></summary>

# Revisión crítica independiente — PFC BIOPET

**Autor:** Panamá Murillo Moisés Antonio · **Repositorio evaluado:** `Grinjoww/PE-U4-PFC-BIOPET` (`v0.9.0-rc`) · **Alcance:** usabilidad, organización, mantenibilidad, escalabilidad, seguridad, experiencia de usuario, pruebas, rendimiento, documentación y despliegue.

## 1. Usabilidad y experiencia de usuario (UX)

**Fortaleza:** evidencia real de esfuerzo en accesibilidad en el frontend (`role="toolbar"`, `aria-live`, `aria-invalid`/`aria-describedby`, diálogo de confirmación con `role="alertdialog"`), poco habitual en un proyecto de curso. Además, una prueba SUS real con 10 participantes externos y consentimiento informado, con intervalo de confianza al 95% calculado correctamente con distribución t de Student para n=10 (media 74.75/100, clasificación "Bueno").

**Problema:** el hallazgo de que el participante con menor puntaje (22.5) fue quien no tenía experiencia previa con apps web quedó documentado como limitación honesta, pero sin traducirse todavía en un cambio de diseño (por ejemplo, onboarding guiado).

**Problema más importante — brecha backend/frontend:** el backend ya expone `CitaController`, `ConsultaController` y `VacunaController` con sus migraciones Flyway, pero el frontend solo tiene componentes para mascotas, vacunas y login. No existen `citas.component.ts` ni `consultas.component.ts`. Para un sistema de "gestión veterinaria", la ausencia de interfaz para citas y consultas médicas es una limitación de usabilidad real: el usuario final no puede usar buena parte de lo que el sistema ya sabe hacer.

## 2. Organización y mantenibilidad

**Fortaleza:** separación por capas clara y convencional en el backend, con un paquete `integration` dedicado a la API externa; decisiones de arquitectura documentadas como ADR, incluyendo la migración de stack a mitad de proyecto (ADR-002).

**Problema:** no encontró documentada la regla de integridad referencial para casos como dar de baja una mascota con citas pendientes. El frontend tiene **cero archivos `*.spec.ts`**: mientras el backend alcanza 109 pruebas y 95.87% de cobertura, el frontend no tiene ninguna red de seguridad automatizada.

## 3. Seguridad

**Fortaleza:** diseño de autenticación más allá de lo típico de un curso — cookies `HttpOnly`/`Secure`/`SameSite=Strict`, JWT con revocación real en Redis, rate limiting con `429`/`Retry-After`, control de acceso de dos capas (rol + propiedad), y que el registro público fuerce siempre `ROLE_DUENO` sin importar lo que mande el cliente en el body.

**Problema (el propio equipo lo admite):** el rate limiting vive en memoria por instancia del backend — si el sistema alguna vez escala a más de una réplica, ese control deja de funcionar como está diseñado.

**Problema:** la contraseña de la cuenta admin sembrada está fija en el código fuente y replicada en el seed SQL — una credencial versionada, del tipo que un checklist OWASP marcaría en una auditoría real.

## 4. Escalabilidad y rendimiento

**Fortaleza:** mediciones de rendimiento con evidencia real (seis corridas de k6 frío/caliente, intervalos de confianza con distribución t, throughput ~91-92 req/s, p95 entre 9-27 ms); verificación aislada del comportamiento de Redis en vez de confiar en métricas globales mezcladas.

**Problema:** todo se midió contra una sola instancia del backend, sin réplicas — las cifras actuales no dicen nada sobre el comportamiento bajo carga real con más usuarios concurrentes de los que atiende una sola instancia.

## 5. Documentación y despliegue

**Fortaleza:** pineo de imágenes Docker de terceros por digest SHA-256, práctica que muchos proyectos profesionales no aplican todavía; Makefile documentado con precisión sobre qué hace y qué **no** hace cada objetivo.

**Problema más concreto de esta revisión:** el README documenta como "endpoints actuales" únicamente `AuthController`, `MascotaController` y `UsuarioController`, pero el código ya incluye `CitaController`, `ConsultaController` y `ExternalApiController` en producción — la tabla de endpoints del README está desactualizada respecto al código real.

**Problema menor:** las mediciones de Lighthouse aún no se han ejecutado ni versionado, aunque la infraestructura (`make lighthouse`, script) ya está lista.

## Cierre (Panamá)

**Fortalezas principales:** seguridad de autenticación bien pensada más allá del mínimo del curso; cultura de evidencia reproducible poco común en proyectos académicos (SUS con estadística correcta, k6 con IC, JaCoCo, ADRs); accesibilidad real en el frontend.

**Debilidades identificadas:** desfase entre documentación (README) y código real en la lista de endpoints; brecha de paridad funcional entre backend y frontend (citas y consultas sin interfaz); ausencia total de pruebas automatizadas en el frontend; rate limiting no preparado para escalar horizontalmente; credencial de administrador fija en código fuente.

**Mejoras recomendadas:** actualizar la tabla de endpoints del README; priorizar componentes de citas y consultas en Angular para la entrega final; incorporar pruebas unitarias mínimas en Angular; mover el rate limiting de login a Redis; externalizar la contraseña admin sembrada.

**Valoración general:** BIOPET está considerablemente por encima del estándar típico de un PFC de esta asignatura — no se queda en "que funcione", sino que mide, documenta decisiones y reconoce sus propias limitaciones con honestidad poco frecuente. Lo que le falta no es esfuerzo, sino cerrar la distancia entre lo que el backend ya sabe hacer y lo que el usuario final puede realmente usar, y llevar al frontend el mismo nivel de disciplina de pruebas que ya tiene el backend.

</details>

## 3. Síntesis cruzada

Ambas revisiones son independientes (Cajas se enfocó en backend/código
fuente con referencias línea a línea; Panamá se enfocó en usabilidad,
paridad backend-frontend y evidencia experimental como SUS y k6), pero
convergen en un diagnóstico común del PFC.

### 3.1 Dónde coinciden

| Punto de coincidencia | Cajas | Panamá |
|---|---|---|
| Desfase documentación ↔ código real | README/pom.xml desactualizados respecto a citas/consultas/vacunas ya implementadas | La tabla de "endpoints actuales" del README omite `CitaController`, `ConsultaController`, `ExternalApiController` |
| Rate limiting de login no apto para escalar | En memoria (`ConcurrentHashMap`), solo protege `/login`, no usa Redis pese a estar disponible | Mismo hallazgo, señalado como limitación que el propio equipo ya admite |
| Credencial de administrador insegura | `admin@biopet.ec` con contraseña fija en `DataInitializer.java` y `db/seed.sql` | Misma observación, enmarcada como hallazgo tipo checklist OWASP |
| Falta de pruebas automatizadas fuera del backend | Sin tests para la integración externa ni para el caso de la fuga de datos en consultas | Cero archivos `*.spec.ts` en todo el frontend Angular |

Ambos revisores llegan de forma independiente a los mismos tres o cuatro
puntos débiles usando evidencia distinta (código fuente vs. experimentación
con usuarios), lo que refuerza la validez de esos hallazgos: no son una
opinión aislada, sino un patrón detectado desde dos ángulos distintos.

### 3.2 Dónde se complementan (no se repiten)

- **Cajas** encontró un fallo de seguridad concreto y grave que Panamá no
  menciona: la **fuga de datos en `ConsultaService.listar()`** (un
  `ROLE_DUENO` ve las consultas clínicas de todas las mascotas, no solo las
  propias), verificado línea a línea y sin test de regresión que lo
  detecte. Es un hallazgo de nivel de código que solo aparece con
  inspección directa del backend.
- **Panamá** encontró una brecha que Cajas no cubre porque su alcance
  excluía el frontend: **el backend ya soporta citas y consultas, pero el
  usuario final no tiene ninguna interfaz para usarlas** (no existen
  `citas.component.ts` ni `consultas.component.ts`). Es un hallazgo de
  paridad funcional, no de código backend.
- **Panamá** aporta evidencia experimental que Cajas no genera: la prueba
  SUS con usuarios reales (74.75/100, "Bueno") y las mediciones de
  rendimiento con k6, que dan una dimensión de experiencia de usuario real
  que la sola inspección de código no puede ofrecer.
- **Cajas** aporta un segundo hallazgo de seguridad que Panamá no llega a
  ver: la **clave de API externa comprometida en `.env.example` y además
  mal cableada** (`app.external-api.key` ausente de `application.yml`), que
  requiere revisar el código de integración con el proveedor externo.

### 3.3 Las 3 mejoras más críticas a priorizar (combinando ambas revisiones)

1. **Corregir la fuga de datos de `ConsultaService.listar()`** (Cajas) —
   es el único hallazgo de las dos revisiones que constituye una falla de
   seguridad activa y explotable con una cuenta `DUENO` normal, no una
   deuda técnica ni una limitación declarada.
2. **Cerrar la brecha backend/frontend en citas y consultas** (Panamá) —
   sin esto, buena parte del trabajo de backend evaluado por Cajas
   (incluida la propia fuga de datos que se corrija) queda inaccesible
   para el usuario real del sistema.
3. **Rate limiting y credenciales fuera del código fuente** (ambos) —
   mover el rate limiting a Redis y externalizar la contraseña admin son
   cambios acotados, señalados por los dos revisores de forma
   independiente, y necesarios antes de cualquier demo o despliegue fuera
   del entorno académico.

### 3.4 Valoración general consolidada del PFC

Las dos revisiones —una desde el código, otra desde la evidencia
experimental y la experiencia de usuario— llegan a la misma conclusión
general por caminos distintos: **BIOPET está claramente por encima del
estándar típico de un PFC de esta asignatura**, con una disciplina de
evidencia (ADRs, mediciones k6, SUS con estadística correcta, cobertura de
pruebas, trazabilidad) poco común en trabajos académicos, y con una
seguridad de autenticación (JWT, blacklist, cookies, control por rol y por
propiedad) más madura que el mínimo esperado. Al mismo tiempo, **no está
listo para producción ni para su siguiente entrega sin trabajo adicional**:
tiene un fallo de autorización real y explotable, una clave comprometida en
el repositorio, y una brecha de usabilidad donde el frontend no expone
funcionalidad que el backend ya implementa. Ninguna de estas debilidades es
estructural: todas son corregibles con cambios acotados, lo que ubica al
proyecto en un estado de "candidato bien auditado, con deuda puntual de
seguridad y de paridad funcional" — una posición sólida para una entrega
académica y una base razonable si el equipo decide llevarlo más allá del
curso.

---

*Documento completo. Fuentes: `REVISION-CAJAS.md` y `REVISION-PANAMA.md`,
entregados por los integrantes correspondientes e integrados íntegros en
las secciones 1 y 2.*
