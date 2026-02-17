# StudioOS

**An AI-native operating system for a solo interior design practice.**

StudioOS replaces the team a solo designer doesn't have. A suite of AI agents handles the work that would normally require a full staff — screening leads, writing proposals, prepping meetings, generating vendor RFQs, tracking timelines — so the designer can focus on actual design work.

## The Vision

Running a solo design practice means wearing every hat — client intake, proposal writing, vendor coordination, project tracking, knowledge management. The actual design work is only a fraction of the job.

StudioOS handles the rest. One designer and a suite of AI agents manage the entire client lifecycle — from the first anonymous inquiry through project completion — delivering the organized, high-touch experience of a large firm from a solo practice.

## Architecture

Three design decisions shape the system:

1. **Structured vs. unstructured split.** Templates, task workflows, billing codes, and RFQ specs are operational data — they live in Postgres where queries are deterministic and fast. Design philosophy, bidding standards, and institutional knowledge are contextual wisdom — they live in a vector store for RAG retrieval.

2. **Tool layer.** Agents never touch the database directly. They call validated Python functions that enforce schema constraints, required fields, and business logic. The agent decides _what_ to do; deterministic code controls _how_ it happens. This gives us observability, testability, safety, and an audit trail for free.

3. **One continuous lifecycle.** There's no artificial boundary between intake and project management. A client moves through screening → proposal → agreement → active project → completion. One orchestrator reads state and routes to the right sub-agent.

