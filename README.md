# 🎯 Claude Agent SDK vs Google ADK — Guía de preparación para challenge

> Guía intensiva (2-3 días) para ir de cero a competente en ambos SDKs. Asume base de Claude Code + skills + MCPs básicos.

---

## 🚀 TL;DR — 30 segundos de contexto

| Aspecto | **Claude Agent SDK** | **Google ADK** |
|---|---|---|
| **Qué es** | Runtime local que empaqueta el loop de Claude Code como librería | Framework de orquestación multi-agente enterprise |
| **Paradigma** | "Dale al agente una computadora" (shell/fs/browser) | "Componé un equipo de agentes modulares" |
| **Lenguajes** | Python, TypeScript | Python, TypeScript, Go, Java |
| **Modelos** | Claude (Opus/Sonnet/Haiku) directo, Vertex, Azure Foundry | Agnóstico: Gemini nativo, Claude, Llama, OpenAI vía LiteLLM |
| **Estado/Memoria** | `SessionStore` adapter (archivo/redis/custom) | `Session` + `State` gestionados por `Runner` |
| **Deploy** | Proceso local / contenedor propio | Agent Engine (Vertex AI), Cloud Run, GKE, self-hosted |
| **Protocolo interop** | MCP (Model Context Protocol) | MCP + A2A (Agent2Agent) |
| **UI dev-time** | Claude Code CLI / tu app | `adk web` (chat UI en localhost:8000) |
| **Optimiza para** | Calidad de razonamiento, autonomía | Costo, throughput, orquestación explícita |

**Regla mental rápida:**
- ¿Necesitás un agente que *actúe* con acceso profundo al SO? → **Claude SDK**
- ¿Necesitás coordinar *varios agentes* con flujos bien definidos? → **Google ADK**
- ¿Challenge pide "construí un sistema multi-agente"? → probablemente **ADK** (aunque Claude SDK también tiene subagents)

---

## 🤖 Parte 1 · Claude Agent SDK a fondo

### 1.1 · Conceptos clave

| Concepto | Qué es | Cuándo lo usás |
|---|---|---|
| **Agent loop** | Ciclo automático: Claude piensa → llama tool → recibe result → vuelve a pensar. El SDK maneja esto por vos. | Siempre (base del SDK) |
| **`query()`** | Función one-shot, async generator. Input → stream de mensajes. | Tareas simples, sin estado entre llamadas |
| **`ClaudeSDKClient`** | Cliente con estado persistente, multi-turno, configurable. | Conversaciones, apps interactivas |
| **Tools (built-in)** | `Read`, `Write`, `Edit`, `Bash`, `Grep`, `Glob`, `WebSearch`, `WebFetch`, `Task`, etc. | Por default el agente los tiene todos — los restringís con `allowed_tools` |
| **Custom tools** | Funciones Python decoradas con `@tool`, empaquetadas como *in-process MCP server*. | Cuando necesitás lógica de negocio específica |
| **MCP servers** | Servidores externos (stdio/HTTP) que exponen tools. Ej: Slack, GitHub, DB. | Integrar con sistemas existentes |
| **Subagents** | Agentes hijos con contexto aislado, tools restringidos, ejecutables en paralelo. | Dividir trabajo complejo, isolar contextos largos |
| **Skills** | Capacidades declarativas empaquetadas (carpeta con `SKILL.md` + archivos). Cargadas bajo demanda. | Lógica reutilizable que no querés en el system prompt |
| **Hooks** | Callbacks en el loop: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`. | Guardrails, logging, policy enforcement |
| **Permission modes** | `default`, `acceptEdits`, `plan`, `bypassPermissions`. | Controlar qué puede hacer el agente sin preguntar |
| **`SessionStore`** | Protocolo para persistir sesiones (5 métodos: append/load/list/delete/list_subkeys). | Apps multi-usuario o que sobreviven reinicios |

### 1.2 · Setup

```bash
# 1. Python 3.11+ (recomendado 3.12)
python --version

# 2. Virtualenv
python -m venv .venv
source .venv/bin/activate  # Windows bash: source .venv/Scripts/activate

# 3. Instalar SDK
pip install claude-agent-sdk

# 4. API key
export ANTHROPIC_API_KEY="sk-ant-..."

# Alternativas:
# export CLAUDE_CODE_USE_VERTEX=1        # via Google Vertex AI
# export CLAUDE_CODE_USE_FOUNDRY=1       # via Azure AI Foundry
```

**Requisitos ocultos:** el SDK lanza un subproceso Node.js (el CLI de Claude Code). Asegurate de tener Node 18+ instalado y en `PATH`. Si no, instalalo desde nodejs.org o con `nvm`.

### 1.3 · Ejemplo 1 — Hello world con `query()`

```python
# hello.py
import asyncio
from claude_agent_sdk import query, AssistantMessage, TextBlock

async def main():
    async for message in query(prompt="Decí hola en 3 idiomas"):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    print(block.text)

asyncio.run(main())
```

**Qué aprender acá:**
- `query()` es **async generator** → siempre `async for`.
- Los mensajes llegan *en streaming*: `AssistantMessage`, `UserMessage`, `ToolUseBlock`, `ToolResultBlock`.
- Sin estado: cada `query()` es una conversación nueva.

### 1.4 · Ejemplo 2 — Agente con tools + permission mode

```python
# fs_agent.py
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Grep", "Glob"],  # read-only: seguro
        permission_mode="acceptEdits",            # no pregunta por cada acción
        cwd="./my-project",                       # directorio de trabajo
        system_prompt="Sos un asistente que audita código Python.",
    )

    prompt = "Buscá todos los TODO en archivos .py y listá los 5 más críticos."
    async for msg in query(prompt=prompt, options=options):
        print(msg)

