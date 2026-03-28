# Ralph Loop on Google Cloud — Implementation Guide

## Overview
Ralph Loop is an orchestration pattern for **long-running autonomous agent tasks**.
Its core principle is **Fresh Context Per Iteration** — each iteration starts a brand-new session, with continuity maintained exclusively through files.

This guide covers how to implement Ralph Loop using Google Cloud native services.

---

## 1. Five Core Principles of Ralph Loop

| Principle | Description | GCP Implementation |
|-----------|-------------|-------------------|
| **Fresh Context** | New session on every iteration | Agent Engine session isolation |
| **External Memory** | File-based continuity between sessions | Cloud Source Repos / GCS |
| **One Task Per Loop** | Only one task per iteration | `max_iterations` + Checker |
| **Backpressure** | Output validation and correction mechanism | Policy Engine + Eval |
| **Plan / Build Separation** | Planning Mode → Build Mode | Planner Agent → Worker Agent |

> **Warning**: Implementing Ralph Loop as a skill or command *inside* the Agent Harness breaks the Fresh Context principle.
> Ralph must always operate **outside** the Agent Harness.

---

## 2. GCP Architecture Mapping

```
[Cloud Scheduler / Cloud Build Trigger]  ← external trigger
        │
        ▼
[Cloud Run Job: Ralph Outer Loop]        ← equivalent to bash while loop
        │
        ├─ Iteration 1 → [Agent Engine: ADK Agent] → git commit
        ├─ Iteration 2 → [Agent Engine: ADK Agent] → git commit
        └─ Iteration N → [Agent Engine: ADK Agent] → done
        │
[Cloud Source Repositories / GitHub]     ← file-based external memory
  fix_plan.md │ specs/* │ AGENT.md
```

### Service Role Mapping

| Ralph Concept | GCP Service |
|---------------|-------------|
| `while :; do ... done` | Cloud Run Jobs (repeating container) |
| External trigger | Cloud Scheduler or Cloud Build Trigger |
| Fresh Context session | Vertex AI Agent Engine (guarantees session isolation) |
| `fix_plan.md` (task tracker) | Cloud Source Repos or GCS bucket |
| `specs/*` (requirements) | Cloud Source Repos directory |
| `git commit` (persistence) | Cloud Source Repositories / GitHub |

---

## 3. Ralph Loop Implementation in This Project

### `agents/ralph_loop.py` — ZDR State Manager

```python
class RalphLoopManager:
    """Manages Zero-Downtime Resilience (ZDR) state persistence."""

    def save_state(self, state: dict):
        """Persists intermediate state — enables recovery after container restart."""
        with open(self.state_file, 'w') as f:
            json.dump(state, f)

    def load_state(self) -> dict:
        """Rehydrates state — the core of ZDR."""
        if os.path.exists(self.state_file):
            with open(self.state_file, 'r') as f:
                return json.load(f)
        return {}
```

### `agents/agent.py` — ADK LoopAgent Orchestration

```python
# Sequential pipeline: Planner → Worker → Checker
pipeline = SequentialAgent(
    name="enterprise_pipeline",
    sub_agents=[planner, worker, checker]
)

# Runs the pipeline repeatedly (inner loop of Ralph Loop)
ralph_loop = LoopAgent(
    name="enterprise_ralph_loop",
    sub_agents=[pipeline],
    max_iterations=10,  # Enterprise cost-control guardrail
)
```

### How the Checker Terminates the Loop

```python
def escalate_issue(reason: str, tool_context: ToolContext) -> str:
    """Goal achieved or fatal error → terminate the loop."""
    tool_context.actions.escalate = True
    return f"Loop terminated: {reason}"
```

The Checker calls `escalate_issue` under one of these conditions:
1. **Goal achieved**: `execution_result` satisfies all requirements
2. **Fatal error**: An unrecoverable error has occurred
3. **Iteration limit**: `max_iterations` reached (handled automatically by ADK)

---

## 4. Backpressure Mechanism

Backpressure = a feedback loop that validates agent output and forces correction

```
Worker executes → output → Checker evaluates
                                  │
              ┌──── Pass ──────────┤
              │                    │
              ▼                    └──── Fail → update plan → re-run Worker
          escalate()
          (loop exits)
```

### Backpressure via Policy Engine

```python
# tools/policy_engine.py
import re

class PolicyEngine:
    """SRE Interception Layer: validates commands before tool execution."""

    def __init__(self):
        self.forbidden_patterns = [
            r"rm -rf",
            r"delete cluster",
            r"drop table",
            r"shutdown",
        ]

    def validate_command(self, command: str) -> bool:
        """Returns False and logs if the command matches a forbidden pattern."""
        for pattern in self.forbidden_patterns:
            if re.search(pattern, command, re.IGNORECASE):
                return False  # Backpressure triggered
        return True

    def interception_wrapper(self, func):
        """Decorator form of the interception layer — wraps any tool function."""
        def wrapper(*args, **kwargs):
            command = kwargs.get("command") or args[0]
            if self.validate_command(command):
                return func(*args, **kwargs)
            raise PermissionError("Command blocked by Policy Engine.")
        return wrapper


# Global singleton instance — shared across all tools
global_policy = PolicyEngine()
```

