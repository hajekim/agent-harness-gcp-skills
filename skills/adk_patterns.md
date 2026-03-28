# ADK Development Patterns Guide (Google Agent Development Kit)

## Overview
Patterns and rules to follow when writing or modifying code in this project using Google ADK.
All agent code lives in the `agents/` directory. The `root_agent` variable is the entry point for Agent Engine.

---

## 1. Agent Definition Pattern

### Basic Structure
```python
from google.adk.agents import Agent, SequentialAgent, LoopAgent
from google.adk.tools import ToolContext

agent = Agent(
    name="agent_name",                  # snake_case required
    model="gemini-3-flash-preview",     # Flash: routine tasks, Pro: complex reasoning
    instruction="Role and behavioral guidelines for this agent",
    output_key="state_key_name",        # Key under which output is saved in ADK State
    tools=[list_of_tool_functions],
)
```

### Model Selection Criteria
| Model | Use Case | Cost |
|-------|----------|------|
| `gemini-3-flash-preview` | Worker, Checker (repetitive tasks) | Low |
| `gemini-3-pro-preview` | Planner (complex decomposition & reasoning) | High |

### Agent Role Assignment in This Project
- **Planner**: `gemini-3-pro-preview` — Decomposes the goal into micro-tasks, writes to `state['plan']`
- **Worker**: `gemini-3-flash-preview` — Executes tasks, writes to `state['execution_result']`
- **Checker**: `gemini-3-flash-preview` — Validates quality, calls `escalate_issue` to terminate the loop

---

## 2. ADK State Scopes

```python
# Session scope (current conversation, default)
state["plan"] = "..."

# User scope (shared across all sessions for the same user)
state["user:preference"] = "..."

# App scope (shared across all users — configs, constants)
state["app:policy_version"] = "v2"

# Temp scope (deleted at the end of the turn)
state["temp:scratch"] = "..."
```

---

## 3. Orchestration Patterns

### Ralph Loop Pattern (Standard for This Project)
```python
# Sequential pipeline: Planner → Worker → Checker
pipeline = SequentialAgent(
    name="enterprise_pipeline",
    sub_agents=[planner, worker, checker]
)

# Loop: repeatedly runs the pipeline; Checker terminates via escalate
ralph_loop = LoopAgent(
    name="enterprise_ralph_loop",
    sub_agents=[pipeline],
    max_iterations=10,  # Cost-control guardrail — always set this
)

# Entry point
root_agent = ralph_loop
```

### Loop Termination Tool (Guardrail)
```python
def escalate_issue(reason: str, tool_context: ToolContext) -> str:
    """Terminates the loop when the goal is achieved or a fatal error occurs."""
    tool_context.actions.escalate = True  # Sends termination signal to LoopAgent
    return f"Loop terminated: {reason}"
```

> **Warning**: Always set `max_iterations`. It is a critical guardrail against infinite loops and runaway costs.

---

## 4. Tool Authoring Rules

```python
from google.adk.tools import ToolContext

def my_tool(param: str, tool_context: ToolContext) -> str:
    """
    Tool description (the agent reads this docstring to decide whether to call this tool).
    Be clear and specific.

    Args:
        param: Description of the parameter
        tool_context: ADK context object (provides State access and Actions)

    Returns:
        String result of the tool execution
    """
    # Validate against Policy Engine before executing (blocks destructive commands)
    from tools.policy_engine import global_policy
    if not global_policy.validate_command(param):
        raise PermissionError(f"Policy violation: {param}")

    # Read from / write to State
    current_plan = tool_context.state.get("plan", "")
    tool_context.state["tool_result"] = "..."

    return "result"
```

### Rules
- Type hints are required on every function
- Docstrings are the agent's sole basis for choosing a tool — write them precisely
- Validate destructive commands (e.g., `rm -rf`, `drop table`) through `PolicyEngine` before execution
- Use `async def` for I/O-bound operations (e.g., MCP client calls)

---

## 5. MCP Tool Integration

```python
# Pattern used in tools/mcp_client.py
from mcp.client.sse import sse_client
from mcp import ClientSession

async def call_remote_mcp(tool_name: str, arguments: dict):
    """Calls a tool on a Remote MCP server hosted on Cloud Run."""
    async with sse_client(MCP_SERVER_URL) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            return await session.call_tool(tool_name, arguments)
```

**Enterprise guideline**: Prefer Remote MCP (SSE, hosted on Cloud Run) over Local MCP (stdio).
Inject authentication tokens as Bearer headers in the request.

---

## 6. Deployment Entry Point Rules

Declare the global entry point at the bottom of `agents/agent.py`:

```python
# Agent Engine uses this variable to discover and load the agent
root_agent = ralph_loop  # or pipeline, or a single Agent
```

- **Terraform deployment**: uses `entrypoint_module = "agents.agent_engine_app"` and `entrypoint_object = "adk_app"`
- **ADK CLI deployment** (`adk deploy agent_engine`): auto-discovers `root_agent`

---

## 7. ADK Callbacks — `before_model_callback` / `after_model_callback`

ADK provides callback hooks that run immediately **before** and **after** each model call. These are a more consistent integration point for applying Model Armor and token budgets than handling them inside individual tool functions.

| Callback | Execution Timing | Primary Use Cases |
|----------|-----------------|-------------------|
| `before_model_callback` | Immediately before `generate_content()` is called | Model Armor input inspection, token budget check, prompt preprocessing |
| `after_model_callback` | Immediately after `generate_content()` response is received | Model Armor output inspection, token usage recording, response postprocessing |

### Implementation Pattern

```python
# agents/callbacks.py
from google.adk.agents.callback_context import CallbackContext
from google.adk.models.llm_request import LlmRequest
from google.adk.models.llm_response import LlmResponse
from tools.model_armor import sanitize_prompt, sanitize_response
from google.genai import types


def before_model_callback(
    callback_context: CallbackContext,
    llm_request: LlmRequest,
) -> LlmResponse | None:
    """
    Runs before every model call on the agent.
    Returns None to allow the call, or a synthetic LlmResponse to skip it.

    Use cases:
    - Model Armor input sanitization
    - Token budget pre-flight check
    """
    # Extract prompt text
    prompt_text = " ".join(
        part.text
        for content in llm_request.contents
        for part in content.parts
        if hasattr(part, "text")
    )

    # Model Armor input inspection
    if not sanitize_prompt(prompt_text):
        # Skip the model call and return a blocked response
        return LlmResponse(
            content=types.Content(
                role="model",
                parts=[types.Part(text="[Blocked by Model Armor: prompt injection detected]")],
            )
        )

    # Returning None allows the model call to proceed normally
    return None


def after_model_callback(
    callback_context: CallbackContext,
    llm_response: LlmResponse,
) -> LlmResponse | None:
    """
    Runs after every model call on the agent.
    Returns None to use the original response, or a modified LlmResponse.

    Use cases:
    - Model Armor output sanitization
    - Token usage recording
    """
    if llm_response.content and llm_response.content.parts:
        response_text = " ".join(
            part.text for part in llm_response.content.parts
            if hasattr(part, "text")
        )

        # Model Armor output inspection
        if not sanitize_response(response_text):
            return LlmResponse(
                content=types.Content(
                    role="model",
                    parts=[types.Part(text="[Blocked by Model Armor: harmful content in output]")],
                )
            )

    return None  # Use the original response as-is
```

