# Diagrama de Negocio — Cloud Provider Analytics

> **Propósito**  
> Representar de forma visual cómo funciona el negocio del proveedor cloud, qué procesos generan información y qué áreas internas consumen esos datos.  
> Esta vista se mantiene deliberadamente en nivel **negocio / información**: muestra origen, fuente, frecuencia y uso analítico, pero no incorpora todavía detalles de Landing, Bronze, Silver, Gold, Spark o Cassandra.

---

## 1. Contexto del negocio

La organización analizada es un **proveedor de servicios cloud** que ofrece cómputo, almacenamiento, bases de datos, networking, analytics e IA generativa a organizaciones cliente.

Los datos del proyecto representan la **operación del proveedor sobre sus clientes**: quiénes son, qué recursos tienen, cómo utilizan los servicios, cuánto se les factura, qué reclamos realizan y qué nivel de satisfacción expresan. No representan el contenido que los clientes almacenan dentro de la nube.

Existen dos velocidades principales de información:

- **Streaming / near real-time:** eventos de uso y costo incremental.
- **Batch:** maestros, usuarios, recursos, facturación, soporte, marketing y encuestas.

---

## 2. Vista integral del negocio y sus datos

```mermaid
flowchart LR

    %% =====================================================
    %% CLIENTE
    %% =====================================================
    subgraph CLIENTE["ORGANIZACIÓN CLIENTE"]
        ORG["Empresa / tenant<br/><b>Contrata servicios cloud</b>"]
        PEOPLE["Usuarios del cliente<br/><b>Operan los servicios</b>"]
        CLOUD["Recursos cloud<br/><b>Compute · Storage · DB · Network · Analytics · GenAI</b>"]

        ORG --> PEOPLE
        ORG --> CLOUD
    end

    %% =====================================================
    %% PROCESOS DEL NEGOCIO
    %% =====================================================
    subgraph OPERACION["OPERACIÓN DEL PROVEEDOR CLOUD"]
        CRM["CRM / Gestión comercial<br/><span>Cliente · plan · industria · lifecycle<br/>Carga: vendedores / proceso comercial</span>"]
        IAM["Cuentas de usuario<br/><span>Roles · actividad · último acceso<br/>Carga: cliente</span>"]
        INVENTORY["Inventario de recursos<br/><span>Servicio · región · estado · tags<br/>Carga: cliente / plataforma</span>"]
        METERING["Medición de uso<br/><span>Requests · CPU · storage · costo<br/>Carga: automática y continua</span>"]
        BILLING["Facturación<br/><span>Subtotal · créditos · impuestos · moneda<br/>Carga: automática</span>"]
        SUPPORT["Mesa de ayuda<br/><span>Tickets · severidad · SLA · CSAT<br/>Carga: cliente y agentes</span>"]
        MARKETING["Marketing<br/><span>Campañas · canales · clic · conversión<br/>Carga: equipo de marketing</span>"]
        SURVEYS["Encuestas NPS<br/><span>Score · comentario · fecha<br/>Carga: cliente</span>"]
    end

    %% Relación cliente -> operación
    ORG --> CRM
    PEOPLE --> IAM
    CLOUD --> INVENTORY
    PEOPLE --> METERING
    CLOUD --> METERING
    ORG --> BILLING
    ORG --> SUPPORT
    ORG --> MARKETING
    ORG --> SURVEYS

    %% =====================================================
    %% FUENTES DE DATOS
    %% =====================================================
    subgraph DATA["DATOS GENERADOS POR LA OPERACIÓN"]
        CUSTOMERS["<b>customers_orgs.csv</b><br/>1 organización por fila<br/>Batch"]
        USERS["<b>users.csv</b><br/>1 usuario por fila<br/>Batch"]
        RESOURCES["<b>resources.csv</b><br/>1 recurso por fila<br/>Batch"]
        USAGE["<b>usage_events_stream/*.jsonl</b><br/>1 evento de uso por fila<br/><b>Streaming / micro-batch</b>"]
        BILL["<b>billing_monthly.csv</b><br/>1 factura por organización / mes<br/>Batch mensual"]
        TICKETS["<b>support_tickets.csv</b><br/>1 ticket por fila<br/>Batch"]
        MKT["<b>marketing_touches.csv</b><br/>1 interacción por fila<br/>Batch"]
        NPS["<b>nps_surveys.csv</b><br/>1 encuesta por organización / fecha<br/>Batch"]
    end

    CRM --> CUSTOMERS
    IAM --> USERS
    INVENTORY --> RESOURCES
    METERING --> USAGE
    BILLING --> BILL
    SUPPORT --> TICKETS
    MARKETING --> MKT
    SURVEYS --> NPS

    %% =====================================================
    %% DOMINIOS DE CONSUMO
    %% =====================================================
    subgraph CONSUMO["CONSUMO ANALÍTICO INTERNO"]
        FINOPS["<b>FINOPS</b><br/>Costos · consumo · revenue<br/>créditos · impuestos · anomalías<br/>eficiencia por organización y servicio"]
        CARE["<b>SOPORTE</b><br/>Volumen de tickets · severidad<br/>cumplimiento de SLA · CSAT<br/>evolución por organización y fecha"]
        PRODUCT["<b>PRODUCTO / USAGE</b><br/>Uso de servicios · requests<br/>métricas operativas · GenAI tokens<br/>carbono por organización y servicio"]
        CX["<b>CONTEXTO COMERCIAL / CX</b><br/>Lifecycle · campañas · conversión<br/>NPS y señales de satisfacción"]
    end

    %% FinOps
    CUSTOMERS --> FINOPS
    RESOURCES --> FINOPS
    USAGE --> FINOPS
    BILL --> FINOPS

    %% Soporte
    CUSTOMERS --> CARE
    TICKETS --> CARE

    %% Producto
    CUSTOMERS --> PRODUCT
    RESOURCES --> PRODUCT
    USAGE --> PRODUCT

    %% Contexto comercial / experiencia
    CUSTOMERS --> CX
    USERS --> CX
    MKT --> CX
    NPS --> CX

    %% Contexto adicional hacia los dominios principales
    NPS -. "señal de satisfacción" .-> PRODUCT
    TICKETS -. "feedback operativo" .-> PRODUCT
    MKT -. "contexto comercial" .-> FINOPS

    %% =====================================================
    %% ESTILOS
    %% =====================================================
    classDef client fill:#dae8fc,stroke:#6c8ebf,color:#1f1f1f,stroke-width:1.5px;
    classDef process fill:#fff2cc,stroke:#d6b656,color:#1f1f1f,stroke-width:1.5px;
    classDef batch fill:#d5e8d4,stroke:#82b366,color:#1f1f1f,stroke-width:1.5px;
    classDef stream fill:#f8cecc,stroke:#b85450,color:#1f1f1f,stroke-width:2px;
    classDef consumer fill:#e1d5e7,stroke:#9673a6,color:#1f1f1f,stroke-width:1.5px;

    class ORG,PEOPLE,CLOUD client;
    class CRM,IAM,INVENTORY,METERING,BILLING,SUPPORT,MARKETING,SURVEYS process;
    class CUSTOMERS,USERS,RESOURCES,BILL,TICKETS,MKT,NPS batch;
    class USAGE stream;
    class FINOPS,CARE,PRODUCT,CX consumer;
```

