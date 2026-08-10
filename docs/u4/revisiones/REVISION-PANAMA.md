# Revisión crítica independiente — PFC BIOPET

**Autor:** Panamá Murillo Moisés Antonio
**Rol en la actividad:** Grupo GA, Unidad IV — revisión crítica #2
**Repositorio evaluado:** `Grinjoww/PE-U4-PFC-BIOPET` (`v0.9.0-rc`, Tercera Entrega)
**Alcance de esta revisión:** usabilidad, organización, mantenibilidad, escalabilidad, seguridad, experiencia de usuario, pruebas, rendimiento, documentación y despliegue.

---

## 1. Usabilidad y experiencia de usuario (UX)

**Fortaleza.** El frontend no se quedó en un CRUD mínimo funcional: hay evidencia real de esfuerzo en accesibilidad. En `mascotas.component.ts` aparecen atributos `role="toolbar"`, `aria-live="assertive"`/`"polite"` para mensajes de error y éxito, `aria-invalid`/`aria-describedby` en cada campo de formulario, y un diálogo de confirmación con `role="alertdialog"` y `aria-modal="true"`. Esto no es lo habitual en un proyecto de curso y demuestra que el equipo pensó en usuarios que dependen de lectores de pantalla, no solo en que "se vea bien".

**Fortaleza reforzada con datos.** A diferencia de la mayoría de PFC que dicen "el sistema es usable" sin respaldo, BIOPET corrió una prueba SUS real con 10 participantes externos, consentimiento informado y una tarea común de onboarding (login, alta, edición, baja lógica, logout). El resultado (media 74.75/100, clasificación "Bueno" según la escala de Bangor et al.) está respaldado con intervalo de confianza al 95% calculado con distribución t de Student, no una aproximación normal forzada para una muestra de n=10. Es una decisión estadísticamente correcta.

**Problema.** El propio reporte SUS señala que el participante con menor puntaje (22.5) fue quien no tenía experiencia previa con aplicaciones web, y el equipo lo documenta como una limitación honesta más que como algo que se ocultó. Sin embargo, no vi que esa observación se haya traducido todavía en un cambio de diseño (por ejemplo, un flujo de onboarding guiado). Queda como hallazgo documentado pero sin acción de mejora visible en el código.

**Problema más importante.** Hay una brecha entre lo que el backend puede hacer y lo que el usuario final puede usar. El backend ya expone `CitaController`, `ConsultaController` y `VacunaController` (con entidades `Cita`, `Consulta`, `Vacuna` y migraciones Flyway `V2__citas.sql`, `V3__consultas.sql`, `V4__vacunas.sql`), pero en `frontend/src/app/features/` solo existen componentes para mascotas, vacunas y login. No hay `citas.component.ts` ni `consultas.component.ts`. Para un sistema que se llama "gestión veterinaria", la ausencia de una interfaz para citas y consultas médicas es una limitación de usabilidad real: el usuario final no puede usar buena parte de lo que el sistema ya sabe hacer.

---

## 2. Organización y mantenibilidad

**Fortaleza.** El backend sigue una separación por capas clara y convencional en Spring Boot: `controller`, `service`, `repository`, `entity`, `dto`, `exception`, `security`, `config`, e incluso un paquete `integration` dedicado exclusivamente al consumo de la API externa (`ExternalApiClient`, `ExternalApiService`, `RestTemplateConfig`). Esa separación facilita que cualquiera del equipo (o un revisor externo como yo) entienda rápido dónde vive cada responsabilidad.

**Fortaleza.** Las decisiones de arquitectura están documentadas como ADR (ADR-002 a ADR-007), incluyendo una nada trivial: la migración de ASP.NET Core 8 a Java 21/Spring Boot 3.2 a mitad de proyecto (ADR-002). Que un equipo documente por qué cambió de stack, en vez de simplemente hacerlo y seguir, dice mucho de la madurez del proceso.

**Problema.** El manejo de errores tiene una asimetría interesante: los códigos HTTP y el formato `ProblemDetail` (RFC 7807) están bien centralizados vía `GlobalExceptionHandler`, pero yo esperaría, dada la cantidad de entidades relacionadas (Mascota → Cita → Consulta → Vacuna), ver también documentado cómo se manejan las reglas de integridad referencial cuando se intenta, por ejemplo, dar de baja una mascota con citas pendientes. No encontré esa regla de negocio documentada en el README ni en los ADR revisados.

**Problema.** El frontend tiene cero archivos `*.spec.ts`. Para un proyecto que en el backend alcanza 109 pruebas y 95.87% de cobertura de línea (verificado con JaCoCo), la ausencia total de pruebas unitarias en Angular es una inconsistencia de mantenibilidad: el backend está fuertemente blindado contra regresiones, el frontend no tiene ninguna red de seguridad automatizada, solo lo que capturen las pruebas manuales o e2e (si existen, no las encontré tampoco).

---

## 3. Seguridad

**Fortaleza.** El diseño de autenticación va más allá de lo típico de un proyecto de curso: cookies `HttpOnly`, `Secure` y `SameSite=Strict`, JWT con revocación real vía lista negra en Redis (no solo expiración por tiempo), rate limiting de login con respuesta `429` y cabecera `Retry-After`, y control de acceso de dos capas (rol + propiedad del recurso, verificado en `MascotaService.verificarPropiedad`). Que el registro público fuerce siempre `ROLE_DUENO` sin importar lo que mande el cliente en el body es exactamente el tipo de detalle que evita una escalación de privilegios trivial.