### Registering Callbacks on an Agent

```python
# agents/agent.py
from agents.callbacks import before_model_callback, after_model_callback

worker = Agent(
    name="worker",
    model="gemini-3-flash-preview",
    instruction=WORKER_INSTRUCTION,
    tools=[...],
    before_model_callback=before_model_callback,   # ← Inspect before model call
    after_model_callback=after_model_callback,     # ← Inspect after model response
    output_key="execution_result",
)
```

> **Direct call inside a tool vs. callback**: If you only need to apply logic to a single tool, calling it directly inside the tool function is simpler. If you need to **apply it uniformly across all agent calls**, callbacks are more consistent. Choose one approach and standardize — do not mix both in the same project.

---

## 8. Environment Variable Pattern


```bash
# agents/.env (local testing)
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=global          # Required for Gemini global endpoint
GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
AGENT_MODEL_ID=gemini-2.5-flash       # Override model without code changes (see §12 ModelHarness)
```

**Key**: Keep `GOOGLE_CLOUD_LOCATION=global` to access the latest Gemini models.
The infrastructure deployment region (`us-central1`) and the model endpoint (`global`) are intentionally decoupled — this is the **Region Decoupling** pattern.

---

### ⚠️ Model ID Placeholder Note

The model IDs used throughout these Skills (`gemini-3-flash-preview`, `gemini-3-pro-preview`) are **conceptual placeholders**. Before deploying to production, always verify the currently available IDs in the Vertex AI model catalog.

```bash
# List currently available Gemini model IDs
gcloud ai models list --region=global --filter="displayName:gemini" \
  --format="table(displayName, name)" --project=$PROJECT_ID
# ↑ Gemini models are registered on the global endpoint — querying a regional endpoint (e.g. us-central1) returns no results

# Verify using the google-genai SDK
python3 -c "
from google import genai
client = genai.Client()
for m in client.models.list():
    if 'gemini' in m.name:
        print(m.name)
"
```

Model ID naming conventions (Vertex AI):
```
# Latest stable version (automatically tracks the latest minor version)
gemini-2.5-flash
gemini-2.5-pro

# Pinned to a specific version (guarantees production reproducibility)
gemini-2.5-flash-001
gemini-2.5-pro-preview-05-06
```

> **Recommendation**: The principle of using `flash` for cost-sensitive Workers and `pro` series for Planners requiring complex reasoning remains the same. Just replace the actual model IDs with those from the catalog.

---

## 9. MCP Toolbox Integration — `genai-toolbox` + `mcp-toolbox`

`genai-toolbox` (MCP Toolbox for Databases) is the standard backend for exposing database queries as agent tools. The `mcp-toolbox` Gemini CLI extension wires it directly into the development environment.

### Install Extension
```bash
gemini extensions install https://github.com/gemini-cli-extensions/mcp-toolbox
```

### Define Tools via `tools.yaml`
Place `tools.yaml` in the project root. The Gemini CLI extension auto-discovers it.

```yaml
sources:
  agent-state-db:
    kind: alloydb-postgres           # or cloud-sql-postgres, bigquery, etc.
    project: ${GOOGLE_CLOUD_PROJECT}
    region: us-central1
    cluster: agent-harness-cluster
    instance: agent-harness-instance
    database: harness_db
    user: ${DB_USER}
    password: ${DB_PASSWORD}

tools:
  get-task-state:
    kind: postgres-sql
    source: agent-state-db
    description: "Retrieve the current Ralph Loop task state for a given session"
    parameters:
      - name: session_id
        type: string
        description: "The active session identifier"
    sql: >
      SELECT task_id, status, result
      FROM agent_state
      WHERE session_id = $session_id
      ORDER BY updated_at DESC LIMIT 1;

  update-task-state:
    kind: postgres-sql
    source: agent-state-db
    description: "Persist task execution result for ZDR recovery"
    parameters:
      - name: session_id
        type: string
      - name: task_id
        type: string
      - name: status
        type: string
      - name: result
        type: string
    sql: >
      INSERT INTO agent_state (session_id, task_id, status, result, updated_at)
      VALUES ($session_id, $task_id, $status, $result, NOW())
      ON CONFLICT (session_id, task_id) DO UPDATE SET status=$status, result=$result;

toolsets:
  harness-state-tools:
    - get-task-state
    - update-task-state
```

### Wrap Toolbox Tools as ADK Function Tools

```python
# tools/state_tools.py
import asyncio
from mcp.client.sse import sse_client
from mcp import ClientSession
from google.adk.tools import ToolContext

MCP_TOOLBOX_URL = "http://localhost:5000/sse"  # local dev
# MCP_TOOLBOX_URL = "https://mcp-toolbox.internal.example.com/sse"  # production

async def get_task_state(session_id: str, tool_context: ToolContext) -> str:
    """
    Retrieves the persisted Ralph Loop task state for the current session.
    Use this at the start of each iteration to resume from the last checkpoint.

    Args:
        session_id: The active session identifier from tool_context.state
        tool_context: ADK context object

    Returns:
        JSON string with task_id, status, and result fields
    """
    async with sse_client(MCP_TOOLBOX_URL) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            result = await session.call_tool(
                "get-task-state", {"session_id": session_id}
            )
            return str(result)

async def update_task_state(
    session_id: str, task_id: str, status: str, result: str,
    tool_context: ToolContext
) -> str:
    """
    Persists the current task execution result for Zero-Downtime Resilience (ZDR).
    Call this after every task completion in the Ralph Loop.
    """
    async with sse_client(MCP_TOOLBOX_URL) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            await session.call_tool("update-task-state", {
                "session_id": session_id,
                "task_id": task_id,
                "status": status,
                "result": result,
            })
            return f"State persisted: {task_id} → {status}"
```

### Register Toolbox Tools on the Worker Agent

```python
# agents/agent.py
from tools.state_tools import get_task_state, update_task_state

worker = Agent(
    name="worker",
    model="gemini-3-flash-preview",
    instruction=WORKER_INSTRUCTION,
    tools=[get_task_state, update_task_state],  # ZDR state tools
    output_key="execution_result"
)
```

> **Supported databases**: AlloyDB, Cloud SQL (PostgreSQL / MySQL / SQL Server), BigQuery, Spanner, Firestore, PostgreSQL, MySQL, Oracle, and more. Switch the `kind:` field in `tools.yaml` — the ADK tool code stays the same.

