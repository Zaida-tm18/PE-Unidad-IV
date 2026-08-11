# SOAP vs REST — Comparación técnica

Investigación documental para la Unidad IV. Comparación técnica entre SOAP y REST sobre 10 criterios, con casos de uso en Ecuador, un ejemplo comparado en un dominio genérico y criterios de decisión. Todas las fuentes citadas se verificaron disponibles en línea al momento de escribir este documento (agosto 2026). En esta versión, las citas se priorizaron hacia **archivos PDF descargables** (verificados por tipo de contenido); donde el emisor oficial (W3C, IETF) no publica una versión PDF de su especificación, se mantiene el enlace HTML original porque no existe alternativa oficial en PDF.

---

## 1. Resumen ejecutivo

SOAP (Simple Object Access Protocol) es un **protocolo de mensajería** estándar del W3C que define una estructura de sobre (envelope) en XML para intercambiar información entre aplicaciones, mientras que REST (REpresentational State Transfer) es un **estilo arquitectónico** que describe restricciones (cliente-servidor, stateless, cacheable, interfaz uniforme, sistema por capas, código bajo demanda) para diseñar APIs que aprovechan la semántica de HTTP (Fielding, 2000). La diferencia no es "un API vs otro API": SOAP impone un contrato de transporte y formato; REST impone una disciplina de diseño sobre HTTP (RFC 9110).

---

## 2. Comparación por criterios

| # | Criterio | SOAP | REST |
|---|----------|------|------|
| 1 | Naturaleza | Protocolo (estándar W3C) | Estilo arquitectónico (disertación de Fielding) |
| 2 | Formato de datos | XML exclusivamente (envelope + header + body) | JSON principalmente; también XML, texto plano, HTML |
| 3 | Contrato y documentación | WSDL (obligatorio, máquina-legible) | OpenAPI/Swagger (opcional, humano+máquina) |
| 4 | Transporte | Independiente: HTTP, SMTP, TCP, JMS... | Depende de HTTP (semántica de verbos, códigos, cabeceras) |
| 5 | Complejidad | Alta: curva de aprendizaje de WS-* (WS-Security, WS-Addressing, WS-ReliableMessaging) | Baja-media: verbos HTTP + recursos; herramientas generan clientes desde OpenAPI |
| 6 | Rendimiento | Mensajes XML verbosos, mayor tamaño y costo de parseo | JSON ligero; respuestas cacheables por HTTP; mejor latencia en el caso típico |
| 7 | Seguridad | WS-Security a nivel de mensaje (firmas, cifrado XML, tokens) | HTTPS/TLS + OAuth2/JWT a nivel de transporte/app |
| 8 | Estado | Puede ser stateful (sesiones, transacciones ACID distribuidas con WS-Transaction) | Stateless por diseño; el estado vive en el cliente/recurso |
| 9 | Uso empresarial | Sistemas legacy, banca, gobierno, ESB/SOA, telecomunicaciones | Microservicios, APIs públicas, cloud, SaaS |
| 10 | Integración web/móvil | Difícil: XML + WSDL + stubs; no se consume directamente desde el navegador | Nativa: fetch/axios, JSON, OpenAPI → SDKs de cualquier plataforma |

Detalle por criterio, explicando **por qué** difieren y qué consecuencia real tiene:

