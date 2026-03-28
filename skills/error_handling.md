# Error Handling Guide — Failure Patterns & Recovery

## Overview
Agent Harnesses fail in predictable ways. This guide catalogs the failure modes specific to the GCP Agent Harness stack and provides concrete handling patterns for each.

Related extensions: [`observability`](https://github.com/gemini-cli-extensions/observability), [`security`](https://github.com/gemini-cli-extensions/security)

---

## 1. Failure Mode Taxonomy

| Failure Type | Where It Occurs | Handling Strategy |
|-------------|-----------------|-------------------|
| Tool execution error | `tools/*.py` | Retry with backoff → escalate after N attempts |
| Model refusal / safety block | `agents/harness.py` | Rewrite prompt → reduce scope → escalate |
| **Model Armor block** | `tools/model_armor.py` | Return `model_armor_block` status — do not retry; log and escalate |
| Policy Engine violation | `tools/policy_engine.py` | Block and inform — do not retry |
| MCP server unreachable | `tools/mcp_client.py` | Retry with exponential backoff → circuit breaker |
| ADK State write failure | Any agent turn | Log + continue with in-memory fallback |
| Agent Engine timeout | `agents/agent.py` | ZDR save → restart → rehydrate |
| `max_iterations` reached | `LoopAgent` | Log warning + alert → human review |
| Memory Bank write failure | `agents/memory.py` | Log + continue — non-blocking |

---

## 2. Tool Error Handling Pattern

The standard pattern for any tool function: validate → execute → catch → retry or escalate.

```python
# tools/base_tool.py
import asyncio
import time
import logging
from functools import wraps
from typing import Callable
from google.adk.tools import ToolContext

logger = logging.getLogger(__name__)

def with_retry(max_attempts: int = 3, backoff_seconds: float = 2.0):
    """
    Decorator: retries a **synchronous** tool function with exponential backoff.
    Does NOT retry on PolicyEngine violations or permanent errors.

    Use `async_with_retry` for `async def` tool functions (I/O-bound, MCP calls, etc.).
    Applying this decorator to an async function will NOT work — it returns a coroutine
    object instead of awaiting it.
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except PermissionError:
                    # Policy violation — never retry
                    raise
                except Exception as e:
                    last_error = e
                    wait = backoff_seconds * (2 ** (attempt - 1))
                    logger.warning(
                        f"[{func.__name__}] attempt {attempt}/{max_attempts} failed: {e}. "
                        f"Retrying in {wait:.1f}s..."
                    )
                    if attempt < max_attempts:
                        time.sleep(wait)
            logger.error(f"[{func.__name__}] all {max_attempts} attempts failed.")
            raise last_error
        return wrapper
    return decorator


def async_with_retry(max_attempts: int = 3, backoff_seconds: float = 2.0):
    """
    Decorator: retries an **async** tool function with exponential backoff.
    Use for `async def` tools (MCP calls, A2A calls, async DB queries, etc.).

    Example:
        @async_with_retry(max_attempts=3, backoff_seconds=1.5)
        async def call_remote_mcp(...): ...
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return await func(*args, **kwargs)
                except PermissionError:
                    raise
                except Exception as e:
                    last_error = e
                    wait = backoff_seconds * (2 ** (attempt - 1))
                    logger.warning(
                        f"[{func.__name__}] attempt {attempt}/{max_attempts} failed: {e}. "
                        f"Retrying in {wait:.1f}s..."
                    )
                    if attempt < max_attempts:
                        await asyncio.sleep(wait)   # non-blocking sleep
            logger.error(f"[{func.__name__}] all {max_attempts} attempts failed.")
            raise last_error
        return wrapper
    return decorator
```

### Apply to any tool:

```python
# tools/grounding.py — sync tool
from tools.base_tool import with_retry
from google.adk.tools import ToolContext

@with_retry(max_attempts=3, backoff_seconds=1.5)
def search_knowledge_base(query: str, tool_context: ToolContext) -> str:
    """
    Searches the enterprise knowledge base (RAG).
    Retries automatically on transient network errors.
    """
    # ... implementation
    pass


# tools/mcp_client.py — async tool (I/O-bound: use async_with_retry)
from tools.base_tool import async_with_retry

@async_with_retry(max_attempts=3, backoff_seconds=1.5)
async def call_remote_mcp_safe(tool_name: str, arguments: dict, tool_context: ToolContext) -> str:
    """Calls a Remote MCP tool with automatic async retry."""
    # ... implementation
    pass
```

---

## 3. MCP Server Unreachable — Circuit Breaker

When a Remote MCP server is down, the agent should degrade gracefully rather than hammering a failed endpoint.

```python
# tools/mcp_client.py
import asyncio
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"       # normal operation
    OPEN = "open"           # server unreachable, fail fast
    HALF_OPEN = "half_open" # testing recovery

class MCPCircuitBreaker:
    def __init__(self, failure_threshold: int = 3, recovery_timeout: float = 30.0):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time: float = 0.0
        self.state = CircuitState.CLOSED

    def call_allowed(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                return True
            return False
        return True  # HALF_OPEN — allow one probe

    def record_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Global circuit breaker instance per MCP server
_circuit = MCPCircuitBreaker(failure_threshold=3, recovery_timeout=30.0)

async def call_remote_mcp_safe(tool_name: str, arguments: dict) -> str:
    """MCP call with circuit breaker — fails fast when server is down."""
    from mcp.client.sse import sse_client
    from mcp import ClientSession
    import os

    if not _circuit.call_allowed():
        return f"[Circuit OPEN] MCP server unavailable. Retrying after {_circuit.recovery_timeout}s."

    try:
        async with sse_client(os.environ["MCP_SERVER_URL"]) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                result = await session.call_tool(tool_name, arguments)
                _circuit.record_success()
                return str(result)
    except Exception as e:
        _circuit.record_failure()
        raise RuntimeError(f"MCP tool '{tool_name}' failed: {e}") from e
```

---

## 4. Error Propagation Between Agents

When a Worker fails, the Checker must receive structured error information to decide whether to retry or escalate.

```python
# agents/agent.py
import json
from google.adk.agents import Agent
from google.adk.tools import ToolContext

# Worker writes structured results — including errors — to State
WORKER_INSTRUCTION = """
You are the worker agent. Execute the task from state['plan'].

If a tool call fails:
1. Record the error in state['execution_result'] as JSON:
   {"status": "error", "tool": "<tool_name>", "error": "<message>", "retry_count": <n>}
2. If retry_count < 3, increment it and try an alternative approach.
3. If retry_count >= 3, set status to "fatal_error" — the Checker will escalate.

On success, write:
   {"status": "success", "result": "<summary>"}
"""

CHECKER_INSTRUCTION = """
You are the checker agent. Evaluate state['execution_result'].

Rules:
- If status == "success": verify quality. If acceptable, call escalate_issue("Goal achieved").
- If status == "error": update state['plan'] with a revised approach. Do NOT escalate.
- If status == "fatal_error": call escalate_issue("Fatal error: " + error message).
- If the same task has failed 3+ times: call escalate_issue("Max retries exceeded").
"""

def escalate_issue(reason: str, tool_context: ToolContext) -> str:
    """Terminates the loop. Called by Checker on success or unrecoverable failure."""
    tool_context.actions.escalate = True
    # Log the termination reason for observability
    import logging
    logging.getLogger(__name__).warning(f"Loop escalated: {reason}")
    return f"Loop terminated: {reason}"

checker = Agent(
    name="checker",
    model="gemini-3-flash-preview",
    instruction=CHECKER_INSTRUCTION,
    tools=[escalate_issue],
)
```

---

## 5. Model Refusal & Safety Block Handling

```python
# agents/harness.py
from google import genai
from google.genai import types
from google.api_core import exceptions as gcp_exceptions

class ModelHarness:
    def generate_safe(self, prompt: str, system_instruction: str = None) -> dict:
        """
        Generates content with graceful handling of safety blocks and refusals.
        Returns a structured dict with status and content.
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
                    ]
                ),
            )

            # Check for safety block (response exists but content was filtered)
            if not resp.candidates:
                return {"status": "safety_block", "result": None,
                        "reason": "Response blocked by safety filters"}

            candidate = resp.candidates[0]
            if candidate.finish_reason.name == "SAFETY":
                return {"status": "safety_block", "result": None,
                        "reason": str(candidate.safety_ratings)}

            return {"status": "success", "result": resp.text}

        except gcp_exceptions.ResourceExhausted as e:
            # Quota exceeded — back off and retry
            return {"status": "quota_exceeded", "result": None, "reason": str(e)}

        except gcp_exceptions.InvalidArgument as e:
            # Prompt too long or malformed
            return {"status": "invalid_prompt", "result": None, "reason": str(e)}
```

---

## 6. `max_iterations` Reached — Alert & Human Review

Hitting `max_iterations` is an abnormal condition. Set up a Cloud Monitoring alert:

```python
# infra/monitoring.py
from google.cloud import monitoring_v3
import os

def create_max_iterations_alert(project_id: str):
    """
    Creates a Cloud Monitoring alert that fires when max_iterations is reached
    without a successful escalate_issue("Goal achieved") call.
    Requires the loop to log a structured warning when it hits the limit.
    """
    client = monitoring_v3.AlertPolicyServiceClient()
    project_name = f"projects/{project_id}"

    policy = monitoring_v3.AlertPolicy(
        display_name="Agent Loop Max Iterations Reached",
        conditions=[
            monitoring_v3.AlertPolicy.Condition(
                display_name="Loop hit max_iterations",
                condition_matched_log=monitoring_v3.AlertPolicy.Condition.LogMatchCondition(
                    filter='resource.type="aiplatform.googleapis.com/ReasoningEngine" '
                           'jsonPayload.message=~"max_iterations"',
                ),
            )
        ],
        alert_strategy=monitoring_v3.AlertPolicy.AlertStrategy(
            notification_rate_limit=monitoring_v3.AlertPolicy.AlertStrategy.NotificationRateLimit(
                period={"seconds": 3600}  # max 1 alert per hour
            )
        ),
        combiner=monitoring_v3.AlertPolicy.ConditionCombinerType.OR,
    )
    client.create_alert_policy(name=project_name, alert_policy=policy)
```

---

## 7. Observability — Debugging Failures in Production

Install the `observability` extension to query logs, traces, and errors in natural language:

```bash
gemini extensions install https://github.com/gemini-cli-extensions/observability
# Requires: gcloud auth login + gcloud auth application-default login
```

### Common Debug Queries

```bash
# Find all failed tool calls in the last hour
gemini "show logs where tool call failed in the agent engine service in the last hour"

# Trace a specific session's reasoning path
gemini "show the trace for session_id abc-123 including all tool calls and their durations"

# Find patterns in errors
gemini "what are the most common error messages from the enterprise_ralph_loop agent today?"

# Check if max_iterations alert fired
gemini "were there any max_iterations alerts in Cloud Monitoring in the last 24 hours?"

# Identify slow tool calls
gemini "which tool calls took longer than 10 seconds in Cloud Trace this week?"
```

### Security Scan Before Deployment

```bash
gemini extensions install https://github.com/gemini-cli-extensions/security

# Scan the tools directory for vulnerabilities before deploying
/security:analyze
# → reviews git diff for OWASP top 10, injection risks, credential exposure

# JSON output for CI/CD gate
/security:analyze --json
```

---

## 8. Error Handling Checklist

- [ ] All tool functions use `@with_retry` for transient errors
- [ ] `PermissionError` from Policy Engine is never retried
- [ ] MCP client uses circuit breaker pattern
- [ ] Worker writes structured JSON to `state['execution_result']` (status + error fields)
- [ ] Checker reads error status and updates plan before retry
- [ ] Checker escalates on `fatal_error` or `retry_count >= 3`
- [ ] `max_iterations` alert configured in Cloud Monitoring
- [ ] `observability` extension installed for production debugging
- [ ] `/security:analyze` runs in CI before every deploy

---

## 9. Token / Cost Management

Four mechanisms for controlling costs in production:

| Mechanism | Implementation Location | Effect |
|---------|-----------|------|
| **Pre-flight token counting** | `tools/token_budget.py` | Blocks requests that exceed budget before sending |
| **Session token budget** | ADK State + `check_and_reserve_tokens()` | Cumulative token cap per session |
| **Model tier routing** | Planner instruction + `state['model_tier']` | Selects Flash / Pro based on complexity |
| **Iteration cap** | `LoopAgent(max_iterations=10)` | Directly limits loop costs |

---

### `tools/token_budget.py` — Pre-flight Token Counting + Budget Enforcement

```python
# tools/token_budget.py
import os
from google import genai
from google.adk.tools import ToolContext

# Session token budget cap — adjustable via environment variable
SESSION_TOKEN_BUDGET = int(os.environ.get("SESSION_TOKEN_BUDGET", "100000"))

# Input token price (USD / 1K tokens) — see cloud.google.com/vertex-ai/pricing
_COST_PER_1K_INPUT = {
    "gemini-3-flash-preview": 0.000075,   # $0.075 / 1M tokens
    "gemini-3-pro-preview":   0.00125,    # $1.25  / 1M tokens
}


def count_tokens(contents: str, model: str) -> int:
    """Counts tokens only, without making a model call."""
    client = genai.Client()
    response = client.models.count_tokens(model=model, contents=contents)
    return response.total_tokens


def estimate_cost_usd(tokens: int, model: str) -> float:
    """Calculates estimated cost (USD) based on input tokens."""
    rate = _COST_PER_1K_INPUT.get(model, 0.001)
    return round((tokens / 1000) * rate, 6)


def check_and_reserve_tokens(
    prompt: str,
    model: str,
    tool_context: ToolContext,
) -> dict:
    """
    Counts tokens for a prompt and checks the session budget before execution.
    Call this at the start of each Worker iteration, before invoking any model.

    Args:
        prompt: The full prompt text to be sent to the model
        model: Model ID (e.g. 'gemini-3-flash-preview')
        tool_context: ADK context object

    Returns:
        {"allowed": True,  "tokens": N, "cost_usd": X} if budget available
        {"allowed": False, "reason": "..."} if budget exceeded — Checker should escalate
    """
    prompt_tokens = count_tokens(prompt, model)
    used = tool_context.state.get("session:tokens_used", 0)
    remaining = SESSION_TOKEN_BUDGET - used

    if prompt_tokens > remaining:
        return {
            "allowed": False,
            "reason": (
                f"Token budget exceeded: need {prompt_tokens:,}, "
                f"remaining {remaining:,} of {SESSION_TOKEN_BUDGET:,}"
            ),
        }

    # Reserve budget — use usage_metadata from the response for actual post-hoc reconciliation
    tool_context.state["session:tokens_used"] = used + prompt_tokens
    tool_context.state["session:cost_usd_estimate"] = (
        tool_context.state.get("session:cost_usd_estimate", 0.0)
        + estimate_cost_usd(prompt_tokens, model)
    )

    return {
        "allowed": True,
        "tokens": prompt_tokens,
        "cost_usd": estimate_cost_usd(prompt_tokens, model),
    }
```

---

### Model Tier Routing — Planner Determines Complexity

```python
# Add model_tier decision to the Planner instruction
PLANNER_INSTRUCTION = """
You are the planner agent. Decompose the goal into micro-tasks and write to state['plan'].

For each task, also set state['model_tier'] based on complexity:
- "flash": routine execution, data retrieval, simple summarization
- "pro": complex architecture design, multi-step migration planning, trade-off analysis

Example:
  state['plan'] = "1. Retrieve error logs\\n2. Design migration strategy"
  state['model_tier'] = "pro"   # ← migration strategy requires deep reasoning
"""

# Worker reads model_tier to self-identify with the appropriate model
WORKER_INSTRUCTION = """
You are the worker agent. Execute the task from state['plan'].

Before starting:
1. Read state['model_tier'] to understand the expected reasoning depth
2. Call check_and_reserve_tokens() with the task prompt and your model ID
3. If check result is {"allowed": False}, write to state['execution_result']:
   {"status": "fatal_error", "error": "Token budget exceeded"}
   — The Checker will escalate.

Then execute the task and write results to state['execution_result'].
"""
```

```python
# agents/agent.py — Worker selection based on model_tier
from tools.token_budget import check_and_reserve_tokens

worker_flash = Agent(
    name="worker_flash",
    model="gemini-3-flash-preview",
    instruction=WORKER_INSTRUCTION,
    tools=[check_and_reserve_tokens, search_knowledge_base, get_task_state, update_task_state],
    output_key="execution_result",
)

worker_pro = Agent(
    name="worker_pro",
    model="gemini-3-pro-preview",
    instruction=WORKER_INSTRUCTION,
    tools=[check_and_reserve_tokens, search_knowledge_base, get_task_state, update_task_state],
    output_key="execution_result",
)
```

> **Simplification option**: Using Flash for all Worker agents and Pro only for the Planner across the entire pipeline can reduce costs by 80% or more. This is the default configuration for this project.

---

### Session Cost Report — Post-execution Aggregation

```python
def get_session_cost_report(tool_context: ToolContext) -> dict:
    """Returns the current session's token usage and estimated cost."""
    return {
        "tokens_used": tool_context.state.get("session:tokens_used", 0),
        "budget":      SESSION_TOKEN_BUDGET,
        "remaining":   SESSION_TOKEN_BUDGET - tool_context.state.get("session:tokens_used", 0),
        "cost_usd":    round(tool_context.state.get("session:cost_usd_estimate", 0.0), 4),
    }
```

---

### Environment Variable Configuration

```bash
# agents/.env
SESSION_TOKEN_BUDGET=100000   # Input token cap per session (default: 100,000)
```

---

### Cloud Monitoring — Token Usage Spike Alerts

Agent Engine automatically records token usage to Cloud Monitoring. Use the following natural language queries to detect anomalies:

```bash
# Using the observability extension
gemini "show total token usage per session for the enterprise_ralph_loop agent today"
gemini "which sessions consumed more than 80,000 tokens in the last 24 hours?"
gemini "what is the average cost per Ralph Loop iteration this week?"
```

---

### Cost Management Checklist

- [ ] Set `SESSION_TOKEN_BUDGET` environment variable (default: 100,000)
- [ ] Include `check_and_reserve_tokens()` call in Worker instruction
- [ ] Include `model_tier` decision logic in Planner instruction
- [ ] Checker handles escalation on `fatal_error: Token budget exceeded`
- [ ] Keep `max_iterations=10` guardrail in place (directly limits loop costs)

---

## 10. Observability — Structured Logs + Cloud Trace Code Integration

Implements Layer 7 Observability for Agent Harness in actual code.
In a Cloud Run environment, Python's standard `logging` is automatically collected as Cloud Logging `jsonPayload`.
Cloud Trace visualizes the agent reasoning path via the OpenTelemetry SDK.

> **Additional dependencies** (add to `requirements.txt`):
> ```text
> opentelemetry-sdk
> opentelemetry-exporter-gcp-trace
> opentelemetry-instrumentation
> ```

---

### `agents/observability.py` — Structured Logs + Trace Initialization

```python
# agents/observability.py
import logging
import os
import json
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter

# ──────────────────────────────────────────────
# 1. Structured logging configuration
# In Cloud Run, Python standard logging is automatically collected as Cloud Logging jsonPayload
# ──────────────────────────────────────────────

class StructuredFormatter(logging.Formatter):
    """
    Formats logs as JSON recognized by Cloud Logging.
    JSON written to stderr/stdout in Cloud Run is automatically parsed as jsonPayload.
    """
    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            "severity": record.levelname,          # Cloud Logging severity field
            "message":  super().format(record),
            "logger":   record.name,
            "agent":    os.environ.get("AGENT_NAME", "agent-harness"),
            "project":  os.environ.get("GOOGLE_CLOUD_PROJECT", ""),
        }
        # Merge extra fields (trace_id, session_id, etc.)
        if hasattr(record, "extra_fields"):
            log_entry.update(record.extra_fields)
        return json.dumps(log_entry, ensure_ascii=False)


def setup_logging(level: int = logging.INFO) -> logging.Logger:
    """
    Configures application-wide structured logging.
    Call once at the top of agents/agent.py or the Cloud Run Job entry point.

    Usage:
        from agents.observability import setup_logging
        logger = setup_logging()
        logger.info("Agent started", extra={"extra_fields": {"session_id": "abc-123"}})
    """
    handler = logging.StreamHandler()
    handler.setFormatter(StructuredFormatter())
    root = logging.getLogger()
    root.handlers.clear()
    root.addHandler(handler)
    root.setLevel(level)
    return logging.getLogger("agent_harness")


# ──────────────────────────────────────────────
# 2. Cloud Trace configuration — OpenTelemetry
# ──────────────────────────────────────────────

def setup_tracing(project_id: str | None = None) -> trace.Tracer:
    """
    Initializes the OpenTelemetry TracerProvider with the Cloud Trace exporter.
    Call once at the top of agents/agent.py.

    Usage:
        from agents.observability import setup_tracing
        tracer = setup_tracing()
    """
    pid = project_id or os.environ["GOOGLE_CLOUD_PROJECT"]
    provider = TracerProvider()
    provider.add_span_processor(
        BatchSpanProcessor(CloudTraceSpanExporter(project_id=pid))
    )
    trace.set_tracer_provider(provider)
    return trace.get_tracer("agent_harness")


# Global singletons — import and use after initialization
logger = setup_logging()
tracer = setup_tracing()
```

---

### Using Trace Spans in Tool Functions

```python
# tools/grounding.py (example — apply the same pattern to other tools)
from agents.observability import logger, tracer

def search_knowledge_base(query: str, tool_context) -> list[str]:
    session_id = tool_context.state.get("session:id", "unknown")

    with tracer.start_as_current_span("search_knowledge_base") as span:
        span.set_attribute("session_id", session_id)
        span.set_attribute("query_length", len(query))

        logger.info(
            "Knowledge base search started",
            extra={"extra_fields": {"session_id": session_id, "query": query[:100]}},
        )
        try:
            results = _do_search(query)          # actual search logic
            span.set_attribute("result_count", len(results))
            logger.info(
                "Knowledge base search completed",
                extra={"extra_fields": {"session_id": session_id, "hits": len(results)}},
            )
            return results
        except Exception as exc:
            span.record_exception(exc)
            span.set_status(trace.StatusCode.ERROR, str(exc))
            logger.error(
                "Knowledge base search failed",
                extra={"extra_fields": {"session_id": session_id, "error": str(exc)}},
            )
            raise
```

---

### Tracing Loop Iterations via ADK Callbacks

> **Important**: Spans must be tracked in a **module-level dict**, not in ADK State.
> ADK State is serialized to JSON for persistence — OpenTelemetry Span objects are not JSON-serializable.
> `tracer.start_span()` returns a Span; call `span.end()` explicitly in `after_model_callback`.

```python
# agents/callbacks.py — add span to before_model_callback
from __future__ import annotations
from typing import Any
from agents.observability import logger, tracer
from opentelemetry import trace as otel_trace

# Module-level registry for active spans.
# Key: "{session_id}.{agent_name}" — uniquely identifies each concurrent LLM call.
# Never store Span objects in ADK State (not JSON-serializable).
_active_spans: dict[str, Any] = {}


def before_model_callback(callback_context, llm_request):
    """Model Armor input check + start iteration trace span."""
    session_id = callback_context.state.get("session:id", "unknown")
    iteration  = callback_context.state.get("session:iteration", 0)

    # Start span and store in module-level dict (NOT in ADK state)
    span_key = f"{session_id}.{callback_context.agent_name}"
    span = tracer.start_span(f"llm_call.iteration_{iteration}")
    span.set_attribute("session_id", session_id)
    span.set_attribute("agent_name", callback_context.agent_name)
    _active_spans[span_key] = span   # ← module-level dict, not callback_context.state

    logger.info(
        "LLM call started",
        extra={"extra_fields": {
            "session_id": session_id,
            "agent": callback_context.agent_name,
            "iteration": iteration,
        }},
    )
    # Model Armor check: see adk_patterns.md §7
    return None


def after_model_callback(callback_context, llm_response):
    """Model Armor output check + explicitly end the trace span."""
    session_id = callback_context.state.get("session:id", "unknown")
    span_key = f"{session_id}.{callback_context.agent_name}"

    # Retrieve and end the span started in before_model_callback
    span = _active_spans.pop(span_key, None)
    if span:
        span.end()   # ← explicit end — spans do NOT close themselves

    logger.info(
        "LLM call completed",
        extra={"extra_fields": {"session_id": session_id}},
    )
    return None
```

---

### Verifying Structured Logs in Cloud Run

When running in Cloud Run, standard output JSON is **automatically collected as Cloud Logging `jsonPayload`**. No additional client configuration is required.

```bash
# Query agent logs from Cloud Logging
gcloud logging read \
  'resource.type="cloud_run_revision" jsonPayload.agent="agent-harness"' \
  --project=$PROJECT_ID --limit=50

# Filter by severity
gcloud logging read \
  'resource.type="cloud_run_revision" jsonPayload.severity="ERROR"' \
  --project=$PROJECT_ID --limit=20

# Using the Gemini CLI observability extension (convenient)
gemini "show ERROR logs from the agent-harness Cloud Run service in the last hour"
gemini "show the Cloud Trace spans for session_id abc-123 and their durations"
```

---

### Initialization in the `agents/agent.py` Entry Point

```python
# Top of agents/agent.py — run before all other imports
from agents.observability import setup_logging, setup_tracing

logger = setup_logging()
tracer = setup_tracing()

logger.info("Agent Harness initializing", extra={"extra_fields": {
    "project": os.environ.get("GOOGLE_CLOUD_PROJECT"),
    "region":  os.environ.get("REGION"),
}})
```

---

### Observability Checklist

- [ ] Add `opentelemetry-sdk`, `opentelemetry-exporter-gcp-trace` to `requirements.txt`
- [ ] Create `agents/observability.py` (`setup_logging`, `setup_tracing`, global singletons)
- [ ] Call `setup_logging()` / `setup_tracing()` once at the top of `agents/agent.py`
- [ ] Apply `tracer.start_as_current_span()` to all tool functions
- [ ] Record LLM call trace spans in `before_model_callback` / `after_model_callback`
- [ ] After Cloud Run deployment, verify `jsonPayload` collection with `gcloud logging read`
- [ ] Verify agent reasoning path visualization in the Cloud Trace console

---

## 11. Rate Limiting — Quota Pre-Control

`ResourceExhausted` (429) is one of the most common failures in production agent harnesses. The standard retry-with-backoff in §2 handles individual call failures, but it does not prevent quota from being exhausted in the first place. This section adds a **token bucket pre-control layer** that proactively throttles requests before hitting the API quota wall.

### Why Pre-Control Instead of Pure Retry

| Approach | Problem |
|----------|---------|
| Retry only | Parallel agents retry simultaneously → quota storm, cascading failures |
| Pre-control | Smooth request rate before calls reach the API → predictable throughput |

### `tools/rate_limiter.py` — Token Bucket Implementation

```python
# tools/rate_limiter.py
"""
Token bucket rate limiter for Gemini API quota pre-control.
Prevents ResourceExhausted (429) by throttling requests before they reach the API.

Quota reference (us-central1 / global endpoint, as of 2025):
  gemini-2.5-flash: 1,000 RPM / 4,000,000 TPM
  gemini-2.5-pro:   360 RPM / 2,000,000 TPM
Verify current limits: https://cloud.google.com/vertex-ai/generative-ai/docs/quotas
"""
import os
import time
import threading
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class TokenBucket:
    """
    Thread-safe token bucket for rate limiting.
    capacity:     Max burst size (tokens)
    refill_rate:  Tokens added per second
    """
    capacity: float
    refill_rate: float        # tokens per second
    _tokens: float = field(init=False)
    _last_refill: float = field(init=False)
    _lock: threading.Lock = field(default_factory=threading.Lock, init=False)

    def __post_init__(self):
        self._tokens = self.capacity
        self._last_refill = time.monotonic()

    def _refill(self) -> None:
        now = time.monotonic()
        elapsed = now - self._last_refill
        self._tokens = min(self.capacity, self._tokens + elapsed * self.refill_rate)
        self._last_refill = now

    def consume(self, tokens: float = 1.0, timeout: float = 30.0) -> bool:
        """
        Blocks until `tokens` are available or `timeout` seconds have elapsed.
        Returns True if tokens were consumed, False if timed out.
        """
        deadline = time.monotonic() + timeout
        while time.monotonic() < deadline:
            with self._lock:
                self._refill()
                if self._tokens >= tokens:
                    self._tokens -= tokens
                    return True
            # Not enough tokens yet — wait and retry
            wait = tokens / self.refill_rate
            time.sleep(min(wait, deadline - time.monotonic()))
        logger.warning(
            "Rate limiter timeout after %.1fs waiting for %.0f tokens",
            timeout, tokens,
        )
        return False


# ─── Global rate limiter singletons ─────────────────────────────────────────
# Adjust these to match your project's actual quota allocation.
# If multiple services share quota, set a fraction of the total here.

_FLASH_RPM  = float(os.environ.get("RATE_LIMIT_FLASH_RPM",  "600"))   # default: 60% of 1,000 RPM
_PRO_RPM    = float(os.environ.get("RATE_LIMIT_PRO_RPM",    "200"))   # default: ~55% of 360 RPM

flash_limiter = TokenBucket(
    capacity=min(_FLASH_RPM / 10, 20),   # burst: up to 20 requests
    refill_rate=_FLASH_RPM / 60,         # RPM → requests per second
)

pro_limiter = TokenBucket(
    capacity=min(_PRO_RPM / 10, 10),
    refill_rate=_PRO_RPM / 60,
)


def acquire_flash_quota(timeout: float = 30.0) -> bool:
    """Acquires a rate-limit token for a gemini-2.5-flash call. Blocks if throttled."""
    return flash_limiter.consume(1.0, timeout=timeout)


def acquire_pro_quota(timeout: float = 60.0) -> bool:
    """Acquires a rate-limit token for a gemini-2.5-pro call. Blocks if throttled."""
    return pro_limiter.consume(1.0, timeout=timeout)
```

### Wrapping ModelHarness with Rate Limiting

```python
# agents/harness.py — add rate limiting before every model call
from tools.rate_limiter import acquire_flash_quota, acquire_pro_quota
from google.api_core import exceptions as gcp_exceptions


class ModelHarness:
    def generate(self, prompt: str, system_instruction: str = None) -> AgentResponse:
        # Pre-control: acquire quota token before calling the API
        model_tier = "pro" if "pro" in self.model_id else "flash"
        acquired = (
            acquire_pro_quota() if model_tier == "pro" else acquire_flash_quota()
        )
        if not acquired:
            raise gcp_exceptions.ResourceExhausted(
                "Rate limiter timeout — quota pre-control rejected the request."
            )

        # Proceed with the API call
        resp = self.client.models.generate_content(...)
        return resp.parsed
```

### Wrapping the `before_model_callback`

For ADK agents that bypass `ModelHarness`, apply rate limiting in the callback:

```python
# agents/callbacks.py — rate limiting in before_model_callback
from tools.rate_limiter import acquire_flash_quota

def before_model_callback(callback_context, llm_request):
    """Rate limit + Model Armor before every LLM call."""
    # Throttle if near quota limit
    if not acquire_flash_quota(timeout=20.0):
        from google.adk.models.llm_response import LlmResponse
        from google.genai import types
        return LlmResponse(
            content=types.Content(
                role="model",
                parts=[types.Part(text="[Rate limited — quota pre-control. Retry later.]")],
            )
        )
    # Model Armor check (see §7 in adk_patterns.md)
    return None
```

### Environment Variables

```bash
# agents/.env — tune to your project's actual quota allocation
RATE_LIMIT_FLASH_RPM=600    # 60% of 1,000 RPM (leaves headroom for bursts)
RATE_LIMIT_PRO_RPM=200      # ~55% of 360 RPM
```

Add to `infra/main.tf` deployment_spec:
```hcl
env {
  name  = "RATE_LIMIT_FLASH_RPM"
  value = var.rate_limit_flash_rpm
}
env {
  name  = "RATE_LIMIT_PRO_RPM"
  value = var.rate_limit_pro_rpm
}
```

### Rate Limiting Checklist

- [ ] `tools/rate_limiter.py` created with `TokenBucket`, `flash_limiter`, `pro_limiter`
- [ ] `RATE_LIMIT_FLASH_RPM` / `RATE_LIMIT_PRO_RPM` in `agents/.env` and `infra/main.tf`
- [ ] `acquire_flash_quota()` called in `ModelHarness.generate()` before the API call
- [ ] `acquire_flash_quota()` called in `before_model_callback` for ADK agents
- [ ] Monitor `ResourceExhausted` errors in Cloud Logging — reduce RPM limits if spikes persist
- [ ] For Cloud Run scaled-out deployments: set limits per **instance** (not total) to avoid over-throttling
- [ ] Same applies to **Vertex AI Agent Engine**: Agent Engine also scales instances horizontally. The `flash_limiter`/`pro_limiter` singletons are per-process — each instance has its own bucket. Set RPM limits accordingly (e.g. `total_quota / expected_instance_count`).
- [ ] Regularly check per-session token usage with the `observability` extension
