I'll document the current data model assumptions, concrete entity schemas, partitioning/indexing guidance, and next implementation tasks so you can start working — I can scaffold these files next if you want.

1) Short plan (what I'll deliver if you ask me to continue)
- Produce a small set of JSON schema examples and Cosmos container definitions
- Add mapping from OpenAPI components → DB entities
- Scaffold DAL interfaces and simple integration tests
- Register these artifacts in `phase4-design` and update PLAN.md task list

2) Executive summary — goals for Phase 4 data model
- Support UI-driven CRUD for sessions, chat history, uploaded documents, tags, examples, user/group RBAC, and background processing status.
- Optimize for Azure Cosmos DB: choose partition keys to avoid hotspots, embed related small arrays (messages metadata) but keep large content (documents, embeddings) separate.
- Provide TTL for ephemeral session data, audit logs stored separately, and structured error/telemetry links for middleware.

3) Core entities (fields, purpose, and partition-key guidance)

- Session
  - Purpose: UI session metadata, session lifecycle and pointer to chat history
  - Key fields: `sessionId` (PK id), `userId`, `createdAt`, `updatedAt`, `status` (active|closed), `title`, `tags[]`, `lastMessagePreview`, `correlationId`
  - Partition key recommendation: `/userId` (high cardinality), or hierarchical: `/userId|sessionId` if using HPK pattern
  - Embedding policy: store small chat metadata in-session; store full chat turns in `ChatTurn` container if chat volume large

- ChatTurn (or Message)
  - Purpose: individual chat messages/assistant responses
  - Key fields: `messageId`, `sessionId`, `userId`, `role` (user/assistant/system), `text`, `createdAt`, `tokens`, `metadata` (source snippets), `referenceDocs[]`
  - Partition key: `/sessionId` (targeted queries by session) or `/userId` if you mainly query by user across sessions
  - Note: If message count per session is low (< few hundred) consider embedding as `messages[]` in `Session`. If unbounded, separate container.

- Document (uploaded file metadata)
  - Purpose: metadata for uploaded files and ingest pipeline state
  - Key fields: `docId`, `ownerId`, `filename`, `blobPath`, `sizeBytes`, `contentType`, `ingestStatus` (queued/processing/done/error), `ingestJobs[]`, `createdAt`, `tags[]`, `sha256`
  - Partition key: `/ownerId` or `/docId` depending query patterns; prefer `/ownerId` for user-scoped lists
  - Large binary stored in Blob Storage only; keep metadata in Cosmos

- IndexEntry (search index pointer)
  - Purpose: link document chunks to cognitive search/index and embeddings container
  - Key fields: `chunkId`, `docId`, `docChunkText` (small preview), `embeddingId`, `searchIndexName`, `vectorId`, `createdAt`
  - Partition key: `/docId` (chunk retrieval per document)

- Embedding (if storing externally)
  - Purpose: store embeddings metadata, not necessarily vectors (vectors may be kept in Cognitive Search or vector DB)
  - Key fields: `embeddingId`, `docChunkId`, `model`, `dim`, `storedIn` (Search|External), `createdAt`
  - Partition key: `/docChunkId`

- Tag / Folder / ExampleEntry
  - Purpose: UI-managed taxonomy and custom examples
  - Key fields: `id`, `ownerId`, `name`, `description`, `createdAt`, `visibility` (private|shared|global)
  - Partition key: `/ownerId` or `/visibility` for cross-tenant lists

- GroupMapping (RBAC)
  - Purpose: store group items used by RBAC check (supports LOCAL_DEBUG fallback)
  - Key fields: `groupId`, `groupName`, `availableResources[]`, `lastSyncedAt`
  - Partition key: `/groupId` (or `/tenantId` if multi-tenant)

- ProcessingJob / UploadStatus
  - Purpose: track background pipeline steps (OCR, chunking, embedding)
  - Key fields: `jobId`, `docId`, `type`, `status`, `startedAt`, `finishedAt`, `attempts`, `diagnosticLogRef`
  - Partition key: `/docId` or `/jobId`