Usage example:
```python
# tools/any_tool.py
from tools.policy_engine import global_policy

def execute_shell(command: str, tool_context) -> str:
    if not global_policy.validate_command(command):
        raise PermissionError(f"Policy violation: {command}")
    # ... execute

# Or using the decorator form
@global_policy.interception_wrapper
def execute_shell(command: str) -> str:
    # ... execute
    pass
```

Policy Engine blocks → retry signal sent to Worker → Backpressure loop activates

---

## 5. File-Based External Memory Structure

In Ralph Loop, cross-session continuity is passed exclusively through files:

```
.agent/
├── PLAN.md            ← Full plan (output of Planning Mode)
├── ACTION.md          ← Execution log (updated every iteration)
├── DEFINE.md          ← Task definition list (plays the role of fix_plan.md)
├── project_context.md ← Project context (plays the role of AGENT.md)
├── rules/             ← Behavioral rules
├── skills/            ← Domain knowledge (this file)
└── workflows/         ← Automated workflows
```

**Ralph execution flow**:
1. Start of each iteration: read `project_context.md` + `DEFINE.md`
2. Check completed tasks: inspect `ACTION.md`
3. Select next task: pick the first incomplete item in `DEFINE.md`
4. After execution: update `ACTION.md`, git commit

---

## 6. Correspondence with the Anthropic Pattern (Reference)

| Anthropic Pattern | This Project's Implementation |
|-------------------|-------------------------------|
| Initializer Agent (Planning Mode) | `planner` Agent + PLAN.md |
| Coding Agent (Build Mode) | `worker` Agent + ACTION.md |
| `feature_list.json` | `DEFINE.md` |
| `claude-progress.txt` | `ACTION.md` |
| git commit (after each feature) | Cloud Source Repos integration |

---

## 7. Debugging and Monitoring

```bash
# Enable telemetry (agents/.env)
GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
```

- **Cloud Trace**: Visualizes the agent's reasoning path for each loop iteration
- **Cloud Logging**: Captures `save_state()` calls and Policy violations
- **Agent Engine Dashboard**: Tracks token usage and latency per iteration

### Infinite-Loop Prevention Checklist
- [ ] Set `max_iterations` (recommended: 10)
- [ ] Register `escalate_issue` tool on the Checker agent
- [ ] Define forbidden patterns in Policy Engine
- [ ] ZDR: call `save_state()` after every completed iteration

---

## 8. Gemini CLI Integration — `gemini-cli-extensions/ralph`

The `ralph` extension brings the Ralph Loop pattern directly into Gemini CLI as a slash command, replacing the manual `while :; do ... done` bash loop.

### Install
```bash
gemini extensions install https://github.com/gemini-cli-extensions/ralph
```

### Usage
```bash
# Start an autonomous loop with a task description
/ralph:loop "Implement all pending items in DEFINE.md one by one" --max-iterations 10

# Stop a running loop
/ralph:cancel

# Loop with explicit completion signal
/ralph:loop "Refactor agents/harness.py" \
  --max-iterations 5 \
  --completion-promise "<task-complete/>"
```

### How It Maps to the GCP Architecture

| GCP Architecture | `ralph` Extension Equivalent |
|-----------------|------------------------------|
| Cloud Run Job (outer loop container) | `AfterAgent` hook (`hooks/stop-hook.sh`) |
| Agent Engine session per iteration | Cleared conversational context each turn |
| Cloud Scheduler trigger | Manual `/ralph:loop` invocation or CI trigger |
| `save_state()` → GCS / file | Stable prompt file read each iteration |
| Ghost protection (Cloud Run restart detection) | Prompt mismatch detection on interruption |

### Key Behavioral Differences

| Behavior | Manual GCP Loop | `ralph` Extension |
|----------|----------------|-------------------|
| Context isolation | New Agent Engine session per Cloud Run Job | Cleared context each turn via `AfterAgent` hook |
| Iteration control | `max_iterations` in `LoopAgent` | `--max-iterations` flag |
| Termination signal | `escalate_issue` tool → `actions.escalate` | `--completion-promise` XML tag in output |
| External memory | Cloud Source Repos / GCS | Local file system (same working directory) |
| Trigger mechanism | Cloud Scheduler / Cloud Build | Manual CLI or CI pipeline |

> **When to use which**: Use the `ralph` extension for local development and prototyping (Tier 1). Use the full GCP architecture (Cloud Scheduler → Cloud Run Jobs → Agent Engine) for production autonomous pipelines (Tier 2/3).