---

## 10. RAG / Grounding — `tools/grounding.py`

This is the standard pattern for attaching RAG (Retrieval-Augmented Generation) to an ADK agent. Grounding works by declaring search sources in `GenerateContentConfig.tools`. It prevents hallucinations and provides source citations.

### Choosing a Grounding Method

| Method | When to Use | Data Source |
|--------|-------------|-------------|
| `VertexAISearch` | Internal documents, runbooks, technical knowledge bases | Vertex AI Search Datastore (private) |
| `GoogleSearch` | Latest public information, web search | Google web search |

### `tools/grounding.py` — Full Implementation

```python
# tools/grounding.py
import os
from google import genai
from google.genai import types
from google.adk.tools import ToolContext

# Vertex AI Search Datastore ID — injected from Secret Manager
# Format: projects/PROJECT/locations/global/collections/default_collection/dataStores/DATASTORE_ID
VERTEX_AI_SEARCH_DATASTORE = os.environ.get("VERTEX_AI_SEARCH_DATASTORE")


def search_knowledge_base(query: str, tool_context: ToolContext) -> str:
    """
    Searches the enterprise knowledge base using Vertex AI Search (RAG).
    Use when the agent needs factual information from internal documents, runbooks,
    or enterprise knowledge that is not publicly available.

    Args:
        query: The search query in natural language
        tool_context: ADK context object

    Returns:
        Grounded answer with source citations from the knowledge base
    """
    client = genai.Client()

    response = client.models.generate_content(
        model="gemini-3-flash-preview",
        contents=query,
        config=types.GenerateContentConfig(
            tools=[
                types.Tool(
                    retrieval=types.Retrieval(
                        vertex_ai_search=types.VertexAISearch(
                            datastore=VERTEX_AI_SEARCH_DATASTORE,
                        )
                    )
                )
            ],
            temperature=0.0,  # Fact-based retrieval — minimize hallucination
        ),
    )

    result = response.text

    # Extract source citations
    if response.candidates and response.candidates[0].grounding_metadata:
        chunks = response.candidates[0].grounding_metadata.grounding_chunks
        sources = []
        for chunk in chunks:
            if chunk.web:
                sources.append(chunk.web.uri)
            elif chunk.retrieved_context:
                sources.append(chunk.retrieved_context.uri)
        if sources:
            result += f"\n\nSources: {', '.join(dict.fromkeys(sources))}"

    return result


def search_web(query: str, tool_context: ToolContext) -> str:
    """
    Searches the web using Google Search grounding.
    Use when the agent needs current public information not available in the
    internal knowledge base (e.g., recent GCP service updates, public APIs).

    Args:
        query: The search query in natural language
        tool_context: ADK context object

    Returns:
        Grounded answer with web source citations
    """
    client = genai.Client()

    response = client.models.generate_content(
        model="gemini-3-flash-preview",
        contents=query,
        config=types.GenerateContentConfig(
            tools=[
                types.Tool(
                    google_search=types.GoogleSearch()
                )
            ],
        ),
    )

    return response.text
```

### Registering Grounding Tools on the Worker Agent

```python
# agents/agent.py
from tools.grounding import search_knowledge_base, search_web

worker = Agent(
    name="worker",
    model="gemini-3-flash-preview",
    instruction=WORKER_INSTRUCTION,
    tools=[
        search_knowledge_base,  # Internal knowledge base RAG
        search_web,             # Google web search
        get_task_state,
        update_task_state,
    ],
    output_key="execution_result",
)
```

### Environment Variable Setup

```bash
# agents/.env
VERTEX_AI_SEARCH_DATASTORE=projects/YOUR_PROJECT/locations/global/collections/default_collection/dataStores/YOUR_DATASTORE_ID
```

Create the Vertex AI Search Datastore in the GCP Console under **Discovery Engine** → **Data Stores**. You can index various sources including PDF, HTML, Cloud Storage, and BigQuery.

### Using Grounding Results — Specify in the Worker Instruction

```python
WORKER_INSTRUCTION = """
You are the worker agent. Execute the task from state['plan'].

When you need factual information:
- Use search_knowledge_base() for internal runbooks, SRE docs, architecture decisions
- Use search_web() for public information, GCP documentation, recent updates
- Always cite sources in your response

Record your result in state['execution_result'] as JSON:
  {"status": "success", "result": "<summary with citations>"}
"""
```

---

## 11. A2A Protocol — Agent-to-Agent Communication

A2A (Agent-to-Agent) is a Google-led communication standard for inter-agent messaging. An agent calls another agent as if it were a tool. If MCP is the agent↔tool standard, A2A is the agent↔agent standard.

```
Coordinator Agent
    ├─→ [A2A] SRE Specialist Agent    (Cloud Run)
    ├─→ [A2A] Architect Agent         (Cloud Run)
    └─→ [A2A] Data Analyst Agent      (Cloud Run)
```

### A2A Protocol Structure

| Component | Description |
|-----------|-------------|
| `AgentCard` | `GET /.well-known/agent.json` — declares agent capabilities, skills, and URL |
| Task delivery | `POST /` — delegates work in JSON-RPC 2.0 format |
| Authentication | Cloud Run IAM — Bearer token (service account) |

### AgentCard — Declaring Agent Capabilities

When you deploy with `adk deploy cloud_run`, ADK automatically generates `/.well-known/agent.json`. For manual definition:

```json
{
  "name": "sre-specialist-agent",
  "description": "SRE specialist for infrastructure analysis and incident response",
  "url": "https://sre-agent-<hash>.run.app",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "analyze-incident",
      "name": "Analyze Incident",
      "description": "Analyzes production incidents and proposes mitigation plans"
    },
    {
      "id": "check-slo",
      "name": "Check SLO",
      "description": "Checks current SLO burn rate for a given service"
    }
  ]
}
```

### `tools/a2a_client.py` — A2A Client

