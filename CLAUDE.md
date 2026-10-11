# AI Creative Studio — CLAUDE.md

## Project Overview

Multi-agent system that turns a single prompt into a complete Instagram campaign.
Six specialized agents collaborate over A2A protocol, built on Google ADK, Cloud Run, and Vertex AI Agent Engine.

**Working directory for all commands:** `workshop/starter/`

## Agent Architecture

| Agent | Role | Default Port (local) |
|---|---|---|
| `brand_strategist` | Market research, competitor analysis | 8082 |
| `copywriter` | Instagram captions (3 variations) | 8083 |
| `designer` | Image generation via Gemini, uploads to GCS | 8084 |
| `critic` | Reviews copy + images, enforces quality gate | 8085 |
| `project_manager` | Campaign timeline + Notion integration | 8086 |
| `creative_director` | Orchestrator — calls all specialists via A2A | ADK web UI |

## Running Locally

### Single agent (ADK dev UI)
```bash
cd workshop/starter
uv run adk web agents --allow_origins='*'
# Opens http://localhost:8000 — pick any agent
```

### Full multi-agent pipeline (all specialists + orchestrator)
Start each specialist in a separate terminal:
```bash
PORT=8082 uv run python agents/brand_strategist/agent.py
PORT=8083 uv run python agents/copywriter/agent.py
PORT=8084 uv run python agents/designer/agent.py
PORT=8085 uv run python agents/critic/agent.py
PORT=8086 uv run python agents/project_manager/agent.py
```

Set URLs in `.env`:
```
STRATEGIST_AGENT_URL=http://localhost:8082
COPYWRITER_AGENT_URL=http://localhost:8083
DESIGNER_AGENT_URL=http://localhost:8084
CRITIC_AGENT_URL=http://localhost:8085
PM_AGENT_URL=http://localhost:8086
```

Then launch the orchestrator:
```bash
uv run adk web agents --allow_origins='*'
```

The Creative Director only connects to agents whose URL is set — partial setups work fine.

## Deploying to Google Cloud

```bash
cd workshop/starter

# 1. Deploy 5 specialists to Cloud Run (writes URLs back to .env)
uv run deploy/deploy_all_specialists.py

# 2. Deploy Creative Director to Vertex AI Agent Engine
uv run deploy/deploy_orchestrator.py --action deploy

# 3. Run a campaign end-to-end
uv run run_campaign.py
```

Teardown:
```bash
bash deploy/teardown_gcp.sh
```

## Environment Setup

```bash
cp .env.example .env
# Fill in: GOOGLE_CLOUD_PROJECT, GOOGLE_CLOUD_PROJECT_NUMBER, CLOUD_RUN_REGION, GCS_IMAGES_BUCKET
gcloud auth application-default login
```

Key env vars:
- `GOOGLE_GENAI_USE_VERTEXAI=1` — routes Gemini calls through Vertex AI
- `GEMINI_MODEL` — text model (default: `gemini-2.5-flash`)
- `GEMINI_IMAGE_MODEL` — image model (default: `gemini-3.1-flash-image`)
- `GCS_IMAGES_BUCKET` — GCS bucket for generated images
- `NOTION_TOKEN` / `NOTION_PROJECT_DATABASE_ID` / `NOTION_TASKS_DATABASE_ID` — optional Notion integration for project manager

## Docker / Cloud Run Notes

- All Dockerfiles use `--platform=linux/amd64` — required when building on Apple Silicon for Cloud Run
- Each agent directory has a `.dockerignore` excluding `.venv` — critical to prevent local arm64 venv from being copied into the image
- CMD uses `/app/.venv/bin/python agent.py` directly (not `uv run`) so the container starts without re-downloading packages

