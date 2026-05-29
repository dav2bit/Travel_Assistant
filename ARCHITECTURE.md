
# Architecture: Armenian B2B Travel AI Support System

> **Stack:** FastAPI · Gemini 2.5 Flash · pgvector · PostgreSQL · SQLAlchemy (async)
> **Pattern:** Clean Architecture + Domain-Driven Design (DDD)
> **Principle:** Inner layers own nothing. Outer layers own everything.

---

## 1. Layered Dependency Rule (The Iron Law)

```
┌─────────────────────────────────────────────────────────┐
│                    infrastructure/                       │
│  FastAPI routes · SQLAlchemy models · Gemini clients     │
│  pgvector queries · PDF processor · session store        │
│                         │                               │
│              depends on ↓                               │
├─────────────────────────────────────────────────────────┤
│                    application/                          │
│  Use cases · RAG orchestrator · Cache pipeline          │
│  Intent resolver · Tour router · FAQ supplement         │
│                         │                               │
│              depends on ↓                               │
├─────────────────────────────────────────────────────────┤
│                      domain/                             │
│  Entities · Value objects · Repository protocols        │
│  Pure Python — zero framework imports                   │
└─────────────────────────────────────────────────────────┘
```

**Absolute rules:**
- `domain/` imports NOTHING outside the Python standard library.
- `application/` imports from `domain/` only. Never from `infrastructure/`.
- `infrastructure/` imports from both `application/` and `domain/`.
- FastAPI route handlers are thin adapters — they translate HTTP to use-case calls and back. No business logic lives in a route.

**Why this matters:** You can swap PostgreSQL for Redis, replace Gemini with OpenAI, or rip out FastAPI entirely — and the domain logic stays untouched.

---

## 2. The Canonical Folder Structure

```
travel_ai/
│
├── domain/                         # Pure Python. Zero framework imports.
│   ├── __init__.py
│   ├── entities/
│   │   ├── __init__.py
│   │   ├── tour.py                 # Tour(id, name, supported_destinations)
│   │   ├── query_intent.py         # QueryIntent(category, tour_ids, unsupported_destination)
│   │   └── answer.py               # Answer(text, source, cache_similarity, rag_chunks_used)
│   ├── value_objects/
│   │   ├── __init__.py
│   │   ├── embedding.py            # Embedding(vector: List[float], dim: int, model: str)
│   │   ├── session_context.py      # SessionContext(session_id, tour_id, last_intent)
│   │   └── answer_source.py        # AnswerSource enum: CACHE | RAG | NO_CONTEXT | REDIRECT
│   └── ports/                      # Abstract interfaces — implemented in infrastructure/
│       ├── __init__.py
│       ├── cache_repository.py     # Protocol: lookup(vec, tour_id, category) -> CacheHit | None
│       ├── rag_repository.py       # Protocol: retrieve(vec, tour_ids, top_k) -> List[str]
│       ├── llm_port.py             # Protocol: generate(prompt) -> str
│       └── embedding_port.py       # Protocol: embed_query(text) -> Embedding
│
├── application/                    # Orchestration only. No DB drivers, no HTTP.
│   ├── __init__.py
│   ├── use_cases/
│   │   ├── __init__.py
│   │   ├── ask_query.py            # AskQueryUseCase — the main pipeline (Step 0–3)
│   │   └── seed_faq.py             # SeedFAQUseCase — admin: embed + store FAQ entry
│   ├── services/
│   │   ├── __init__.py
│   │   ├── intent_service.py       # _detect_intent_gemini + regex fallback
│   │   ├── tour_router.py          # Tour resolution: Gemini → vector → session → text
│   │   ├── cache_pipeline.py       # Two-stage strict cache (Stage 1: local, Stage 2: Gemini)
│   │   ├── rag_pipeline.py         # Retrieve chunks + FAQ supplement + groundedness prompt
│   │   └── hallucination_guard.py  # Unsupported destination detection + redirect logic
│   └── dto/
│       ├── __init__.py
│       ├── ask_request.py          # AskRequest(query, session_id)
│       └── ask_response.py         # AskResponse(answer, source, latency_ms, tour_id)
│
├── infrastructure/                 # All I/O lives here. Replaceable.
│   ├── __init__.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── app.py                  # FastAPI app factory + CORS middleware
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   ├── ask.py              # POST /ask  — thin adapter, calls AskQueryUseCase
│   │   │   ├── faq.py              # POST /faq  — calls SeedFAQUseCase
│   │   │   └── logs.py             # GET  /logs — paginated admin view
│   │   └── middleware/
│   │       └── timing.py           # Request timing middleware
│   ├── db/
│   │   ├── __init__.py
│   │   ├── engine.py               # async_engine + AsyncSessionLocal
│   │   ├── models.py               # SQLAlchemy ORM: FAQTemplate, KnowledgeBase, ChatLog
│   │   ├── repositories/
│   │   │   ├── __init__.py
│   │   │   ├── cache_repo.py       # Implements domain/ports/cache_repository.py
│   │   │   └── rag_repo.py         # Implements domain/ports/rag_repository.py
│   │   └── migrations/             # Alembic migration scripts
│   │       └── versions/
│   ├── llm/
│   │   ├── __init__.py
│   │   ├── gemini_chat.py          # GenerativeModel wrapper (implements llm_port.py)
│   │   ├── gemini_embedder.py      # Gemini embedding API (implements embedding_port.py)
│   │   └── local_embedder.py       # SentenceTransformer wrapper
│   └── session/
│       ├── __init__.py
│       └── in_memory_store.py      # _session_tour + _session_history dicts (swap for Redis)
│
├── tests/
│   ├── unit/
│   │   ├── domain/
│   │   │   └── test_query_intent.py
│   │   └── application/
│   │       ├── test_tour_router.py
│   │       ├── test_cache_pipeline.py
│   │       └── test_hallucination_guard.py
│   └── integration/
│       ├── conftest.py             # Shared fixtures: test DB, mock Gemini
│       ├── test_ask_pipeline.py    # End-to-end: POST /ask with real pgvector
│       └── test_faq_seeding.py
│
├── config.py                       # Pydantic-settings (reads .env) — shared by all layers
├── main.py                         # Entrypoint: imports infrastructure/api/app.py
├── ARCHITECTURE.md                 # This file
├── rules.json                      # Linter/Claude rules for architecture enforcement
├── .env                            # Secret values — never committed
├── .env.example                    # Template for new developers
├── requirements.txt
└── alembic.ini
```