asyncio.run(main())
```

**Qué aprender:**
- `allowed_tools`: lista blanca. Sin esto, el agente puede correr `Bash` y borrar cosas.
- `permission_mode`: `"default"` pregunta, `"acceptEdits"` aprueba edits, `"plan"` solo planea, `"bypassPermissions"` no pregunta nada (⚠️ peligroso).
- `cwd`: scope del filesystem. Vital para seguridad.

### 1.5 · Ejemplo 3 — Custom tool con `@tool`

```python
# custom_tool.py
import asyncio
from claude_agent_sdk import (
    tool, create_sdk_mcp_server, ClaudeAgentOptions, ClaudeSDKClient
)

@tool(
    "get_stock_price",
    "Devuelve el precio de una acción dado su ticker",
    {"ticker": str}
)
async def get_stock_price(args):
    ticker = args["ticker"].upper()
    # Acá iría una llamada real a API (yfinance, polygon, etc.)
    fake_prices = {"AAPL": 192.5, "GOOG": 178.3, "MSFT": 420.1}
    price = fake_prices.get(ticker, 0.0)
    return {
        "content": [
            {"type": "text", "text": f"{ticker}: USD {price}"}
        ]
    }

async def main():
    server = create_sdk_mcp_server(
        name="finance-tools",
        version="1.0.0",
        tools=[get_stock_price],
    )

    options = ClaudeAgentOptions(
        mcp_servers={"finance": server},
        allowed_tools=["mcp__finance__get_stock_price"],  # naming: mcp__<server>__<tool>
    )

    async with ClaudeSDKClient(options=options) as client:
        await client.query("¿Cuánto está AAPL y GOOG? Comparalos.")
        async for msg in client.receive_response():
            print(msg)

asyncio.run(main())
```

**Qué aprender:**
- `@tool(name, description, schema)` — el schema es un dict `{"param": type}` y Claude lo usa para saber qué pasarte.
- `create_sdk_mcp_server()` corre *in-process* (sin IPC, sin subprocess). Rápido y fácil de debuggear.
- Naming convention: `mcp__<server_name>__<tool_name>`. **Si olvidás esto, el tool no se activa.**
- Los tools son **async**: si llamás APIs, usá `httpx.AsyncClient`, no `requests`.

### 1.6 · Ejemplo 4 — Subagent con contexto aislado

```python
# subagent_demo.py
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

async def main():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Grep", "Task"],  # Task es el tool para spawnear subagents
        system_prompt=(
            "Sos un orquestador. Para tareas de research técnico, "
            "delegás a subagents especializados usando el tool Task."
        ),
    )

    async with ClaudeSDKClient(options=options) as client:
        await client.query(
            "Investigá en paralelo: "
            "(1) patrones de autenticación OAuth2 en este repo, "
            "(2) cómo se manejan sesiones en el backend. "
            "Spawnea un subagent para cada uno y sintetizá al final."
        )
        async for msg in client.receive_response():
            print(msg)

asyncio.run(main())
```

**Qué aprender:**
- El agente padre decide *cuándo* spawnear subagents (no lo hacés vos directamente).
- Cada subagent tiene su propio context window → ideal para tareas con mucho output (buscar en 100 archivos) sin contaminar el contexto principal.
- Solo el **mensaje final** del subagent vuelve al padre → ahorra tokens.

### 1.7 · Ejemplo 5 — Hook para guardrail

```python
# hook_demo.py
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, HookMatcher

async def block_rm_rf(input_data, tool_use_id, context):
    """Bloquea cualquier comando peligroso antes de ejecutarse."""
    tool = input_data.get("tool_name")
    args = input_data.get("tool_input", {})
    if tool == "Bash" and "rm -rf" in args.get("command", ""):
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "rm -rf bloqueado por policy",
            }
        }
    return {}

async def main():
    options = ClaudeAgentOptions(
        allowed_tools=["Bash", "Read"],
        hooks={
            "PreToolUse": [HookMatcher(matcher="Bash", hooks=[block_rm_rf])]
        },
    )
    async with ClaudeSDKClient(options=options) as client:
        await client.query("Limpiá los archivos temporales con rm -rf /tmp/*")
        async for msg in client.receive_response():
            print(msg)

asyncio.run(main())
```

**Qué aprender:**
- Hooks disponibles: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `SubagentStop`.
- Son la base de los **guardrails de producción**: logging, auditoría, policy enforcement, redacción de PII.
- Devolver `permissionDecision: "deny"` **antes** de que el tool corra = bloqueo.

### 1.8 · Flujo mental del Claude SDK (diagrama textual)

```
Usuario → prompt
    ↓
[ClaudeSDKClient.query()]
    ↓
Claude decide: ¿qué tool usar?
    ↓
[PreToolUse hook] ← podés bloquear acá
    ↓
Ejecutar tool (built-in o MCP)
    ↓
[PostToolUse hook] ← podés modificar resultado
    ↓
Claude recibe result → decide: ¿otro tool? ¿respuesta final?
    ↓
(loop hasta que no haya más tool calls)
    ↓
