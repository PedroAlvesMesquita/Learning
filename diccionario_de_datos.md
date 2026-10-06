# Diccionario de Datos — Cloud Provider Analytics

Este documento describe todas las tablas del datalake, su grano, campos, tipos, relaciones y problemas de calidad detectados.

---

## 1. Tabla: customers_orgs

### Descripción
Organizaciones clientes del proveedor de nube.

### Grano del dato
Una fila = una organización cliente.

### Tamaño
- Filas: 80  
- Columnas: 11  

### Identificador primario
- org_id

### Relaciones
- users.org_id  
- resources.org_id  
- billing_monthly.org_id  
- support_tickets.org_id  
- marketing_touches.org_id  
- nps_surveys.org_id  

### Campos
#### org_id
- Tipo: string  
- Formato: "org_" + 8 caracteres  
- Calidad: sin duplicados  

#### org_name
- Tipo: string  
- Observación: no es identificador confiable  

#### industry
- Tipo: string  
- Valores: 10 categorías  
- Calidad: sin problemas  

#### hq_region
- Tipo: string  
- Valores: 7 regiones  
- Calidad: limpio  

#### plan_tier
- Tipo: string  
- Valores: free, standard, pro, enterprise  
- Problema: inconsistente con is_enterprise  

#### is_enterprise
- Tipo: boolean  
- Problema: no coincide con plan_tier  

#### signup_date
- Tipo: string (fecha)  
- Problema: guardada como texto  

#### sales_rep
- Tipo: string  
- Valores: rep_a a rep_e  

#### lifecycle_stage
- Tipo: string  
- Valores: lead, prospect, active, at_risk, churned  

#### marketing_source
- Tipo: string  
- Valores: partner, event, organic, ads, referral  

#### nps_score
- Tipo: int  
- Problemas: 11 vacíos, 1 fuera de rango (101)

---

## 2. Tabla: nps_surveys

### Descripción
Encuestas NPS respondidas por los clientes.

### Grano del dato
Una fila = una encuesta por cliente y fecha.

### Tamaño
- Filas: 92  
- Columnas: 4  

### Identificador
- org_id + survey_date

### Relaciones
- customers_orgs.org_id

### Campos
#### org_id
- Tipo: string  
- Calidad: sin huérfanos  

#### survey_date
- Tipo: string (fecha)  
- Problema: 10 encuestas antes del alta del cliente  

#### nps_score
- Tipo: int  
- Problema: 19 vacíos  

#### comment
- Tipo: string  
- Valores: 6 categorías  
- Problema: 10 vacíos

---

## 3. Tabla: users

### Descripción
Usuarios pertenecientes a cada organización.

### Grano del dato
Una fila = un usuario.

### Tamaño
- Filas: 800  
- Columnas: 7  

### Identificador
- user_id

### Relaciones
- customers_orgs.org_id

### Campos
#### user_id
- Tipo: string  
- Formato: "user_" + 8 caracteres  

#### org_id
- Tipo: string  
- Calidad: sin huérfanos  

#### email
- Tipo: string  

#### role
- Tipo: string  
- Valores: admin, developer, devops, data_engineer, ml_engineer, analyst  

#### active
- Tipo: boolean  

#### created_at
- Tipo: string (fecha)  
- Problema: 249 usuarios creados antes del alta del cliente  

#### last_login
- Tipo: string (fecha)  
- Problemas: 139 vacíos, 232 antes de created_at

---

## 4. Tabla: resources

### Descripción
Recursos alquilados por los clientes.

### Grano del dato
Una fila = un recurso.

### Tamaño
- Filas: 400  
- Columnas: 7  

### Identificador
- resource_id

### Relaciones
- customers_orgs.org_id  
- usage_events_stream.resource_id

### Campos
#### resource_id
- Tipo: string  
- Formato: "res_" + 8 caracteres  

#### org_id
- Tipo: string  
- Calidad: sin huérfanos  

#### service
- Tipo: string  
- Valores: compute, storage, database, networking, analytics, genai  

#### region
- Tipo: string  
- Valores: 7 regiones  

