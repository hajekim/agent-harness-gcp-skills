# CONTEXT.md — Development Context & Change Log

This file documents the iterative improvement history of the `agent-harness-gcp-skills` library.
Use it to understand why specific design decisions were made and what has changed across sessions.

---

## Current State (as of 2026-03-28)

**Repository**: https://github.com/hajekim/agent-harness-gcp-skills
**Status**: Stable — all planned improvements complete
**Skill files**: 8 skill files + 1 trigger file (`skills/TRIGGERS.md`)

---

## Skill File Section Map

| File | Sections |
|------|----------|
| `adk_patterns.md` | §1–§16 (16 sections) |
| `error_handling.md` | §1–§11 (11 sections) |
| `gcp_setup.md` | §1–§10 (10 sections, including secret rotation pattern) |
| `infra_and_cicd.md` | §1–§8 (8 sections, including VPC private access) |
| `agent_harness_gcp.md` | Layers 1–8 |
| `ralph_loop_gcp.md` | §1–§9 |
| `memory_and_state.md` | §1–§6 |
| `evaluation.md` | §1–§5 |

---

## Change History

### Round 1 — Initial Library Publication
**Commit**: `87fea28`

Initial creation of the skills library covering the full Agent Harness build lifecycle:
- `gcp_setup.md`, `adk_patterns.md`, `agent_harness_gcp.md`, `ralph_loop_gcp.md`
- `memory_and_state.md`, `infra_and_cicd.md`, `error_handling.md`, `evaluation.md`
- `TRIGGERS.md` with English and Korean trigger phrases
- `README.md` with extension-to-skill mapping and phase-by-phase workflow

---

### Round 2 — Bug Fixes & Code Validation
**Commit**: `d0b66c4`

Code validation via `ast.parse` (syntax check) and pure-Python unit tests (no GCP credentials required) identified and fixed the following bugs:

| File | Bug | Fix |
|------|-----|-----|
| `evaluation.md` | `test_grounding.py` mocked `discoveryengine` instead of `genai`; asserted `list` instead of `str` | Rewrote `TestSearchKnowledgeBase` with `@patch("tools.grounding.genai")` and `isinstance(result, str)` |
| `agent_harness_gcp.md` | Layer 5 pseudo-code block marked as ` ```python ` caused `SyntaxError` | Changed to plain ` ``` ` block |
| `ralph_loop_gcp.md` | `execute_shell()` decorator-form function had comment-only body (`SyntaxError`) | Added `pass` statement |

**Design decisions validated during this round:**
- `search_knowledge_base()` returns `str` (not `list`) — uses `genai.Client().models.generate_content()` with `types.VertexAISearch`
- Pseudo-code / architecture diagrams must not be in Python code blocks

---

### Round 3 — High-Priority Improvements
**Commits**: `d0b66c4` (partially), `31c2018`

Four high-priority improvements based on architectural review:

#### 1. ADK Runner §14 (`adk_patterns.md`)
**Why**: `root_agent` defines the agent, but the Runner that executes it was undocumented for local dev and testing.

Added:
- `InMemoryRunner` pattern for local dev and CI tests
- `Runner` + `DatabaseSessionService` for persistent local sessions
- Event processing loop (`is_final_response()`, `get_function_calls()`, `get_function_responses()`)
- Integration test patterns using `pytest.mark.asyncio`
- Runner Checklist

#### 2. Human-in-the-Loop (HitL) §15 (`adk_patterns.md`)
**Why**: Policy Engine handles automatic blocks, but some actions (e.g., `terraform destroy`, prod DB migration) need human judgment, not automatic block.

Added:
- `tools/approval_gate.py` — `request_human_approval()`, `check_approval_status()`, `_send_approval_notification()`
- Pattern: Worker writes `{"status": "APPROVAL_REQUIRED", ...}` → Checker routes to `request_human_approval()` → loop pauses via `tool_context.actions.escalate = True`
- Webhook notification (Slack / PagerDuty) via `APPROVAL_WEBHOOK_URL`
- HitL Checklist

#### 3. ModelHarness env var (`adk_patterns.md` §12, §8; `infra_and_cicd.md`)
**Why**: Hardcoded model ID (`"gemini-3-flash-preview"`) required code changes for every model upgrade.

Changed:
- `self.model_id = os.environ.get("AGENT_MODEL_ID", "gemini-2.5-flash")`
- Added `AGENT_MODEL_ID` to `agents/.env` example (§8) and `infra/main.tf` deployment_spec
- Added `variable "agent_model_id"` to `infra/variables.tf`

#### 4. Rollback Strategy §6 (`infra_and_cicd.md`)
**Why**: Agent Engine has no built-in revision pinning; rollback procedure was undocumented.

Added:
- When-to-rollback decision table (signal → action mapping)
- Agent Engine rollback via Terraform (git tag discipline + `source.tar.gz` rebuild)
- Cloud Run traffic split: `gcloud run services update-traffic --to-revisions=<rev>=100`
- Canary pattern: 10%/90% traffic split
- Terraform state recovery (`terraform state list`, `terraform import`, GCS version restore)
- Cloud Build re-trigger via `git revert`
- Rollback Checklist

---

### Round 4 — Medium-Priority Improvements
**Commit**: `31c2018`

Five medium-priority improvements:

#### 1. ParallelAgent §16 (`adk_patterns.md`)
**Why**: `SequentialAgent` and `LoopAgent` were documented but `ParallelAgent` was missing despite being essential for fan-out/fan-in patterns (e.g., parallel diagnostics).

Added:
- Fan-out/Fan-in pattern: `ParallelAgent` (gather) → Synthesizer Agent (combine)
- State isolation rules: each sub-agent must have a **unique `output_key`** — shared keys cause race conditions
- Nested `LoopAgent` inside `ParallelAgent` with cost multiplication warning
- ParallelAgent Checklist