AssistantMessage final → Usuario
```

---

## 🧩 Parte 2 · Google ADK a fondo

### 2.1 · Conceptos clave

| Concepto | Qué es | Cuándo lo usás |
|---|---|---|
| **`Agent` / `LlmAgent`** | Agente base con LLM, instruction, tools, sub_agents. | Unidad fundamental de ADK |
| **`SequentialAgent`** | Workflow agent: corre sub-agentes en orden, pasa output al siguiente. | Pipelines (parse → extract → summarize) |
| **`ParallelAgent`** | Workflow agent: corre sub-agentes concurrentemente. | Fan-out a APIs independientes |
| **`LoopAgent`** | Workflow agent: itera hasta que un criterio se cumple. | Generator-critic, refinamiento iterativo |
| **`BaseAgent`** | Clase base para lógica custom (no LLM). | Transformaciones deterministas en el flujo |
| **Tools** | Funciones Python, `FunctionTool`, `AgentTool`, `MCPTool`, built-ins (Google Search, Code Exec). | Dar capacidades al agente |
| **`sub_agents`** | Lista de agentes que un padre puede invocar (delegación LLM-driven). | Coordinator/Router pattern |
| **`Session`** | Contenedor de estado + historial de eventos. Persistente. | Toda app con >1 turno |
| **`State`** | Dict compartido entre agentes dentro de una sesión. Prefijos: `user:`, `app:`, `temp:`. | Pasar datos entre agentes |
| **`Runner`** | Objeto que ejecuta la sesión, maneja eventos, servicios. | Entrypoint de la app |
| **`Event`** | Item del stream: output de agente, tool call, tool result. | Iterás sobre ellos |
| **`outputKey`** | Guarda el output del agente en `state[outputKey]`. | Pasar resultados entre agentes |
| **A2A (Agent2Agent)** | Protocolo abierto para comunicación entre agentes de distintos frameworks. | Interop con LangGraph, CrewAI, etc. |
| **Agent Engine** | Runtime managed en Vertex AI para deploy de agentes. | Producción en GCP |
| **`adk web` / `adk run`** | CLI que abre una UI chat local o REPL. | Development / testing |

### 2.2 · Setup

```bash
# 1. Python 3.11+
python --version

# 2. Virtualenv
python -m venv .venv
source .venv/bin/activate

# 3. Instalar ADK
pip install google-adk

# 4. Credenciales — Gemini API (más simple)
export GOOGLE_API_KEY="..."
# O vía Vertex AI:
# gcloud auth application-default login
# export GOOGLE_GENAI_USE_VERTEXAI=True
# export GOOGLE_CLOUD_PROJECT="tu-proyecto"
# export GOOGLE_CLOUD_LOCATION="us-central1"
```

**Estructura de proyecto que espera ADK:**

```
my_agent_project/
├── agent/
│   ├── __init__.py         # from . import agent
│   ├── agent.py            # define root_agent
│   └── .env                # variables de entorno
```

**Crítico:** `adk web` y `adk run` buscan un atributo llamado `root_agent` en `agent.py`. Si no está, fallan silenciosamente.

### 2.3 · Ejemplo 1 — Agent simple con tool

```python
# agent/agent.py
from datetime import datetime
from zoneinfo import ZoneInfo
from google.adk.agents import Agent

def get_current_time(city: str) -> dict:
    """Devuelve la hora actual en una ciudad.
    Args:
        city: Nombre de la ciudad (ej: 'Santiago', 'Tokyo').
    Returns:
        dict con status y hora, o error.
    """
    timezones = {
        "santiago": "America/Santiago",
        "tokyo": "Asia/Tokyo",
        "new york": "America/New_York",
    }
    tz_name = timezones.get(city.lower())
    if not tz_name:
        return {"status": "error", "error_message": f"No conozco {city}"}
    now = datetime.now(ZoneInfo(tz_name))
    return {"status": "success", "time": now.strftime("%Y-%m-%d %H:%M:%S %Z")}

root_agent = Agent(
    name="time_agent",
    model="gemini-2.0-flash",
    description="Agente que responde qué hora es en distintas ciudades.",
    instruction=(
        "Sos un asistente de horarios. Usá get_current_time para "
        "responder. Si no conocés la ciudad, decilo claramente."
    ),
    tools=[get_current_time],
)
```

```bash
# Correr interactivo
adk run agent

# UI web
adk web
# Abrí http://localhost:8000
```

**Qué aprender:**
- La **docstring** de la función es el schema que ve el LLM. Escribila bien: descripción + `Args:` + `Returns:`.
- El return debe ser **serializable** (dict/list/str/int). Convención ADK: `{"status": "success|error", ...}`.
- `name` del agente sin espacios (convención snake_case).
- Modelos comunes: `gemini-2.0-flash`, `gemini-2.5-pro`, `gemini-flash-latest`.

### 2.4 · Ejemplo 2 — SequentialAgent (pipeline)

```python
# agent/agent.py
from google.adk.agents import LlmAgent, SequentialAgent

parser = LlmAgent(
    name="parser",
    model="gemini-2.0-flash",
    instruction="Extraé el texto plano del documento de entrada. Ignorá formato.",
    output_key="parsed_text",  # guarda output en state["parsed_text"]
)

extractor = LlmAgent(
    name="extractor",
    model="gemini-2.0-flash",
    instruction=(
        "Del texto en state['parsed_text'], extraé entidades: "
        "personas, lugares, fechas. Devolvé JSON."
    ),
    output_key="entities",
)