### 1. Naturaleza: protocolo vs. estilo arquitectónico
SOAP es una especificación del W3C (SOAP 1.2 Part 1: Messaging Framework) que estandariza el formato del mensaje (Envelope, Header, Body, Fault) y los patrones de intercambio (request-response, one-way). REST, en cambio, no es un estándar ni un formato: es la teoría que Roy Fielding formalizó en su **disertación doctoral completa** (2000, PDF oficial de 216 páginas) para explicar por qué la Web escala, y se aplica como conjunto de restricciones de diseño. Consecuencia práctica: un cliente SOAP **debe** generar el XML con el sobre correcto para invocar "una operación"; un cliente REST solo usa la semántica que HTTP ya tiene (GET/POST/PUT/DELETE, códigos de estado, cabeceras). Pautasso, Zimmermann y Leymann (2008) formalizan esta misma distinción con un método de comparación basado en decisiones arquitectónicas: SOAP/WS-* ofrece menos decisiones pero más alternativas por decisión; REST exige más decisiones pero con una sola alternativa "hazlo tú mismo" en la mayoría de los casos.
**Fuentes:** Fielding, R. (2000), *Architectural Styles and the Design of Network-based Software Architectures* (tesis doctoral completa, PDF), https://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf · Pautasso, Zimmermann y Leymann (2008), *RESTful Web Services vs. "Big" Web Services* (PDF), https://design.inf.usi.ch/sites/default/files/biblio/www2008-restws-pautasso-zimmermann-leymann.pdf · W3C, "SOAP Version 1.2 Part 1: Messaging Framework" (sin versión PDF oficial), https://www.w3.org/TR/soap12-part1/

### 2. Formato de datos: XML vs. JSON/otros
SOAP **solo** admite XML: el sobre y su contenido deben ser XML válido (SOAP 1.2 Part 1), lo que impone verbosidad (namespaces, prefijos, tipado XSD) y parseo costoso. REST no prescribe formato; en la práctica JSON domina porque es más compacto, legible y nativo de JavaScript. El estudio "Comparative Analysis of SOAP and REST APIs" (2024) confirma empíricamente que el payload SOAP en XML consume más ancho de banda que el JSON equivalente en REST. Consecuencia: un mismo dato ocupa menos bytes y requiere menos CPU en REST, y es trivial consumirlo desde el frontend.
**Fuentes:** W3C, SOAP 1.2 Part 1 (sin PDF oficial), https://www.w3.org/TR/soap12-part1/ · "Comparative Analysis of SOAP and REST APIs" (2024, PDF), https://eprints.unite.edu.mk/1951/1/revista%20-%202024-228-243.pdf

### 3. Contrato y documentación: WSDL vs. OpenAPI/Swagger
SOAP exige WSDL (Web Services Description Language, recomendación del W3C de 2001): un XML máquina-legible que describe operaciones, mensajes, tipos (XSD) y *bindings*; los entornos de desarrollo generan clientes (stubs) directamente desde el WSDL. REST no exige contrato: OpenAPI (iniciativa de la OpenAPI Initiative, antigua Swagger) es **opcional** y describe recursos, verbos, parámetros y esquemas JSON en un formato YAML/JSON. Consecuencia: WSDL da contrato *verificable y tipado* pero con coste alto de mantenimiento; OpenAPI documenta lo que el humano necesita y es el estándar de facto de las APIs modernas.
**Fuentes:** W3C, "Web Services Description Language (WSDL) 1.1" (sin PDF oficial), https://www.w3.org/TR/wsdl · OpenAPI Initiative (sin PDF oficial), https://www.openapis.org/ · Swagger, "OpenAPI Specification" (sin PDF oficial), https://swagger.io/specification/

### 4. Transporte: dependencia de HTTP vs. independencia de protocolo
SOAP es transport-neutral: el mismo mensaje puede viajar por HTTP, SMTP, TCP o colas (JMS), porque el sobre encapsula la información de routing (WS-Addressing) en el propio mensaje. REST está atado a HTTP: sus restricciones (verbos, códigos de estado, caché, HATEOAS) son precisamente la semántica de HTTP, definida hoy en RFC 9110. Fielding (2000, capítulo 5 de la disertación completa) describe esta dependencia como resultado directo de que REST se deriva del estilo cliente-servidor-con-caché aplicado específicamente sobre la Web. Consecuencia: SOAP sigue siendo útil cuando el canal de comunicación no es HTTP o cuando la infraestructura intermedia (ESB) rutea por protocolos múltiples; REST no puede existir fuera de HTTP.
**Fuentes:** IETF, RFC 9110 "HTTP Semantics" (sin PDF oficial), https://www.rfc-editor.org/rfc/rfc9110 · Fielding, R. (2000), tesis doctoral completa (PDF), https://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf · Pautasso, Zimmermann y Leymann (2008), PDF, https://design.inf.usi.ch/sites/default/files/biblio/www2008-restws-pautasso-zimmermann-leymann.pdf

