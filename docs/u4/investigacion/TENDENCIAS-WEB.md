# Tendencias web modernas: Jamstack, PWA e IA generativa

**Autor:** Panamá Murillo Moisés Antonio
**Actividad:** Grupo GA, Unidad IV — Investigación sobre tendencias de desarrollo web
**Contexto de aplicación:** PFC BIOPET (`Grinjoww/PE-U4-PFC-BIOPET`)

---

## Jamstack

Jamstack (JavaScript, APIs, Markup) es un enfoque de arquitectura web que desacopla por completo el frontend del backend. La idea central no es que el sitio no tenga nada dinámico, sino que ningún servidor propio necesita renderizar HTML en cada petición: el frontend se pre-construye en archivos estáticos servidos desde una CDN, y el contenido dinámico llega vía llamadas asíncronas a APIs.

Un estudio experimental que construyó dos versiones idénticas de una misma aplicación web (un sitio de reservas hoteleras), una en arquitectura monolítica y otra en Jamstack, y midió su rendimiento con Google Lighthouse y GTmetrix, encontró diferencias sustanciales a favor de Jamstack en las métricas de carga inicial: el First Contentful Paint fue de 1.1 s frente a 1.6 s de la versión monolítica, y el Time to Interactive de apenas 0.8-0.9 s frente a 14 s [1]. Sin embargo, el mismo estudio encontró que la versión monolítica obtuvo puntajes de Accesibilidad (75 frente a 57) y SEO (91 frente a 80) más altos en Lighthouse que la versión Jamstack, y que el Fully Loaded Time terminó siendo más rápido en el sitio monolítico una vez cargados todos los recursos [1]. Es decir, la ventaja de Jamstack es real pero no incondicional: depende de qué métrica importa más para el caso de uso concreto.

La limitación más relevante es que Jamstack asume que el backend ya expone una API bien definida y estable; si la lógica de negocio cambia constantemente o requiere renderizado con datos muy personalizados por usuario, el enfoque se vuelve incómodo y termina necesitando renderizado en el servidor de todos modos (SSR híbrido). Funciona mejor en sitios de contenido, documentación, marketing, blogs o aplicaciones donde el frontend consume una API relativamente estable.

## PWA (Progressive Web App)

Una PWA es una aplicación web que adopta capacidades tradicionalmente exclusivas de apps nativas: se puede instalar en la pantalla de inicio sin pasar por una tienda de aplicaciones, funciona (al menos parcialmente) sin conexión, y puede enviar notificaciones push. El componente técnico central es el *service worker*, un script que el navegador ejecuta en segundo plano, independiente del ciclo de vida de la pestaña, y que intercepta las peticiones de red para decidir si las responde desde una caché local o las deja pasar a la red real, lo cual es la base técnica del funcionamiento offline.

Un aspecto que suele quedar fuera de la conversación puramente técnica sobre PWA es la accesibilidad. Una revisión sistemática que evaluó sitios PWA frente a sitios web tradicionales equivalentes contra las pautas WCAG encontró que las PWA no garantizan por sí solas mejor accesibilidad: la tecnología PWA resuelve problemas de rendimiento y de experiencia offline, pero el cumplimiento de las pautas de accesibilidad depende de decisiones de implementación independientes de si el sitio es o no una PWA [2]. En otras palabras, convertir un sitio en PWA no es, por sí mismo, una mejora de accesibilidad; ambas cosas deben diseñarse a propósito.

Esto habilita el funcionamiento offline: el service worker puede servir la última versión conocida de la interfaz (o incluso datos) aunque el dispositivo pierda conexión, y sincronizar cambios pendientes cuando la vuelve a recuperar. Para aplicaciones que se usan en campo, con conectividad inestable, esta capacidad es más que un lujo cosmético. La utilidad real en aplicaciones modernas está en que combina la facilidad de distribución de la web (no requiere instalación desde una tienda ni revisión de terceros) con una experiencia percibida cercana a la de una app nativa.

## IA generativa en desarrollo web

La IA generativa se ha vuelto parte cotidiana del ciclo de desarrollo: generación y autocompletado de código, redacción y mantenimiento de documentación técnica, generación de casos de prueba, y asistencia en decisiones de diseño de interfaz. Un experimento controlado de Microsoft Research con GitHub Copilot, en el que se pidió a programadores implementar un servidor HTTP en JavaScript lo más rápido posible, encontró que el grupo con acceso al asistente de IA completó la tarea un 55.8% más rápido que el grupo de control [3]. También empieza a integrarse dentro del producto final, no solo como herramienta del equipo: chatbots de soporte, búsqueda semántica, generación de resúmenes o recomendaciones dentro de la propia aplicación.

