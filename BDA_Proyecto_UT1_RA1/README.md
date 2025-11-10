# Proyecto UT1 · Sensores de CO₂ (Micro‑batch, idempotencia, Parquet/SQLite)

Trabajo práctico de **Sistemas de Big Data**. Pipeline mínimo que:
1) **Genera** lecturas simuladas (NDJSON) con anomalías controladas.  
2) **Ingesta** en micro‑lotes con **checkpoint** e **idempotencia** (SQLite + WAL).  
3) **Limpia/valida** (rangos, dominios, horario lectivo) y **cuarentena**.  
4) **Persiste** en **Parquet** (raw/clean, particionado) y en **SQLite** (`ut1.db`).  
5) **(Opcional)** Genera **`output/reporte.md`** desde el notebook.

---

## 🗂️ Estructura del repositorio

```
project/
├─ data/
│  └─ drops/                  # Aquí se genera/lee lecturas.log (NDJSON)
├─ docs/                      # Documentación (ingesta, calidad, modelado, lecciones)
├─ output/
│  ├─ parquet/
│  │  ├─ raw/                 # Parquet de eventos crudos
│  │  └─ clean/               # Parquet de eventos limpios
│  ├─ quality/                # (opcional) reportes de calidad
│  ├─ checkpoints/            # Offsets (.offset) para read_new_lines()
│  ├─ ut1.db                  # SQLite (tablas: raw_events, clean_events, quarantine)
│  └─ reporte.md              # Reporte final en Markdown (opcional)
├─ notebooks/
│  └─ master_notebook.ipynb   # Notebook principal: genera → ingesta → limpia → exporta
└─ README.md
```

> ⚠️ El notebook crea la estructura automáticamente si no existe.

---

## ✅ Requisitos

- **Python 3.11+**
- **conda** (Miniforge/Anaconda) para aislar el entorno
- `requirements.txt` (ya incluido; se generó con `pip freeze > requirements.txt`)

---

## 🧪 Preparación del entorno (Conda + VS Code/Jupyter)

```bash
# 1) Crear y activar entorno
conda create -n ut1_co2 python=3.11 -y
conda activate ut1_co2

# 2) Instalar dependencias del proyecto
pip install -r requirements.txt

# 3) (Opcional) Registrar kernel para Jupyter/VS Code
python -m ipykernel install --user --name ut1_co2 --display-name "Python (ut1_co2)"
```

En **VS Code**: Selecciona el kernel **Python (ut1_co2)** en la barra superior del notebook.

---

## ▶️ Ejecución (desde el notebook)

Abre `project/notebooks/master_notebook.ipynb` y ejecuta, en orden:

1. **Configuración:** imports, constantes, bandas de calidad y rutas (crea estructura `project/...`).  
2. **Simulación:** genera `project/data/drops/lecturas.log` (NDJSON) con anomalías controladas.  
3. **Ingesta (micro‑lote):**
   - `ensure_db(DB)` crea tablas en `output/ut1.db`  
   - `read_new_lines()` lee **solo** nuevas líneas usando `checkpoints/`  
   - `upsert_raw()` inserta con **idempotencia** (`ON CONFLICT(event_id) DO UPDATE ...`)  
4. **Limpieza y calidad:**
   - Valida `co2_ppm` (300–5000), `aula` no vacía, y **horario lectivo** definido
   - Válidos → `clean_events` y `parquet/clean/`
   - Inválidos → `quarantine` con `cause` y `parquet/raw/` si procede
5. **(Opcional) Reporte:** ejecutar la celda final que escribe `output/reporte.md` (incluye métricas y fecha de generación).

> Re‑ejecutar ingesta no duplica datos: **idempotencia** por `event_id` + `_ingest_ts` (“último gana”).

---

## 🧾 Reporte (`output/reporte.md`)

Estructura sugerida (adaptada a CO₂):  
- **Titular** (ppm media, nº alertas)  
- **KPIs** (válidos, cuarentena, alertas >1500 ppm, aula con mayor variabilidad)  
- **Top aulas** por ppm media  
- **Evolución por hora** (tabla)  
- **Calidad y cobertura** (causas de cuarentena)  
- **Persistencia** (rutas Parquet/SQLite)  
- **Conclusiones** (ventilación/horarios/sensores a revisar)

> Si deseas que el reporte se **genere automáticamente**: deja activa la celda final del notebook que lo escribe.

---

## 🧰 Comandos útiles

```bash
# Ver remotos y rama actual (antes de hacer push)
git status
git remote -v

# (Opcional) cambiar remoto al repo personal
git remote set-url origin git@github.com:b0rjen/BDA_Proyecto_UT1_RA1.git

# Añadir/commit/push
git add .
git commit -m "UT1 CO2: simulacion, ingesta, limpieza, parquet y reporte"
git push origin main
```

---

## 🧹 Mantenimiento / reset

En el notebook tienes dos utilidades:

- `clear_database(DB)`: elimina tablas `raw_events`, `clean_events`, `quarantine` **sin borrar** `ut1.db`.  
- `reset_pipeline()`: borra `output/parquet/`, `output/checkpoints/` y `output/ut1.db` para rehacer todo.

**Nota sobre SQLite (WAL):** al usar `PRAGMA journal_mode=WAL;` se crean `ut1.db-wal` y `ut1.db-shm`.  
SQLite los gestiona automáticamente. Si deseas consolidar manualmente:  
```python
import sqlite3
with sqlite3.connect("project/output/ut1.db") as con:
    con.execute("PRAGMA wal_checkpoint(FULL);")
```

---

## 🐞 Problemas frecuentes

- **Permisos al crear carpetas** dentro de Google Drive/Cloud: mueve el proyecto a una ruta local (p. ej. `~/Proyectos/UT1_CO2/`) y vuelve a ejecutar la celda de creación de rutas.  
- **“database is locked”**: asegúrate de cerrar conexiones (`with sqlite3.connect(...)`) o usa el modo WAL (ya activado en `upsert_raw`).  
- **No se ve Mermaid en `reporte.md` en VS Code**: instala **Markdown Preview Mermaid Support** y encierra el diagrama en bloque ```mermaid.
```

---

## 📄 Licencia / Autoría

Trabajo académico de la asignatura **Sistemas de Big Data**.  
Datos simulados con fines docentes.  
Autor: **Borja Ramos (b0rjen)** · 2025.