**Problema, y el propio equipo lo admite.** El rate limiting vive en un `ConcurrentHashMap` en memoria, por instancia del backend. El README lo reconoce como limitación, pero vale la pena remarcarlo desde una revisión externa: si el sistema alguna vez corre con más de una réplica del backend (necesario para escalar), ese control de seguridad deja de funcionar tal como está, porque cada instancia contaría los intentos fallidos por separado.

**Problema.** La cuenta de administrador sembrada (`admin@biopet.ec`) tiene su contraseña fija en `DataInitializer.java` y replicada en `db/seed.sql`. El README es honesto al señalar que esto debe eliminarse o externalizarse antes de cualquier despliegue fuera del entorno académico, pero hoy en día sigue siendo una credencial de código fuente versionada en el repositorio, lo cual es justo el tipo de hallazgo que un checklist de OWASP marcaría en una auditoría real.

---

## 4. Escalabilidad y rendimiento

**Fortaleza.** Las mediciones de rendimiento no son una captura de pantalla suelta: hay seis corridas de k6 (frío/caliente, tres repeticiones), con intervalos de confianza calculados con distribución t, throughput sostenido de ~91-92 req/s y latencias p95 entre 9 y 27 ms según el escenario. También se verificó de forma aislada el comportamiento del caché de Redis (`DBSIZE`, política `noeviction`, TTL decreciente), en lugar de confiar ciegamente en métricas globales de `keyspace_hits` que mezclarían el caché de mascotas con la verificación de blacklist JWT.

**Problema.** Todo esto se midió contra una sola instancia del backend, sin réplicas. Es una limitación explícita y correcta de reportar, pero significa que las cifras de throughput actuales no dicen nada sobre cómo se comportaría el sistema bajo carga real con más usuarios concurrentes de los que puede atender una sola instancia de Tomcat.

---

## 5. Documentación y despliegue

**Fortaleza.** El pineo de imágenes Docker de terceros por digest SHA256 (no solo por tag) es una práctica que muchos proyectos profesionales todavía no aplican, y aquí está explicada con el comando exacto para actualizarla. El `Makefile` documenta con precisión qué hace cada objetivo y, más importante, qué **no** hace (`make test` no aplica el umbral de cobertura; `make down` no borra volúmenes, `make reset-db` sí).

**Problema, el más concreto de esta revisión.** El README documenta como "endpoints actuales" únicamente `AuthController`, `MascotaController` y `UsuarioController`. Pero el código del repositorio ya incluye `CitaController`, `ConsultaController` y `ExternalApiController` en producción, con sus propias entidades y migraciones Flyway. Es decir, la tabla de endpoints del README está desactualizada respecto al código real: no refleja funcionalidad que ya existe y que un consumidor de la API (o un evaluador) necesitaría conocer.

**Problema menor.** El README afirma que las mediciones de Lighthouse "todavía no se han ejecutado y no tienen resultados versionados", lo cual verifiqué y es correcto (no existe carpeta `docs/mediciones/lighthouse/`). Pero sí existe ya un objetivo `make lighthouse` funcional y un script `run-lighthouse.sh`, así que la infraestructura para cerrar ese hueco ya está lista; solo falta ejecutarla y versionar el resultado antes de la entrega final.

---

## Cierre

**Fortalezas principales:** seguridad de autenticación bien pensada más allá del mínimo del curso (revocación real, cookies seguras, control por rol y propiedad); cultura de evidencia reproducible poco común en proyectos académicos (SUS con estadística correcta, k6 con IC, JaCoCo, ADR documentados); accesibilidad real en el frontend (ARIA, no solo HTML semántico superficial).

**Debilidades identificadas:** desfase entre documentación (README) y código real en la lista de endpoints; brecha de paridad funcional entre backend y frontend (citas y consultas no tienen interfaz de usuario); ausencia total de pruebas automatizadas en el frontend; rate limiting no preparado para escalar horizontalmente; credencial de administrador fija en código fuente.

**Mejoras recomendadas:**
1. Actualizar la tabla de endpoints del README para incluir Citas, Consultas y la API externa.
2. Priorizar para la entrega final al menos los componentes de citas y consultas en Angular, aunque sea con funcionalidad básica, para cerrar la brecha backend/frontend.
3. Incorporar pruebas unitarias mínimas en Angular (aunque sea sobre los servicios, que son más fáciles de testear que los componentes).
4. Mover el rate limiting de login a Redis (ya está integrado en el proyecto para la blacklist de JWT) para que sea válido ante múltiples instancias del backend.
5. Externalizar la contraseña de la cuenta admin sembrada antes de cualquier entrega o demo pública del repositorio.

**Valoración general.** BIOPET está considerablemente por encima del estándar típico de un PFC de esta asignatura: no se queda en "que funcione", sino que mide, documenta decisiones y reconoce sus propias limitaciones con honestidad poco frecuente (el README es notablemente autocrítico). Lo que le falta no es esfuerzo, sino cerrar la distancia entre lo que el backend ya sabe hacer y lo que el usuario final puede realmente usar, y llevar al frontend el mismo nivel de disciplina de pruebas que ya tiene el backend.
