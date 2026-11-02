# TrendAnalyzer · GLOSSARY

> Glosario de componentes, tecnologías, conceptos y **documentos del repositorio**.
> Cada entrada tiene dos partes: **Qué es** (la tecnología o término explicado en general) y **Rol en TrendAnalyzer** (qué hace concretamente aquí).
> **Versión 2.64 · 1 nov 2026** · Alineado con `ARCHITECTURE.md` v2.64 y `PLAN.md` (todas las fases cerradas). El diagrama `docs/trend_analyzer_multicloud.drawio` está en la v2.63.
> Si este glosario y `ARCHITECTURE.md` discrepan, manda `ARCHITECTURE.md`; este archivo se corrige en el mismo commit.

---

## Índice

1. [Visión general en 30 segundos](#1-visión-general-en-30-segundos)
2. [El repositorio y sus documentos](#2-el-repositorio-y-sus-documentos)
3. [Fuentes y simulación](#3-fuentes-y-simulación)
4. [Google Cloud Platform (plano caliente)](#4-google-cloud-platform-plano-caliente)
5. [Puente GCP → AWS](#5-puente-gcp--aws)
6. [Amazon Web Services (plano de lakehouse)](#6-amazon-web-services-plano-de-lakehouse)
7. [Procesamiento y NLP](#7-procesamiento-y-nlp)
8. [DataOps e infraestructura](#8-dataops-e-infraestructura)
9. [Calidad y gobernanza](#9-calidad-y-gobernanza)
10. [Conceptos transversales](#10-conceptos-transversales)
11. [Fuera del alcance (mencionados pero no usados en v2)](#11-fuera-del-alcance-mencionados-pero-no-usados-en-v2)
12. [Tabla resumen: componente → rol](#12-tabla-resumen-componente--rol)

---

## 1. Visión general en 30 segundos

TrendAnalyzer analiza **tendencias y emociones** en conversaciones digitales sobre marcas colombianas de bebidas (Postobón como marca propia; Pepsi y Sprite como competidores). Funciona en **dos nubes**:

- **GCP** = plano *caliente*: recibe los eventos en segundos, los valida, deduplica, anonimiza, les aplica NLP en español, detecta tendencias y avisa por correo en minutos.
- **AWS** = plano de *lakehouse*: guarda todo en capas (Raw → Trusted → Curated) junto con 180 días de histórico, lo modela y lo sirve a un tablero.

Las fuentes (**Reddit, Facebook e Instagram**) están **simuladas**: un generador de comentarios en español colombiano produce el texto y el simulador inventa el resto (usuarios, engagement, fechas). Todo lo que viene después es un pipeline real. El objetivo es **ingeniería de datos** (ingesta, calidad, linaje, latencia), no conclusiones de negocio válidas.

```
Generador → ta-stream-simulator → Pub/Sub → Dataflow ─┬─► BigQuery (posts_enriched, trend_windows)
                                                       ├─► GCS (raw/, enriched/, dlq/)
                                                       └─► ta-alerts → notificador → correo
                              enriched/ ── gcs-to-s3 (cada 15 min, OIDC) ──┐
                                                                           ▼
  S3 raw/ (+ histórico) → Glue (cada hora) → trusted/ → curated/ → Athena → QuickSight (SPICE)
```

**Estado (v2.63):** las 9 fases y sus 28 lotes están cerrados, las 48 HU cumplidas y el ensayo completo de la sustentación pasó en la nube el 29-oct-2026 (`PLAN.md` §3).

---

## 2. El repositorio y sus documentos

El repositorio se organiza en dos carpetas con una regla simple: **en `docs/` se documenta; en `project/` se construye**. Los workflows de CI/CD, `.gitignore`, `.pre-commit-config.yaml` y la configuración de gitleaks viven en la raíz.

```
.                       README.md · .gitignore · .pre-commit-config.yaml · .gitleaks.toml · .github/workflows/ (ci.yml, cd.yml)
├── docs/               ARCHITECTURE · PLAN · CHANGELOG · HU · INFORME · SUSTENTACION · DEMO · RUNBOOK
│                       GOVERNANCE · CATALOG · LINEAGE · ACTIVE_SERVICES · GLOSSARY · PREGUNTAS · NOTAS
│                       validation/ (una guía por lote) · diagrama .drawio / .jpg
└── project/            contracts/ · infra/ · scripts/ · src/ · evidencia/ · docker-compose*.yml · LOCAL_DEV.md
```

### `ARCHITECTURE.md` — la fuente de verdad

**Qué es:** el documento maestro de arquitectura y contratos (versión vigente **2.64**). Un *documento vivo*: se versiona y se actualiza con cada decisión. El historial de versiones está en `CHANGELOG.md`, y el plan y el estado, en `PLAN.md`.

**Rol en TrendAnalyzer:** es **lo más importante del repositorio**, porque está pensado para que baste con leerlo (o adjuntarlo a una petición) para construir cualquier pieza sin ver el código ya hecho. Contiene, entre otros:

| Sección | Para qué sirve |
|---|---|
| §0 | Cómo usar el documento: qué leer según lo que se quiera hacer. |
| §1 | La decisión central (fuentes simuladas) y las diferencias con el diagrama FINAL y el README. |
| §2 | Diagrama de la arquitectura objetivo (Mermaid). |
| §3 | El simulador y el generador (§3.1–§3.12) y una subsección por lote con lo que entregó y sus «Interfaces del lote» (§3.13–§3.33). |
| §4 | Flujo extremo a extremo, etapa por etapa, con su latencia objetivo. |
| **§5** | **Los contratos**: envelope, Pub/Sub, GCS, BigQuery, puente, S3/Glue/Athena/QuickSight, IAM, outputs de Terraform, reglas de calidad (§5.10) y parámetros operativos (§5.12). Es la sección que hace innecesario enviar código. |
| §6 | Estructura del repositorio. |
| §7 | Remite a `PLAN.md` (fases y lotes). |
| §8 | Protocolo para pedir un lote de trabajo y cómo se entrega su validación. |
| §9 | Las 37 ADR, los riesgos y las preguntas abiertas de la Fase 0. |
| §10 | Remite a `PLAN.md` §3 (estado). |

**Regla de oro:** si dos componentes se comunican, su interfaz vive en §5; si algo no está en §5, no es una interfaz y ningún otro componente debe depender de él. **Cuando una interfaz cambia, se cambia primero en `ARCHITECTURE.md` y luego en el código**, en el mismo commit.

### `GLOSSARY.md` — este archivo

**Qué es:** el vocabulario del proyecto: tecnologías, componentes y términos propios.

**Rol en TrendAnalyzer:** da un lenguaje común al equipo (Miguel, Camila y Alejandro) y sirve de apoyo para el informe y la sustentación. Explica el *qué* y el *para qué* de cada pieza; el *cómo exacto* (formatos, nombres) está en `ARCHITECTURE.md` §5. Se actualiza cada vez que aparece un término, componente o decisión nueva.

### `HU.md` — historias de usuario

**Qué es:** una **historia de usuario (HU)** describe una necesidad desde quien la tiene ("como analista quiero…") junto con sus **criterios de aceptación**, que dicen cuándo está cumplida. `HU.md` reúne 48, en formato **HU-000** (HU-001 a HU-048), con una tabla resumen de estado.

**Rol en TrendAnalyzer:** define **qué significa "terminado"**. Cada fase y cada lote de `PLAN.md` §1 declara las HU que cubre, y la **matriz de trazabilidad** (`PLAN.md` §2) asigna cada HU a exactamente un lote (o a la Fase 0). Una fase se cierra cuando todas sus HU cumplen sus criterios; hoy las 48 están cumplidas. `HU.md` es la fuente de verdad de las historias: si se agrega o reasigna una, se cambia allí y en `PLAN.md` §2 en el mismo commit. Las antiguas *US* del README se relacionan con las HU (p. ej. US-14 → HU-031).

### `PLAN.md` — fases, trazabilidad y estado

**Qué es:** el plan de trabajo por fases y lotes, sin fechas.

**Rol en TrendAnalyzer:** §1 describe cada fase y sus lotes con las HU que cubren; §2 es la matriz Fase · Lote → HU con su responsable; §3 es **la única tabla de estado** (✅ cerrado · 🟨 en curso · ⬜ pendiente), con una nota por lote: qué se cerró, con qué validación y qué contratos se tocaron. Convención: cerrar un lote solo edita su fila de §3. No define contratos: para cualquier interfaz manda `ARCHITECTURE.md` §5.

### `CHANGELOG.md` — historial de versiones

**Qué es:** el registro de cambios del proyecto, una entrada por versión, la más reciente primero.

**Rol en TrendAnalyzer:** cada versión (v2.2 → v2.64) dice qué lote la motiva y qué cambió por área: código, contratos, decisiones, correcciones encontradas al validar y documentos. Es donde quedan fechadas las **decisiones del usuario** y los problemas que aparecieron en las validaciones (filas «Corregido»). Antes vivía en `ARCHITECTURE.md` §1.4.

### `README.md`

**Qué es:** la puerta de entrada del repositorio.

**Rol en TrendAnalyzer:** resume entornos, marcas, decisiones del generador, stack, estado por fases, estructura y un *quick start*. Es breve y apunta a `ARCHITECTURE.md` para el detalle; no debe contradecirlo.

### `NOTAS.md`

**Qué es:** notas de trabajo del equipo.

**Rol en TrendAnalyzer:** espacio informal para apuntes. **No es fuente de verdad**: lo que se decida y deba respetar el código se pasa a `ARCHITECTURE.md`.

### `INFORME.md` — informe técnico

**Qué es:** el informe técnico final del proyecto (HU-044).

**Rol en TrendAnalyzer:** resume el problema, por qué las fuentes son simuladas, la arquitectura, las decisiones principales, los contratos, la **tabla de KPI medidos** frente al objetivo, las limitaciones, los modelos de NLP con su licencia y cómo reproducir cada cifra. La tabla de KPI no se escribe a mano: la genera `kpi_report.py` desde la evidencia de las validaciones, y una prueba exige que esté al día (ADR-37).

### `SUSTENTACION.md` y `DEMO.md` — guiones

**Qué es:** el guion de la sustentación (HU-045) y el de la demo del pico.

**Rol en TrendAnalyzer:** `SUSTENTACION.md` dice qué encender desde T − 80 min (el baseline de las alertas necesita ~66 min de datos), qué muestra cada integrante en ~20 min, la ruta forzada hasta el tablero, el plan B y el registro del ensayo del 29-oct-2026. `DEMO.md` detalla la ola de demo con `demo_scenario.py`.

### `RUNBOOK.md` — operación

**Qué es:** el manual de operación (HU-045).

**Rol en TrendAnalyzer:** cómo encender y apagar, reiniciar el simulador, reprocesar desde `raw/` (el lakehouse se reconstruye entero desde S3; el stream no se reinyecta), atender la DLQ y qué hacer ante cada alerta (tendencia, operación, gasto y presupuesto).

### `ACTIVE_SERVICES.md` — qué cuesta dinero

**Qué es:** la guía de los servicios que cobran mientras están encendidos.

**Rol en TrendAnalyzer:** dice cómo encender, comprobar y apagar cada servicio, en local y en la nube, con `on.py`, `off.py` y `status.py` o con los comandos manuales. Es la referencia antes y después de cada validación o demo.

### `GOVERNANCE.md`, `CATALOG.md` y `LINEAGE.md` — gobierno de datos

**Qué es:** los tres entregables de gobernanza del Lote 5.3 (HU-034, HU-035 y HU-036).

**Rol en TrendAnalyzer:** `GOVERNANCE.md` recorre las 11 áreas de DAMA-DMBOK con su evidencia y sus brechas, y trae la nota de la Ley 1581. `CATALOG.md` es el diccionario de datos **generado** desde las DDL (11 tablas, 108 columnas, cada una con su clase de privacidad). `LINEAGE.md` es el linaje de extremo a extremo y por columna, con un diagrama Mermaid hecho a mano (ADR-29).

### `docs/validation/` — guías de validación

**Qué es:** una guía paso a paso por cada lote validado (`validation-2-2.md` a `validation-8-1.md`).

**Rol en TrendAnalyzer:** cada guía va por partes (preflight, pruebas, Terraform, local, nube y apagado) con el resultado esperado de cada paso, más el cierre y los problemas frecuentes. Su `project/scripts/validate_{fase}_{lote}.py` la corre de forma automática (ver *Validación de un lote*).

### `PREGUNTAS.md` — banco de preguntas

**Qué es:** preguntas abiertas de sustentación por rol, con respuestas guía, en formato Obsidian.

**Rol en TrendAnalyzer:** apoyo para preparar la sustentación. No es fuente de verdad.

### Diagrama (`trend_analyzer_multicloud.drawio` y `.jpg`)

**Qué es:** el diagrama editable de la arquitectura (formato draw.io) y su exportación a imagen.

**Rol en TrendAnalyzer:** la vista gráfica que muestra el README. El `.drawio` está en la **v2.63**: además de la arquitectura base (simulador, Pub/Sub, Dataflow, puente, lakehouse y QuickSight), muestra la rama de alertas (`ta-alerts` → notificador → correo), Cloud Build con `ta-images`, la observabilidad (Monitoring, CloudWatch y SNS), el CI/CD con OIDC y los costos. El `.jpg` se exporta a mano desde el `.drawio` con diagrams.net. El diagrama en texto es el Mermaid de `ARCHITECTURE.md` §2.

### `project/contracts/`

**Qué es:** la copia **legible por máquina** de los contratos de §5: JSON Schemas (envelope, envelope enriquecido, un payload por red social, atributos de Pub/Sub), DDL de BigQuery y Athena, ejemplos válidos e inválidos y pruebas.

**Rol en TrendAnalyzer:** convierte los contratos en algo **verificable**. Sus 18 pruebas comprueban que BigQuery, Athena y el envelope son coherentes entre sí; el simulador se valida contra los schemas con 1.000 eventos por fuente; las tablas de Glue (`tables.json`) y los schemas de BigQuery se generan desde estas DDL con pruebas de sincronía. Los comentarios de las DDL de Athena son además la fuente del catálogo (§9). Todo corre en la CI en cada PR. Se entregó en el Lote 0.1 (HU-001).

### Otras carpetas de `project/`

| Ruta | Contenido |
|---|---|
| `infra/{gcp,aws}/` | Terraform: `bootstrap/` (estado remoto), un módulo por pieza en `modules/` y el entorno en `environments/dev/` (4 raíces, ~30 módulos). |
| `scripts/` | Puntos de entrada: `on.py`, `off.py`, `status.py`; `validate_{fase}_{lote}.py`; `demo_scenario.py`, `smoke_e2e.py`, `kpi_report.py`, `quicksight.py`, `run_dq.py`, `profile_data.py`; `load_historical.py`, `export_gcs_to_s3.py`, `generate_comments.py`; `build_*_image.sh`, `dataflow_job.sh`, `dataflow_template.py`, `local_*.sh` y `verify_*.py`. |
| `src/common/` | Librería compartida (envelope, `event_id`, HMAC, hashtags, `label_map`, fila enriquecida, Parquet, llave HMAC y tablas de BigQuery) que usan Dataflow **y** el loader histórico, para que stream e histórico sean comparables. |
| `src/simulator/` | Simulador, generador de comentarios, caos, escenario de demo y Dockerfile. |
| `src/ingest_check/` | Verificador de la ingesta y topología del emulador (Lotes 1.4 y 1.5). |
| `src/streaming/` | Pipeline de Dataflow: validación, deduplicación, DLQ, anonimización, NLP, tendencias, alertas y plantilla Flex. |
| `src/historical/`, `src/exporter/` | Loader histórico (Lote 3.1) y exportador `gcs-to-s3` (Lote 3.2). |
| `src/lakehouse/` | Jobs de Glue Raw → Trusted → Curated (con aristas), catálogo (`catalog_spec`) y consultas. |
| `src/quality/`, `src/dq/` | Profiling (Lote 5.1) y Great Expectations (Lote 5.2). |
| `src/smoke/`, `src/serving/`, `src/alerts/`, `src/demo/`, `src/report/` | Prueba de humo, tablero de QuickSight, notificador de alertas, cliente del escenario de demo y tabla de KPI del informe. |
| `src/ops/` | Encendido, apagado y estado, y la lógica de cada `validate_*.py`, con sus pruebas. |
| `evidencia/` | JSON y logs que dejan las validaciones; de ahí sale la tabla de KPI del informe. |
| `docker-compose*.yml`, `local/`, `LOCAL_DEV.md` | Entorno local sin credenciales: emulador de Pub/Sub, simulador, validador y Dataflow con `DirectRunner` (también con NLP). |

### Lote

**Qué es:** una unidad de trabajo de tamaño razonable, numerada `fase.lote` (p. ej. Lote 2.2). El plan tiene 28.

**Rol en TrendAnalyzer:** cada lote entrega un conjunto cerrado de archivos con pruebas, en las rutas de `ARCHITECTURE.md` §6, junto con qué criterios de aceptación de cada HU cubre y una lista de **"contratos tocados"** (cambios que exigen actualizar §5 primero). Se pide con la plantilla de §8. Pasa por dos estados: **entregado** (código y pruebas listos) y **cerrado** (validado de punta a punta, casi siempre en la nube). Al cerrarlo se edita solo su fila en `PLAN.md` §3. Los 28 lotes están cerrados.

### Validación de un lote (`validate_*.py`) y evidencia

**Qué es:** la comprobación reproducible de que un lote funciona, desde las pruebas hasta la nube.

**Rol en TrendAnalyzer:** cada lote validado tiene una guía (`docs/validation/validation-{fase}-{lote}.md`), un script (`project/scripts/validate_{fase}_{lote}.py`, con su lógica en `src/ops/` y pruebas sin red) y evidencia en `project/evidencia/` (`validacion_{fase}_{lote}.json` y los informes de los verificadores). Las partes suelen ser A (preflight), B (pruebas), C (Terraform: `plan`, `apply` y sin deriva), L (local), las de nube (D, E, F…) y el apagado (`off`). La evidencia nunca lleva secretos, `user_hash` ni textos completos.

### Fase

**Qué es:** agrupación de lotes por objetivo: 0 Alineación, 1 Simulador, 2 Streaming GCP, 3 Histórico y puente, 4 Lakehouse AWS, 5 Calidad y gobernanza, 6 Serving y alertas, 7 CI/CD y costos, 8 Informe.

**Rol en TrendAnalyzer:** ordena el plan por dependencias, **sin fechas** desde la v2.10, hacia dos entregas que se cumplen por contenido: **Sprint 2** (fases 0 a 5) y **Sprint Final** (fases 6 a 8). Las fases 1, 2 y 3 se trabajaron en paralelo una vez cerrada la Fase 0. El estado de cada una se marca en `PLAN.md` §3; las nueve están cerradas (✅).

### Responsables (M, C, A)

**Qué es:** iniciales de reparto de trabajo: **M** = Miguel (infra/IAM/CI-CD), **C** = Camila (pipelines), **A** = Alejandro (simulador/calidad).

**Rol en TrendAnalyzer:** aparecen en la matriz de trazabilidad y en la tabla de estado; el valor `owner=alejandro` también va como etiqueta en los recursos.

### ADR (*Architecture Decision Record*)

**Qué es:** registro corto de una decisión de arquitectura, con la alternativa descartada.

**Rol en TrendAnalyzer:** `ARCHITECTURE.md` §9 recoge **37 ADR**. Las más citadas: ADR-01 fuentes simuladas, ADR-02 simulador en Cloud Run, ADR-04 un solo envelope, ADR-05 NLP en español con RoBERTuito, ADR-06 anonimizar en el primer paso de cada plano, ADR-07 puente con Cloud Run Job y OIDC, ADR-08 partition projection, ADR-09 aristas en lugar de Neptune, ADR-12 región `us-central1`, ADR-13 mapeo categoría → sentimiento, ADR-14 estado remoto de Terraform, ADR-17 deduplicación en dos capas, ADR-20 tendencias en Dataflow, ADR-25 y ADR-26 Trusted y Curated con Glue Python (sin dbt), ADR-28 Great Expectations a demanda, ADR-29 catálogo en las DDL y linaje a mano, ADR-31 QuickSight con SPICE, ADR-32 alertas, ADR-33 ola de demo, ADR-34 CI/CD, ADR-35 costos, ADR-36 observabilidad y ADR-37 informe generado desde la evidencia. `INFORME.md` §4 resume las que más pesan.

### Decisión del usuario y valores adoptados por defecto

**Qué es:** dos formas de registrar quién decidió algo y con qué firmeza.

**Rol en TrendAnalyzer:** una **decisión del usuario** la tomó el equipo de forma explícita y queda fechada en `CHANGELOG.md` (p. ej. el 26-oct-2026: Curated sin dbt y el horario de Glue encendido cada hora). Un valor **adoptado por defecto** es una propuesta que se aplicó sin objeción; vive en `ARCHITECTURE.md` §5.12 y, para cambiarlo, se cambia ahí primero y luego en el código.

### Versionado de documentos (`vX.Y`) y `schema_version`

**Qué es:** dos números distintos. La versión del documento (hoy **v2.64**) cuenta los cambios de `ARCHITECTURE.md` y se registra en `CHANGELOG.md`. `schema_version` es la versión del **contrato del envelope** (hoy `"1.0"`).

**Rol en TrendAnalyzer:** el documento sube de versión con cualquier cambio. `schema_version` solo cambia con un cambio **incompatible** del envelope; aun así, como los schemas rechazan campos no declarados, un campo opcional nuevo exige actualizar primero el schema y a los consumidores. Otros contratos tienen su propia versión, como el mensaje `ta-alert` 1.0 o el watermark del exportador (`ta-export-watermark@1`), y algunas reglas llevan la versión en su nombre (`text-clean@1`, `trusted-to-curated@1.1`).

### Contratos en CI y pre-commit

**Qué es:** controles automáticos antes de aceptar un cambio.

**Rol en TrendAnalyzer:** en cada PR y push a `main`, la CI valida la salida del simulador contra `contracts/*.schema.json` (≥ 1.000 eventos por fuente, caos apagado), corre las 18 pruebas de contratos y la de identidad (nada publicado lleva `handle` ni `user_id`) y escanea todo el historial con gitleaks. En local, el pre-commit corre gitleaks y `terraform fmt`. `terraform.tfvars` y `backend.hcl` están en `.gitignore`; se versionan sus `*.example`.

---

## 3. Fuentes y simulación

### Reddit / Facebook / Instagram (fuentes simuladas)

**Qué es:** Reddit es un foro organizado en subreddits; Facebook es una red social con páginas de marca, publicaciones y reacciones; Instagram es una red visual centrada en publicaciones y sus comentarios.

**Rol en TrendAnalyzer:** son las tres fuentes públicas que la empresa querría monitorear. Solo se pueden consumir por APIs REST/OAuth (polling), que no es streaming real, y dependen de aprobaciones y límites de acceso (OAuth en Reddit, revisión de app y permisos en Meta). Por eso **no se consumen por API**: cada una se *simula*. El campo `source` indica qué red se simula (`reddit`, `facebook` o `instagram`) y el `payload` imita el estilo de cada red.

| Fuente | `kind` | Contexto simulado |
|---|---|---|
| Reddit | `post` 30 % · `comment` 70 % | Subreddit de una lista (`r/colombia`, `r/medellin`…) |
| Facebook | `post` 15 % · `comment` 85 % | Página de la marca comentada |
| Instagram | `comment` 100 % | Publicación de la cuenta de la marca |

### Generador de comentarios

**Qué es:** un módulo Python (`src/simulator/generator/comment_generator.py`) que produce pares `marca,comentario` a partir de plantillas, en español coloquial colombiano, con jerga, abreviaturas, typos y emojis.

**Rol en TrendAnalyzer:** es el **origen del texto**. No es un servicio aparte: corre en el mismo proceso del simulador y también en batch para el histórico. Se expone como una función pura `generate(rng, marca=None, sentimiento=None)` con un `random.Random` inyectado, lo que da determinismo. Sus **categorías** (`gusto`, `asco`, `nostalgia`, `adiccion` e `informativo`) se mapean a sentimiento (`gusto → positive`, `asco → negative`, `informativo → neutral`; `nostalgia` y `adiccion → null`, por ambiguas; ADR-13). Reemplazó a los datasets de emociones en inglés de la v2.1, que no hablaban de las marcas del negocio.

### Marcas y `brands.yaml`

**Qué es:** el archivo de configuración (`src/simulator/config/brands.yaml`) que define cada marca: clave canónica, nombre visible, alias, rol, página de Facebook y cuenta de Instagram.

**Rol en TrendAnalyzer:** v1 usa **Postobón** (`postobon`, propia), **Pepsi** (`pepsi`) y **Sprite** (`sprite`), ambas competidoras. CocaCola y Colombiana aparecen en el script original pero quedan fuera de v1. La **clave canónica** (minúsculas, sin tildes) es la que viaja en `brands[]` y la que usan los joins y `dim_brand`. El simulador elige la marca con pesos que varían con `seq`, para crear tendencias reproducibles.

### Hashtags (`hashtags[]`)

**Qué es:** etiquetas con `#` que las personas escriben en sus publicaciones (`#Postobón`, `#Colombia`).

**Rol en TrendAnalyzer:** el texto conserva cada hashtag tal como se escribió, y el campo `hashtags[]` los guarda en una **forma canónica**: minúsculas, sin `#`, sin tildes, solo `[a-z0-9_]`, sin repetidos y en orden de aparición (`#Postobón` ⇒ `"postobon"`). Así las claves de `edges_hashtag_cooccurrence` y los joins son estables, igual que las claves de marca.

### `ta-stream-simulator`

**Qué es:** un servicio propio (Python, asyncio) empaquetado como contenedor y desplegado en Cloud Run.

**Rol en TrendAnalyzer:** es el "falso internet". Con `min = max = 1` y CPU siempre asignada corre **3 workers**, uno por fuente: cada uno llama al generador, construye el envelope y lo publica a Pub/Sub cada 1 a 3 segundos (~0,5 mensajes/s por fuente, ~1,5 en total). Fabrica los campos que el generador no trae (usuarios, seguidores, engagement, hashtags, fechas, subreddit/página/cuenta, `kind` y `parent_id`). Vive en GCP, junto a Pub/Sub, para no cruzar credenciales entre nubes. Expone `GET /healthz` (sondas de Cloud Run), `GET /health` (el mismo estado, consultable desde fuera) y `/control/scenario`. Está **apagado por defecto** (`simulator_running = false`); lo encienden `on.py` o las validaciones, y el cursor sobrevive en GCS.

### Semilla (`SIM_SEED`) y reproducibilidad

**Qué es:** el valor inicial del que sale todo el aleatorio.

**Rol en TrendAnalyzer:** misma semilla + fuente + `seq` ⇒ mismo comentario. Cada evento usa su propio `Random(f"{seed}|{fuente}|{seq}")`; **no se usa `hash()` de Python**, que varía entre procesos y rompería la reproducibilidad. Los escenarios de demo son la excepción y quedan marcados en `sim.scenario`.

### Cursor (`sim-state`) y `seq`

**Qué es:** un cursor es un marcador de "voy por aquí", como un marcapáginas. `seq` es el número de orden de cada evento dentro de su fuente (0, 1, 2…); cada fuente lleva su propia cuenta.

**Rol en TrendAnalyzer:** el simulador guarda un JSON por fuente en `gs://{GCP_PROJECT}-ta-datasets/sim-state/`:

```
sim-state/reddit.json       {"seed": 7, "seq": 1207, "updated_at": "2026-10-29T14:03:12Z"}
sim-state/facebook.json     (igual, con su propio seq)
sim-state/instagram.json
```

- **Por qué hace falta.** Los comentarios son deterministas: misma semilla + fuente + `seq` ⇒ mismo comentario y mismo `event_id`. Si el simulador volviera a `seq = 0` en cada arranque, reenviaría todo lo que ya emitió, y todo serían duplicados. Con el cursor, tras un reinicio de Cloud Run o un `off.py` seguido de `on.py`, cada fuente sigue desde el último número guardado.
- **Cada cuánto se guarda.** Cada 50 mensajes por fuente (≈ 100 s al ritmo normal). Por eso un reinicio puede repetir hasta 50 eventos: si el último guardado dice 1200 y el servicio se cayó en el 1230, al volver emite otra vez del 1200 al 1229, con los mismos `event_id`, y Dataflow los descarta (ver *Deduplicación*).
- **Si cambia la semilla.** Si `SIM_SEED` no coincide con el `seed` del archivo, el cursor se ignora y la fuente empieza en `seq = 0`. No choca con lo anterior porque la semilla forma parte del `event_id`.
- **Quién lo escribe y quién lo lee.** Solo lo escribe el simulador (`sa-simulator` tiene `storage.objectUser` en el bucket). Lo leen, sin modificarlo, `verify_ingest.py` (compara el `seq` del cursor con el último que llegó a Pub/Sub y, con `restart-check`, que la cuenta siga después de un reinicio) y `preflight.sh`, que lo imprime antes de cada validación.
- **Historial.** El bucket tiene versionado: cada guardado deja una versión nueva y las anteriores se borran a los 7 días.
- **Al apagar.** `simulator_running = false` borra el servicio de Cloud Run, pero no el cursor: al volver a encender, sigue donde iba.
- **En local** (`docker-compose`), el cursor vive en el volumen Docker `sim-state`, montado en `/state` (`SIM_STATE_DIR`): `local_dev.sh down` lo conserva y `reset` lo borra.

### Snapshots del generador (`snapshots/`)

**Qué es:** un *snapshot* es una foto fija de unos datos en un momento dado. Aquí es un CSV con comentarios producidos por el generador.

**Rol en TrendAnalyzer:** es **opcional**: ningún componente del pipeline lo lee ni lo necesita.

- **Cómo se hace.** Desde `project/`, con `python scripts/generate_comments.py --n 50000 --seed 7 --out comentarios.csv`. El CSV trae `marca,comentario,sentimiento` (más `categoria` con `--categoria`): `marca` es la clave canónica (`postobon`) y `sentimiento` queda vacío en las categorías ambiguas. Si se quiere guardar, se sube a mano a `gs://{GCP_PROJECT}-ta-datasets/snapshots/comentarios-{seed}-{n}.csv`.
- **Reproducible.** La fila `i` sale de `Random(f"{seed}|batch|{i}")`: misma semilla y mismos argumentos dan el mismo archivo, byte a byte.
- **Para qué sirve.** Revisar el texto del generador sin encender nada (vocabulario, jerga, mezcla de sentimientos), mostrar una muestra en el informe o en la sustentación y poder regenerar exactamente esa misma muestra después.
- **Ojo: no son los comentarios del stream.** El simulador siembra cada evento con `{seed}|{fuente}|{seq}` y elige la marca con pesos que cambian con `seq`; el CSV usa `{seed}|batch|{i}`. Es el mismo generador y el mismo estilo, pero no los mismos comentarios.
- **Por qué no se versiona en git.** Pesa varios MB y se regenera en segundos con la misma semilla (50.000 filas en ~3,5 s), así que guardarlo en el repositorio no aporta nada.

### Escenarios de demo (`/control/scenario`)

**Qué es:** un endpoint protegido del simulador que, durante un tiempo, fuerza una marca y un sentimiento y sube la frecuencia de una red.

**Rol en TrendAnalyzer:** permite mostrar una alerta real en la sustentación sin esperar una tendencia (Lote 6.3, HU-040, ADR-33). La ola **se suma** al tráfico normal: la red emite ×`m` y solo la fracción `1 − 1/m` de los eventos lleva la marca y el sentimiento forzados, así las demás marcas no caen ni dan alertas falsas de `volume_down`. Admite `POST`, `GET` y `DELETE`; `multiplier` de 2 a 10 y `duration_s` de 60 a 3.600; un escenario a la vez, que termina solo. Los eventos quedan marcados en `sim.scenario`. Lo protege el IAM de Cloud Run (sin token responde 403) y lo dispara el presentador con `scripts/demo_scenario.py`. Por defecto: `postobon`, `negative`, ×3, 300 s, `instagram`.

### Caos controlado (`SIM_CHAOS_ENABLED`)

**Qué es:** inyección deliberada de defectos en los datos.

**Rol en TrendAnalyzer:** produce duplicados (0.5 %), eventos tardíos (1 %), campos nulos (1 %) y mensajes malformados (0.2 %) para que los módulos de **calidad** tengan algo real que detectar y medir. Se puede apagar para obtener la línea base de los KPIs. Los defectos se registran en `sim.chaos`.

### Envelope canónico (v1.0)

**Qué es:** el "sobre" JSON estándar que envuelve cada evento: id, origen, fuente, generador, `seq`, fechas, autor, texto, marcas, hashtags, un `payload` específico de cada red y el bloque `sim`.

**Rol en TrendAnalyzer:** es **el contrato más importante del proyecto**. Tanto el stream como el histórico lo usan, por eso ambos terminan en la misma tabla Trusted. Existe en dos formas: el **envelope** (con `author.user_id` y `author.handle`, solo hasta Dataflow/loader) y el **envelope enriquecido** (sin esos campos, con `user_hash`, engagement normalizado y resultados de NLP).

### `event_id`

**Qué es:** un identificador único por evento, calculado con `uuid5` a partir de origen, fuente, semilla y `seq`.

**Rol en TrendAnalyzer:** es **determinista**: reenviar el mismo evento tras un reinicio produce el mismo id, lo que permite a Dataflow deduplicar. Es la clave de unicidad de todo el sistema. Los duplicados se miden por `event_id`, **no por texto**: el generador usa plantillas y el mismo texto puede repetirse sin ser un duplicado.

### Ground truth (`sim.label_sentiment`)

**Qué es:** la respuesta "correcta" conocida, usada para evaluar un modelo.

**Rol en TrendAnalyzer:** el generador conoce el sentimiento del comentario que produjo (`positive`, `negative`, `neutral` o `null`). Sirve **solo** para medir la precisión del NLP; nunca se usa como feature ni como respuesta del pipeline. Como la asociación marca→sentimiento es artificial y el vocabulario acotado, las métricas serán optimistas, y el informe lo declara.

### Histórico (*full load*)

**Qué es:** una carga masiva única de registros pasados.

**Rol en TrendAnalyzer:** el Lote 3.1 generó **50.000 registros** de los últimos 180 días con el **mismo `EventFactory`** del simulador y otra semilla (`HIST_SEED = 3101`), repartidos por igual entre las tres fuentes. Los eventos se eligen según un **plan de tendencias** (estacionalidad semanal y horaria, crecimiento, Pepsi al alza, Postobón a la baja, una campaña de Sprite y una crisis de Postobón) para que haya historia que comparar. Usa el mismo envelope (`origin = historical`), el mismo pool de usuarios, la misma llave HMAC y el mismo NLP. Quedó en 12.102 archivos en `s3://…/raw/origin=historical/`; reejecutarlo no duplica nada.

### `EventFactory`

**Qué es:** la clase del simulador que fabrica un evento completo (envelope y mensajes de Pub/Sub) a partir de la fuente, el `seq` y la hora.

**Rol en TrendAnalyzer:** garantiza que el stream, el histórico, la prueba de humo y las olas de prueba produzcan exactamente el mismo tipo de evento: todos la reutilizan con distintas semillas en lugar de tener cada uno su propio generador.

### Centinelas y campo `generator`

**Qué es:** un *centinela* es un evento de prueba con identidad conocida que se sigue por el sistema; `generator` es el campo del envelope que dice quién produjo el evento.

**Rol en TrendAnalyzer:** los comentarios normales llevan `generator = comment-gen@1.0`. Los verificadores publican centinelas con `ta-verify@…` (p. ej. `ta-verify@2.1`, o la ola de prueba `ta-verify@6.2`), que el job de Trusted **descarta** para que no lleguen al análisis. La prueba de humo usa como centinela un comentario normal con la semilla reservada 990048, precedido de un calentamiento `ta-verify@2.1` (ADR-30).

---

## 4. Google Cloud Platform (plano caliente)

### Cloud Storage (GCS)

**Qué es:** almacenamiento de objetos (archivos) en la nube de Google. Un *bucket* es un contenedor de archivos con un nombre único en todo GCP. Las "carpetas" de adentro son en realidad prefijos del nombre de cada archivo (`raw/source=reddit/…`). Es barato y duradero, y se paga por lo guardado y por las operaciones.

**Rol en TrendAnalyzer:** hay dos buckets de datos, más el del estado de Terraform y uno que crea Cloud Build. Sus nombres empiezan con el ID del proyecto (`{GCP_PROJECT}-…`, p. ej. `project-0568d9ba-848f-413c-88b-ta-lake`) justamente porque tienen que ser únicos en todo GCP. Los dos de datos los crea Terraform (módulo `gcs_bucket`) con acceso uniforme y sin acceso público posible.

#### `{GCP_PROJECT}-ta-datasets`: estado del simulador

| Carpeta | Qué guarda | Quién escribe y quién lee | Retención |
|---|---|---|---|
| `sim-state/` | Un JSON por fuente con el cursor del simulador (ver *Cursor* en §3). | Escribe el simulador; leen `verify_ingest.py` y `preflight.sh`. | La versión vigente no expira; las anteriores, 7 días. |
| `snapshots/` | CSV opcionales del generador (ver *Snapshots* en §3). | Se suben a mano; nada los lee. | No expiran. |

El nombre es histórico: hasta la v2.1 guardaba los datasets de emociones en inglés de los que el simulador sacaba el texto. Desde la v2.2 el texto sale del generador y el bucket solo conserva estas dos carpetas.

#### `{GCP_PROJECT}-ta-lake`: salida de Dataflow

| Carpeta | Qué guarda | Quién escribe y quién lee | Retención |
|---|---|---|---|
| `raw/` | Cada evento válido y único **tal como llegó**, con el *handle* simulado, en `jsonl.gz` por `source/dt/hr`. Sirve para auditar y reprocesar. | Escribe Dataflow. Acceso restringido: es lo único que **nunca sale de GCP**. | 30 días |
| `enriched/` | Los mismos eventos ya anonimizados y con NLP, en Parquet con el esquema plano de `posts_enriched`. Es lo único que viaja a AWS. | Escribe Dataflow; lee el exportador, que no puede leer ninguna otra carpeta. | No expira |
| `dlq/` | Mensajes rechazados, con el cuerpo original y el motivo, en `jsonl.gz` por `dt`. | Escribe Dataflow. | 30 días |
| `export-state/` | `watermark.json`: hasta dónde copió el exportador (ver *Watermark* en §7). | Lo lee y escribe solo el exportador. | No expira |
| `_dataflow/` | `temp/` y `staging/` (temporales y el paquete de código que se sube al lanzar el job) y `templates/` (la especificación de la plantilla Flex, `ta-stream.json`). | `dataflow_job.sh`, `dataflow_template.py` y el propio job. | — |

Las particiones `dt`/`hr` de `raw/`, `enriched/` y `dlq/` son de la hora de **proceso** en UTC, y cada archivo se nombra por el inicio de su ventana. Ejemplo: `raw/source=reddit/dt=2026-10-29/hr=14/part-20261029T140500Z-p000.jsonl.gz`.

#### Otros buckets

- `{GCP_PROJECT}-ta-tfstate`: el estado remoto de Terraform del entorno `dev` de GCP (ver *Bootstrap de Terraform* en §8).
- `{GCP_PROJECT}_cloudbuild`: lo crea Cloud Build por su cuenta. Ahí sube el código de cada build antes de construir la imagen, por eso el CD necesita permiso de escritura en él.

### Cloud Run

**Qué es:** servicio de Google para ejecutar contenedores sin administrar servidores. Escala automáticamente y se paga por uso.

**Rol en TrendAnalyzer:** aloja dos servicios privados (solo los invoca quien tiene permiso de IAM). `ta-stream-simulator` corre con CPU siempre asignada y una sola instancia, para que no haya emisiones duplicadas ni se pise el cursor; el plan B por costo es una VM `e2-small` con el mismo contenedor (ADR-02). `ta-alert-notifier` recibe las alertas de `ta-alerts` y escala a cero. El *frontend* de Cloud Run reserva algunas rutas terminadas en `z`, por eso desde fuera se consulta `GET /health` y no `/healthz`.

### Cloud Run Job

**Qué es:** variante de Cloud Run para tareas que *corren, terminan y se apagan* (en lugar de quedarse escuchando).

**Rol en TrendAnalyzer:** ejecuta `ta-gcs-to-s3`, el exportador que copia datos enriquecidos de GCS a S3.

### Cloud Scheduler

**Qué es:** un "cron" administrado: dispara tareas según un horario.

**Rol en TrendAnalyzer:** `ta-gcs-to-s3-cada-15-min` lanza el exportador cada 15 minutos (UTC). Se crea **en pausa** (estado de ahorro) y lo reanudan `on.py` o las validaciones. Los escenarios de demo no lo usan: los dispara el presentador.

### Pub/Sub

**Qué es:** servicio de mensajería asíncrona. Un productor publica mensajes en un **topic** y uno o más consumidores los reciben mediante una **suscripción**. Desacopla a quien produce de quien consume y absorbe picos de tráfico.

**Rol en TrendAnalyzer:** es el "tubo" entre la simulación y el procesamiento, y también lleva las alertas.

| Recurso | Para qué |
|---|---|
| `ta-raw-reddit`, `ta-raw-facebook`, `ta-raw-instagram` | Un topic por fuente; el simulador publica aquí. |
| `ta-raw-{source}-df` | Suscripciones *pull* que consume Dataflow. |
| `ta-dlq` y `ta-dlq-df` | *Dead-letter queue*: mensajes que no pasan validación, y la suscripción con la que el mismo job los lleva a `dlq_events`. |
| `ta-alerts` y `ta-alerts-notifier` | Mensajes `ta-alert` que publica Dataflow, y la suscripción *push* hacia el notificador de Cloud Run. |
| `ta-alerts-incidents` | Copia de cada notificación de la política de correo, para medir la alerta sin leer el buzón. |
| `ta-verify-*` | Suscripciones temporales de los verificadores; ningún componente depende de ellas. |

Retiene mensajes 7 días y entrega **al menos una vez**, por eso la deduplicación es obligatoria. Cada mensaje del stream lleva los atributos `event_id`, `source`, `origin` y `schema_version`; `event_id` se usa como `id_label`.

### Emulador de Pub/Sub y entorno local

**Qué es:** una versión local de Pub/Sub que corre en Docker, sin credenciales de nube.

**Rol en TrendAnalyzer:** el `docker-compose` levanta el emulador, el simulador y un consumidor que valida contra los schemas, de modo que el simulador se desarrolle y pruebe antes de que exista la infraestructura (HU-047).

### Dataflow

**Qué es:** servicio administrado de Google para ejecutar pipelines de **Apache Beam**, tanto en modo streaming (datos continuos) como batch.

**Rol en TrendAnalyzer:** es el **cerebro del plano caliente**. El job `ta-stream-dev` (1 worker `e2-standard-2`, sin autoescalado, Streaming Engine, imagen `dataflow-nlp`) lee de Pub/Sub y, en segundos, por cada evento:

1. Valida el envelope (los inválidos van a la DLQ).
2. Deduplica por `event_id` y escribe `raw/`.
3. Anonimiza el autor con HMAC y normaliza el engagement.
4. Aplica NLP (sentimiento y emoción).
5. Escribe en BigQuery (`posts_enriched`) y en `enriched/`.
6. Calcula ventanas de tendencia (`trend_windows`).
7. Publica las alertas en `ta-alerts`, con *cooldown*.

Se lanza con `dataflow_job.sh` o desde la plantilla Flex (nunca los dos a la vez) y se **drena** fuera de las validaciones y demos (~0,25 USD/h encendido). Techo medido: 1,5 a 2,5 mensajes/s con el NLP en CPU.

### Plantilla Flex de Dataflow

**Qué es:** una forma de empaquetar un pipeline de Dataflow como una imagen de contenedor (el *lanzador*) más un archivo de especificación, para lanzarlo con parámetros sin tener el código fuente a mano.

**Rol en TrendAnalyzer:** la imagen `dataflow-launcher` y la especificación `ta-stream-{etiqueta}.json` en `_dataflow/templates/` permiten lanzar el mismo job con los parámetros de `terraform output` (`dataflow_template.py run`). Se eligió Flex porque la plantilla clásica no admite bien parámetros en tiempo de ejecución para streaming en Python (ADR-20).

### BigQuery

**Qué es:** almacén de datos analítico de Google. Permite consultar con SQL volúmenes enormes con muy baja latencia, sin administrar servidores.

**Rol en TrendAnalyzer:** es la **analítica caliente** en GCP (dataset `ta_stream`, creado por Terraform). Contiene:

- `posts_enriched`: un registro por post, ya anonimizado y con sentimiento y emoción, escrito con *streaming inserts* (~16 s desde Pub/Sub). Particionada por fecha (expira a 90 días) y agrupada por `source` y `brand_primary`.
- `trend_windows`: una fila por panel de cada ventana de 5 minutos por marca y fuente, con volumen, sentimiento promedio (con signo, de −1 a 1), z-scores y la bandera `is_alert` (90 días). La vista **`trend_windows_latest`** deja la versión vigente de cada ventana.
- `dlq_events`: mensajes rechazados, desde `ta-dlq-df` (30 días).
- `dq_results`: resultados de Great Expectations, una fila por comprobación (sin expiración).

### Secret Manager

**Qué es:** bóveda para guardar secretos (llaves, contraseñas, tokens) con control de acceso y versionado.

**Rol en TrendAnalyzer:** guarda la **llave HMAC de anonimización** (`ta-hmac-key`, 32 bytes en hex). Solo `sa-dataflow` puede leerla, desde el worker; el loader histórico la lee con las credenciales del desarrollador. El valor se carga fuera de Terraform (`hmac_key.sh`) para que no quede en el estado, y **no se rota**, porque cambiaría todos los `user_hash`. Ya no hay tokens de redes sociales.

### Cloud Monitoring + Logging

**Qué es:** servicios de Google para métricas, dashboards, alertas y recolección de logs.

**Rol en TrendAnalyzer:** cubren tres cosas, todas con aviso por correo:

- **Alertas de tendencia:** el notificador escribe un log estructurado (`alert=trend`) y la política basada en logs `ta-alerta-tendencia` envía el correo (y una copia a `ta-alerts-incidents`).
- **Operación** (Lote 7.2): el tablero «TrendAnalyzer · operación» (tasa de `ta-raw-*`, atraso, latencia p95 de lectura, marca de agua, DLQ, alertas y exportador) y cuatro alertas: `ta-op-dataflow-fallido`, `ta-op-latencia` (marca de agua > 5 min), `ta-op-dlq-creciente` (> 30 en 10 min) y `ta-op-exportador-fallido`.
- **Gasto por olvido** (Lote 7.3): `ta-gasto-dataflow-encendido` (> 4 h) y `ta-gasto-simulador-encendido` (> 8 h).

Todo el código escribe logs JSON a stdout. Por ahora el único destinatario es el correo de Alejandro (`alert_emails`, fuera del repositorio).

### `ta-alert-notifier` y política basada en logs

**Qué es:** una *política basada en logs* es una alerta de Cloud Monitoring que se dispara cuando aparece en Cloud Logging una línea que cumple un filtro.

**Rol en TrendAnalyzer:** cierra la cadena de las alertas de tendencia (Lote 6.2, ADR-32) sin SMTP ni secretos: Dataflow publica en `ta-alerts`, la suscripción *push* llama al servicio `ta-alert-notifier` con el ID token de `sa-alerts`, el servicio escribe un log `WARNING` con los datos de la alerta y la política `ta-alerta-tendencia` manda el correo. Cada alerta abre su propio incidente, así que llega un correo por alerta. Medido: de 87 a 160 s entre el fin de la ventana y la notificación.

### Service accounts (cuentas de servicio)

**Qué es:** identidades no humanas que usan los programas para autenticarse y tener permisos.

**Rol en TrendAnalyzer:** aplican **mínimo privilegio**:

- `sa-simulator`: publica en `ta-raw-*` y usa el bucket de datasets.
- `sa-dataflow`: consume Pub/Sub, publica en `ta-dlq` y `ta-alerts`, escribe en BigQuery y en el lake, lee la llave HMAC y descarga las imágenes de `ta-images`.
- `sa-exporter`: lee solo `enriched/` (condición de IAM), escribe solo en `export-state/` y emite el token OIDC para AWS.
- `sa-alerts`: identidad del notificador; es la única que puede invocarlo.
- `sa-github-cd`: identidad del CD de GitHub (publicar imágenes y leer el estado para el `plan`).

---

## 5. Puente GCP → AWS

### `gcs-to-s3` (exportador)

**Qué es:** un Cloud Run Job propio (`ta-gcs-to-s3`) que copia archivos de un bucket de GCS a un bucket de S3.

**Rol en TrendAnalyzer:** es el **único camino de datos entre las dos nubes** (Lote 3.2, ADR-22). Cada 15 minutos lee los Parquet nuevos de `enriched/`, con un *watermark* por fecha de creación y una hora de solape, y los sube **byte a byte** a `s3://…/raw/origin=stream/` con SSE-KMS y el mismo checksum que en GCS. Es idempotente: si el objeto ya existe con el mismo tamaño, lo omite, y solo avanza el watermark después de subir todo. Solo sale el dato ya anonimizado. Su Scheduler se crea en pausa.

### OIDC / `AssumeRoleWithWebIdentity`

**Qué es:** mecanismo de federación de identidad. En lugar de guardar llaves permanentes, un programa presenta un token firmado por su proveedor de identidad y la otra nube le entrega credenciales temporales.

**Rol en TrendAnalyzer:** se usa dos veces, **sin llaves estáticas**. (1) El exportador: `sa-exporter` presenta su ID token de Google y asume el rol `ta-gcp-exporter-role` en AWS; Google es un proveedor integrado en AWS (`accounts.google.com`) y la confianza exige la audiencia y el identificador de esa cuenta de servicio. El plan B con llaves en Secret Manager no hizo falta. (2) El CD de GitHub: el token de GitHub Actions, solo de este repositorio y de `main`, se cambia por credenciales de `sa-github-cd` en GCP y del rol `ta-github-cd` en AWS.

---

## 6. Amazon Web Services (plano de lakehouse)

### Amazon S3

**Qué es:** almacenamiento de objetos de AWS, equivalente a Cloud Storage. Es la base de casi cualquier data lake.

**Rol en TrendAnalyzer:** es el **Lakehouse**. Un único bucket (`ta-dev-{account_id}-lake`), privado, con SSE-KMS y acceso solo por TLS, organizado en capas:

| Carpeta | Contenido |
|---|---|
| `raw/` | Datos tal como llegan, ya anonimizados: `origin=stream` (copia de `enriched/`) y `origin=historical`. Solo los escriben sus productores. |
| `trusted/` | `trusted/posts/`: stream e histórico unidos, deduplicados, con tipos correctos y `text_clean`; un Parquet por `source/dt`. |
| `curated/` | Modelo analítico listo para consumo: hechos, dimensiones, agregados y aristas. |
| `quality/` | Reportes de cada corrida de los jobs de Glue (`raw_to_trusted/`, `trusted_to_curated/`, con copia en `_latest/`) y de Great Expectations (`ge/{run_id}/`); opcional, el del profiling. |
| `athena-results/` | Resultados de consultas de Athena (se borran a los 7 días). |
| `_glue/` | Scripts de los jobs (los sube Terraform) y su carpeta temporal. |

Todo se guarda en **Parquet + Snappy**.

### AWS Glue ETL

**Qué es:** servicio de integración de datos administrado. Ejecuta jobs de transformación (ETL) con Apache Spark o, para volúmenes pequeños, como scripts de Python (*Python shell*), sin administrar servidores.

**Rol en TrendAnalyzer:** mueve los datos entre capas con dos jobs **Glue Python shell 3.9 con pyarrow** (sin Spark ni dbt), en el workflow `ta-wf-raw-to-curated`:

- `ta-job-raw-to-trusted` (1 DPU, Lote 4.2): reconstruye Trusted entero desde los dos orígenes de Raw; descarta inválidos y centinelas, deduplica por `event_id` (se queda con la copia de menor `process_time`), corrige tipos y calcula `text_clean`.
- `ta-job-trusted-to-curated` (0,0625 DPU, Lotes 4.3 y 4.4): recalcula las 9 tablas de Curated (hechos, dimensiones, agregados y aristas), concilia antes de escribir y sube solo lo que cambió.

Son idempotentes: mismo input, mismos bytes. Una corrida del workflow cuesta ≈ 0,0196 USD (≈ 14,1 USD/mes con el horario cada hora).

### DPU (*Data Processing Unit*)

**Qué es:** la unidad con la que AWS Glue mide y cobra la capacidad de un job (1 DPU = 4 vCPU y 16 GB de memoria; un job Python shell puede usar 1 o 0,0625 DPU).

**Rol en TrendAnalyzer:** Trusted corre con 1 DPU (pico medido de 582 MB) y Curated con 0,0625 DPU (pico de 457 MB). Se eligió Python shell porque un job Spark exige un mínimo de 2 DPU por corrida, más caro para ~221.000 filas (ADR-25, ADR-26).

### Conciliación y reporte de corrida

**Qué es:** *conciliar* es comprobar que los totales de una salida cuadran con su entrada antes de darla por buena; el *reporte de corrida* es el registro de lo que hizo cada ejecución.

**Rol en TrendAnalyzer:** el job de Curated hace **16 comprobaciones** (p. ej. que `fact_posts` tenga las mismas filas que Trusted, que los agregados y las aristas sumen lo esperado y que las llaves foráneas existan) y, si una falla, **no escribe**. Los dos jobs dejan un reporte en `quality/{fecha}/…` y en `quality/_latest/` con lo leído, descartado, deduplicado y escrito, y el sha256 de cada archivo; la corrida siguiente lo usa para subir solo lo que cambió. Los reportes solo llevan conteos, sin datos de filas.

### Glue Data Catalog

**Qué es:** catálogo central de metadatos: qué tablas existen, sus columnas, tipos y ubicación de los datos.

**Rol en TrendAnalyzer:** es el **catálogo oficial** del proyecto (reemplaza a Dataplex). Define las bases `ta_raw`, `ta_trusted` y `ta_curated` con sus 11 tablas, que Terraform crea desde las DDL de `contracts/athena/` (sin crawlers) y Athena consulta. Cada tabla tiene su descripción y cada columna un comentario que empieza por su **clase de privacidad** (Lote 5.3, HU-034).

### Amazon Athena

**Qué es:** servicio para consultar archivos en S3 con **SQL estándar**, sin servidores ni cargas previas. Se paga por datos escaneados.

**Rol en TrendAnalyzer:** es la **capa de consulta** sobre el lakehouse. Usa el workgroup `ta-analytics` (motor v3, configuración forzada, resultados cifrados con SSE-KMS), que cancela toda consulta de más de **1 GiB** escaneado. Tiene consultas guardadas de ejemplo: dos sobre Raw y cinco sobre Curated (HU-030). QuickSight se conecta a través de Athena con el rol `ta-quicksight-role`.

### Partition projection

**Qué es:** una forma de decirle a Athena cómo se llaman y qué valores tienen las particiones, calculándolas en la consulta en lugar de registrarlas una a una (sin crawlers).

**Rol en TrendAnalyzer:** es más simple y barato que los crawlers (ADR-08). **Raw** se proyecta por origen, fuente, fecha y hora, con claves `part_origin`, `part_source`, `part_dt`, `part_hr` (el prefijo `part_` evita chocar con las columnas `origin` y `source`, que también viajan dentro del Parquet). **Trusted** se proyecta por `source` y `dt`, con `dt` igual a la fecha de `event_time` en UTC.

### Amazon EventBridge

**Qué es:** bus de eventos y programador de AWS. Reacciona a eventos (reglas) o dispara acciones según un horario (*EventBridge Scheduler*).

**Rol en TrendAnalyzer:** dos usos. (1) El horario `ta-wf-raw-to-curated-cada-hora` de **EventBridge Scheduler** (`rate(1 hour)`) arranca el workflow de Glue con el rol `ta-lakehouse-scheduler-role`; se creó deshabilitado y está **encendido** desde el 26-oct-2026 por decisión del equipo (es el estado normal). (2) Reglas sobre los cambios de estado de los jobs de Glue: los fallos van a CloudWatch Logs y alimentan la alarma `ta-glue-job-fallido`. Resolvió la "flecha huérfana" del diagrama FINAL.

### Amazon QuickSight

**Qué es:** herramienta de inteligencia de negocios (BI) de AWS para crear dashboards interactivos.

**Rol en TrendAnalyzer:** es la **capa de visualización** para Marketing, Gerencia y Analistas (Lote 6.1). Suscripción Enterprise en `us-east-1` (prueba gratis hasta ≈ 26-nov-2026; se da de baja después de la sustentación). El tablero `ta-tablero-tendencias` tiene las hojas «Tendencias» (HU-037) y «Relaciones» (HU-038) sobre 4 datasets SPICE con SQL propio sobre `ta_curated`, refrescados cada hora. Las horas se muestran en `America/Bogota` (convertidas en el SQL; el almacenamiento es UTC) y ningún dataset contiene `user_hash`. Se entra con los roles IAM `ta-quicksight-autor` y `ta-quicksight-lector` (grupos `ta-autores` y `ta-lectores`). Lo de adentro de QuickSight se despliega como código con `quicksight.py`.

### SPICE

**Qué es:** el motor en memoria de QuickSight. Importa una copia de los datos para que los dashboards respondan casi instantáneamente.

**Rol en TrendAnalyzer:** es la razón por la que se puede cumplir el KPI de dashboard < 3 s sin depender de la velocidad de cada consulta a Athena.

### Amazon CloudWatch

**Qué es:** servicio de métricas, logs y alarmas de AWS.

**Rol en TrendAnalyzer:** observabilidad del plano AWS (Lote 7.2): logs de los jobs de Glue y de sus cambios de estado, el tablero `ta-lakehouse` (últimas corridas, fallos y bytes de Athena) y la alarma `ta-glue-job-fallido`, que avisa por el topic SNS `ta-observabilidad` (falta confirmar la suscripción del correo). Es el equivalente de Cloud Monitoring en GCP.

### IAM + KMS

**Qué es:** **IAM** gestiona identidades y permisos. **KMS** gestiona llaves de cifrado.

**Rol en TrendAnalyzer:** IAM define quién puede hacer qué, con mínimo privilegio:

| Rol | Puede |
|---|---|
| `ta-gcp-exporter-role` | Solo escribir en `raw/origin=stream/` (lo asume el exportador por OIDC). |
| `ta-historical-loader` | Solo escribir en `raw/origin=historical/`. |
| `ta-glue-role` | Leer todo el lake y escribir solo en `trusted/`, `curated/`, `quality/` y `_glue/temp/`; en Raw no. |
| `ta-quicksight-role` | Consultar con Athena en `ta-analytics` y leer solo `curated/`. |
| `ta-quicksight-autor` / `ta-quicksight-lector` | Entrar al tablero como autor o lector, sin acceso a los datos. |
| `ta-lakehouse-scheduler-role` | Solo arrancar el workflow de Glue. |
| `ta-github-cd` | Solo lectura (`ViewOnlyAccess`) para el `terraform plan` del CD. |

Los roles productores no pueden leer ni borrar. KMS cifra los objetos de S3 con la llave `alias/ta-dev-lake` (SSE-KMS con *bucket key*).

### Loader histórico (`src/historical/`)

**Qué es:** un programa que genera y carga un lote grande de registros de una sola vez (*full load*).

**Rol en TrendAnalyzer:** produce el **dataset histórico** (ver *Histórico* en §3) con `scripts/load_historical.py`: genera los eventos, anonimiza con HMAC, corre el NLP en lote con la imagen de NLP local y sube Parquet por `source/dt/hr` de `event_time` a `s3://…/raw/origin=historical/` asumiendo el rol `ta-historical-loader`. Sus etapas se pueden reanudar. Entra directo a AWS, sin pasar por Pub/Sub; su `process_time` es la hora de la carga.

---

## 7. Procesamiento y NLP

### Apache Beam

**Qué es:** framework de código abierto para definir pipelines de datos en un modelo unificado (batch y streaming). Dataflow es uno de sus *runners*.

**Rol en TrendAnalyzer:** el pipeline de `src/streaming/` está escrito en Beam 2.77. Se prueba en local con el `DirectRunner` (en su variante `BundleBasedDirectRunner`) antes de ejecutarlo en Dataflow, y lo que funciona en local se vuelve a validar en la nube, porque el comportamiento no siempre es el mismo (p. ej. los *timers* que retenían la marca de agua solo fallaban en Dataflow).

### NLP preentrenado (`RunInference`) y RoBERTuito

**Qué es:** *Natural Language Processing* es la rama de la IA que interpreta texto. Un modelo **preentrenado** ya fue entrenado por otros y solo se aplica. `RunInference` es el transform de Beam para ejecutar un modelo dentro del pipeline. **RoBERTuito** es un modelo de lenguaje para español de redes sociales; la librería `pysentimiento` ofrece versiones ya entrenadas para sentimiento y emoción.

**Rol en TrendAnalyzer:** para cada texto calcula `sentiment_label` (`negative/neutral/positive`) y `emotion_label` (`joy, sadness, anger, fear, disgust, surprise, neutral`), con sus puntajes. Usa `robertuito-sentiment-analysis` y `robertuito-emotion-analysis` (ADR-05), **horneados** con su commit en la imagen `dataflow-nlp`, sin depender de Hugging Face en ejecución; el mapeo de etiquetas vive en `src/common/label_map.yaml` y cada fila registra su `nlp_model_version`. Procesa lotes de hasta 16 textos; si un lote falla, sus filas siguen **sin NLP** en lugar de descartarse. Las filas del stream procesadas antes de activarlo (`NLP_SINCE` = 2026-10-22T23:00Z) quedaron sin NLP y se conservan con NLP nulo. Las *model cards* no declaran licencia y los modelos quedan sujetos a las de sus datos de entrenamiento (TASS 2020, EmoEvent), muchas no comerciales: es compatible con el uso académico (`INFORME.md` §8). Precisión medida: ver §9.

### Anonimización HMAC-SHA256

**Qué es:** HMAC es una función de hash con llave secreta: el mismo dato siempre produce el mismo código, pero sin la llave no se puede calcular ni revertir. Con la llave y la lista de identificadores originales sí se puede recalcular el vínculo, por eso técnicamente es una **seudonimización** (ver §9).

**Rol en TrendAnalyzer:** convierte `author.user_id` en `user_hash` (64 caracteres en hex) y descarta `author.handle`. Se hace en el **primer paso de cada plano** (Dataflow o loader), de modo que ningún identificador en claro cruza a AWS (ADR-06). Cumple el KPI de 100 % de identificadores anonimizados antes de la capa analítica y conserva el análisis de relaciones: el mismo usuario tiene siempre el mismo hash en el stream y en el histórico.

### Deduplicación

**Qué es:** eliminar registros repetidos.

**Rol en TrendAnalyzer:** se hace en dos lugares. En **Dataflow**, en dos capas (ADR-17): el `id_label` de Pub/Sub descarta reentregas cercanas, y un estado por `event_id` en ventanas fijas de 1 h de tiempo de proceso cubre las copias más tardías (sin *timers*, que retenían la marca de agua). En **Trusted**, el job `ta-job-raw-to-trusted` deduplica sobre todo el lake y se queda con la copia de menor `process_time`. Es necesaria porque Pub/Sub entrega *al menos una vez*, el simulador puede repetir hasta 50 mensajes tras reiniciarse y el caos inyecta duplicados. Medido: 0,0012 % de duplicados en el Raw del stream y 0 % en Trusted.

### Dead-letter queue (DLQ)

**Qué es:** destino alterno para mensajes que no se pudieron procesar.

**Rol en TrendAnalyzer:** los mensajes malformados no se pierden ni rompen el pipeline: van al topic `ta-dlq` (con el cuerpo tal como llegó y los atributos `dlq_error_type`, `dlq_error_detail`, `dlq_source` y `dlq_received_at`), a `gs://…/dlq/` y, por `ta-dlq-df`, a la tabla `dlq_events`. Como el cuerpo se conserva, se pueden volver a publicar después de corregir el problema. Si llegan más de 30 en 10 minutos, salta la alerta `ta-op-dlq-creciente`.

### Normalización de engagement

**Qué es:** llevar métricas con nombres distintos en cada red a un vocabulario común.

**Rol en TrendAnalyzer:** `likes` = `score` (Reddit) | `reactions_total` (Facebook) | `like_count` (Instagram); `shares` = `shares_count` (Facebook) y `null` en Reddit e Instagram; `comments` = `num_comments` (Reddit) | `comments_count` (Facebook, Instagram).

### Ventanas deslizantes, z-score y `trend_windows`

**Qué es:** en streaming, una **ventana** agrupa eventos por tiempo; una ventana *deslizante* se solapa con la siguiente. El **z-score** mide a cuántas desviaciones estándar está un valor de su media.

**Rol en TrendAnalyzer:** ventanas de 5 minutos que avanzan cada minuto, en **tiempo del evento**, por marca y fuente, con el volumen y el sentimiento promedio. Un DoFn con estado compara cada ventana con el **baseline** de las 60 anteriores (las vacías cuentan como 0) y calcula el z-score de volumen y de sentimiento. `is_alert` exige baseline completo, al menos 5 posts, un z-score de 3 o más en valor absoluto y una ventana completa (nunca las que cierra un drenado). Cada panel de una ventana es una fila de `trend_windows`, y `trend_windows_latest` deja la vigente. Sustenta el KPI de detección en menos de 5 minutos.

### Parámetros operativos de las alertas

**Qué es:** reglas para evitar ruido en las alertas.

**Rol en TrendAnalyzer:**

- **Tolerancia a tardíos** de 30 min: un evento más tardío se escribe pero no recalcula ventanas (contador `late_events`); los paneles tardíos son acumulativos, uno por minuto.
- **Arranque en frío:** sin baseline no hay alertas. Como el baseline vive en el estado del job, **cada job nuevo pasa ~1 h sin alertas** (por eso la demo enciende todo 80 min antes).
- **Tipos y *cooldown*:** cada métrica que pasa el umbral da un tipo (`volume_up`, `volume_down`, `sentiment_up` o `sentiment_down`) y se publica un mensaje `ta-alert` 1.0, con un *cooldown* de 15 min por marca × fuente × tipo en tiempo del evento.

### Watermark

**Qué es:** la palabra se usa en dos sentidos. En Dataflow, la *marca de agua* es la estimación de "hasta qué momento del tiempo del evento ya llegaron todos los datos": cuando pasa el final de una ventana, esa ventana se puede cerrar. En el exportador es simplemente el marcador de lo último que ya se copió.

**Rol en TrendAnalyzer:**

- **En Dataflow** decide cuándo se cierran las ventanas de tendencia y qué evento cuenta como tardío, así se manejan los tardíos que genera el caos controlado (ver *Parámetros operativos de las alertas*). Si se atrasa más de 5 minutos, salta la alerta `ta-op-latencia`.
- **En el exportador** es el archivo `gs://{GCP_PROJECT}-ta-lake/export-state/watermark.json`:

```
{"schema": "ta-export-watermark@1", "cursor_time_created": "2026-10-29T14:15:02.311Z",
 "updated_at": "…", "run_id": "…", "exporter": "gcs-to-s3@1.0", "last_run": {…}}
```

`cursor_time_created` es la fecha de creación en GCS del archivo de `enriched/` más nuevo que ya se copió. En cada corrida, el exportador revisa los archivos creados después de `cursor − 1 h` (la hora de solape cubre los que tardaron en aparecer), copia a S3 los que falten y, solo si todo salió bien, avanza el cursor, que nunca retrocede. Así cada corrida revisa solo lo nuevo y no todo `enriched/`.

### Parquet y esquema plano

**Qué es:** Parquet es un formato de archivo **columnar** y comprimido, muy eficiente para analítica.

**Rol en TrendAnalyzer:** es el formato de todo lo que vive en `enriched/` (GCS) y en S3. Los Parquet usan el **esquema plano** de BigQuery `posts_enriched` (`user_hash`, `followers`, `label_sentiment`, `brand_primary` aplanados y `payload` como JSON en texto), no el JSON anidado del envelope. Reduce costo y acelera Athena.

### `text_clean` (regla `text-clean@1`)

**Qué es:** la versión normalizada del texto de cada comentario, pensada para análisis de texto.

**Rol en TrendAnalyzer:** la calcula el job de Trusted sin tocar `text`: normaliza Unicode, quita emojis y símbolos, URLs, menciones y hashtags (ya están en `hashtags`), pasa a minúsculas conservando tildes, ñ y puntuación, y colapsa los espacios. Garantías: nunca es nula si `text` no lo es, nunca es más larga que `text` (regla DQ-57) y aplicarla dos veces da lo mismo. La versión de la regla va en su nombre para poder cambiarla sin ambigüedad.

---

## 8. DataOps e infraestructura

### Terraform

**Qué es:** herramienta de **Infraestructura como Código (IaC)**: la infraestructura se describe en archivos y se crea o modifica con comandos.

**Rol en TrendAnalyzer:** aprovisiona todo en `infra/gcp/` e `infra/aws/`: 4 raíces (un *bootstrap* y un entorno `dev` por nube) y ~30 módulos, uno por pieza. Cada raíz expone `outputs` (topics, buckets, ARNs) que el código consume por variables de entorno en lugar de hardcodearlos. El estado es **remoto** desde el primer día (ADR-14). Se autentica sin llaves en el código: en AWS con el perfil de AWS CLI y en GCP con Application Default Credentials. Si un corte de red deja un `errored.tfstate`, los scripts lo suben al estado remoto (`terraform state push`) automáticamente.

### Bootstrap de Terraform

**Qué es:** la preparación inicial que debe existir antes del resto de la infraestructura.

**Rol en TrendAnalyzer:** el Lote 1.0 (HU-046) configuró el proyecto GCP existente (región `us-central1`), habilitó 19 APIs, creó los presupuestos y el **estado remoto**, es decir, el lugar donde Terraform anota qué recursos creó y cómo están:

- **GCP:** el bucket `{GCP_PROJECT}-ta-tfstate` (prefijo `env/dev`), con versionado; conserva las 10 versiones anteriores más recientes.
- **AWS:** el bucket `ta-dev-{AWS_ACCOUNT_ID}-tfstate` (clave `env/dev/terraform.tfstate`), con 90 días de versiones, y la tabla DynamoDB `ta-dev-tflock`, el *candado* que impide que dos `apply` escriban el estado a la vez.

Cada *bootstrap* vive en un directorio aparte con estado local, `prevent_destroy` en los recursos de estado y, en AWS, `allowed_account_ids` (ADR-15, ADR-16). Está aplicado desde la v2.12.

### Entornos y regiones

**Qué es:** dónde vive cada cosa.

**Rol en TrendAnalyzer:** GCP en `us-central1`, elegida por las cuotas de vCPU y Dataflow verificadas en la cuenta de crédito (ADR-12; la primera opción fue `us-east1`), y AWS en `us-east-1`. Solo existe el entorno `dev`. Los dos planos procesan en Estados Unidos, lo que tiene implicaciones de transferencia internacional bajo la Ley 1581 (§9).

### GitHub y GitHub Actions

**Qué es:** GitHub aloja el repositorio. GitHub Actions ejecuta flujos automáticos (CI/CD) cuando hay un push o un pull request.

**Rol en TrendAnalyzer:** versiona el código y los contratos (Lote 7.1, ADR-34). La **CI** (`ci.yml`) corre en cada PR y push a `main` **sin credenciales de nube**: pruebas, contratos, identidad, `terraform fmt/validate`, la etapa de prueba de las imágenes y gitleaks sobre todo el historial (6 trabajos, ~3,7 min). El **CD** (`cd.yml`) usa OIDC, solo corre en `main` cuando cambian la infraestructura o los servicios, publica las imágenes con Cloud Build y corre `terraform plan` en las dos nubes; está activo desde la v2.63. El `apply` sigue siendo manual.

### Docker, Artifact Registry y Cloud Build

**Qué es:** Docker empaqueta una aplicación con todo lo que necesita (Python, librerías, modelos) en una *imagen*, un paquete que corre igual en cualquier máquina; un *contenedor* es una imagen en ejecución. **Artifact Registry** es el almacén de imágenes de Google y **Cloud Build** las construye en la nube, sin necesidad de tener Docker en el computador.

**Rol en TrendAnalyzer:** el repositorio **`ta-images`** (Artifact Registry, `us-central1`) guarda las cinco imágenes del proyecto. La dirección completa de cada una es `us-central1-docker.pkg.dev/{GCP_PROJECT}/ta-images/{imagen}:{etiqueta}`.

| Imagen | Qué lleva | Quién la usa | Se construye con | Etiquetas |
|---|---|---|---|---|
| `stream-simulator` | El simulador y el generador | Cloud Run `ta-stream-simulator` | `build_simulator_image.sh` | Commit corto |
| `alert-notifier` | El notificador de alertas | Cloud Run `ta-alert-notifier` | `build_alerts_image.sh` | Huella del código (`h-…`) y `:latest` |
| `gcs-to-s3` | El exportador | Cloud Run Job `ta-gcs-to-s3` | `build_exporter_image.sh` | `{commit}-{huella}` y `:latest` |
| `dataflow-nlp` | Beam y los dos modelos RoBERTuito ya descargados | Los workers de Dataflow | `build_nlp_image.sh` | Commit corto y `:latest` |
| `dataflow-launcher` | El código del pipeline para la plantilla Flex | `dataflow_template.py run` | `dataflow_template.py build` | Commit (o fecha) y `:latest` |

- **Cómo se construye.** El script sube el código a Cloud Build (al bucket `{GCP_PROJECT}_cloudbuild`), que corre las pruebas dentro de la imagen (en las de Dataflow, una prueba de humo) y solo la publica en `ta-images` si pasan. Con el modo `docker` usa el Docker local. Así nunca se publica una imagen con pruebas rotas.
- **Etiquetas.** El commit o la huella dicen exactamente de qué versión del código salió cada imagen (la huella cambia si cambia el código aunque no haya commit); `:latest` apunta a la más reciente. Cloud Run despliega la etiqueta concreta que dice `terraform.tfvars` (`simulator_image`, `exporter_image`, `alerts_notifier_image`), así que un build nuevo no cambia lo desplegado hasta el siguiente `apply`. Dataflow, en cambio, usa `dataflow-nlp:latest` salvo que `TA_NLP_IMAGE` diga otra.
- **Limpieza automática.** El repositorio conserva las 10 versiones más recientes de cada imagen y borra las que no tienen etiqueta después de 7 días, para no pagar el almacenamiento de imágenes viejas.
- **Permisos.** `sa-dataflow` solo puede descargar (`artifactregistry.reader`); `sa-github-cd` puede publicar (`artifactregistry.writer`).
- **CD.** En cada push a `main` que toque los servicios, `cd.yml` reconstruye y publica `stream-simulator`, `gcs-to-s3` y `alert-notifier` con estos mismos scripts. Las dos de Dataflow se construyen a mano.

### Great Expectations

**Qué es:** librería para definir y ejecutar pruebas de calidad sobre los datos ("este campo no puede ser nulo", "este id debe ser único").

**Rol en TrendAnalyzer:** la versión **1.24.1** mide la calidad de Raw, Trusted y Curated (Lote 5.2, ADR-28): lleva a GX las 63 reglas del profiling (DQ-01…DQ-63) más 14 de Curated (DQ-64…DQ-77), en total **239 comprobaciones** por corrida. Corre **a demanda** desde WSL (`run_dq.py`, con su venv `~/.venvs/ta-gx`) sobre una copia local del lake y escribe los resultados en `ta_stream.dq_results` y en `s3://…/quality/{día}/ge/{run_id}/`, sin datos de filas. También mide la precisión del NLP (HU-033).

### dbt (descartado)

**Qué es:** herramienta para transformar datos con SQL de manera versionada, con tests y documentación automática.

**Rol en TrendAnalyzer:** el plan original lo usaba para Curated (`dbt-athena`), sus pruebas y el linaje (*dbt docs*). **Se descartó** por decisión del equipo (26-oct-2026): Curated se construye con Glue Python (ADR-26), la calidad se prueba con Great Expectations (ADR-28) y el linaje es un documento hecho a mano (ADR-29). Las menciones que quedan en el README son anteriores a esa decisión.

### Prueba de humo end-to-end (`smoke_e2e.py`)

**Qué es:** una prueba rápida que sigue un solo evento de punta a punta.

**Rol en TrendAnalyzer:** `scripts/smoke_e2e.py` (Lote 5.4, HU-048, ADR-30) publica su propio centinela (un comentario normal con `event_id` conocido) y lo sigue por BigQuery, `enriched/`, S3, Trusted y Curated, disparando el exportador y el workflow de Glue sin esperar sus horarios. Mide la latencia de cada etapa contra su objetivo y comprueba que no aparezcan `user_id` ni `handle` en ninguna. Última corrida: **430 s** de Pub/Sub a `fact_posts` (15,9 s hasta BigQuery). Es la fuente de dos KPI del informe.

### Prefijo `ta-` y etiquetas

**Qué es:** convención de nombres.

**Rol en TrendAnalyzer:** todo recurso lleva el prefijo `ta-` y las etiquetas `project=trend-analyzer`, `env=dev`, `owner=alejandro` y `component=<pieza>` (p. ej. `ingest`, `lakehouse`, `serving`), lo que permite agrupar costos y responsables. En AWS, `project` ya está activada como etiqueta de asignación de costos; `component` está pendiente de activar en Billing.

### `terraform.tfvars`, `backend.hcl` y secretos

**Qué es:** los archivos de valores y de backend de Terraform, que pueden contener datos sensibles.

**Rol en TrendAnalyzer:** están en `.gitignore`; se versionan sus `*.example`. Nunca hay secretos en el repositorio: gitleaks lo vigila en el pre-commit y en la CI. Los IDs de cuenta, los correos de alerta y los proyectos van en configuración, no en el código. Excepción conocida: un token de Mastodon antiguo quedó en el historial público y está pendiente de revocar.

### `on.py`, `off.py`, `status.py` y estado de ahorro

**Qué es:** los scripts de operación que encienden, apagan y muestran lo que cobra; el *estado de ahorro* es el estado por defecto, con todo lo costoso apagado.

**Rol en TrendAnalyzer:** `on.py` enciende el simulador, Dataflow con NLP, el exportador y el horario del lakehouse (con `--stream-only`, solo los dos primeros); `off.py` drena Dataflow, apaga el simulador y pausa el exportador; `status.py` dice en solo lectura qué está encendido y si el flujo de punta a punta está completo o cortado. Por defecto el simulador está apagado y el Scheduler del exportador en pausa; el horario de Glue cada hora es la excepción aceptada. El *preflight* de cada validación parte de ahí.

### Control de costos

**Qué es:** el conjunto de medidas para no gastar más de lo previsto.

**Rol en TrendAnalyzer:** presupuestos de 400000 COP en GCP y 50 USD en AWS, que solo avisan por correo (al 50, 80/90 y 100 % y por pronóstico); **alertas de gasto por olvido** (Dataflow > 4 h, simulador > 8 h; ADR-35); el estado de ahorro con los scripts de operación; Dataflow de un solo worker; retenciones y ciclos de vida; y la etiqueta `component`. Referencias: Dataflow ~0,25 USD/h y el workflow de Glue ≈ 14,1 USD/mes con el horario.

### gitleaks y pre-commit

**Qué es:** gitleaks busca secretos (llaves, tokens) en el código y en el historial de git; *pre-commit* corre controles automáticos antes de cada commit.

**Rol en TrendAnalyzer:** `.pre-commit-config.yaml` corre gitleaks (v8.28.0) y `terraform fmt` en local, y la CI escanea todo el historial en cada PR. `.gitleaksignore` deja pasar dos hallazgos revisados a mano: una llave falsa de una prueba y el token de Mastodon del historial, que sigue pendiente de revocar.

---

## 9. Calidad y gobernanza

### Profiling

**Qué es:** análisis exploratorio de un conjunto de datos para conocer su estructura, nulos, distribuciones y anomalías.

**Rol en TrendAnalyzer:** `scripts/profile_data.py` (Lote 5.1, HU-031, ADR-24) perfila Raw y Trusted desde una copia local de solo lectura de S3, para conocer el estado real de los datos antes de fijar las reglas de calidad. En Raw (221.637 filas) encontró las filas sin NLP, 6 centinelas, 2 `event_id` repetidos, muchos archivos pequeños y un vocabulario acotado; en Trusted, 54 de 54 reglas cumplidas. De ahí salió el catálogo DQ-01…DQ-63. La evidencia solo lleva agregados (`evidencia/profiling_*_5_1.{json,md}`).

**Términos del informe:** *TTR* (tipos / tokens), *MATTR* (TTR medio en ventanas deslizantes de 500 tokens; no depende del tamaño del texto; 0,479 en Raw), *hapax* (palabras que aparecen una sola vez), *distinct-2* (bigramas distintos / bigramas) y *esqueleto* (el texto en minúsculas, sin tildes, hashtags ni signos y con la marca enmascarada, para medir cuánto se repiten las plantillas). Definiciones exactas en `ARCHITECTURE.md` §3.20.

### Calidad de datos y KPIs

**Qué es:** medición de si los datos son completos, únicos, consistentes y oportunos.

**Rol en TrendAnalyzer:** cada KPI tiene una regla medible y una evidencia. En Trusted, la capa confiable: completitud 100 % (objetivo ≥ 95 %), unicidad 100 % (≥ 99 %), duplicados 0 % (< 1 %) y 100 % de identificadores anonimizados. Además: ingesta a BigQuery en 15,9 s (< 60 s), detección de tendencias en 87–160 s (< 5 min) y respuesta del tablero de 2,83 s en el peor clic (< 3 s). La tabla completa, con su evidencia y el comando para volver a medir cada cifra, está en `INFORME.md` §6. El KPI de reducción del 50 % del tiempo de elaboración de informes de marketing no se midió.

### Catálogo de reglas DQ y `dq_results`

**Qué es:** la lista numerada de reglas de calidad del proyecto, cada una con su umbral, su gravedad y el KPI al que aporta.

**Rol en TrendAnalyzer:** el profiling del Lote 5.1 propuso **DQ-01…DQ-63** sobre Raw y Trusted (`ARCHITECTURE.md` §5.10), cada una con su expectativa de Great Expectations sugerida y su predicado SQL; el 5.2 sumó **DQ-64…DQ-77** para Curated. Ejemplos: no nulos obligatorios, unicidad de `event_id`, `user_hash` con forma de HMAC, hashtags canónicos, `text_clean` nunca más largo que `text` y precisión del NLP ≥ 0,88 (DQ-62). Cada corrida escribe una fila por comprobación en `ta_stream.dq_results` (`layer` ∈ `stream`, `raw`, `trusted`, `curated`).

### Precisión del NLP (accuracy y F1 macro)

**Qué es:** *accuracy* es la proporción de predicciones correctas; *F1 macro* es el promedio, entre clases, de la media armónica entre precisión y exhaustividad, y no se deja inflar por la clase más frecuente.

**Rol en TrendAnalyzer:** compara el `sentiment_label` del modelo con la verdad del simulador (`label_sentiment`), sin las filas sin etiqueta: accuracy **0,9197** y F1 macro **0,9167** sobre 109.365 filas (HU-033). Es un KPI **informativo** y optimista, porque las plantillas expresan el sentimiento con claridad y la verdad no es una anotación humana.

### Catálogo y linaje

**Qué es:** el *catálogo* describe qué datos existen y qué significan. El *linaje* muestra de dónde vienen y cómo se transforman.

**Rol en TrendAnalyzer:** el **catálogo** vive en el Glue Data Catalog: la descripción de cada tabla y el comentario de cada columna salen de las DDL de `contracts/athena/`, Terraform los lleva a Glue y el diccionario generado es `docs/CATALOG.md` (Lote 5.3, HU-034). El **linaje** está en `docs/LINEAGE.md`: un documento y un diagrama Mermaid hechos a mano, de extremo a extremo y por columna, comprobado contra el catálogo (HU-035). No hay *dbt docs* ni OpenLineage (ADR-29).

### Clases de privacidad

**Qué es:** una etiqueta que dice qué tan sensible es una columna desde el punto de vista de los datos personales.

**Rol en TrendAnalyzer:** cada comentario de columna del catálogo empieza con una de cuatro clases: `[anonimizado]` (solo las 5 columnas `user_hash`), `[cuasi_identificador]` (8: `followers` y `payload` en Raw y Trusted, y los resúmenes de actividad de `dim_user`, que cruzados con otras fuentes ayudarían a reidentificar), `[texto_libre]` (3) y `[sin_dato_personal]` (92). Una prueba exige que `[anonimizado]` esté exactamente en `user_hash`. Curated no lleva texto libre, y sus únicos cuasi-identificadores son los resúmenes por seudónimo de `dim_user`.

### Seudonimización

**Qué es:** reemplazar un identificador por un código que, por sí solo, no revela a la persona, pero que con información guardada aparte (como una llave) permite volver a vincularlo. A diferencia de la anonimización, en principio es reversible.

**Rol en TrendAnalyzer:** es lo que hace el HMAC de `user_hash`, y así lo nombra la nota de la Ley 1581 en `GOVERNANCE.md`. Por eso la llave está restringida a `sa-dataflow` y no se rota (rotarla rompería la unión entre stream e histórico).

### DAMA-DMBOK

**Qué es:** guía de referencia internacional de buenas prácticas en gestión de datos.

**Rol en TrendAnalyzer:** `GOVERNANCE.md` recorre sus 11 áreas de conocimiento (gobierno, arquitectura, modelado, almacenamiento y operaciones, seguridad, integración, documentos y contenido, datos maestros y de referencia, *data warehousing* y BI, metadatos y calidad), con la evidencia de cada una (rutas que el validador comprueba) y sus brechas. También define los roles: dueño (el equipo), administrador de calidad y gobernanza (Alejandro), custodios técnicos y consumidores.

### Ley 1581 (Colombia)

**Qué es:** ley colombiana de protección de datos personales (Habeas Data).

**Rol en TrendAnalyzer:** aunque los datos son sintéticos, se documenta el tratamiento como si fueran reales (nota en `GOVERNANCE.md`): seudonimización con HMAC en el primer paso; acceso restringido al `raw/` de GCS (el único lugar con el *handle* simulado, 30 días); minimización (solo `enriched/` sale de GCP y Curated no lleva texto ni identificadores); retención limitada; transferencia internacional (los datos se procesan en Estados Unidos; por confirmar contra la norma vigente si fueran reales); y qué haría falta para los derechos del titular. Los controles que cita la nota se verificaron en la nube en solo lectura (G1 a G7).

### Retención

**Qué es:** cuánto tiempo se conserva cada dato.

**Rol en TrendAnalyzer:** Pub/Sub 7 días; GCS `raw/` y `dlq/` 30 días; BigQuery `posts_enriched` y `trend_windows` 90 días (el histórico completo vive en S3), `dlq_events` 30 días y `dq_results` sin expiración; S3 `athena-results/` 7 días; logs de Glue 30 días. Del estado de Terraform se conservan las 10 últimas versiones anteriores en GCS y 90 días de versiones en S3.

---

## 10. Conceptos transversales

### Multicloud

**Qué es:** usar más de un proveedor de nube en una misma solución.

**Rol en TrendAnalyzer:** GCP ofrece Pub/Sub, Dataflow y BigQuery para el streaming; AWS ofrece S3, Glue, Athena y QuickSight para el lakehouse. El **único camino de datos** entre las dos es `gcs-to-s3`, que solo lleva datos anonimizados y se autentica por OIDC; las dos nubes comparten el contrato del envelope. El costo de la decisión: dos modelos de identidad, dos raíces de Terraform y el puente que operar.

### Streaming vs. batch

**Qué es:** *streaming* procesa datos continuamente, evento por evento; *batch* los procesa en bloques periódicos.

**Rol en TrendAnalyzer:** el plano GCP es streaming (segundos). El plano AWS es batch (cada hora). El histórico es un *full load* batch único.

### Near-real-time

**Qué es:** procesamiento con latencia de segundos a pocos minutos, sin ser instantáneo.

**Rol en TrendAnalyzer:** es el objetivo del plano GCP: el KPI exige detectar tendencias en menos de 5 minutos para el 95 % de los eventos válidos. Se midió con dos alertas reales: 159,8 s y 106,9 s del fin de la ventana a la notificación (87 s en el ensayo). El plano AWS no es *near-real-time*: con los horarios normales el dato llega a Curated en ≤ 1 h 30 min y al tablero en ≤ 2 h 30 min.

### Lakehouse

**Qué es:** arquitectura que combina el almacenamiento barato y flexible de un *data lake* con la estructura y consultas SQL de un *data warehouse*.

**Rol en TrendAnalyzer:** es lo que implementa AWS: archivos Parquet en S3 + catálogo en Glue + consultas SQL en Athena.

### Arquitectura por capas (Raw → Trusted → Curated)

**Qué es:** organizar el lago en niveles de refinamiento creciente (similar a la "arquitectura medallón").

**Rol en TrendAnalyzer:**

- **Raw:** tal cual llega, ya anonimizado. En S3, el del stream es copia byte a byte de `enriched/`: el `raw/` de GCS, que tiene el *handle*, nunca sale de GCP.
- **Trusted:** limpio, deduplicado y validado; une stream e histórico, descarta centinelas, particiona por `dt = DATE(event_time)` y añade `text_clean`. Lo escribe `ta-job-raw-to-trusted`.
- **Curated:** modelado para análisis (estrella, agregados y aristas), sin texto libre ni identificadores. Lo escribe `ta-job-trusted-to-curated`.

### Modelo estrella (`fact_posts`, `dim_*`)

**Qué es:** diseño analítico con una tabla central de hechos rodeada de tablas de dimensiones.

**Rol en TrendAnalyzer:** `fact_posts` (grano `event_id`, particionada por `dt`) guarda un registro por post con sus métricas; `dim_date`, `dim_source`, `dim_brand` y `dim_user` describen el contexto (`dim_user` solo con `user_hash`, primera y última aparición, número de posts y máximo de seguidores). Las llaves foráneas son **naturales** (`source`, `brand_primary`, `user_hash` y `date_key` como `yyyymmdd`), sin llaves sustitutas, y todas las tablas se regeneran enteras en cada corrida, así que no guardan historia de cambios. Se complementa con agregados por `brand_primary` (`agg_emotion_by_brand_hour`, `agg_trend_daily`).

### Tablas de aristas (`edges_*`)

**Qué es:** representación de relaciones como pares origen-destino con un peso (la base de un grafo).

**Rol en TrendAnalyzer:** cubren el "análisis de relaciones" sin base de grafos (ADR-09, ADR-27). `edges_user_brand`: una fila por usuario y marca, con `weight` = filas del usuario que mencionan la marca (una comparativa cuenta para sus dos marcas); hoy 15.395 pares. `edges_hashtag_cooccurrence`: una fila por par de hashtags, con `weight` = filas donde aparecen juntos y siempre `hashtag_a < hashtag_b`; hoy 136 pares de 25 hashtags. Las escribe el mismo job de Curated y alimentan la hoja «Relaciones» del tablero.

### Idempotencia

**Qué es:** propiedad de una operación que, aplicada varias veces, da el mismo resultado que aplicada una.

**Rol en TrendAnalyzer:** permite reintentar cualquier etapa sin duplicar: el `event_id` es determinista; Dataflow escribe archivos con nombres deterministas por ventana; el exportador omite objetos ya copiados y avanza su watermark solo al terminar; reejecutar el histórico no sube nada; Trusted reemplaza un archivo por partición (mismo Raw, mismos bytes) y Curated sube solo los archivos cuyo sha256 cambió.

### Tiempo del evento y tiempo de proceso

**Qué es:** el *tiempo del evento* es cuándo ocurrió algo (`event_time`); el *tiempo de proceso* es cuándo el sistema lo procesó (`process_time`).

**Rol en TrendAnalyzer:** las ventanas de tendencia y el *cooldown* usan el tiempo del evento, para que un evento tardío caiga en su ventana correcta. Las particiones cambian según la capa: en GCS y en el Raw del stream, `dt`/`hr` son la hora de proceso; en el histórico, Trusted y Curated, `dt` es la fecha del evento en UTC (`ARCHITECTURE.md` §9 #9). En el histórico, `process_time` es la hora de la carga, así que la latencia solo tiene sentido en el stream.

### Entrega «al menos una vez» (*at-least-once*)

**Qué es:** garantía de mensajería en la que ningún mensaje se pierde, pero alguno puede llegar más de una vez.

**Rol en TrendAnalyzer:** es la garantía de Pub/Sub y del simulador (que tras un reinicio repite hasta 50 mensajes). Se compensa con identidad determinista y deduplicación por `event_id`, en lugar de intentar una entrega exactamente una vez.

### Contrato

**Qué es:** el acuerdo explícito sobre la forma en que dos componentes se hablan (formato, nombres, campos obligatorios).

**Rol en TrendAnalyzer:** es la idea que organiza el proyecto: cada interfaz está escrita en `ARCHITECTURE.md` §5, tiene su copia ejecutable en `project/contracts/` y se valida en CI. Quien construye un componente solo necesita el contrato del vecino, no su código.

### JSON Schema

**Qué es:** un estándar para describir y validar la forma de un documento JSON (campos, tipos, valores permitidos).

**Rol en TrendAnalyzer:** los `*.schema.json` de `contracts/` (borrador 2020-12) definen qué es un evento válido; por ejemplo, el schema enriquecido prohíbe `handle` y `user_id`, con lo que la anonimización queda verificable.

---

## 11. Fuera del alcance (mencionados pero no usados en v2)

| Componente | Qué es | Por qué quedó fuera |
|---|---|---|
| **Mastodon / RSS** | Red social descentralizada y formato de titulares | Se reemplazaron por Facebook e Instagram simulados (v2.2). |
| **Datasets de emociones** (TEC, GoEmotions, SemEval-2007, DailyDialog, ISEAR) | Colecciones públicas de textos etiquetados con emoción | Estaban en inglés, no hablaban de las marcas y obligaban a inyectarlas; los reemplazó el generador (v2.2). |
| **Modelo en inglés** (`emotion-english-distilroberta-base`) | Modelo de emoción para inglés | Sustituido por RoBERTuito en español (ADR-05). |
| **CocaCola y Colombiana** | Marcas que trae el script original del generador | Fuera de v1; se pueden añadir en `brands.yaml`. |
| **Firestore** | Base de datos documental de GCP | El estado del simulador vive en un objeto de GCS. |
| **Cloud SQL** | Base relacional administrada | No hay datos transaccionales en v1. |
| **Amazon Neptune** | Base de datos de grafos de AWS | Se cubre con tablas de aristas consultables en Athena; queda como extensión (ADR-09). |
| **Dataplex / OpenLineage** | Gobernanza de GCP / estándar abierto de linaje | Catálogo en Glue Data Catalog y linaje hecho a mano en `LINEAGE.md` (ADR-29). |
| **dbt** (`dbt-athena`, *dbt docs*) | Transformación SQL con pruebas y documentación | Descartado por el equipo: Curated con Glue Python, calidad con Great Expectations y linaje a mano (ADR-26, ADR-28, ADR-29). |
| **Spark en Glue** (`glueetl`) | Jobs de Glue sobre Spark | Mínimo 2 DPU por corrida, más caro para ~221.000 filas; se usa Python shell con pyarrow (ADR-25). |
| **Glue Crawlers** | Descubren particiones y esquemas | Se usa partition projection, con tablas creadas por Terraform desde las DDL (ADR-08, ADR-23). |
| **Storage Transfer Service / llaves de IAM** | Alternativas para copiar de GCS a S3 | El puente es un Cloud Run Job con OIDC, sin llaves de larga vida (ADR-07, ADR-22). |
| **Conector QuickSight → BigQuery** | Leer BigQuery desde QuickSight | Sin verificar; el camino principal es Athena → QuickSight (`ARCHITECTURE.md` §9 #6). |
| **`src/collectors/`** | Conectores REST reales de Mastodon, Reddit y RSS | Código del enfoque anterior; ya no está en el repositorio. |

---

## 12. Tabla resumen: componente → rol

| Componente | Nube / ámbito | Rol en una frase |
|---|---|---|
| `ARCHITECTURE.md` | Repositorio | Fuente de verdad: contratos, decisiones y una sección por lote. |
| `PLAN.md` | Repositorio | Fases, lotes, trazabilidad hacia las HU y la única tabla de estado. |
| `CHANGELOG.md` | Repositorio | Historial de versiones y decisiones fechadas. |
| `HU.md` | Repositorio | Historias de usuario y criterios de aceptación (qué es "terminado"). |
| `INFORME.md` · `SUSTENTACION.md` · `RUNBOOK.md` | Repositorio | Informe con los KPI medidos, guion de la sustentación y manual de operación. |
| `GOVERNANCE.md` · `CATALOG.md` · `LINEAGE.md` | Repositorio | Gobierno de datos, diccionario de datos y linaje. |
| `GLOSSARY.md` | Repositorio | Vocabulario común del proyecto. |
| `contracts/` | Repositorio | Contratos ejecutables: schemas, DDL, ejemplos y pruebas. |
| `docs/validation/` + `validate_*.py` + `evidencia/` | Repositorio | Validación reproducible de cada lote y su evidencia. |
| Lote / Fase | Plan | Unidad de entrega / agrupación hacia Sprint 2 y Sprint Final. |
| ADR | Plan | Decisión de arquitectura con su alternativa descartada. |
| Generador de comentarios | GCP (en el simulador) | Produce el texto y la marca de cada comentario en español. |
| `ta-stream-simulator` | GCP (Cloud Run) | Reproduce Reddit, Facebook e Instagram como si estuvieran en vivo y fuerza olas de demo. |
| Cloud Storage | GCP | Estado del simulador y lake de GCP (`raw/`, `enriched/`, `dlq/`). |
| Cloud Run / Run Job | GCP | Simulador y notificador de alertas (servicios) y exportador (job). |
| Cloud Scheduler | GCP | Dispara el exportador cada 15 minutos (en pausa por defecto). |
| Pub/Sub | GCP | Mensajería del stream, de la DLQ y de las alertas. |
| Dataflow | GCP | Valida, deduplica, anonimiza, aplica NLP, calcula tendencias y publica alertas. |
| BigQuery | GCP | Analítica caliente: posts enriquecidos, ventanas, DLQ y resultados de calidad. |
| Secret Manager | GCP | Custodia la llave HMAC de anonimización. |
| Monitoring + Logging | GCP | Tablero de operación y alertas de tendencia, operación y gasto por correo. |
| `gcs-to-s3` + OIDC | GCP → AWS | Único puente de datos entre nubes; copia solo datos anonimizados. |
| Loader histórico | → AWS | Carga única de 180 días, generada con el mismo `EventFactory`. |
| Amazon S3 | AWS | Lakehouse por capas Raw → Trusted → Curated, más los reportes de calidad. |
| AWS Glue (Python shell) | AWS | Construye Trusted y Curated con pyarrow, cada hora. |
| Glue Data Catalog | AWS | Catálogo oficial, con descripciones y clases de privacidad. |
| Athena (+ partition projection) | AWS | Consultas SQL sobre S3 sin crawlers, con límite de 1 GiB. |
| EventBridge (Scheduler y reglas) | AWS | Arranca el workflow cada hora y registra los estados de Glue. |
| QuickSight (+ SPICE) | AWS | Tablero de tendencias y relaciones para Marketing, Gerencia y Analistas. |
| CloudWatch + SNS | AWS | Observabilidad del lakehouse y alarma de fallos de Glue. |
| IAM + KMS | AWS | Permisos mínimos por rol y cifrado SSE-KMS. |
| Terraform | DataOps | Toda la infraestructura como código, con estado remoto. |
| GitHub Actions | DataOps | CI sin credenciales y CD con OIDC. |
| Docker + Artifact Registry + Cloud Build | DataOps | Construyen, prueban y distribuyen las imágenes. |
| `on.py` / `off.py` / `status.py` | DataOps | Encienden, apagan y muestran lo que cobra. |
| Great Expectations | Calidad | 239 comprobaciones sobre Raw, Trusted y Curated, a demanda. |
| `smoke_e2e.py` | Calidad | Sigue un centinela de Pub/Sub a Curated y mide cada etapa. |
| Modelos RoBERTuito (pysentimiento) | GCP (Dataflow) | Clasifican el sentimiento y la emoción de cada texto en español. |
| HMAC-SHA256 | GCP / loader | Seudonimiza a los usuarios antes de que el dato salga de GCP. |
