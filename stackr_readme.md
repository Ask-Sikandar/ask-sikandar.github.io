# Memox AI Sales Assistant — Pacific Container Co. POC

A RAG-based AI sales assistant that answers prospect questions about shipping containers using company documentation. Built with Django + DRF, Next.js, WebSockets, pgvector, and Ollama.

## Architecture

```
Browser ──WebSocket──▶ Django Channels (Daphne ASGI)
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
              AgentHandler         REST API (DRF)
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
   IntentClassifier Retriever  LLMClient
   (keywords)   (pgvector)  (Ollama stream)
                    │
              EmbeddingService
             (sentence-transformers)
```

All AI services implement abstract interfaces (`IEmbedder`, `IRetriever`, etc.) — concrete classes are injected via `settings.SERVICE_CLASSES`. Tests swap in mocks with zero infrastructure.

## Quick Start (Docker)

```bash
# 1. Clone and start all services
docker-compose up --build

# 2. The backend auto-runs migrations on startup.
# 3. Ingest sample documents (one-time):
docker-compose exec backend uv run python manage.py shell -c "
from pathlib import Path
from apps.documents.models import Document
from services.ingestion import ingest_document

docs_dir = Path('sample_docs')
for f in docs_dir.glob('*.md'):
    doc = Document.objects.create(title=f.stem.replace('_', ' ').title(), content=f.read_text(), source_type='markdown')
    ingest_document(doc.id)
    print(f'Ingested {doc.title} ({doc.chunks.count()} chunks)')
"

# 4. Open http://localhost:3000 — chat widget ready
# 5. Admin panel: http://localhost:3000/admin/documents
# 6. Lead dashboard: http://localhost:3000/admin/leads
```

> **Note:** Ollama pulls `llama3.2:3b` (~2GB) on first start. This takes a few minutes.

## Local Development (without Docker)

### Prerequisites
- Python 3.12+ with uv
- PostgreSQL 16 with pgvector extension
- Redis 7
- Ollama running locally (`ollama pull llama3.2:3b`)
- Node.js 20+

### Backend

```bash
cp .env.example .env
uv sync --extra dev
createdb memox
uv run python manage.py migrate
uv run daphne -b 0.0.0.0 -p 8000 config.asgi:application
```

### Frontend

```bash
cd frontend
npm install
NEXT_PUBLIC_API_URL=http://localhost:8000 NEXT_PUBLIC_WS_URL=ws://localhost:8000 npm run dev
```

## Running Tests

```bash
# Unit tests (no DB required)
uv run pytest tests/ -v

# All tests including integration (PostgreSQL required for DB tests)
uv run pytest tests/ -v
# DB tests auto-skip if PostgreSQL is not reachable

# With coverage
uv run pytest tests/ --cov=. --cov-report=term-missing
```

## Eval Script

```bash
# With real DB + ingested documents
uv run python eval/eval_quality.py --verbose

# Mock mode (no DB required)
uv run python eval/eval_quality.py --mock
```

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET/POST | `/api/documents/` | List / upload documents |
| POST | `/api/documents/<id>/ingest/` | Trigger chunking + embedding |
| GET | `/api/documents/<id>/chunks/` | List chunks |
| POST | `/api/ai/chat/` | Single-turn chat (REST) |
| GET | `/api/ai/chat/history/<lead_id>/` | Conversation history |
| GET | `/api/leads/` | Lead list (dashboard) |
| WS | `/ws/chat/<lead_id>/` | Bidirectional chat with streaming |

### Chat request/response

```json
POST /api/ai/chat/
{ "lead_id": "uuid", "message": "How much is a 40ft container?" }

{
  "message_id": "uuid",
  "content": "A 40ft standard container is $3,850 [Source: Pricing Sheet]...",
  "intent": "pricing",
  "sources": [{"title": "Pricing Sheet", "excerpt": "...", "score": 0.94}],
  "components": [{"type": "product_comparison", "data": {"products": [...]}}],
  "lead_score": 10,
  "handoff_triggered": false
}
```

## Key Design Decisions

See [DECISIONS.md](DECISIONS.md) for full reasoning.

- **pgvector** — same Postgres instance, one less service, atomic transactions
- **Paragraph-first chunking** — preserves semantic units; tables kept atomic
- **Keyword intent classification** — sub-ms, fully testable, accurate for 4 categories
- **SOLID service layer** — ABCs + `SERVICE_CLASSES` registry; tests inject mocks
- **Lead scoring dashboard** — added beyond spec: conversion=25pts, pricing=10pts

## Answer Quality Evaluation

A golden-set harness (`eval/eval_quality.py`) runs 10 questions against the
live pipeline (real retrieval + real LLM, not mocked), scoring each answer by
embedding cosine similarity against expected reference content, alongside
intent-classification accuracy.

```bash
docker-compose exec backend uv run python eval/eval_quality.py --verbose
```

| Metric | Result |
|---|---|
| Avg. semantic similarity | _fill in after running_ |
| Pass / Warn / Fail (of 10) | _fill in after running_ |
| Intent classification accuracy | _fill in after running_ |
| Avg. response latency | _fill in after running_ |

_Numbers below the fold — run the command above against the live stack and drop the output here._