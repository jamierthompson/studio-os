# StudioOS

**An AI-native operating system for interior design practices.**

StudioOS is a suite of AI agents modeled on the roles design firms actually hire — studio manager, project manager, sourcing specialist, procurement coordinator, transcriptionist, controller. For a solo designer, the agents are the team. For a 10-person firm, the same agents make every team member dramatically more efficient.

## The Vision

Running a design practice means staffing a dozen operational roles on top of the actual design work — client intake, project management, product sourcing, vendor coordination, financial tracking, meeting documentation. The design work is only a fraction of the job.

StudioOS handles the operations. Six AI agents manage the entire client lifecycle — from the first anonymous inquiry through project completion — so the designer focuses on design. A solo practice delivers the organized, high-touch experience of a large firm. A staffed firm moves faster with fewer operational bottlenecks.

## Architecture

Three design decisions shape the system:

1. **Structured vs. unstructured split.** Templates, task workflows, billing codes, and RFQ specs are operational data — they live in Postgres where queries are deterministic and fast. Design philosophy, bidding standards, and institutional knowledge are contextual wisdom — they live in a vector store for RAG retrieval.

2. **Tool layer.** Agents never touch the database directly. They call validated Python functions that enforce schema constraints, required fields, and business logic. The agent decides _what_ to do; deterministic code controls _how_ it happens. Every agent generates its own documents (proposals, RFQs, reports, meeting notes) through shared document generation tools — the templates enforce brand consistency regardless of which agent produces the output.

3. **Role-based agents spanning one continuous lifecycle.** Agents are modeled on real job roles, not lifecycle phases. A Project Manager doesn't stop existing when the contract is signed. A Studio Manager handles intake _and_ ongoing client relationships. One orchestrator reads client/project state and routes to the right role.

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
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐    │  │
│  │  │ Studio Mgr   │  │ Project Mgr  │  │ Sourcing Spec. │    │  │
│  │  │ Intake,      │  │ Timeline,    │  │ Product        │    │  │
│  │  │ screening,   │  │ tasks,       │  │ research,      │    │  │
│  │  │ scheduling,  │  │ proposals,   │  │ specs,         │    │  │
│  │  │ onboarding   │  │ budget       │  │ alternatives   │    │  │
│  │  └──────────────┘  └──────────────┘  └────────────────┘    │  │
│  │                                                            │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐    │  │
│  │  │ Procurement  │  │ Controller   │  │ Transcription- │    │  │
│  │  │ Coordinator  │  │ Invoicing,   │  │ ist            │    │  │
│  │  │ RFQs, POs,   │  │ revenue,     │  │ Meeting notes, │    │  │
│  │  │ vendor       │  │ profitability│  │ action items   │    │  │
│  │  │ logistics    │  │ reports      │  │                │    │  │
│  │  └──────────────┘  └──────────────┘  └────────────────┘    │  │
│  │                                                            │  │
│  │  Every agent spans the full client lifecycle.              │  │
│  │  Every agent generates its own documents via shared tools. │  │
│  └────────────────────────────────────────────────────────────┘  │
│                               │                                  │
│  ┌────────────────────────────┴───────────────────────────────┐  │
│  │                      Tool Layer                            │  │
│  │  Validated Python functions + document generation tools    │  │
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

## The Agent System

Six agents modeled on the roles design firms actually hire. Each agent spans the entire client lifecycle — there's no artificial boundary between pre-contract and active project. The orchestrator reads client/project state and routes to the right role.

### Studio Manager

The front door of the practice. Handles the screening chat (lead qualification), manages the design questionnaire, runs onboarding (welcome sequence, account setup, deposit tracking), and owns client relationship touchpoints throughout the project — check-ins, approval requests, status summaries. The agent can run autonomously, or it can draft and queue communications for the studio manager to review and send.

**Generates:** welcome packets, onboarding documents, client correspondence

### Project Manager

