# Entity-Relationship Diagram

```mermaid
erDiagram
    user {
        uuid id PK
        string name
        string email UK
        enum role "engineer | procurement | factory"
    }

    photo {
        uuid id PK
        string filename
        string mime_type
        string object_key
        int width
        int height
        uuid uploaded_by FK
        timestamp created_at
    }

    annotation {
        uuid id PK
        uuid photo_id FK
        uuid author_id FK
        enum shape_type "circle | rectangle"
        float x "0-1 ratio"
        float y "0-1 ratio"
        float width "0-1 ratio, rectangle only"
        float height "0-1 ratio, rectangle only"
        float radius "0-1 ratio, circle only"
        string color "hex"
        timestamp created_at
    }

    thread {
        uuid id PK
        uuid photo_id FK
        uuid annotation_id FK "nullable"
        timestamp created_at
    }

    message {
        uuid id PK
        uuid thread_id FK
        uuid author_id FK
        text body
        timestamp created_at
    }

    user ||--o{ photo : "uploaded_by"
    user ||--o{ annotation : "author_id"
    user ||--o{ message : "author_id"

    photo ||--o{ annotation : "photo_id"
    photo ||--o{ thread : "photo_id"

    annotation ||--o| thread : "annotation_id (optional)"

    thread ||--o{ message : "thread_id"
```

## Relationship Summary

| Entity A | Relationship | Entity B | Cardinality | Via |
|---|---|---|---|---|
| `user` | uploads | `photo` | one-to-many | `photo.uploaded_by` |
| `user` | authors | `annotation` | one-to-many | `annotation.author_id` |
| `user` | authors | `message` | one-to-many | `message.author_id` |
| `photo` | contains | `annotation` | one-to-many | `annotation.photo_id` |
| `photo` | has | `thread` | one-to-many | `thread.photo_id` |
| `annotation` | spawns | `thread` | one-to-zero-or-one | `thread.annotation_id` (nullable) |
| `thread` | contains | `message` | one-to-many | `message.thread_id` |

## Key design notes

- **Coordinates as ratios (0–1)**: `x`, `y`, `width`, `height`, `radius` are fractions of image dimensions so annotations survive responsive/retina layouts
- **Annotation → Thread is optional 1:1**: an annotation always spawns a thread (with the annotation text as the first message body), but a thread can exist without an annotation (photo-level discussion)
- **Cascade deletes**: deleting a photo cascades to its annotations, threads (annotation-level + photo-level), and messages via foreign keys
