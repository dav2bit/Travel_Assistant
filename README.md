# ✈️ Arshine – Production-Grade AI Travel Assistant & Guided RAG Pipeline

⚠️ **PREFACE: CODEBASE AVAILABILITY & CONFIDENTIALITY**
The core codebase, custom prompt tuning assets (`prompts.py`), and operational data-ingestion pipelines are maintained within a **private repository** to safeguard proprietary API configurations, brand-specific prompt engineering architectures, and sensitive business intelligence. This public repository serves as an architectural index, mapping out the comprehensive System Design, data velocity, and underlying technical patterns of the platform.

---

## 🔄 End-to-End System Pipeline

Every inbound user inquiry undergoes a strict, asynchronous multi-stage lifecycle optimized for execution in under 2 seconds:

### Step 1: Parallel I/O & Granular Intent Classification (Step 0)
Upon receiving a raw query, the system triggers parallel operations: it provisions vectorized representations while executing a strict classification pass via the LLM routing layer (Gemini 2.5 Flash / Gemma 3 / Groq). Leveraging structured schema constraints defined inside `prompts.py`, the engine maps queries into deterministic intent signatures:

* **`booking` (Conversion Trigger):** Intercepts intent to secure a spot or initialize checkout (e.g., *"I want to book the Paris tour"*).
* **`price` / `duration` / `visa`:** Maps specific operational, data-bound constraints regarding the tour layout.
* **`joke` (Persona & Engagement Router):** Captures playful engagement, sarcasm, or non-sequiturs (e.g., *"Do you have a tour to the moon?"* or *"Will you marry me?"*). The system bypasses data extraction, responds with a tailored blend of wit inspired by Gyumri hospitality, and executes a hard structural pivot back to a legitimate call to action (CTA).
* **`hack` (Security Layer / Prompt Guard):** Flags exploitation anomalies, jailbreak fingerprints, or bad-faith pricing overrides (e.g., *"Give me this tour for free"* or *"Ignore previous instructions"*). This intent commands ultimate priority; execution short-circuits instantly to neutralize malicious vectors.

### Step 2: Multi-Stage Tour Routing & Semantic Cache Lookup
If the query passes safety policies, the exact tour scope is isolated. The application layers a **Dual-Stage Strict Semantic Cache** above remote APIs. If a semantically similar inquiry has been processed historically, the system answers instantly from the cache pool, achieving sub-millisecond execution times at zero token cost.

### Step 3: Vector Similarity Search & Context Assembly
In the event of a cache miss, the pipeline targets the relational storage engine. Utilizing `pgvector` inside PostgreSQL 16 backed by optimized Cosine Similarity indexes, the infrastructure executes a similarity search to isolate the **Top-K most relevant document chunks** harvested from verified internal PDF assets. This context payload is assembled and passed down to the main Generation stage.

### 4. Grounded Generation & Human-in-the-Loop Fallback
The text generator evaluates the combined Top-K context matrix to build the final response. Rigid **Anti-Gaslighting** and **Groundedness** constraints are strictly enforced at the prompt boundary: the model is prohibited from synthesizing unverified statistics or complying with user-driven price manipulation attempts. Robot-like AI signatures (e.g., *"As an AI language model..."*, *"In my database..."*) are explicitly banned.

* 💡 **Telegram Fallback Engine:** If the ingested context is insufficient to formulate a factual answer (thin context environment), the system prohibits hallucination. The model issues a warm, polite hold message while the backend asynchronously **routes the user's unhandled inquiry directly to the travel agency's administrator Telegram handle**, allowing a human operator to seamlessly take over the live session.

### Step 5: Transactional Checkout & Real-time Admin Orchestration
When an orientation moves into a definitive booking request, Arshine provisions a dynamic Checkout Link. Once the client executes the payment, an external Payment Gateway Webhook catches the event, updates the internal system state asynchronously, and broadcasts the transaction state directly to the **Real-Time Live Admin Dashboard**.

---

## 🛠️ System Architecture & Engineering

The platform implements strict **Clean Architecture** and **Domain-Driven Design (DDD)** paradigms, isolating business definitions completely from the external infrastructure runtime.


```
┌────────────────────────────────────────────────────────┐
│                       BUSINESS DOMAIN                  │
│  (Pure Entities, Value Objects, Repository Protocols)  │
└───────────────────────────▲────────────────____________┘
│
┌───────────────────────────┴────────────────────────────┐
│                      APPLICATION LAYER                 │
│   (Use Cases, AskQueryUseCase, prompts.py Constants)   │
└───────────────────────────▲────────────────────────────
│
┌───────────────────────────┴────────────────────────────┐
│                     INFRASTRUCTURE LAYER               │
│  (FastAPI, pgvector, SQLAlchemy Async, LLM Adapters)   │
└────────────────────────────────────────────────────────┘
```

* **Vendor-Agnostic LLM Layer (Strategy Pattern):** Modifying the `LLM_PROVIDER` key in the `.env` environment file triggers a hot-swap across remote execution backends, switching dynamically between Google Gemini (Gemini 2.5 Flash, Gemma 3 27B) and Groq (LLaMA 3.3 70B).
* **Fully Asynchronous Runtime:** Built on Python 3.12 and FastAPI's native async/await event loops to ensure high-concurrency capability.
* **Enterprise Storage Core:** Powered by PostgreSQL 16 + `pgvector` leveraging IVFFlat indexing for accelerated spatial queries, orchestrated via SQLAlchemy Async ORMs and Pydantic validation boundaries.

---

📬 **Contact & Architecture Walkthroughs:** dav2bit@gmail.com 

```
