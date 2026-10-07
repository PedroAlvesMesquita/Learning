# Diagrama de Negocio — Cloud Provider Analytics

> Vista simplificada del negocio y de los principales flujos de información.  
> El objetivo es mostrar **qué ocurre en el negocio, qué datos genera y quién los utiliza**, sin mezclar todavía detalles de Landing, Bronze, Silver, Gold o Cassandra.

```mermaid
flowchart LR

    %% =========================
    %% ACTORES / CONTEXTO
    %% =========================
    subgraph A["CLIENTE"]
        ORG["Organización cliente"]
        PEOPLE["Usuarios"]
        CLOUD["Recursos cloud"]
    end

    %% =========================
    %% OPERACIÓN DEL PROVEEDOR
    %% =========================
    subgraph B["OPERACIÓN DEL PROVEEDOR CLOUD"]
        CRM["Gestión comercial<br/>y ciclo de vida"]
        USAGE["Uso de servicios<br/>cloud"]
        BILL["Facturación<br/>mensual"]
        SUPPORT["Soporte<br/>y SLA"]
        MKT["Marketing<br/>y campañas"]
        NPS["Satisfacción<br/>y NPS"]
    end

    %% =========================
    %% DATOS GENERADOS
    %% =========================
    subgraph C["DATOS GENERADOS"]
        D_CUSTOMER["Clientes"]
        D_USERS["Usuarios"]
        D_RESOURCES["Recursos"]
        D_USAGE["Eventos de uso<br/>Streaming"]
        D_BILL["Facturación"]
        D_SUPPORT["Tickets"]
        D_MKT["Interacciones de marketing"]
        D_NPS["Encuestas NPS"]
    end

    %% =========================
    %% CONSUMIDORES
    %% =========================
    subgraph D["CONSUMO ANALÍTICO"]
        FINOPS["FinOps<br/>costos · revenue · anomalías"]
        CARE["Soporte<br/>tickets · SLA · CSAT"]
        PRODUCT["Producto / Usage<br/>uso · requests · GenAI · carbono"]
    end

    %% Cliente y operación
    ORG --> CRM
    ORG --> PEOPLE
    ORG --> CLOUD
    PEOPLE --> USAGE
    CLOUD --> USAGE
    ORG --> BILL
    ORG --> SUPPORT
    ORG --> MKT
    ORG --> NPS

    %% Operación y datos
    CRM --> D_CUSTOMER
    PEOPLE --> D_USERS
    CLOUD --> D_RESOURCES
    USAGE --> D_USAGE
    BILL --> D_BILL
    SUPPORT --> D_SUPPORT
    MKT --> D_MKT
    NPS --> D_NPS

    %% Datos y consumo
    D_CUSTOMER --> FINOPS
    D_RESOURCES --> FINOPS
    D_USAGE --> FINOPS
    D_BILL --> FINOPS

    D_CUSTOMER --> CARE
    D_SUPPORT --> CARE

    D_CUSTOMER --> PRODUCT
    D_RESOURCES --> PRODUCT
    D_USAGE --> PRODUCT
    D_NPS --> PRODUCT

    %% Estilos
    classDef actor fill:#dae8fc,stroke:#6c8ebf,color:#1f1f1f,stroke-width:1.5px;
    classDef process fill:#fff2cc,stroke:#d6b656,color:#1f1f1f,stroke-width:1.5px;
    classDef batch fill:#d5e8d4,stroke:#82b366,color:#1f1f1f,stroke-width:1.5px;
    classDef stream fill:#f8cecc,stroke:#b85450,color:#1f1f1f,stroke-width:1.5px;
    classDef consumer fill:#e1d5e7,stroke:#9673a6,color:#1f1f1f,stroke-width:1.5px;

    class ORG,PEOPLE,CLOUD actor;
    class CRM,USAGE,BILL,SUPPORT,MKT,NPS process;
    class D_CUSTOMER,D_USERS,D_RESOURCES,D_BILL,D_SUPPORT,D_MKT,D_NPS batch;
    class D_USAGE stream;
    class FINOPS,CARE,PRODUCT consumer;
```

## Lectura del diagrama

El modelo se resume en cuatro bloques:

1. **Cliente:** la organización contrata recursos cloud y dispone de usuarios que los utilizan.
2. **Operación:** el proveedor gestiona clientes, registra el uso de los servicios, factura, atiende tickets y ejecuta acciones de marketing y satisfacción.
3. **Datos generados:** cada proceso produce una fuente analítica. Los eventos de uso son el único flujo continuo; el resto corresponde principalmente a información batch.
4. **Consumo analítico:** los datos se utilizan principalmente por **FinOps**, **Soporte** y **Producto / Usage**.

### Alcance de esta vista

Este diagrama representa el **negocio y sus flujos de información**. La arquitectura técnica —Landing, Bronze, Silver, Gold, Spark y Cassandra/AstraDB— se documenta por separado para evitar mezclar la interpretación del negocio con la implementación de la solución.
