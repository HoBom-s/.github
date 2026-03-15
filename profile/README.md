 # HoBom

  A polyglot microservices platform built with **Transactional Outbox Pattern**, **Hexagonal Architecture**, and **event-driven communication** via Kafka and gRPC.
  

  ## Architecture Overview

                      ┌─────────────────────┐
                      │      Client App      │                                             
                      └──────────┬──────────┘
                                 │
                                 ▼
                      ┌─────────────────────┐                                              
                      │      API Gateway     │
                      └──────────┬──────────┘
                                 │
            ┌────────────────────┼─────────────────────┐
            ▼                    ▼                      ▼
     ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐
     │  for-hobom-      │  │  hobom-space-    │  │  hobom-llm-      │
     │  backend         │  │  backend         │  │  service          │
     │  (NestJS)        │  │  (.NET 10)       │  │  (Python/FastAPI) │
     │  REST + gRPC     │  │  REST             │  │  REST + gRPC     │
     │  MongoDB         │  │  PostgreSQL       │  │  Gemini 2.5      │
     └────────┬─────────┘  └──────────────────┘  └──────────────────┘
              │
              │ outbox (PENDING)
              │
      gRPC poll (5s) │
              ▼
     ┌─────────────────┐
     │  hobom-event-   │
     │  processor      │        ┌──────────┐
     │  (Go)           │───────▶│  Kafka   │
     └────────┬────────┘        └─────┬────┘
              │                       │
     gRPC markAsSent                  ▼
              │              ┌──────────────────┐
              ▼              │  hobom-internal- │
      outbox → SENT          │  backend         │
                             │  (Kotlin/Spring) │
                             │  PostgreSQL       │
                             └──────────────────┘
                                      │
                             ┌────────┴────────┐
                             ▼                 ▼
                        MAIL_MESSAGE     PUSH_MESSAGE
                        (SMTP)           (Feign → backend)

  ## Services

  | Service | Stack | Port | Role |
  |---------|-------|------|------|
  | **for-hobom-backend** | NestJS 11 / TypeScript / MongoDB | 8080 (REST), 50051 (gRPC) | Client-facing API, outbox producer, notification store |
  | **hobom-event-processor** | Go / Gin / Redis | 8082 | Outbox polling → Kafka publish → gRPC callback |
  | **hobom-internal-backend** | Kotlin / Spring Boot / PostgreSQL | — | Kafka consumer, message strategy execution (mail, push), Notion integration |
  | **hobom-space-backend** | .NET 10 / ASP.NET Core / PostgreSQL | 8083 | Confluence-style document management (Space, Page, Comment, Version, Search) |
  | **hobom-llm-service** | Python / Google Gemini 2.5 Flash | 3000 (REST), gRPC | LLM-powered Q&A bot, study material generation, mock exam creation |
  | **hobom-buf-proto** | Protocol Buffers | — | Shared gRPC contract definitions (git submodule) |

  ## Key Design Decisions

  ### Transactional Outbox Pattern

  Business logic and event publishing share the same database transaction. This guarantees **at-least-once delivery** without distributed transactions.

```
  Business Operation + Outbox Insert  →  Single MongoDB Transaction
                                              │
                                      Event Processor polls (gRPC)
                                              │
                                      Kafka publish + markAsSent callback
```

  **Event Types**: `MESSAGE`, `HOBOM_LOG`, `LAW_CHANGED`
  **Lifecycle**: `PENDING` → `SENT` / `FAILED`

  ### Hexagonal Architecture

  All backend services follow Ports & Adapters:

  domain/
  ├── model/                # Entities, Value Objects, Repository interfaces
  └── ports/
      ├── in/               # Use case interfaces (input ports)
      └── out/              # External dependency contracts (output ports)
  application/              # Use case implementations
  adapters/
  ├── in/                   # REST controllers, gRPC handlers, DTOs
  └── out/                  # Persistence adapters, query adapters

  ### Internal vs External API Separation

  To prevent **infinite loops** when services call each other:

  | Path Pattern | Outbox Created | Usage |
  |---|---|---|
  | `/api/v1/...` | Yes | Client-facing endpoints |
  | `/internal/...` | No | Inter-service calls only |

  ## Tech Stack Summary

  | Concern | Technology |
  |---------|-----------|
  | Languages | TypeScript, Go, Kotlin, C# |
  | Databases | MongoDB (Atlas), PostgreSQL |
  | Messaging | Apache Kafka |
  | Caching / DLQ | Redis |
  | API Protocols | REST (OpenAPI/Swagger), gRPC (Protocol Buffers) |
  | Auth | JWT + Passport, API Key (internal) |
  | LLM | Google Gemini 2.5 Flash |
  | Web Scraping | Puppeteer + Chromium (law.go.kr) |
  | Observability | Pino (structured logging), Discord webhooks (5xx alerts), distributed tracing |
  | CI/CD | Jenkins (shared library), GitHub Actions (PR checks) |
  | Containerization | Docker (multi-stage Alpine builds) |
