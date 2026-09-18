# Readme de 1. Pipeline y 2. sistema-decision:


------------------- 1. PIPELINE -------------------

# Pipeline ClinPGx — TFM Farmacogenómica

Este proyecto descarga los datos públicos de ClinPGx, los organiza y prepara
todo lo necesario para revisar cada tabla comparándola con su documentación
oficial. La revisión en sí la hace un LLM (local, con Ollama, o pegando el
prompt a mano en ChatGPT/Claude si no tienes GPU), pero el pipeline que
descarga y prepara los datos es 100% determinista: no llama a ninguna IA.

## Estructura
C:
│   config.yaml
│   prompt_template.md
│   README.md
│   requirements.txt
│   run.py
│
├───scripts
│       00_setup.py
│       01_download.py
│       02_extract.py
│       03_readmes.py
│       04_inventory.py
│       05_prompts.py
│       06_reviews.py
│       07_review_ollama.py
│
├───src
│   └───clinpgx
│           config.py
│           paths.py
│           reviews.py
│           steps.py
│           __init__.py
│
└───work
    │   latest.txt
    └───20260915_114426
        ├───aggregated
        ├───documentation
        ├───downloads
        ├───extracted
        │   ├───chemicals
        │   ├───clinicalVariants
        │   ├───drugLabels
        │   ├───drugs
        │   ├───genes
        │   ├───guidelineAnnotations.json
        │   ├───occurrences
        │   ├───pathways-biopax
        │   ├───pathways-tsv
        │   ├───pathways.json
        │   ├───phenotypes
        │   ├───relationships
        │   ├───summaryAnnotations
        │   ├───variantAnnotations
        │   └───variants
        │
        ├───inventory
        ├───prompts
        └───reviews

## Idea general

Cada archivo que distribuye ClinPGx trae un TSV con los datos y un README.pdf
con la documentación. El problema es que a veces la documentación no coincide
del todo con lo que hay realmente en la tabla (columnas que cambiaron de
nombre, campos que ya no se usan, etc.). Para detectar esto de forma
sistemática en las 18 tablas que hay, generamos un prompt por tabla que
junta el perfil real del TSV (columnas, nulos, ejemplos de valores) con el
texto de la documentación, y se lo pasamos a un LLM para que compare ambas
cosas y señale discrepancias.

## Fases

1. **Descarga** (paso 01) — baja los 15 ZIPs desde la API de ClinPGx.
2. **Extracción** (paso 02) — descomprime cada ZIP en su propia carpeta.
3. **Lectura de READMEs** (paso 03) — convierte cada README.pdf a texto plano.
4. **Inventario** (paso 04) — hace la lista de qué tabla va con qué documentación.
5. **Generación de prompts** (paso 05) — arma un prompt por tabla.
6. **Revisión con LLM** (paso 07, opcional) — el LLM compara documentación y realidad.
7. **Validación de reviews** (paso 06) — valida y consolida en `tables.yaml`.

El paso 07 es el único que toca una IA. El resto son puro Python.

## Instalación

pip install -r requirements.txt

