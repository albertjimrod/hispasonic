# CONTEXT.md — Hispasonic

> Documento de retoma de contexto. Destilado de README.md, reports/README.md,
> requirements.txt, jupyterlab_new.yaml, hispaok/hispaok_pipeline.py, `git log` y
> `git status` (28-08-2026). No hay DECISIONS.md, TODO.md ni JOURNAL/docs en el repo.

## 1. Qué es

Pipeline de datos de extremo a extremo y análisis de mercado sobre anuncios de
sintetizadores/teclados de segunda mano publicados en Hispasonic (la mayor
comunidad hispanohablante de tecnología musical). Cubre el ciclo completo:
scraping del sitio → CSV por sesión → dataset unificado y limpio → EDA →
modelo de oferta/demanda por ciudad y precio. Es un proyecto analítico
personal (autor: Alberto Jiménez), no un servicio.

## 2. Stack

- **Lenguaje:** Python 3.9 (entorno canónico). Todo el trabajo vive en
  notebooks Jupyter + un script standalone.
- **Entorno:** conda `jupyterlab_new` definido en `jupyterlab_new.yaml`
  (JupyterLab 4.4). Paquetes clave fijados: pandas 2.2.3, numpy 2.0.1,
  pyarrow 21, matplotlib 3.9.2, seaborn 0.13.2, altair 5.5, scipy 1.13.1,
  scikit-learn 1.6.1, xgboost 2.1.1, shap 0.49.1, numba 0.60, streamlit 1.50.0.
- **Scraping:** `requests` + `BeautifulSoup4`. Fallback de detección de marca
  con **Ollama local** (`mistral:latest` en `localhost:11434`) y búsqueda web
  vía `ddgs` (DuckDuckGo, sin API key).
- **Legacy:** `requirements.txt` es un volcado de entorno conda Python 3.8
  (incluye scrapy, selenium, sqlalchemy, psycopg2, ipython-sql). Se conserva
  como referencia; **no** es el setup recomendado.
- **Despliegue:** ninguno. Se ejecuta en local. `streamlit` está fijado en el
  entorno pero **no existe ninguna app** en el repo — PENDIENTE DE CONFIRMAR si
  hay intención de dashboard.

## 3. Arquitectura

Dos bloques que hoy conviven sin integración formal:

**A. Scraper (`hispaok/`)** — genera un CSV por sesión de scrapeo (`hpwAAAAMD.csv`).
- `hispaok_pipeline.py`: script ejecutable en 10 fases — (1) recolección de URLs
  de anuncios paginando el listado, (2) descarga de HTMLs individuales con
  reanudación y reintentos con backoff, (3) construcción de un diccionario de
  marcas a partir de una lista `sintes` hardcodeada (~500 entradas) indexada por
  primera palabra para soportar marcas multi-palabra, (4) parseo de HTMLs y
  extracción de campos (marca, precio, ciudad, fecha, flags de acción
  vendo/compro/cambio/…), (5) fallback de marca con Ollama + búsqueda web,
  (6) autodescubrimiento de marcas nuevas desde los slugs de las URLs,
  (7) filtro IA (Ollama YES/NO) de esos candidatos, (8) construcción del
  DataFrame, (9) normalización de fechas relativas ("hace 3 semanas" → fecha
  absoluta), (10) guardado a CSV. Las marcas nuevas validadas se persisten en
  `hispaok/nuevas_marcas.json` y se recargan en cada ejecución.
- `01_from_web_to_csv_togit.ipynb`: versión notebook del mismo scraper (origen
  histórico del script).
- Rutas absolutas hardcodeadas (`/home/ion/Documentos/...`, `/home/ion/csv_hispa/...`)
  → el script no es portable tal cual.

**B. Análisis (`notebooks/`)** — consume los CSV en `data/raw/`.
- `01_etl.ipynb`: carga los **12 CSV** de `data/raw/` (ago-2022 → may-2024,
  **5.962 filas**), inspecciona columnas contra un esquema `CORE_COLUMNS` de 16
  campos, descarta columnas extra (`user`, `anon_user`, `Unnamed: 0`), añade
  `source_file`, corrige tipos (datetime, float, Int64 nullable), analiza nulos
  (solo 3 en `description`) y documenta duplicados. Salida:
  `data/processed/hispasonic_unified.csv` (5.962 × 17). No genera figuras.
- `02_eda.ipynb`: EDA — evolución temporal, precios, marcas, ciudades,
  engagement (`seen`), matriz de correlaciones. Genera **figuras 01–18** en
  `reports/figures/`.
- `03_supply_demand.ipynb`: define oferta (`sell==1`), demanda (`buy==1 or
  search==1`), ratio D/O, HHI de concentración, score compuesto de atractivo por
  ciudad, y test de hipótesis (transversal y con rezago t→t+1). Genera
  **figuras 19–27**.
- `reports/README.md`: galería con descripción de las 27 figuras.

**Datos:** `data/raw/` (12 CSV, versionados) → `data/processed/` (gitignored,
pero el `.csv` unificado existe en local) → `reports/figures/` (27 PNG).

## 4. Decisiones clave

