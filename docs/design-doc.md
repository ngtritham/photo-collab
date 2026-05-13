# PhotoCollab — System Design Document

## 1. Overview

PhotoCollab is a photo annotation and threaded messaging service for large manufacturing companies. Factory workers, mechanical engineers, and procurement teams resolve part-quality questions via annotated photos — a factory worker photographs a scratched part, circles the defect, asks "Is this ok?", and an engineer replies in a thread.

**Constraints**: single engineer, no code written, time pressure to prove viability. Readers: engineering leads and managers.

## 2. Architecture

A **modular monolith** deployed as one service with three bounded contexts (Photo, Annotation, Thread) as internal packages with strict module interfaces. Single deployable keeps iteration fast; contexts can be extracted to separate services later without rewriting interfaces.

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
│      └───────────┴───────────┘      │
│  ┌───────────────────────────────┐  │
│  │         Auth Middleware       │  │
│  └───────────────────────────────┘  │
│  ┌──────────────────┐ ┌──────────┐  │
│  │  WebSocket Hub   │ │Thumbnail │  │
│  │ photo_id→conns   │ │ Worker   │  │
│  └──────────────────┘ └──────────┘  │
└──────────┬──────────────────────────┘
           │
     ┌─────┴──────────┬──────────────┐
     ▼                ▼              ▼
┌──────────┐  ┌──────────────┐  ┌────────┐
│PostgreSQL│  │ Object Store │  │JWT IdP │
│(TypeORM) │  │ GCS / S3     │  │(Clerk) │
└──────────┘  └──────────────┘  └────────┘
  self-managed   fully managed    fully managed
```

- **Client** — Vite + React SPA; REST for CRUD, WebSocket for real-time annotation and thread events
- **Monolith** — TypeScript; handles business logic, auth verification, WebSocket fan-out, async thumbnail generation (sharp.js)
- **PostgreSQL** — relational data (photos, annotations, threads, messages, users); self-managed on VM or managed PG (Cloud SQL / RDS)
- **Object Store** — photo binaries and thumbnails; fully managed (GCS or S3)
- **JWT IdP** — external identity provider (Clerk or Auth0); monolith verifies tokens, extracts name and role claims

## 3. Technical Stack

| Layer            | Technology                              |
| ---------------- | --------------------------------------- |
| Frontend         | TypeScript, React, Vite, SVG overlays   |
| Backend          | TypeScript (Node.js)                    |
| API              | REST + WebSocket                        |
| ORM              | TypeORM                                 |
| Database         | PostgreSQL                              |
| Object Storage   | GCS / S3 (fully managed)                |
| Image Processing | sharp.js (in-process worker)            |
| Auth             | External IdP — JWT (Clerk / Auth0)      |

## 4. Data Model

```
user ────< photo          (uploaded_by)
user ────< annotation     (author_id)
user ────< message        (author_id)

photo ────< annotation    (photo_id)
photo ────< thread        (photo_id)

annotation? ──── thread   (annotation_id nullable)
thread ────< message      (thread_id)
```

**Key tables**

- **photo** — `id, filename, mime_type, object_key, width, height, uploaded_by, created_at`
- **annotation** — `id, photo_id, author_id, shape_type, x, y, width, height, radius, color, created_at`
- **thread** — `id, photo_id, annotation_id (nullable), created_at`
- **message** — `id, thread_id, author_id, body, created_at`
- **user** — `id, name, email, role (engineer | procurement | factory)`

Annotation coordinates stored as **ratios (0–1)** of image dimensions — annotations survive responsive layout without recalculation. Each annotation owns one thread; the first message carries the annotation text, avoiding duplication.

## 5. Design Decisions & Trade-offs

| Decision           | Choice                               | Trade-off                                                        |
| ------------------ | ------------------------------------ | ---------------------------------------------------------------- |
| Architecture style | Modular monolith                     | Simple deploy today; extraction cost when team grows             |
| Photo upload       | Through monolith (not presigned URL) | Simpler auth; monolith is bandwidth bottleneck at scale          |
| Thumbnails         | In-process async (sharp.js)          | No dedicated worker infra; competes for CPU with API             |
| Multi-tenancy      | None — single tenant, single DB      | Fastest path to value; migration needed when adding tenants      |
| Real-time          | WebSocket embedded in monolith       | No fan-out infra; horizontal scaling needs Redis pub/sub later   |
| Annotations        | Structured DB rows + SVG overlay     | Not pixel-flattened; enables edit history and search later       |
| Testing            | Integration smoke tests on critical paths | Coverage debt; justified by single-engineer velocity        |

## 6. Non-Functional Requirements

**Security**
- All traffic over TLS. JWT verified on every request via Auth Middleware; token passed as `Authorization: Bearer` (REST) and query param at WS handshake.
- Role claims (`engineer | procurement | factory`) enforced at module boundary — factory role cannot delete photos or annotations.
- Object store buckets private; photos served via short-lived signed URLs (15 min TTL) generated by monolith. No public bucket.
- Secrets (DB credentials, IdP client secret) via environment variables / secret manager — never in source.

**Performance**
- Target: p99 REST < 500 ms, WebSocket push < 100 ms after DB write.
- Photo upload cap: 20 MB. Thumbnail generation async — does not block upload response.
- Designed for ~50 concurrent users at launch. WebSocket hub in-memory; Redis pub/sub added before horizontal scale-out.

**DevOps**
- Monolith containerised (Docker). Single-region deploy: one VM or managed container service (Cloud Run / Fly.io) + managed PostgreSQL + object store bucket.
- CI via GitHub Actions: lint → type-check → test on every PR. Docker image built and pushed to registry on merge to main.
- One environment at launch (production). Staging added when second engineer joins.

**Testing**
- Unit tests on pure business logic (coordinate ratio conversion, role checks).
- Integration tests on three critical paths: photo upload → annotation → thread creation, WebSocket broadcast, and auth rejection on missing/invalid JWT.
- Manual smoke test on every deploy until CI coverage warrants automation.

**Monitoring & Observability**
- Structured JSON logs (request ID, user ID, duration, status) on every HTTP and WS event.
- Error rate and p99 latency dashboards (Cloud Monitoring or Datadog). Alert on sustained 5xx rate > 1% or p99 > 2 s.
- WebSocket active-connection count as leading indicator of adoption.
- Object store and DB storage metrics tracked to anticipate capacity needs.

## 7. Future Considerations

- **Multi-tenancy** — add `tenant_id` to all tables; switch uploads to presigned URLs
- **Horizontal scale** — extract WebSocket hub to Redis pub/sub; add load balancer
- **Offline support** — service worker queues photos + annotations locally, syncs on reconnect (factory floor wifi unreliable)
- **ML defect detection** — vision model proposes annotation circle; Python microservice alongside monolith