summarizer = LlmAgent(
    name="summarizer",
    model="gemini-2.0-flash",
    instruction=(
        "Con state['parsed_text'] y state['entities'], "
        "escribí un resumen de 3 líneas."
    ),
    output_key="summary",
)

root_agent = SequentialAgent(
    name="document_pipeline",
    sub_agents=[parser, extractor, summarizer],
)
```

**Qué aprender:**
- `output_key` guarda el output del agente en `state`. **Así se pasan datos entre agentes.**
- En el `instruction` de un agente posterior, referenciás `state['clave']` (ADK inyecta el valor).
- El orden en `sub_agents=[...]` **es el orden de ejecución**.

### 2.5 · Ejemplo 3 — ParallelAgent + synthesizer

```python
from google.adk.agents import LlmAgent, ParallelAgent, SequentialAgent

profiler = LlmAgent(
    name="company_profiler",
    model="gemini-2.0-flash",
    instruction="Perfilá la empresa del input: industria, tamaño, HQ.",
    output_key="profile",
)

news_finder = LlmAgent(
    name="news_finder",
    model="gemini-2.0-flash",
    instruction="Buscá 3 noticias recientes relevantes de la empresa.",
    output_key="news",
)

financial = LlmAgent(
    name="financial_analyst",
    model="gemini-2.0-flash",
    instruction="Análisis financiero somero de la empresa.",
    output_key="financials",
)

research_fanout = ParallelAgent(
    name="research_parallel",
    sub_agents=[profiler, news_finder, financial],
)

reporter = LlmAgent(
    name="reporter",
    model="gemini-2.0-flash",
    instruction=(
        "Sintetizá un reporte ejecutivo usando: "
        "state['profile'], state['news'], state['financials']."
    ),
    output_key="report",
)

root_agent = SequentialAgent(
    name="company_detective",
    sub_agents=[research_fanout, reporter],
)
```

**Qué aprender:**
- `ParallelAgent` corre todos sus sub-agentes **concurrentemente** (async).
- Para sintetizar, envolvés el `ParallelAgent` en un `SequentialAgent` con el synthesizer al final.
- Cada rama paralela debe tener un **`output_key` distinto**, sino se pisan.

### 2.6 · Ejemplo 4 — Coordinator con delegación (LLM-driven)

```python
from google.adk.agents import LlmAgent

brainstormer = LlmAgent(
    name="brainstormer",
    model="gemini-2.0-flash",
    description="Experto en ideas creativas para viajes.",
    instruction="Generá ideas de destinos según intereses del usuario.",
)

planner = LlmAgent(
    name="attraction_planner",
    model="gemini-2.0-flash",
    description="Experto en armar itinerarios con atracciones concretas.",
    instruction="Dado un destino, armá un plan día por día.",
)

root_agent = LlmAgent(
    name="travel_coordinator",
    model="gemini-2.0-flash",
    description="Coordinador de viajes. Delega a especialistas.",
    instruction=(
        "Si el usuario pide ideas, transferí a brainstormer. "
        "Si ya tiene destino y quiere itinerario, transferí a attraction_planner."
    ),
    sub_agents=[brainstormer, planner],
)
```

**Qué aprender:**
- Cuando un `LlmAgent` tiene `sub_agents`, el LLM puede **transferir la conversación** a uno de ellos.
- El `description` de cada sub-agente es lo que el coordinator usa para decidir.
- Es delegación **LLM-driven** (distinto de `SequentialAgent` que es determinista).

### 2.7 · Ejemplo 5 — Runner + Session (programático, sin `adk web`)

```python
import asyncio
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

async def main():
    from agent.agent import root_agent  # importá tu agente

    session_service = InMemorySessionService()
    session = await session_service.create_session(
        app_name="my_app", user_id="user_123"
    )
    runner = Runner(
        agent=root_agent,
        app_name="my_app",
        session_service=session_service,
    )

    user_message = types.Content(
        role="user",
        parts=[types.Part(text="Quiero viajar a Japón en abril.")],
    )

    async for event in runner.run_async(
        user_id="user_123",
        session_id=session.id,
        new_message=user_message,
    ):
        if event.is_final_response():
            print("FINAL:", event.content.parts[0].text)
        elif event.get_function_calls():
            for fc in event.get_function_calls():
                print(f"TOOL: {fc.name}({fc.args})")

asyncio.run(main())
```

**Qué aprender:**
- `Runner` es el motor de ejecución. **No llamás al agente directo** — siempre vía Runner.
- `SessionService`: `InMemorySessionService` para dev, `DatabaseSessionService` o `VertexAiSessionService` para prod.
- `runner.run_async()` devuelve un **stream de Events**. Filtrás con `is_final_response()`, `get_function_calls()`, etc.

### 2.8 · Ejemplo 6 — Deploy a Vertex AI Agent Engine (cheatsheet)

```python
# deploy.py
import vertexai
from vertexai.agent_engines import AgentEngine
from agent.agent import root_agent

vertexai.init(
    project="mi-proyecto-gcp",
    location="us-central1",
    staging_bucket="gs://mi-bucket-staging",
)

remote_app = AgentEngine.create(
    agent_engine=root_agent,
    requirements=[
        "google-adk>=1.0.0",
        "google-cloud-aiplatform[agent_engines]",
    ],
    extra_packages=["./agent"],
)

