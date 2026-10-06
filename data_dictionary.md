# Diccionario de Datos — Cloud Provider Analytics

> **Propósito**  
> Documentar de forma concisa y operativa las fuentes provistas para el proyecto **Cloud Provider Analytics**, con foco en la primera entrega: comprensión del dato, grano, relaciones, tipos, calidad, trazabilidad y riesgos iniciales.  
> Este documento describe el estado observado de las fuentes y propone criterios de tipificación y calidad para las capas posteriores del Data Lake, sin adelantar decisiones de implementación innecesarias para esta etapa.

---

## 1. Convenciones generales

### 1.1 Capas de referencia

| Capa | Uso esperado en el proyecto |
|---|---|
| **Landing** | Archivos originales, sin modificación. |
| **Bronze** | Mismo grano que la fuente, esquema explícito, tipos controlados y columnas técnicas de ingesta. |
| **Silver** | Datos normalizados y conformados, con tratamiento de nulos, outliers, integridad y compatibilidad de versiones. |
| **Gold** | Marts orientados a FinOps, Soporte y Producto / Usage. |

### 1.2 Criterio para reglas de calidad

| Severidad | Criterio sugerido |
|---|---|
| **Error** | El registro no puede interpretarse de forma confiable o rompe una regla estructural / referencial crítica. Puede requerir quarantine. |
| **Warning** | El dato es utilizable, pero presenta una inconsistencia o anomalía que debe quedar marcada. |
| **Info** | Particularidad conocida del dataset que no invalida el registro. |

### 1.3 Tipos

- **Tipo origen:** representación observada en los archivos de Landing.
- **Tipo objetivo:** tipo sugerido para el procesamiento en Bronze / Silver.
- Los tipos objetivo son propuestas iniciales y podrán ajustarse durante la implementación.

---

## 2. Inventario de fuentes

| Fuente | Grano | Identificador / clave | Relación principal | Volumen observado | Ingesta | Dominio principal |
|---|---|---|---|---:|---|---|
| `customers_orgs.csv` | 1 organización cliente | `org_id` | Maestro principal | 80 filas | Batch | Cliente / dimensión |
| `users.csv` | 1 usuario de una organización | `user_id` | `org_id -> customers_orgs` | 800 filas | Batch | Cliente |
| `resources.csv` | 1 recurso cloud | `resource_id` | `org_id -> customers_orgs` | 400 filas | Batch | Producto / Usage |
| `billing_monthly.csv` | 1 factura por organización y mes | `invoice_id`; también `org_id + month` es único | `org_id -> customers_orgs` | 240 filas | Batch mensual | FinOps |
| `support_tickets.csv` | 1 ticket de soporte | `ticket_id` | `org_id -> customers_orgs` | 1.000 filas | Batch | Soporte |
| `marketing_touches.csv` | 1 contacto de marketing | `touch_id` | `org_id -> customers_orgs` | 1.500 filas | Batch | Marketing |
| `nps_surveys.csv` | 1 encuesta NPS por organización y fecha | `org_id + survey_date` | `org_id -> customers_orgs` | 92 filas | Batch | Customer Experience |
| `usage_events_stream/*.jsonl` | 1 evento de uso de un recurso | `event_id` | `org_id -> customers_orgs`; `resource_id -> resources` | 43.200 eventos | Streaming / micro-batch | Producto / FinOps |

---

## 3. Relaciones principales

| Origen | Campo | Destino | Cardinalidad esperada | Observación |
|---|---|---|---|---|
| `users` | `org_id` | `customers_orgs.org_id` | N:1 | No se detectaron huérfanos. |
| `resources` | `org_id` | `customers_orgs.org_id` | N:1 | No se detectaron huérfanos. |
| `billing_monthly` | `org_id` | `customers_orgs.org_id` | N:1 | Todos los clientes tienen 3 facturas. |
| `support_tickets` | `org_id` | `customers_orgs.org_id` | N:1 | No se detectaron huérfanos. |
| `marketing_touches` | `org_id` | `customers_orgs.org_id` | N:1 | No se detectaron huérfanos. |
| `nps_surveys` | `org_id` | `customers_orgs.org_id` | N:1 | 60 de 80 clientes tienen al menos una encuesta. |
| `usage_events_stream` | `org_id` | `customers_orgs.org_id` | N:1 | Integridad completa observada. |
| `usage_events_stream` | `resource_id` | `resources.resource_id` | N:1 | Integridad completa observada. |

---

