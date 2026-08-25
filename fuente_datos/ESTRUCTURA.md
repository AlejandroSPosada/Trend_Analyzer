# Esquema de Datos e Ingesta — Proyecto Trend Analyzer

**Fecha de actualización:** Agosto 2026  
**Propósito:** Definir los esquemas de entrada para las fuentes de datos reales integradas (`Mastodon` y `RSS Noticias`), detallando sus atributos, tipos de datos y transformaciones requeridas para el modelado en el pipeline.

---

## 1. Mastodon (Timeline Público)

* **Tipo de ingesta:** Streaming / API REST polling
* **Formato original:** JSON
* **Estado:** Operativo (HTTP 200)

Los campos más importantes para el procesamiento/analítica son id, created_at, content

### Tabla de Atributos

| Campo / Ruta JSON | Tipo de Dato | Ejemplo Registro 1 | Ejemplo Registro 2 | Requerido en Pipeline |
| :--- | :--- | :--- | :--- | :--- |
| `id` | String | `"117156446756584402"` | `"117156370233429584"` | **Sí** (Clave Primaria) |
| `created_at` | Timestamp (ISO8601) | `"2026-08-25T13:45:08.445Z"` | `"2026-08-25T13:25:40.735Z"` | **Sí** (Eje Temporal) |
| `edited_at` | Timestamp / Null | `null` | `"2026-08-25T13:30:50.137Z"` | No |
| `visibility` | String | `"public"` | `"public"` | No (Filtro) |
| `language` | String (ISO 639-1) | `"es"` | `"es"` | **Sí** (Filtro de idioma) |
| `url` | String (URL) | `"https://mastodon.social/@desdeabajo/117156446756584402"` | `"https://mastodon.social/@desdeabajo/117156370233429584"` | **Sí** (Linaje / Fuente) |
| `content` | String (HTML) | `"<p>#Colombia 🇨🇴 \| Después de recorrer un trayecto...</p>"` | `"<p>#Colombia 🇨🇴 \| (...) Nos acercamos a Henry Ramírez...</p>"` | **Sí** (Texto principal) |
| `replies_count` | Integer | `0` | `0` | **Sí** (Métrica de Engagement) |
| `reblogs_count` | Integer | `0` | `0` | **Sí** (Métrica de Engagement) |
| `favourites_count` | Integer | `0` | `0` | **Sí** (Métrica de Engagement) |
| `account.id` | String | `"113872789823214347"` | `"113872789823214347"` | **Sí** (Dimensión Usuario) |
| `account.username` | String | `"desdeabajo"` | `"desdeabajo"` | **Sí** (Dimensión Usuario) |
| `account.followers_count` | Integer | `304` | `304` | **Sí** (Métrica de Alcance) |
| `account.statuses_count` | Integer | `1874` | `1874` | No |
| `tags` | Array[Object] | `[{"name": "colombia"}, {"name": "choco"}, ...]` | `[{"name": "colombia"}, {"name": "buenaventura"}, ...]` | **Sí** (Clasificación Temática) |
| `media_attachments` | Array[Object] | `[{"type": "image", "url": "https://files.mastodon.social/..."}]` | `[{"type": "image", "url": "https://files.mastodon.social/..."}]` | No (Enriquecimiento) |
| `application.name` | String | `"Web"` | `"Web"` | No |

### Notas de Transformación (Mastodon)
- **Limpieza de Texto:** El campo `content` contiene etiquetas HTML (`<p>`, `<a>`, `<span>`). Se debe aplicar una etapa de Parsing/Regex para extraer solo el texto plano preprocesado para NLP.
- **Normalización de Tags:** Extraer el atributo `name` del array `tags` para generar un vector/lista plana de palabras clave.

---

## 2. RSS / Noticias (Ej. El Tiempo)

* **Tipo de ingesta:** Scheduled Batch (Micro-batching vía Cloud Scheduler)
* **Formato original:** XML / RSS (Parsed vía `feedparser` a Diccionario)
* **Estado:** Operativo (HTTP 200)

En los feeds RSS parsed de feedparser, summary contiene la bajada o entradilla de la noticia, title la cabecera, published la fecha con zona horaria (-05:00) y tags las categorías del portal.

### Tabla de Atributos

| Campo / Ruta JSON | Tipo de Dato | Ejemplo Registro 1 | Ejemplo Registro 2 | Requerido en Pipeline |
| :--- | :--- | :--- | :--- | :--- |
| `id` | String (URL / URI) | `"https://www.eltiempo.com/economia/sectores/exclusivo-drummond-..."` | `"https://www.eltiempo.com/economia/sectores/el-nino-podria-impulsar-..."` | **Sí** (Clave Primaria) |
| `title` | String | `"Exclusivo \| Drummond renuncia a su regasificadora..."` | `"El Niño podría impulsar el precio del café..."` | **Sí** (Titular) |
| `published` | String (ISO8601) | `"2026-08-25T00:30:00-05:00"` | `"2026-08-24T23:10:19-05:00"` | **Sí** (Eje Temporal) |
| `published_parsed` | Array (Tuple UTC) | `[2026, 8, 25, 10, 30, 0, 1, 237, 0]` | `[2026, 8, 25, 4, 10, 19, 1, 237, 0]` | No (Opcional si usa `published`) |
| `link` | String (URL) | `"https://www.eltiempo.com/economia/sectores/exclusivo-drummond-..."` | `"https://www.eltiempo.com/economia/sectores/el-nino-podria-impulsar-..."` | **Sí** (Linaje) |
| `summary` | String | `"En esta entrevista, el country manager de Drummond Energy..."` | `"Brasil concentra buena parte de la oferta global y..."` | **Sí** (Resumen de noticia) |
| `tags` | Array[Object] | `[{"term": "Sectores", "scheme": null, "label": null}]` | `[{"term": "Sectores", "scheme": null, "label": null}]` | **Sí** (Categoría del Medio) |
| `links` | Array[Object] | `[{"rel": "alternate", ...}, {"rel": "enclosure", "href": "https://..."}]` | `[{"rel": "alternate", ...}, {"rel": "enclosure", "href": "https://..."}]` | No (Multimedia) |
| `slash_comments` | String / Integer | `"0"` | `"0"` | No |

### Notas de Transformación (RSS)
- **Estandarización Temporal:** Convertir `published` a UTC unificado para cruzar tendencias con Mastodon.
- **Concatenación de Contenido:** Para análisis de sentimiento/temas, concatenar `title` + `" "` + `summary`.

---

## 3. Estado de la Fuente: Reddit

```text
[Reddit] Error: Faltan REDDIT_CLIENT_ID / REDDIT_CLIENT_SECRET. Regístralos como variables de entorno.