### Lectura del diagrama

El flujo puede leerse de izquierda a derecha:

1. La **organización cliente** contrata servicios, administra usuarios y opera recursos cloud.
2. Esa actividad interactúa con distintos **procesos del proveedor**: CRM, cuentas, inventario, medición, facturación, soporte, marketing y encuestas.
3. Cada proceso deja una **fuente de datos identificable**, con un grano y una velocidad de generación determinados.
4. Las fuentes son utilizadas por distintos **dominios analíticos internos**, principalmente FinOps, Soporte y Producto / Usage.
5. Algunas fuentes aportan **contexto transversal**. Por ejemplo, NPS y tickets pueden complementar la lectura de Producto, mientras que marketing aporta contexto comercial, pero no son la fuente principal de las métricas operativas.

---

## 3. Mapa proceso → dato → uso

| Proceso de negocio | Quién / qué lo genera | Fuente resultante | Grano | Velocidad | Uso analítico principal |
|---|---|---|---|---|---|
| Gestión comercial / CRM | Vendedores y proceso comercial | `customers_orgs.csv` | 1 organización | Batch | Contexto maestro para todos los dominios |
| Gestión de cuentas | Cliente | `users.csv` | 1 usuario | Batch | Actividad y composición de usuarios |
| Inventario cloud | Cliente / plataforma | `resources.csv` | 1 recurso | Batch | Producto, capacidad y contexto de costos |
| Medición de uso | Plataforma | `usage_events_stream/*.jsonl` | 1 evento de uso | **Streaming** | Uso, costos incrementales, requests, GenAI y carbono |
| Facturación | Sistema de billing | `billing_monthly.csv` | 1 organización / mes | Batch mensual | Revenue, créditos, impuestos y FX |
| Mesa de ayuda | Cliente y agentes | `support_tickets.csv` | 1 ticket | Batch | Severidad, SLA y CSAT |
| Marketing | Equipo de marketing | `marketing_touches.csv` | 1 interacción | Batch | Campañas, clics y conversiones |
| Encuestas | Cliente | `nps_surveys.csv` | 1 organización / fecha | Batch | NPS y experiencia del cliente |

---

## 4. Dominios de negocio y preguntas que deben responder

### FinOps

**Objetivo:** entender el comportamiento económico y de consumo de cada organización.

Fuentes principales: `customers_orgs`, `resources`, `usage_events_stream` y `billing_monthly`.

Preguntas de referencia:

- ¿Cuánto cuesta y cuánto consume cada organización por servicio y por día?
- ¿Qué servicios concentran el mayor costo acumulado?
- ¿Existen incrementos de costo anómalos o fuera del comportamiento esperado?
- ¿Cuál es el revenue mensual considerando créditos, impuestos y conversión de moneda?
- ¿Cómo se distribuyen consumo y costos entre clientes, regiones y servicios?

