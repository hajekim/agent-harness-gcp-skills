# GCP Setup Guide — Prerequisites & Local Development Workflow

## Overview
Everything that must be in place **before** writing or deploying any Agent Harness code.
This guide covers GCP project configuration, required APIs, IAM, authentication, and local development workflow.

The [`gcloud` extension](https://github.com/gemini-cli-extensions/gcloud) automates most of these steps via natural language inside Gemini CLI.

### Install the `gcloud` Extension
```bash
gemini extensions install https://github.com/gemini-cli-extensions/gcloud
# Verify
gemini mcp list
```
Once installed, the extension exposes a `run_gcloud_command` MCP tool. You can then ask Gemini to perform any `gcloud` operation in plain English instead of memorizing command syntax.

---

## 1. Project & Authentication

### Set active project and authenticate
```bash
gcloud config set project YOUR_PROJECT_ID
gcloud auth login                          # browser-based user auth
gcloud auth application-default login      # ADC for SDK / local dev
```

With the `gcloud` extension:
```
gemini "set my active GCP project to YOUR_PROJECT_ID and authenticate with application default credentials"
```

### Required environment variables (set in shell or `.env`)
```bash
export PROJECT_ID="your-project-id"
export REGION="us-central1"              # Agent Engine / Cloud Run deployment region
export GOOGLE_CLOUD_PROJECT=$PROJECT_ID
export GOOGLE_CLOUD_LOCATION="global"   # Gemini 3 model endpoint — keep as global
```

---

## 2. Enable Required GCP APIs

Run all at once:
```bash
gcloud services enable \
  aiplatform.googleapis.com \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com \
  cloudtrace.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com \
  storage.googleapis.com \
  iam.googleapis.com \
  modelarmor.googleapis.com \
  cloudscheduler.googleapis.com \
  discoveryengine.googleapis.com \
  --project=$PROJECT_ID
```

With the `gcloud` extension:
```
gemini "enable all APIs required for an Agent Harness on GCP: Vertex AI, Cloud Run, Cloud Build, Artifact Registry, Secret Manager, Cloud Trace, Logging, Monitoring, Cloud Storage, and IAM"
```

### API Purpose Reference

| API | Purpose in Agent Harness |
|-----|--------------------------|
| `aiplatform.googleapis.com` | Vertex AI Agent Engine, Gemini models, Memory Bank |
| `run.googleapis.com` | Cloud Run (MCP servers, Ralph outer loop jobs) |
| `cloudbuild.googleapis.com` | CI/CD pipeline |
| `artifactregistry.googleapis.com` | Container image storage |
| `secretmanager.googleapis.com` | API keys, DB credentials |
| `cloudtrace.googleapis.com` | Agent reasoning trace (Layer 7) |
| `logging.googleapis.com` | Centralized log collection |
| `monitoring.googleapis.com` | Metrics, dashboards, alerts |
| `storage.googleapis.com` | Terraform state backend, artifact storage |
| `iam.googleapis.com` | Service accounts, per-agent identities |
| `modelarmor.googleapis.com` | Model Armor input/output inspection (prompt injection defense) |
| `cloudscheduler.googleapis.com` | External trigger for production Ralph Loop |
| `discoveryengine.googleapis.com` | Vertex AI Search — required for RAG/Grounding (see `adk_patterns.md §10`) |

---

## 3. IAM — Service Accounts & Roles

### Create a dedicated service account for the Agent Harness
```bash
# Create service account
gcloud iam service-accounts create agent-harness-sa \
  --display-name="Agent Harness Service Account" \
  --project=$PROJECT_ID

export SA_EMAIL="agent-harness-sa@${PROJECT_ID}.iam.gserviceaccount.com"
```

### Assign minimum required roles
```bash
# Vertex AI — run agents, access models, use Memory Bank
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/aiplatform.user"

# Cloud Run — deploy and invoke MCP servers
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/run.developer"

# Cloud Build — trigger CI/CD pipelines
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/cloudbuild.builds.editor"

# Artifact Registry — push/pull container images
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/artifactregistry.writer"

# Secret Manager — read API keys and credentials
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/secretmanager.secretAccessor"

# Cloud Storage — read/write Terraform state and artifacts
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/storage.objectAdmin"

# Logging & Monitoring — write telemetry
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/logging.logWriter"
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/monitoring.metricWriter"
```

With the `gcloud` extension:
```
gemini "create a service account called agent-harness-sa and grant it the minimum roles needed to run an ADK agent on Vertex AI Agent Engine, deploy MCP servers to Cloud Run, and write logs and traces"
```

### IAM Role Summary

| Role | Layer | Required For |
|------|-------|-------------|
| `roles/aiplatform.user` | 2, 3, 5 | Agent Engine, Gemini API, Memory Bank |
| `roles/run.developer` | 2, 4 | Cloud Run deployment, MCP server invoke |
| `roles/cloudbuild.builds.editor` | 2 | CI/CD pipeline execution |
| `roles/artifactregistry.writer` | 2 | Container image push |
| `roles/secretmanager.secretAccessor` | 6 | Read credentials at runtime |
| `roles/storage.objectAdmin` | 2, 5 | Terraform state, ZDR state files |
| `roles/logging.logWriter` | 7 | Cloud Logging telemetry |
| `roles/monitoring.metricWriter` | 7 | Cloud Monitoring metrics |

---

## 4. Artifact Registry — Container Repository

```bash
gcloud artifacts repositories create agent-harness-repo \
  --repository-format=docker \
  --location=$REGION \
  --description="Agent Harness container images" \
  --project=$PROJECT_ID

# Authenticate Docker to the registry
gcloud auth configure-docker ${REGION}-docker.pkg.dev
```

---

## 5. Secret Manager — Store Credentials

```bash
# Store database password (example)
echo -n "your-db-password" | gcloud secrets create agent-harness-db-password \
  --data-file=- \
  --project=$PROJECT_ID

# Grant the service account access
gcloud secrets add-iam-policy-binding agent-harness-db-password \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/secretmanager.secretAccessor" \
  --project=$PROJECT_ID
```

Read a secret at runtime (Python):
```python
from google.cloud import secretmanager

def get_secret(secret_id: str, project_id: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")
```

### Secret Rotation Pattern

Secret Manager supports automatic rotation via Cloud Scheduler + Cloud Functions. The recommended pattern for the Agent Harness:

**Step 1 — Add a new secret version (do not delete the old one yet)**

```bash
# Add new version with updated credentials
echo -n "new-db-password" | gcloud secrets versions add agent-harness-db-password \
  --data-file=- \
  --project=$PROJECT_ID

# Verify version list
gcloud secrets versions list agent-harness-db-password --project=$PROJECT_ID
```

**Step 2 — Validate the new secret is working**

The `get_secret()` call always reads `versions/latest`, so a new Cloud Run revision automatically picks up the new version on the next call. Validate with a canary deployment (see `infra_and_cicd.md §6`).

**Step 3 — Disable the old version (not delete — keep for rollback)**

```bash
# Disable old version (VERSION = previous version number, e.g. "1")
gcloud secrets versions disable VERSION \
  --secret=agent-harness-db-password \
  --project=$PROJECT_ID

# If rollback is needed: re-enable old version
gcloud secrets versions enable VERSION \
  --secret=agent-harness-db-password \
  --project=$PROJECT_ID
```

**Automated rotation with Cloud Scheduler:**

```bash
# Create a Cloud Scheduler job to trigger rotation every 90 days
gcloud scheduler jobs create http rotate-agent-harness-secrets \
  --schedule="0 2 1 */3 *" \
  --uri="https://REGION-cloudfunctions.net/rotate-secret" \
  --message-body='{"secret_id": "agent-harness-db-password"}' \
  --oidc-service-account-email=$SA_EMAIL \
  --project=$PROJECT_ID \
  --location=$REGION
```

> **Key principle**: Always add a new version before disabling the old one. Never delete secret versions — disabled versions can be re-enabled for rollback. Cloud Run reads `versions/latest` automatically, so no code changes are needed after rotation.

---

## 6. `requirements.txt` — Core Dependencies

```text
# requirements.txt
google-cloud-aiplatform[adk,agent_engines]   # ADK + Vertex AI Agent Engine
google-genai                                  # Gemini API (genai.Client)
pydantic                                      # Structured output for AgentResponse
pyyaml                                        # tools.yaml parsing (genai-toolbox)
mcp                                           # MCP client (Remote MCP integration)
httpx                                         # A2A client HTTP calls
google-auth                                   # Cloud Run ID token auth (A2A)
google-cloud-modelarmor                       # Model Armor input/output inspection
google-cloud-storage                          # External memory for Cloud Run Jobs (GCS)
opentelemetry-sdk                             # Cloud Trace — span creation
opentelemetry-exporter-gcp-trace             # Cloud Trace exporter
opentelemetry-instrumentation                 # Auto-instrumentation support
```

> **Install**: `pip install -r requirements.txt` or `pip install -e .` (when using pyproject.toml)

---

## 7. Local Development Workflow

### `pyproject.toml` — Project Package Definition

`pyproject.toml` must exist at the project root for `pip install -e .` to work.

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "agent-harness"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "google-cloud-aiplatform[adk,agent_engines]",
    "google-genai",
    "pydantic",
    "pyyaml",
    "mcp",
    "httpx",
    "google-auth",
    "google-cloud-modelarmor",
    "google-cloud-storage",
    "opentelemetry-sdk",
    "opentelemetry-exporter-gcp-trace",
    "opentelemetry-instrumentation",
]