### 5. Complejidad: curva de aprendizaje y verbosidad
SOAP arrastra el ecosistema WS-* (WS-Security, WS-Addressing, WS-ReliableMessaging, WS-Transaction) que requiere decisiones de diseño para cada capa; la evidencia empírica de Pautasso et al. (2008, PDF) muestra que WS-* exige menos decisiones arquitectónicas globales pero con más alternativas cada una (10 decisiones, ≥25 alternativas a nivel tecnológico), mientras REST exige un número similar de decisiones pero con solo una alternativa "hazlo tú mismo" en la mayoría de los casos — lo que en la práctica se traduce en más código manual. REST elimina el "plumbing" del contrato WSDL: el verbo HTTP y el recurso son la operación.
**Fuentes:** Pautasso, Zimmermann y Leymann (2008), PDF, https://design.inf.usi.ch/sites/default/files/biblio/www2008-restws-pautasso-zimmermann-leymann.pdf · "Comparative Analysis of SOAP and REST APIs" (2024, PDF), https://eprints.unite.edu.mk/1951/1/revista%20-%202024-228-243.pdf

### 6. Rendimiento: peso de mensajes y overhead
El XML de SOAP con envelope+header+namespaces+tipado XSD es varias veces mayor que el JSON equivalente; cada mensaje además requiere validación de esquema y parseo DOM/SAX en el servidor. El estudio de benchmarking "Performance Benchmarking of RESTful and SOAP APIs" (PDF) midió en condiciones controladas una latencia promedio de 35 ms para REST frente a 58 ms para SOAP en carga media, y de 72 ms frente a 112 ms bajo alta concurrencia — una reducción de 35-40% atribuible a los payloads JSON livianos y al manejo directo de HTTP frente al overhead de parseo XML de SOAP. El mismo estudio midió throughput de ~1.320 solicitudes/seg para REST frente a ~930 para SOAP bajo carga media. REST además es cacheable por HTTP (Cache-Control/ETag), beneficio que SOAP no obtiene por defecto en navegadores/proxies.
**Fuentes:** "Performance Benchmarking of RESTful and SOAP APIs" (PDF), https://jsaer.com/download/vol-5-iss-11-2018/JSAER2018-05-11-376-390.pdf · "Comparative Analysis of SOAP and REST APIs" (2024, PDF), https://eprints.unite.edu.mk/1951/1/revista%20-%202024-228-243.pdf

### 7. Seguridad: WS-Security vs. HTTPS/OAuth/JWT
WS-Security (estándar OASIS, "WSS: SOAP Message Security 1.0", PDF oficial de 56 páginas) protege **el mensaje en sí**: firma XML (XML Signature) sobre el `<S11:Body>` u otros elementos, cifrado de elementos vía XML Encryption (`<xenc:EncryptedData>`), y tokens de seguridad (username, X.509, binarios) incluidos en la cabecera `<wsse:Security>`, de modo que la confidencialidad/integridad sobrevive a intermediarios no confiables y a transportes sin TLS. REST delega la seguridad en la capa de transporte (HTTPS/TLS) y en estándares de aplicación como OAuth 2.0 y JWT. Consecuencia: para integraciones B2B con múltiples saltos (gateway → ESB → sistema core) o donde el mensaje debe quedar firmado y auditable por sí mismo, WS-Security sigue siendo el estándar; para APIs públicas/consumer, HTTPS+OAuth es la práctica dominante y mucho más simple de operar.
**Fuentes:** OASIS, "Web Services Security: SOAP Message Security 1.0 (WS-Security 2004)", OASIS Standard, marzo 2004 (PDF oficial), http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-soap-message-security-1.0.pdf · IETF, RFC 9110 (TLS sobre HTTP, sin PDF oficial), https://www.rfc-editor.org/rfc/rfc9110