El riesgo principal no es que la IA se equivoque ocasionalmente (eso es esperable), sino la confianza excesiva sin verificación. Un estudio académico que evaluó 1,689 programas generados por GitHub Copilot a partir de escenarios ligados a debilidades de seguridad de alto riesgo (lista "Top 25" CWE de MITRE) encontró que aproximadamente el 40% de esos programas contenían vulnerabilidades explotables [4]; es decir, código que compila y funciona pero tiene fallas de seguridad sutiles, justamente el tipo de salida que un desarrollador con poca experiencia podría aceptar sin cuestionarla. A eso se suman documentación que suena coherente pero describe un comportamiento que el sistema no tiene realmente, o pruebas generadas que validan el camino feliz sin cubrir casos límite. El límite más importante para un equipo de desarrollo es tratar la salida de estas herramientas como un borrador que necesita revisión humana, especialmente en código que toca autenticación, autorización o manejo de datos sensibles.

## Reflexión crítica

Ninguna de estas tres tendencias debería adoptarse porque "está de moda". Jamstack no mejora nada si la aplicación es intrínsecamente dinámica y personalizada por usuario en cada vista; una PWA no aporta valor si los usuarios reales de un sistema siempre tienen conexión estable; y la IA generativa mal usada puede introducir deuda técnica más rápido de lo que la elimina. El criterio correcto es siempre el mismo: ¿qué problema concreto del sistema resuelve esta tecnología, y ese problema existe de verdad en este contexto?

## Aplicación a BIOPET

De las tres, la que más valor concreto aportaría a BIOPET en su estado actual es **PWA**, no Jamstack. BIOPET es, por diseño, una aplicación intensamente dinámica y personalizada por rol y por propietario (un `DUENO` solo ve sus propias mascotas, el control de acceso depende del usuario autenticado en cada petición), justo el escenario donde Jamstack aporta menos y una arquitectura cliente-servidor tradicional como la que ya tiene tiene más sentido.

PWA, en cambio, resuelve un problema real del dominio: una clínica veterinaria no siempre tiene conectividad perfecta, y el personal (veterinario, auxiliar) podría necesitar consultar el historial de una mascota o registrar una consulta en una zona con señal débil. Convertir el frontend Angular en PWA —agregar un service worker con estrategia de caché para el listado de mascotas ya consultado, y una cola de sincronización para operaciones de escritura pendientes cuando se recupere la conexión— encaja de forma natural con la arquitectura que ya existe, sin requerir rediseñar el backend.

Sobre IA generativa, su aplicación más razonable en BIOPET no sería dentro del producto (un chatbot no resuelve ningún problema real del dominio veterinario descrito), sino en el propio proceso de desarrollo: ya se usa implícitamente para mantener la documentación al día (algo que, como señalo en mi revisión crítica, el proyecto tiene pendiente: el README no refleja los endpoints de Citas y Consultas que ya existen en el código). Una tarea concreta y de bajo riesgo sería usar IA generativa para detectar automáticamente ese tipo de desfase entre código y documentación antes de cada entrega, en lugar de para generar funcionalidad nueva sin supervisión — precisamente porque, como muestra la evidencia citada arriba [4], el código generado por estas herramientas requiere revisión humana antes de aceptarse sin cuestionamientos, sobre todo en un sistema que ya maneja autenticación JWT y control de acceso por rol y propiedad.

## Referencias

[1] K. R. Shah y P. D. Joshi, "Comparative Performance Analysis of JAMstack and Monolithic Web Architectures," *International Journal of Computer Applications*, vol. 187, no. 71, pp. 51–61, ene. 2026, doi: 10.5120/ijca2026926162. [Acceso abierto].

[2] K. I. Roumeliotis y N. D. Tselikas, "Evaluating Progressive Web App Accessibility for People with Disabilities," *Network*, vol. 2, no. 2, pp. 350–369, jun. 2022, doi: 10.3390/network2020022. [Acceso abierto, CC BY 4.0].

[3] S. Peng, E. Kalliamvakou, P. Cihon, y M. Demirer, "The Impact of AI on Developer Productivity: Evidence from GitHub Copilot," arXiv:2302.06590, 2023, doi: 10.48550/arXiv.2302.06590. [Acceso abierto].

[4] H. Pearce, B. Ahmad, B. Tan, B. Dolan-Gavitt, y R. Karri, "Asleep at the Keyboard? Assessing the Security of GitHub Copilot's Code Contributions," arXiv:2108.09293, 2021 (presentado en IEEE Symposium on Security and Privacy, 2022), doi: 10.48550/arXiv.2108.09293. [Acceso abierto].

> Las fichas bibliográficas de evidencia de cada fuente (metadatos, DOI y resumen del contenido consultado) están adjuntas en `docs/u4/evidencias/panama/`. El PDF de la fuente correspondiente se sube por separado en esa misma carpeta.