[project.optional-dependencies]
dev = [
    "pytest",
    "pytest-asyncio",
    "pytest-cov",
]

[tool.setuptools.packages.find]
where = ["."]
include = ["agents*", "tools*"]

[tool.pytest.ini_options]
asyncio_mode = "auto"          # pytest-asyncio: automatically applied to all async tests
testpaths = ["tests"]
```

> **`asyncio_mode = "auto"`**: With this setting, you do not need to add `@pytest.mark.asyncio` to each async test function.

---

### Project setup
```bash
# 1. Clone and enter the project
git clone https://github.com/hajekim/sample-agent-harness-on-gcp.git
cd sample-agent-harness-on-gcp

# 2. Create virtual environment
python3 -m venv .venv
source .venv/bin/activate          # macOS / Linux
# .venv\Scripts\activate           # Windows

# 3. Install in editable mode (includes all dependencies + dev tools)
pip install -e ".[dev]"

# 4. Copy and configure environment variables
cp .env.example agents/.env
# Edit agents/.env: set GOOGLE_CLOUD_PROJECT and verify GOOGLE_CLOUD_LOCATION=global
```

### Local testing with ADK

```bash
# Option A: Browser-based UI (recommended for interactive testing)
adk web agents/
# Opens http://localhost:8000 — chat with the agent in a browser

