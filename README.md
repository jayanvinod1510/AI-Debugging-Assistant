## AI Debugging Assistant

Tool for Debugging using AI

# AI Debugging Assistant — Architecture

## Overview

The **AI Debugging Assistant** is a Frappe-based intelligent incident investigation system that automatically collects evidence from multiple sources, reasons about the probable cause using an LLM, and proposes remediation patches as GitLab merge requests — all with safety gates and mandatory human review before any code is merged.

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Framework | Frappe (Python) |
| Database | MySQL/MariaDB (Frappe ORM) |
| Vector Store | Qdrant |
| LLM Providers | Ollama, OpenAI, Mock |
| Evidence Sources | Elasticsearch, Prometheus, GitLab, Frappe Error Log, Frappe DB |
| Embedding Models | Ollama (`qwen3-embedding`), OpenAI (`text-embedding-3-small`) |

---

## Directory Structure

```
ai_debugging_assistant/
├── config/
│   └── settings.py              # AIDebugRuntimeSettings — validation & defaults
├── ai_debugging_assistant/
│   ├── api/                     # Frappe-whitelisted REST endpoints
│   │   ├── incident.py          # Incident CRUD + human review
│   │   ├── query.py             # Query assistant
│   │   ├── ingestion.py         # Trigger ingestion jobs
│   │   ├── overview.py          # System overview
│   │   ├── sources.py           # Data source management
│   │   └── settings.py          # Settings API
│   ├── connectors/              # Multi-system evidence collectors
│   │   ├── error_log_connector.py
│   │   ├── elasticsearch_connector.py
│   │   ├── prometheus_connector.py
│   │   ├── gitlab_connector.py
│   │   ├── db_state_connector.py
│   │   ├── alert_connector.py
│   │   └── normalizer.py        # All evidence → common schema
│   ├── ingestion/               # Offline data ingestion pipeline
│   │   ├── pipeline.py          # Orchestrator
│   │   ├── normalizer.py
│   │   ├── chunker.py           # Character-based chunking with overlap
│   │   ├── embedder.py          # Ollama / OpenAI / Mock
│   │   └── indexer.py           # Qdrant upsert
│   ├── stores/
│   │   └── vector_store.py      # Qdrant client wrapper
│   ├── retrieval/
│   │   ├── search.py            # Embed query → cosine similarity search
│   │   └── evidence_builder.py  # Search results → reasoning-ready format
│   ├── reasoning/
│   │   ├── pipeline.py          # Reasoning orchestrator (standard + tool-calling)
│   │   ├── prompts.py           # Prompt construction
│   │   ├── tool_registry.py     # Tool schemas for tool-calling mode
│   │   ├── tool_prompts.py      # Tool-calling prompts
│   │   └── tool_executor.py     # Execute tool calls
│   ├── investigation/
│   │   ├── pipeline.py          # Full incident investigation flow
│   │   ├── collector.py         # Orchestrate all connectors
│   │   ├── evidence_pack_builder.py  # Merge live + historical evidence
│   │   └── safety_gate.py       # Pre-remediation safety checks
│   ├── remediation/
│   │   ├── planner.py           # Build remediation plan
│   │   ├── policy.py            # Execution policy (repo/branch/file scope)
│   │   ├── patch_builder.py     # Generate patch content
│   │   ├── content_generator.py # LLM-generated code fixes
│   │   └── gitlab_mr_service.py # Create GitLab merge requests
│   ├── services/
│   │   ├── incident_service.py  # Incident lifecycle
│   │   ├── query_service.py     # Query execution
│   │   └── ingestion_service.py # Ingestion job management
│   ├── doctype/                 # Frappe ORM models (see Data Models below)
│   ├── integrations/
│   │   └── ollama_client.py     # Ollama generate + embed API wrapper
│   ├── telemetry/               # Metrics collection
│   └── jobs/                    # Background job placeholders
└── www/                         # Web pages (minimal)
```

---

## Data Flow

### 1. Ingestion Pipeline (offline preparation)

```
Frappe Error Logs
    → fetch_recent_error_logs()
    → normalize_error_log_record()   # common schema
    → chunk_error_document()         # text splitting with overlap
    → embed_texts()                  # Ollama / OpenAI → vectors
    → upsert_chunks()                # store in Qdrant
    → Index AI Debug Document/Chunk docs in Frappe DB
```

### 2. Query Pipeline

```
User query (POST /api/query)
    → Create AI Debug Query doc
    → embed query → Qdrant similarity search
    → evidence_builder: results → reasoning format
    → run_reasoning() → LLM → { summary, probable_causes, next_steps, confidence_score }
    → Save AI Debug Response doc
```

### 3. Incident Investigation Pipeline