# 4. Diccionario por fuente

## 4.1 `customers_orgs.csv`

### Ficha de la fuente

| Propiedad | Valor |
|---|---|
| **Descripción** | Maestro de organizaciones cliente del proveedor cloud. |
| **Grano** | Una fila por organización. |
| **Clave primaria** | `org_id` |
| **Volumen observado** | 80 registros, 11 columnas. |
| **Duplicados de PK** | No detectados. |
| **Uso principal** | Dimensión base para FinOps, Soporte, Producto y otros análisis por organización. |
| **Modo de ingesta** | Batch |

### Campos

| Campo | Tipo origen | Tipo objetivo | Nullable | Descripción | Valores / dominio observado | Regla de calidad | Problema detectado | Tratamiento propuesto |
|---|---|---|---|---|---|---|---|---|
| `org_id` | string | string | No | Identificador único de la organización. | Formato `org_` + 8 caracteres. | No nulo, único y con formato válido. | Ninguno detectado. | Mantener como clave natural. |
| `org_name` | string | string | No | Nombre de la organización. | Texto. | No vacío. | El nombre contiene numeración artificial y no debe usarse como identificador. | Mantener como atributo descriptivo. |
| `industry` | string | string | No | Industria / rubro de la organización. | Education, Media, Manufacturing, E-commerce, Retail, Healthcare, Energy, Fintech, Gaming, Government. | Debe pertenecer al catálogo permitido. | Ninguno detectado. | Normalizar casing si fuera necesario. |
| `hq_region` | string | string | No | Región de la casa central del cliente. | `sa-east`, `us-east`, `us-west`, `eu-central`, `eu-west`, `ap-south`, `ap-northeast`. | Debe pertenecer al catálogo de regiones. | No se observaron variantes en esta fuente. | Validar contra catálogo común. |
| `plan_tier` | string | string | No | Plan contratado por el cliente. | free, standard, pro, enterprise. | Debe pertenecer al dominio definido. | Relación semántica ambigua con `is_enterprise`. | Mantener ambos campos hasta confirmar significado. |
| `is_enterprise` | boolean | boolean | No | Indicador de cliente enterprise. | `true`, `false`. | Debe ser booleano. | No coincide sistemáticamente con `plan_tier = enterprise`. | No derivar un campo a partir del otro. |
| `signup_date` | string | date | No | Fecha de alta del cliente. | Formato `YYYY-MM-DD`. | Fecha válida. | Llega como texto. | Castear a `date`. |
| `sales_rep` | string | string | No | Representante comercial asignado. | `rep_a` a `rep_e`. | Debe pertenecer al dominio observado. | Ninguno detectado. | Mantener. |
| `lifecycle_stage` | string | string | No | Etapa del ciclo de vida del cliente. | lead, prospect, active, at_risk, churned. | Debe pertenecer al dominio definido. | Ninguno detectado. | Mantener. |
| `marketing_source` | string | string | No | Fuente de adquisición del cliente. | partner, event, organic, ads, referral. | Debe pertenecer al dominio definido. | Ninguno detectado. | Mantener. |
| `nps_score` | numeric | integer | Sí | Puntaje NPS asociado a la organización. | Escala esperada: -100 a 100. | `-100 <= nps_score <= 100` o nulo. | 11 nulos y 1 valor fuera de rango (`101`). | Nulos permitidos; fuera de rango como dato inválido / flag. |

### Riesgos / observaciones

- El significado de `is_enterprise` no está suficientemente documentado respecto de `plan_tier`.
- `nps_score` de esta fuente no coincide de forma evidente con las encuestas históricas de `nps_surveys`; se recomienda tratar `nps_surveys` como fuente temporal de encuestas y no asumir equivalencia directa.

---

## 4.2 `nps_surveys.csv`

### Ficha de la fuente

| Propiedad | Valor |
|---|---|
| **Descripción** | Encuestas NPS respondidas por organizaciones cliente. |
| **Grano** | Una encuesta por organización y fecha. |
| **Clave natural** | `org_id + survey_date` |
| **Volumen observado** | 92 registros, 4 columnas. |
| **Relación principal** | `org_id -> customers_orgs.org_id` |
| **Modo de ingesta** | Batch |

### Campos

