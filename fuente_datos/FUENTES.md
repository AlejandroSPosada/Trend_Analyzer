# Fuentes de Datos — Análisis de Viabilidad

**Proyecto:** Trend Analyzer
**Fecha del análisis:** Agosto 2026
**Objetivo:** Determinar si las fuentes de datos propuestas (Mastodon, Reddit, noticias/RSS, datasets históricos) pueden obtenerse mediante APIs reales y funcionales, cuáles son sus límites, y si conviene usarlas reales o simuladas.

---

## 1. Mastodon — API real, viable sin fricción

Mastodon tiene una API REST y de streaming completamente abierta y documentada, sin necesidad de aprobación manual ni pago.

- **Timelines públicos reales:** existen endpoints para timeline público federado, timeline local, y timeline por hashtag, disponibles desde la versión 1.0 del protocolo, incluyendo el timeline público filtrado por medios desde la versión 2.4.
- **Streaming en vivo:** hay un endpoint de streaming dedicado que permite recibir toots en tiempo real vía WebSocket/SSE, ideal para el componente de ingesta streaming (< 5 min de latencia) exigido por el proyecto.
- **Límites:** 300 solicitudes por cada ventana de 5 minutos por cuenta de usuario (puede variar según la instancia), y 7500 solicitudes por 5 minutos por IP para llamadas anónimas/no autenticadas.
- **Registro:** se puede crear una app en cualquier instancia (mastodon.social, etc.) en minutos, sin revisión manual.

**Veredicto:** totalmente viable como fuente real. Es, de las tres fuentes evaluadas, la que menos fricción de acceso presenta.

---

## 2. Reddit — funcional, pero con un obstáculo serio de acceso

La API existe y es técnicamente sólida, pero el panorama de acceso cambió significativamente durante 2025–2026.

- **Límites técnicos:** 100 solicitudes por minuto para apps autenticadas por OAuth en el tier gratuito, medidas en una ventana móvil de 10 minutos, visible en tiempo real vía el header `X-Ratelimit-Remaining`.
- **El problema real no es el rate limit, es el registro:** Reddit cerró el registro de apps self-service a finales de 2025 con su "Responsible Builder Policy". Ahora cada cliente OAuth nuevo, gratuito o pagado, pasa por un ticket de aprobación manual con cola lenta y opaca, con posibilidad real de rechazo silencioso.
- **Uso comercial vs. no comercial:** el monitoreo de marca o competidores no califica como "proyecto personal" aunque sea gratuito — Reddit lo considera uso comercial y exige la vía de aprobación paga, lo cual es ambiguo para un caso de estudio universitario que simula una empresa.
- **Tier pagado:** de necesitarse, ronda los 12.000 USD/mes para 50 millones de llamadas, o ~0.24 USD por 1.000 llamadas — inviable con créditos educativos.
- **Alternativa real:** existe un sustituto de Pushshift, gratuito, sin autenticación, con datos de Reddit desde diciembre de 2005 hasta el presente y ~120.000 solicitudes por hora, disponible como REST API o archivos Parquet consultables con DuckDB. Limitación: la búsqueda de texto completo solo funciona dentro de un subreddit específico, sin búsqueda global de palabras clave.

**Veredicto:** la API oficial es un riesgo de cronograma — se puede solicitar acceso y quedar en cola de aprobación sin respuesta durante semanas, justo cuando el proyecto tiene sprints con fecha fija. El archivo tipo-Pushshift es una alternativa real más confiable si se necesitan datos reales de Reddit sin depender de la aprobación oficial.

---

## 3. Noticias (News API vs. RSS) — la "API de noticias" es la trampa, no el RSS

- **NewsAPI.org** (la más citada en tutoriales): el plan Developer es gratuito, pero restringido solo a entornos de desarrollo, limitado a 100 solicitudes por día, con artículos retrasados, y no se puede usar en ningún entorno de producción o staging. El CORS solo funciona en localhost — no sería desplegable en una arquitectura cloud como la planeada.
- **GNews:** mismo problema — 100 solicitudes/día gratis, pero también bloqueada para uso comercial/producción en el tier gratuito.
- **RSS puro (sin "API de noticias" comercial):** aquí no hay restricción real. Los feeds RSS de medios (El Tiempo, El Colombiano, Portafolio, etc.) son públicos, no requieren key, no tienen rate limit impuesto por un proveedor tercero, y encajan con la fuente "consultada periódicamente" del alcance del proyecto. Es más simple de integrar con Cloud Scheduler que cualquier API de noticias comercial.

**Veredicto:** evitar las "News APIs" comerciales para producción; usar RSS directo de medios colombianos/regionales como fuente real de noticias.

---

## 4. Conclusión: ¿simular o usar fuentes reales?

Se recomienda un **enfoque híbrido**, no una elección binaria — esto además calza con lo que pide el profesor ("el reto principal es cómo se van a obtener o generar mediante datos sintéticos").

| Fuente | Recomendación | Motivo |
|---|---|---|
| **Mastodon** | Real, 100% | API abierta, streaming real, sin fricción de aprobación. Mejor demo de "ingesta en tiempo real". |
| **RSS de noticias** | Real, 100% | Gratis, sin key, sin bloqueo de producción. No usar NewsAPI.org/GNews comerciales. |
| **Reddit** | Real con plan B | Solicitar el acceso OAuth cuanto antes (la cola de aprobación puede tardar semanas). Si para el Sprint 1 no ha llegado la aprobación, usar el archivo abierto tipo-Pushshift como fuente real alternativa, o datos sintéticos solo como contingencia de última instancia — documentando esa decisión en la matriz de riesgos. |
| **Datasets históricos** | Real | Usar datasets abiertos existentes (Kaggle, Hugging Face, archivos de Reddit/noticias) en vez de generarlos sintéticamente — más rápido y más creíble para la evaluación. |

### Razón de fondo

El curso evalúa explícitamente la definición de "producto de datos" con esquema, calidad y frecuencia de actualización reales — esto es mucho más defendible con datos reales imperfectos (nulos, duplicados, formatos inconsistentes) que con datos simulados "limpios", porque la capa de gobernanza y calidad (Great Expectations, linaje, catálogo) necesita fricción real para demostrar valor.

Simular todo ahorraría dolores de cabeza de registro/aprobación, pero restaría puntos de credibilidad justo en los componentes que más pesan en la rúbrica: Gestión y Gobernanza de Datos, y DataOps.

### Próximo paso sugerido

1. Solicitar cuanto antes el acceso OAuth a la API de Reddit (para no perder tiempo de cola de aprobación).
2. Armar la lista de feeds RSS colombianos relevantes para el caso de bebidas/consumo masivo.
3. Documentar en la matriz de riesgos del proyecto la contingencia de Reddit (archivo tipo-Pushshift o datos sintéticos) en caso de que la aprobación no llegue a tiempo.