- **Retener los duplicados (2.059 filas, 34,5%)** identificados por clave
  compuesta `[description, price, published, city]`. Un anuncio que aparece en 3
  scrapes estuvo activo 3 periodos; NB03 depende de conteos por fecha de scrape,
  así que eliminarlos distorsionaría la señal temporal. Alternativa descartada:
  deduplicar.
- **Almacenamiento en CSV plano**, no base de datos. El README antiguo
  (`README.sync-conflict-*.md`) y las imágenes `postgresql.png`/`csv_postgre.png`
  reflejan una idea previa de SQLite/PostgreSQL que se abandonó.
- **Entorno conda Python 3.9 (`jupyterlab_new.yaml`) como canónico**;
  `requirements.txt` (Python 3.8) queda como legacy.
- **Detección de marca sin APIs de pago:** lista hardcodeada + diccionario por
  primera palabra + autodescubrimiento por slug + validación con Ollama local +
  fallback de búsqueda con `ddgs`. Alternativa descartada: LLM/API en la nube.
- **Exclusión de sesiones con datos corruptos** en cualquier análisis de precio:
  2022-12-31, 2023-03-01 y 2023-06-04 tienen todos los precios a 0 (artefacto de
  scraping). La sesión 2022-08-30 tiene la extracción de marca rota (760 → "-").
- **La caída de oferta de inicios de 2023** (oferta a la mitad, sincronizada en
  todas las ciudades) se trata como **cambio de régimen / artefacto de cadencia
  de scraping**, no como evento de mercado.
- **Hipótesis central rechazada:** el ratio D/O no predice el precio
  (r = −0,291, p = 0,053). Único caso con correlación rezagada significativa:
  **Valencia** (r = 0,819, p = 0,002). Conclusión: los precios de sintes se
  referencian a nivel nacional, no por presión local de oferta/demanda.
- **`Eurorack` se trata como categoría/formato, no como marca.**

## 5. Estado actual

**Funciona:** la cadena completa ETL → EDA → oferta/demanda es ejecutable y
produce el dataset unificado y las 27 figuras. READMEs muy detallados
(esquema, notas de calidad de datos, hallazgos, conclusiones). Diagramas Mermaid
del pipeline añadidos en el último commit (`17dc582`).

**Rama:** `main`, al día con `origin/main`. Existe además una rama local basura
`main.sync-conflict-20260701-172617-TKXRY5E` (generada por Syncthing) que
debería borrarse.

**Cambios sin commitear:**
- Modificado (no en stage): `.gitignore` (añade `.ipynb_checkpoints/`),
  `hispaok/01_from_web_to_csv_togit.ipynb` (reescritura grande: +1.497 / −379).
- En stage (index): `notebooks/01_etl.html`, `notebooks/02_eda.html`,
  `notebooks/03_supply_demand.html` (exports HTML pesados) y
  `notebooks/03_supply_demand-Copy1.ipynb` (copia).
- Sin seguimiento: **`hispaok/hispaok_pipeline.py`** (el script standalone nuevo,
  nunca commiteado), `hispaok/nuevas_marcas.json`,
  `hispaok/01_from_web_to_csv_togit (Copia).ipynb`, y varios archivos
  **sync-conflict de Syncthing**: `README.sync-conflict-*.md`,
  `hispaok/01_from_web_to_csv_togit.sync-conflict-*.ipynb`.

**Riesgos / deuda:**
- Contaminación por Syncthing (archivos y rama de conflicto duplicados).
- `hispaok_pipeline.py` con rutas absolutas no portables y sin CLI/config.
- Dos READMEs divergentes (el versionado, actual; el sync-conflict, obsoleto
  con marco "SQLite / 10.000+ posts").
- No hay `LICENSE` en el árbol pese a que los README lo referencian —
  PENDIENTE DE CONFIRMAR.
- `xgboost`/`shap` fijados pero sin notebook de modelado todavía.

## 6. Próximos pasos (priorizados)

1. **Limpiar el ruido de Syncthing:** borrar los `*.sync-conflict-*`
   (README y notebook), el `01_..._togit (Copia).ipynb` y la rama local
   `main.sync-conflict-*`. Reconciliar el README antiguo (o descartarlo).
2. **Decidir el destino de `hispaok/hispaok_pipeline.py`:** si se commitea,
   parametrizar rutas (argparse / `.env`), quitar constantes `/home/ion/...` y
   aclarar si el scraper canónico es el script o el notebook.
3. **Commitear o descartar** `hispaok/01_from_web_to_csv_togit.ipynb`
   modificado y `hispaok/nuevas_marcas.json`.
4. **Reconsiderar el stage actual:** los `.html` exportados y el notebook
   `-Copy1` son binarios grandes; valorar `.gitignore` en vez de commit.
5. **Streamlit:** construir el dashboard que justifique la dependencia, o
   eliminarla del entorno — PENDIENTE DE CONFIRMAR la intención.
6. **Externalizar la lista `sintes`** de marcas del código a un archivo de datos
   (JSON/CSV) para mantenimiento.
7. **Añadir `LICENSE`** (MIT, según los README).
8. **Modelado** de precio con xgboost/shap (log-transform o estimadores
   robustos por el sesgo a la derecha), tratando el corte de inicios de 2023
   como cambio de régimen.
