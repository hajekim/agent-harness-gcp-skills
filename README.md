# Agent Harness on Google Cloud — Skills

A skill library for AI coding agents (Gemini CLI, Claude Code, and compatible tools) to **build, deploy, and operate an enterprise-grade Agent Harness on Google Cloud** — from first `gcloud auth` to production Terraform deployment.

These skills encode the architecture decisions, code patterns, GCP service wiring, and operational runbooks that would otherwise have to be re-explained every session.

---

## What Is an Agent Harness?

An Agent Harness is the **infrastructure software system that wraps an LLM** — not the model itself, but everything around it: tool integration, context management, memory, safety guardrails, and orchestration. Think of the agent as the engine, and the harness as the full vehicle.

On Google Cloud, the harness is built from three layers:

```
┌──────────────────────────────────────────────────────┐
│  Agent Harness  (batteries included)                  │
│  → ADK + Vertex AI Agent Engine + MCP + A2A          │
├──────────────────────────────────────────────────────┤
│  Agent Runtime  (durable execution, persistence)      │
│  → Vertex AI Agent Engine / Cloud Run / GKE           │
├──────────────────────────────────────────────────────┤
│  Agent Framework  (abstractions, building blocks)     │
│  → Google Agent Development Kit (ADK)                 │
└──────────────────────────────────────────────────────┘
```

---

## Skills in This Repository

9 files (8 skill files + 1 trigger file), organized by build phase:

| Phase | File | Description |
|-------|------|-------------|
| **Prerequisites** | [`skills/gcp_setup.md`](skills/gcp_setup.md) | GCP API enablement, IAM (incl. `roles/run.invoker`), service accounts, gcloud auth, `pyproject.toml`, `requirements.txt`, Secret Manager rotation, local dev workflow |
| **Framework** | [`skills/adk_patterns.md`](skills/adk_patterns.md) | ADK agent definition, State scopes, orchestration patterns, tool authoring, MCP + genai-toolbox, Model Armor callbacks, RAG/Grounding, A2A protocol, ModelHarness, complete `agent.py` assembly, Runner (`InMemoryRunner` + `DatabaseSessionService`), Human-in-the-Loop approval gate, ParallelAgent fan-out/fan-in |
| **Architecture** | [`skills/agent_harness_gcp.md`](skills/agent_harness_gcp.md) | Full 8-layer GCP architecture, Model Armor integration, file-to-layer mapping, three-tier deployment strategy, Region Decoupling pattern |
| **Autonomy** | [`skills/ralph_loop_gcp.md`](skills/ralph_loop_gcp.md) | Ralph Loop 5 principles, ZDR, backpressure, PolicyEngine singleton, GCS external memory, Cloud Run Job + Cloud Scheduler production outer loop |
| **State** | [`skills/memory_and_state.md`](skills/memory_and_state.md) | Memory Bank API, DatabaseSessionService, ZDR deep dive, genai-toolbox DB state tools, memory architecture decision tree |
| **Infrastructure** | [`skills/infra_and_cicd.md`](skills/infra_and_cicd.md) | Terraform IaC (with Agent Engine us-central1 validation), `package_for_terraform.sh`, three deployment methods, Cloud Build eval-gate pipeline, rollback strategy, VPC Private Access |
| **Reliability** | [`skills/error_handling.md`](skills/error_handling.md) | Failure taxonomy, retry/backoff (sync + async), circuit breaker, error propagation, token/cost management, structured logging + Cloud Trace, rate limiting (token bucket quota pre-control) |
| **Quality** | [`skills/evaluation.md`](skills/evaluation.md) | Golden dataset structure, eval runner, eval-gate CI/CD, unit test patterns (`ToolContext` mock, `conftest.py`, `pytest-asyncio`), code-review + security integration |
| **Triggers** | [`skills/TRIGGERS.md`](skills/TRIGGERS.md) | English and Korean trigger phrases mapped to each skill file — for AI agent skill routing |

---

## How to Use

### Skill Triggers

[`skills/TRIGGERS.md`](skills/TRIGGERS.md) maps natural language phrases to each skill file — in both **English** and **Korean**. AI agents automatically load the corresponding skill when a user's request matches a listed trigger phrase.

| Trigger Language | Example |
|-----------------|---------|
| English | "How do I implement the Ralph Loop on GCP?" → `ralph_loop_gcp.md` |
| Korean | "ADK 에이전트 정의 방법" → `adk_patterns.md` |

---

### With Gemini CLI