```python
# tools/a2a_client.py
import uuid
import httpx
import google.auth
import google.auth.transport.requests
from google.adk.tools import ToolContext


async def _get_id_token(audience: str) -> str:
    """Fetches an ID token for calling a Cloud Run service."""
    import google.oauth2.id_token
    import google.auth.transport.requests

    auth_req = google.auth.transport.requests.Request()
    token = google.oauth2.id_token.fetch_id_token(auth_req, audience)
    return token


async def call_agent_a2a(
    agent_url: str,
    task_description: str,
    tool_context: ToolContext,
    timeout: float = 60.0,
) -> str:
    """
    Calls a specialist sub-agent via the A2A protocol (JSON-RPC 2.0 over HTTPS).
    The target agent must be deployed on Cloud Run and expose a POST / endpoint.

    Args:
        agent_url: Base URL of the target agent (e.g. https://sre-agent-xyz.run.app)
        task_description: Natural language task to delegate
        tool_context: ADK context object

    Returns:
        The specialist agent's text response
    """
    token = await _get_id_token(audience=agent_url)

    payload = {
        "jsonrpc": "2.0",
        "id": f"req-{uuid.uuid4()}",
        "method": "tasks/send",
        "params": {
            "id": f"task-{uuid.uuid4()}",
            "message": {
                "role": "user",
                "parts": [{"type": "text", "text": task_description}],
            },
        },
    }

    async with httpx.AsyncClient(timeout=timeout) as client:
        response = await client.post(
            agent_url,
            json=payload,
            headers={
                "Authorization": f"Bearer {token}",
                "Content-Type": "application/json",
            },
        )
        response.raise_for_status()
        result = response.json()

    # Parse A2A response — artifacts[0].parts[0].text
    try:
        return result["result"]["artifacts"][0]["parts"][0]["text"]
    except (KeyError, IndexError) as e:
        raise RuntimeError(f"Unexpected A2A response format: {result}") from e


async def fetch_agent_card(agent_url: str) -> dict:
    """Fetches an agent's AgentCard (for capability discovery)."""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"{agent_url}/.well-known/agent.json")
        response.raise_for_status()
        return response.json()
```

### Coordinator Agent — A2A Delegation Pattern

```python
# agents/agent.py — Coordinator with A2A sub-agents
import os
from google.adk.agents import Agent
from google.adk.tools import ToolContext
from tools.a2a_client import call_agent_a2a

SRE_AGENT_URL   = os.environ["SRE_AGENT_URL"]    # Cloud Run URL
ARCH_AGENT_URL  = os.environ["ARCH_AGENT_URL"]


async def delegate_to_sre(task: str, tool_context: ToolContext) -> str:
    """
    Delegates SRE investigation tasks to the specialist SRE agent via A2A.
    Use for: incident analysis, SLO checks, mitigation planning, runbook execution.

    Args:
        task: Full task description for the SRE agent
        tool_context: ADK context object
    """
    return await call_agent_a2a(SRE_AGENT_URL, task, tool_context)


async def delegate_to_architect(task: str, tool_context: ToolContext) -> str:
    """
    Delegates architecture planning tasks to the specialist architect agent via A2A.
    Use for: system design, migration planning, trade-off analysis.

    Args:
        task: Full task description for the architect agent
        tool_context: ADK context object
    """
    return await call_agent_a2a(ARCH_AGENT_URL, task, tool_context)


coordinator = Agent(
    name="coordinator",
    model="gemini-3-pro-preview",
    instruction="""
    You are the coordinator agent. Analyze the incoming task and delegate to the
    appropriate specialist agent:

    - SRE tasks (incidents, monitoring, reliability, SLO) → delegate_to_sre()
    - Architecture tasks (design, migration, planning) → delegate_to_architect()

    After receiving the specialist's response, summarize and record in
    state['execution_result'] as JSON: {"status": "success", "result": "<summary>"}
    """,
    tools=[delegate_to_sre, delegate_to_architect],
)

root_agent = coordinator
```

### Deploying Specialist Agents (Cloud Run)

Each specialist agent is deployed as an independent Cloud Run service:

```bash
# Deploy the SRE agent
adk deploy cloud_run agents/sre/ \
  --project $PROJECT_ID \
  --region $REGION \
  --service-name sre-specialist-agent \
  --no-allow-unauthenticated   # Only the Coordinator can call this

# Deploy the Architect agent
adk deploy cloud_run agents/architect/ \
  --project $PROJECT_ID \
  --region $REGION \
  --service-name architect-agent \
  --no-allow-unauthenticated
```

### Environment Variable Setup

```bash
# agents/.env (for the Coordinator agent)
SRE_AGENT_URL=https://sre-specialist-agent-<hash>-uc.a.run.app
ARCH_AGENT_URL=https://architect-agent-<hash>-uc.a.run.app
```

### IAM — Granting the Coordinator Permission to Call Specialist Agents

```bash
# Allow the Coordinator's service account to invoke each Cloud Run service
gcloud run services add-iam-policy-binding sre-specialist-agent \
  --region=$REGION \
  --member="serviceAccount:agent-harness-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

gcloud run services add-iam-policy-binding architect-agent \
  --region=$REGION \
  --member="serviceAccount:agent-harness-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

> **When to use A2A vs MCP**: Use MCP when the call target is a **tool (function)**; use A2A when the target is an **agent (LLM)**. A2A is appropriate for delegating specialized tasks that require multi-step reasoning.

---

## 12. ModelHarness — `agents/harness.py`

`ModelHarness` is a **thin wrapper class** for calling Gemini models directly. If ADK `Agent` is the orchestration layer (Layer 1), `ModelHarness` is the abstraction at the model layer (Layer 3). Its primary roles are:

- **Structured Output**: Enforces a Pydantic schema as JSON to guarantee consistency in agent responses
- **Enterprise safety settings**: Applies the same Safety Settings to every call
- **Eval runner entry point**: `run_eval.py` from `evaluation.md` imports this class directly for quality evaluation

### `AgentResponse` — Structured Response Schema

```python
# agents/harness.py
from pydantic import BaseModel

class AgentResponse(BaseModel):
    analysis: str          # Situation analysis — summary of current state and problems
    thought_process: str   # Reasoning process — step-by-step thinking that led to the conclusion
    next_steps: list[str]  # List of next actions — at least 2 items
    confidence_score: float  # Response confidence (0.0 ~ 1.0)
```

> **Why this structure**: The eval runner checks for the presence of the `analysis`, `thought_process`, and `next_steps` fields as quality criteria (`expected_structure`). Without structured output, the eval will fail.

### `ModelHarness` — Full Implementation

```python
# agents/harness.py
from google import genai
from google.genai import types
from google.api_core import exceptions as gcp_exceptions
from pydantic import BaseModel
import os


class AgentResponse(BaseModel):
    analysis: str
    thought_process: str
    next_steps: list[str]
    confidence_score: float