| Campo | Tipo origen | Tipo objetivo | Nullable | Descripción | Valores / dominio observado | Regla de calidad | Problema detectado | Tratamiento propuesto |
|---|---|---|---|---|---|---|---|---|
| `org_id` | string | string | No | Organización que respondió la encuesta. | FK a `customers_orgs`. | Debe existir en `customers_orgs`. | No se detectaron huérfanos. | Mantener y validar integridad. |
| `survey_date` | string | date | No | Fecha de la encuesta. | `YYYY-MM-DD`. | Fecha válida. | Llega como texto; 10 encuestas son anteriores al alta del cliente. | Castear a `date`; marcar inconsistencia temporal como warning. |
| `nps_score` | numeric | integer | Sí | Puntaje NPS de la encuesta. | Valores observados entre -16 y 68. | Si existe, debe estar entre -100 y 100. | 19 nulos. | Permitir nulo; mantener valor válido. |
| `comment` | string | string | Sí | Comentario asociado a la encuesta. | 6 categorías observadas + vacío. | Campo opcional. | 10 nulos; en la práctica se comporta como categoría, no texto libre. | Mantener como categórico mientras se confirme semántica. |

### Riesgos / observaciones

- 10 encuestas (~11 %) tienen fecha anterior al alta del cliente.
- El `nps_score` de `customers_orgs` no coincide con la primera, última ni media de las encuestas observadas.

---

## 4.3 `users.csv`

### Ficha de la fuente

| Propiedad | Valor |
|---|---|
| **Descripción** | Usuarios pertenecientes a las organizaciones cliente. |
| **Grano** | Una fila por usuario. |
| **Clave primaria** | `user_id` |
| **Volumen observado** | 800 registros, 7 columnas. |
| **Relación principal** | `org_id -> customers_orgs.org_id` |
| **Modo de ingesta** | Batch |

### Campos

| Campo | Tipo origen | Tipo objetivo | Nullable | Descripción | Valores / dominio observado | Regla de calidad | Problema detectado | Tratamiento propuesto |
|---|---|---|---|---|---|---|---|---|
| `user_id` | string | string | No | Identificador único del usuario. | Formato `user_` + 8 caracteres. | No nulo y único. | Ninguno detectado. | Mantener. |
| `org_id` | string | string | No | Organización a la que pertenece el usuario. | FK a `customers_orgs`. | Debe existir en maestro de clientes. | No hay huérfanos. | Validar integridad referencial. |
| `email` | string | string | No | Correo del usuario. | Patrón `user_xxx@example.com`. | Formato de email válido. | Dato sintético. | Mantener como atributo descriptivo. |
| `role` | string | string | No | Rol funcional del usuario. | admin, developer, devops, data_engineer, ml_engineer, analyst. | Debe pertenecer al dominio observado. | Ninguno detectado. | Mantener. |
| `active` | boolean | boolean | No | Indicador de usuario activo. | `true`, `false`. | Debe ser booleano. | Ninguno detectado. | Mantener. |
| `created_at` | string | date | No | Fecha de creación del usuario. | `YYYY-MM-DD`. | Fecha válida. | 249 usuarios fueron creados antes del alta de su organización. | Castear; inconsistencia temporal como warning. |
| `last_login` | string | date | Sí | Última fecha de ingreso. | `YYYY-MM-DD`. | Si existe, fecha válida y esperablemente `>= created_at`. | 139 nulos; 232 casos con `last_login < created_at`. | Nulo permitido; flag temporal para casos inconsistentes. |

### Riesgos / observaciones

- Las inconsistencias temporales son frecuentes y parecen sistémicas, no casos aislados.
- No se recomienda descartar automáticamente estos registros en la primera definición de calidad.

---

## 4.4 `resources.csv`

### Ficha de la fuente

| Propiedad | Valor |
|---|---|
| **Descripción** | Recursos cloud contratados / utilizados por las organizaciones. |
| **Grano** | Una fila por recurso. |
| **Clave primaria** | `resource_id` |
| **Volumen observado** | 400 registros, 7 columnas. |
| **Relación principal** | `org_id -> customers_orgs.org_id` |
| **Modo de ingesta** | Batch |

### Campos

