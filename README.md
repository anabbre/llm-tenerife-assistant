# Asistente turístico de Tenerife (Entrega Final – LLMs)

Asistente conversacional que responde sobre lugares de interés en Tenerife combinando:

- **RAG (Retrieval-Augmented Generation):** recuperación semántica sobre `data/TENERIFE.pdf`.
- **Diálogo multiturno:** historial breve con recorte de contexto.
- **Function Calling (tools):**
  - `get_weather(fecha, lugar)` → pronóstico *simulado* con validación y Pydantic.
  - `get_sunlight(fecha, lugar)` → amanecer/atardecer reales usando `astral`.
- **Documentación y reproducibilidad:** notebook autocontenido con logs y base vectorial local.
- **Evaluación y métricas:** bloque de pruebas que muestra citaciones, llamadas a tools y latencias.

---

## 1) Estructura del repositorio

```
llm-tenerife-assistant/
├─ data/
│ └─ TENERIFE.pdf
├─ notebooks/
│ └─ llm_tenerife_assistant.ipynb
├─ chroma_db/                              # persistencia del índice (autogenerado)
├─ logs/                                   # ficheros de log (autogenerado)
├─ pictures/                               # capturas para el README
├─ docs/ # documentación                   # Informe_Final.pdf
├─ .env                                    # variables de entorno (no se versiona)
├─ .env.example                            # plantilla de variables de entorno
├─ requirements.txt
├─ README.md
└─ .gitignore
```

> **Nota:** el notebook está dentro de `notebooks/`. Las rutas se calculan desde ahí, por eso se usa `Path.cwd().parent`.

---

## 2) Requisitos

- **Python 3.11** (recomendado).
- Cuenta de OpenAI y API key.

Paquetes principales (ver `requirements.txt`):

```
openai==1.40.0
python-dotenv==1.0.1
chromadb==0.5.4
pypdf==5.0.0
sentence-transformers==3.0.1
pydantic==2.7.0
tiktoken==0.7.0
loguru==0.7.2
rich==13.7.1
jupyter==1.0.0
ipykernel==6.29.0
astral==3.2
```

---

## 3) Configuración del entorno

### 3.1. Crear entorno y kernel

```bash
# En la raíz del proyecto
python3.11 -m venv .venv311
source .venv311/bin/activate   # Windows: .venv311\Scripts\activate

pip install --upgrade pip
pip install -r requirements.txt

# Kernel Jupyter visible en VS Code
python -m ipykernel install --user --name llm-tenerife-311 --display-name "Python (llm-tenerife-311)"
```

### 3.2. Variables de entorno

El proyecto utiliza un archivo `.env` para gestionar las claves y parámetros de configuración.  
En el repositorio se incluye un **`.env.example`** como plantilla.

Copia el archivo de ejemplo y edítalo para añadir tu API key:

```bash
cp .env.example .env
```
Contenido de .env.example:
```
OPENAI_API_KEY=__REEMPLAZA_CON_TU_API_KEY__
OPENAI_MODEL=gpt-4o-mini
OPENAI_TEMPERATURE=0.20
OPENAI_TOP_P=1.0
OPENAI_MAX_TOKENS=600
```
> .env está incluido en .gitignore, por lo que no se versiona y tus claves permanecen seguras.
El modelo por defecto es `gpt-4o-mini`. Se puede cambiar en `.env` o dentro del notebook.

---

## 4) Ejecución del notebook

1. Abrir `notebooks/llm_tenerife_assistant.ipynb` y **seleccionar kernel** `Python (llm-tenerife-311)`.
2. Ejecutar las celdas en orden:
   - **Configuración:** rutas, `.env`, logs, parámetros del modelo (visibles).
   - **Indexación (RAG):** lectura del PDF, *chunking*, generación de embeddings y carga en ChromaDB.
   - **Prueba de recuperación:** consulta simple para verificar que devuelve fragmentos y páginas.
   - **LLM + RAG:** función `ask(...)` y clase `TenerifeChat` para diálogo multiturno.
   - **Tools:** definición de `get_weather` (simulado) y `get_sunlight` (con `astral`), orquestación de function calling.
   - **Orquestación:** función `chat_with_tools(...)`.
   - **Evaluación:** conjunto de preguntas reproducibles.

---

## 5) Diseño y flujo

### 5.1. RAG
- **Chunking por tokens** (≈450 con solape 80) usando `tiktoken` para mejorar el *recall*.
- **Embeddings locales** con `sentence-transformers` (`all-MiniLM-L6-v2`).
- **Vector store** persistente con **ChromaDB**.

### 5.2. Generación
- *System prompt* que obliga a responder **solo con el contexto** recuperado.
- Se incluyen **citaciones** al final con formato `[TENERIFE.pdf · p. X, Y]`.

### 5.3. Diálogo multiturno
- Historial con `deque` y recorte por turnos para no exceder límites de tokens.
- Cada turno vuelve a recuperar contexto para la nueva pregunta.

### 5.4. Function calling (tools)
- **`get_weather(fecha, lugar)`**  
  - Validación de fecha ISO `AAAA-MM-DD` y normalización de lugar contra un diccionario de topónimos (Santa Cruz, La Laguna, Costa Adeje, etc.).  
  - Respuesta simulada: resumen + temperatura min/máx.
- **`get_sunlight(fecha, lugar)`**  
  - Cálculo real de amanecer/atardecer con `astral` usando coordenadas de Tenerife.  
- **Orquestación:**  
  - `chat_with_tools(...)` realiza la doble llamada al modelo (decide → ejecuta tool → redacta).  

---

## 6) Evaluación y métricas

Bloque de pruebas (`notebook`):

```python
tests = [
    "Itinerario sencillo por Santa Cruz para medio día.",
    "¿Qué recomiendan ver en La Orotava?",
    "¿Cómo estará el tiempo el 2025-09-12 en Costa Adeje?",
    "Dime el pronóstico para 2025-09-13 en Santa Cruz de Tenerife",
    "Pronóstico para 2025-09-14 en La Laguna",
    "¿A qué hora amanece y atardece el 2025-09-12 en La Laguna?",
]
```

El script calcula automáticamente:
- % de respuestas con citación,
- nº de llamadas a cada tool,
- latencia media por consulta.

**Ejemplo de métricas:**

```
=== MÉTRICAS ===
{
 'n_tests': 6,
 '%_respuestas_con_citación': 100.0,
 'latencia_media_s': 7.3,
 'calls_get_weather(heurístico)': 3,
 'calls_get_sunlight(heurístico)': 1
}
```

---

## 7) Decisiones y justificación

- **Parámetros del modelo** expuestos: cumplen requisito del enunciado y facilitan reproducibilidad.
- **RAG + citaciones**: asegura respuestas ancladas a la guía.
- **ChromaDB persistente**: evita reindexar cada vez.
- **Pydantic + logs**: bonus de validación robusta y trazabilidad.
- **Métricas**: bonus por evaluación cuantitativa de respuestas.

---

## 11) Autora  

* Ana Belén Ballesteros Redondo  
  🔗 [LinkedIn](https://www.linkedin.com/in/ana-belén-ballesteros-redondo/)  


📑 [Informe Final](docs/Informe_Final.pdf) — documento completo de diseño, resultados y análisis.

