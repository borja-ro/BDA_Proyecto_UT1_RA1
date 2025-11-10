# 📊 Proyecto UT1 — Sensores de CO₂ (Microbatch con idempotencia)

## 1. Descripción general
Este pipeline simula lecturas de sensores de CO₂ en aulas, realiza la ingesta incremental
en SQLite con trazabilidad e idempotencia, limpia los datos según reglas de calidad y
almacena los resultados en formato Parquet particionado.

A continuación se muestra la arquitectura general del flujo de datos.

---

### 1.1 Diagrama general del flujo

```mermaid
flowchart TD
    %% ====== SECCIÓN 1: CONFIGURACIÓN Y ESTRUCTURA ======
    A1[🧭 1. Configuración inicial\n - Librerías\n - Parámetros Pandas\n - Constantes y rutas del proyecto] --> A2[📁 Creación de estructura de carpetas\n(project/data/output/...)]
    A2 --> A3[📊 Definición de BANDAS y HORARIO_LECTIVO]

    %% ====== SECCIÓN 2: GENERACIÓN DE DATOS ======
    subgraph B[2️⃣ Generación / Adquisición de datos]
        B1[🧩 Función generar_evento_id()] --> B2[⚙️ Función simular_lecturas()\n→ crea lecturas.log (NDJSON)\ncon anomalías]
        B2 --> B3[📂 Archivo generado:\nproject/data/drops/lecturas.log]
    end
    A3 --> B

    %% ====== SECCIÓN 3: INGESTA DE DATOS ======
    subgraph C[3️⃣ Ingesta de datos (microbatch)]
        C1[🗄️ ensure_db()\n→ crea tablas raw_events, clean_events, quarantine] --> C2[🔁 upsert_raw()\n→ inserta datos con idempotencia y WAL]
        C2 --> C3[📥 read_new_lines()\n→ lee solo líneas nuevas del log usando checkpoint]
        C3 --> C4[🧩 ingest_microbatch()\n→ orquesta lectura + inserción en raw_events]
        C4 -->|Checkpoint .offset| CKPT[(📄 output/checkpoints)]
    end
    B3 --> C

    %% ====== SECCIÓN 4: LIMPIEZA Y CALIDAD ======
    subgraph D[4️⃣ Limpieza y valoración de calidad]
        D1[🧹 clean_and_export()\n→ aplica reglas de validación:\n• out_of_range\n• missing_aula\n• out_of_hours\n• malformed] --> D2[✅ clean_events (válidos)]
        D1 --> D3[🚫 quarantine (inválidos)]
        D1 --> D4[💾 Exportación Parquet:\noutput/parquet/raw\noutput/parquet/clean]
    end
    C4 --> D

    %% ====== SECCIÓN 5: MANTENIMIENTO ======
    subgraph E[5️⃣ Mantenimiento y control]
        E1[⚡ Índices en SQLite\n(ix_raw_ts, ix_clean_ts_aula)] --> E2[🧽 clear_database()\n→ limpia tablas manteniendo DB]
        E2 --> E3[🧨 reset_pipeline()\n→ borra Parquet, checkpoints y ut1.db]
    end
    D --> E

    %% ====== ARCHIVOS DE SALIDA ======
    E --> F[📦 output/ut1.db\n(output/parquet/, quality/, checkpoints/)]

    %% ====== ETAPA FINAL ======
    F --> G[🧾 reporte.md\nResumen y visualización final]

    %% VISUAL STYLE
    classDef phase fill:#002b36,stroke:#ffffff,stroke-width:2px,color:#fff;
    class A1,A2,A3,B,C,D,E,F,G phase;
```

---

### 1.2 Versión horizontal (pipeline ETL)

```mermaid
flowchart LR
    %% ETAPA 1: CONFIGURACIÓN
    A1([🧭 Configuración inicial\nImports, rutas y parámetros]) --> 
    A2([📁 Estructura de carpetas\nproject/data/output]) --> 

    %% ETAPA 2: GENERACIÓN
    B1([⚙️ Simulación NDJSON\n(simular_lecturas → lecturas.log)]) --> 

    %% ETAPA 3: INGESTA
    C1([🗄️ Ingesta micro-batch\nensure_db + upsert_raw]) --> 
    C2([🧩 Checkpoint + idempotencia\n.read_new_lines / .offset]) --> 

    %% ETAPA 4: LIMPIEZA
    D1([🧹 Limpieza y calidad\nclean_and_export → reglas:\nout_of_range, out_of_hours, missing_aula]) --> 
    D2([💾 Exportación Parquet\n(raw / clean) + SQLite]) --> 

    %% ETAPA 5: MANTENIMIENTO
    E1([⚡ Índices en SQLite\nix_raw_ts / ix_clean_ts_aula]) --> 
    E2([🧽 clear_database / reset_pipeline]) --> 

    %% ETAPA 6: SALIDA FINAL
    F1([📦 Salidas finales:\nut1.db + parquet/ + reporte.md])

    %% STYLE
    classDef block fill:#073642,stroke:#eee,stroke-width:2px,color:#fff,font-size:12px;
    class A1,A2,B1,C1,C2,D1,D2,E1,E2,F1 block;
```

---

## 2. Interpretación
El flujo del pipeline sigue una arquitectura **ETL simplificada**, con separación clara entre:
- **Ingesta (Bronce):** captación incremental y almacenamiento crudo.
- **Limpieza (Plata):** validación, deduplicación y cuarentena de registros.
- **Almacenamiento / Reporte (Oro):** persistencia en SQLite y Parquet, con generación final de KPIs y alertas.

---
