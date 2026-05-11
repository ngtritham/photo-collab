# PhotoCollab — System Design Document

## 1. Overview

PhotoCollab is a photo annotation and threaded messaging service for large manufacturing companies. Factory workers, mechanical engineers, and procurement teams communicate via annotated photos — a factory worker photographs a scratched part, circles the defect with a question, and an engineer responds in a thread.

**Constraints**: single engineer, no code written, time pressure to show viability. Readers: engineering leads and managers.

## 2. Architecture

A **modular monolith** deployed as one service. The monolith contains three bounded contexts (Photo, Annotation, Thread) as internal packages with strict module boundaries. A single deployable keeps iteration fast; extracting modules into separate services later is realistic because the interfaces between contexts are already clean.

```
┌─────────────┐     ┌─────────────┐
│  Vite+React  │────▶│  WebSocket  │
│    Client    │     │  (ws://)    │
└──────┬───────┘     └──────┬──────┘
       │ REST (HTTP)        │
       ▼                    ▼
┌──────────────────────────────────────┐
│         Modular Monolith             │
│  ┌────────┐ ┌──────────┐ ┌────────┐ │
│  │ Photo  │ │Annotation│ │ Thread │ │
│  │Module  │ │ Module   │ │ Module │ │
│  └───┬────┘ └────┬─────┘ └───┬────┘ │
│      │           │           │      │
│  ┌───┴───────────┴───────────┴───┐  │
│  │         Auth Middleware       │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │      WebSocket Hub           │  │
│  │  (photo_id → connections)    │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │   Thumbnail Worker (sharp.js)│  │
│  └───────────────────────────────┘  │
└──────────┬──────────────────────────┘
           │
     ┌─────┴─────────┬──────────────┐
     ▼               ▼              ▼
┌──────────┐  ┌──────────────┐  ┌──────┐
│PostgreSQL│  │ Object Store │  │ JWT  │
│(TypeORM) │  │   (photos)   │  │ IdP  │
└──────────┘  └──────────────┘  └──────┘
```

- **Client** (Vite + React SPA) communicates via REST for CRUD and WebSocket for real-time thread updates
- **Monolith** handles all business logic, auth verification, and WebSocket connection management
- **PostgreSQL** stores all relational data (photos metadata, annotations, threads, messages, users)
- **Object Store** stores photo binaries and generated thumbnails
- **JWT IdP** handles authentication; the monolith verifies tokens and extracts user claims (name, role)

## 3. Technical Stack

| Layer            | Technology                              |
| ---------------- | --------------------------------------- |
| Frontend         | TypeScript, React, Vite, SVG (overlays) |
| Backend          | TypeScript                              |
| API              | REST + WebSocket                        |
| ORM              | TypeORM                                 |
| Database         | PostgreSQL                              |
| Object Storage   | Cloud object store (GCS/S3)             |
| Image Processing | sharp.js                                |
| Auth             | External IdP (JWT)                      |

## 4. Database ER Diagram

```
user ────< photo          (uploaded_by)
user ────< annotation     (author_id)
user ────< message        (author_id)

photo ────< annotation    (photo_id)
photo ────< thread        (photo_id)

annotation? ──── thread  (annotation_id nullable)
thread ────< message      (thread_id)
```

### Key tables

**photo** — id, filename, mime_type, object_key, width, height, uploaded_by, created_at
**annotation** — id, photo_id, author_id, shape_type, x, y, width, height, radius, color, created_at
**thread** — id, photo_id, annotation_id (nullable), created_at
**message** — id, thread_id, author_id, body, created_at
**user** — id, name, email, role (engineer/procurement/factory)

Coordinates stored as **ratios (0–1)** of image dimensions so annotations survive responsive layouts. Each annotation gets its own thread; the first message body contains the annotation text, avoiding data duplication.

## 5. Key Design Decisions & Trade-offs

| Decision           | Choice                                    | Trade-off                                                        |
| ------------------ | ----------------------------------------- | ---------------------------------------------------------------- |
| Architecture style | Modular monolith                          | Simple deploy today; extraction cost when team grows             |
| Photo upload       | Through monolith (not presigned URL)      | Simpler auth, but monolith becomes bandwidth bottleneck at scale |
| Thumbnails         | In-process async (sharp.js)               | No dedicated worker infra; competes for CPU with API             |
| Multi-tenancy      | None (single tenant, single DB)           | Fastest path to value; migration needed if adding tenants        |
| Real-time          | WebSocket embedded in monolith            | No fan-out infra; horizontal scaling needs Redis pub/sub later   |
| Annotations        | Structured data in DB + SVG overlay       | Not pixel-flattened; enables edit history and search later       |
| Testing            | Integration smoke tests on critical paths | Coverage debt; justified by single-engineer velocity             |

## 6. Future Considerations

- **Multi-tenancy**: add `tenant_id` to all tables, switch to presigned URL uploads
- **Horizontal scale**: extract WebSocket hub to Redis pub/sub, add more monolith instances behind a load balancer
- **Version history**: snapshot + delta approach for annotation edit history
- **ML features**: smart scratch detection, similar-part suggestions — introduce Python microservice
- **Staging environment**: add when second engineer joins
