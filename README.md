# StudioOS

**An AI-native operating system for interior design practices.**

StudioOS is a single intelligent orchestrator backed by a robust tool layer, a RAG knowledge base, and a library of role profiles modeled on the jobs design firms actually hire — studio manager, project manager, sourcing specialist, procurement coordinator, controller, transcriptionist. When a task needs to be completed, the orchestrator composes a focused agent on the fly: the right role profile, the right tools, the right context. For a solo designer, these agents run the practice. For a staffed firm, the same system makes every team member dramatically more efficient. The architecture doesn't change based on firm size.

## The Vision

Running a design practice means staffing a dozen operational roles on top of the actual design work — client intake, project management, product sourcing, vendor coordination, financial tracking, meeting documentation. The design work is only a fraction of the job.

StudioOS handles the operations. One orchestrator dynamically deploys task-scoped agents across the entire client lifecycle — from the first anonymous inquiry through project completion — so the designer focuses on design. A solo practice delivers the organized, high-touch experience of a large firm. A staffed firm moves faster with fewer operational bottlenecks.

## Architecture

Four design decisions shape the system:

1. **Structured vs. unstructured split.** Templates, task workflows, billing codes, and RFQ specs are operational data — they live in Postgres where queries are deterministic and fast. Design philosophy, bidding standards, and institutional knowledge are contextual wisdom — they live in a vector store for RAG retrieval.

2. **Tool layer.** Agents never touch the database directly. They call validated Python functions that enforce schema constraints, required fields, and business logic. The agent decides _what_ to do; deterministic code controls _how_ it happens. Every agent generates its own documents through shared document generation tools — the templates enforce brand consistency regardless of which role profile is active.

3. **Dynamic agent composition.** There are no persistent sub-agents. The orchestrator reads client/project state, determines what needs to happen, and spawns a task-scoped agent with a composed system prompt (drawn from role profiles), a selected tool subset, and the relevant context. The agent executes the task and completes. Cross-domain tasks that would require routing between fixed agents instead get a single agent composed from multiple role profiles.

4. **Adaptive workflows.** Project workflow templates are guidelines, not scripts. The orchestrator uses them as institutional knowledge — the firm's accumulated wisdom about how projects generally flow — and adapts them to each client's specific scope, constraints, and preferences. Deviations are classified by impact and routed through appropriate approval tiers. Over time, consistently approved deviations feed back into the templates, so the system learns the firm's evolving practices.

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
│                  │                        │  • Financial reports │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Orchestrator                            │  │
│  │         Reads state → composes agent → executes task       │  │
│  │                                                            │  │
│  │  Role Profiles          Dynamic Agents (ephemeral)         │  │
│  │  ┌────────────────┐    ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐       │  │
│  │  │ Studio Manager │──▶ │ Prompt + tools + context  │       │  │
│  │  │ Project Manager│──▶ │ composed per task         │       │  │
│  │  │ Sourcing Spec. │──▶ │                           │       │  │
│  │  │ Procurement    │──▶ │ Cross-role tasks blend    │       │  │
│  │  │ Controller     │──▶ │ multiple profiles         │       │  │
│  │  │ Transcriptionist──▶ │                           │       │  │
│  │  └────────────────┘    └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                               │                                  │
│  ┌────────────────────────────┴───────────────────────────────┐  │
│  │                      Tool Layer                            │  │
│  │  Validated Python functions + document generation tools    │  │
│  │  + human-in-the-loop approval routing                      │  │
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

## Role Profiles

Six role profiles modeled on the jobs design firms actually hire. These aren't separate agents — they're prompt fragments and tool configurations the orchestrator draws from when composing a task-scoped agent. A simple task uses one profile. A cross-domain task blends multiple profiles into a single agent with the combined context and tools.

### Studio Manager

The front door of the practice. Handles the screening chat (lead qualification), manages the design questionnaire, runs onboarding (welcome sequence, account setup, deposit tracking), and owns client relationship touchpoints throughout the project — check-ins, approval requests, status summaries. Agents with this profile can run autonomously, or they can draft and queue communications for the actual studio manager to review.

**Tools:** `screen_lead`, `manage_questionnaire`, `run_onboarding`, `draft_email`, `schedule_meeting`

### Project Manager

The coordination backbone. Interprets workflow templates to build project-specific plans, tracks phase transitions, schedules milestones (including all meetings), manages task dependencies, and generates proposals (because scope and fee structure are PM reasoning). When a deviation from the workflow template is needed, the PM-profiled agent proposes changes through the tool layer, which classifies them by impact and routes to the appropriate approval tier.

**Tools:** `build_project_plan`, `update_project_plan`, `track_tasks`, `generate_proposal`, `schedule_milestone`

### Sourcing Specialist