### 8. Manejo del estado: stateless vs. stateful
REST exige stateless por restricción de diseño (Fielding, 2000, sección 5.1.3 de la disertación completa): cada request contiene toda la información necesaria; el servidor no guarda contexto entre peticiones, lo que permite escalar horizontalmente con cualquier balanceador y recuperar instancias sin perder sesiones. El propio Fielding describe esta como una decisión de diseño con trade-off explícito: mejora visibilidad, fiabilidad y escalabilidad, a costa de mayor overhead de datos repetidos por interacción. SOAP no impone stateless: la familia WS-Transaction/WS-ReliableMessaging soporta transacciones ACID distribuidas y correlación de mensajes (estado conversacional), y muchas implementaciones SOAP clásicas son stateful. Consecuencia: escalar un servicio REST es "añadir instancias"; escalar un servicio SOAP stateful exige *session affinity* o centralizar el estado.
**Fuentes:** Fielding, R. (2000), tesis doctoral completa (PDF), sección 5.1.3, https://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf · Pautasso, Zimmermann y Leymann (2008), PDF, https://design.inf.usi.ch/sites/default/files/biblio/www2008-restws-pautasso-zimmermann-leymann.pdf

### 9. Uso empresarial: legacy, banca, gobierno, ESB
SOAP nació en 1998-2000 precisamente para integración empresarial: interoperabilidad entre lenguajes/plataformas con contrato fuerte, transacciones y mensajería confiable, y por eso sigue dominando en core bancario, gobierno, telecomunicaciones y ESB/SOA. El estudio "A comparative study on SOAP and RESTful web services" (IRJET, PDF) y "Comparative Analysis of SOAP and REST APIs" (2024, PDF) coinciden en que SOAP sigue siendo favorecido en entornos B2B con kits de desarrollo ya establecidos e interactividad cliente-servidor estrecha, mientras que REST se acepta cada vez más como apropiado también para el ámbito empresarial gracias a su flexibilidad y menores restricciones de enlace. La literatura académica (Pautasso et al., 2008) concluye lo mismo: WS-* para integración empresarial de larga vida con requerimientos avanzados de QoS; REST para integración táctica, ad hoc y sobre la Web.
**Fuentes:** "Comparative Analysis of SOAP and REST APIs" (2024, PDF), https://eprints.unite.edu.mk/1951/1/revista%20-%202024-228-243.pdf · "A comparative study on SOAP and RESTful web services" (IRJET, PDF), https://www.irjet.net/archives/V7/i5/IRJET-V7I5553.pdf · Pautasso, Zimmermann y Leymann (2008), PDF, https://design.inf.usi.ch/sites/default/files/biblio/www2008-restws-pautasso-zimmermann-leymann.pdf