#### 2. Rate Limiting §11 (`error_handling.md`)
**Why**: `ResourceExhausted` (429) is a common production failure. Retry-with-backoff (§2) handles individual failures but doesn't prevent quota storms from parallel agents.

Added:
- `tools/rate_limiter.py` — `TokenBucket` (thread-safe, blocking `consume()`), `flash_limiter`, `pro_limiter`
- `RATE_LIMIT_FLASH_RPM` / `RATE_LIMIT_PRO_RPM` env vars (default: 60% of quota limit)
- Integration: `acquire_flash_quota()` in `ModelHarness.generate()` and `before_model_callback`
- Cloud Run per-instance guidance (limits apply per instance, not total)

#### 3. Master Environment Variable Reference §10 (`gcp_setup.md`)
**Why**: Env vars were scattered across 8 skill files with no single reference point.

Added:
- Consolidated `agents/.env.example` with all 20+ variables
- Variable-to-skill cross-reference table (Variable → Set In → Consumed By → Skill Reference)

#### 4. `tools.yaml` AlloyDB region (`memory_and_state.md`)
**Why**: `region: us-central1` was hardcoded; should be injected from `$REGION` env var for portability.

Changed: `region: us-central1` → `region: ${REGION}`

#### 5. OSV-Scanner fix (`infra_and_cicd.md`)
**Why**: `pip install osv-scanner` does not exist on PyPI — OSV-Scanner is a Go binary.

Changed: `pip install osv-scanner` → `ghcr.io/google/osv-scanner:latest` container image

---

### Round 5 — Low-Priority Improvements & Reference Fixes
**Commit**: `eda8c4a`

#### 1. Internal Reference Error Fixes
**Why**: Section numbers shifted when §8 (Cloud Storage) was added to `gcp_setup.md`, breaking references in `infra_and_cicd.md`. Cross-reference table also had wrong section numbers.

Fixed:
- `infra_and_cicd.md:29` — tfstate bucket reference: `gcp_setup.md Section 7` → `gcp_setup.md §8`
- `infra_and_cicd.md:516` — deployment checklist: `gcp_setup.md Section 7` → `gcp_setup.md §8`
- `gcp_setup.md` cross-reference table: `DATABASE_URL`, `VERTEX_AI_SEARCH_DATASTORE`, `MODEL_ARMOR_*` changed from §4 (Artifact Registry) → §5 (Secret Manager) — correct section for credential storage

#### 2. Secret Rotation Pattern (`gcp_setup.md §5`)
**Why**: Secret Manager usage was documented (create + read) but rotation procedure was missing. Rotation is a compliance requirement for production.

Added:
- Add new version (never delete old version for rollback capability)
- Validate canary deployment with new secret
- Disable (not delete) old version
- `gcloud scheduler jobs create` for automated 90-day rotation
- Key principle: `versions/latest` means Cloud Run picks up new versions automatically — no code changes needed

#### 3. VPC Private Access Pattern (`infra_and_cicd.md §8`)
**Why**: All services were communicating over public endpoints by default. Enterprise deployments require network isolation between Cloud Run, AlloyDB, and internal MCP servers.

Added:
- Terraform: VPC network, subnet, Serverless VPC Access connector, AlloyDB Private Services Access
- Internal-only ingress for specialist agents (`INGRESS_TRAFFIC_INTERNAL_ONLY`)
- Cloud NAT for outbound access to Gemini global endpoint (required even with `ALL_TRAFFIC` VPC routing)
- Private Service Connect for Vertex AI (optional, highest security)
- VPC Checklist

---

## Key Design Decisions

### Region Decoupling
`GOOGLE_CLOUD_LOCATION=global` (Gemini model endpoint) is intentionally separate from `REGION=us-central1` (infrastructure). Gemini 2.5 models are only available on the global endpoint; Agent Engine is only available in `us-central1`.

### Model ID Placeholder Convention
Model IDs throughout skills use `gemini-2.5-flash` / `gemini-2.5-pro` as current stable defaults, with `AGENT_MODEL_ID` env var allowing runtime override. Previous conceptual placeholders (`gemini-3-*`) replaced with verified production IDs.

### Terraform Agent Engine Region Validation
A `lifecycle` precondition in `infra/main.tf` enforces `region = "us-central1"` at plan time, preventing accidental deployment to unsupported regions.

### Code Validation Without GCP
All skill code is validated via:
1. `ast.parse()` — syntax correctness for every Python block
2. Pure-Python unit tests — logic correctness for non-GCP code
3. GCP-dependent code — syntax-only validation (no live credentials)

### Never Delete Secret Versions
Secret Manager versions are disabled (not deleted) after rotation. Disabled versions can be re-enabled for rollback. The `versions/latest` pointer always resolves to the newest enabled version automatically.

---

## Trigger Coverage (TRIGGERS.md)

~297 triggers total across 9 skill categories:
- **English**: ~149 triggers (noun-form, question, intent, problem-based, comparison, keyword-style, beginner)
- **Korean**: ~148 triggers (noun형, 질문형, 대화형, 문제-기반, 의도형)

---

## Pending / Known Limitations

- Model IDs (`gemini-2.5-flash`, `gemini-2.5-pro`) are current as of 2026-03 — verify with `gcloud ai models list --region=global` before production deployment
- Agent Engine Memory Bank provisioning (`infra/memory_bank_config.py`) is not yet in the Terraform module — documented in `memory_and_state.md` but requires manual run
- VPC Private Access Terraform module (`infra/vpc.tf`) is a reference pattern, not part of the base `infra/main.tf`
