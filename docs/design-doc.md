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

## 6. API Specifications

### REST Endpoints

| Method   | Path                      | Description                           |
| -------- | ------------------------- | ------------------------------------- |
| `POST`   | `/photos`                 | Upload a photo (multipart)            |
| `GET`    | `/photos`                 | List photos (paginated, newest first) |
| `GET`    | `/photos/:id`             | Get photo metadata + URLs             |
| `DELETE` | `/photos/:id`             | Delete photo + all children           |
| `POST`   | `/photos/:id/annotations` | Create annotation                     |
| `GET`    | `/photos/:id/annotations` | List annotations for a photo          |
| `PUT`    | `/annotations/:id`        | Update annotation position/shape      |
| `DELETE` | `/annotations/:id`        | Remove annotation                     |
| `GET`    | `/photos/:id/threads`     | List threads for a photo              |
| `GET`    | `/annotations/:id/thread` | Get thread for an annotation          |
| `POST`   | `/threads/:id/messages`   | Post a message                        |
| `GET`    | `/threads/:id/messages`   | List messages (oldest first)          |
| `GET`    | `/me`                     | Current user profile                  |

### WebSocket Protocol

```
Connect:   WS /ws?photo_id=<id>&token=<jwt>
```

Server pushes:

```json
{"type": "new_annotation",   "data": { ...annotation }}
{"type": "annotation_updated", "data": { ...annotation }}
{"type": "annotation_deleted", "data": { "id": "..." }}
{"type": "new_message",      "data": { ...message }}
```

Client sends:

```json
{ "type": "ping" }
```

The WebSocket hub maintains a map of `photo_id → Set<Connection>`. When a new message or annotation is created via REST, the monolith pushes the event to all connected clients viewing that photo. No polling needed.

## 7. Scenarios

### Scenario 1: Scratch on a part (happy path)

1. **Upload**: Factory worker opens the photo viewer in-browser, uploads a JPEG of the scratched part via `POST /photos`. Monolith stores the binary in object store, inserts metadata row in PostgreSQL, starts async thumbnail generation.
2. **Annotate**: Worker clicks "Add annotation" on the photo, drags a circle around the scratch. The client sends `POST /photos/:id/annotations` with shape coordinates. Monolith inserts annotation and its associated thread (annotation_id set). The first message body is "Is this ok?".
3. **Real-time push**: The monolith pushes `new_annotation` and `new_message` events over WebSocket to all connected clients viewing that photo. An engineer who has the photo open sees the circle appear in real-time.
4. **Respond**: Engineer clicks the circle, sees the thread panel open with the question, types a reply. `POST /threads/:id/messages` stores the message and pushes `new_message` via WebSocket.
5. **Notify**: The factory worker sees the response appear instantly.

### Scenario 2: Photo-level discussion (no annotation)

1. An engineer opens a photo of a standard bracket and wants to ask procurement about pricing.
2. They click "Start discussion" (no circle needed). This creates a thread with `annotation_id = null` linked directly to the photo.
3. Procurement team members see the new thread appear in real-time and respond.

### Scenario 3: Concurrent annotations

1. Two engineers open the same photo simultaneously.
2. Engineer A circles the surface finish; Engineer B circles a hole dimension.
3. Both `POST /photos/:id/annotations` calls are processed independently. Each creates its own annotation + thread.
4. Both annotations are broadcast via WebSocket. Each engineer sees both circles appear regardless of who created them.
5. Threads are independent — discussion about surface finish and hole dimension do not interfere.

## 8. Future Considerations

- **Multi-tenancy**: add `tenant_id` to all tables, switch to presigned URL uploads
- **Horizontal scale**: extract WebSocket hub to Redis pub/sub, add more monolith instances behind a load balancer
- **Version history**: snapshot + delta approach for annotation edit history
- **ML features**: smart scratch detection, similar-part suggestions — introduce Python microservice
- **Staging environment**: add when second engineer joins
