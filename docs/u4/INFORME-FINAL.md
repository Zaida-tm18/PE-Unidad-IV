# Informe Final — Unidad IV — PFC BIOPET

**Grupo GA**

> Este archivo es el **esqueleto navegable** del informe final. Las
> secciones que me corresponden (arquitectura, MVC, REST y evidencias) están
> desarrolladas o enlazadas a su documento fuente. Las secciones que
> dependen de Cajas y Panamá quedan marcadas con `🔲 PENDIENTE` y con la
> referencia exacta al archivo que deben producir, para pegar su contenido
> sin reestructurar nada.

---

## Portada

- Título: Informe Final — Unidad IV — Grupo GA — PFC BIOPET
- Asignatura / curso / periodo: `[ 🔲 completar con datos institucionales ]`
- Integrantes: `[Tu nombre]` (responsable de arquitectura/MVC/REST/evidencias),
  Cajas (revisión crítica #1 + SOAP vs REST), Panamá (revisión crítica #2 +
  Jamstack/PWA/IA)
- Fecha: `[ 🔲 completar ]`

## Resumen

`[ 🔲 completar al final, cuando todas las secciones estén cerradas — 200-300
palabras resumiendo qué es BIOPET, qué se revisó, qué se investigó y las
conclusiones principales. ]`

## Abstract

`[ 🔲 versión en inglés del resumen anterior, misma extensión. ]`

## 1. Introducción

`[ 🔲 completar — contexto de la Unidad IV: el PFC ya está terminado por sus
integrantes originales; el rol del grupo GA es revisar, investigar y
aportar al informe final, sin nuevas funcionalidades. ]`

## 2. Desarrollo

### 2.1 Descripción del PFC seleccionado

✅ Ver [`docs/u4/PFC-SELECCIONADO.md`](PFC-SELECCIONADO.md) — secciones 1 a 5
(qué es BIOPET, problema que resuelve, alcance funcional, stack y
arquitectura general). Integrar tal cual como subsección aquí.

### 2.2 Flujo MVC

✅ Ver [`docs/u4/PFC-SELECCIONADO.md`](PFC-SELECCIONADO.md) — sección 6
(flujo MVC real con clases y métodos verificados: Angular → `MascotaController`
→ `MascotaService` → `MascotaRepository` → PostgreSQL → JSON).

### 2.3 API REST

✅ Ver [`docs/u4/EVIDENCIA-API-REST.md`](EVIDENCIA-API-REST.md) — acceso a
Swagger (ruta real `/api/docs`) y evidencia de al menos 3 endpoints
ejecutados con método, ruta, objetivo, código HTTP, resultado y capturas.

### 2.4 Retroalimentación cruzada (revisiones de Cajas y Panamá)

✅ Ver [`docs/u4/RETROALIMENTACION-PFC.md`](RETROALIMENTACION-PFC.md) —
ambas revisiones completas (Cajas: 13 aspectos técnicos con referencias a
código; Panamá: 5 aspectos con evidencia experimental SUS/k6) más una
síntesis cruzada que identifica coincidencias, complementos y las 3
mejoras más críticas combinando ambas. Hallazgo principal a destacar en la
defensa: la fuga de datos en `ConsultaService.listar()` (Cajas) y la
brecha backend/frontend en citas y consultas (Panamá).

### 2.5 SOAP vs REST

✅ Ver [`docs/u4/investigacion/SOAP-VS-REST.md`](investigacion/SOAP-VS-REST.md)
(Cajas) — comparación en 10 criterios (naturaleza, formato de datos,
contrato/documentación, transporte, complejidad, rendimiento, seguridad,
manejo del estado, uso empresarial, integración web/móvil), con 14 fuentes
académicas y normativas citadas dentro del texto, casos de uso reales en
Ecuador (SRI en SOAP para facturación electrónica, Kushki en REST para
pagos), ejemplo de petición SOAP vs. REST equivalente, y criterios de
decisión explícitos. Es, por sí solo, el bloque del informe con mayor
densidad de citas y referencias verificables (ver sección 5).

### 2.6 Jamstack, PWA e IA generativa

✅ Ver [`docs/u4/investigacion/TENDENCIAS-WEB.md`](investigacion/TENDENCIAS-WEB.md)
(Panamá) — cubre Jamstack, PWA e IA generativa, con reflexión crítica
(ninguna tendencia debe adoptarse "por moda") y aplicación razonada a
BIOPET: concluye que **PWA** es la tendencia de mayor valor concreto para
el proyecto (uso en campo con conectividad inestable), que Jamstack no
aplica bien a un sistema intensamente dinámico y personalizado por rol como
BIOPET, y que la IA generativa aportaría más en el propio proceso de
desarrollo (detectar desfases documentación-código, el mismo problema que
señala la revisión de Cajas) que dentro del producto.

> ⚠️ **Nota de calidad para la defensa:** a diferencia de `SOAP-VS-REST.md`,
> este documento no incluye una lista de fuentes citadas en el texto. Es
> un desarrollo argumentativo sólido, pero conviene que Panamá agregue al
> menos 2-3 referencias (documentación oficial de Jamstack.org, MDN sobre
> Service Workers, o algún estudio sobre adopción de IA en desarrollo de
> software) antes de la entrega final, para que esta sección también
> cumpla el requisito de citas dentro del texto que sí cumplen las demás
> secciones del informe.

## 3. Conclusiones

> Borrador listo para revisión final del grupo; ajustar tono/extensión
> según lo que pida la rúbrica.