| Campo | Tipo origen | Tipo objetivo | Nullable | Descripción | Valores / dominio observado | Regla de calidad | Problema detectado | Tratamiento propuesto |
|---|---|---|---|---|---|---|---|---|
| `resource_id` | string | string | No | Identificador único del recurso. | Formato `res_` + 8 caracteres. | No nulo y único. | Ninguno detectado. | Mantener. |
| `org_id` | string | string | No | Organización propietaria del recurso. | FK a `customers_orgs`. | Debe existir en maestro. | No se detectaron huérfanos. | Validar integridad. |
| `service` | string | string | No | Tipo de servicio cloud. | compute, storage, database, networking, analytics, genai. | Debe pertenecer al catálogo permitido. | Ninguno detectado. | Normalizar contra catálogo común. |
| `region` | string | string | No | Región donde corre el recurso. | Mismos 7 códigos regionales observados en clientes. | Debe pertenecer al catálogo de regiones. | Ninguno detectado. | Mantener. |
| `created_at` | string | date | No | Fecha de creación del recurso. | `YYYY-MM-DD`. | Fecha válida. | 119 recursos creados antes del alta del cliente. | Castear; marcar inconsistencia temporal como warning. |
| `state` | string | string | No | Estado operativo del recurso. | running, stopped, terminated. | Debe pertenecer al dominio observado. | Ninguno detectado. | Mantener. |
| `tags_json` | string / null | array/map estructurado | Sí | Etiquetas asociadas al recurso. | `env`, `team`, `costcenter`, `pii`, `backup`. | JSON parseable cuando existe. | 83 nulos; estructura JSON embebida en texto. | Parsear en Silver; mantener raw en Bronze. |

### Riesgos / observaciones

- `pii:true` puede utilizarse para identificar recursos con posible exposición a datos personales y es relevante para gobierno / seguridad.
- `costcenter` puede aportar valor a FinOps.

---

## 4.5 `billing_monthly.csv`

### Ficha de la fuente

| Propiedad | Valor |
|---|---|
| **Descripción** | Facturación mensual por organización cliente. |
| **Grano** | Una factura por organización y mes. |
| **Clave primaria** | `invoice_id` |
| **Clave natural alternativa** | `org_id + month` |
| **Volumen observado** | 240 registros = 80 clientes x 3 meses. |
| **Modo de ingesta** | Batch mensual |
| **Dominio principal** | FinOps |

### Campos

| Campo | Tipo origen | Tipo objetivo | Nullable | Descripción | Valores / dominio observado | Regla de calidad | Problema detectado | Tratamiento propuesto |
|---|---|---|---|---|---|---|---|---|
| `invoice_id` | string | string | No | Identificador único de factura. | Formato `inv_` + 8 caracteres. | No nulo y único. | Ninguno detectado. | Mantener. |
| `org_id` | string | string | No | Organización facturada. | FK a `customers_orgs`. | Debe existir en maestro. | No hay huérfanos. | Validar integridad. |
| `month` | string | date | No | Mes facturado, representado por el primer día del mes. | 2025-06-01, 2025-07-01, 2025-08-01. | Fecha válida; idealmente día = 1. | Llega como texto; 5 facturas son de meses previos al alta del cliente. | Castear; warning temporal. |
| `subtotal` | numeric | decimal/double | No | Monto antes de impuestos. | Numérico. | Definición de signo debe ser consistente con negocio. | 13 facturas con subtotal negativo e impuesto positivo. | Flag de inconsistencia; no corregir automáticamente. |
| `credits` | numeric / null | decimal/double | Sí | Créditos / descuentos a favor del cliente. | Numérico. | Si nulo, requiere decisión de negocio. | 137 nulos (~57 %). | Hipótesis inicial: nulo puede representar 0; validar antes de imputar. |
| `taxes` | numeric | decimal/double | No | Impuestos asociados. | Observado como 21 % del subtotal en magnitud. | Coherencia con subtotal y regla impositiva esperada. | En subtotales negativos aparece impuesto positivo. | Flag y revisión de semántica. |
| `currency` | string | string | No | Moneda de facturación. | USD, ARS, EUR. | Debe pertenecer al catálogo permitido. | Un mismo cliente puede cambiar de moneda entre meses. | Mantener; no asumir moneda fija por cliente. |
| `exchange_rate_to_usd` | numeric | double | No | Cantidad de USD equivalentes a 1 unidad de moneda origen. | Numérico. | Para USD se esperaría 1, sujeto a confirmación. | Facturas USD presentan tasas entre 0,85 y 1,12. | Marcar como inconsistencia / decisión pendiente. |

### Riesgos / decisiones abiertas

- No está confirmado si `subtotal`, `credits` y `taxes` están expresados en `currency` o ya convertidos a USD.
- Debe confirmarse si una tasa distinta de 1 para facturas USD es ruido intencional.
- No corregir automáticamente importes hasta tener definición de negocio.

---

## 4.6 `support_tickets.csv`

### Ficha de la fuente