# Option B: Single query via CLI
adk run agents/ "Analyze the payment API 500 errors and propose a mitigation plan"

# Option C: Run the raw harness module directly
python agents/harness.py
python agents/ralph_loop.py
```

### Local Development Checklist
- [ ] `pyproject.toml` exists at the project root
- [ ] `gcloud auth application-default login` completed
- [ ] `agents/.env` created from `.env.example` with correct `GOOGLE_CLOUD_PROJECT`
- [ ] `GOOGLE_CLOUD_LOCATION=global` set (do not change to a regional endpoint)
- [ ] Virtual environment activated (`source .venv/bin/activate`)
- [ ] `pip install -e ".[dev]"` completed successfully (including dev dependencies)
- [ ] `pytest tests/` passes locally before pushing
- [ ] `adk web agents/` launches without errors at `localhost:8000`

---

## 8. Cloud Storage — Terraform State Backend

Before running Terraform, create a GCS bucket to store state remotely:
```bash
gcloud storage buckets create gs://${PROJECT_ID}-tfstate \
  --location=$REGION \
  --project=$PROJECT_ID

# Enable versioning for state recovery
gcloud storage buckets update gs://${PROJECT_ID}-tfstate \
  --versioning
```

Add the backend block to `infra/main.tf` (or create `infra/backend.tf`):
```hcl
terraform {
  backend "gcs" {
    bucket = "YOUR_PROJECT_ID-tfstate"
    prefix = "agent-harness/state"
  }
}
```

---

## 9. Verification Checklist

Run these to confirm the environment is ready before deployment:

```bash
# APIs enabled
gcloud services list --enabled --filter="name:aiplatform OR name:run OR name:cloudbuild" \
  --project=$PROJECT_ID

# Service account exists with correct roles
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:agent-harness-sa"

# ADK can reach Vertex AI
adk run agents/ "Hello — confirm you can reach Gemini 3"

# Terraform state backend accessible
cd infra && terraform init
```

With the `gcloud` extension:
```
gemini "verify that all required APIs are enabled and that the agent-harness-sa service account has the correct IAM roles in project YOUR_PROJECT_ID"
```

---

## 10. Master Environment Variable Reference

A single consolidated reference for every environment variable used across the Agent Harness stack. Copy this to `agents/.env.example` and fill in the values.

```bash
# ═══════════════════════════════════════════════════════════════════════
# agents/.env.example — Master environment variable reference
# Copy to agents/.env for local development.
# Production values are injected via infra/main.tf deployment_spec
# and Secret Manager (see §4).
# ═══════════════════════════════════════════════════════════════════════

