# Guide — Building a Production AI Assistant for an Insurance Company

> Start here. This file explains **what** we are building, **how the journey works**, and gives you the
> **architecture map** that every lesson zooms into.

---

## 1. What you will build

A **backend service** that powers an AI customer assistant:

- 💬 Any chat tool (web widget, mobile app, WhatsApp bridge...) connects to it over a **socket**.
- 📚 It answers product and procedure questions from the company's documents using **RAG**.
- 🔐 It verifies the customer with an **OTP over SMS/email** before showing personal data.
- 🛠️ It performs **actions** (check claims, submit a claim, email a policy) by calling the **company API**.
- 🧠 It keeps **sessions** and conversation memory, streams answers, and scales horizontally.

At the end of the journey you have a tested, observable, containerized app ready for production.

## 2. How the journey works

```mermaid
flowchart TB
    subgraph You["You"]
        Y1["Say: continue / next / status"]
    end
    subgraph Skill["Journey skill (.claude/skills/ai-assistant-journey)"]
        K1["Reads TRACKER → finds current step"]
        K2["Reads PROJECT_BRIEF + TECH_RULES"]
        K3["Teaches the lesson<br/>(why → concept → diagram → build → verify)"]
        K4["Updates TRACKER + lesson checklist"]
    end
    subgraph Files["docs/"]
        F1["TRACKER.md<br/>stages · steps · status"]
        F2["PROJECT_BRIEF.md<br/>what the app does"]
        F3["TECH_RULES.md<br/>stack + rules R-xx"]
        F4["lessons/stage-NN/X.Y-*.md<br/>one file per step"]
    end
    Y1 --> K1 --> K2 --> K3 --> K4
    K1 -.reads.-> F1
    K2 -.reads.-> F2 & F3
    K3 -.writes.-> F4
    K4 -.writes.-> F1
```

| File | Purpose |
|------|---------|
| [GUIDE.md](GUIDE.md) | This overview + architecture map |
| [TRACKER.md](TRACKER.md) | **Where you are.** All stages/steps with status, decisions log |
| [PROJECT_BRIEF.md](PROJECT_BRIEF.md) | **What** the app does — edit it to change the business scope |
| [TECH_RULES.md](TECH_RULES.md) | **How** it is built — stack and engineering rules (`R-xx`) |
| [lessons/](lessons/) | One Markdown lesson per step, kept forever as your notes |
| `.claude/skills/ai-assistant-journey/` | The tutor's rules — lets you resume from any stage in a new session |

**To resume anytime:** open Claude Code in this folder and say **"continue the journey"**
(or type `/ai-assistant-journey`). Other useful phrases: *"status"*, *"start step 3.2"*,
*"explain again"*, *"quiz me"*, *"change rule R-xx to ..."*.

**Every lesson has:** 🎯 Goal · 🤔 Why · 🧠 Concepts · 🗺️ Diagram · 🛠️ Build · ✅ Verify ·
⚠️ Production notes · 🧪 Exercise · 💭 Reflection · ☑️ Checklist.

**Status legend:** ⬜ Not started · 🟨 In progress · ✅ Done · ⏭️ Skipped · 🔁 Needs revisit

> 💡 **Viewing diagrams in VS Code:** install the extension *"Markdown Preview Mermaid Support"*
> (`bierner.markdown-mermaid`), then open any `.md` file and press `Ctrl+Shift+V`. GitHub renders them natively.

## 3. Architecture map

```mermaid
flowchart LR
    subgraph Clients["Chat tools — not built here"]
        W["Web widget"]
        M["Mobile app"]
        B["WhatsApp / other bridge"]
    end

    subgraph Backend["AI Assistant Backend (this project)"]
        GW["Channel Gateway<br/>Socket.IO · WebSocket"]
        SES["Session Manager<br/>history · memory"]
        ORC["Conversation Orchestrator<br/>routing · agent loop · state machine"]
        POL["Tool Policy<br/>verification · confirmation"]
        RAG["RAG Engine<br/>hybrid search · rerank · cite"]
        TOOLS["Tool Registry<br/>policies · claims · documents"]
        VER["Verification Service<br/>OTP"]
        ADM["Admin REST API<br/>knowledge · audits"]
        WRK["Background Workers<br/>ingestion · email · SMS"]
    end

    subgraph Data["Data"]
        PG[("PostgreSQL<br/>+ pgvector")]
        RD[("Redis<br/>sessions · OTP · queue · pub/sub")]
    end

    subgraph Ext["External services"]
        LLM["LLM provider<br/>Claude / OpenAI / Ollama"]
        EMB["Embedding model"]
        CAPI["Company Insurance API"]
        NOTIF["Email · SMS providers"]
    end

    W & M & B <--> GW
    GW <--> SES
    GW <--> ORC
    ORC --> RAG
    ORC --> POL --> TOOLS
    POL <--> VER
    ORC <--> LLM
    RAG --> EMB
    RAG --> PG
    SES --> RD
    SES --> PG
    VER --> RD
    VER --> WRK
    TOOLS --> CAPI
    ADM --> WRK
    WRK --> EMB
    WRK --> PG
    WRK --> NOTIF
```