| Propiedad | Valor |
|---|---|
| **Descripción** | Tickets de soporte creados por organizaciones cliente. |
| **Grano** | Una fila por ticket. |
| **Clave primaria** | `ticket_id` |
| **Volumen observado** | 1.000 registros, 8 columnas. |
| **Relación principal** | `org_id -> customers_orgs.org_id` |
| **Dominio principal** | Soporte |

### Campos

| Campo | Tipo origen | Tipo objetivo | Nullable | Descripción | Valores / dominio observado | Regla de calidad | Problema detectado | Tratamiento propuesto |
|---|---|---|---|---|---|---|---|---|
| `ticket_id` | string | string | No | Identificador único del ticket. | Formato `tkt_` + 8 caracteres. | No nulo y único. | Ninguno detectado. | Mantener. |
| `org_id` | string | string | No | Organización que generó el ticket. | FK a `customers_orgs`. | Debe existir en maestro. | No hay huérfanos. | Validar integridad. |
| `category` | string | string | No | Categoría del problema. | performance, security, availability, usability, integration, billing. | Debe pertenecer al dominio definido. | Ninguno detectado. | Mantener. |
| `severity` | string | string | No | Severidad del ticket. | low, medium, high, critical. | Debe pertenecer al dominio definido. | Ninguno detectado. | Mantener. |
| `created_at` | string | date/timestamp | No | Fecha de apertura. | Fecha válida. | Fecha válida. | 209 tickets son anteriores al alta del cliente. | Castear; warning temporal. |
| `resolved_at` | string / null | date/timestamp | Sí | Fecha de resolución. | Fecha válida. | Si existe, debe ser `>= created_at`. | 240 nulos, consistentes con tickets abiertos. | Permitir nulo; validar orden temporal. |
| `csat` | numeric / null | integer | Sí | Satisfacción con la atención. | Escala esperada 1 a 5. | Si existe, `1 <= csat <= 5`. | 254 nulos; 40 valores fuera de escala; 172 tickets abiertos tienen CSAT. | Fuera de escala como inválido; tickets abiertos con CSAT como warning. |
| `sla_breached` | boolean | boolean | No | Indicador de incumplimiento de SLA. | `true`, `false`. | Debe ser booleano. | No muestra relación consistente con duración del ticket; faltan umbrales SLA por severidad. | Mantener como dato fuente; no recalcular sin definición de SLA. |

### Riesgos / observaciones

- No se dispone de los objetivos / umbrales de SLA por severidad, por lo que no puede recalcularse el breach de forma independiente.
- Hay tickets cerrados en septiembre, posteriores al período principal de otros datasets; esto es plausible y no se considera error.

---

## 4.7 `marketing_touches.csv`

### Ficha de la fuente

| Propiedad | Valor |
|---|---|
| **Descripción** | Interacciones de marketing con organizaciones cliente o prospectos. |
| **Grano** | Un contacto de campaña por organización, canal y fecha. |
| **Clave primaria** | `touch_id` |
| **Volumen observado** | 1.500 registros, 7 columnas. |
| **Relación principal** | `org_id -> customers_orgs.org_id` |
| **Modo de ingesta** | Batch |

### Campos

| Campo | Tipo origen | Tipo objetivo | Nullable | Descripción | Valores / dominio observado | Regla de calidad | Problema detectado | Tratamiento propuesto |
|---|---|---|---|---|---|---|---|---|
| `touch_id` | string | string | No | Identificador único del contacto. | Formato `mkt_` + 8 caracteres. | No nulo y único. | Ninguno detectado. | Mantener. |
| `org_id` | string | string | No | Organización contactada. | FK a `customers_orgs`. | Debe existir en maestro. | No hay huérfanos. | Validar integridad. |
| `campaign` | string | string | No | Campaña de marketing. | welcome, genai_launch, security_week, webinar_finops, upgrade_pro, upgrade_enterprise. | Debe pertenecer al dominio observado. | Ninguno detectado. | Mantener. |
| `channel` | string | string | No | Canal de contacto. | email, ads, event, in_app. | Debe pertenecer al dominio observado. | Ninguno detectado. | Mantener. |
| `timestamp` | string | date | No | Fecha del contacto. | Fecha sin hora. | Fecha válida. | El nombre sugiere timestamp, pero solo contiene fecha. | Castear a `date`; documentar semántica real. |
| `clicked` | boolean | boolean | No | Indicador de clic / interacción. | `true`, `false`. | Debe ser booleano. | Ninguno estructural. | Mantener. |
| `converted` | boolean | boolean | No | Indicador de conversión. | `true`, `false`. | Debe ser booleano. | 96 de 144 conversiones ocurrieron sin clic; no necesariamente error. | Mantener como comportamiento observado. |