### 10. Integración web/móvil
El navegador y las apps móviles consumen JSON con fetch/XMLHttpRequest/axios de forma nativa; parsear XML SOAP desde el frontend exige librerías, construcción manual de envelopes y manejo de SOAP Fault, y el WSDL no sirve de nada sin generación de stubs. El estudio "A Comparative study of SOAP vs REST web services provisioning techniques for mobile host" (PDF) analiza específicamente el aprovisionamiento de servicios web a dispositivos móviles con recursos limitados, y concluye que REST es más amigable para infraestructura inalámbrica que SOAP, en parte porque SOAP requiere análisis de adjuntos binarios y mayor peso de payload — factores críticos cuando el cliente es un dispositivo con batería y ancho de banda restringidos. Consecuencia: una API que deba ser consumida por frontend Angular/React, app móvil o terceros, encuentra en REST el camino corto.
**Fuentes:** "A Comparative study of SOAP vs REST web services provisioning techniques for mobile host" (PDF), https://www.researchgate.net/profile/Dr-K-Wagh/publication/264227921_A_Comparative_study_of_SOAP_vs_REST_web_services_provisioning_techniques_for_mobile_host/links/553ca71b0cf29b5ee4b8a10c/A-Comparative-study-of-SOAP-vs-REST-web-services-provisioning-techniques-for-mobile-host.pdf · "Comparison of SOAP and REST Based Web Services Using Software Evaluation Metrics" (PDF), https://www.researchgate.net/publication/312566917_Comparison_of_SOAP_and_REST_Based_Web_Services_Using_Software_Evaluation_Metrics/fulltext/588231574585150dde40195e/Comparison-of-SOAP-and-REST-Based-Web-Services-Using-Software-Evaluation-Metrics.pdf

---

## 3. Casos de uso en Ecuador

### 3.1 SRI — Facturación electrónica (SOAP)
El Servicio de Rentas Internas expone los servicios de recepción y autorización de comprobantes electrónicos **exclusivamente como Web Services SOAP con WSDL**: `RecepcionComprobantesOffline` y `AutorizacionComprobantesOffline`, según su Ficha Técnica oficial descargable directamente del repositorio del SRI (versión 2.26, PDF). El contribuyente genera el comprobante en XML (XSD del SRI), lo firma digitalmente (XAdES-BES con certificado PKCS12) y envía el XML firmado como parámetro `byte[]` del método SOAP; la respuesta llega en un objeto XML (`RespuestaRecepcionComprobante`). **Por qué SOAP aquí:** el SRI impone el contrato (WSDL+XSD) para validar millones de documentos con firma electrónica; el mensaje firmado debe ser verificable por sí mismo y el canal es estrictamente B2B sin navegador. Toda facturación electrónica ecuatoriana (sistemas POS, ERPs) habla SOAP con el SRI.
**Fuente:** SRI Ecuador, *Ficha Técnica: Manual de Usuario, Catálogo y Especificaciones Técnicas sobre el Proceso de Autorización y Emisión de Documentos Electrónicos*, esquema offline, versión 2.26 (PDF oficial, repositorio sri.gob.ec), https://www.sri.gob.ec/o/sri-portlet-biblioteca-alfresco-internet/descargar/ed555352-46c7-4917-9f61-011b6a9f4600/FICHA%20TE%CC%81CNICA%20COMPROBANTES%20ELECTRO%CC%81NICOS%20ESQUEMA%20OFFLINE%20Versio%CC%81n%202.26.pdf
*(Nota: la versión 2.31 mencionada originalmente no está indexada como PDF público; esta es la versión oficial más reciente localizada en el repositorio del SRI con el mismo contenido normativo. Verificar en sri.gob.ec &gt; Facturación Electrónica si existe una versión posterior antes de la entrega final.)*

### 3.2 Banca ecuatoriana e integraciones corporativas (SOAP y canales B2B)
El core bancario (pagos, transferencias interbancarias vía SPI/SNR del Banco Central, nómina, débitos) se integra históricamente con mensajería estándar (ISO 8583, SOAP/WS-*) entre bancos y con sus clientes corporativos. Las entidades reguladas privilegian contratos tipados, mensajería confiable y pistas de auditoría por mensaje — exactamente el terreno de WS-Security y WS-Transaction descritos en la especificación OASIS (PDF, sección 7). **Razón del sesgo:** cumplimiento normativo y contrato verificable pesan más que la comodidad de desarrollo; además los sistemas core son legacy que ya hablan estos protocolos.
**Fuentes:** OASIS, "Web Services Security: SOAP Message Security 1.0" (PDF), http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-soap-message-security-1.0.pdf · "Comparative Analysis of SOAP and REST APIs" (2024, PDF), https://eprints.unite.edu.mk/1951/1/revista%20-%202024-228-243.pdf