# ── GCP Core ───────────────────────────────────────────────────────────
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=global          # Gemini global endpoint — do NOT set to us-central1
REGION=us-central1                    # Infrastructure deployment region (Cloud Run, AlloyDB, etc.)
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account-key.json  # Local dev only

# ── Agent Model ─────────────────────────────────────────────────────────
AGENT_MODEL_ID=gemini-2.5-flash       # Override model without code changes (see adk_patterns.md §12)
                                      # Verify available IDs: gcloud ai models list --region=global

# ── ADK / Agent Engine ─────────────────────────────────────────────────
GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
AGENT_ENGINE_RESOURCE_NAME=projects/PROJECT_ID/locations/us-central1/reasoningEngines/RESOURCE_ID

# ── Database / Session Persistence (ZDR) ───────────────────────────────
DATABASE_URL=postgresql://user:password@host:5432/harness_db
DB_USER=harness_user
DB_PASSWORD=your-db-password          # Inject from Secret Manager in production

# ── MCP Toolbox ────────────────────────────────────────────────────────
MCP_TOOLBOX_URL=http://localhost:5000/sse   # Local dev (genai-toolbox server)
# MCP_TOOLBOX_URL=https://mcp-toolbox-<hash>.run.app/sse  # Production

# ── Vertex AI Search (RAG / Grounding) ─────────────────────────────────
VERTEX_AI_SEARCH_DATASTORE=projects/PROJECT_ID/locations/global/collections/default_collection/dataStores/DATASTORE_ID

# ── Model Armor ─────────────────────────────────────────────────────────
MODEL_ARMOR_TEMPLATE_ID=your-template-id
MODEL_ARMOR_LOCATION=us-central1      # Model Armor service region (regional API — NOT global)
                                      # Independent of GOOGLE_CLOUD_LOCATION=global

# ── A2A Specialist Agents ───────────────────────────────────────────────
SRE_AGENT_URL=https://sre-specialist-agent-<hash>-uc.a.run.app
ARCH_AGENT_URL=https://architect-agent-<hash>-uc.a.run.app

# ── Human-in-the-Loop Approvals ─────────────────────────────────────────
APPROVAL_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK

# ── Rate Limiting (see error_handling.md §11) ───────────────────────────
RATE_LIMIT_FLASH_RPM=600              # 60% of 1,000 RPM quota — leave headroom for bursts
RATE_LIMIT_PRO_RPM=200               # ~55% of 360 RPM quota

# ── Observability ───────────────────────────────────────────────────────
LOG_LEVEL=INFO                        # DEBUG | INFO | WARNING | ERROR
```

### Variable-to-Skill Cross-Reference

| Variable | Set In | Consumed By | Skill Reference |
|----------|--------|-------------|-----------------|
| `GOOGLE_CLOUD_PROJECT` | §1 | All | gcp_setup.md §1 |
| `GOOGLE_CLOUD_LOCATION` | §1 | `ModelHarness`, ADK agent | adk_patterns.md §8 |
| `REGION` | §1 | Terraform, Cloud Run | infra_and_cicd.md §1 |
| `AGENT_MODEL_ID` | §10 | `ModelHarness` | adk_patterns.md §12 |
| `DATABASE_URL` | §5 | `DatabaseSessionService`, MCP Toolbox | memory_and_state.md §2 |
| `VERTEX_AI_SEARCH_DATASTORE` | §5 | `search_knowledge_base` | adk_patterns.md §10 |
| `MODEL_ARMOR_TEMPLATE_ID` | §5 | `tools/model_armor.py` | agent_harness_gcp.md Layer 6 |
| `MODEL_ARMOR_LOCATION` | §5 | `tools/model_armor.py` | agent_harness_gcp.md Layer 6 |
| `APPROVAL_WEBHOOK_URL` | §10 | `tools/approval_gate.py` | adk_patterns.md §15 |
| `RATE_LIMIT_FLASH_RPM` | §10 | `tools/rate_limiter.py` | error_handling.md §11 |
| `MCP_TOOLBOX_URL` | §10 | `tools/state_tools.py` | adk_patterns.md §9 |