The coordination backbone. Owns the project workflow, plans the timeline, tracks phase transitions, schedules milestones (including all meetings — sales, site visits, design presentations), manages task dependencies, and generates proposals (because the PM understands scope and fee structure). When a meeting needs scheduling, a proposal needs writing, or a phase needs transitioning, this is the agent that knows it's time.

**Generates:** proposals, project timelines, status reports, meeting agendas

### Sourcing Specialist

Product research and specification. When the designer needs a dining table that fits a specific aesthetic, budget, and room dimension, this agent queries trade catalogs, pulls options from DesignRAG's knowledge base, compares specs, and presents curated alternatives for the designer's approval. This is the role that hits DesignRAG the hardest — it needs aesthetic context, material palettes, and trade vendor knowledge. In a firm, this is the work a junior designer or design assistant does.

**Generates:** product spec sheets, sourcing presentations, comparison documents

### Procurement Coordinator

Everything that happens after the designer approves a product selection. Generates RFQs from templates, issues purchase orders, manages vendor communication, tracks orders, coordinates freight, handles receiving, and manages damage claims. Distinct from Sourcing because the skill set is logistics, not taste. Heavily database-driven — order status, vendor contacts, delivery schedules.

**Generates:** RFQs, purchase orders, vendor correspondence

### Controller