---

## 3. The Query Pipeline (Step-by-Step)

Every `POST /ask` request passes through exactly these stages in order:

```
Request
  │
  ▼
[Step 0] Parallel I/O ──────────────────────────────────────────────────
  ├── local_embed(query)          → local_vec   (384-dim, CPU, free)
  ├── gemini_embed(query)         → gemini_vec  (1536-dim, API)
  └── detect_intent(query, sess)  → (intent, tour_ids, unsupported)
                                     │
  ▼                                  │
[Step 1] Tour Resolution ◄───────────┘
  Priority chain (first non-null wins):
    1. Gemini intent detector  → tour_ids list (supports multi-tour)
    2. Vector detection        → pgvector cosine on faq_templates
    3. Session memory          → _session_tour[session_id]
    4. Text pattern matching   → Armenian Unicode regex
  ─ If unsupported_destination=True: tour_ids = [], skip chain entirely ─
                                     │
  ▼                                  │
[Step 1b] Hallucination Guard ◄──────┘
  IF unsupported_destination:
    → _gemini_unsupported_redirect(query)   ← constrained prompt, no RAG
    → RETURN immediately (skip Steps 2–3)
                                     │
  ▼
[Step 2] Strict Semantic Cache
  SKIP if: is_general OR is_multi_tour OR no query_category
  Stage 1: cosine(local_vec, faq.embedding) ≥ strict_threshold (0.60)
           filtered by: tour_id = ? AND category = ?
  Stage 2: cosine(gemini_vec, faq.gemini_embedding) ≥ strict_threshold
  HIT  → return cached answer immediately
  MISS → continue to Step 3
                                     │
  ▼
[Step 3] RAG Generation
  3a. retrieve(gemini_vec, tour_ids, top_k) from knowledge_base
  3b. if multi_tour OR (general AND no chunks):
        supplement with faq_templates structured data
  3c. if no chunks AND specific category AND not general:
        source = "no_context", skip LLM call
  3d. _gemini_generate(query, chunks, tour_ids)
        ─ Groundedness Rule prepended to EVERY prompt ─
        ─ Tour-scoping note isolates context per tour ─
        → answer
```

---

## 4. The Cache Rules

The semantic cache is **strict-mode only**. There is no fuzzy global cache.

| Condition | Cache behavior |
|-----------|---------------|
| `is_general = True` | **Bypassed** — broad questions need full RAG |
| `is_multi_tour = True` | **Bypassed** — comparison queries need combined context |
| `query_category = None` | **Bypassed** — can't scope without a category |
| `tour_id = None` | **Bypassed** — can't scope without a tour |
| All four known | **Active** — Stage 1 → Stage 2 two-stage check |

**Hard filters applied before cosine similarity:**
```sql
WHERE tour_id = :tour_id AND category = :category AND is_active = TRUE
```
This makes the similarity threshold safe to lower to 0.60 — structural false positives are eliminated by the hard filters before any vector comparison runs.

**Two-stage design:**
- Stage 1 uses the local 384-dim model (zero API calls, sub-millisecond).
- Stage 2 upgrades to 1536-dim Gemini only when Stage 1 produces a candidate hit.
- Both stages must pass for the cache to serve an answer.

