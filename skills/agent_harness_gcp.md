# Agent Harness on Google Cloud — Architecture Guide

## Overview
An Agent Harness is an **infrastructure software system that wraps an LLM agent**.
It is not the agent itself (the brain) — it is the **body** in which the agent operates: providing tools, context, memory, safety guardrails, and workflow structure.

This project is an enterprise-grade sample that implements an Agent Harness using Google Cloud native services.

---

## 1. The Three-Layer Agent Stack

```
┌─────────────────────────────────────────────────┐
│  Agent Harness  (batteries included)             │
│  → This project (ADK + Agent Engine)             │
├─────────────────────────────────────────────────┤
│  Agent Runtime  (durable execution, persistence) │
│  → Vertex AI Agent Engine / Cloud Run / GKE      │
├─────────────────────────────────────────────────┤
│  Agent Framework  (abstractions, building blocks)│
│  → Google ADK                                    │
└─────────────────────────────────────────────────┘
```

---

## 2. GCP Agent Harness — 8-Layer Architecture

### Layer 1: ADK (Development Framework)
- **Role**: Define agents, tools, and orchestration in code
- **Entry point**: `agents/agent.py` → `root_agent = ralph_loop`
- **Key classes**: `Agent`, `SequentialAgent`, `LoopAgent`, `ToolContext`