Si se desea ejecutar la revisión automática con Ollama, hay que
instalar Ollama (https://ollama.com/download) y descargar uno de estos
dos modelos:

# Modelo rápido (~30 s por tabla, 13/18 válidas)
ollama pull lfm2.5:8b-a1b-q4_K_M

# Modelo de mayor capacidad (~9,5 min por tabla, reintentos)
ollama pull gemma4:26b-a4b-it-q4_K_M

# Por utlimo para la ultima prueba:
ollama pull qwen3.6:35b-a3b-q4_K_M

-------------

Pipeline completo + revisión automática con Ollama:
python run.py --ollama


Ejecutar solo un paso concreto:
python run.py --solo 05


Ejecutar desde un paso en adelante (por si algo falló a mitad):
python run.py --desde 04


## Qué se genera

Cada vez que se ejecuta el pipeline se crea una carpeta nueva dentro de `work/`
con la fecha y hora, así que nunca pisas una ejecución anterior por accidente:


work/
├── latest.txt              -> apunta al run más reciente
└── 20260115_103000/
    ├── download_manifest.yaml    (MD5 y tamaño de cada ZIP descargado)
    ├── extraction_manifest.yaml  (cuántos ficheros salieron de cada ZIP)
    ├── downloads/                (los ZIP tal cual se descargaron)
    ├── extracted/                (cada ZIP descomprimido en su carpeta)
    ├── documentation/            (los README.pdf convertidos a .txt)
    ├── inventory/
    │   └── table_documentation_map.yaml
    ├── prompts/                  (un prompt.md por tabla)
    ├── reviews/                  (un review.yaml por tabla, rellenado por ti/LLM)
    └── aggregated/
        └── tables.yaml           <- esto es lo que usas en el análisis


## Reejecutar sin repetir trabajo

- Si ejecutas el paso 01 (descarga) dos veces, la segunda vez se salta los
  ZIPs que ya están descargados (comprueba que el fichero existe y no está
  vacío). Igual con el paso 02, 03 y 04.
- Los pasos 05 y 06 sí se recalculan siempre, porque son rápidos y baratos.
- El paso 07 (Ollama) se salta las tablas que ya tienen su `.review.yaml`,
  para poder cortar la ejecución a mitad y continuar otro día sin repetir
  las tablas ya revisadas.

## Notas sobre el modelo de Ollama

El nombre del modelo está escrito directamente en `scripts/07_review_ollama.py` (variable `MODELO`).



------------------- 2. SISTEMA-DECISIÓN -------------------

# tfm-sistema-decision

Sistema de soporte a la decisión clínica basado en LLM local, con y sin RAG.

## Estructura

tfm-sistema-decision/
├── casos_auto.py             Extrae los 10 casos de las tablas del pipeline
├── casos/casos.yaml          Se genera automáticamente
├── prompts/
│   ├── system_con_rag.md     System prompt con reglas de grounding
│   └── system_sin_rag.md     System prompt sin grounding
├── src/cds/
│   ├── verificador.py        3 checks: YAML válido, evidencia, no alucina
│   ├── runner.py             Ejecuta la matriz con patrón SKIP
│   └── reporte.py            Genera 3 tablas CSV + 3 gráficos PNG
├── resultados/                Un .json por ejecución
├── reportes/                  Salida final
└── run_experimento.py         Lanzador único

## Objetivo

Evaluar el efecto del RAG (Retrieval-Augmented Generation) en la precisión
de un LLM local al responder consultas farmacogenómicas. Se comparan
3 modelos x 3 temperaturas x 2 modos RAG x 10 casos clínicos.

## Ground truth

Las respuestas correctas provienen de la base de datos ClinPGx/PharmGKB,
concretamente del fichero `var_drug_ann.tsv`. Cada caso incluye nivel de
evidencia (1A-4), significancia clínica, recomendación y PMID.

## Requisitos

- Python 3.11+
- Ollama con los 3 modelos descargados:
  - qwen3.5:2b-q4_K_M
  - lfm2.5:8b-a1b-q4_K_M
  - gemma4:26b-a4b-it-q4_K_M
- El pipeline `tfm-pipeline` debe existir como carpeta hermana y estar ejecutado.

## Instalación

pip install -r requirements.txt


## Uso

Experimento completo (180 ejecuciones):
python run_experimento.py

Prueba rápida con un solo modelo:
python run_experimento.py --solo-modelo lfm2.5:8b-a1b-q4_K_M

Solo generar el reporte:
python run_experimento.py --reporte

## Salida

En `reportes/`:

- `tabla_temp0.1.csv`, `tabla_temp0.35.csv`, `tabla_temp0.9.csv`
- `tabla_global.csv`
- `barras.png`, `heatmap.png`, `lineas.png`

## Verificador

Cada respuesta se evalúa con 3 checks:

- **C1**: la respuesta es YAML válido con los campos obligatorios.
- **C3**: el nivel de evidencia coincide con el ground truth.
- **C6**: no alucina (el PMID citado existe en el contexto).

Veredictos: PASS (los 3 checks OK), FAIL (alguno falla), ABSTENCION
(el modelo reconoce no tener información suficiente).

## Nota

Este proyecto no forma parte del pipeline ETL. Solamente utiliza el resultado del pipeline, ubicado en una carpeta separada.
