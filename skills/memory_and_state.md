# Memory & State Management Guide

## Overview
An Agent Harness requires two distinct memory layers:
- **Short-term memory**: conversation history and task state within a session (ADK Session + State)
- **Long-term memory**: knowledge that persists across sessions and agents (Memory Bank)

And one resilience mechanism:
- **ZDR (Zero-Downtime Resilience)**: persisting intermediate state so any container restart can resume from the last checkpoint

Related extensions: [`mcp-toolbox`](https://github.com/gemini-cli-extensions/mcp-toolbox), [`genai-toolbox`](https://github.com/googleapis/genai-toolbox), [`alloydb`](https://github.com/gemini-cli-extensions/alloydb), [`cloud-sql-postgresql`](https://github.com/gemini-cli-extensions/cloud-sql-postgresql)

---

## 1. Short-term Memory — ADK Session & State

### Session Service Options

| Service | Environment | Notes |
|---------|-------------|-------|
| `InMemorySessionService` | Local dev / testing | Lost on restart — never use in production |
| `DatabaseSessionService` (Cloud SQL) | Production | ADK-native, horizontally scalable |
| `VertexAiSessionService` | Agent Engine (built-in) | Zero-config when deployed to Agent Engine |
| Firestore | Production (serverless) | Auto-scaling, no connection pool management |
| Memorystore (Redis) | Production (low-latency) | Sub-millisecond reads, ideal for high-frequency state updates |

### DatabaseSessionService with Cloud SQL (Production Standard)

```python
# agents/session.py
import os
from google.adk.sessions import DatabaseSessionService

def create_session_service() -> DatabaseSessionService:
    """
    Creates a production-grade session service backed by Cloud SQL PostgreSQL.
    Connection string is injected via Secret Manager at runtime.
    """
    db_url = os.environ.get("DATABASE_URL")
    # Format: postgresql+pg8000://user:password@host:5432/dbname
    # For Cloud SQL via Unix socket: postgresql+pg8000://user:pass@/dbname?unix_sock=/cloudsql/PROJECT:REGION:INSTANCE
    return DatabaseSessionService(db_url=db_url)

session_service = create_session_service()
```

### State Scopes in Practice

```python
from google.adk.agents import Agent
from google.adk.tools import ToolContext

# Within a tool or agent callback:
def update_loop_progress(task_id: str, result: str, tool_context: ToolContext) -> str:
    """Records task completion in ADK State at multiple scopes."""

    # Session scope — current loop iteration only
    tool_context.state["current_task"] = task_id
    tool_context.state["execution_result"] = result

    # User scope — survives across sessions for the same user
    completed = tool_context.state.get("user:completed_tasks", [])
    completed.append(task_id)
    tool_context.state["user:completed_tasks"] = completed

    # App scope — shared read-only config available to all agents
    max_iter = tool_context.state.get("app:max_iterations", 10)

    # Temp scope — discarded at end of turn, use for scratch work
    tool_context.state["temp:last_tool_output"] = result

    return f"Progress recorded: {task_id} completed ({len(completed)} total)"
```

---

## 2. Long-term Memory — Memory Bank

Memory Bank is Google Cloud's managed long-term memory service. It stores and retrieves facts across sessions and agents, enabling knowledge accumulation over the Ralph Loop lifecycle.

### Setup via `infra/memory_bank_config.py`

```python
# infra/memory_bank_config.py
import vertexai
from vertexai.preview import reasoning_engines
# ⚠️ MemoryBank is currently only available in vertexai.preview.
# Check https://cloud.google.com/vertex-ai/docs/release-notes for GA promotion status.
# ReasoningEngine (Agent Engine calls) is GA: from vertexai import agent_engines

def create_memory_bank(project_id: str, location: str = "us-central1") -> str:
    """
    Provisions a Memory Bank instance for the Agent Harness.
    Run once during infrastructure setup.
    Returns the Memory Bank resource name.
    """
    vertexai.init(project=project_id, location=location)

    memory_bank = reasoning_engines.MemoryBank.create(
        display_name="agent-harness-memory-bank",
        description="Long-term memory for Agent Harness — persists knowledge across Ralph Loop sessions",
    )
    print(f"Memory Bank created: {memory_bank.resource_name}")
    return memory_bank.resource_name
```

### Memory Bank API Patterns

```python
# agents/memory.py
import os
import vertexai
from vertexai.preview import reasoning_engines   # MemoryBank is still in preview
# ⚠️ May move to vertexai.agent_engines or vertexai.memory_banks on SDK updates.

MEMORY_BANK_ID = os.environ.get("MEMORY_BANK_ID")  # set in agents/.env

def add_memory(content: str, session_id: str) -> None:
    """
    Stores a fact or task result in long-term Memory Bank.
    Call at the end of each Ralph Loop iteration to accumulate knowledge.
    """
    memory_bank = reasoning_engines.MemoryBank(MEMORY_BANK_ID)
    memory_bank.add_memories(
        user_id=session_id,
        messages=[{"role": "assistant", "content": content}]
    )

def search_memory(query: str, session_id: str, top_k: int = 5) -> list[str]:
    """
    Retrieves relevant past knowledge before starting a new task.
    Inject results into the Worker agent's context via ADK State.
    """
    memory_bank = reasoning_engines.MemoryBank(MEMORY_BANK_ID)
    results = memory_bank.generate_memories(
        user_id=session_id,
        query=query,
        top_k=top_k
    )
    return [r.content for r in results.memories]
```

### When to Use Memory Bank vs. ADK State

| Scenario | Use |
|----------|-----|
| Store task result for the next turn in the same session | ADK `state["execution_result"]` |
| Track which tasks have been completed across all Ralph Loop sessions | Memory Bank (`add_memory`) |
| Retrieve relevant past decisions before planning a new task | Memory Bank (`search_memory`) |
| Share configuration constants across all agents | ADK `state["app:key"]` |
| Cache a temporary computation result within one turn | ADK `state["temp:key"]` |

---

## 3. ZDR — Zero-Downtime Resilience

ZDR ensures that a container restart (Cloud Run scale-down, Agent Engine instance replacement) does not lose progress. The pattern: **persist → restart → rehydrate → continue**.

### `agents/ralph_loop.py` — ZDR State Manager

```python
import json
import os
from dataclasses import dataclass, asdict
from typing import Optional

@dataclass
class LoopState:
    session_id: str
    current_task_index: int
    completed_tasks: list[str]
    last_result: Optional[str] = None
    iteration_count: int = 0

class RalphLoopManager:
    """Manages ZDR state persistence for the Ralph Loop."""

    def __init__(self, state_file: str = ".agent_state.json"):
        self.state_file = state_file

    def save_state(self, state: LoopState) -> None:
        """Persist loop state after every completed iteration."""
        with open(self.state_file, "w") as f:
            json.dump(asdict(state), f, indent=2)

    def load_state(self) -> Optional[LoopState]:
        """Rehydrate state on container restart. Returns None if no state exists."""
        if not os.path.exists(self.state_file):
            return None
        with open(self.state_file, "r") as f:
            data = json.load(f)
        return LoopState(**data)

    def clear_state(self) -> None:
        """Remove state file after successful completion."""
        if os.path.exists(self.state_file):
            os.remove(self.state_file)
```

### ZDR Integration in the Ralph Loop

```python
# agents/agent.py (with ZDR)
from agents.ralph_loop import RalphLoopManager, LoopState
from agents.memory import add_memory, search_memory

manager = RalphLoopManager()

def run_loop(session_id: str, tasks: list[str]) -> None:
    # Rehydrate — resume from last checkpoint if restarted
    state = manager.load_state() or LoopState(
        session_id=session_id,
        current_task_index=0,
        completed_tasks=[],
    )

    # Inject past memory into the Planner's context
    past_knowledge = search_memory(
        query=tasks[state.current_task_index],
        session_id=session_id
    )

    # Execute one task (ADK LoopAgent handles the Planner→Worker→Checker cycle)
    # ... (ADK runner invocation)

    # Persist after successful completion
    state.completed_tasks.append(tasks[state.current_task_index])
    state.current_task_index += 1
    state.iteration_count += 1
    manager.save_state(state)

    # Store result in long-term Memory Bank
    add_memory(
        content=f"Completed: {tasks[state.current_task_index - 1]}\nResult: {state.last_result}",
        session_id=session_id
    )
```

---

## 4. Database-Backed State via genai-toolbox

For enterprise deployments, replace the JSON file with a managed database using `genai-toolbox`.

### Install Extensions
```bash
# Gemini CLI development environment
gemini extensions install https://github.com/gemini-cli-extensions/mcp-toolbox

# For AlloyDB-backed state
gemini extensions install https://github.com/gemini-cli-extensions/alloydb

# For Cloud SQL PostgreSQL-backed state
gemini extensions install https://github.com/gemini-cli-extensions/cloud-sql-postgresql
```

### Create the Agent State Table

```sql
-- Run once during infra setup (via alloydb or cloud-sql-postgresql extension)
CREATE TABLE IF NOT EXISTS agent_state (
    session_id      TEXT NOT NULL,
    task_id         TEXT NOT NULL,
    status          TEXT NOT NULL CHECK (status IN ('pending', 'running', 'completed', 'failed')),
    result          TEXT,
    iteration_count INT DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (session_id, task_id)
);

CREATE INDEX idx_agent_state_session ON agent_state(session_id);
```

With the `alloydb` extension:
```
gemini "create a table called agent_state in my AlloyDB database with columns for session_id, task_id, status, result, iteration_count, and timestamps. Use (session_id, task_id) as the primary key."
```

### `tools.yaml` for Agent State Tools

```yaml
# tools.yaml (project root — auto-discovered by mcp-toolbox extension)
sources:
  agent-state-db:
    kind: alloydb-postgres
    project: ${GOOGLE_CLOUD_PROJECT}
    region: us-central1
    cluster: agent-harness-cluster
    instance: agent-harness-instance
    database: harness_db

tools:
  get-loop-state:
    kind: postgres-sql
    source: agent-state-db
    description: "Get the current Ralph Loop state for a session — use at iteration start for ZDR rehydration"
    parameters:
      - name: session_id
        type: string
    sql: >
      SELECT task_id, status, result, iteration_count
      FROM agent_state
      WHERE session_id = $session_id AND status != 'completed'
      ORDER BY created_at ASC LIMIT 1;

  save-loop-state:
    kind: postgres-sql
    source: agent-state-db
    description: "Persist task result after completion — required for ZDR"
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
      ON CONFLICT (session_id, task_id)
      DO UPDATE SET status=$status, result=$result, updated_at=NOW();

  list-completed-tasks:
    kind: postgres-sql
    source: agent-state-db
    description: "List all completed tasks for a session — use by Planner to avoid re-doing work"
    parameters:
      - name: session_id
        type: string
    sql: >
      SELECT task_id, result FROM agent_state
      WHERE session_id = $session_id AND status = 'completed'
      ORDER BY updated_at ASC;
```

---

## 5. Memory Architecture Decision Tree

```
Do you need memory across multiple users?
│
├─ YES → Memory Bank (long-term, semantic search)
│
└─ NO → Do you need it across multiple sessions for the same user?
         │
         ├─ YES → Memory Bank (user scope) or DatabaseSessionService
         │
         └─ NO → Is it needed across multiple turns in one session?
                  │
                  ├─ YES → ADK State (session scope): state["key"]
                  │
                  └─ NO → Is it computation scratch within one turn?
                           │
                           ├─ YES → ADK State (temp scope): state["temp:key"]
                           │
                           └─ NO → Local variable (no persistence needed)
```

## 6. Environment Variables for Memory

```bash
# agents/.env
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=global

# Memory Bank
MEMORY_BANK_ID=projects/YOUR_PROJECT/locations/us-central1/reasoningEngines/YOUR_MEMORY_BANK_ID

# DatabaseSessionService (Cloud SQL / AlloyDB)
DATABASE_URL=postgresql+pg8000://user:password@/dbname?unix_sock=/cloudsql/PROJECT:REGION:INSTANCE

# ZDR state file location (local dev)
AGENT_STATE_FILE=.agent_state.json
```