### Riesgos / observaciones

- Los contactos anteriores al alta del cliente son plausibles porque marketing puede actuar sobre prospects; no deben marcarse como error temporal.
- Las tasas de clic y conversión son muy homogéneas entre campañas y canales, lo que limita análisis comparativos de efectividad.

---

## 4.8 `usage_events_stream/*.jsonl`

### Ficha de la fuente

| Propiedad | Valor |
|---|---|
| **Descripción** | Eventos de uso de recursos cloud, fragmentados en archivos JSONL para simular micro-lotes. |
| **Grano** | Un evento de uso de un recurso en un instante. |
| **Clave primaria** | `event_id` |
| **Volumen observado** | 120 archivos x 360 eventos = 43.200 eventos. |
| **Relaciones** | `org_id -> customers_orgs`; `resource_id -> resources`. |
| **Modo de ingesta** | Structured Streaming / micro-batch. |
| **Evolución de esquema** | `schema_version = 1` y `2`. |

### Campos

| Campo | Tipo origen | Tipo objetivo | Nullable | Descripción | Valores / dominio observado | Regla de calidad | Problema detectado | Tratamiento propuesto |
|---|---|---|---|---|---|---|---|---|
| `event_id` | string | string | No | Identificador único del evento. | ID. | No nulo y único. | No se detectaron duplicados en la muestra. | Deduplicar igualmente por idempotencia en streaming. |
| `org_id` | string | string | No | Organización asociada al evento. | FK a `customers_orgs`. | Debe existir en maestro. | Integridad completa observada. | Validar integridad referencial. |
| `resource_id` | string | string | No | Recurso que generó el evento. | FK a `resources`. | Debe existir y pertenecer al `org_id` del evento. | Integridad completa observada. | Validar relación recurso-organización. |
| `timestamp` | string | timestamp | No | Fecha y hora UTC del evento. | ISO-8601, ej. `2025-08-17T01:55:00Z`. | Timestamp válido. | Llega como texto. | Castear a timestamp UTC. |
| `service` | string | string | No | Servicio cloud. | compute, storage, database, networking, analytics, genai. | Debe pertenecer al catálogo permitido. | Ninguno detectado. | Normalizar. |
| `region` | string | string | No | Región del recurso. | Catálogo regional. | Debe pertenecer al catálogo. | Ninguno detectado. | Normalizar. |
| `metric` | string | string | No | Métrica medida. | requests, cpu_hours, storage_gb_hours. | Debe pertenecer al dominio observado. | Todos los servicios pueden registrar las 3 métricas; poco realista pero consistente en el dataset. | Mantener como característica del dataset. |
| `value` | string / numeric / null | double | Sí | Valor de la métrica. | Numérico. | Debe ser casteable a número cuando exista. | 1.309 valores llegan como texto; 877 nulos. | Safe cast; registros no convertibles a quarantine. |
| `unit` | string / null | string | Sí | Unidad asociada a la métrica. | requests -> count; cpu_hours -> hours; storage_gb_hours -> gb_hours. | Si existe `value`, se espera `unit` no nulo. | 2.075 nulos; 2.038 con `value` presente. | Inferir desde `metric` si la regla de negocio lo permite y mantener flag de imputación. |
| `cost_usd_increment` | numeric | double | No | Costo incremental del evento en USD. | Numérico. | Regla propuesta: `>= -0.01` o flag de anomalía. | 216 negativos (~0,5 %) y outliers extremos. | Conservar valor original + `anomaly_flag`; no eliminar automáticamente. |
| `schema_version` | integer | integer | No | Versión del esquema del evento. | 1, 2. | Debe pertenecer a `{1,2}`. | Ninguno detectado. | Usar para compatibilizar campos opcionales. |
| `carbon_kg` | ausente / numeric | double | Sí | Carbono generado por el evento. | Disponible en v2. | Esperado según `schema_version`. | No existe físicamente en v1. | Incorporar como nullable en esquema unificado. |
| `genai_tokens` | ausente / numeric | long | Sí | Tokens de IA generativa consumidos. | Disponible en v2 y servicio `genai`. | Solo aplicable a eventos GenAI de v2. | No existe en v1 y no aplica a otros servicios. | Nullable condicionado por versión y servicio. |