### Layer 2: Agent Runtime (Execution Environment)
| Environment | Best For | Deployment | Extension |
|-------------|----------|------------|-----------|
| **Vertex AI Agent Engine** (recommended) | Python agents, managed sessions & memory | `adk deploy agent_engine` or Terraform | — |
| Cloud Run | Multi-language, serverless, MCP server hosting | `adk deploy cloud_run` | [`cloud-run-mcp`](https://github.com/GoogleCloudPlatform/cloud-run-mcp) |
| GKE | Complex stateful workloads, strict security isolation | `kubectl apply` | [`gke-mcp`](https://github.com/GoogleCloudPlatform/gke-mcp) |

**`cloud-run-mcp`** — deploy and manage Cloud Run services (including MCP servers) via natural language:
```bash
# Install
gemini extensions install https://github.com/GoogleCloudPlatform/cloud-run-mcp
# or: npx -y @google-cloud/cloud-run-mcp

# Deploy the current working directory to Cloud Run
/deploy
# Get logs for a Cloud Run service
/logs
```

**`gke-mcp`** — manage GKE clusters, generate AI workload manifests, analyze upgrade risks:
```bash
gemini extensions install https://github.com/GoogleCloudPlatform/gke-mcp
```

**`devops`** — design and deploy full CI/CD pipelines (Cloud Build + Artifact Registry + Cloud Deploy):
```bash
gemini extensions install https://github.com/gemini-cli-extensions/devops
/cicd:deploy   # auto-detect and deploy project
/cicd:design   # AI-designed pipeline with required GCP infrastructure
```

### Layer 3: AI Models (Reasoning Engine)
- **Gemini 3 Pro**: Planner — complex decomposition and reasoning (`gemini-3-pro-preview`)
- **Gemini 3 Flash**: Worker / Checker — repetitive execution (`gemini-3-flash-preview`)
- **Region Decoupling**: infrastructure region (`us-central1`) ≠ model endpoint (`global`)

**`vertex` extension** — manage Vertex AI prompts and run prompt optimization jobs from the CLI:
```bash
gemini extensions install https://github.com/gemini-cli-extensions/vertex

# Manage system_instruction.txt as a versioned Vertex AI Prompt
gemini "create a prompt with content from prompts/system_instruction.txt \
        and display name 'harness-system-instruction'"
gemini "list prompts"
gemini "run a data-driven optimization job on prompt <id>"
```

### Layer 4: Tool Integration (The Agent's Hands)
```
Built-in Tools    → Google Search, Code Execution, RAG (tools/grounding.py)
MCP Servers       → Hosted on Cloud Run, SSE transport (tools/mcp_client.py)
Custom Functions  → Internal APIs, business logic (tools/*.py)
Apigee API Hub    → Enterprise API management at scale
```

**Extensions for Layer 4:**

| Extension | Tools Provided | Install |
|-----------|---------------|---------|
| [`mcp-toolbox`](https://github.com/gemini-cli-extensions/mcp-toolbox) | Custom DB tools via `tools.yaml` | `gemini extensions install ...` |
| [`bigquery-data-analytics`](https://github.com/gemini-cli-extensions/bigquery-data-analytics) | `execute_sql`, `forecast`, `search_catalog`, `analyze_contribution` | `gemini extensions install ...` |
| [`bigquery-conversational-analytics`](https://github.com/gemini-cli-extensions/bigquery-conversational-analytics) | `ask_data_insights`, `search_catalog` (Conversational Analytics API) | `gemini extensions install ...` |
| [`dataplex`](https://github.com/gemini-cli-extensions/dataplex) | `search_entries`, `lookup_entry`, `search_aspect_types` | `gemini extensions install ...` |
| [`firebase`](https://github.com/gemini-cli-extensions/firebase) | Firestore, Remote Config, Crashlytics, Hosting | `gemini extensions install ...` |
| [`cloud-run-mcp`](https://github.com/GoogleCloudPlatform/cloud-run-mcp) | `deploy-file-contents`, `list-services`, `get-service-log` | `npx -y @google-cloud/cloud-run-mcp` |

**Communication protocols**:
- `MCP`: agent ↔ tool (open standard led by Anthropic)
- `A2A`: agent ↔ agent (open standard led by Google)

### Layer 5: Memory (State Persistence)
```
# Development / testing
InMemorySessionService       — lost on instance restart

# Production (built into Agent Engine)
Agent Engine Sessions        — short-term memory (conversation history within a session)
Memory Bank                  — long-term memory (knowledge shared across sessions)

# ZDR (Zero-Downtime Resilience)
RalphLoopManager.save_state()   — persists intermediate state to GCS (production)
RalphLoopManager.load_state()   — rehydrates state on container restart from GCS
# Note: JSON file backend is used for local dev only (Cloud Run has no persistent local storage)
# See ralph_loop_gcp.md §3 for the GCS-backed production implementation
```

**Core principle**: In production, always use **Stateless Agent Application + External State Store** — any instance must be able to handle any request.

### Layer 6: Security & Governance (Safety Guardrails)
| Service | Role | Where Implemented |
|---------|------|-------------------|
| **Policy Engine** | Blocks destructive commands (SRE Interception) | `tools/policy_engine.py` |
| **Model Armor** | Prompt injection defense, harmful content filtering | Vertex AI configuration |
| **IAM** | Per-agent least-privilege identity | `infra/main.tf` |
| **Secret Manager** | Secure storage of API keys and credentials | Injected as env vars |
| **Safety Settings** | Gemini-level content safety thresholds | `agents/harness.py` |

```python
# agents/harness.py — enterprise safety configuration
safety_settings=[
    types.SafetySetting(category='HARM_CATEGORY_HATE_SPEECH', threshold='BLOCK_ONLY_HIGH'),
    types.SafetySetting(category='HARM_CATEGORY_DANGEROUS_CONTENT', threshold='BLOCK_ONLY_HIGH'),
]
```

**Model Armor — Prompt Injection & Jailbreak Defense Configuration**

Model Armor is a managed defense layer that inspects inputs and outputs **before and after** model calls. While Safety Settings are Gemini's internal filters, Model Armor is an external gate that sits in front of them.

```
User input
    │
    ▼
[Model Armor: sanitize_user_prompt()]   ← Blocks prompt injection and harmful content
    │  BLOCK → return model_armor_block
    ▼
[Gemini: generate_content()]            ← Safety Settings applied
    │
    ▼
[Model Armor: sanitize_model_response()]  ← Blocks harmful content and malicious URLs in output
    │  BLOCK → return model_armor_block
    ▼
User response
```

**Step 1 — Enable API and IAM**

```bash
# Enable the Model Armor API
gcloud services enable modelarmor.googleapis.com --project=$PROJECT_ID

# Grant the service account permission to use Model Armor
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:agent-harness-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/modelarmor.user"
```

**Step 2 — Create a protection template (`infra/model_armor_setup.py`)**

```python
# infra/model_armor_setup.py — run once during infrastructure setup
from google.cloud import modelarmor_v1
import os

PROJECT_ID = os.environ["GOOGLE_CLOUD_PROJECT"]
LOCATION   = os.environ.get("MODEL_ARMOR_LOCATION", "us-central1")
# ↑ Model Armor service region — completely separate from GOOGLE_CLOUD_LOCATION=global.
#   Model Armor is a regional GCP API (endpoint: {LOCATION}-modelarmor.googleapis.com).
#   us-central1 is the recommended region for Model Armor templates.
#   Do NOT confuse with the Gemini model endpoint (GOOGLE_CLOUD_LOCATION=global).
TEMPLATE_ID = "harness-protection-template"


def create_protection_template() -> str:
    """Creates a Model Armor protection template for the Agent Harness."""
    client = modelarmor_v1.ModelArmorClient(
        client_options={"api_endpoint": f"{LOCATION}-modelarmor.googleapis.com"}
    )

    template = modelarmor_v1.Template(
        filter_config=modelarmor_v1.FilterConfig(
            # RAI filter — harmful content (block at MEDIUM confidence and above)
            rai_settings=modelarmor_v1.RaiFilterSettings(
                rai_filters=[
                    modelarmor_v1.RaiFilter(
                        filter_type=modelarmor_v1.RaiFilterType.DANGEROUS_CONTENT,
                        confidence_level=modelarmor_v1.DetectionConfidenceLevel.MEDIUM_AND_ABOVE,
                    ),
                    modelarmor_v1.RaiFilter(
                        filter_type=modelarmor_v1.RaiFilterType.HATE_SPEECH,
                        confidence_level=modelarmor_v1.DetectionConfidenceLevel.MEDIUM_AND_ABOVE,
                    ),
                ]
            ),
            # Prompt injection + jailbreak defense (LOW confidence and above — strict setting)
            pi_and_jailbreak_filter_settings=modelarmor_v1.PiAndJailbreakFilterSettings(
                filter_enforcement=(
                    modelarmor_v1.PiAndJailbreakFilterSettings
                    .PiAndJailbreakFilterEnforcement.ENABLED
                ),
                confidence_level=modelarmor_v1.DetectionConfidenceLevel.LOW_AND_ABOVE,
            ),
            # Malicious URL detection
            malicious_uri_filter_settings=modelarmor_v1.MaliciousUriFilterSettings(
                filter_enforcement=(
                    modelarmor_v1.MaliciousUriFilterSettings
                    .MaliciousUriFilterEnforcement.ENABLED
                ),
            ),
        )
    )

    created = client.create_template(
        parent=f"projects/{PROJECT_ID}/locations/{LOCATION}",
        template_id=TEMPLATE_ID,
        template=template,
    )
    print(f"Model Armor template created: {created.name}")
    return created.name


if __name__ == "__main__":
    create_protection_template()
```

**Step 3 — `tools/model_armor.py` — Input/Output inspection integration**

```python
# tools/model_armor.py
import os
import logging
from google.cloud import modelarmor_v1

logger = logging.getLogger(__name__)

PROJECT_ID  = os.environ["GOOGLE_CLOUD_PROJECT"]
LOCATION    = os.environ.get("MODEL_ARMOR_LOCATION", "us-central1")
# ↑ Model Armor service region (regional API endpoint: {LOCATION}-modelarmor.googleapis.com).
#   This is NOT the Gemini model endpoint — keep GOOGLE_CLOUD_LOCATION=global for model calls.
TEMPLATE_ID = os.environ.get("MODEL_ARMOR_TEMPLATE_ID", "harness-protection-template")

_TEMPLATE_NAME = (
    f"projects/{PROJECT_ID}/locations/{LOCATION}/templates/{TEMPLATE_ID}"
)
_client = modelarmor_v1.ModelArmorClient(
    client_options={"api_endpoint": f"{LOCATION}-modelarmor.googleapis.com"}
)


def sanitize_prompt(user_prompt: str) -> bool:
    """
    Checks user input for prompt injection, jailbreak, and harmful content.
    Returns True if safe to forward to the model, False if blocked.
    """
    response = _client.sanitize_user_prompt(
        request=modelarmor_v1.SanitizeUserPromptRequest(
            name=_TEMPLATE_NAME,
            user_prompt_data=modelarmor_v1.DataItem(text=user_prompt),
        )
    )
    result = response.sanitization_result
    is_safe = (
        result.filter_match_state != modelarmor_v1.FilterMatchState.MATCH_FOUND
    )
    if not is_safe:
        logger.warning("Model Armor blocked prompt: %s", result.filter_results)
    return is_safe


def sanitize_response(model_response: str) -> bool:
    """
    Checks model output for harmful content and malicious URLs.
    Returns True if safe to return to the user, False if blocked.
    """
    response = _client.sanitize_model_response(
        request=modelarmor_v1.SanitizeModelResponseRequest(
            name=_TEMPLATE_NAME,
            model_response_data=modelarmor_v1.DataItem(text=model_response),
        )
    )
    result = response.sanitization_result
    is_safe = (
        result.filter_match_state != modelarmor_v1.FilterMatchState.MATCH_FOUND
    )
    if not is_safe:
        logger.warning("Model Armor blocked response: %s", result.filter_results)
    return is_safe
```

**Step 4 — Integrate into `ModelHarness.generate_safe()`**

```python
# agents/harness.py — add Model Armor to generate_safe()
from tools.model_armor import sanitize_prompt, sanitize_response

def generate_safe(self, prompt: str, system_instruction: str = None) -> dict:
    # Step 1: Input inspection — block prompt injection and jailbreak attempts
    if not sanitize_prompt(prompt):
        return {"status": "model_armor_block", "result": None,
                "reason": "Input blocked by Model Armor (injection or harmful content)"}

    # Step 2: Model call (see error_handling.md Section 5)
    try:
        resp = self.client.models.generate_content(...)
        ...
        output = resp.text
    except ...:
        ...

    # Step 3: Output inspection — block harmful content and malicious URLs
    if not sanitize_response(output):
        return {"status": "model_armor_block", "result": None,
                "reason": "Output blocked by Model Armor (harmful content or malicious URL)"}

    return {"status": "success", "result": output}
```

**Environment Variables**

```bash
# agents/.env
MODEL_ARMOR_LOCATION=us-central1
MODEL_ARMOR_TEMPLATE_ID=harness-protection-template
```

**Safety Settings vs Model Armor — Role Distinction**

| Defense Layer | Position | What It Blocks |
|--------------|----------|----------------|
| **Model Armor** | Before and after model call (external gate) | Prompt injection, jailbreak, malicious URLs, RAI |
| **Safety Settings** | Inside the Gemini model | Harmful content categories (model-level filter) |
| **Policy Engine** | Before tool execution | Destructive command patterns (business logic level) |

> **Requirement**: The `google-cloud-modelarmor` package must be installed. Add it to `requirements.txt`.

### Layer 7: Observability (Monitoring)
```bash
# Required env vars (agents/.env and infra/main.tf deployment_spec.env block)
GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
```
- **Cloud Trace**: Visualizes agent reasoning paths and tool call chains
- **Cloud Logging**: Centralized log collection (policy violations, errors)
- **Agent Engine Dashboard**: Real-time token usage, latency, and error rate

**`observability` extension** — query Cloud Logging, Monitoring, Trace, and Error Reporting in natural language:
```bash
gemini extensions install https://github.com/gemini-cli-extensions/observability

# Examples
gemini "show me error logs from the harness agent in the last hour"
gemini "what is the p99 latency for the enterprise_ralph_loop agent?"
gemini "show traces where tool calls took longer than 5 seconds"
gemini "list recent error reports for the agent engine service"
```

**`security` extension** — scan code changes for vulnerabilities and CVEs before deployment:
```bash
gemini extensions install https://github.com/gemini-cli-extensions/security

/security:analyze              # analyze current branch diff
/security:analyze --json       # machine-readable output for CI/CD gate
```

### Layer 8: Multi-Agent Patterns

| Pattern | Implementation | When to Use |
|---------|---------------|-------------|
| **Sequential** | `SequentialAgent([planner, worker, checker])` | Single-domain tasks |
| **Iterative (Ralph Loop)** | `LoopAgent` + `escalate_issue` tool | Long-running autonomous execution |
| **Coordinator** | Delegate to specialized agents via A2A protocol | Multi-domain composite tasks |

**Coordinator → Specialist Pattern (A2A)**:
```python
# Coordinator delegates to SRE and Architect specialist agents via A2A
coordinator = Agent(
    name="coordinator",
    model="gemini-3-pro-preview",
    tools=[delegate_to_sre, delegate_to_architect],  # A2A call tools
)
root_agent = coordinator
```

Each specialist agent is independently deployed to Cloud Run and declares its capabilities via `/.well-known/agent.json` (AgentCard). See `adk_patterns.md §11` for the full A2A implementation pattern (AgentCard, JSON-RPC call, IAM setup).

---

## 3. File-to-Layer Mapping

| File | Layer(s) | Role |
|------|----------|------|
| `agents/agent.py` | 1, 8 | ADK agent definitions + orchestration |
| `agents/harness.py` | 3 | Gemini model abstraction + safety settings |
| `agents/ralph_loop.py` | 5 | ZDR state management + LoopAgent |
| `tools/policy_engine.py` | 6 | SRE Interception (command validation) |
| `tools/model_armor.py` | 6 | Model Armor input/output sanitization |
| `tools/mcp_client.py` | 4 | Remote MCP integration (SSE) |
| `tools/grounding.py` | 4 | RAG integration |
| `infra/main.tf` | 2 | Terraform IaC (Agent Engine provisioning) |
| `infra/memory_bank_config.py` | 5 | Memory Bank configuration |
| `ci-cd/cloudbuild.yaml` | 2 | Cloud Build CI/CD pipeline |
| `eval/run_eval.py` | 6 | Quality validation guardrail |
| `prompts/system_instruction.txt` | 1 | Agent system instructions |

---

## 4. Three-Tier Deployment Strategy

### Tier 1: Prototyping (Instant Start)
```bash
adk deploy cloud_run \
  --project=$PROJECT_ID \
  --region=us-central1 \
  --with-ui
```

### Tier 2: Production — ADK CLI (Method A)
```bash
# ⚠️ Agent Engine is only available in us-central1. --region must always be set to us-central1.
adk deploy agent_engine agents \
  --project $PROJECT_ID \
  --region us-central1 \
  --display_name "harness-production" \
  --validate-agent-import
```

### Tier 3: Enterprise — Terraform GitOps (Method B, recommended)
```bash
# Step 1: Build the artifact
./ci-cd/package_for_terraform.sh

# Step 2: Provision with Terraform
cd infra && terraform init
terraform apply -var="project_id=$PROJECT_ID" -var="region=$REGION"
```

---

## 5. Agent Principles (`prompts/system_instruction.txt`)

Agents in this project follow four principles:
1. **Fact-based**: Respond only based on provided context and tool results
2. **Safety first**: Refuse destructive operations and escalate to the Policy Engine
3. **Efficiency**: Minimize unnecessary reasoning; use Gemini Thinking at the appropriate level
4. **Resilience (ZDR)**: Record all critical intermediate state after every iteration — to ADK State (session), AlloyDB (production), or GCS (Cloud Run Jobs). Memory Bank is for long-term cross-session knowledge, not ZDR checkpoints.

---

## 6. Key Design Decisions

### Region Decoupling
- **Why**: Agent Engine containers deploy to `us-central1`, but the latest Gemini 3 models are only accessible via the `global` endpoint
- **How**: Set `GOOGLE_CLOUD_LOCATION=global` as an environment variable
- **Where**: `agents/.env` (local) + `infra/main.tf` `deployment_spec.env` block (production)

### Why Policy Engine Is a Separate Module
- Enforces SRP: agent logic (`agent.py`) is decoupled from security policy (`policy_engine.py`)
- Policy rules can be updated without touching agent code
- The decorator pattern applies validation consistently to any tool function

### `max_iterations=10` Guardrail
- Prevents infinite loops and controls enterprise costs
- Normal flow: Checker calls `escalate_issue` before the limit is reached
- Hitting the limit signals an abnormal condition — set up a Cloud Monitoring alert for this event