**Option A — Reference in `GEMINI.md`** (recommended):

```markdown
<!-- GEMINI.md in your project root -->

## Skills — Agent Harness on GCP
@agent-harness-gcp-skills/skills/TRIGGERS.md
@agent-harness-gcp-skills/skills/gcp_setup.md
@agent-harness-gcp-skills/skills/adk_patterns.md
@agent-harness-gcp-skills/skills/agent_harness_gcp.md
@agent-harness-gcp-skills/skills/ralph_loop_gcp.md
@agent-harness-gcp-skills/skills/memory_and_state.md
@agent-harness-gcp-skills/skills/infra_and_cicd.md
@agent-harness-gcp-skills/skills/error_handling.md
@agent-harness-gcp-skills/skills/evaluation.md
```

**Option B — Copy to project**:

```bash
cp -r agent-harness-gcp-skills/skills your-project/.gemini/skills/
```

### With Claude Code / Other Agents

Copy the `skills/` folder into your project's agent context directory:

```bash
cp -r agent-harness-gcp-skills/skills your-project/.agent/skills/
```

Reference in `project_context.md`:
```markdown
## Domain Skills
See `.agent/skills/` for GCP Agent Harness patterns.
```

---

## Combined Setup — With `agentic-design-patterns-skills`

These skills cover the **GCP infrastructure layer**. Pair them with [`hajekim/agentic-design-patterns-skills`](https://github.com/hajekim/agentic-design-patterns-skills) which covers the **agentic design pattern layer** (28 patterns: Routing, Parallelization, A2A, Memory Management, etc.).

Together they form a complete skill stack:

```markdown
<!-- GEMINI.md — full stack -->

## Skills — Agentic Design Patterns (What pattern to use)
@agentic-design-patterns-skills/skills/parallelization/SKILL.md
@agentic-design-patterns-skills/skills/multi-agent-collaboration/SKILL.md
@agentic-design-patterns-skills/skills/memory-management/SKILL.md
@agentic-design-patterns-skills/skills/tool-use/SKILL.md
@agentic-design-patterns-skills/skills/planning/SKILL.md

## Skills — Agent Harness on GCP (How to implement and deploy on GCP)
@agent-harness-gcp-skills/skills/TRIGGERS.md
@agent-harness-gcp-skills/skills/gcp_setup.md
@agent-harness-gcp-skills/skills/adk_patterns.md
@agent-harness-gcp-skills/skills/agent_harness_gcp.md
@agent-harness-gcp-skills/skills/ralph_loop_gcp.md
@agent-harness-gcp-skills/skills/memory_and_state.md
@agent-harness-gcp-skills/skills/infra_and_cicd.md
@agent-harness-gcp-skills/skills/error_handling.md
@agent-harness-gcp-skills/skills/evaluation.md
```

| Skill Library | Answers | Primary Framework |
|---------------|---------|-------------------|
| `agentic-design-patterns-skills` | *Which agentic pattern should I use?* | Google ADK / LangChain / CrewAI |
| `agent-harness-gcp-skills` (this repo) | *How do I build and deploy it on GCP?* | ADK + Vertex AI + Terraform |

---

## Extensions & Skills Integration Guide

Each Gemini CLI extension automates a specific layer of the Agent Harness build process. This section explains **which extension to use at each phase**, **how it connects to the corresponding skill**, and **what commands to run**.

### Install All Extensions

```bash
# Core infrastructure
gemini extensions install https://github.com/gemini-cli-extensions/gcloud
gemini extensions install https://github.com/gemini-cli-extensions/ralph
npx -y @google-cloud/cloud-run-mcp
gemini extensions install https://github.com/GoogleCloudPlatform/gke-mcp

# Data & state
gemini extensions install https://github.com/gemini-cli-extensions/mcp-toolbox
gemini extensions install https://github.com/gemini-cli-extensions/alloydb
gemini extensions install https://github.com/gemini-cli-extensions/cloud-sql-postgresql
gemini extensions install https://github.com/gemini-cli-extensions/bigquery-data-analytics

# Observability & quality
gemini extensions install https://github.com/gemini-cli-extensions/observability
gemini extensions install https://github.com/gemini-cli-extensions/security
gemini extensions install https://github.com/gemini-cli-extensions/devops
gemini extensions install https://github.com/gemini-cli-extensions/vertex
```

---

### Extension-to-Skill Mapping

| Extension | Skill | Primary Role |
|-----------|-------|--------------|
| [`gcloud`](https://github.com/gemini-cli-extensions/gcloud) | `gcp_setup.md` | Automate API enablement, IAM, project setup via natural language |
| [`ralph`](https://github.com/gemini-cli-extensions/ralph) | `ralph_loop_gcp.md` | Run Ralph Loop locally with `/ralph:loop` command |
| [`cloud-run-mcp`](https://github.com/GoogleCloudPlatform/cloud-run-mcp) | `infra_and_cicd.md` | Deploy MCP servers and Cloud Run services |
| [`gke-mcp`](https://github.com/GoogleCloudPlatform/gke-mcp) | `infra_and_cicd.md` | Manage GKE clusters for enterprise Tier 3 runtime |
| [`devops`](https://github.com/gemini-cli-extensions/devops) | `infra_and_cicd.md` | Design and deploy CI/CD pipelines with `/cicd:design` |
| [`mcp-toolbox`](https://github.com/gemini-cli-extensions/mcp-toolbox) | `adk_patterns.md`, `memory_and_state.md` | Custom DB tools via `tools.yaml` in Gemini CLI |
| [`alloydb`](https://github.com/gemini-cli-extensions/alloydb) | `memory_and_state.md` | AlloyDB-backed session state and ZDR |
| [`cloud-sql-postgresql`](https://github.com/gemini-cli-extensions/cloud-sql-postgresql) | `memory_and_state.md` | Cloud SQL PostgreSQL-backed session service |
| [`bigquery-data-analytics`](https://github.com/gemini-cli-extensions/bigquery-data-analytics) | `adk_patterns.md` | BigQuery as an agent tool |
| [`observability`](https://github.com/gemini-cli-extensions/observability) | `error_handling.md` | Natural language queries for logs, traces, metrics |
| [`security`](https://github.com/gemini-cli-extensions/security) | `error_handling.md`, `evaluation.md` | Pre-deploy security scan + CVE detection |
| [`vertex`](https://github.com/gemini-cli-extensions/vertex) | `agent_harness_gcp.md` | Manage and optimize `system_instruction.txt` as Vertex AI Prompts |
| [`genai-toolbox`](https://github.com/googleapis/genai-toolbox) | `adk_patterns.md`, `memory_and_state.md` | MCP server backend for enterprise database tools |

---

### Phase-by-Phase Workflow

#### Phase 1 — GCP Setup (`gcp_setup.md` + `gcloud`)

The `gcloud` extension exposes a `run_gcloud_command` MCP tool. Instead of memorizing CLI syntax, describe what you need in natural language and Gemini executes the correct `gcloud` command.

```bash
# 1. Set project and authenticate
gemini "set my active GCP project to my-project-id and authenticate with application default credentials"

# 2. Enable all required APIs in one step
gemini "enable Vertex AI, Cloud Run, Cloud Build, Artifact Registry, Secret Manager, Cloud Trace, Logging, Monitoring, Cloud Storage, IAM, Model Armor, and Cloud Scheduler APIs in project my-project-id"

# 3. Create the service account with minimum required roles
gemini "create a service account called agent-harness-sa and grant it the minimum roles needed to run an ADK agent on Vertex AI Agent Engine, deploy MCP servers to Cloud Run, and write logs and traces"

# 4. Create Artifact Registry repository
gemini "create a Docker repository called agent-harness-repo in us-central1 for project my-project-id"
```

**Skill reference**: See `gcp_setup.md` §1–4 for the exact IAM roles and API list behind each command above.

---

#### Phase 2 — Agent Development (`adk_patterns.md` + `vertex` + `mcp-toolbox`)

**`vertex` extension** — manages `prompts/system_instruction.txt` as a versioned Vertex AI Prompt, enabling A/B testing and rollback without redeployment:

```bash
# Create a versioned prompt from your system instruction file
gemini "create a prompt with content from prompts/system_instruction.txt \
  named agent-harness-system-instruction in project my-project-id"

# Compare two prompt versions against the golden dataset
gemini "compare prompt versions 1 and 2 of agent-harness-system-instruction \
  using eval/golden_dataset.json and report which version scores higher"

# Promote the better version
gemini "update the agent to use version 2 of agent-harness-system-instruction"
```

**`mcp-toolbox` extension** — wires `tools.yaml` into Gemini CLI for local tool testing before deploying to Agent Engine:

```bash
# Install and start the MCP Toolbox server
gemini extensions install https://github.com/gemini-cli-extensions/mcp-toolbox

# Test a tool defined in tools.yaml directly from CLI
gemini "call the get-task-state tool with session_id=test-session-001 and show the result"

# Validate that tools.yaml is correctly configured
gemini "list all tools available in tools.yaml and confirm they connect to the database"
```

**Skill reference**: See `adk_patterns.md` §9 for `tools.yaml` structure and §13 for complete `agent.py` assembly.

---

#### Phase 3 — State & Memory Setup (`memory_and_state.md` + `alloydb` / `cloud-sql-postgresql`)

Choose **AlloyDB** for high-throughput production workloads or **Cloud SQL PostgreSQL** for standard deployments. Both extensions use the same interaction pattern.

```bash
# Option A: AlloyDB-backed state
gemini extensions install https://github.com/gemini-cli-extensions/alloydb
gemini "create a table called agent_state in my AlloyDB database with columns \
  for session_id, task_id, status, result, iteration_count, and timestamps. \
  Use (session_id, task_id) as the primary key."

# Option B: Cloud SQL PostgreSQL-backed state
gemini extensions install https://github.com/gemini-cli-extensions/cloud-sql-postgresql
gemini "create the same agent_state table in my Cloud SQL PostgreSQL instance"

# Verify the schema
gemini "describe the agent_state table and confirm the primary key is correct"

# Test a state read/write cycle
gemini "insert a test row into agent_state with session_id=smoke-test and status=pending, \
  then read it back to confirm the DatabaseSessionService can reach the database"
```

**Skill reference**: See `memory_and_state.md` §1–4 for `DatabaseSessionService` setup, ZDR patterns, and `tools.yaml` DB tool definitions.

---

#### Phase 4 — Local Ralph Loop Development (`ralph_loop_gcp.md` + `ralph`)

The `ralph` extension runs the Ralph Loop locally — no Cloud Run Job or Cloud Scheduler needed during development.

```bash
# Run a single Ralph Loop iteration locally
/ralph:loop "Analyze the payment API 500 errors from the last 24 hours and propose a mitigation plan"

# Run with a specific DEFINE.md task list
/ralph:loop --file .agent/DEFINE.md

# Inspect the loop's reasoning trace after execution
/ralph:status

# Reset the loop state and start fresh
/ralph:reset
```

The `ralph` extension uses the same `root_agent` entry point as the production Cloud Run Job — the only difference is the trigger mechanism (local CLI vs. Cloud Scheduler cron).

| | Local (`ralph` extension) | Production (Cloud Run Job) |
|--|--------------------------|---------------------------|
| Trigger | `/ralph:loop` command | Cloud Scheduler (cron) |
| Outer loop | Extension manages the `while` loop | `ci-cd/outer_loop.py` |
| External memory | Local `.agent/` directory | GCS bucket |
| Agent runtime | Local Python process | Vertex AI Agent Engine |

**Skill reference**: See `ralph_loop_gcp.md` §8 for the `ralph` extension commands and §9 for the production Cloud Run Job + Cloud Scheduler setup.

---

#### Phase 5 — Quality Gates (`evaluation.md` + `security`)

Run quality and security checks **before every deployment**. These integrate directly into the Cloud Build pipeline (Step 2 and Step 3 of `cloudbuild.yaml`).

```bash
# Security scan — detects OWASP top 10, injection risks, credential exposure
gemini extensions install https://github.com/gemini-cli-extensions/security
/security:analyze                    # Review current diff
/security:analyze --json             # Machine-readable output for CI gate

# Code review — AI-powered review before committing
gemini extensions install https://github.com/gemini-cli-extensions/code-review
/code-review                         # Review staged changes in agents/ and tools/

# Run the eval-gate locally before pushing
python eval/run_eval.py --threshold 0.8

# Check which test cases are failing
gemini "run the eval suite against the golden dataset and show me which cases scored below 0.8 and why"
```

**Skill reference**: See `evaluation.md` §3–4 for `run_eval.py` internals and §4 for how the eval-gate blocks Cloud Build on failure.

---

#### Phase 6 — Deployment (`infra_and_cicd.md` + `devops` + `cloud-run-mcp`)

**`devops` extension** — generates and validates the Cloud Build pipeline configuration:

```bash
gemini extensions install https://github.com/gemini-cli-extensions/devops

# Generate a Cloud Build pipeline for the Agent Harness
/cicd:design "I need a Cloud Build pipeline that runs unit tests, a security scan, \
  an eval-gate against a golden dataset, then packages and deploys to Vertex AI Agent Engine \
  in us-central1. Block deployment if eval score is below 0.8."

# Validate the existing cloudbuild.yaml
/cicd:validate ci-cd/cloudbuild.yaml

# Trigger a manual build to test the pipeline
gemini "trigger a Cloud Build run for the agent-harness repository on branch main"
```

**`cloud-run-mcp` extension** — deploys MCP servers (genai-toolbox, custom tools) to Cloud Run:

```bash
# Deploy the genai-toolbox MCP server as a Cloud Run service
gemini "deploy the genai-toolbox server defined in tools.yaml to Cloud Run in us-central1 \
  as a service named agent-toolbox, using the agent-harness-sa service account"

# Deploy the Ralph outer loop as a Cloud Run Job
gemini "create a Cloud Run Job named ralph-outer-loop using image \
  us-central1-docker.pkg.dev/my-project/agent-harness-repo/outer-loop:latest \
  with environment variables GOOGLE_CLOUD_PROJECT, GOOGLE_CLOUD_LOCATION=global, \
  AGENT_ENGINE_ID, and EXTERNAL_MEMORY_BUCKET"

# Check Cloud Run service status
gemini "show the status and latest revision of the agent-toolbox Cloud Run service"
```

**Skill reference**: See `infra_and_cicd.md` §3–4 for all deployment methods and the full `cloudbuild.yaml`.

---

#### Phase 7 — Observability (`error_handling.md` + `observability`)

The `observability` extension queries Cloud Logging, Cloud Trace, and Cloud Monitoring in natural language — no gcloud filter syntax required.

```bash
gemini extensions install https://github.com/gemini-cli-extensions/observability

# ── Logging ───────────────────────────────────────────────────────────────
# Find all failed tool calls in the last hour
gemini "show logs where tool call failed in the agent engine service in the last hour"

# Find Model Armor blocks
gemini "show ERROR logs from the agent-harness Cloud Run service where the message contains 'Model Armor blocked'"

# ── Tracing ───────────────────────────────────────────────────────────────
# Trace a specific session's full reasoning path
gemini "show the Cloud Trace spans for session_id ralph-1a2b3c4d and their durations"

# Identify slow tool calls
gemini "which tool calls took longer than 10 seconds in Cloud Trace this week?"

# ── Monitoring ────────────────────────────────────────────────────────────
# Check if the max_iterations alert fired
gemini "were there any max_iterations alerts in Cloud Monitoring in the last 24 hours?"

# Token usage anomaly detection
gemini "which sessions consumed more than 80,000 tokens in the last 24 hours?"
gemini "what is the average cost per Ralph Loop iteration this week?"

# ── Incident response ─────────────────────────────────────────────────────
# Root cause analysis after a failure
gemini "the enterprise_ralph_loop agent failed at 14:30 UTC today. \
  Show me all ERROR logs, relevant trace spans, and any monitoring alerts \
  in the 10 minutes before and after the failure."
```

**Skill reference**: See `error_handling.md` §7 for the full observability query reference and §10 for the `agents/observability.py` structured logging + Cloud Trace implementation.

---

### End-to-End Integration Architecture

The diagram below shows how extensions, skills, and GCP services connect across the full build lifecycle:

```
┌─────────────────────────────────────────────────────────────────────┐
│  GEMINI CLI  (with GEMINI.md referencing all skills + TRIGGERS.md)  │
├──────────┬──────────┬──────────┬───────────┬──────────┬─────────────┤
│ gcloud   │  ralph   │mcp-tool  │observabi- │security  │  devops /   │
│extension │extension │box ext.  │lity ext.  │extension │cloud-run-   │
│          │          │          │           │          │mcp / vertex │
├──────────┴──────────┴──────────┴───────────┴──────────┴─────────────┤
│          Skill files loaded from GEMINI.md (@-references)            │
│  gcp_setup │ adk_patterns │ agent_harness │ ralph_loop │ memory     │
│  infra     │ error_handling │ evaluation  │ TRIGGERS   │            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ natural language → structured actions
┌──────────────────────────────▼──────────────────────────────────────┐
│                        GOOGLE CLOUD PLATFORM                        │
├─────────────┬──────────────┬──────────────┬────────────┬────────────┤
│  Vertex AI  │  Cloud Run   │  Cloud Build │  Cloud SQL │  Cloud     │
│ Agent Engine│  (MCP tools, │  (CI/CD,     │  / AlloyDB │  Logging / │
│ (us-central1│   outer loop)│   eval-gate) │  (state)   │  Trace /   │
│  only)      │              │              │            │  Monitoring│
└─────────────┴──────────────┴──────────────┴────────────┴────────────┘
```

**Key connection rules:**
- `GOOGLE_CLOUD_LOCATION=global` — always used for Gemini model calls (all extensions respect this)
- `REGION=us-central1` — always used for Agent Engine and infrastructure deployment
- `TRIGGERS.md` — loaded first so the AI agent knows which skill to activate per user request

---

## Build Order — Start Here

Follow this sequence when building an Agent Harness from scratch:

```
1. gcp_setup.md          ← Enable APIs, create IAM, set up local dev
        ↓
2. adk_patterns.md       ← Write agent.py, harness.py with correct patterns
        ↓
3. agent_harness_gcp.md  ← Understand the full 8-layer architecture
        ↓
4. ralph_loop_gcp.md     ← Add Ralph Loop orchestration + ZDR
        ↓
5. memory_and_state.md   ← Wire Memory Bank + DB state backend
        ↓
6. infra_and_cicd.md     ← Deploy with Terraform + Cloud Build
        ↓
7. error_handling.md     ← Add retry, circuit breaker, alerting
        ↓
8. evaluation.md         ← Golden dataset + eval-gate CI/CD
```

---

## Reference Project

All skills are derived from **[sample-agent-harness-on-gcp](https://github.com/hajekim/sample-agent-harness-on-gcp)** — an enterprise-grade Agent Harness built with Google ADK, Vertex AI Agent Engine, Terraform, and Cloud Build.

```
sample-agent-harness-on-gcp/
├── agents/
│   ├── agent.py           # Planner → Worker → Checker → LoopAgent  [Skills 2, 3, 4]
│   ├── harness.py         # ModelHarness + AgentResponse + safety settings  [Skill 2]
│   ├── ralph_loop.py      # ZDR state manager  [Skills 4, 5]
│   ├── callbacks.py       # before/after_model_callback + Model Armor  [Skill 2]
│   └── observability.py   # Structured logging + Cloud Trace setup  [Skill 7]
├── tools/
│   ├── policy_engine.py   # SRE Interception Layer + global_policy singleton  [Skills 3, 7]
│   ├── mcp_client.py      # Remote MCP integration (SSE)  [Skills 2, 5]
│   ├── grounding.py       # RAG/Grounding (Vertex AI Search + Google Search)  [Skill 2]
│   ├── model_armor.py     # sanitize_prompt / sanitize_response  [Skills 2, 3]
│   └── token_budget.py    # check_and_reserve_tokens + cost tracking  [Skill 7]
├── infra/
│   ├── main.tf            # Terraform IaC for Vertex AI Agent Engine  [Skill 6]
│   ├── variables.tf       # Terraform variables (us-central1 validation)  [Skill 6]
│   └── memory_bank_config.py  # Memory Bank provisioning  [Skill 5]
├── ci-cd/
│   ├── cloudbuild.yaml         # Cloud Build CI/CD pipeline  [Skills 6, 8]
│   ├── package_for_terraform.sh  # Artifact builder  [Skill 6]
│   └── outer_loop.py           # Cloud Run Job outer loop (GCS backend)  [Skill 4]
├── eval/
│   └── run_eval.py        # Quality validation + eval-gate  [Skill 8]
├── tests/
│   ├── conftest.py        # ToolContext mock fixtures  [Skill 8]
│   └── test_*.py          # Unit tests per tool  [Skill 8]
└── prompts/
    └── system_instruction.txt  # Agent system prompt  [Skills 2, 3]
```

---

## Related Resources

- [Google Agent Development Kit (ADK)](https://google.github.io/adk-docs/)
- [Vertex AI Agent Engine](https://cloud.google.com/agent-builder/agent-development-kit/overview)
- [MCP Toolbox for Databases](https://github.com/googleapis/genai-toolbox)
- [Build a Remote MCP Server on Cloud Run](https://cloud.google.com/run/docs/tutorials/deploy-remote-mcp-server)
- [Multi-agent AI system in Google Cloud](https://docs.cloud.google.com/architecture/multiagent-ai-system)
- [The Anatomy of an Agent Harness](https://blog.langchain.com/the-anatomy-of-an-agent-harness) — LangChain Blog
- [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — Anthropic Engineering
- [agentic-design-patterns-skills](https://github.com/hajekim/agentic-design-patterns-skills) — Companion skill library (28 agentic design patterns)
