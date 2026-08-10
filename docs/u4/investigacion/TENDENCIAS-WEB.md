# Tendencias web modernas: Jamstack, PWA e IA generativa

**Autor:** Panamá Murillo Moisés Antonio
**Actividad:** Grupo GA, Unidad IV — Investigación sobre tendencias de desarrollo web
**Contexto de aplicación:** PFC BIOPET (`Grinjoww/PE-U4-PFC-BIOPET`)

---

## Jamstack

Jamstack (JavaScript, APIs, Markup) es un enfoque de arquitectura web que desacopla por completo el frontend del backend: el frontend se genera como archivos estáticos (o pre-renderizados) servidos desde una CDN, mientras que toda la lógica dinámica —autenticación, base de datos, procesamiento de negocio— se delega a APIs externas o *serverless functions*. La idea central no es que el sitio no tenga nada dinámico, sino que ningún servidor propio necesita renderizar HTML en cada petición: el contenido dinámico llega vía llamadas asíncronas a APIs.

La ventaja principal es el rendimiento: servir archivos estáticos desde una red de distribución de contenido reduce drásticamente el tiempo de primera carga en comparación con un servidor que arma cada página en el momento. También reduce superficie de ataque, porque no hay un motor de plantillas del lado del servidor procesando cada petición, y simplifica el escalamiento porque los archivos estáticos son triviales de replicar. La limitación más relevante es que Jamstack asume que el backend ya expone una API bien definida y estable; si la lógica de negocio cambia constantemente o requiere renderizado con datos muy personalizados por usuario, el enfoque se vuelve incómodo y termina necesitando renderizado en el servidor de todos modos (SSR híbrido). Funciona mejor en sitios de contenido, documentación, marketing, blogs o aplicaciones donde el frontend consume una API relativamente estable.

## PWA (Progressive Web App)

Una PWA es una aplicación web que adopta capacidades tradicionalmente exclusivas de apps nativas: se puede instalar en la pantalla de inicio sin pasar por una tienda de aplicaciones, funciona (al menos parcialmente) sin conexión, y puede enviar notificaciones push. El componente técnico central es el *service worker*, un script que el navegador ejecuta en segundo plano, independiente del ciclo de vida de la pestaña, y que intercepta las peticiones de red para decidir si las responde desde una caché local o las deja pasar a la red real.

Esto habilita el funcionamiento offline: el service worker puede servir la última versión conocida de la interfaz (o incluso datos) aunque el dispositivo pierda conexión, y sincronizar cambios pendientes cuando la vuelve a recuperar. Para aplicaciones que se usan en campo, con conectividad inestable, esta capacidad es más que un lujo cosmético. La utilidad real en aplicaciones modernas está en que combina la facilidad de distribución de la web (no requiere instalación desde una tienda ni revisión de terceros) con una experiencia percibida cercana a la de una app nativa.

## IA generativa en desarrollo web

La IA generativa se ha vuelto parte cotidiana del ciclo de desarrollo: generación y autocompletado de código, redacción y mantenimiento de documentación técnica, generación de casos de prueba, y asistencia en decisiones de diseño de interfaz. También empieza a integrarse dentro del producto final, no solo como herramienta del equipo: chatbots de soporte, búsqueda semántica, generación de resúmenes o recomendaciones dentro de la propia aplicación.

El riesgo principal no es que la IA se equivoque ocasionalmente (eso es esperable), sino la confianza excesiva sin verificación: código generado que compila pero tiene fallas de seguridad sutiles, documentación que suena coherente pero describe un comportamiento que el sistema no tiene realmente, o pruebas generadas que validan el camino feliz sin cubrir casos límite. El límite más importante para un equipo de desarrollo es tratar la salida de estas herramientas como un borrador que necesita revisión humana, especialmente en código que toca autenticación, autorización o manejo de datos sensibles.

## Reflexión crítica

Ninguna de estas tres tendencias debería adoptarse porque "está de moda". Jamstack no mejora nada si la aplicación es intrínsecamente dinámica y personalizada por usuario en cada vista; una PWA no aporta valor si los usuarios reales de un sistema siempre tienen conexión estable; y la IA generativa mal usada puede introducir deuda técnica más rápido de lo que la elimina. El criterio correcto es siempre el mismo: ¿qué problema concreto del sistema resuelve esta tecnología, y ese problema existe de verdad en este contexto?

## Aplicación a BIOPET

De las tres, la que más valor concreto aportaría a BIOPET en su estado actual es **PWA**, no Jamstack. BIOPET es, por diseño, una aplicación intensamente dinámica y personalizada por rol y por propietario (un `DUENO` solo ve sus propias mascotas, el control de acceso depende del usuario autenticado en cada petición), justo el escenario donde Jamstack aporta menos y una arquitectura cliente-servidor tradicional como la que ya tiene tiene más sentido.

PWA, en cambio, resuelve un problema real del dominio: una clínica veterinaria no siempre tiene conectividad perfecta, y el personal (veterinario, auxiliar) podría necesitar consultar el historial de una mascota o registrar una consulta en una zona con señal débil. Convertir el frontend Angular en PWA —agregar un service worker con estrategia de caché para el listado de mascotas ya consultado, y una cola de sincronización para operaciones de escritura pendientes cuando se recupere la conexión— encaja de forma natural con la arquitectura que ya existe, sin requerir rediseñar el backend.

Sobre IA generativa, su aplicación más razonable en BIOPET no sería dentro del producto (un chatbot no resuelve ningún problema real del dominio veterinario descrito), sino en el propio proceso de desarrollo: ya se usa implícitamente para mantener la documentación al día (algo que, como señalo en mi revisión crítica, el proyecto tiene pendiente: el README no refleja los endpoints de Citas y Consultas que ya existen en el código). Una tarea concreta y de bajo riesgo sería usar IA generativa para detectar automáticamente ese tipo de desfase entre código y documentación antes de cada entrega, en lugar de para generar funcionalidad nueva sin supervisión.