---

## 9. Production Outer Loop — Cloud Run Job + Cloud Scheduler

If the `ralph` extension is the local `while` loop, then in production the **Cloud Run Job is the outer loop container** and **Cloud Scheduler is the trigger**. Each execution processes one task and exits — this is the GCP implementation of the "One Task Per Loop" principle.

```
Cloud Scheduler (cron)
    │  Runs every hour
    ▼
Cloud Run Job: ralph-outer-loop
    │  Selects 1 incomplete task from DEFINE.md
    │  Calls Agent Engine (new session = Fresh Context)
    │  Updates ACTION.md / marks task complete in DEFINE.md
    │  Container exits upon completion
    ▼
Cloud Source Repositories / GCS (external memory)
```

### `ci-cd/outer_loop.py` — Outer Loop Script (GCS Backend)

> **Key point**: Cloud Run Job starts a new container on every execution. Local files do not persist between runs. `DEFINE.md` and `ACTION.md` must be **stored in GCS** to persist state across containers.

```python
# ci-cd/outer_loop.py
"""
Ralph Loop outer loop — container script executed as a Cloud Run Job.
Stores external memory (DEFINE.md, ACTION.md) in GCS to persist state across containers.
"""
import os
import sys
from datetime import datetime
from google.cloud import storage
from vertexai import agent_engines   # GA path: vertexai>=1.71 (google-cloud-aiplatform[adk,agent_engines])
# ⚠️ Legacy SDK: from vertexai.preview import reasoning_engines  ← Deprecated

PROJECT_ID             = os.environ["GOOGLE_CLOUD_PROJECT"]
LOCATION               = os.environ.get("GOOGLE_CLOUD_LOCATION", "global")  # Gemini model endpoint — must be global
# NOTE: LOCATION is not passed directly to agent_engines calls here.
# Agent Engine SDK reads GOOGLE_CLOUD_LOCATION from the environment automatically
# via the vertexai.init() / ADK runtime initialization.
AGENT_ENGINE_ID        = os.environ["AGENT_ENGINE_ID"]
EXTERNAL_MEMORY_BUCKET = os.environ["EXTERNAL_MEMORY_BUCKET"]  # GCS bucket name
DEFINE_BLOB            = os.environ.get("DEFINE_BLOB", "agent-memory/DEFINE.md")
ACTION_BLOB            = os.environ.get("ACTION_BLOB", "agent-memory/ACTION.md")

_storage = storage.Client()


def _read_gcs(blob_name: str) -> str:
    """Reads file contents from GCS."""
    return _storage.bucket(EXTERNAL_MEMORY_BUCKET).blob(blob_name).download_as_text()


def _write_gcs(blob_name: str, content: str) -> None:
    """Writes file contents to GCS (overwrites existing content)."""
    _storage.bucket(EXTERNAL_MEMORY_BUCKET).blob(blob_name).upload_from_string(
        content, content_type="text/plain"
    )


def read_pending_tasks() -> list[str]:
    """Returns incomplete tasks (- [ ] ...) from DEFINE.md in GCS."""
    content = _read_gcs(DEFINE_BLOB)
    return [
        line.strip()[6:].strip()
        for line in content.splitlines()
        if line.strip().startswith("- [ ]")
    ]


def mark_task_done(task: str) -> None:
    """Marks the given task as complete (- [x]) in DEFINE.md in GCS."""
    content = _read_gcs(DEFINE_BLOB)
    # Line-by-line replace: strip() handles any leading indentation in DEFINE.md
    lines = content.splitlines(keepends=True)
    replaced = False
    for i, line in enumerate(lines):
        if line.strip() == f"- [ ] {task}" and not replaced:
            lines[i] = line.replace("- [ ]", "- [x]", 1)
            replaced = True
    _write_gcs(DEFINE_BLOB, "".join(lines))


def append_action_log(task: str, result: str) -> None:
    """Appends execution result and timestamp to ACTION.md in GCS."""
    try:
        existing = _read_gcs(ACTION_BLOB)
    except Exception:
        existing = ""  # Create new file if ACTION.md does not yet exist
    entry = (
        f"\n## {datetime.now().strftime('%Y-%m-%d %H:%M')} — {task}\n"
        f"{result}\n"
    )
    _write_gcs(ACTION_BLOB, existing + entry)


def run_one_iteration(task: str) -> str:
    """
    Executes a single task on Agent Engine.
    Generates an independent session ID per task → guarantees Fresh Context Per Iteration.
    """
    # GA SDK: vertexai.agent_engines (vertexai.preview.reasoning_engines is deprecated)
    agent = agent_engines.get_reasoning_engine(AGENT_ENGINE_ID)
    # WARNING: hash() is non-deterministic across processes (PYTHONHASHSEED).
    # For a stable session ID, use: hashlib.md5(task.encode()).hexdigest()[:8]
    session_id = f"ralph-{abs(hash(task)):08x}"
    response = agent.query(input=task, session_id=session_id)
    return str(response)


def main():
    pending = read_pending_tasks()

    if not pending:
        print("All tasks in DEFINE.md are completed. Outer loop exits.")
        sys.exit(0)

    # One Task Per Loop — process only the first incomplete task
    task = pending[0]
    print(f"[outer_loop] Processing: {task}")

    result = run_one_iteration(task)

    mark_task_done(task)
    append_action_log(task, result)

    remaining = len(pending) - 1
    print(f"[outer_loop] Done. {remaining} task(s) remaining.")


if __name__ == "__main__":
    main()
```