### Soporte

**Objetivo:** medir la operación de atención al cliente y la calidad del servicio prestado.

Fuentes principales: `customers_orgs` y `support_tickets`.

Preguntas de referencia:

- ¿Cuántos tickets se generan por organización, fecha, categoría y severidad?
- ¿Cómo evolucionan los tickets críticos?
- ¿Qué proporción incumple SLA?
- ¿Cuál es el nivel de satisfacción CSAT en los tickets resueltos?
- ¿Qué clientes presentan mayor presión de soporte?

### Producto / Usage

**Objetivo:** entender cómo se utilizan los servicios y detectar patrones operativos relevantes.

Fuentes principales: `customers_orgs`, `resources` y `usage_events_stream`.

Preguntas de referencia:

- ¿Qué servicios utiliza cada organización y con qué intensidad?
- ¿Cómo evolucionan requests, CPU y almacenamiento?
- ¿Cuántos tokens GenAI se consumen por día cuando la información está disponible?
- ¿Cuál es el costo asociado al uso de GenAI?
- ¿Qué nivel de emisiones de carbono se observa en los eventos que lo informan?

### Contexto comercial y experiencia del cliente

**Objetivo:** complementar la lectura operacional con información de relación, adquisición y satisfacción.

Fuentes principales: `customers_orgs`, `users`, `marketing_touches` y `nps_surveys`.

Preguntas de referencia:

- ¿En qué etapa del ciclo de vida se encuentra cada organización?
- ¿Qué canales y campañas generan interacciones y conversiones?
- ¿Cómo evoluciona el NPS de las organizaciones?
- ¿Existen señales de satisfacción o riesgo que puedan contextualizar el uso, la facturación o el soporte?

> Este bloque se considera **contextual** en la primera versión del proyecto. Los tres dominios analíticos requeridos como foco principal son FinOps, Soporte y Producto / Usage.

---

## 5. Relaciones de negocio clave

```mermaid
flowchart LR
    ORG["Organización"] -->|tiene| USERS["Usuarios"]
    ORG -->|posee| RES["Recursos cloud"]
    USERS -->|utilizan| RES
    RES -->|generan| EVT["Eventos de uso"]
    EVT -->|generan| COST["Costo incremental"]
    ORG -->|recibe| INV["Facturación mensual"]
    ORG -->|genera| TKT["Tickets de soporte"]
    ORG -->|recibe| CAM["Contactos de marketing"]
    ORG -->|responde| NPS["Encuestas NPS"]

    classDef entity fill:#dae8fc,stroke:#6c8ebf,color:#1f1f1f;
    classDef activity fill:#d5e8d4,stroke:#82b366,color:#1f1f1f;
    classDef event fill:#f8cecc,stroke:#b85450,color:#1f1f1f;

    class ORG,USERS,RES entity;
    class INV,TKT,CAM,NPS activity;
    class EVT,COST event;
```

Esta vista resume las relaciones semánticas sin reemplazar al DER. El **DER documenta claves y cardinalidades**; este diagrama explica qué significa cada vínculo desde el punto de vista del negocio.

---

## 6. Alcance y límites de esta vista

| Incluido | No incluido en este diagrama |
|---|---|
| Procesos que generan los datos | Implementación Spark |
| Quién carga o produce cada información | Landing / Bronze / Silver / Gold |
| Nombre y grano de las fuentes | Reglas detalladas de calidad |
| Diferencia entre batch y streaming | Quarantine |
| Áreas internas consumidoras | Particionado Parquet |
| Preguntas analíticas principales | Modelo físico de Cassandra / AstraDB |
| Relaciones semánticas de negocio | Claves PK/FK detalladas del DER |

La separación permite que cada artefacto tenga una responsabilidad clara:

- **Diagrama de negocio:** explica el caso y el flujo de información.
- **Diccionario de datos:** documenta campos, grano, tipos, calidad y problemas detectados.
- **DER:** documenta entidades, claves y relaciones.
- **Arquitectura:** explica cómo los datos pasan por ingesta, Data Lake, procesamiento y serving.

---

## 7. Leyenda visual

| Color | Significado |
|---|---|
| Azul | Organización cliente y elementos propios del cliente |
| Amarillo | Procesos operativos del proveedor cloud |
| Verde | Fuentes batch |
| Rojo / salmón | Fuente streaming / near real-time |
| Violeta | Áreas o dominios de consumo analítico |
| Línea continua | Flujo principal de información |
| Línea punteada | Información contextual / complementaria |

---

## 8. Síntesis

El negocio puede resumirse como una cadena simple:

**Organización cliente → operación de servicios cloud → generación de datos → análisis interno para FinOps, Soporte y Producto.**

La principal diferencia de velocidad está en `usage_events_stream`, que representa eventos continuos de utilización y costo. El resto de las fuentes aporta información maestra, financiera, comercial, de soporte o de experiencia que se procesa principalmente por batch.

Esta vista busca conservar suficiente información para justificar el caso de negocio y el flujo de datos de la primera entrega, sin trasladar al diagrama de negocio decisiones técnicas que pertenecen a la arquitectura de implementación.