| Component | Responsibility | Built in stage |
|-----------|----------------|----------------|
| Channel Gateway | Socket connections, auth, event protocol, streaming | 4 |
| Session Manager | Session lifecycle, history, memory, expiry | 4 |
| Conversation Orchestrator | Decides what to do per message; runs the LLM ↔ tools loop | 5 |
| RAG Engine | Ingest, chunk, embed, retrieve, rerank, answer with citations | 3 |
| Verification Service | Customer identification, OTP send/verify, verification levels | 6 |
| Tool Policy | Enforces "verified?" and "confirmed?" **in code** before any tool runs | 6 |
| Tool Registry | Typed tools that call the company API | 7 |
| Admin REST API | Knowledge upload/reindex, audit queries, health | 3, 7 |
| Background Workers | Ingestion, email, SMS — off the request path | 3, 6 |
| AI provider adapters | Swappable LLM + embedding clients | 2 |

### The golden rule
> **The LLM proposes, the code decides.** The model may *ask* to call `get_claim_status`, but the
> Tool Policy checks the session's verification level and blocks it if the customer has not passed OTP.
> Prompts are guidance, not security.

## 4. The three core flows

### 4.1 Knowledge question (RAG) — no verification needed
```mermaid
sequenceDiagram
    autonumber
    participant C as Chat tool
    participant G as Gateway
    participant O as Orchestrator
    participant R as RAG Engine
    participant DB as Postgres+pgvector
    participant L as LLM
    C->>G: message "Does car insurance cover windshields?"
    G->>O: handle(session, message)
    O->>R: retrieve(query)
    R->>DB: hybrid search (vector + full-text)
    DB-->>R: top chunks
    R-->>O: reranked chunks + sources
    O->>L: system prompt + history + chunks + question
    L-->>O: streamed tokens
    O-->>G: tokens
    G-->>C: stream reply + citations
```

### 4.2 Personal question — OTP first
```mermaid
sequenceDiagram
    autonumber
    participant C as Customer (chat tool)
    participant O as Orchestrator
    participant L as LLM
    participant P as Tool Policy
    participant V as Verification
    participant N as SMS / Email
    participant API as Company API
    C->>O: "What's the status of my claim?"
    O->>L: prompt + available tools
    L-->>O: call get_claims()
    O->>P: allowed? (session level = anonymous)
    P-->>O: ❌ needs OTP
    O-->>C: "Please share your policy number"
    C->>O: "CAR-2025-00187"
    O->>API: look up registered contact
    API-->>O: phone +9665••••89
    O->>V: create OTP
    V->>N: send code (background job)
    O-->>C: "Code sent to phone ending ••89"
    C->>O: "482913"
    O->>V: verify code
    V-->>O: ✅ level = verified (15 min)
    O->>P: allowed? get_claims()
    P-->>O: ✅
    O->>API: GET /customers/{id}/claims
    API-->>O: claims
    O->>L: tool result
    L-->>O: answer tokens
    O-->>C: "Claim CLM-88412 is under assessment..."
```

### 4.3 Session & verification lifecycle
```mermaid
stateDiagram-v2
    [*] --> Anonymous: socket connects
    Anonymous --> Identifying: personal question / action
    Identifying --> OtpSent: customer found, code sent
    Identifying --> Anonymous: not found / gave up
    OtpSent --> Verified: correct code
    OtpSent --> OtpSent: wrong code (attempts < 3)
    OtpSent --> Locked: 3 wrong codes
    Verified --> AwaitingConfirmation: write action proposed
    AwaitingConfirmation --> Verified: customer says yes → action runs / no → cancelled
    Verified --> Anonymous: verification expired
    Locked --> Anonymous: lock window passes
    Anonymous --> [*]: session idle timeout
    Verified --> [*]: session idle timeout
```

## 5. Journey map

```mermaid
flowchart TB
    subgraph A["Part A — Foundations"]
        S0["0 · Foundations<br/>concepts · skeleton · infra"]
        S1["1 · Backend base<br/>logging · DB · errors · tests"]
        S2["2 · AI providers<br/>LLM · streaming · embeddings · prompts"]
    end
    subgraph B["Part B — Intelligence"]
        S3["3 · RAG knowledge<br/>ingest · chunk · search · cite"]
        S4["4 · Chat gateway<br/>protocol · sockets · sessions"]
        S5["5 · Orchestrator<br/>routing · tools · flows · guardrails"]
    end
    subgraph C["Part C — Trust & Actions"]
        S6["6 · OTP verification"]
        S7["7 · Company actions<br/>mock API · resilience · audit"]
    end
    subgraph D["Part D — Production"]
        S8["8 · Quality & evals"]
        S9["9 · Performance & hardening"]
        S10["10 · Deployment"]
    end
    S0 --> S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7 --> S8 --> S9 --> S10
```

Full step list with status: **[TRACKER.md](TRACKER.md)**.

## 6. Target repository layout

```
mini-rag/
├─ .claude/skills/ai-assistant-journey/   # tutor skill (resume from any stage)
├─ app/                                   # the backend (layout in TECH_RULES.md §2)
├─ mock_company_api/                      # fake insurance API for development
├─ tests/                                 # unit · integration · e2e · evals
├─ docker/                                # compose + Dockerfiles
├─ docs/
│  ├─ GUIDE.md · TRACKER.md · PROJECT_BRIEF.md · TECH_RULES.md
│  ├─ protocol/                           # socket event contract (stage 4)
│  └─ lessons/stage-NN-*/X.Y-*.md         # one lesson per step
├─ pyproject.toml · uv.lock · .env.example
```