### Evolución de esquema observada

| Versión | Período observado | Registros | Diferencias |
|---|---|---:|---|
| `1` | 03/07/2025 al 17/07/2025 | 10.800 | No incluye `carbon_kg` ni `genai_tokens`. |
| `2` | 18/07/2025 al 31/08/2025 | 32.400 | Agrega `carbon_kg` y, para `genai`, `genai_tokens`. |

### Riesgos / observaciones

- 7.371 eventos (~17 %) tienen timestamp anterior a `resources.created_at`.
- No hay eventos anteriores al alta de la organización.
- Se observan exactamente 720 eventos por día durante 60 días, señal de dataset sintético balanceado.
- Los problemas temporales parecen concentrarse en la confiabilidad de `resources.created_at` más que en el stream.

---

# 5. Reglas de calidad transversales

## 5.1 Integridad referencial

| ID | Regla | Severidad | Acción inicial sugerida |
|---|---|---|---|
| `DQ-RI-001` | Todo `org_id` transaccional debe existir en `customers_orgs`. | Error | Quarantine / rechazo controlado. |
| `DQ-RI-002` | Todo `resource_id` de `usage_events_stream` debe existir en `resources`. | Error | Quarantine / rechazo controlado. |
| `DQ-RI-003` | El `resource_id` del evento debe pertenecer al mismo `org_id` informado en el evento. | Error | Quarantine / rechazo controlado. |

## 5.2 Fechas y consistencia temporal

| ID | Regla | Severidad | Acción inicial sugerida |
|---|---|---|---|
| `DQ-TIME-001` | `users.created_at >= customers_orgs.signup_date`. | Warning | Flag de inconsistencia temporal. |
| `DQ-TIME-002` | `users.last_login >= users.created_at` cuando `last_login` exista. | Warning | Flag. |
| `DQ-TIME-003` | `resources.created_at >= customers_orgs.signup_date`. | Warning | Flag. |
| `DQ-TIME-004` | `support_tickets.resolved_at >= support_tickets.created_at` cuando exista. | Error | Quarantine si se viola. |
| `DQ-TIME-005` | `usage_events.timestamp >= resources.created_at`. | Warning | Flag; revisar confiabilidad de `resources.created_at`. |
| `DQ-TIME-006` | `billing_monthly.month` no debería ser anterior al alta del cliente. | Warning | Flag. |
| `DQ-TIME-007` | `nps_surveys.survey_date` no debería ser anterior al alta del cliente. | Warning | Flag. |

## 5.3 Dominios y rangos

| ID | Regla | Severidad | Acción inicial sugerida |
|---|---|---|---|
| `DQ-DOM-001` | `nps_score` debe estar entre -100 y 100. | Error | Invalidar valor / quarantine según implementación. |
| `DQ-DOM-002` | `csat` debe estar entre 1 y 5 cuando exista. | Error | Invalidar valor / quarantine según implementación. |
| `DQ-DOM-003` | Los campos categóricos deben pertenecer a sus dominios documentados. | Error | Rechazo o quarantine si no existe categoría válida. |
| `DQ-DOM-004` | `schema_version` debe ser 1 o 2. | Error | Quarantine. |

## 5.4 Reglas específicas de usage

| ID | Regla | Severidad | Acción inicial sugerida |
|---|---|---|---|
| `DQ-USAGE-001` | `event_id` no debe ser nulo. | Error | Quarantine. |
| `DQ-USAGE-002` | `event_id` debe ser único dentro del horizonte de deduplicación. | Error | Deduplicar. |
| `DQ-USAGE-003` | Si `value` existe, debería existir `unit`. | Warning / Error según implementación | Inferir unidad si es determinística y mantener flag; caso contrario quarantine. |
| `DQ-USAGE-004` | `value` debe poder convertirse a tipo numérico cuando exista. | Error | Safe cast; quarantine si falla. |
| `DQ-USAGE-005` | `cost_usd_increment >= -0.01` o debe marcarse como anomalía. | Warning | Mantener valor y generar flag. |
| `DQ-USAGE-006` | `genai_tokens` solo aplica cuando `schema_version = 2` y `service = genai`. | Warning / Error | Validar coherencia semántica. |

---

# 6. Problemas conocidos y decisiones pendientes