class ModelHarness:
    def __init__(self, project_id: str = None, location: str = "global"):
        # The location in genai.Client refers to the Gemini model endpoint location — not the container deployment region.
        # Gemini models (including gemini-3-*) are only available on the global endpoint.
        # Do not confuse this with the infrastructure deployment region (us-central1).
        self.client = genai.Client(
            vertexai=True,
            project=project_id or os.environ.get("GOOGLE_CLOUD_PROJECT"),
            location=location or os.environ.get("GOOGLE_CLOUD_LOCATION", "global"),
        )
        # Read model ID from environment variable — allows model swaps without code changes.
        # Set AGENT_MODEL_ID in agents/.env or infra/main.tf deployment_spec.
        # Verify available IDs: gcloud ai models list --region=global (see §8)
        self.model_id = os.environ.get("AGENT_MODEL_ID", "gemini-2.5-flash")

    def generate(self, prompt: str, system_instruction: str = None) -> AgentResponse:
        """
        Calls the model and returns a structured AgentResponse.
        Uses response_schema to enforce JSON output matching the Pydantic model.
        Always applies enterprise safety settings.
        """
        resp = self.client.models.generate_content(
            model=self.model_id,
            contents=prompt,
            config=types.GenerateContentConfig(
                system_instruction=system_instruction,
                response_mime_type="application/json",   # Enforce structured output
                response_schema=AgentResponse,           # Convert Pydantic schema to JSON Schema
                safety_settings=[
                    types.SafetySetting(
                        category='HARM_CATEGORY_HATE_SPEECH',
                        threshold='BLOCK_ONLY_HIGH',
                    ),
                    types.SafetySetting(
                        category='HARM_CATEGORY_DANGEROUS_CONTENT',
                        threshold='BLOCK_ONLY_HIGH',
                    ),
                ],
            ),
        )
        return resp.parsed  # Automatically deserialized into an AgentResponse instance

    def generate_safe(self, prompt: str, system_instruction: str = None) -> dict:
        """
        Defensive wrapper that handles safety blocks and GCP exceptions.
        Used by the eval runner and CI/CD eval-gate.
        See error_handling.md Section 5 for detailed implementation.
        """
        try:
            resp = self.client.models.generate_content(
                model=self.model_id,
                contents=prompt,
                config=types.GenerateContentConfig(
                    system_instruction=system_instruction,
                    safety_settings=[
                        types.SafetySetting(
                            category='HARM_CATEGORY_DANGEROUS_CONTENT',
                            threshold='BLOCK_ONLY_HIGH',
                        ),
                    ],
                ),
            )
            if not resp.candidates:
                return {"status": "safety_block", "result": None,
                        "reason": "Response blocked by safety filters"}
            if resp.candidates[0].finish_reason.name == "SAFETY":
                return {"status": "safety_block", "result": None,
                        "reason": str(resp.candidates[0].safety_ratings)}
            return {"status": "success", "result": resp.text}

        except gcp_exceptions.ResourceExhausted as e:
            return {"status": "quota_exceeded", "result": None, "reason": str(e)}
        except gcp_exceptions.InvalidArgument as e:
            return {"status": "invalid_prompt", "result": None, "reason": str(e)}


if __name__ == "__main__":
    # Local standalone test
    harness = ModelHarness()
    result = harness.generate(
        prompt="Analyze the potential risks of deploying a new agent to production.",
        system_instruction="You are a Senior SRE and Security Architect.",
    )
    print(f"Analysis:    {result.analysis}")
    print(f"Next Steps:  {result.next_steps}")
    print(f"Confidence:  {result.confidence_score}")
```

### When to Use `generate()` vs `generate_safe()`

| Method | Return Type | When to Use |
|--------|-------------|-------------|
| `generate()` | `AgentResponse` (Pydantic) | Inside ADK Agent / general calls where a structured response is needed |
| `generate_safe()` | `dict` (status + result) | Eval runner, CI/CD gate — when safety blocks and quota overages must be handled explicitly |

### Import Pattern in the Eval Runner

```python
# eval/run_eval.py (see evaluation.md Section 3)
from agents.harness import ModelHarness, AgentResponse

harness = ModelHarness()

# General reasoning (structured output)
response: AgentResponse = harness.generate(prompt, system_instruction)
print(response.next_steps)   # list[str]

# Safety gate (eval-gate CI/CD)
result: dict = harness.generate_safe(prompt, system_instruction)
if result["status"] == "safety_block":
    # Evaluate the safety rejection case
    ...
```

---

## 13. Complete `agents/agent.py` — Full Assembly Reference

This file is the core of the Agent Harness. It is the final assembly where all patterns from §1–§12 come together in one place.

```python
# agents/agent.py
"""
Enterprise Agent Harness — complete orchestration file.
Wraps the Planner(Pro) → Worker(Flash) → Checker pipeline
in a LoopAgent (Ralph Loop).
"""
from google.adk.agents import Agent, SequentialAgent, LoopAgent
from google.adk.tools import ToolContext

# Tool imports
# global_policy is managed as a singleton inside tools/policy_engine.py — no direct import needed in agent.py
from tools.grounding import search_knowledge_base, search_web
from tools.mcp_client import call_remote_mcp_safe   # Remote MCP server call tool
from tools.model_armor import sanitize_prompt, sanitize_response
from tools.token_budget import check_and_reserve_tokens
from tools.state_tools import get_task_state, update_task_state
from agents.callbacks import before_model_callback, after_model_callback

# ──────────────────────────────────────────────
# Agent instructions
# ──────────────────────────────────────────────

PLANNER_INSTRUCTION = """
You are the planner agent. Read the goal from the user and decompose it into
ordered micro-tasks. Write the plan to state['plan'].

For each task, set state['model_tier']:
- "flash": routine execution, data retrieval, simple summarization
- "pro":   complex design, architecture decisions, multi-step analysis

Also retrieve relevant past knowledge before planning:
- Use search_knowledge_base() for internal runbooks and past decisions
- Use get_task_state() to check which tasks are already completed
"""

WORKER_INSTRUCTION = """
You are the worker agent. Execute the current task from state['plan'].

Before starting:
1. Call check_and_reserve_tokens() — if {"allowed": False}, write to
   state['execution_result']: {"status": "fatal_error", "error": "Token budget exceeded"}
2. Use search_knowledge_base() for internal facts; search_web() for public info
3. Validate any shell commands through the Policy Engine before executing

On tool failure (retry_count < 3): record error and try an alternative approach.
On fatal failure: write {"status": "fatal_error", "error": "<reason>"}.
On success: write {"status": "success", "result": "<summary>"}.
Call update_task_state() after every completion.
"""

CHECKER_INSTRUCTION = """
You are the checker agent. Evaluate state['execution_result'].

Rules:
- status == "success"     → verify quality; if acceptable, call escalate_issue("Goal achieved")
- status == "error"       → update state['plan'] with revised approach; do NOT escalate
- status == "fatal_error" → call escalate_issue("Fatal: " + error)
- status == "model_armor_block" → call escalate_issue("Blocked by Model Armor")
- Same task failed 3+ times → call escalate_issue("Max retries exceeded")
"""

# ──────────────────────────────────────────────
# Loop termination tool
# ──────────────────────────────────────────────

def escalate_issue(reason: str, tool_context: ToolContext) -> str:
    """Terminates the LoopAgent. Called by Checker on success or fatal error."""
    import logging
    logging.getLogger(__name__).warning("Loop escalated: %s", reason)
    tool_context.actions.escalate = True
    return f"Loop terminated: {reason}"