- **Arquitectura y flujo MVC:** BIOPET implementa una separación de capas
  Spring Boot consistente y verificable (`controller` → `service` →
  `repository` → PostgreSQL), documentada con nombres de clase y método
  reales en este informe. Esta misma disciplina de capas, sin embargo, es
  la causa raíz de uno de los hallazgos más importantes de la revisión de
  Cajas: al duplicarse la lógica de control de acceso en cuatro servicios
  casi idénticos, una de las copias quedó incompleta.
- **API REST:** el diseño de la API sigue convenciones REST correctas
  (verbos, códigos de estado, RFC 7807), documentado con Swagger en su
  ruta real (`/api/docs`, no la genérica que aparecía en el README). La
  revisión de Cajas confirma este diseño como una fortaleza, aunque señala
  inconsistencias menores entre módulos (patrones de ruta, matriz de
  permisos sin documentar).
- **Retroalimentación cruzada:** las dos revisiones independientes,
  llegando por caminos distintos (inspección de código vs. evidencia
  experimental), coinciden en los mismos puntos débiles centrales —lo que
  refuerza su validez— y se complementan en hallazgos que ninguna revisión
  sola hubiera cubierto (la fuga de datos de Cajas; la brecha
  backend-frontend de Panamá).
- **SOAP vs REST:** la elección de estilo arquitectónico no es una cuestión
  de preferencia técnica sino de quién define el contrato: SOAP donde el
  proveedor lo exige (SRI, banca), REST donde el equipo controla el
  contrato y el consumidor es un frontend o app móvil — exactamente el
  escenario de BIOPET, que por eso está correctamente construido en REST.
- **Jamstack, PWA e IA generativa:** de las tres tendencias, solo PWA
  resuelve un problema real del dominio de BIOPET (conectividad inestable
  en campo); Jamstack no encaja con un sistema tan dinámico y personalizado
  por rol; la IA generativa aportaría más en el proceso de desarrollo
  (mantener la documentación sincronizada con el código) que dentro del
  producto.

## 4. Trabajos futuros

1. **Seguridad (corto plazo, según Cajas):** corregir la fuga de datos en
   `ConsultaService.listar()`, rotar y recablear correctamente la clave de
   la API externa, mover el rate limiting de login a Redis y externalizar
   la contraseña del administrador sembrado.
2. **Paridad funcional (corto plazo, según Panamá):** implementar los
   componentes Angular de citas y consultas para que el frontend exponga
   lo que el backend ya soporta.
3. **Pruebas (ambas revisiones):** agregar el test de autorización de
   listados por rol `DUENO` en consultas (habría detectado la fuga);
   incorporar pruebas unitarias mínimas en el frontend Angular.
4. **PWA (según la investigación de tendencias):** agregar un service
   worker con estrategia de caché para el listado de mascotas y una cola de
   sincronización para escrituras pendientes en conectividad inestable.
5. **Funcionalidad pendiente ya declarada por el propio equipo (SRS,
   sección 2.3):** historial clínico completo, telemetría IoT,
   recomendaciones asistidas, facturación y reportes avanzados.

## 5. Referencias (formato APA)

El mínimo de 5 fuentes con citas dentro del texto se cumple ampliamente al
consolidar las tres partes del informe: 5 fuentes propias (arquitectura,
MVC, REST) + 14 fuentes de la investigación SOAP vs REST de Cajas, todas
citadas dentro del texto de sus respectivas secciones.

**Fuentes propias (arquitectura, MVC, REST):**

1. Spring. (2024). *Spring Boot Reference Documentation* (versión 3.2).
   VMware. https://docs.spring.io/spring-boot/docs/3.2.x/reference/html/
2. Spring. (2024). *Spring Data JPA Reference Documentation*. VMware.
   https://docs.spring.io/spring-data/jpa/docs/current/reference/html/
3. springdoc-openapi. (2024). *springdoc-openapi documentation* (versión 2.5).
   https://springdoc.org/
4. PostgreSQL Global Development Group. (2024). *PostgreSQL 16 Documentation*.
   https://www.postgresql.org/docs/16/
5. Fielding, R. T. (2000). *Architectural styles and the design of
   network-based software architectures* [Tesis doctoral, University of
   California, Irvine]. https://ics.uci.edu/~fielding/pubs/dissertation/top.htm

**Fuentes de la investigación SOAP vs REST (Cajas)** — lista completa en
[`docs/u4/investigacion/SOAP-VS-REST.md`](investigacion/SOAP-VS-REST.md),
sección 6; incluye entre otras: Fielding (2000, tesis completa), Pautasso,
Zimmermann y Leymann (2008, WWW2008), OASIS (2004, WS-Security 1.0), SRI
Ecuador (Ficha Técnica de comprobantes electrónicos v2.26), W3C (SOAP 1.2 y
WSDL 1.1) e IETF (RFC 9110).

🔲 **Pendiente:** Panamá debe agregar al menos 2-3 referencias formales a
`TENDENCIAS-WEB.md` (ver nota en la sección 2.6) para que esa sección
también quede respaldada con citas dentro del texto, no solo argumentación.

---

## Anexo — Evidencias para la defensa

✅ Ver [`docs/u4/EVIDENCIA-API-REST.md`](EVIDENCIA-API-REST.md), sección 3:
sistema funcionando, Swagger UI en `/api/docs`, y los 3 endpoints
ejecutados en vivo.