print("Deployed:", remote_app.resource_name)
```

**Qué aprender:**
- Empaquetás el agente + dependencias → Agent Engine lo corre como servicio managed.
- Te da: auth, IAM, Cloud Trace, autoscaling.
- Costos: pagás por invocación + compute. No hay free tier significativo → no dejes corriendo cosas de prueba.

### 2.9 · Flujo mental del Google ADK

```
Usuario → Runner.run_async(message)
    ↓
[Session + State cargan contexto]
    ↓
root_agent decide:
  ├─ Si es LlmAgent: LLM piensa → (tool call | transfer_to_agent | respuesta)
  ├─ Si es SequentialAgent: corre sub_agents en orden, State fluye
  ├─ Si es ParallelAgent: corre sub_agents concurrentes, State merge
  └─ Si es LoopAgent: itera hasta stop condition
    ↓
Cada step emite Events → stream al usuario
    ↓
State persiste en Session → disponible para siguiente turno
```

---

## ⚖️ Parte 3 · Diferencias conceptuales clave

### 3.1 · Paradigma

- **Claude SDK**: el agente es **una entidad autónoma** con acceso a un entorno. Le das un objetivo y él decide cómo lograrlo. Más "agéntico" en el sentido tradicional.
- **ADK**: construís un **grafo de agentes** con flujo explícito. Mayor control, menos autonomía. El LLM decide *dentro* de un agente, pero vos decidís el orquesta.

### 3.2 · Estado

- **Claude SDK**: el estado vive en el **context window** del agente + `SessionStore` opcional. La memoria "larga" la implementás vos (vía tools, archivos, DB).
- **ADK**: `State` explícito en cada `Session`. Acceso con prefijos: `user:pref` (permanente por usuario), `app:config` (global), `temp:calc` (solo este turno).

### 3.3 · Comunicación entre agentes

- **Claude SDK**: subagents son **hijos** del padre. Comunicación padre→hijo con prompt, hijo→padre con mensaje final. **No hay comunicación hermano-hermano.**
- **ADK**: 3 mecanismos:
  1. Compartir `State` (todos leen/escriben el mismo dict).
  2. `AgentTool` (un agente invoca a otro como si fuera un tool).
  3. `transfer_to_agent` (handoff completo: el otro agente toma la conversación).

### 3.4 · Modelos

- **Claude SDK**: Claude only (Opus/Sonnet/Haiku).
- **ADK**: multimodelo vía LiteLLM. Podés mezclar Gemini (root) + Claude (subagente crítico) + Llama (subagente barato) en el mismo sistema.

### 3.5 · Deployment

- **Claude SDK**: vos empaquetás (Docker, Lambda, lo que sea). Subprocess de Node.js requerido.
- **ADK**: `adk deploy` → Agent Engine / Cloud Run con 1 comando. Python puro.

### 3.6 · Observabilidad

- **Claude SDK**: OpenTelemetry trace propagation (W3C). Logs vía hooks.
- **ADK**: Cloud Trace integrado + Arize Phoenix para eval. Events ya estructurados.

### 3.7 · Cuándo uno pisa al otro

- **ADK puede usar Claude como modelo** vía integración Anthropic → si el challenge pide Gemini Y Claude en el mismo sistema, ADK es la ruta.
- **Claude SDK puede consumir MCP servers** → si ya tenés tools MCP, ambos SDKs los reutilizan.
- **A2A protocol** te permite que un agente ADK hable con un agente Claude SDK si envolvés Claude SDK en un A2A server (más work).

---

## 📅 Parte 4 · Path de estudio (2-3 días)

### Día 1 — Claude Agent SDK (4-5 h)

| Bloque | Tiempo | Tarea |
|---|---|---|
| 1.1 | 30 min | Leer [Agent SDK overview](https://platform.claude.com/docs/en/agent-sdk/overview) + [quickstart](https://platform.claude.com/docs/en/agent-sdk/quickstart). |
| 1.2 | 30 min | Instalar, correr **Ejemplo 1 (hello world)**. Verificar que streamea. |
| 1.3 | 45 min | **Ejemplo 2**: tools + permission modes. Probar bloqueando un tool y ver cómo reacciona. |
| 1.4 | 1 h | **Ejemplo 3**: custom tool con `@tool`. Hacer un tool que llame una API real (ej: https://api.coingecko.com/api/v3/simple/price). |
| 1.5 | 1 h | **Ejemplo 4**: subagents. Pedirle al padre que use `Task` para dividir una tarea de búsqueda. |
| 1.6 | 45 min | **Ejemplo 5**: hook `PreToolUse` que bloquee comandos `Bash` que contengan palabras peligrosas. |
| 1.7 | 30 min | Leer sobre [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) (formato `SKILL.md`). |

**Checkpoint día 1:** ¿Podés explicar qué es el agent loop, cómo se declara un custom tool, y cuándo usar subagents? Si no → repasá.

### Día 2 — Google ADK (4-5 h)

| Bloque | Tiempo | Tarea |
|---|---|---|
| 2.1 | 30 min | Leer [Technical Overview](https://google.github.io/adk-docs/get-started/about/) + [quickstart](https://google.github.io/adk-docs/get-started/quickstart/). |
| 2.2 | 30 min | Crear estructura `agent/agent.py`, correr **Ejemplo 1** con `adk web`. Ver la UI chat. |
| 2.3 | 45 min | **Ejemplo 2**: `SequentialAgent` pipeline. Meter 3 agentes encadenados, observar cómo fluye `state`. |
| 2.4 | 1 h | **Ejemplo 3**: `ParallelAgent` + synthesizer. Ver concurrencia en los logs. |
| 2.5 | 1 h | **Ejemplo 4**: Coordinator con `sub_agents`. Probar que el LLM delegue bien. |
| 2.6 | 45 min | **Ejemplo 5**: `Runner` programático. Integrar en un script normal (sin `adk web`). |
| 2.7 | 30 min | Leer [Multi-agent patterns](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/) (los 8 patrones). |

**Checkpoint día 2:** ¿Podés diferenciar `LlmAgent` vs `SequentialAgent` vs `ParallelAgent`? ¿Sabés qué hace `output_key`? Si no → repasá.

### Día 3 — Integración + challenge prep (3-4 h)

| Bloque | Tiempo | Tarea |
|---|---|---|
| 3.1 | 45 min | Leer sobre [A2A protocol](https://google.github.io/adk-docs/a2a/) — conceptos, no deep dive. |
| 3.2 | 45 min | Leer [Claude como model en ADK](https://google.github.io/adk-docs/agents/models/anthropic/) — cómo mezclar. |
| 3.3 | 1 h | **Ejercicio combinado** (ver Parte 6, ejercicio 5). |
| 3.4 | 30 min | Revisar [adk-samples](https://github.com/google/adk-samples/tree/main/python/agents) — agarrar 1-2 ideas. |
| 3.5 | 30 min | Revisar [claude-agent-sdk-demos](https://github.com/anthropics/claude-agent-sdk-demos). |
| 3.6 | 30 min | Releer **Parte 8 (trampas comunes)** de este doc. |

**Checkpoint día 3:** Podés, sin mirar, escribir de memoria un agente simple en ambos SDKs. Identificás qué herramienta usar según el enunciado del challenge.

---

## ✅ Parte 5 · Checklist de conceptos (challenge-ready)

Marcá lo que ya sabés explicar **sin ayuda**:

### Claude Agent SDK

- [ ] Diferencia entre `query()` y `ClaudeSDKClient`.
- [ ] Qué es el **agent loop** y cómo lo automatiza el SDK.
- [ ] Cómo declarar un **custom tool** con `@tool`.
- [ ] Naming convention de tools MCP (`mcp__<server>__<tool>`).
- [ ] Los 4 **permission modes** y cuándo usar cada uno.
- [ ] Cómo **restringir tools** con `allowed_tools` y por qué es crítico.
- [ ] Qué es un **subagent** y cómo se spawnea (`Task` tool).
- [ ] Hooks disponibles (`PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`) y casos de uso.
- [ ] Qué es una **Skill** y cuándo usarla vs un tool.
- [ ] Cómo conectar un **MCP server externo** (stdio/HTTP).
- [ ] `SessionStore`: para qué sirve y cuándo lo implementás.

### Google ADK

- [ ] Diferencia entre `LlmAgent`, `SequentialAgent`, `ParallelAgent`, `LoopAgent`.
- [ ] Qué hace `output_key` y cómo fluye `state`.
- [ ] Cómo declarar un **tool** (función Python con docstring correcto).
- [ ] Convención de return dict (`{"status": ..., ...}`).
- [ ] Estructura de proyecto (`agent/agent.py` con `root_agent`).
- [ ] Uso de `adk run`, `adk web`, `adk deploy`.
- [ ] Qué es `Runner` y cómo se usa programáticamente.
- [ ] `Session` vs `State`; prefijos `user:`, `app:`, `temp:`.
- [ ] Delegación LLM-driven (`sub_agents` en `LlmAgent`) vs determinista (workflow agents).
- [ ] Protocolo **A2A** — qué problema resuelve.
- [ ] Qué es **Agent Engine** y cómo se deploya.
- [ ] Usar **modelos no-Google** (Claude, Llama) vía LiteLLM.

### Cross-cutting

- [ ] Cuándo elegir uno vs el otro dado un enunciado.
- [ ] MCP como denominador común entre ambos.
- [ ] Diferencia conceptual: autonomía vs orquestación.
- [ ] Seguridad: guardrails en Claude SDK (hooks) vs ADK (callbacks + IAM).

---

## 🏋️ Parte 6 · 5 ejercicios progresivos

### Ejercicio 1 — 🔹 Claude SDK: tool para clima

**Objetivo:** escribir un custom tool que consulte una API y lo use el agente.

**Pasos:**
1. Creá `weather_agent.py`.
2. Declará un tool `get_weather(city: str)` que pegue a https://wttr.in/{city}?format=j1 (API sin key).
3. Configurá `ClaudeSDKClient` con ese MCP server.
4. Preguntale: "¿Cuál es el clima en Santiago y en Tokyo? Compará."

**Éxito:** el agente llama el tool 2 veces (una por ciudad) y compara en respuesta final.

**Extensión:** agregá un hook `PostToolUse` que logee a un archivo todas las tool calls.

---

### Ejercicio 2 — 🔹 Claude SDK: subagents paralelos

**Objetivo:** orquestador con 3 subagents concurrentes.

**Pasos:**
1. Agente padre: "Investigador de inversiones".
2. Permitile usar `Task`. En el system_prompt instruí que **siempre** use subagents en paralelo para:
   - buscar noticias recientes
   - analizar balance
   - revisar social sentiment
3. Que el padre sintetice en una recomendación buy/hold/sell.

**Input:** "Analizá Apple como inversión."

**Éxito:** en los logs ves 3 subagents ejecutándose, cada uno con context propio, y una respuesta final sintetizada.

---

### Ejercicio 3 — 🔸 ADK: pipeline de procesamiento de tickets

**Objetivo:** `SequentialAgent` con 3 etapas.

**Pasos:**
1. `classifier`: recibe texto de ticket de soporte → clasifica en `bug | feature | question`. `output_key="category"`.
2. `prioritizer`: recibe `state['category']` + texto original → asigna `low | medium | high`. `output_key="priority"`.
3. `responder`: redacta respuesta inicial según category + priority. `output_key="response"`.
4. Envolvé en `SequentialAgent`.
5. Corré con `adk web`.

**Input:** "La app se cierra cada vez que abro el perfil. Pierdo los cambios."

**Éxito:** ves los 3 outputs en `state` y una respuesta final coherente.

---

### Ejercicio 4 — 🔸 ADK: fan-out + synthesizer

**Objetivo:** `ParallelAgent` con 3 ramas + `SequentialAgent` para sintetizar.

**Pasos:**
1. 3 agentes paralelos:
   - `pros_agent`: argumentos a favor del input.
   - `cons_agent`: argumentos en contra.
   - `neutral_agent`: preguntas pendientes / data faltante.
2. `synthesizer_agent`: con `state['pros']`, `state['cons']`, `state['neutral']` → recomendación balanceada.
3. `root_agent = SequentialAgent([ParallelAgent([...]), synthesizer_agent])`.

**Input:** "¿Debería la empresa adoptar una semana laboral de 4 días?"

**Éxito:** los 3 outputs se generan concurrentemente (mirá timestamps en logs) y el synthesizer los referencia a todos.

---

### Ejercicio 5 — 🔺 Combinado: ADK orquesta, Claude SDK ejecuta

**Objetivo:** sistema donde ADK decide qué hacer, y un tool MCP (que internamente usa Claude SDK) ejecuta tareas de código.

**Opción A (fácil):**
1. En ADK, usá `LiteLlm` con modelo Claude directamente.
2. Ver [Claude en ADK](https://google.github.io/adk-docs/agents/models/anthropic/).

**Opción B (avanzada):**
1. Creá un MCP server standalone que expone un tool `execute_code_task(description: str)`.
2. Adentro del tool, corré un `ClaudeSDKClient` con tools `Read`/`Write`/`Bash` que ejecute la tarea real.
3. Desde ADK, consumilo como `MCPTool`.
4. Creá un `LlmAgent` coordinator que decida cuándo llamar este tool.

**Input:** "Escribí un script Python que genere un CSV con los primeros 100 números primos."

**Éxito:** ADK recibe el pedido, delega a Claude SDK que escribe el archivo, y ADK reporta "listo, CSV generado en /tmp/primes.csv".

---

## 📚 Parte 7 · Glosario + recursos oficiales

### Glosario express

- **Agent loop**: ciclo think → tool → observe → think hasta terminar.
- **Tool calling**: capacidad del LLM de emitir JSON estructurado que invoca una función.
- **MCP (Model Context Protocol)**: estándar abierto de Anthropic para que agentes consuman tools/resources externos.
- **A2A (Agent2Agent)**: estándar de Google para que agentes de distintos frameworks hablen entre sí.
- **Subagent**: agente hijo con contexto aislado (Claude SDK).
- **Workflow agent**: agente "orquestador" determinista en ADK (`SequentialAgent`, etc.).
- **State**: dict compartido en una Session de ADK.
- **Hook / Callback**: función que se ejecuta en un punto del loop (pre-tool, post-tool, etc.).
- **Skill**: paquete declarativo de capacidades (carpeta + `SKILL.md`) cargable bajo demanda.
- **Permission mode**: nivel de autonomía del agente (Claude SDK).
- **Runner**: motor de ejecución de ADK; maneja sesiones y eventos.
- **Session / outputKey**: mecanismos de estado de ADK.

### Claude Agent SDK — recursos

- Docs: https://platform.claude.com/docs/en/agent-sdk/overview
- Quickstart: https://platform.claude.com/docs/en/agent-sdk/quickstart
- Python ref: https://platform.claude.com/docs/en/agent-sdk/python
- TS ref: https://platform.claude.com/docs/en/agent-sdk/typescript
- Subagents: https://platform.claude.com/docs/en/agent-sdk/subagents
- Custom tools: https://platform.claude.com/docs/en/agent-sdk/custom-tools
- MCP: https://platform.claude.com/docs/en/agent-sdk/mcp
- Skills: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- Repo Python: https://github.com/anthropics/claude-agent-sdk-python
- Demos: https://github.com/anthropics/claude-agent-sdk-demos
- Blog post oficial: https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk

### Google ADK — recursos

- Docs principal: https://google.github.io/adk-docs
- Quickstart: https://google.github.io/adk-docs/get-started/quickstart/
- Multi-agentes: https://google.github.io/adk-docs/agents/multi-agents/
- Workflow agents: https://google.github.io/adk-docs/agents/workflow-agents/
- A2A: https://google.github.io/adk-docs/a2a/
- Agent Engine: https://docs.cloud.google.com/agent-builder/agent-engine/overview
- Deploy quickstart: https://docs.cloud.google.com/agent-builder/agent-engine/quickstart-adk
- Repo Python: https://github.com/google/adk-python
- Samples: https://github.com/google/adk-samples
- Codelab crash course: https://codelabs.developers.google.com/onramp/instructions
- Claude en ADK: https://google.github.io/adk-docs/agents/models/anthropic/
- Multi-agent patterns blog: https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/

### Comparativa externa (útil para fijar conceptos)

- https://composio.dev/content/claude-agents-sdk-vs-openai-agents-sdk-vs-google-adk
- https://prabha.ai/writing/2025/12/21/claude-agent-sdk-vs-google-adk/

---

## ⚠️ Parte 8 · Trampas comunes (errores que te van a caer)

### Claude SDK

1. **Olvidar el prefijo `mcp__` en `allowed_tools`.** Si declarás un tool custom llamado `greet` en un server `my-tools`, el tool es `mcp__my-tools__greet`, no `greet`. Sin el prefijo, el agente no lo usa.
2. **No especificar `cwd`.** El agente por default opera en el directorio desde donde ejecutaste Python. Si corrés desde `/`, puede ver (y modificar) TODO tu filesystem. Siempre definí `cwd`.
3. **`permission_mode="bypassPermissions"` en producción.** Es gatillo para ransomware-like. Usalo solo en scripts batch controlados.
4. **Tool functions síncronas.** El SDK espera **async**. Si hacés `def my_tool(args):` en vez de `async def`, va a tirar error raro.
5. **Node.js no instalado.** El SDK lanza `claude` (CLI Node). Sin Node 18+ en PATH, el error es `FileNotFoundError` críptico.
6. **Confundir Skills con Tools.** Skills son declarativas (carpeta `SKILL.md` que Claude lee), Tools son funciones. Un challenge que pide "agregar una capacidad reutilizable" probablemente quiere una Skill.
7. **Asumir que el subagent ve el contexto del padre.** No — cada subagent arranca con contexto limpio. Lo que necesite, se lo pasás en el prompt.

### Google ADK

1. **Sin `root_agent` en `agent.py`.** `adk web` y `adk run` buscan ese nombre específico. Si lo llamás `main_agent`, no anda.
2. **Docstring mal formateado en tools.** El LLM lee la docstring para saber qué hace el tool. Si no tiene `Args:` y `Returns:`, los params se pasan mal o no se llama.
3. **Return no-serializable.** Si tu tool devuelve un objeto `datetime` o un `pandas.DataFrame`, el LLM recibe `<object at 0x...>`. Convertí a dict/str.
4. **`output_key` duplicado en `ParallelAgent`.** Si dos ramas paralelas escriben la misma clave de state, se pisan. Siempre keys únicos.
5. **Instruction vaga.** "Sos útil" no es instruction. Describí comportamiento, tools esperados, formato de output.
6. **Confundir `sub_agents` (delegación LLM-driven) con `SequentialAgent.sub_agents` (orden determinista).** En el primero el LLM decide a quién transferir; en el segundo, el framework corre todos en orden. Distintos casos de uso.
7. **Deploy sin `requirements` completas.** `AgentEngine.create()` empaqueta tu código + requirements. Si olvidaste una dep, el agente crashea en prod.
8. **No usar `adk web` durante desarrollo.** Es la herramienta de debug más útil: ves state, events, tool calls en tiempo real. Resistí la tentación de armar todo programático desde el día 1.
9. **Costos de Vertex.** Agent Engine no es barato. Para práctica, usá **Gemini API con API key** (gratis hasta cuota).

### Ambos

1. **No leer el changelog reciente.** Ambos SDKs se mueven rápido; métodos deprecan seguido. Si un ejemplo del blog no funciona, revisá versión.
2. **Asumir paridad Python↔TS.** Hay gaps (especialmente en ADK, donde TS es más nuevo). Si el challenge es TS, validá features específicos.
3. **Meter lógica determinista en el LLM.** Regla: si podés escribirlo en 5 líneas de Python, no se lo pidas al LLM. En ADK usá `BaseAgent` custom; en Claude SDK usá un tool.
4. **No pensar en errores.** ¿Qué pasa si el tool falla? ¿Si el LLM alucina args inválidos? Challenges serios testean esto. En ambos SDKs tenés retry, error handling en tools, y hooks/callbacks para capturar.

---

## 🎯 Estrategia final (2h antes del challenge)

1. **Releé Parte 1 y 2** escaneando solo los códigos. Que te suene familiar cada línea.
2. **Mirá el enunciado 2 veces** antes de tocar teclado.
3. **Preguntá (si es entrevista):** "¿Prioridad en velocidad de delivery o en robustez?" → cambia el approach.
4. **Decidí SDK según enunciado:**
   - "agente autónomo que…" → Claude SDK
   - "sistema multi-agente que…" → ADK
   - "integrá con GCP…" → ADK
   - "acceso a filesystem/código…" → Claude SDK
   - "no sé todavía" → empezá con ADK si esperan ver estructura; Claude SDK si esperan ver resultado rápido.
5. **Estructura de tu solución** (aplica a ambos):
   - Agente mínimo que anda → `git commit`.
   - Agregá tools → commit.
   - Agregá error handling / guardrails → commit.
   - Tests o demo script → commit.
6. **Explicá trade-offs.** Los challenges no se ganan por código, se ganan por decisiones articuladas.

---

**Última actualización:** 2026-04-23. Confirmá versiones de SDKs antes de clonar código (`pip show claude-agent-sdk google-adk`).
