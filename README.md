# StudioOS

**An AI-native operating system for a solo interior design practice.**

StudioOS is a suite of interconnected AI tools designed to augment a one-person design studio — handling the knowledge retrieval, client communication, project management, and visual generation that would normally require a full team. Each project is a standalone service that works independently but connects via REST APIs to form a cohesive workflow.

## The Vision

Interior design firms rely on institutional knowledge: material specs, building codes, vendor catalogs, past project decisions. In a large firm, this knowledge lives across team members. A solo practitioner has to hold it all — or lose it.

StudioOS solves this by building AI-powered tools that capture, organize, and surface design knowledge on demand. Instead of replacing the designer, these tools handle the cognitive overhead so the designer can focus on what they do best: creating spaces that work for people.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                       StudioOS                           │
│                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │  DesignRAG   │    │  ClientHub  │    │  MoodBoard  │  │
│  │  (Knowledge) │◄──►│  (Comms)    │◄──►│  (Visual)   │  │
│  └──────┬───────┘    └──────┬──────┘    └─────────────┘  │
│         │                   │                             │
│         └─────────┬─────────┘                             │
│                   ▼                                       │
│          ┌─────────────────┐                              │
│          │   ProjectPilot  │                              │
│          │  (Orchestrator) │                              │
│          └─────────────────┘                              │
└──────────────────────────────────────────────────────────┘
```

All services communicate over **HTTP/REST**, making each project independently deployable and testable.

## Projects

### 1. [DesignRAG](https://github.com/jamierthompson/design-rag) — Knowledge Layer
**Status:** In progress

RAG system over institutional design knowledge. Upload design documents (PDFs, Markdown), then ask questions and get cited answers grounded in your source material.

- **Stack:** FastAPI, ChromaDB, OpenAI embeddings, LangChain text splitting
- **Key feature:** Source citations on every answer — know exactly where information came from

### 2. ClientHub — Client Communication
**Status:** Planned

AI-assisted client communication tool. Draft proposals, summarize meeting notes, and maintain a searchable history of client interactions and decisions.

### 3. MoodBoard — Visual Generation
**Status:** Planned

AI-powered mood board and concept visualization. Generate design concepts, material palettes, and spatial layouts from text descriptions and reference images.

### 4. ProjectPilot — Orchestrator
**Status:** Planned

Project management layer that ties everything together. Track project timelines, coordinate between services, and provide a unified dashboard for the design practice.

## How Projects Connect

Each project is a standalone FastAPI service with its own database and API. ProjectPilot acts as the orchestration layer, calling into the other services as needed:

- **DesignRAG** provides knowledge retrieval — when a client asks about material options, ProjectPilot queries DesignRAG for relevant specs and past project decisions
- **ClientHub** handles communication context — meeting summaries and client preferences feed into DesignRAG's knowledge base
- **MoodBoard** generates visuals — concept boards pull material information from DesignRAG and client preferences from ClientHub

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

## Built in Public

This project is being built openly as a learning exercise and portfolio piece. Each repository has a clean commit history that tells the story of how it was built — from initial scaffold through feature implementation and testing.

## License

All StudioOS projects are released under the [MIT License](LICENSE).