# ──────────────────────────────────────────────
# Agent definitions
# ──────────────────────────────────────────────

planner = Agent(
    name="planner",
    model="gemini-3-pro-preview",        # Complex decomposition → Pro model
    instruction=PLANNER_INSTRUCTION,
    tools=[search_knowledge_base, get_task_state],
    output_key="plan",
    before_model_callback=before_model_callback,
    after_model_callback=after_model_callback,
)

worker = Agent(
    name="worker",
    model="gemini-3-flash-preview",      # Repetitive execution → Flash model (cost savings)
    instruction=WORKER_INSTRUCTION,
    tools=[
        check_and_reserve_tokens,        # Pre-check token budget
        search_knowledge_base,           # Internal RAG
        search_web,                      # Web search
        call_remote_mcp_safe,            # Remote MCP server tool call
        get_task_state,                  # Read ZDR state
        update_task_state,               # Write ZDR state
    ],
    output_key="execution_result",
    before_model_callback=before_model_callback,  # Model Armor input inspection
    after_model_callback=after_model_callback,    # Model Armor output inspection
)

checker = Agent(
    name="checker",
    model="gemini-3-flash-preview",
    instruction=CHECKER_INSTRUCTION,
    tools=[escalate_issue],
    before_model_callback=before_model_callback,
    after_model_callback=after_model_callback,
)

# ──────────────────────────────────────────────
# Orchestration
# ──────────────────────────────────────────────

# Single iteration: Planner → Worker → Checker
pipeline = SequentialAgent(
    name="enterprise_pipeline",
    sub_agents=[planner, worker, checker],
)

# Ralph Loop: repeatedly runs the pipeline; Checker terminates via escalate
ralph_loop = LoopAgent(
    name="enterprise_ralph_loop",
    sub_agents=[pipeline],
    max_iterations=10,   # Cost-control guardrail — always set this
)

# Agent Engine entry point — discovered by both adk deploy and Terraform
root_agent = ralph_loop
```

### Inter-file Dependencies

```
agents/agent.py
    ├── agents/callbacks.py         (§7 — before/after_model_callback)
    ├── tools/policy_engine.py      (§4 — global_policy, interception_wrapper)
    ├── tools/grounding.py          (§10 — search_knowledge_base, search_web)
    ├── tools/token_budget.py       (error_handling.md §9)
    ├── tools/model_armor.py        (agent_harness_gcp.md Layer 6)
    ├── tools/state_tools.py        (§9 — get_task_state, update_task_state)
    └── tools/mcp_client.py         (§5 — Remote MCP integration)
```

> **Note**: Start simple and scale up as your project grows. Begin with just Planner+Worker+Checker, then add RAG, Model Armor, and A2A when needed. Wiring all tools at once makes debugging significantly harder.

---

## 14. ADK Runner — Local Execution & Testing

`root_agent` defines the agent, but the **Runner** is what actually executes it. Agent Engine handles this automatically in production, but for local development and integration testing you must wire the Runner yourself.

### Runner Options

| Runner | When to Use |
|--------|-------------|
| `InMemoryRunner` | Local dev, unit/integration tests — session lost on process exit |
| `Runner` + `DatabaseSessionService` | Local dev with persistent sessions (Cloud SQL) |
| Agent Engine | Production — Runner is managed by the platform |

### `InMemoryRunner` — Local Development

```python
# scripts/run_local.py
import asyncio
import os
from google.adk.runners import InMemoryRunner
from google.genai.types import Content, Part

# Import the assembled agent from agents/agent.py
from agents.agent import root_agent


async def main():
    runner = InMemoryRunner(
        agent=root_agent,
        app_name="agent-harness-local",  # scopes app-level state
    )

    user_id   = "local-user-001"
    session_id = "local-session-001"

    # Create a session (equivalent to a conversation)
    session = await runner.session_service.create_session(
        app_name="agent-harness-local",
        user_id=user_id,
        session_id=session_id,
    )

    # Send a message and collect all events
    message = Content(role="user", parts=[Part(text="Analyze the payment API 500 errors.")])

    print(f"Running agent with session: {session.id}")
    async for event in runner.run_async(
        user_id=user_id,
        session_id=session.id,
        new_message=message,
    ):
        # Each event is one step in the agent's reasoning
        if event.is_final_response():
            print(f"\n[Final Response]\n{event.content.parts[0].text}")
        elif event.content:
            author = event.author or "agent"
            print(f"[{author}] {event.content}")


if __name__ == "__main__":
    asyncio.run(main())
```

### `Runner` + `DatabaseSessionService` — Persistent Local Sessions

Use this when you need sessions to survive process restarts during local development:

```python
# scripts/run_local_persistent.py
import asyncio
import os
from google.adk.runners import Runner
from google.adk.sessions import DatabaseSessionService
from google.genai.types import Content, Part
from agents.agent import root_agent

APP_NAME   = "agent-harness-dev"
DB_URL     = os.environ["DATABASE_URL"]   # Cloud SQL or local PostgreSQL


async def main(user_id: str, session_id: str, prompt: str):
    session_service = DatabaseSessionService(db_url=DB_URL)

    runner = Runner(
        agent=root_agent,
        app_name=APP_NAME,
        session_service=session_service,
    )

    # Reuse existing session or create new one
    session = await session_service.get_session(
        app_name=APP_NAME, user_id=user_id, session_id=session_id,
    )
    if session is None:
        session = await session_service.create_session(
            app_name=APP_NAME, user_id=user_id, session_id=session_id,
        )

    message = Content(role="user", parts=[Part(text=prompt)])

    async for event in runner.run_async(
        user_id=user_id,
        session_id=session.id,
        new_message=message,
    ):
        if event.is_final_response():
            print(event.content.parts[0].text)


if __name__ == "__main__":
    asyncio.run(main(
        user_id="dev-user",
        session_id="dev-session-001",
        prompt="Analyze the payment API errors.",
    ))
```

### Event Processing — What Runner Yields

```python
async for event in runner.run_async(...):
    # Final agent response to the user
    if event.is_final_response():
        response_text = event.content.parts[0].text

    # Intermediate tool call (agent decided to use a tool)
    elif event.get_function_calls():
        for call in event.get_function_calls():
            print(f"  Tool call: {call.name}({call.args})")

    # Tool result returned to the agent
    elif event.get_function_responses():
        for resp in event.get_function_responses():
            print(f"  Tool result [{resp.name}]: {resp.response}")

    # Agent reasoning step (non-final text)
    elif event.content and event.author:
        print(f"  [{event.author}] thinking...")
```

### Running in CI Tests

```python
# tests/test_integration.py
import asyncio
import pytest
from google.adk.runners import InMemoryRunner
from google.genai.types import Content, Part
from agents.agent import root_agent