Product research and specification. When the designer needs options that fit a specific aesthetic, investment level, and dimension, agents with this profile query vendor catalogs, pull from DesignRAG's knowledge base, compare specs, and present curated alternatives. This profile draws the heaviest on DesignRAG — it needs aesthetic context, material palettes, and trade vendor knowledge.

**Tools:** `query_designrag`, `search_products`, `compare_specs`, `generate_spec_sheet`

### Procurement Coordinator

Everything that happens after the designer approves a product selection. RFQ generation, purchase orders, vendor communication, order tracking, freight coordination, receiving, damage claims. Distinct from Sourcing because the skill set is logistics, not taste. Heavily database-driven.

**Tools:** `generate_rfq`, `create_purchase_order`, `track_order`, `draft_vendor_email`, `log_receiving`

### Controller

Tracks the firm's financial picture — not the client's project investment (that's the PM), but the practice's revenue, expenses, invoicing, accounts receivable, and profitability per project. Generates financial reports and dashboards for the principal.

**Tools:** `generate_invoice`, `track_payment`, `log_expense`, `generate_financial_report`, `calculate_profitability`

### Transcriptionist

Meeting recordings in, structured notes and action items out. Action items feed directly back into the project plan as tasks. Meeting notes become project records stored in S3.

**Tools:** `process_transcript`, `extract_action_items`, `store_meeting_notes`

### Cross-Role Composition

Some tasks don't map to a single role. The orchestrator handles this by composing agents from multiple profiles:

- **Proposal generation** blends PM (scope, timeline, dependencies) + Controller (fee calculations, billing codes, margin targets) + Studio Manager (client context from screening and onboarding activities)
- **Project kickoff** blends PM (activating the workflow, scheduling milestones) + Studio Manager (onboarding flow, welcome packet) + Controller (deposit tracking, invoicing schedule)
- **Vendor selection** blends Sourcing (product research, spec comparison) + Procurement (vendor terms, lead times, logistics feasibility)

## Adaptive Workflows

Project workflow templates are the firm's accumulated wisdom about how projects generally flow — not rigid scripts. The system treats them as guidelines that a PM-profiled agent adapts to each client's specific reality.

### Three Layers

**Workflow templates** are institutional knowledge stored in Postgres. "A typical whole-home renovation moves through these phases, with roughly these tasks, in roughly this order, with these dependencies." A firm might have several: whole-home renovation, single-room refresh, new construction, commercial. Each task in the template carries metadata — `is_milestone`, `is_client_touchpoint`, `can_skip`, `typical_duration`, `dependencies`.

**Project context** is what the Studio Manager profile collects — scope, investment, timeline constraints, client preferences, site conditions. "Kitchen and primary bath, hard deadline for Thanksgiving, condo with freight elevator blackout hours."

**The project plan** is what the PM-profiled agent produces — the template adapted to the context. It might compress a phase for a tight deadline, skip a task that doesn't apply, add a custom task the template doesn't include, or reorder a sequence based on site constraints.

### Human-in-the-Loop Approval

Not every deviation from the template needs human review. The tool layer classifies each proposed change by impact and routes it to the appropriate tier:

**Auto-approve + log:** Duration adjustments, task reordering within a phase, adding subtasks. Low-impact changes that a real PM would make without asking.

**Flag for review:** Skipping a task entirely, adding a new phase, changing phase order, significantly compressing a milestone. The change appears in the designer's activity feed for review — the agent continues working on non-dependent tasks.

**Require approval before proceeding:** Skipping a milestone, removing a client touchpoint, changes that affect the client experience. The agent pauses the affected workflow branch until the designer approves.

The classification isn't agent reasoning — it's metadata on the template tasks. A task marked `is_milestone: true, can_skip: false` automatically triggers the approval tier when anyone tries to remove it.

### The Learning Loop

When a designer consistently approves the same kind of deviation — "yes, always skip as-is site measure for new construction" — the system surfaces it as a suggestion to update the template. Approved deviations feed back into the workflow templates over time, so the system learns the firm's evolving practices.

This is where StudioOS crosses from tool to partner: it doesn't just execute your process, it learns your process.

## Projects

### 1. [DesignRAG](https://github.com/jamierthompson/design-rag) — Knowledge Layer

**Status:** Core pipeline complete

RAG system over the unstructured institutional knowledge corpus — trade standards, pricing philosophy, meeting procedures, site guidelines, and institutional knowledge. The orchestrator's spawned agents query this for the "why" and "how" guidance that doesn't reduce to database columns. Sourcing-profiled agents are the heaviest consumers, pulling aesthetic context and trade vendor knowledge for product research.

- FastAPI API with ingestion, query, and document management endpoints
- PDF and Markdown ingestion with LLM-based document classification
- RAG query pipeline with cited answers, optional LLM reranking, and token budget management
- Evaluation harness with 25 hand-labeled Q&A pairs
- **Stack:** FastAPI, ChromaDB, OpenAI, LangChain, pytest, Docker

### 2. [Studio Data](https://github.com/jamierthompson/studio-data) — Data Layer + Tool Layer

**Status:** In progress

The Postgres schema, S3 document storage, and validated tool functions that every spawned agent calls. This is the foundational infrastructure that makes the agent system reliable instead of fragile.

- **Database:** Clients, projects, workflow templates (phases, task types, metadata, dependencies), trade templates, meeting templates, email templates, billing codes, pricing rules, financial records (invoices, payments, expenses), deviation log, and a full activity log
- **Document Storage:** S3 (production) / MinIO (local dev) for generated agreements, proposals, RFQs, purchase orders, invoices, meeting notes, and financial reports
- **Tool Functions:** Validated Python functions for client management, project lifecycle, adaptive workflow management, product sourcing, RFQ generation, purchase order management, invoice generation, financial reporting, meeting scheduling, document storage, email drafting, task tracking, and questionnaire management — each enforces business logic, classifies deviations, routes approvals, and logs to the audit trail
- **Document Generation:** Shared tools (`generate_proposal`, `generate_rfq`, `generate_invoice`, `draft_email`, etc.) available to all spawned agents — templates enforce brand consistency regardless of which role profile is active
- **Data Migrations:** Load cleaned operational data (workflow templates, RFQ templates, email templates, agreement clauses, questionnaire fields, billing codes) into Postgres

### 3. Agent System — Dynamic Orchestration

**Status:** Not started

A single orchestrator that composes task-scoped agents from role profiles, selected tool subsets, and relevant context. No persistent sub-agents — each task gets a focused agent that executes and completes.

- **Orchestrator:** Reads `clients.status` and `projects.phase`, determines what needs to happen next, composes the appropriate agent (role profile + tools + context), manages the human-in-the-loop approval queue, logs every decision
- **Role Profiles:** Six versioned prompt fragments (Studio Manager, Project Manager, Sourcing Specialist, Procurement Coordinator, Controller, Transcriptionist) that can be composed individually or blended for cross-domain tasks
- **Adaptive Workflow Engine:** PM-profiled agents interpret workflow templates as guidelines, propose context-specific plans, deviations are classified by template metadata and routed through approval tiers, approved patterns feed back into templates over time
- **Human-in-the-Loop:** Three-tier approval system (auto-approve, flag for review, require approval) driven by task metadata in workflow templates, surfaced in the designer's activity feed

### 4. StudioOS — The Application

**Status:** Not started

A single React (Vite) application with three access levels — one app with role-based views.

- **Unauthenticated:** Chat widget for lead screening, embedded on marketing site
- **Authenticated Client:** Project overview, documents served via S3 pre-signed URLs, approvals, communication thread, design questionnaire
- **Authenticated Admin:** All-clients pipeline view, cross-project dashboards, agent activity feed with approval queue (AI vs. human audit trail), knowledge search via DesignRAG, direct agent commands, transcript upload, document management, financial dashboards, workflow template management

## The Client Lifecycle

Every client moves through one continuous workflow. The orchestrator spawns the right agent for each task — no handoffs between persistent agents, no artificial phase boundaries.

```
Visitor lands on site → Screening (chat) → Sales Meeting (in-person)
    → Proposal + Onboarding (agreement, deposit, questionnaire)
    → Active Project (Conceptual → Detailed → Purchasing + Construction → Installation → Reveal → Punch List)
```

Example flow through the system:

1. **Orchestrator** detects new inquiry → spawns agent with Studio Manager profile → screens via chat, qualifies lead
2. **Orchestrator** detects qualified lead → spawns agent with PM profile → schedules sales meeting as milestone
3. **Orchestrator** detects meeting scheduled → spawns agent with Studio Manager profile → sends confirmation email
4. **Orchestrator** detects post-meeting state → spawns agent with PM + Controller profiles → generates design agreement (scope, fees, timeline, payment schedule)
5. **Orchestrator** detects signed agreement → spawns agent with Studio Manager profile → runs onboarding (welcome packet, questionnaire, deposit tracking)
6. **Orchestrator** detects onboarding complete → spawns agent with PM profile → builds project plan from workflow template + client context, flags deviations for review
7. **Orchestrator** detects conceptual phase active → spawns agent with Sourcing profile → researches products, presents curated options
8. **Orchestrator** detects approved selections → spawns agent with Procurement profile → generates RFQs, issues POs
9. **Orchestrator** detects transcript uploaded → spawns agent with Transcriptionist profile → produces structured notes, action items feed back to project plan
10. **Orchestrator** detects milestone reached → spawns agent with Controller profile → generates invoice, tracks payment

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
| 3     | Agent System — dynamic orchestration     | Not started |
| 4     | StudioOS — React application             | Not started |

## License

All StudioOS projects are released under the [MIT License](LICENSE).