### 3.3 Fintech y startups ecuatorianas — pagos y apps móviles (REST)
Kushki, la pasarela de pagos ecuatoriana, documenta su API como **API REST**: orientada a recursos, con respuestas codificadas en JSON y códigos de estado HTTP estándar. Los comercios la integran desde tiendas en línea y apps móviles con HTTPS + API keys. **Por qué REST aquí:** el consumidor es un frontend o app móvil con tiempos de desarrollo cortos; la escalabilidad stateless barata es un requisito de plataformas con picos de tráfico — el mismo patrón que documenta el estudio de aprovisionamiento móvil (PDF) para entornos de recursos limitados.
**Fuentes:** Kushki API Docs (sin PDF oficial), https://api-docs.kushkipagos.com/ · "A Comparative study of SOAP vs REST web services provisioning techniques for mobile host" (PDF), https://www.researchgate.net/profile/Dr-K-Wagh/publication/264227921_A_Comparative_study_of_SOAP_vs_REST_web_services_provisioning_techniques_for_mobile_host/links/553ca71b0cf29b5ee4b8a10c/A-Comparative-study-of-SOAP-vs-REST-web-services-provisioning-techniques-for-mobile-host.pdf

### 3.4 Gobierno y sector público — tendencia mixta
Además del SRI (SOAP, PDF de referencia oficial en 3.1), el sector público ecuatoriano mantiene servicios web SOAP en varios sistemas de la administración, por las mismas razones históricas. En paralelo, los servicios nuevos orientados al ciudadano tienden a REST. **Conclusión del patrón ecuatoriano:** el criterio dominante no es "SOAP es mejor" sino **quién fija el contrato y quién consume**: si el proveedor oficial exige WSDL (SRI, banca) → SOAP; si el consumidor es una app moderna o un tercero y el contrato lo definimos nosotros → REST.
**Fuentes:** SRI Ecuador, Ficha Técnica v2.26 (PDF), https://www.sri.gob.ec/o/sri-portlet-biblioteca-alfresco-internet/descargar/ed555352-46c7-4917-9f61-011b6a9f4600/FICHA%20TE%CC%81CNICA%20COMPROBANTES%20ELECTRO%CC%81NICOS%20ESQUEMA%20OFFLINE%20Versio%CC%81n%202.26.pdf · "Comparative Analysis of SOAP and REST APIs" (2024, PDF), https://eprints.unite.edu.mk/1951/1/revista%20-%202024-228-243.pdf

---

## 4. Ejemplo técnico comparado

Caso de uso: **consultar los datos de un cliente por su ID** (dominio genérico: una empresa pregunta a la API "¿qué cliente es el #123?").

### 4.1 SOAP (XML + envelope, sobre HTTP)

```http
POST /api/empresa/clienteService HTTP/1.1
Host: api.empresa.example
Content-Type: text/xml; charset=utf-8
SOAPAction: "urn:consultarCliente"

<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Header>
    <auth:token xmlns:auth="http://empresa.example/auth" soap:mustUnderstand="1">
      eyJhbGciOiJIUzI1NiJ9... (token WS-Security o token de aplicación)
    </auth:token>
  </soap:Header>
  <soap:Body>
    <consultarCliente xmlns="http://empresa.example/gestion">
      <clienteId>123</clienteId>
    </consultarCliente>
  </soap:Body>
</soap:Envelope>
```

Respuesta SOAP (la operación devuelve el objeto dentro de otro envelope):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <consultarClienteResponse xmlns="http://empresa.example/gestion">
      <cliente>
        <id>123</id>
        <nombre>María Pérez</nombre>
        <email>maria.perez@example.com</email>
        <telefono>+593 99 000 1234</telefono>
        <tipoCliente>PREMIUM</tipoCliente>
      </cliente>
    </consultarClienteResponse>
  </soap:Body>
