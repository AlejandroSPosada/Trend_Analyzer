# Prueba de humo E2E · smoke-20261029T212744Z

- Resultado: **OK**
- Centinela: `2cfd26d6-093d-57e7-973e-6eb806f804ba` (reddit, `comment-gen@1.0`), publicado 2026-10-29T21:28:02.243Z
- Total publicación → `fact_posts`: 430.1 s
- Identidad: sin `user_id`/`handle` en bigquery, curated, gcs_enriched, s3_raw, trusted; mismo `user_hash` en 5 etapas: sí

- Calentamiento (`ta-verify@2.1`, Trusted lo descarta): 14.0 s hasta BigQuery

| Etapa | Desde | Latencia (s) | Objetivo (s) | Desde la publicación (s) | OK |
|---|---|---|---|---|---|
| bigquery | published | 15.9 | 60 | 15.9 | ✅ |
| gcs_enriched | published | 136.3 | 900 | 136.3 | ✅ |
| s3_raw | gcs_enriched | 58.5 | 300 | 194.8 | ✅ |
| trusted | s3_raw | 154.8 | 900 | 349.5 | ✅ |
| curated | trusted | 80.6 | 600 | 430.1 | ✅ |
