# Project Overview: AI-Powered Travel Assistant — Arshine

**Version:** 2.0 | **Status:** Active Development | **Confidentiality:** Internal

---

## Table of Contents

1. [Project Objective](#1-project-objective)
2. [Core Features](#2-core-features)
3. [How It Works — Technical Stack](#3-how-it-works--technical-stack)
4. [System Architecture](#4-system-architecture)
5. [Future Roadmap](#5-future-roadmap)

---

## 1. Project Objective

### The "Why"

The travel industry is highly dependent on human consultants answering repetitive,
time-sensitive questions from potential clients — questions about tour prices, visa
requirements, itineraries, and availability. This dependency creates three measurable
business problems:

| Problem | Business Impact |
|---|---|
| Managers handle the same questions repeatedly | Lost time, reduced capacity for high-value sales |
| Inquiries outside business hours go unanswered | Lost leads, reduced conversion rates |
| Response quality is inconsistent across staff | Uneven client experience, potential liability |

**Arshine** is an AI-powered automated travel assistant purpose-built to solve these
problems. It operates 24 hours a day, 7 days a week, responds instantly, and draws its
answers exclusively from the company's own verified tour documentation — ensuring
accuracy, consistency, and brand alignment in every interaction.

The core business goals are:

- **Reduce manager workload** by automating responses to Tier-1 inquiries (price,
  duration, visa, inclusions, schedule), freeing the team to focus on closing deals.
- **Increase conversion rates** by providing potential clients with precise, factual
  answers at the moment of interest, rather than asking them to wait.
- **Protect the company** by strictly refusing to answer off-topic questions, preventing
  the system from becoming a liability or a general-purpose chatbot.
- **Scale support capacity** without proportionally scaling headcount.

---

## 2. Core Features

### 2.1 Tour-Specific Knowledge Answering

The assistant answers client questions by reading directly from the company's own uploaded
PDF documents (tour programs, pricing sheets, visa guidelines). It does not rely on
general internet knowledge. Every answer is grounded in the company's proprietary data,
which means:

- Prices, itineraries, and conditions stated by the assistant are always sourced from the
  company's official materials.
- The assistant covers all key inquiry categories: **price**, **duration**, **visa
  requirements**, **cancellation policy**, **child discounts**, **departure schedule**,
  and **tour inclusions**.
- Multi-tour comparisons are supported — a client can ask "Compare France and Italy" and
  receive a structured, accurate side-by-side answer.

### 2.2 Strict Topic Filtering

The assistant is explicitly prohibited from engaging in conversations unrelated to the
company's tour catalog. If a client asks about a destination the company does not offer
(e.g., Japan, Germany, Thailand), the assistant:

1. Warmly acknowledges the inquiry.
2. Clearly states that no active tours are available for that destination.
3. Proactively redirects the client to the available catalog (France, Italy, Spain).

This boundary is enforced at three independent layers — LLM classification, text pattern
matching, and session state validation — making it highly reliable.

### 2.3 Smart Greeting System

The assistant is aware of conversation context. It greets the client warmly on their
**first message only**, and proceeds directly to answering on all subsequent messages.
This mirrors natural human conversation and avoids the robotic, repetitive greetings
common in lower-quality chatbot implementations.

### 2.4 Graceful Fallback and Redirect Handling

When the requested information is not available in the knowledge base, the assistant does
not fabricate an answer or produce a generic error. Instead, it:

- Acknowledges the limitation honestly.
- Suggests the closest relevant alternative from the available tour catalog.
- Invites the client to continue the conversation.

This ensures that every interaction — even those that cannot be fully resolved — ends
with a positive, forward-moving client experience rather than a dead end.

### 2.5 Personality and Tone

The assistant speaks fluent Eastern Armenian and is designed to feel like a warm,
knowledgeable travel consultant rather than a rigid automated system. It handles
humorous or sarcastic questions with appropriate wit, maintains a professional yet
friendly register, and adapts its response length to the nature of the question — brief
for direct factual queries, comprehensive for broad exploratory ones.

---

## 3. How It Works — Technical Stack

### 3.1 Core Technologies

| Component | Technology | Purpose |
|---|---|---|
| Web Framework | **Python 3.12 + FastAPI** | High-performance async API server |
| Primary LLM | **Google Gemini 2.5 Flash Lite** | Natural language generation and intent classification |
| Embedding (Remote) | **Google Gemini Embedding-001** | High-accuracy semantic vector search (1,536 dimensions) |
| Embedding (Local) | **Sentence Transformers (MiniLM)** | Fast, zero-cost local pre-filtering (384 dimensions) |
| Vector Database | **PostgreSQL + pgvector** | Stores and queries document embeddings |
| ORM | **SQLAlchemy (async)** | Database access layer |
| Configuration | **Pydantic Settings** | Environment-based configuration management |

### 3.2 The Request Pipeline

When a client sends a message, the following sequence executes automatically in under
two seconds:

```
Client Message
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 0 — Parallel Processing (fired simultaneously)    │
│  • Local embedding of the query (384-dim, free, <1ms)   │
│  • Remote embedding via Gemini API (1,536-dim)          │
│  • Intent classification via Gemini LLM (JSON output)   │
└─────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 1 — Tour & Intent Resolution                      │
│  Determines which tour the client is asking about       │
│  using a 4-stage priority chain:                        │
│  Gemini intent → Vector similarity → Session memory     │
│  → Text pattern matching                                │
│  Unsupported destinations are caught here and           │
│  redirected immediately.                                │
└─────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 2 — Semantic Cache Lookup (two-stage)             │
│  Stage 1: Fast local cosine similarity check            │
│  Stage 2: High-accuracy Gemini embedding check          │
│  Hard filter: tour_id + category must match exactly.    │
│  Cache hit → instant answer, zero LLM API cost.         │
└─────────────────────────────────────────────────────────┘
      │ (cache miss)
      ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 3 — RAG Generation                               │
│  Retrieves the most relevant chunks from the            │
│  knowledge base (PDF documents), supplements with       │
│  structured FAQ data, then calls Gemini to generate     │
│  a grounded Armenian-language answer.                   │
│  Groundedness rule enforced: model is forbidden from    │
│  inventing facts not present in the retrieved context.  │
└─────────────────────────────────────────────────────────┘
      │
      ▼
   Response returned to client + interaction logged
```

### 3.3 API Quota Management

The system is engineered to operate comfortably within Google's free-tier API limits:

| Model | Purpose | Free Daily Limit |
|---|---|---|
| Gemini 2.5 Flash Lite | Intent classification + Answer generation | 1,500 req/day |
| Gemini Embedding-001 | Semantic vector search | 1,500 req/day |
| Local MiniLM | Cache pre-filter | Unlimited (on-device) |

A built-in **Quota Monitor** dashboard (accessible at `/`) displays real-time usage
progress bars that turn yellow at 80% and red at 90% capacity, with automatic daily
reset at UTC midnight.

---

## 4. System Architecture

### 4.1 Clean Architecture — Domain-Driven Design

The project is structured according to **Clean Architecture** principles, separating
concerns into three concentric layers with a strict inward-only dependency rule:

```
┌─────────────────────────────────────────────────┐
│              INFRASTRUCTURE LAYER               │
│  (Gemini API, PostgreSQL, pgvector, FastAPI)    │
│                                                 │
│   ┌─────────────────────────────────────────┐  │
│   │         APPLICATION LAYER               │  │
│   │  (AskQuestionUseCase — pipeline logic)  │  │
│   │                                         │  │
│   │   ┌───────────────────────────────┐    │  │
│   │   │        DOMAIN LAYER           │    │  │
│   │   │  (Entities, Protocols,        │    │  │
│   │   │   Business Rules)             │    │  │
│   │   └───────────────────────────────┘    │  │
│   └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
         Dependencies point inward only →
```

| Layer | Responsibility | Examples |
|---|---|---|
| **Domain** | Core business entities and contracts | `Tour`, `IntentResult`, `Answer`, Protocol interfaces |
| **Application** | Orchestration of the pipeline | `AskQuestionUseCase`, prompt constants, helper functions |
| **Infrastructure** | All external I/O | Gemini API adapters, PostgreSQL repositories, session store |

### 4.2 Engineering Standards

This architecture was chosen deliberately to meet senior engineering standards:

- **Testability:** Every component can be tested in isolation using lightweight in-memory
  stubs. No test requires a live database or API connection.
- **Replaceability:** Swapping the LLM provider (e.g., from Gemini to OpenAI) requires
  changing exactly one file — the infrastructure adapter — with zero changes to business
  logic.
- **Scalability:** The stateless application layer and async I/O model allow the system
  to handle concurrent requests efficiently. Session state is isolated behind a protocol,
  ready to be backed by Redis for multi-instance deployments.
- **Maintainability:** Business rules (prompts, thresholds, tour catalog) are defined as
  named constants in the application layer, not scattered across the codebase.
- **Safety:** Three independent hallucination-prevention layers ensure the model cannot
  invent tours, prices, or itineraries that are not present in the company's documents.

---

## 5. Future Roadmap

The following enhancements are planned for subsequent development phases:

### Phase 1 — Analytics & Observability
- Persistent analytics database for long-term interaction logging and trend analysis.
- Management dashboard showing top questions, cache hit rates, unanswered query patterns,
  and daily/monthly usage statistics.
- Automated reporting: weekly summary emails to management with key KPIs.

### Phase 2 — Operational Intelligence
- Visual quota dashboards with historical charts and cost-projection estimates.
- Automated alerts when API usage approaches daily limits.
- A/B testing framework to compare prompt variations and measure response quality.

### Phase 3 — Platform Integration
- **Telegram Bot:** Direct integration allowing clients to interact with Arshine via the
  Telegram messaging platform, meeting clients where they already communicate.
- **WhatsApp Business API:** Integration with Meta's WhatsApp Business platform for
  broader reach, particularly relevant for the Armenian and regional travel market.
- **Website Widget:** Embeddable chat widget for direct deployment on the company's
  existing website, requiring no client-side installation.

### Phase 4 — Knowledge Expansion
- Self-service document upload portal for managers to add new tours to the knowledge
  base without developer involvement.
- Support for additional languages (Russian, English) to serve international clients.
- Automated FAQ generation from uploaded documents to pre-populate the semantic cache.

---

## Summary

Arshine represents a production-grade, AI-powered customer engagement system built to
the standards expected of enterprise software. It is not a prototype or a proof of
concept — it is a deployable product with robust error handling, architectural integrity,
and a clear path for future growth.

The investment in Clean Architecture and senior engineering practices ensures that the
system can be maintained, extended, and scaled by any competent engineering team without
requiring a rewrite. The business logic is protected behind well-defined contracts,
the infrastructure is swappable, and every design decision is documented.

**This system is ready to represent the company to its clients.**

---

*Document prepared by the Engineering Team. For technical questions, contact the project lead.*
