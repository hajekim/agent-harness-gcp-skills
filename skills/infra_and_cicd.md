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
    bucket = "YOUR_PROJECT_ID-tfstate"   # created in gcp_setup.md Section 7
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

  # Step 2: Security scan — check for vulnerabilities in code changes
  - name: 'gcr.io/cloud-builders/gcloud'
    id: security-scan
    entrypoint: bash
    args:
      - -c
      - |
        # Run OSV-Scanner on dependencies
        pip install osv-scanner --quiet
        osv-scanner --lockfile=requirements.txt || true

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

## 6. Deployment Checklist

### Before First Deploy
- [ ] GCS tfstate bucket created (see `gcp_setup.md` Section 7)
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
