# Arquitectura v1 · Patrón Lambda · 07/10/2026

## Vista general

En rojo, el camino streaming; en verde, el camino batch. Los dos caminos se encuentran en Silver, donde cada evento se une con los maestros limpios (flecha "referencia").

```mermaid
---
title: Arquitectura v1 · Patrón Lambda · 07/10/2026
config:
  flowchart:
    wrappingWidth: 400
---
flowchart TB
  subgraph FUENTES["Fuentes"]
    S1["Medición de uso<br/>eventos JSONL · continuo"]
    S2["7 sistemas batch<br/>CSV · diario"]
  end
  subgraph INGESTA["Ingesta"]
    I1["Ingesta continua"]
    I2["Ingesta programada diaria"]
  end
  subgraph LANDING["Landing"]
    L["Archivos originales<br/>no se modifican"]
  end
  subgraph BRONZE["Bronze · Parquet"]
    B1["usage_events<br/>por día, sin duplicados"]
    B2["7 tablas<br/>tipos correctos"]
  end
  subgraph SILVER["Silver"]
    SV1["Eventos<br/>limpios, controlados y unidos"]
    SV2["Maestros limpios"]
    Q["Cuarentena"]
  end
  subgraph GOLD["Gold"]
    G1["cost_anomaly_mart<br/>10 minutos o menos"]
    G2["4 tablas P1 a P5<br/>antes de las 6:00"]
  end
  subgraph CASS["Cassandra / AstraDB"]
    C1["Tabla de anomalías"]
    C2["Una tabla por consulta"]
  end
  subgraph CONSUMO["Consumo"]
    U1["FinOps<br/>monitoreo de costos"]
    U2["FinOps · Soporte · Producto<br/>P1 a P5"]
  end
  TR["Capacidades transversales<br/>calidad · gobierno · seguridad · metadatos · observabilidad"]

  S1 --> I1 --> L
  S2 --> I2 --> L
  L -- "Structured Streaming" --> B1
  L -- "Spark batch" --> B2
  B1 -- "cada pocos minutos" --> SV1
  B2 -- "diario" --> SV2
  SV2 -. "referencia" .-> SV1
  SV1 -.-> Q
  SV2 -.-> Q
  SV1 -- "cada pocos minutos" --> G1
  SV1 -. "lectura diaria" .-> G2
  SV2 -- "nocturno" --> G2
  G1 -- "carga continua" --> C1
  G2 -- "carga diaria" --> C2
  C1 --> U1
  C2 --> U2
  CONSUMO ~~~ TR

  classDef stream fill:#f8cecc,stroke:#b85450,color:#000
  classDef batch fill:#d5e8d4,stroke:#82b366,color:#000
  classDef land fill:#dae8fc,stroke:#6c8ebf,color:#000
  classDef quar fill:#e51400,stroke:#b20000,color:#fff
  classDef trans fill:#000,stroke:#000,color:#fff
  class S1,I1,B1,SV1,G1,C1,U1 stream
  class S2,I2,B2,SV2,G2,C2,U2 batch
  class L land
  class Q quar
  class TR trans
```

## Vista detallada

Diagrama completo con las tablas de cada zona, sus particiones y sus reglas. Hacer clic en la imagen para verla en tamaño completo.

![Arquitectura v1 · vista detallada](arquitectura_v1_detalle.svg)