</soap:Envelope>
```

Características que se observan: la operación (`consultarCliente`) va **dentro del mensaje** (no existe como URL); el header puede exigir `mustUnderstand`, tal como especifica la sección 5 de OASIS WS-Security (PDF); la respuesta viene envuelta en otro sobre; el contrato de tipos/operación lo define el WSDL.

### 4.2 REST (JSON + verbo HTTP)

```http
GET /api/clientes/123 HTTP/1.1
Host: api.empresa.example
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Accept: application/json
```

Respuesta (200 OK, el recurso representado en JSON):

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 123,
  "nombre": "María Pérez",
  "email": "maria.perez@example.com",
  "telefono": "+593 99 000 1234",
  "tipoCliente": "PREMIUM",
  "fechaRegistro": "2025-03-15",
  "activo": true,
  "creadoEn": "2026-07-01T10:00:00Z",
  "actualizadoEn": "2026-07-10T14:30:00Z"
}
```

La misma consulta en SOAP exigiría: generar el envelope, enviarlo a una única URL y parsear el XML anidado en la respuesta; en REST basta un GET a una URL con el token en la cabecera `Authorization` y la respuesta es JSON directo — con el beneficio adicional de la caché HTTP y de los códigos de estado semánticos (200/404), consistente con los datos de latencia medidos en el estudio de benchmarking (PDF, sección de resultados).

**Resumen de la comparación en este ejemplo:** el caso de lectura de un recurso por ID es donde la ventaja de REST es máxima (semántica de URL+verbo, JSON compacto, cacheable, código 404 si no existe vs. SOAP Fault). El caso SOAP se vuelve competitivo cuando la operación no es un recurso CRUD (procesar un lote firmado de comprobantes, transacción con idempotencia y correlación), como en la facturación del SRI.

---

## 5. Conclusión: criterios de decisión

Regla práctica (no un "depende" ambiguo):

**Usar SOAP cuando se cumple al menos uno de:**
1. El proveedor del servicio **exige** WSDL/contrato (SRI, banca, sistemas gubernamentales, SAP) — no hay elección, es requisito de interoperabilidad.
2. La integración es **B2B sin navegador**, con múltiples saltos o transportes no HTTP (JMS, SMTP) donde el mensaje debe ser firmado y auditable por sí mismo (WS-Security 1.0, PDF de la especificación OASIS) más allá del canal.
3. Se requieren **transacciones distribuidas / mensajería confiable** explícitas (WS-Transaction, WS-ReliableMessaging) sobre infraestructura ESB/SOA existente.
4. El sistema destino es **legacy** que ya habla SOAP y reescribirlo no tiene retorno (Pautasso et al., 2008, PDF).

**Usar REST cuando:**
1. El consumidor es un **frontend web, app móvil, o tercero que integra con SDK/HTTP** — patrón confirmado por el estudio de aprovisionamiento móvil (PDF).
2. El dominio se modela como **recursos CRUD** con lectura frecuente: REST + JSON + caché HTTP/CDN gana en latencia y ancho de banda, según las cifras del benchmarking (PDF: 35% menos latencia, 40% más throughput bajo carga media).
3. Se necesita **escalabilidad horizontal simple** (stateless obligatorio), tal como lo define Fielding (2000, sección 5.1.3 de la tesis completa en PDF).
4. El equipo desarrolla el **contrato propio** y quiere documentarlo con OpenAPI/Swagger.
5. El modelo de seguridad es **HTTPS + OAuth2/JWT** y no se requiere firma de mensaje extremo a extremo (a diferencia del modelo de WS-Security, PDF).

