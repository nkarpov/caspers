# Caspers

Describe a business, get a fully deployable Databricks demo.

Caspers is an AI coding agent skill that builds coherent streaming data, Spark Declarative Pipelines, AI agents, and apps from a natural language description. Say "convenience store" or "craft brewery" and get seed data with PK/FK constraints, a streaming event simulation, and infrastructure-as-code — ready to `databricks bundle deploy`.

## Quick Start

```bash
git clone https://github.com/nkarpov/caspers.git
cd caspers
# Open in your AI coding agent (Claude Code, Cursor, etc.)
# Run: /build-business <your business idea>
```

## How It Works

1. **Describe your business** — the skill asks clarifying questions and builds a Blueprint
2. **Data flows first** — seed tables (managed Delta with PK/FK) + streaming events get generated and deployed
3. **You verify** — sample DBSQL queries let you inspect the data before moving on
4. **Build on top** — choose your next layer: SDP pipeline, AI agent, app, or more data
5. **Stay coherent** — the skill validates cross-layer contracts at every step

The repo starts as a skeleton and becomes your business.

## What Gets Generated

| File | Purpose | Maturity |
|---|---|---|
| `BLUEPRINT.md` | Living business blueprint — tracks all decisions | Core |
| `databricks.yml` | DABs infrastructure declaration | Core |
| `data/seed_generator.py` | Managed Delta tables with PK/FK constraints | Full |
| `data/canonical_generator.py` | State machine event dataset with GPS routing | Full |
| `data/replay.py` | Streaming replay engine | Full |
| `pipelines/transforms.py` | SDP bronze → silver → gold | Planned |
| `agents/agent.py` | MLflow agent definition | Planned |
| `apps/app/` | FastAPI + frontend | Planned |

**Full** = curated patterns, reference code, worked examples. **Planned** = the skill generates these using the blueprint and LLM general knowledge, but no specialized recipes yet.

## Architecture

```
                    ┌→ SDP (Spark Declarative Pipeline)
Data (required) ────┼→ Agent (AI reasoning)
                    ├→ App (UI + database)
                    └→ More data (documents, additional datasets)
```

Data is the prerequisite. Everything else branches from it in any order.
