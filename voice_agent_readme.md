# PHI Collection Voice Agent — Patient Registration over the Phone

A phone-reachable voice AI agent that conversationally registers patients, validates their data as it is spoken, persists it, and exposes the same records through a REST API.

Built with Vapi (telephony + STT/TTS + turn-taking), an LLM for conversation and tool calling, FastAPI, SQLAlchemy, and Pydantic. Dockerised and deployed.

> **Note:** the live phone number, deployment URL, and all credentials are intentionally omitted here. Everything sensitive is environment-configured — see [Security & Data Handling](#security--data-handling).

## Architecture

```
Caller (phone) ──▶ Vapi (STT + TTS + turn-taking + LLM)
                      │
                      │  tool calls: find_patient_by_phone /
                      │  create_patient / update_patient
                      │  HTTPS webhook, shared-secret authenticated
                      ▼
              FastAPI backend  ──▶  Service layer  ──▶  Database
                      ▲                  ▲
                      │                  │
              REST API /patients   (same functions, same validation)
```

The single most important property of this design: **the voice agent and the public REST API are two entry points into the same service functions.** A field collected over a phone call travels the exact same validation and persistence path as one submitted by `curl`. There is one set of rules, so voice and API can never drift apart.

## Why This Was Non-Trivial

Voice input is uniquely hostile to structured data collection. Callers self-correct mid-sentence, give dates in ambiguous formats, spell things phonetically, and volunteer fields out of order. A form has a submit button; a phone call does not. The engineering problem is turning an unbounded, error-prone audio stream into a validated database row without the caller ever feeling like they are filling in a form.

## Conversation Design

The system prompt is version-controlled and pushed to the platform by a script, so the assistant's behaviour is reproducible rather than dashboard-configured. It encodes a deliberate flow:

| Step | Design reasoning |
|------|------------------|
| Collect phone number **first**, then look it up before anything else | Detects returning callers up front, so the flow branches into an update rather than creating a duplicate record after five minutes of wasted questions |
| Collect required fields a couple at a time | Long field lists are unusable in audio; small batches keep turn-taking natural |
| Offer optional fields (insurance, emergency contact, language) exactly once | Prevents nagging and keeps average call length down |
| Mandatory read-back with explicit confirmation before saving | The only reliable defence against transcription errors on names, dates, and addresses |
| Field-level re-prompting on validation failure | When the API returns which field is bad, the agent re-asks that one field instead of restarting — the error response shape was designed specifically to make this possible |
| "Start over" escape hatch | Callers get stuck; they need an exit that isn't hanging up |

Three tools are exposed to the model — `find_patient_by_phone`, `create_patient`, `update_patient` — each defined as an explicit JSON schema and dispatched onto a single service function. The tool schemas are the contract between the model and the backend.

## Security & Data Handling

Health data raises the bar, and the system was built accordingly:

- **Authenticated webhook.** Every tool-call request must carry a shared secret header. The endpoint is public by necessity; it is not open.
- **No hardcoded credentials anywhere in source.** All keys, secrets, and connection strings load from environment variables through typed settings (`pydantic-settings`), sourced from a local `.env` in development and the platform's variables panel in production. The repository ships an `.env.example` with names only.
- **Soft deletes.** Records are never physically removed — a deletion timestamp is set and the row is excluded from reads. Patient history is not destructible through the API.
- **Errors don't leak internals.** Unhandled exceptions are logged server-side with a full traceback; the caller receives a generic error code.
- **Structured audit logging.** Each completed registration or update emits a single structured log line, giving a reviewable trail of what was collected on every call.

Acknowledged and documented as a limitation rather than glossed over: this build does **not** encrypt PII at rest, and was operated with synthetic data only. Real PHI would require encryption or tokenisation at rest, a compliant hosting posture, and access controls beyond a shared secret.

## API Surface

All responses use a consistent envelope — `{"data": ..., "error": null}`, or `{"data": null, "error": {"code", "message", "fields"}}` on failure. The `fields` array is what makes precise voice re-prompting possible.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/patients` | List with `last_name`, `date_of_birth`, `phone_number` filters + pagination |
| `GET` | `/patients/:id` | Fetch one |
| `POST` | `/patients` | Create (`201`) |
| `PUT` | `/patients/:id` | Partial update |
| `DELETE` | `/patients/:id` | Soft delete |
| `GET` | `/health` | Healthcheck |
| `POST` | `/vapi/webhook` | Tool-call webhook (authenticated) |

Status codes: `200`/`201` success, `404` missing or soft-deleted, `422` field-level validation failure naming each bad field, `500` unexpected error.

## Testing

28 pytest cases, covering:

- CRUD happy paths across the REST surface
- Per-field validation failures returning `422` with the offending field named
- `404` behaviour on missing and soft-deleted records
- Soft-delete semantics and query filters
- Webhook authentication, tool dispatch, and duplicate-caller detection

FastAPI's dependency injection makes swapping the database session in tests trivial, so the suite runs with no external infrastructure.

## Observability

- ASGI middleware logs every request: method, path, status, duration.
- Completed voice registrations and updates log their final collected payload as a structured event.
- Production logs are emitted as single-line JSON for aggregator parsing; development logs stay human-readable plain text.

## Key Trade-offs

Decisions made consciously, with the reasoning recorded:

- **SQLite over Postgres** — zero-setup, file-based persistence on a mounted volume, appropriate for the scope. Its single-writer concurrency limit would matter under real multi-instance load; Postgres is the production answer.
- **`create_all()` at startup over Alembic** — fine for a single-table, additive schema; real migrations would be required before renaming or dropping columns in production.
- **Corrections handled in the prompt, not in backend state** — the backend only ever sees the final confirmed payload, keeping it fully stateless. The cost is that mid-call correction quality is delegated to the system prompt rather than enforced in code.
- **`update_patient` requires an ID from a prior lookup** — there is no voice-driven "update by spelling your name" fallback if the caller's phone number itself has changed.
- **Container runs as root** — the platform mounts persistent volumes root-owned, which conflicted with a non-root user. Dropped the non-root user rather than adding a boot-time `chown` workaround, given the time budget; would revisit for a real deployment.

## What I'd Build Next

- Appointment scheduling immediately after registration
- Multi-language switching mid-call
- Call transcript storage linked to the patient record
- A read-only dashboard over the patient list
- Rate limiting and abuse protection on the public webhook beyond the shared secret
- Postgres + migrations, encryption at rest, and a compliance review before any real patient data

## Skills Demonstrated

Voice AI and telephony integration · LLM function/tool calling · prompt engineering as a versioned artefact · FastAPI and REST API design · Pydantic validation and schema design · SQLAlchemy data modelling · webhook authentication · secrets management · structured logging and observability · Docker · cloud deployment with persistent storage · pytest with dependency-injected test doubles · documenting trade-offs honestly.
