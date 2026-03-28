# Infrastructure & CI/CD Guide

## Overview
This guide covers how to provision and manage Agent Harness infrastructure using Terraform (IaC) and automate the build-test-deploy cycle with Cloud Build.

Related extensions: [`cloud-run-mcp`](https://github.com/GoogleCloudPlatform/cloud-run-mcp), [`gke-mcp`](https://github.com/GoogleCloudPlatform/gke-mcp), [`devops`](https://github.com/gemini-cli-extensions/devops)

---

## 1. Terraform IaC — What Gets Provisioned

The `infra/main.tf` file provisions the following resources for the Agent Harness:

```
infra/
├── main.tf               ← All resource definitions
├── backend.tf            ← GCS remote state backend
├── variables.tf          ← Input variable declarations
├── terraform.tfvars      ← Variable values (gitignored)
└── memory_bank_config.py ← Memory Bank provisioning script (run separately)
```

### Remote State Backend (`infra/backend.tf`)

Always configure remote state before the first `terraform apply`:
```hcl
terraform {
  backend "gcs" {
    bucket = "YOUR_PROJECT_ID-tfstate"   # created in gcp_setup.md §8
    prefix = "agent-harness/state"
  }
}
```

### Core Resource: Vertex AI Agent Engine (`infra/main.tf`)

```hcl
terraform {
  required_providers {
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 7.0"
    }
  }
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}

# Agent Engine runtime (deploys the ADK agent as a managed service)
# ⚠️ Vertex AI Agent Engine is only available in the us-central1 region.
#    var.region is restricted to us-central1 via a validation block.
resource "google_vertex_ai_reasoning_engine" "agent_engine" {
  provider     = google-beta
  display_name = "enterprise-harness-agent"
  region       = var.region   # validation enforces us-central1 only

  spec {
    agent_framework = "google-adk"

    source_code_spec {
      inline_source {
        # Built by ci-cd/package_for_terraform.sh
        source_archive = filebase64("${path.module}/source.tar.gz")
      }
      python_spec {
        version           = "3.13"
        entrypoint_module = "agents.agent_engine_app"
        entrypoint_object = "adk_app"
        requirements_file = "requirements.txt"
      }
    }

    deployment_spec {
      # Region decoupling: container runs in var.region, model calls go to global
      env { name = "GOOGLE_CLOUD_LOCATION";                          value = "global" }
      env { name = "GOOGLE_CLOUD_PROJECT";                           value = var.project_id }
      env { name = "GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY";     value = "true" }
      env { name = "OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT"; value = "true" }
      env { name = "MEMORY_BANK_ID";                                 value = var.memory_bank_id }
      env { name = "AGENT_MODEL_ID";                                 value = var.agent_model_id }
    }
  }
}

# Artifact Registry for container images (MCP servers, Cloud Run Jobs)
resource "google_artifact_registry_repository" "agent_harness" {
  provider      = google-beta
  location      = var.region
  repository_id = "agent-harness-repo"
  format        = "DOCKER"
  description   = "Agent Harness container images"
}

# GCS bucket for Terraform state (created separately — see gcp_setup.md)
# Declared here as a data source to reference in outputs
data "google_storage_bucket" "tfstate" {
  name = "${var.project_id}-tfstate"
}
```

### Variables (`infra/variables.tf`)

```hcl
variable "project_id" {
  type        = string
  description = "GCP Project ID"
}

variable "region" {
  type        = string
  default     = "us-central1"
  description = "Deployment region for Agent Engine and Cloud Run. Agent Engine is only available in us-central1."

  validation {
    condition     = var.region == "us-central1"
    error_message = "Vertex AI Agent Engine is only available in us-central1. Do not change this value."
  }
}

variable "memory_bank_id" {
  type        = string
  description = "Vertex AI Memory Bank resource name (created by memory_bank_config.py)"
  default     = ""
}

variable "agent_model_id" {
  type        = string
  description = "Gemini model ID for ModelHarness — swap without code changes. Verify with: gcloud ai models list --region=global"
  default     = "gemini-2.5-flash"
}
```

### `terraform.tfvars` (gitignored)
```hcl
project_id     = "your-project-id"
region         = "us-central1"
memory_bank_id = "projects/your-project/locations/us-central1/reasoningEngines/123456"
```

---

## 2. `package_for_terraform.sh` — What It Does

The script at `ci-cd/package_for_terraform.sh` bridges the gap between `adk deploy` (which handles packaging internally) and Terraform (which requires a pre-built artifact):

```bash
#!/bin/bash
# ci-cd/package_for_terraform.sh
set -e

echo "=== Building Agent Harness source archive for Terraform ==="

# 1. Install ADK packaging tools
pip install google-adk --quiet

# 2. Use ADK's internal packager to create a deployable archive
#    This generates the agent_engine_app.py wrapper and bundles dependencies
# ⚠️ Agent Engine is only available in us-central1. $REGION must be us-central1.
adk deploy agent_engine agents/ \
  --project $PROJECT_ID \
  --region $REGION \               # must be us-central1
  --dry-run \                      # build artifact only, do not deploy
  --output-dir infra/

# 3. The ADK packager produces source.tar.gz in the output directory
#    Terraform's filebase64() reads this file
echo "=== Artifact ready: infra/source.tar.gz ==="
```

> **Why this script exists**: ADK CLI handles the Agent Engine packaging internally (creating the `agent_engine_app.py` entrypoint, bundling `requirements.txt`). Terraform needs the pre-built archive to use `filebase64()`. The script runs in CI (Step 1 of Tier 3 deployment) so Terraform always receives a reproducible, validated artifact.

---

## 3. Deployment Methods — Full Reference

### Method A: ADK CLI (Tier 2 — Direct Deploy)
```bash
# One command — ADK handles packaging, upload, and Agent Engine provisioning
# ⚠️ Agent Engine is only available in us-central1. $REGION=us-central1 is required.
adk deploy agent_engine agents/ \
  --project $PROJECT_ID \
  --region $REGION \               # must be us-central1
  --display_name "harness-production" \
  --validate-agent-import

# Deploy as Cloud Run service with development UI
adk deploy cloud_run agents/ \
  --project $PROJECT_ID \
  --region $REGION \
  --with-ui
```

### Method B: Terraform GitOps (Tier 3 — Recommended for Production)
```bash
# Step 1: Build the deployable artifact
chmod +x ci-cd/package_for_terraform.sh
./ci-cd/package_for_terraform.sh

# Step 2: Initialize Terraform (first time only)
cd infra
terraform init

# Step 3: Review the plan
terraform plan -var="project_id=$PROJECT_ID" -var="region=$REGION"

# Step 4: Apply
terraform apply -var="project_id=$PROJECT_ID" -var="region=$REGION"
```

### Method C: `cloud-run-mcp` Extension (MCP Server Deployment)

For deploying the MCP servers that the Agent Harness tools connect to:
```bash
# Install
gemini extensions install https://github.com/GoogleCloudPlatform/cloud-run-mcp
# or: npx -y @google-cloud/cloud-run-mcp

# Deploy current directory (MCP server code) to Cloud Run
/deploy

# Or deploy specific files
gemini "deploy the tools/mcp_client.py as a Cloud Run service named agent-harness-mcp"

# Get logs to debug a deployed MCP server
/logs
gemini "show error logs from the agent-harness-mcp Cloud Run service"
```

### Method D: `gke-mcp` Extension (GKE Deployment — Tier 3 Enterprise)
```bash
gemini extensions install https://github.com/GoogleCloudPlatform/gke-mcp

# Create an AI-optimized GKE cluster
gemini "create a GKE cluster optimized for AI inference workloads in us-central1"

# Generate a deployment manifest for the Agent Harness
gemini "generate a GKE manifest for a Python agent application using the agent-harness-sa service account"

# Check upgrade risks before GKE updates
/gke-upgrade-risk-report
```

---

## 4. Cloud Build CI/CD Pipeline

### `ci-cd/cloudbuild.yaml` — The Full Pipeline

```yaml
# ci-cd/cloudbuild.yaml
steps:
  # Step 1: Install dependencies and run unit tests
  - name: 'python:3.13'
    id: test
    entrypoint: bash
    args:
      - -c
      - |
        pip install -e . --quiet
        python -m pytest tests/ -v --tb=short

  # Step 2: Security scan — check for vulnerabilities in dependencies
  # OSV-Scanner is a Go binary distributed as a container image.
  # Do NOT use `pip install osv-scanner` — it does not exist on PyPI.
  - name: 'ghcr.io/google/osv-scanner:latest'
    id: security-scan
    args:
      - '--lockfile=requirements.txt'
      - '--format=table'
    # Exit code 1 = vulnerabilities found. Use `|| true` to treat as non-blocking warning.
    # Remove `|| true` to make the build fail on any vulnerability (recommended for prod).
    # args: ['--lockfile=requirements.txt', '--format=sarif', '--output=osv-results.sarif']

  # Step 3: Run evaluation against golden dataset (eval-gate)
  - name: 'python:3.13'
    id: eval
    entrypoint: bash
    args:
      - -c
      - |
        pip install -e . --quiet
        python eval/run_eval.py --threshold 0.8
    env:
      - 'GOOGLE_CLOUD_PROJECT=${PROJECT_ID}'
      - 'GOOGLE_CLOUD_LOCATION=global'
    waitFor: [test]

  # Step 4: Build the Terraform artifact (only if eval passes)
  - name: 'python:3.13'
    id: package
    entrypoint: bash
    args:
      - -c
      - |
        pip install google-adk --quiet
        ./ci-cd/package_for_terraform.sh
    env:
      - 'PROJECT_ID=${PROJECT_ID}'
      - 'REGION=${_REGION}'
    waitFor: [eval]

  # Step 5: Apply Terraform (deploy to Agent Engine)
  - name: 'hashicorp/terraform:1.8'
    id: deploy
    dir: infra
    entrypoint: sh
    args:
      - -c
      - |
        terraform init -backend-config="bucket=${PROJECT_ID}-tfstate"
        terraform apply -auto-approve \
          -var="project_id=${PROJECT_ID}" \
          -var="region=${_REGION}"
    waitFor: [package]

substitutions:
  _REGION: us-central1

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'
```

### Set Up Cloud Build Trigger

```bash
# Connect GitHub repository
gcloud builds triggers create github \
  --repo-name=sample-agent-harness-on-gcp \
  --repo-owner=YOUR_GITHUB_USER \
  --branch-pattern="^main$" \
  --build-config=ci-cd/cloudbuild.yaml \
  --substitutions="_REGION=us-central1" \
  --project=$PROJECT_ID
```

With the `devops` extension:
```bash
gemini extensions install https://github.com/gemini-cli-extensions/devops

# AI-designed pipeline — generates cloudbuild.yaml with all required GCP infrastructure
/cicd:design

# Deploy the current project (auto-detects Python ADK agent → Agent Engine)
/cicd:deploy
```

---

## 5. Artifact Registry — Image Management

```bash
# Tag and push a container image (e.g., MCP server)
IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/agent-harness-repo/mcp-server:${TAG}"

docker build -t $IMAGE ./tools/
docker push $IMAGE

# Deploy the image to Cloud Run (MCP server)
gcloud run deploy agent-harness-mcp \
  --image=$IMAGE \
  --region=$REGION \
  --no-allow-unauthenticated \
  --service-account=$SA_EMAIL \
  --project=$PROJECT_ID
```

---

## 6. Rollback Strategy

### When to Roll Back

| Situation | Signal | Action |
|-----------|--------|--------|
| Agent quality dropped after deploy | `eval/run_eval.py` pass rate < threshold | Terraform rollback to previous revision |
| Agent Engine deployment failed mid-apply | Terraform error output | `terraform apply` with previous `source.tar.gz` |
| Cloud Run MCP server regression | Error rate spike in Cloud Monitoring | `gcloud run services update-traffic` |
| CI/CD pipeline broke — need to restore known good | Cloud Build failed | Git revert + re-trigger |

---

### Agent Engine Rollback via Terraform

Agent Engine does not support revision pinning like Cloud Run. Rollback means re-deploying the previous artifact.

```bash
# Step 1: Identify the last known-good Git commit
git log --oneline ci-cd/ infra/ agents/ | head -10

# Step 2: Check out the previous source archive from Git history
#         (assumes package_for_terraform.sh output is committed, or rebuild from tag)
git checkout <previous-commit> -- infra/source.tar.gz

# Step 3: Re-apply Terraform with the restored artifact
cd infra
terraform apply -var="project_id=$PROJECT_ID" -var="region=us-central1" -auto-approve

# Step 4: Verify agent quality with eval
python eval/run_eval.py --threshold 0.8

# Step 5: Tag the rollback event
git tag rollback-$(date +%Y%m%d-%H%M) && git push origin --tags
```

> **Best practice**: Always tag deployments (`git tag deploy-YYYY-MM-DD-HH-MM`) so `source.tar.gz` can be reconstructed from a known commit.

### Agent Engine Rollback — Alternative: Rebuild from Tag

If `source.tar.gz` is not committed (preferred for large binaries), rebuild from the tagged commit:

```bash
# Rebuild artifact from a known-good tag
git checkout deploy-2025-03-28-1400

./ci-cd/package_for_terraform.sh  # regenerates infra/source.tar.gz

cd infra && terraform apply \
  -var="project_id=$PROJECT_ID" \
  -var="region=us-central1" \
  -auto-approve
```

---

### Cloud Run MCP Server Rollback (Traffic Split)

Cloud Run keeps previous revisions. Roll back instantly without redeployment:

```bash
# List revisions for the MCP server service
gcloud run revisions list \
  --service=agent-harness-mcp \
  --region=$REGION \
  --project=$PROJECT_ID

# Instantly shift 100% traffic back to the previous revision
gcloud run services update-traffic agent-harness-mcp \
  --region=$REGION \
  --project=$PROJECT_ID \
  --to-revisions=agent-harness-mcp-00005-abc=100   # previous revision name

# Canary: send 10% to new, 90% to previous (for gradual validation)
gcloud run services update-traffic agent-harness-mcp \
  --region=$REGION \
  --project=$PROJECT_ID \
  --to-revisions=agent-harness-mcp-00006-xyz=10,agent-harness-mcp-00005-abc=90
```

---

### Terraform State Recovery

If `terraform apply` failed mid-run and the state is inconsistent:

```bash
# 1. Check what Terraform currently knows
terraform state list

# 2. Inspect a specific resource
terraform state show google_vertex_ai_reasoning_engine.agent_engine

# 3. If a resource was created outside Terraform (drift), import it
terraform import google_vertex_ai_reasoning_engine.agent_engine \
  projects/$PROJECT_ID/locations/us-central1/reasoningEngines/$ENGINE_ID

# 4. Restore previous state from GCS versioned backup
gcloud storage objects list gs://${PROJECT_ID}-tfstate/agent-harness/state \
  --versions
gcloud storage cp \
  "gs://${PROJECT_ID}-tfstate/agent-harness/state#<generation>" \
  ./terraform.tfstate.backup
```

> **Prevention**: GCS versioning on the tfstate bucket (enabled in `gcp_setup.md §8`) means every state change is preserved — previous states can be restored from GCS object versions.

---

### Cloud Build — Re-trigger Previous Successful Build

```bash
# List recent builds and their status
gcloud builds list --limit=10 --project=$PROJECT_ID

# Re-trigger the last successful build
LAST_GOOD_BUILD=$(gcloud builds list \
  --filter="status=SUCCESS" \
  --limit=1 \
  --format="value(id)" \
  --project=$PROJECT_ID)

gcloud builds log $LAST_GOOD_BUILD --project=$PROJECT_ID  # verify it's the right one

# Re-run the same build configuration from git
git revert HEAD --no-edit && git push origin main  # triggers Cloud Build via branch trigger
```

---

### Rollback Checklist

- [ ] Git tags on every production deploy (`deploy-YYYY-MM-DD-HH-MM`)
- [ ] GCS versioning enabled on tfstate bucket (prevents state loss)
- [ ] `eval/run_eval.py` run after every rollback to confirm quality restored
- [ ] Cloud Run traffic split used for gradual rollout before full cutover
- [ ] Rollback event logged in Cloud Logging for audit trail

---

## 7. Deployment Checklist

### Before First Deploy
- [ ] GCS tfstate bucket created (see `gcp_setup.md §8`)
- [ ] `infra/backend.tf` configured with bucket name
- [ ] `terraform.tfvars` created with `project_id` and `region` (gitignored)
- [ ] Service account has all required IAM roles
- [ ] Artifact Registry repository created
- [ ] Memory Bank provisioned via `infra/memory_bank_config.py`

### Method B (Terraform) Checklist
- [ ] `./ci-cd/package_for_terraform.sh` completed without errors
- [ ] `infra/source.tar.gz` exists
- [ ] `terraform init` succeeded (remote backend connected)
- [ ] `terraform plan` shows expected resources only

### Method C (cloud-run-mcp) Checklist
- [ ] Extension installed: `gemini extensions install .../cloud-run-mcp`
- [ ] `gcloud auth application-default login` completed
- [ ] MCP server code tested locally before deploy

### CI/CD (Method C) Checklist
- [ ] Cloud Build trigger created and linked to repository
- [ ] `${PROJECT_ID}` substitution variable set in trigger
- [ ] `eval/run_eval.py` returns exit code 0 on passing quality threshold
- [ ] Cloud Build service account has `roles/aiplatform.user` to run eval

---

## 8. VPC Private Access — Network Isolation

By default, Cloud Run services and AlloyDB communicate over public endpoints. For production enterprise deployments, private networking eliminates public internet exposure between internal services.

### Architecture

```
Cloud Run (agent-harness)
    │
    │  Private IP (VPC connector)
    ▼
VPC Network (agent-harness-vpc)
    ├── Cloud Run → AlloyDB (Private Service Connect)
    ├── Cloud Run → MCP Toolbox (internal load balancer)
    └── Cloud Run → Cloud Run (service-to-service, internal ingress)
```

### Terraform — VPC + Serverless VPC Connector

```hcl
# infra/vpc.tf

# VPC network
resource "google_compute_network" "agent_harness_vpc" {
  name                    = "agent-harness-vpc"
  auto_create_subnetworks = false
  project                 = var.project_id
}

# Subnet for Cloud Run workloads
resource "google_compute_subnetwork" "agent_harness_subnet" {
  name          = "agent-harness-subnet"
  ip_cidr_range = "10.8.0.0/28"
  region        = var.region
  network       = google_compute_network.agent_harness_vpc.id
  project       = var.project_id
}

# Serverless VPC Access connector — bridges Cloud Run to the VPC
resource "google_vpc_access_connector" "agent_harness_connector" {
  name          = "agent-harness-connector"
  region        = var.region
  project       = var.project_id
  network       = google_compute_network.agent_harness_vpc.name
  ip_cidr_range = "10.8.1.0/28"   # Separate /28 range for the connector

  min_instances = 2
  max_instances = 10
}

# AlloyDB private service connection
resource "google_compute_global_address" "alloydb_private_ip" {
  name          = "alloydb-private-ip"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.agent_harness_vpc.id
  project       = var.project_id
}

resource "google_service_networking_connection" "alloydb_private_vpc" {
  network                 = google_compute_network.agent_harness_vpc.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.alloydb_private_ip.name]
}
```

### Attach the VPC Connector to Cloud Run

```hcl
# infra/main.tf — Cloud Run service with VPC connector
resource "google_cloud_run_v2_service" "agent_harness" {
  name     = var.service_name
  location = var.region
  project  = var.project_id

  template {
    vpc_access {
      connector = google_vpc_access_connector.agent_harness_connector.id
      egress    = "ALL_TRAFFIC"   # Route all outbound traffic through VPC
      # egress = "PRIVATE_RANGES_ONLY"  # Only private IPs through VPC (public APIs still direct)
    }

    # ... rest of template (containers, env, etc.)
  }
}
```

### Internal-Only Ingress for Service-to-Service Calls

When Cloud Run services call each other (e.g., Coordinator → Specialist Agent), restrict ingress to internal traffic only:

```hcl
# The specialist agent only accepts internal (VPC) traffic
resource "google_cloud_run_v2_service" "sre_specialist_agent" {
  name     = "sre-specialist-agent"
  location = var.region
  project  = var.project_id

  ingress = "INGRESS_TRAFFIC_INTERNAL_ONLY"   # Block all public internet traffic

  # ... template
}
```

```bash
# Equivalent gcloud command
gcloud run services update sre-specialist-agent \
  --ingress=internal \
  --region=$REGION \
  --project=$PROJECT_ID
```

### Cloud NAT — Outbound Access to Public APIs (Gemini global endpoint)

With `ALL_TRAFFIC` routing through VPC, Cloud Run loses direct internet access. Cloud NAT restores outbound access to public APIs (Gemini, Secret Manager, etc.) without exposing inbound ports:

```hcl
# infra/vpc.tf — Cloud Router + NAT
resource "google_compute_router" "agent_harness_router" {
  name    = "agent-harness-router"
  region  = var.region
  network = google_compute_network.agent_harness_vpc.id
  project = var.project_id
}

resource "google_compute_router_nat" "agent_harness_nat" {
  name                               = "agent-harness-nat"
  router                             = google_compute_router.agent_harness_router.name
  region                             = var.region
  project                            = var.project_id
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}
```

> **Why Cloud NAT**: The Gemini global endpoint (`generativelanguage.googleapis.com`) is a public API. Even with full VPC routing, the agent must be able to reach it. Cloud NAT provides outbound-only internet access without any inbound exposure.

### Private Service Connect for Vertex AI (Optional — High Security)

For stricter environments, use Private Service Connect to reach Vertex AI endpoints without traversing the public internet:

```bash
# Create a Private Service Connect endpoint for Vertex AI
gcloud compute addresses create vertexai-psc-endpoint \
  --global \
  --purpose=PRIVATE_SERVICE_CONNECT \
  --addresses=10.8.2.0 \
  --network=agent-harness-vpc \
  --project=$PROJECT_ID

gcloud compute forwarding-rules create vertexai-psc-rule \
  --global \
  --network=agent-harness-vpc \
  --address=vertexai-psc-endpoint \
  --target-google-apis-bundle=all-apis \
  --project=$PROJECT_ID
```

### VPC Private Access Checklist

- [ ] `agent-harness-vpc` and `agent-harness-subnet` created via Terraform
- [ ] Serverless VPC Access connector created and attached to Cloud Run
- [ ] AlloyDB provisioned with private IP via Private Services Access
- [ ] `DATABASE_URL` uses private IP (not public endpoint)
- [ ] Internal-only ingress set for specialist agents (`INGRESS_TRAFFIC_INTERNAL_ONLY`)
- [ ] Cloud NAT configured if using `egress = "ALL_TRAFFIC"`
- [ ] Verify private connectivity: `gcloud run services describe` shows VPC connector attached