### `ci-cd/Dockerfile.outer_loop` — Container Image

```dockerfile
# ci-cd/Dockerfile.outer_loop
FROM python:3.13-slim
WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt --quiet

COPY . .

CMD ["python", "ci-cd/outer_loop.py"]
```

### Step 0 — Set Up GCS External Memory Bucket (One-Time Setup)

```bash
# Create GCS bucket (separate from Terraform state bucket)
MEMORY_BUCKET="${PROJECT_ID}-agent-memory"
gcloud storage buckets create gs://$MEMORY_BUCKET \
  --location=$REGION \
  --project=$PROJECT_ID

# Grant read/write permissions to service account
gcloud storage buckets add-iam-policy-binding gs://$MEMORY_BUCKET \
  --member="serviceAccount:agent-harness-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectUser"

# Upload initial DEFINE.md to GCS (after writing the task list)
gcloud storage cp .agent/DEFINE.md gs://$MEMORY_BUCKET/agent-memory/DEFINE.md
```

### Step 1 — Build & Push Container Image

```bash
IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/agent-harness-repo/ralph-outer-loop:latest"

docker build -f ci-cd/Dockerfile.outer_loop -t $IMAGE .
docker push $IMAGE
```

### Step 2 — Create Cloud Run Job

```bash
gcloud run jobs create ralph-outer-loop \
  --image=$IMAGE \
  --region=$REGION \
  --service-account=agent-harness-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  --set-env-vars="GOOGLE_CLOUD_PROJECT=${PROJECT_ID},GOOGLE_CLOUD_LOCATION=global,AGENT_ENGINE_ID=${AGENT_ENGINE_ID},EXTERNAL_MEMORY_BUCKET=${PROJECT_ID}-agent-memory" \
  # ↑ GOOGLE_CLOUD_LOCATION=global is fixed — Gemini model endpoint. Do not mix with ${REGION} (deployment region)
  --max-retries=2 \
  --parallelism=1 \
  --project=$PROJECT_ID

# Run manually to verify behavior
gcloud run jobs execute ralph-outer-loop --region=$REGION --project=$PROJECT_ID
```

### Step 3 — Set Up Cloud Scheduler Trigger

```bash
# Grant Cloud Run Jobs execution permission (service account needs roles/run.invoker)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:agent-harness-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# Create Scheduler — runs at the top of every hour
gcloud scheduler jobs create http ralph-loop-trigger \
  --location=$REGION \
  --schedule="0 * * * *" \
  --uri="https://${REGION}-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/${PROJECT_ID}/jobs/ralph-outer-loop:run" \
  --message-body='{}' \
  --oidc-service-account-email="agent-harness-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project=$PROJECT_ID

# Run Scheduler immediately (for testing)
gcloud scheduler jobs run ralph-loop-trigger --location=$REGION --project=$PROJECT_ID
```

### Full Execution Flow Summary

```
1. Cloud Scheduler triggers Cloud Run Job every hour
2. outer_loop.py runs → selects 1 incomplete task from DEFINE.md
3. Calls Agent Engine (ADK LoopAgent) → new session (Fresh Context)
4. Records result in ACTION.md, marks task complete in DEFINE.md
5. Container exits (exit 0)
6. When all tasks are complete, exit 0 → Scheduler repeats an empty job
```

### Production Outer Loop Checklist

- [ ] Create GCS external memory bucket and grant service account permissions
- [ ] Upload initial `DEFINE.md` to GCS (`gcloud storage cp`)
- [ ] Set `AGENT_ENGINE_ID` / `EXTERNAL_MEMORY_BUCKET` environment variables
- [ ] Add `google-cloud-storage` package to `requirements.txt`
- [ ] Run Cloud Run Job manually for the first time to verify GCS read/write behavior
- [ ] Create Cloud Scheduler trigger (adjust execution frequency based on task complexity)
- [ ] Verify Cloud Run Job logs in Cloud Logging
- [ ] Establish a procedure to disable Scheduler or refresh DEFINE.md after all tasks are complete