- AuditLog
  - Purpose: immutable audit events for compliance (store minimal info; keep full details in blob if large)
  - Key fields: `auditId`, `resourceId`, `actorId`, `action`, `result`, `timestamp`, `correlationId`
  - Partition key: `/resourceId` or hash of `resourceId` to distribute

4) Query patterns and implications
- UI list sessions for a user → container partitioned by `userId` gives cheap single-partition query.
- Load chat history for session → partition by `sessionId` or store messages in Session to make single read.
- Search: text search and vector lookups served by Cognitive Search — store index pointers in `IndexEntry` to recover docs for display.
- Group/RBAC checks: infrequent reads but must be quick — keep mapping in-memory cache with TTL; store master mapping in `GroupMapping` container.
- Background jobs: frequent writes/updates — partition by `docId` so job status updates are targeted.

5) Embedding and document storage rules
- Do NOT store large vectors in Cosmos items (2 MB limit). Use Cognitive Search vector field or an external vector DB. Keep references (IDs) in Cosmos.
- Store chunk text previews trimmed to ~2–4 KB to help UI context preview without fetching full blob.

6) Consistency, TTL, and lifecycle
- Sessions: TTL (e.g., 7–30 days) if ephemeral; or keep permanent if required for audit.
- Audit logs: do not TTL; keep indefinitely or export to blob/archive.
- Processing jobs: TTL after completion + retention window (e.g., 90 days).

7) Indexing and RU considerations
- Add composite indexes for common queries (userId + updatedAt), and include `status` for filtered queries.
- Avoid cross-partition scans for hot queries (e.g., global lists) — use search service for global discovery.
- Use caching (in-memory TTL for group mappings and blob listing ETags) to reduce RU consumption.

8) RBAC & Middleware interactions
- JWT extraction middleware must filter only `typ == "groups"` claims (as implemented earlier).
- Middleware attaches `userId`, `groupIds`, `correlationId` to request context for DB writes.
- RBAC checks require quick lookups: maintain an in-memory cached set of permitted group IDs per resource mapped from `GroupMapping`.

9) Sample minimal JSON documents

- Session (embedded small messages example)
{
  "id": "sess_123",
  "userId": "user_abc",
  "title": "Claim review",
  "status": "active",
  "createdAt": "2026-02-06T10:00:00Z",
  "updatedAt": "2026-02-06T10:05:00Z",
  "messages": [
    {"messageId":"m1","role":"user","text":"What is my next step?","createdAt":"..."},
    {"messageId":"m2","role":"assistant","text":"Recommend ...","createdAt":"..."}
  ],
  "tags":["tax","appeal"],
  "correlationId":"corr-789"
}

- Document metadata
{
  "id": "doc_456",
  "ownerId": "user_abc",
  "filename": "evidence.pdf",
  "blobPath": "documents/user_abc/evidence.pdf",
  "sizeBytes": 345678,
  "ingestStatus": "done",
  "createdAt": "2026-02-05T12:00:00Z",
  "tags":["case-123"],
  "sha256":"..."
}

10) Migration & backwards-compatibility
- Create a migration plan to copy old files/records into new containers/partitions.
- Provide script that reads old `eva-jp-specific` JSON files and inserts into Cosmos with new schema mapping, preserving original `createdAt`.

11) Tests and acceptance criteria
- Unit tests for DAL (CRUD) and for partitioned queries.
- Integration test: create session → append messages → retrieve full session with messages and doc references.
- RBAC test: JWT claims filtered, group intersection logic returns correct allow/deny results.

12) Next actionable tasks I can execute now
- Create `phase4-design/DATA-MODEL-NOTE.md` with the note above (I can add JSON schema examples and container templates).
- Scaffold JSON Schema files for `Session`, `ChatTurn`, `Document`, `GroupMapping`.
- Scaffold simple DAL interface `app/backend/db/models.py` (pydantic models) and `app/backend/db/cosmos_client.py` with basic CRUD for `Session` and `Document`.
- Add three unit tests `tests/test_models.py`, `tests/test_rbac_mapping.py`, `tests/test_session_lifecycle.py`.

Tell me which of the next actionable tasks you want me to run and I’ll implement and push the files into `phase4-design` and backend per the Phase 4 plan.