| ID | Fuente | Problema / hallazgo | Impacto potencial | Tratamiento inicial | Estado |
|---|---|---|---|---|---|
| `ISS-001` | customers | `nps_score = 101` fuera de rango. | KPI NPS incorrecto. | Flag / invalidar valor. | Identificado |
| `ISS-002` | customers | `plan_tier` e `is_enterprise` no son consistentes entre sí. | Segmentación ambigua. | Mantener ambos hasta aclarar semántica. | Pendiente definición |
| `ISS-003` | nps | Encuestas anteriores al alta del cliente. | Inconsistencia temporal. | Warning. | Identificado |
| `ISS-004` | users | Usuarios creados antes del alta y logins anteriores a creación. | Inconsistencia temporal frecuente. | Warning; no descartar automáticamente. | Identificado |
| `ISS-005` | resources | Recursos creados antes del alta del cliente. | Afecta validaciones temporales del stream. | Warning y revisión de confiabilidad de fecha. | Identificado |
| `ISS-006` | billing | Subtotales negativos con impuestos positivos. | Revenue / impuestos incorrectos. | Flag; no corregir sin regla de negocio. | Identificado |
| `ISS-007` | billing | `exchange_rate_to_usd` distinto de 1 para USD. | Conversión monetaria dudosa. | Validar definición. | Pendiente definición |
| `ISS-008` | billing | Moneda de los importes no confirmada. | Riesgo en revenue normalizado a USD. | Confirmar con docente / fuente. | Pendiente definición |
| `ISS-009` | support | CSAT fuera de escala 1-5. | KPI de satisfacción incorrecto. | Invalidar / quarantine según implementación. | Identificado |
| `ISS-010` | support | SLA breach no puede recalcularse por falta de umbrales. | Limitación analítica. | Conservar flag fuente; documentar limitación. | Pendiente definición |
| `ISS-011` | marketing | Conversiones sin clic y tasas muy homogéneas. | Baja capacidad explicativa del dataset. | Mantener; documentar limitación. | Identificado |
| `ISS-012` | usage | `value` puede venir como string. | Error de tipificación. | Safe cast. | Definido |
| `ISS-013` | usage | `unit` nulo con `value` presente. | Semántica incompleta. | Inferencia por métrica + flag, si se aprueba. | Pendiente implementación |
| `ISS-014` | usage | Costos negativos y outliers extremos. | FinOps / anomalías. | Mantener + anomaly flag. | Definido |
| `ISS-015` | usage/resources | Eventos anteriores a `resources.created_at`. | Inconsistencia temporal. | Warning; revisar calidad de fecha de recurso. | Identificado |

---

# 7. Resumen de calidad observado

| Control | Resultado observado |
|---|---|
| PK duplicadas en archivos con identificador propio | No detectadas en las muestras analizadas. |
| Huérfanos por `org_id` | No detectados en las fuentes revisadas. |
| Huérfanos por `resource_id` en usage | No detectados. |
| Fechas almacenadas como texto | Frecuente en varias fuentes. |
| Inconsistencias temporales | Presentes en users, resources, tickets, NPS y billing. |
| Valores fuera de rango | NPS y CSAT. |
| Tipos ambiguos | Principalmente `usage.value`. |
| Evolución de esquema | Presente en `usage_events_stream` entre v1 y v2. |
| Outliers / anomalías | Costos negativos y picos extremos en usage; subtotales negativos en billing. |

---

# 8. Trazabilidad con el proyecto

Este diccionario busca soportar los siguientes puntos de la primera entrega:

- inventario y perfil inicial de las fuentes;
- definición de grano e identificadores;
- frecuencia y modo de ingesta;
- tipos y problemas de tipificación;
- relaciones e integridad referencial;
- problemas de calidad y riesgos;
- supuestos y decisiones todavía abiertas;
- base documental para la posterior implementación de Bronze, Silver y reglas de quarantine.

No pretende reemplazar el análisis exploratorio completo ni adelantar una implementación detallada del pipeline. Las reglas y tratamientos aquí propuestos deben validarse durante la etapa técnica y actualizarse a medida que el proyecto evolucione.

---

## 9. Próximos pasos sugeridos

1. Validar con el equipo / docente las decisiones pendientes de `billing_monthly`.
2. Confirmar el significado de `is_enterprise` respecto de `plan_tier`.
3. Definir qué reglas producirán **quarantine** y cuáles solamente **flags** en Silver.
4. Vincular las reglas `DQ-*` con la implementación Spark cuando se avance a la segunda entrega.
5. Mantener este documento versionado junto con los cambios de esquema y decisiones del proyecto.
