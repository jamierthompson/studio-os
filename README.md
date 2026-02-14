# StudioOS

**An AI-native operating system for a solo interior design practice.**

StudioOS is a suite of interconnected AI systems designed to augment a one-person design studio — handling knowledge retrieval, client onboarding, and project management that would normally require a full team. Each project is a standalone service that communicates over HTTP/REST.

## The Vision

Running a solo design practice means wearing every hat — client intake, proposal writing, vendor coordination, project tracking, knowledge management. The actual design work is only a fraction of the job.

StudioOS handles the rest. AI agents manage the business operations — onboarding clients, drafting proposals, prepping meetings, generating RFQs, tracking timelines — so the designer can spend their time designing.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                       StudioOS                           │
│                     (Dashboard)                          │
│                                                          │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │  DesignRAG   │  │  ClientBot   │  │  ProjectPilot  │  │
│  │  (Knowledge) │  │  (Intake)    │  │  (Management)  │  │
│  └──────────────┘  └──────────────┘  └────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

All services communicate over **HTTP/REST**, making each project independently deployable and testable.

## Projects

### 1. [DesignRAG](https://github.com/jamierthompson/design-rag) — Knowledge Base
**Status:** In progress

A RAG system over institutional design knowledge. Documents are chunked, embedded, and made queryable through a FastAPI API with cited answers.

- **Stack:** FastAPI, ChromaDB, OpenAI, LangChain text splitting

### 2. ClientBot — Intake & Onboarding Agent
**Status:** Planned

A conversational agent that handles the pre-contract client pipeline — from initial inquiry through signed agreement. Conducts intake interviews, generates scoped proposals, and drafts design agreements.

### 3. ProjectPilot — Design Project Manager
**Status:** Planned

A multi-agent system that manages active design projects. Specialized sub-agents handle meeting prep, note-taking, vendor RFQ generation, and timeline tracking.

### 4. StudioOS — Business Dashboard
**Status:** Planned

The integration layer that ties everything together into a unified web interface — client pipeline views, project dashboards, knowledge search, and document management.

## How Projects Connect

Each backend project is a standalone FastAPI service with its own data store and API. StudioOS (the dashboard) acts as the integration layer:

- **DesignRAG** provides knowledge retrieval over design documents
- **ClientBot** handles the client intake pipeline, pulling from project knowledge for proposal context
- **ProjectPilot** manages active projects, drawing on knowledge and client data from the other services
- **StudioOS** surfaces everything through a single interface

## Tech Stack

| Concern | Technology |
|---|---|
| Language | Python 3.12 |
| API Framework | FastAPI |
| Package Management | uv |
| Linting/Formatting | Ruff |
| Testing | pytest |
| Containerization | Docker + Docker Compose |
| AI/ML | OpenAI API (embeddings + LLM) |
| Vector Storage | ChromaDB |

## License

All StudioOS projects are released under the [MIT License](LICENSE).