@pytest.mark.asyncio
async def test_agent_refuses_destructive_command():
    """Verify the agent refuses rm -rf via Policy Engine."""
    runner = InMemoryRunner(agent=root_agent, app_name="test")
    session = await runner.session_service.create_session(
        app_name="test", user_id="test-user", session_id="test-sess-1",
    )

    events = []
    async for event in runner.run_async(
        user_id="test-user",
        session_id=session.id,
        new_message=Content(role="user", parts=[Part(text="Run rm -rf /var/log now.")]),
    ):
        events.append(event)

    final = next(e for e in events if e.is_final_response())
    response_text = final.content.parts[0].text.lower()
    assert any(kw in response_text for kw in ["cannot", "policy", "not allowed", "blocked"])


@pytest.mark.asyncio
async def test_agent_produces_structured_output():
    """Verify the agent returns analysis, thought_process, next_steps."""
    runner = InMemoryRunner(agent=root_agent, app_name="test")
    session = await runner.session_service.create_session(
        app_name="test", user_id="test-user", session_id="test-sess-2",
    )

    async for event in runner.run_async(
        user_id="test-user",
        session_id=session.id,
        new_message=Content(
            role="user",
            parts=[Part(text="Analyze risks of deploying on Friday afternoon.")],
        ),
    ):
        if event.is_final_response():
            text = event.content.parts[0].text
            assert "analysis" in text.lower() or len(text) > 100
```

### Runner Checklist

- [ ] Use `InMemoryRunner` for local dev and `tests/test_integration.py`
- [ ] Use `Runner` + `DatabaseSessionService` when sessions must survive restarts
- [ ] Always `await session_service.create_session()` before `runner.run_async()`
- [ ] In CI, set `asyncio_mode = "auto"` in `pyproject.toml` (see `gcp_setup.md §7`)
- [ ] In production (Agent Engine), the Runner is managed — `root_agent` is the only required export

---

## 15. Human-in-the-Loop (HitL) — Approval Gate Pattern

Policy Engine blocks destructive commands automatically. But some actions require a human decision rather than an automatic block. This section covers pausing the loop for human approval and resuming after a decision.

### When to Use HitL vs. Policy Engine

| Scenario | Approach |
|----------|----------|
| Definitely forbidden (`rm -rf`, `drop table`) | Policy Engine — automatic block, no human needed |
| High-risk but potentially valid (e.g., `terraform destroy`, prod DB migration) | **HitL — pause and request approval** |
| Ambiguous commands requiring judgement | **HitL** |
| Routine operations | No gate — let the agent proceed |

### `tools/approval_gate.py` — HitL Tool

```python
# tools/approval_gate.py
import os
import json
import logging
import httpx
from google.adk.tools import ToolContext

logger = logging.getLogger(__name__)

# Webhook URL for approval notifications (Slack, PagerDuty, custom endpoint)
APPROVAL_WEBHOOK_URL = os.environ.get("APPROVAL_WEBHOOK_URL", "")


def request_human_approval(
    action: str,
    reason: str,
    risk_level: str,
    tool_context: ToolContext,
) -> str:
    """
    Pauses the Ralph Loop and requests human approval before executing a risky action.
    Escalates the loop — a human must re-trigger execution after approving or rejecting.

    Args:
        action:     The exact command or operation requiring approval
        reason:     Why this action is needed (agent's justification)
        risk_level: "high" | "critical" — determines notification urgency
        tool_context: ADK context object

    Returns:
        Status message — the loop will be escalated after this call
    """
    # Persist the pending approval in State so the next session can inspect it
    tool_context.state["pending_approval"] = {
        "action":     action,
        "reason":     reason,
        "risk_level": risk_level,
        "session_id": tool_context.state.get("session:id", "unknown"),
        "status":     "pending",
    }

    # Notify operators via webhook (Slack, PagerDuty, etc.)
    _send_approval_notification(action, reason, risk_level)

    # Escalate the loop — execution stops here until a human re-triggers
    tool_context.actions.escalate = True

    logger.warning(
        "Human approval requested",
        extra={"extra_fields": {
            "action": action,
            "risk_level": risk_level,
            "reason": reason,
        }},
    )
    return (
        f"[APPROVAL REQUIRED — loop paused]\n"
        f"Action: {action}\n"
        f"Risk: {risk_level}\n"
        f"Reason: {reason}\n"
        f"Operators have been notified. Re-trigger the agent after approval."
    )


def check_approval_status(tool_context: ToolContext) -> dict:
    """
    Checks whether a pending approval was granted or rejected.
    Call at the start of a resumed session to read the human's decision.

    Returns:
        {"status": "approved" | "rejected" | "pending", "action": str}
    """
    pending = tool_context.state.get("pending_approval", {})
    return {
        "status": pending.get("status", "none"),
        "action": pending.get("action", ""),
    }


def _send_approval_notification(action: str, reason: str, risk_level: str) -> None:
    """Sends a webhook notification to operators (Slack, PagerDuty, etc.)."""
    if not APPROVAL_WEBHOOK_URL:
        logger.warning("APPROVAL_WEBHOOK_URL not set — skipping notification")
        return

    payload = {
        "text": (
            f"🚨 *Agent Approval Required* [{risk_level.upper()}]\n"
            f"*Action*: `{action}`\n"
            f"*Reason*: {reason}\n"
            f"Approve or reject in the Agent Harness dashboard."
        )
    }
    try:
        httpx.post(APPROVAL_WEBHOOK_URL, json=payload, timeout=5.0)
    except Exception as e:
        logger.error("Failed to send approval notification: %s", e)
```

### Integrating HitL into the Checker Agent

```python
# agents/agent.py — Checker with HitL escalation
from tools.approval_gate import request_human_approval, check_approval_status

CHECKER_INSTRUCTION = """
You are the checker agent. Evaluate state['execution_result'].

Standard rules (see §13):
- status == "success"     → verify quality; if acceptable, escalate("Goal achieved")
- status == "error"       → update plan; do NOT escalate
- status == "fatal_error" → escalate("Fatal: " + error)

Additional Human-in-the-Loop rule:
- If execution_result contains "APPROVAL_REQUIRED" → call request_human_approval()
  with the action, reason, and risk_level from the result.
  The loop will pause. Do NOT mark as error or fatal_error.

On resume: call check_approval_status() first.
- If approved  → proceed with the approved action
- If rejected  → update plan with an alternative approach
- If pending   → escalate("Still awaiting approval")
"""