```
Incident trigger (API / AI Incident Rule)
    → Create AI Incident Run doc  [status: Triggered]
    │
    ├─ collect_live_evidence()     [status: Collecting Evidence]
    │   ├── Elasticsearch (logs)
    │   ├── Prometheus (metrics)
    │   ├── GitLab (recent commits)
    │   ├── Frappe DB (runtime state)
    │   └── Alert systems
    │
    ├─ build_investigation_evidence_pack()  [status: Investigating]
    │   ├── Live evidence from collectors
    │   └── Historical evidence from Qdrant
    │
    ├─ run_reasoning()  →  { summary, probable_causes, confidence_score }
    │
    ├─ evaluate_safety_gate()
    │   ├── confidence ≥ 0.6
    │   ├── evidence ≥ 5 items
    │   ├── has logs OR metrics
    │   ├── service identified
    │   ├── severity ≠ Critical
    │   └── remediation allowed by rule
    │
    ├─ [if PASS] build_remediation_plan()
    │       → evaluate_remediation_execution_policy()
    │           ├── repo whitelisted
    │           ├── branch in {main, master, develop, saas-master}
    │           ├── target files in {config/, services/, settings/, hooks/, utils/}
    │           └── plan type allowed
    │       → build_patch_request()
    │       → create_merge_request_from_patch()  [GitLab MR]
    │
    └─ Update AI Incident Run  [status: Awaiting Review]
```

### 4. Human Review

```
Reviewer calls update_incident_review()
    → Approved  → Incident closed; MR can be merged
    → Rejected  → Incident escalated or closed without merge
```

---

## Reasoning Modes

| Mode | Provider | Description |
|------|----------|-------------|
| Standard | Ollama / OpenAI / Mock | Single LLM call → structured JSON response |
| Tool-Calling | Ollama only | Model iteratively calls tools to fetch more evidence; loops until final answer or `max_tool_calls` |

**Reasoning output schema:**
```json
{
  "summary": "...",
  "probable_causes": [{ "cause": "...", "confidence": 0.7, "reasoning": "..." }],
  "next_steps": ["..."],
  "limitations": "...",
  "confidence_score": 0.75,
  "answer_quality": "High"
}
```

---

## Evidence Schema (normalized)

All connectors emit a common structure:

| Field | Description |
|-------|-------------|
| `evidence_type` | logs / metrics / changes / state / alerts |
| `source_name` | connector name |
| `title` | short label |
| `summary` | human-readable summary |
| `raw_content` | full raw text |
| `event_timestamp` | ISO timestamp |
| `severity` | low / medium / high / critical |
| `site_name` | Frappe site |
| `environment` | production / staging / etc. |
| `service_name` | affected service |
| `metadata` | connector-specific extras |

---

## Data Models (Doctypes)

| Doctype | Purpose |
|---------|---------|
| `AI Debug Settings` | Global config singleton (LLM, embedding, Qdrant, policies) |
| `AI Debug Source` | Data source definitions |
| `AI Debug Query` | User questions; tracks status + timing |
| `AI Debug Response` | LLM answer with evidence, causes, confidence |
| `AI Debug Document` | Ingested document (error log, metric snapshot) |
| `AI Debug Chunk` | Vector-indexed text fragment; linked to parent document |
| `AI Debug Ingestion Job` | Ingestion run tracking |
| `AI Debug Feedback` | User feedback on response quality |
| `AI Incident Run` | Full incident execution record; stores evidence + reasoning + MR result |
| `AI Incident Rule` | Rules controlling auto-trigger and remediation permissions |

---

## API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST query_assistant` | Run a query; returns reasoning result |
| `GET list_queries` | List recent queries |
| `POST create_and_run_incident` | Create + run incident investigation |
| `POST run_existing_incident` | Re-run an existing incident |
| `GET list_incident_runs` | List incidents with summary |
| `GET get_incident_run` | Full incident details |
| `POST update_incident_review` | Human approval / rejection |

---

## Configuration (AI Debug Settings)

| Setting | Description |
|---------|-------------|
| `enabled` | Master on/off |
| `llm_provider` | `Ollama` / `OpenAI` / `Mock` |
| `chat_model` | e.g. `qwen3:8b` |
| `embedding_provider` | `Ollama` / `OpenAI` / `Mock` |
| `embedding_model` | e.g. `qwen3-embedding:0.6b` |
| `reasoning_provider` | `Ollama` / `OpenAI` / `Mock` |
| `qdrant_url` | Vector store URL |
| `qdrant_collection_name` | Qdrant collection |
| `chunk_size`, `chunk_overlap` | Ingestion parameters |
| `default_top_k`, `max_context_chunks` | Retrieval limits |
| `similarity_threshold` | Minimum relevance score |
| `enable_tool_calling`, `max_tool_calls` | Tool-calling mode (Ollama only) |
| `telemetry_enabled` | Logging of prompts + outputs |

---

## Telemetry

Stored per-investigation:
- `retrieval_ms`, `generation_ms`, `total_ms`
- `confidence_score`, `answer_quality`
- `retrieved_chunk_count`, `unique_document_count`
- Optional: `raw_prompt_json`, `raw_model_output_json` (if `store_raw_*` flags enabled)

---

## Safety Design

The system has **three gates** before any code change reaches production:

1. **Safety Gate** — confidence, evidence quantity, severity, scope checks before planning remediation
2. **Execution Policy** — repo whitelist, branch allowlist, file-path scope before creating a GitLab MR
3. **Human Review** — a reviewer must explicitly approve before any MR is merged

The system proposes; humans decide.


#### License

mit