---

## 5. The Hallucination Guardrails

Three independent layers prevent LLM hallucination:

### Layer 1 — Intent-time Destination Validation
Gemini's intent detector returns `"unsupported_destination": true` when the query names a country not in the catalog. `tour_ids` is forced to `[]` and the session-memory fallback is skipped entirely.

```json
{ "intent": "price", "target_tours": [], "unsupported_destination": true }
```

### Layer 2 — Text-pattern Fallback
When Gemini's intent API is unavailable (quota exhausted), `_detect_unsupported_destination(query)` applies a regex covering known non-catalog countries (Japan, Germany, Thailand, Dubai, Greece, Turkey, Egypt, China, USA) in both Latin script and Armenian transliteration. Supported tour patterns are checked first — a match there suppresses the unsupported flag.

### Layer 3 — Groundedness Rule in Every RAG Prompt
Every call to `_gemini_generate` prepends:
```
STRICT RULE: You MUST answer using ONLY the facts explicitly stated in the
Context below. You are FORBIDDEN from inventing, estimating, or assuming any
tour details, prices, dates, destinations, or itineraries that are not
present in the Context.
```
This prevents hallucination even when the context is thin or partially relevant.

### Unsupported Destination Prompt (Separate Function)
`_gemini_unsupported_redirect(query)` is a completely separate prompt that:
- Never receives tour data as context
- Explicitly states we don't carry that destination
- Is forbidden from describing or pricing the unsupported tour
- Warmly redirects to France, Italy, Spain

The main `_gemini_generate` is **never called** with empty `tour_ids` + empty context.

---

## 6. Session Memory Design

Session state is currently stored in two Python dicts (in-process):

| Store | Key | Value | Purpose |
|-------|-----|-------|---------|
| `_session_tour` | `session_id` | `tour_id` | Remember the last confirmed tour per session |
| `_session_history` | `session_id` | `[{query, intent, tour}]` (max 3) | Feed to Gemini intent detector for elliptical follow-ups |

**Swap path to Redis:** Both stores implement the same simple `get/set` interface. Replacing them with an async Redis client requires touching only `infrastructure/session/` — no application logic changes.

**Elliptical query support:** Armenian users often ask follow-ups by tour name alone ("իcк Italiаyin"). The session history (last 3 turns) is passed to Gemini's intent model, which inherits the previous intent category and applies it to the newly named tour.

---

## 7. Embedding Strategy

| Layer | Model | Dimensions | Purpose |
|-------|-------|-----------|---------|
| Local (CPU) | `paraphrase-multilingual-MiniLM-L12-v2` | 384 | Cache Stage 1 — zero API cost, <1ms |
| Gemini API | `gemini-embedding-001` | 1536 | Cache Stage 2 + RAG retrieval |

FAQ entries carry **both** embeddings. Knowledge base chunks carry only the Gemini embedding. This is intentional: RAG retrieval always needs the higher-quality semantic match; the cache's fast pre-filter only needs good-enough local similarity.

---

## 8. What Lives Where — Quick Reference

| Component | Current file | Target location |
|-----------|-------------|-----------------|
| `Settings` | `config.py` | `config.py` (shared, keep at root) |
| ORM models | `database.py` | `infrastructure/db/models.py` |
| `AsyncSession` factory | `database.py` | `infrastructure/db/engine.py` |
| `LocalEmbedder` | `embeddings.py` | `infrastructure/llm/local_embedder.py` |
| `GeminiEmbedder` | `embeddings.py` | `infrastructure/llm/gemini_embedder.py` |
| `SYSTEM_INSTRUCTION` | `main.py` | `infrastructure/llm/gemini_chat.py` |
| `_INTENT_SYSTEM_PROMPT` | `main.py` | `infrastructure/llm/gemini_chat.py` |
| `_detect_intent_gemini` | `main.py` | `application/services/intent_service.py` |
| `_detect_tour_from_text` + regex tour patterns | `main.py` | `application/services/tour_router.py` |
| `_detect_unsupported_destination` | `main.py` | `application/services/hallucination_guard.py` |
| `_cache_lookup_strict` | `main.py` | `application/services/cache_pipeline.py` |
| `_rag_retrieve` | `main.py` | `infrastructure/db/repositories/rag_repo.py` |
| `_faq_context_for_tours` | `main.py` | `infrastructure/db/repositories/rag_repo.py` |
| `_gemini_generate` | `main.py` | `application/services/rag_pipeline.py` |
| `_gemini_unsupported_redirect` | `main.py` | `application/services/hallucination_guard.py` |
| `/ask` route body | `main.py` | `application/use_cases/ask_query.py` + `infrastructure/api/routes/ask.py` |
| `_session_tour`, `_session_history` | `main.py` | `infrastructure/session/in_memory_store.py` |
