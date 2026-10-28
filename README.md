# Corona Asesor — Agente de revestimientos

> Proyecto de hackathon en equipo (**AgentSprint by ReshapeX**, julio 2026).
> Repositorio original: [DevSolidloper/corona-agent](https://github.com/DevSolidloper/corona-agent).
> Equipo: Santiago Marín Cardona, DevSolidloper y Daniel Garzón.

## Mi contribución

Estos puntos salen de mis commits en el repositorio original (`git log --author="Daniel Garzon"`); el resto del proyecto (catálogo y carga de datos, RAG con Chroma, motor de reglas, cálculos, tests y la base del orquestador) fue trabajo de mis compañeros.

- **Orquestador con Gemini:** adapté el loop de tool use de `src/agent/orchestrator.py` a function calling de Gemini (`google-genai`) y dejé el esquema de herramientas independiente del proveedor (`AGENT_TOOLS`), junto con `src/config.py` y `.env.example`.
- **Contexto institucional bajo demanda:** quité el contexto de la empresa del system prompt y lo expuse como la tool `consultar_contexto_institucional` (por sección), para no gastar tokens en cada turno.
- **UI Streamlit:** rediseñé `app.py` con la paleta de marca de Corona y agregué el tema en `.streamlit/config.toml`.
- **Frontend web y API:** construí la interfaz Next.js/React/TypeScript (`web/`: chat, panel de candidatos, traza visual de tools, descarga de cotización en PDF, modo claro/oscuro) y el backend puente FastAPI (`api_server.py`, con sesión por cookie y CORS) que expone el mismo `CoronaAgent` sin duplicar lógica.
- **Scripts de demo:** `start-demo.ps1` y `check-demo.ps1` para levantar y verificar backend y frontend.
- **Documentación de demo:** guiones de demo y de cobertura de tools en `docs/`, y la actualización de este README con la arquitectura de dos interfaces.

## Cómo evitamos alucinaciones

El LLM no es fuente de datos: solo entiende la conversación, decide qué herramienta llamar y explica el resultado. Cada afirmación con número, producto o compatibilidad sale de una herramienta real (`src/agent/tools.py`, sin mocks):

| Qué afirma el agente | De dónde sale | Código |
|---|---|---|
| Productos, precios, disponibilidad, m² por caja | **DuckDB** en modo solo lectura | `buscar_revestimientos`, `buscar_pegantes`, `buscar_boquillas`, `get_producto` |
| Justificación técnica | **Chroma (RAG)**: fragmentos de fichas técnicas con metadatos (`sku`, `section`, `pdf_id`) para citar; búsqueda semántica y, si falla, búsqueda por palabra clave | `buscar_evidencia` |
| Compatibilidad | **Motor de reglas** (ambiente, humedad, tráfico, pegante, junta, disponibilidad). El estado global es el peor de las reglas y, si falta un dato, devuelve *Requiere revisión* en vez de suponer que cumple | `validar_compatibilidad` |
| Áreas, cajas, pegante, boquilla, presupuesto | **Funciones puras y deterministas**, que devuelven también el desglose del cálculo y marcan `requiere_revision` si falta un dato | `calcular_*` |
| Datos de la empresa (contacto, garantías, links) | Contexto institucional en JSON, consultado por sección | `consultar_contexto_institucional` |

Además, el system prompt (`src/agent/prompts.py`) prohíbe inventar precios, disponibilidad, rendimientos, medidas o compatibilidades, y exige usar una herramienta para cualquier número. Cada llamada a una tool queda registrada en la traza (`trace`), que ambas interfaces muestran para poder auditar de dónde salió cada respuesta.

Agente de IA que ayuda a planear la instalación de **pisos y paredes** y sus
**pegantes y boquillas** con productos de **Corona Colombia**. Entiende el
proyecto del usuario, pide lo que falta, consulta el catálogo real, calcula
cantidades, valida compatibilidad, valida el presupuesto y explica la
recomendación **con evidencia técnica**.

> Proyecto para **AgentSprint by ReshapeX**. Diseñado para llegar a
> **Progress nivel 4**: todas las respuestas *grounded* por herramientas de
> conocimiento (el agente no inventa).

---

## Arquitectura

Un **único agente orquestador** (Google / Gemini, function calling) que
coordina herramientas. El agente vive **una sola vez en Python**
(`src/agent/orchestrator.py`) y tiene **dos interfaces** que lo consumen:

| Capa | Tecnología | Responsabilidad |
|------|-----------|-----------------|
| Razonamiento | **Gemini** (`google-genai`, function calling) | Entiende al usuario, planea, explica |
| Datos exactos | **DuckDB** | Qué productos cumplen filtros, precios, disponibilidad, m²/caja |
| Evidencia | **ChromaDB** | Búsqueda semántica (VoyageAI) sobre fragmentos de fichas técnicas (RAG) |
| Compatibilidad | Motor de reglas | Compatible / Incompatible / Requiere revisión |
| Cantidades | Funciones deterministas | Área, desperdicio, cajas, pegante, boquilla, presupuesto |
| UI (demo original) | **Streamlit** (`app.py`) | Chat + traza de herramientas embebida en el mismo proceso Python |
| UI (demo visual) | **Next.js/React** (`web/`) + **FastAPI** (`api_server.py`) | Chat con diseño de marca, panel de candidatos, trace visual y descarga de PDF, consumiendo el mismo `CoronaAgent` Python por HTTP |

Separación de responsabilidades: **DuckDB** filtra, **Chroma** justifica, el
**motor de reglas** valida, los **cálculos** cuantifican y el **LLM** entiende y
explica. Cada componente es verificable en ejecución (no solo mencionado).

Las dos UI son alternativas, no capas apiladas: **Streamlit** llama al
`CoronaAgent` directamente en el mismo proceso (más simple, ideal para probar
el agente en aislamiento); **Next.js** llama a `api_server.py` por HTTP, que
envuelve ese mismo `CoronaAgent` (interfaz con más cuidado visual). La lógica
de negocio (tools, cálculos, reglas, RAG) no está duplicada: vive una sola vez
en `src/`.

```
corona-agent/
├── app.py                     # UI Streamlit (demo original, llama al agente en el mismo proceso)
├── api_server.py              # FastAPI: expone CoronaAgent por HTTP para el frontend web/
├── run_ingest.py              # Carga JSON→DuckDB y fichas→Chroma
├── start-demo.ps1             # Levanta backend (8000) + frontend web (3000) para la demo
├── check-demo.ps1             # Chequea si ambos servidores de la demo siguen vivos
├── .env.example                # Plantilla de variables (sin secretos)
├── .streamlit/config.toml     # Tema visual de la UI Streamlit
├── data/
│   ├── raw/                   # JSON estructurado (revestimientos, pegantes, boquillas)
│   ├── fichas/                # Fragmentos de fichas técnicas (fichas.json / *.txt)
│   └── processed/             # Artefactos generados (corona.duckdb, chroma/)
├── src/
│   ├── config.py              # Configuración desde .env
│   ├── agent/                 # Orquestador, tools, prompt, estado
│   ├── knowledge/             # DuckDB, Chroma, ingesta
│   ├── rules/                 # Motor de reglas de compatibilidad
│   ├── calc/                  # Cálculos deterministas
│   └── output/                # Generación de cotización en PDF
├── web/                        # Frontend alternativo Next.js/React (ver web/README.md)
├── docs/                       # Guiones de demo y de cobertura de tools
└── tests/                      # Tests de cálculos y reglas
```

---

## Puesta en marcha

### Opción A — Solo Streamlit (más simple)

```bash
# 1) Dependencias
pip install -r requirements.txt

# 2) Variables de entorno
cp .env.example .env
#   edita .env: GOOGLE_API_KEY (obligatoria) y VOYAGE_API_KEY (para RAG semántico)

# 3) La base ya viene construida en data/processed/. Verifícala:
python run_ingest.py          # -> DuckDB: OK products=505 · Chroma: OK fragmentos=3754

# 4) Correr la demo
streamlit run app.py
```

### Opción B — Frontend Next.js (demo visual)

Requiere el backend FastAPI corriendo en `:8000` y el frontend Node en
`:3000`. El script `start-demo.ps1` levanta ambos y espera a que respondan:

```powershell
# Desde la raíz del repo (Windows/PowerShell)
powershell -ExecutionPolicy Bypass -File start-demo.ps1
```

Abre `http://localhost:3000`. Para verificar que ambos servidores siguen
vivos en cualquier momento:

```powershell
powershell -ExecutionPolicy Bypass -File check-demo.ps1
```

O manualmente, en dos terminales separadas:

```bash
# Terminal 1 — backend (raíz del repo)
uvicorn api_server:app --reload --port 8000

# Terminal 2 — frontend (web/)
cd web
npm install
npm run dev
```

Detalles de arquitectura, decisiones y limitaciones del frontend web están en
[`web/README.md`](web/README.md).

Correr los tests (no requieren API key):

```bash
pytest -q
```

---

## Datos: base ya construida

El equipo de datos entrega la base **ya hecha** (no hay que ingestar):

- `data/processed/corona.duckdb` — una tabla **`products`** (505 filas). Los
  atributos técnicos viven en un JSON `specifications`; la capa `duckdb_store.py`
  los **normaliza** (deriva interior/exterior y humedad desde *Áreas de uso*,
  tráfico desde *Uso*, y parsea el formato del nombre).
- `data/processed/chroma/` — colección **`corona_fichas`** (3.754 fragmentos de
  fichas técnicas, embeddings **VoyageAI `voyage-4-large`**). Metadatos por
  fragmento: `sku, category, subcategory, section, pdf_id`.

Categorías reales: `Revestimientos` (subcategorías `Pisos` / `Paredes`),
`Pegantes`, `Boquillas`.

> ⚠️ La base está en `.gitignore` (son ~58 MB de binarios). Consíguela del
> equipo de datos y déjala en `data/processed/`. La demo la lee de ahí.

### Qué trae y qué NO trae el catálogo (y cómo lo maneja el agente)
- **Sí:** categoría, subcategoría, precio (revestimientos casi siempre; pegantes
  y boquillas de forma parcial), disponibilidad, acabado, diseño, áreas de uso,
  presentación (kg) y rango de junta (derivado de la subcategoría).
- **No:** m² por caja, ni rendimiento numérico del pegante/boquilla (viene como
  texto: *"ir a ficha técnica…"*). Cuando falta un dato, el cálculo NO se
  inventa: devuelve **"Requiere revisión"** y apunta a `buscar_evidencia`, que
  recupera el dato de la ficha técnica (Chroma). Con `VOYAGE_API_KEY` la búsqueda
  es semántica; sin ella, cae a un **fallback por palabra clave** que igual
  muestra evidencia.

---

## Cómo evita alucinaciones (grounding)

- El LLM **no** produce números ni precios de memoria: área, cajas, kg de
  pegante/boquilla, costos y presupuesto se calculan con funciones deterministas.
- Productos, precios y disponibilidad salen de **DuckDB**.
- Las justificaciones citan fragmentos reales de fichas desde **Chroma**.
- Si falta un dato, el resultado se marca **"Requiere revisión"** en vez de
  inventarlo.
- La UI muestra la **traza de herramientas** de cada respuesta: se puede auditar
  qué consultó y calculó el agente.

---

## Capacidades adicionales

- **Consultas generales de Corona.** El contexto institucional
  (`data/contexto_agente.json`: empresa, contacto, garantías, financiación,
  tiendas, categorías fuera de catálogo con sus links) se **inyecta en el system
  prompt** (`prompts.py`), no es RAG. Así el agente responde temas generales al
  instante y, ante una categoría fuera del catálogo (sanitarios, griferías,
  pinturas…), **redirige con el link real en vez de decir "no"**.
- **Imágenes de producto.** El normalizador expone `imagen`/`imagenes` (368/505
  productos traen foto). El agente las muestra con markdown `![](url)`; tanto
  Streamlit como la UI Next.js las renderizan.
- **Links de producto.** Cada producto trae `url`; el agente **pregunta** si los
  quieres antes de listarlos.
- **Cotización en PDF.** La herramienta `generar_cotizacion_pdf`
  (`src/output/cotizacion.py`, fpdf2) crea un PDF con el desglose, total y
  validación de presupuesto; la UI muestra un botón de descarga. Los PDF se
  guardan en `data/cotizaciones/`.
- **Estilo sin emojis**, profesional, definido en el system prompt.

## Componentes de agente implementados (Technical Checklist)

1. **LLM / razonamiento** — Gemini (`google-genai`) como orquestador.
2. **Tool use / function calling** — 13 herramientas reales: `calcular_area`,
   `buscar_revestimientos`, `buscar_pegantes`, `buscar_boquillas`,
   `get_producto`, `calcular_cajas`, `calcular_pegante`, `calcular_boquilla`,
   `validar_compatibilidad`, `calcular_presupuesto`,
   `generar_cotizacion_pdf`, `buscar_evidencia` y
   `consultar_contexto_institucional`.
3. **Knowledge tools / RAG** — DuckDB (estructurado) + Chroma (semántico).
4. **Planning / orquestación** — loop multi-paso que encadena las herramientas.
5. **Guardrails + Observabilidad** — política "no inventar" + traza auditable.

---

## Seguridad

- Secretos solo en `.env` / `web/.env.local` (ambos ignorados por git). Nunca
  hardcodeados.
- `.env.example` documenta las variables sin valores reales.
- `api_server.py` restringe CORS a `localhost:3000`/`127.0.0.1:3000` y valida
  que las descargas de cotización (`GET /cotizacion/{filename}`) no salgan de
  `data/cotizaciones/` (protección contra path traversal).