Tracks the firm's financial picture — not the client's project investment (that's the PM), but the practice's revenue, expenses, invoicing, accounts receivable, and profitability per project. Generates financial reports and dashboards for the principal. In a solo practice, this is the agent that replaces the monthly bookkeeper visit. In a larger firm, it gives the actual bookkeeper real-time visibility and automates report generation.

**Generates:** invoices, financial reports, profitability analyses, AR aging reports

### Transcriptionist

Meeting recordings in, structured notes and action items out. The narrowest role, but a real one — firms hire note-takers for client meetings, or the designer spends an hour after every meeting writing up notes manually. Action items feed directly back to the PM as tasks. Meeting notes become project records stored in S3.

**Generates:** structured meeting notes, action item lists

### Orchestrator

Routes based on `clients.status` and `projects.phase` — it asks "what needs to happen next for this client?" and matches intent to role. The PM determines it's proposal time and assembles the inputs; the PM generates the proposal through shared document tools. The PM's task list says "source dining chairs" and the orchestrator routes to Sourcing. A transcript is uploaded and the orchestrator routes to the Transcriptionist, whose action items feed back to the PM. Every routing decision is logged.

## Projects

### 1. [DesignRAG](https://github.com/jamierthompson/design-rag) — Knowledge Layer

**Status:** Core pipeline complete

RAG system over the unstructured institutional knowledge corpus — trade standards, pricing philosophy, meeting procedures, site guidelines, and institutional knowledge. Agents query this for the "why" and "how" guidance that doesn't reduce to database columns. The Sourcing Specialist is the heaviest consumer, pulling aesthetic context and trade vendor knowledge for product research.

- FastAPI API with ingestion, query, and document management endpoints
- PDF and Markdown ingestion with LLM-based document classification
- RAG query pipeline with cited answers, optional LLM reranking, and token budget management
- Evaluation harness with 25 hand-labeled Q&A pairs
- **Stack:** FastAPI, ChromaDB, OpenAI, LangChain, pytest, Docker

### 2. [Studio Data](https://github.com/jamierthompson/studio-data) — Data Layer + Tool Layer

**Status:** In progress

The Postgres schema, S3 document storage, and validated tool functions that every agent calls. This is the foundational infrastructure that makes the agent system reliable.

- **Database:** Clients, projects, project workflow, trade templates, meeting templates, email templates, billing codes, pricing rules, financial records (invoices, payments, expenses), and a full activity log
- **Document Storage:** S3 (production) / MinIO (local dev) for generated agreements, proposals, RFQs, purchase orders, invoices, meeting notes, and financial reports
- **Tool Functions:** Validated Python functions for client management, project lifecycle, product sourcing, RFQ generation, purchase order management, invoice generation, financial reporting, meeting scheduling, document storage, email drafting, task tracking, and questionnaire management — each enforces business logic and logs to the audit trail
- **Document Generation:** Shared tools (`generate_proposal`, `generate_rfq`, `generate_invoice`, `draft_email`, etc.) available to all agents — templates enforce brand consistency regardless of which agent produces the output
- **Data Migrations:** Load cleaned operational data (project process CSV, RFQ templates, email templates, agreement clauses, questionnaire fields, billing codes) into Postgres

### 3. Agent System — Role-Based Orchestration

**Status:** Not started

Six agents modeled on real design firm roles, coordinated by a single orchestrator. All agents interact with data exclusively through the tool layer and can query DesignRAG for institutional knowledge. Each agent generates its own documents through shared document generation tools.

- **Studio Manager:** Lead qualification via chat, design questionnaire, onboarding flow, client relationship touchpoints, scheduling
- **Project Manager:** Project task workflow, timeline planning, milestone scheduling, phase transitions, proposal generation, budget tracking, task dependencies
- **Sourcing Specialist:** Product research via DesignRAG + trade catalogs, spec comparison, alternative curation, product presentations
- **Procurement Coordinator:** RFQ generation, purchase orders, vendor communication, order tracking, freight coordination, receiving, damage claims
- **Controller:** Invoice generation, revenue tracking, expense tracking, project profitability, AR aging, financial reports and dashboards
- **Transcriptionist:** Audio/transcript → structured meeting notes + action items, action items feed back to PM as tasks
- **Orchestrator:** Routes based on `clients.status` and `projects.phase`, matches intent to role, handles phase transitions with validation, logs every routing decision

### 4. StudioOS — The Application

**Status:** Not started

A single React (Vite) application with three access levels — one app with role-based views.

- **Unauthenticated:** Chat widget for lead screening, embedded on marketing site
- **Authenticated Client:** Project overview, documents served via S3 pre-signed URLs, approvals, communication thread, design questionnaire
- **Authenticated Admin:** All-clients pipeline view, cross-project dashboards, agent activity feed (AI vs. human audit trail), knowledge search via DesignRAG, direct agent commands, transcript upload, document management, financial dashboards

## The Client Lifecycle

Every client moves through one continuous workflow. The agents don't hand off between lifecycle phases — each role participates wherever its expertise is needed.

```
Visitor lands on site → Screening (chat) → Sales Meeting (in-person)
    → Proposal + Onboarding (agreement, deposit, questionnaire)
    → Active Project (Conceptual → Detailed → Purchasing + Construction → Installation → Reveal → Punch List)
```

Example flow through the agents:

1. **Studio Manager** screens the inquiry via chat, qualifies the lead
2. **Project Manager** schedules the sales meeting as a milestone
3. **Studio Manager** sends the meeting confirmation
4. **Project Manager** generates the proposal (scope, fees, timeline)
5. **Studio Manager** runs onboarding (agreement, deposit, welcome packet, questionnaire)
6. **Project Manager** activates the project task workflow, plans the timeline
7. **Sourcing Specialist** researches products for the conceptual phase
8. **Procurement Coordinator** generates RFQs and POs for approved selections
9. **Transcriptionist** processes each meeting → notes + action items → PM
10. **Controller** generates invoices at milestones, tracks payments, reports profitability

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

| Phase | Project                                  | Status      |
| ----- | ---------------------------------------- | ----------- |
| 0     | Data prep (task cleanup, doc extraction) | Complete    |
| 1     | DesignRAG — knowledge retrieval service  | Complete    |
| 2     | Studio Data - Data Layer + Tool Layer    | In progress |
| 3     | Agent System — role-based orchestration  | Not started |
| 4     | StudioOS — React application             | Not started |

## License

All StudioOS projects are released under the [MIT License](LICENSE).