```
┌──────────────────────────────────────────────────────────────────┐
│                          StudioOS                                │
│              (React app — role-based views)                      │
│                                                                  │
│  Unauthenticated │  Authenticated Client  │  Authenticated Admin │
│  • Chat widget   │  • Project dashboard   │  • All clients       │
│  • Screening     │  • Documents           │  • All projects      │
│                  │  • Approvals           │  • Agent activity    │
│                  │  • Communication       │  • Knowledge search  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Agent System                            │  │
│  │                   (Orchestrator)                           │  │
│  │                                                            │  │
│  │  Pre-Contract Agents        Active Project Agents          │  │
│  │  ┌──────────────────┐      ┌──────────────────────────┐    │  │
│  │  │ Screening Agent  │      │ Meeting Prep Agent       │    │  │
│  │  │ Meeting Scheduler│      │ Meeting Notes Agent      │    │  │
│  │  │ Proposal Agent   │      │ RFQ Generator Agent      │    │  │
│  │  │ Onboarding Agent │      │ Timeline & Task Agent    │    │  │
│  │  │ Questionnaire Agt│      │ Communication Agent      │    │  │
│  │  └──────────────────┘      └──────────────────────────┘    │  │
│  └────────────────────────────┬───────────────────────────────┘  │
│                               │                                  │
│  ┌────────────────────────────┴───────────────────────────────┐  │
│  │                      Tool Layer                            │  │
│  │                (Validated Python functions)                │  │
│  └─────────┬──────────────────────────────┬───────────────────┘  │
│            │                              │                      │
│  ┌─────────┴────────────┐  ┌──────────────┴───────────┐          │
│  │  Postgres + ChromaDB │  │       S3 / MinIO         │          │
│  │  (data + vectors)    │  │      (documents)         │          │
│  └──────────────────────┘  └──────────────────────────┘          │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                      DesignRAG                             │  │
│  │           (Knowledge retrieval service)                    │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Projects

### 1. [DesignRAG](https://github.com/jamierthompson/design-rag) — Knowledge Layer
**Status:** Core pipeline complete

RAG system over the unstructured institutional knowledge corpus — trade standards, pricing philosophy, meeting procedures, site guidelines, and institutional knowledge. Agents query this for the "why" and "how" guidance that doesn't reduce to database columns.

- FastAPI API with ingestion, query, and document management endpoints
- PDF and Markdown ingestion with LLM-based document classification
- RAG query pipeline with cited answers, optional LLM reranking, and token budget management
- Evaluation harness with 25 hand-labeled Q&A pairs (Recall@5: 1.00, Answer Accuracy: 4.72/5)
- **Stack:** FastAPI, ChromaDB, OpenAI, LangChain, pytest, Docker

### 2. Data Layer + Tool Layer — Foundation
**Status:** Not started

The Postgres schema, S3 document storage, and validated tool functions that every agent calls. This is the foundational infrastructure that makes the agent system reliable instead of fragile.

- **Database:** Clients, projects, 160-task project workflow template, trade templates, meeting templates, email templates, billing codes, pricing rules, and a full activity log
- **Document Storage:** S3 (production) / MinIO (local dev) for generated agreements, proposals, RFQs, and meeting notes
- **Tool Functions:** Validated Python functions for client management, project lifecycle, RFQ generation, meeting scheduling, document storage, email drafting, task tracking, and questionnaire management — each enforces business logic and logs to the audit trail
- **Data Migrations:** Load cleaned operational data (project process CSV, RFQ templates, email templates, agreement clauses, questionnaire fields, billing codes) into Postgres

### 3. Agent System — Unified Orchestrator
**Status:** Not started

A single orchestration system managing the entire client lifecycle. One orchestrator reads client/project state from the database and routes to the right sub-agent. All sub-agents interact with data exclusively through the tool layer and can query DesignRAG for institutional knowledge.

- **Pre-Contract:** Screening Agent (lead qualification via chat), Meeting Scheduler, Proposal Agent (fee calculation + document generation), Onboarding Agent (agreement, deposit, account creation), Questionnaire Agent (full design questionnaire)
- **Active Project:** Meeting Prep Agent (agenda generation), Meeting Notes Agent (transcript → structured notes + action items), RFQ Generator Agent (template-driven vendor specs), Timeline & Task Agent (phase tracking + status updates), Communication Agent (templated client correspondence)
- **Orchestrator:** Routes based on `clients.status` and `projects.phase`, handles phase transitions with validation, logs every routing decision

### 4. StudioOS — The Application
**Status:** Not started

A single React (Vite) application with three access levels — one app with role-based views.

- **Unauthenticated:** Chat widget for lead screening, embedded on marketing site
- **Authenticated Client:** Project overview, documents served via S3 pre-signed URLs, approvals, communication thread, design questionnaire
- **Authenticated Designer:** All-clients pipeline view, cross-project dashboards, agent activity feed (AI vs. human audit trail), knowledge search via DesignRAG, direct agent commands, transcript upload, document management

## The Client Lifecycle

Every client moves through one continuous workflow managed by the Agent System:

```
Visitor lands on site → Screening (chat) → Sales Meeting (in-person)
    → Proposal + Onboarding (agreement, deposit, questionnaire)
    → Active Project (Conceptual → Detailed → Purchasing → Installation → Reveal → Punch List)
```

The boundary between pre-contract and active project is a phase transition, not a system boundary. The orchestrator handles routing across the entire lifecycle.

## Tech Stack

| Concern            | Technology                                      |
| ------------------ | ----------------------------------------------- |
| Language           | Python 3.12                                     |
| API Framework      | FastAPI                                         |
| Database           | Postgres (structured data) + ChromaDB (vectors) |
| Document Storage   | S3 (production) / MinIO (local dev)             |
| Embeddings         | OpenAI `text-embedding-3-small`                 |
| LLM                | OpenAI (generation)                             |
| Text Splitting     | LangChain `RecursiveCharacterTextSplitter`      |
| Package Management | uv                                              |
| Linting/Formatting | Ruff                                            |
| Testing            | pytest                                          |
| Containerization   | Docker + Docker Compose                         |
| Frontend           | React (Vite)                                    |

## Current Status

| Phase | Project                                    | Status         |
| ----- | ------------------------------------------ | -------------- |
| 0     | Data prep (task cleanup, doc extraction)   | Complete       |
| 1     | DesignRAG — knowledge retrieval service    | In progress    |
| 2     | Data Layer + Tool Layer                    | Not started    |
| 3     | Agent System                               | Not started    |
| 4     | StudioOS — React application               | Not started    |

## License

All StudioOS projects are released under the [MIT License](LICENSE).