#### created_at
- Tipo: string (fecha)  
- Problema: 119 recursos creados antes del alta del cliente  

#### state
- Tipo: string  
- Valores: running, stopped, terminated  

#### tags_json
- Tipo: string (JSON)  
- Problema: 83 vacíos

---

## 5. Tabla: billing_monthly

### Descripción
Facturas mensuales por cliente.

### Grano del dato
Una fila = una factura mensual por cliente.

### Tamaño
- Filas: 240  
- Columnas: 8  

### Identificador
- invoice_id

### Relaciones
- customers_orgs.org_id

### Campos
#### invoice_id
- Tipo: string  

#### org_id
- Tipo: string  

#### month
- Tipo: string (fecha)  
- Problema: 5 facturas antes del alta del cliente  

#### subtotal
- Tipo: float  
- Problema: 13 negativos  

#### credits
- Tipo: float  
- Problema: 137 vacíos  

#### taxes
- Tipo: float  
- Regla: 21% del subtotal  

#### currency
- Tipo: string  
- Valores: USD, ARS, EUR  

#### exchange_rate_to_usd
- Tipo: float  
- Problema: USD ≠ 1

---

## 6. Tabla: support_tickets

### Descripción
Reclamos de soporte técnico.

### Grano del dato
Una fila = un ticket.

### Tamaño
- Filas: 1000  
- Columnas: 8  

### Identificador
- ticket_id

### Relaciones
- customers_orgs.org_id

### Campos
#### ticket_id
- Tipo: string  

#### org_id
- Tipo: string  

#### category
- Tipo: string  
- Valores: performance, security, availability, usability, integration, billing  

#### severity
- Tipo: string  
- Valores: low, medium, high, critical  

#### created_at
- Tipo: string (fecha)  

#### resolved_at
- Tipo: string (fecha)  
- Problemas: 240 vacíos, 34 fuera del período  

#### csat
- Tipo: int  
- Problemas: 254 vacíos, 40 fuera de escala  

#### sla_breached
- Tipo: boolean  
- Problema: no correlaciona con tiempos reales

---

## 7. Tabla: marketing_touches

### Descripción
Contactos de marketing enviados a los clientes.

### Grano del dato
Una fila = un contacto.

### Tamaño
- Filas: 1500  
- Columnas: 7  

### Identificador
- touch_id

### Relaciones
- customers_orgs.org_id

### Campos
#### touch_id
- Tipo: string  

#### org_id
- Tipo: string  

#### campaign
- Tipo: string  
- Valores: welcome, genai_launch, security_week, webinar_finops, upgrade_pro, upgrade_enterprise  

#### channel
- Tipo: string  
- Valores: email, ads, event, in_app  

#### timestamp
- Tipo: string (fecha)  

#### clicked
- Tipo: boolean  

#### converted
- Tipo: boolean  
- Observación: 67% de conversiones sin clic

---

## 8. Tabla: usage_events_stream

### Descripción
Eventos de uso en tiempo real.

### Grano del dato
Una fila = un evento de uso.

### Tamaño
- Filas: 43.200  
- Columnas: 13  

### Identificador
- event_id

### Relaciones
- customers_orgs.org_id  
- resources.resource_id

### Campos
#### event_id
- Tipo: string  

#### org_id
- Tipo: string  

#### resource_id
- Tipo: string  

#### timestamp
- Tipo: string (fecha-hora)  

#### service
- Tipo: string  

#### region
- Tipo: string  

#### metric
- Tipo: string  
- Valores: requests, cpu_hours, storage_gb_hours  

#### value
- Tipo: float / string  
- Problemas: 1309 como texto, 877 vacíos  

#### unit
- Tipo: string  
- Problemas: 2075 vacíos  

#### cost_usd_increment
- Tipo: float  
- Problemas: 216 negativos  

#### schema_version
- Tipo: int  
- Valores: 1, 2  

#### carbon_kg
- Tipo: float  
- Solo versión 2  

#### genai_tokens
- Tipo: int  
- Solo servicio genai + versión 2  

---

# Fin del diccionario
