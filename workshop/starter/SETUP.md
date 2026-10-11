# AI Creative Studio — Setup Guide

A multi-agent system that turns a single prompt into a complete Instagram campaign. Six specialized agents collaborate over A2A protocol, built on Google ADK, Cloud Run, and Vertex AI Agent Engine.

**Agents:** `creative_director` (orchestrator) · `brand_strategist` · `copywriter` · `designer` · `critic` · `project_manager`

---

## Prerequisites

- Python 3.11+
- [uv](https://astral.sh/uv) package manager
- Google Cloud project with billing enabled
- Vertex AI, Cloud Run, Cloud Build, Artifact Registry, and Cloud Storage APIs enabled

```bash
gcloud services enable \
  aiplatform.googleapis.com \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  storage.googleapis.com \
  artifactregistry.googleapis.com
```

---

## Environment Setup

```bash
cp .env.example .env
```

Fill in the required values:

| Variable | Description | Required |
|---|---|---|
| `GOOGLE_CLOUD_PROJECT` | GCP project ID | ✅ |
| `GOOGLE_CLOUD_PROJECT_NUMBER` | Numeric project ID — `gcloud projects describe $PROJECT_ID --format="value(projectNumber)"` | ✅ |
| `CLOUD_RUN_REGION` | Region for Cloud Run and Agent Engine, e.g. `us-central1` | ✅ |
| `GCS_IMAGES_BUCKET` | GCS bucket name for generated images | ✅ |
| `GEMINI_MODEL` | Model for the Creative Director, e.g. `gemini-3.6-flash` | ✅ |
| `GEMINI_IMAGE_MODEL` | Image generation model, e.g. `gemini-3.1-flash-image` | ✅ |
| `NOTION_TOKEN` | Notion integration for the project manager agent | optional |

Then authenticate:

```bash
gcloud auth application-default login
```

---

## Running Locally

### Single agent (quickest start)

Opens the ADK dev UI at `http://localhost:8000`. Pick any agent from the dropdown.

```bash
uv run adk web agents --allow_origins='*'
```

### Full multi-agent pipeline

Start each specialist in a separate terminal:

```bash
# Terminal 1
PORT=8082 uv run python agents/brand_strategist/agent.py

# Terminal 2
PORT=8083 uv run python agents/copywriter/agent.py

# Terminal 3
PORT=8084 uv run python agents/designer/agent.py

# Terminal 4
PORT=8085 uv run python agents/critic/agent.py

# Terminal 5
PORT=8086 uv run python agents/project_manager/agent.py
```

Add the URLs to `.env`:

```
STRATEGIST_AGENT_URL=http://localhost:8082
COPYWRITER_AGENT_URL=http://localhost:8083
DESIGNER_AGENT_URL=http://localhost:8084
CRITIC_AGENT_URL=http://localhost:8085
PM_AGENT_URL=http://localhost:8086
```

Then launch the orchestrator:

```bash
# Terminal 6 — opens at localhost:8000, select creative_director
uv run adk web agents --allow_origins='*'
```

> The Creative Director only connects to agents whose URL is set in `.env` — partial setups work fine.

---

## Cloud Deployment

### 1. Create GCS bucket

```bash
gsutil mb -l us-central1 gs://YOUR_PROJECT_ID-campaign-images
```

### 2. Deploy specialists to Cloud Run

Builds and pushes Docker images for all 5 specialists. Agent URLs are written back to `.env` automatically.

```bash
uv run deploy/deploy_all_specialists.py
```

> **Note:** Dockerfiles use `--platform=linux/amd64`. Required when building on Apple Silicon for Cloud Run compatibility.

### 3. Deploy Creative Director to Vertex AI Agent Engine

Packages the orchestrator and deploys it to Agent Engine. `AGENT_ENGINE_ID` is written back to `.env` when complete. Takes 5–10 minutes.

```bash
uv run deploy/deploy_orchestrator.py --action deploy
```

### 4. Run a campaign end-to-end

```bash
uv run run_campaign.py
```

Edit the campaign brief inside `run_campaign.py` to test with your own prompt.

### Teardown

Deletes all Cloud Run services and the Agent Engine resource.

```bash
bash deploy/teardown_gcp.sh
```

---

## Redeploying After Changes

Any change to agent code, models (`.env`), or the Creative Director's system prompt requires a redeploy. Run specialists first, then the orchestrator.

```bash
# Redeploy specialists (model or env changes)
uv run deploy/deploy_all_specialists.py

# Redeploy Creative Director (prompt or model changes)
uv run deploy/deploy_orchestrator.py --action deploy
```