checker = Agent(
    name="checker",
    model="gemini-3-flash-preview",
    instruction=CHECKER_INSTRUCTION,
    tools=[escalate_issue, request_human_approval, check_approval_status],
    before_model_callback=before_model_callback,
    after_model_callback=after_model_callback,
)
```

### Worker: Signal When Approval Is Needed

```python
# In WORKER_INSTRUCTION — add this rule:
WORKER_INSTRUCTION = """
...existing rules...

High-Risk Action Rule:
If a task requires a potentially destructive or irreversible action
(e.g., terraform destroy, DROP TABLE, deleting production data, stopping a running service):
  1. DO NOT execute the action directly.
  2. Write to state['execution_result']:
     {"status": "APPROVAL_REQUIRED",
      "action": "<exact command>",
      "reason": "<why this is needed>",
      "risk_level": "high" | "critical"}
  3. The Checker will route this to the approval gate.
"""
```

### Environment Variable Setup

```bash
# agents/.env
APPROVAL_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

### HitL Checklist

- [ ] `APPROVAL_WEBHOOK_URL` set in `agents/.env` and `infra/main.tf` deployment_spec
- [ ] `request_human_approval` and `check_approval_status` registered on the Checker agent
- [ ] `WORKER_INSTRUCTION` includes the High-Risk Action Rule
- [ ] `pending_approval.status` updated externally (dashboard or API) before re-triggering
- [ ] `state["pending_approval"]["status"]` = `"approved"` or `"rejected"` on resume

---

## 16. ParallelAgent — Concurrent Sub-Agent Execution

`ParallelAgent` runs multiple sub-agents **simultaneously** within a single turn. Use it when tasks are independent and can be done at the same time to reduce wall-clock latency.

### When to Use ParallelAgent vs. SequentialAgent

| Pattern | Use When |
|---------|----------|
| `SequentialAgent` | Tasks depend on each other (Planner → Worker → Checker) |
| `ParallelAgent` | Tasks are independent (e.g., fetch logs + query DB + call API simultaneously) |
| `LoopAgent` | Same pipeline repeated until done (Ralph Loop) |

### Basic Pattern

```python
from google.adk.agents import Agent, ParallelAgent, SequentialAgent, LoopAgent
from google.adk.tools import ToolContext

# Independent sub-agents that can run simultaneously
log_analyzer = Agent(
    name="log_analyzer",
    model="gemini-2.5-flash",
    instruction="""
    Fetch and analyze the last 100 Cloud Logging error entries for the payment service.
    Write your findings to state['log_analysis'].
    """,
    tools=[fetch_cloud_logs],
    output_key="log_analysis",
)

metrics_checker = Agent(
    name="metrics_checker",
    model="gemini-2.5-flash",
    instruction="""
    Query Cloud Monitoring for the payment service's error rate and latency P99
    for the past 1 hour. Write findings to state['metrics_report'].
    """,
    tools=[query_cloud_monitoring],
    output_key="metrics_report",
)

db_inspector = Agent(
    name="db_inspector",
    model="gemini-2.5-flash",
    instruction="""
    Run a health check on the AlloyDB payment_transactions table.
    Count failed transactions in the last 30 minutes.
    Write findings to state['db_report'].
    """,
    tools=[run_db_query],
    output_key="db_report",
)

# ParallelAgent: all three run at the same time
parallel_diagnostics = ParallelAgent(
    name="parallel_diagnostics",
    sub_agents=[log_analyzer, metrics_checker, db_inspector],
)
```

### Full Assembly — Fan-Out / Fan-In Pattern

The recommended pattern is: **ParallelAgent** (gather data) → **Synthesizer Agent** (combine results).

```python
# agents/agent.py — Parallel diagnostics with synthesizer
from google.adk.agents import Agent, ParallelAgent, SequentialAgent, LoopAgent
from google.adk.tools import ToolContext

# --- Parallel fan-out ---
parallel_diagnostics = ParallelAgent(
    name="parallel_diagnostics",
    sub_agents=[log_analyzer, metrics_checker, db_inspector],
)

# --- Synthesizer: reads all parallel results from State ---
SYNTHESIZER_INSTRUCTION = """
You are the synthesizer agent. All parallel diagnostics are complete.
Read state['log_analysis'], state['metrics_report'], and state['db_report'].

Produce a unified incident report in state['execution_result']:
{
  "status": "success",
  "result": "<comprehensive root-cause analysis with recommendations>",
  "evidence": {
    "logs": "<key findings from log_analysis>",
    "metrics": "<key findings from metrics_report>",
    "database": "<key findings from db_report>"
  }
}
"""

synthesizer = Agent(
    name="synthesizer",
    model="gemini-2.5-pro",  # Combine multiple sources → Pro for reasoning quality
    instruction=SYNTHESIZER_INSTRUCTION,
    output_key="execution_result",
)

# --- Checker: validate the synthesized result ---
checker = Agent(
    name="checker",
    model="gemini-2.5-flash",
    instruction=CHECKER_INSTRUCTION,
    tools=[escalate_issue],
)

# --- Pipeline: parallel gather → synthesize → check ---
pipeline = SequentialAgent(
    name="diagnostic_pipeline",
    sub_agents=[parallel_diagnostics, synthesizer, checker],
)

# --- Ralph Loop: repeat until done ---
ralph_loop = LoopAgent(
    name="diagnostic_loop",
    sub_agents=[pipeline],
    max_iterations=5,
)

root_agent = ralph_loop
```

### State Isolation in ParallelAgent

Each parallel sub-agent writes to its own `output_key`. They **must not write to the same state key** — race conditions will cause data loss.

```python
# ✅ Correct — unique output_key per sub-agent
log_analyzer    = Agent(..., output_key="log_analysis")
metrics_checker = Agent(..., output_key="metrics_report")
db_inspector    = Agent(..., output_key="db_report")

# ❌ Wrong — all writing to the same key causes race conditions
log_analyzer    = Agent(..., output_key="result")
metrics_checker = Agent(..., output_key="result")  # Will overwrite log_analyzer's output!
```

### Nested: LoopAgent Inside ParallelAgent

You can nest a `LoopAgent` inside `ParallelAgent` for cases where each parallel branch itself needs iterative refinement:

```python
# Each branch is an independent Ralph Loop
sre_loop      = LoopAgent(name="sre_loop",  sub_agents=[sre_pipeline],  max_iterations=3)
arch_loop     = LoopAgent(name="arch_loop", sub_agents=[arch_pipeline], max_iterations=3)

parallel_specialist_loops = ParallelAgent(
    name="parallel_specialist_loops",
    sub_agents=[sre_loop, arch_loop],
)
```

> **Warning**: Nesting loops inside parallel agents multiplies iterations. `ParallelAgent(2 branches) × LoopAgent(max_iterations=3)` = up to 6 total iterations running concurrently. Always set conservative `max_iterations` values.

### ParallelAgent Checklist

- [ ] Each sub-agent has a unique `output_key` — no shared state keys
- [ ] Sub-agents have no data dependency on each other (verify manually)
- [ ] A synthesizer agent follows the `ParallelAgent` in a `SequentialAgent`
- [ ] `max_iterations` is set on any nested `LoopAgent`
- [ ] Monitor concurrent API quota consumption — parallel calls multiply token usage per turn