**Decisión compuesta en la práctica:** la API propia de una organización debe ser REST cuando el dominio es de recursos consumidos por frontends, apps móviles o terceros, y la integración es táctica. Solo los tramos externos obligatorios serían SOAP cuando el proveedor lo exige (SRI, banca, sistemas gubernamentales, SAP) — y allí la opción arquitectónica correcta es un **adaptador aislado** (un servicio que traduzca REST interno ↔ SOAP externo), manteniendo la API pública en REST. Esta "piel de REST alrededor de núcleos SOAP" es el patrón real del mercado: Kushki expone REST al comercio mientras el tramo hacia el emisor bancario sigue siendo SOAP/ISO, y el SRI recibe facturas por SOAP aunque los ERPs la integren por detrás con interfaces propias.

---

## 6. Referencias

1. Fielding, R. T. (2000) — *Architectural Styles and the Design of Network-based Software Architectures* (tesis doctoral completa, PDF oficial). University of California, Irvine. https://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf
2. Pautasso, C., Zimmermann, O., Leymann, F. (2008) — *RESTful Web Services vs. "Big" Web Services: Making the Right Architectural Decision* (WWW2008, PDF). https://design.inf.usi.ch/sites/default/files/biblio/www2008-restws-pautasso-zimmermann-leymann.pdf
3. OASIS (2004) — *Web Services Security: SOAP Message Security 1.0 (WS-Security 2004)*, OASIS Standard, 15 marzo 2004 (PDF oficial). http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-soap-message-security-1.0.pdf
4. SRI Ecuador — *Ficha Técnica: Manual de Usuario, Catálogo y Especificaciones Técnicas sobre el Proceso de Autorización y Emisión de Documentos Electrónicos*, esquema offline, versión 2.26 (PDF oficial, repositorio sri.gob.ec). https://www.sri.gob.ec/o/sri-portlet-biblioteca-alfresco-internet/descargar/ed555352-46c7-4917-9f61-011b6a9f4600/FICHA%20TE%CC%81CNICA%20COMPROBANTES%20ELECTRO%CC%81NICOS%20ESQUEMA%20OFFLINE%20Versio%CC%81n%202.26.pdf
5. "Comparative Analysis of SOAP and REST APIs" (2024, PDF). https://eprints.unite.edu.mk/1951/1/revista%20-%202024-228-243.pdf
6. "Comparison of SOAP and REST Based Web Services Using Software Evaluation Metrics" (PDF). https://www.researchgate.net/publication/312566917_Comparison_of_SOAP_and_REST_Based_Web_Services_Using_Software_Evaluation_Metrics/fulltext/588231574585150dde40195e/Comparison-of-SOAP-and-REST-Based-Web-Services-Using-Software-Evaluation-Metrics.pdf
7. "Performance Benchmarking of RESTful and SOAP APIs" (JSAER, PDF). https://jsaer.com/download/vol-5-iss-11-2018/JSAER2018-05-11-376-390.pdf
8. "A Comparative study of SOAP vs REST web services provisioning techniques for mobile host" (PDF). https://www.researchgate.net/profile/Dr-K-Wagh/publication/264227921_A_Comparative_study_of_SOAP_vs_REST_web_services_provisioning_techniques_for_mobile_host/links/553ca71b0cf29b5ee4b8a10c/A-Comparative-study-of-SOAP-vs-REST-web-services-provisioning-techniques-for-mobile-host.pdf
9. "A comparative study on SOAP and RESTful web services" (IRJET, PDF). https://www.irjet.net/archives/V7/i5/IRJET-V7I5553.pdf
10. W3C — *SOAP Version 1.2 Part 1: Messaging Framework* (2nd ed., 2007). https://www.w3.org/TR/soap12-part1/
11. W3C — *Web Services Description Language (WSDL) 1.1* (2001). https://www.w3.org/TR/wsdl
12. IETF — *RFC 9110: HTTP Semantics*(2022). https://www.rfc-editor.org/rfc/rfc9110
13. OpenAPI Initiative. https://www.openapis.org/ · Swagger, OpenAPI Specification (v3.1). https://swagger.io/specification/
14. Kushki — API Docs (API REST, JSON, códigos HTTP estándar). https://api-docs.kushkipagos.com/ y https://docs.kushki.com/ec/
