# Phase 4 — Data Model Notes

Date: 2026-02-06

This document captures the Phase 4 data model decisions and the recommended container definitions, partitioning guidance, and query patterns to support the backend middleware and UI features.

Goals
- Support UI-driven CRUD for sessions, chat history, uploaded documents, tags/examples, RBAC group mappings, and background processing status.
- Optimize for Azure Cosmos DB: high-cardinality partition keys, embed small related arrays, keep large blobs in Azure Blob Storage or Cognitive Search.

Core Entities (summary)
- Session: metadata for UI sessions, store small message previews; partition by `/userId`.
- ChatTurn: message records when unbounded; partition by `/sessionId` or embed if messages per session are small.
- Document: uploaded file metadata; binary content stays in Blob Storage; partition by `/ownerId`.
- IndexEntry: mapping from document chunk → search/vector index; partition by `/docId`.
- Embedding: metadata reference only; vectors stored in Cognitive Search or external vector store.
- GroupMapping: RBAC mapping documents; partition by `/groupId` (cache in-memory with TTL).
- ProcessingJob: background pipeline job state; partition by `/docId`.
- AuditLog: immutable audit events; partition by `/resourceId` or hashed key for distribution.

Key Rules
- Do not store large vectors in Cosmos (2 MB item limit). Store vectors in cognitive search or vector DB; keep references in Cosmos.
- Use TTL for ephemeral data (sessions) and retention policies for audits (export/archive to blob if required).
- Add composite indexes for common query patterns (e.g., `userId + updatedAt`).

Migration
- Provide migration scripts to map old JSON files into new containers, preserving `createdAt` and `correlationId`.

Next artifacts to add
- JSON Schema files for each entity
- Cosmos container definitions (RU guidance, indexing policy examples)
- Pydantic models and a small DAL wrapper
- Unit tests for CRUD and RBAC intersection logic

Reference: See `phase4-design/COSMOS-CONTAINER-DEFS.json` (planned) and `I:\EVA-JP-v1.2\app\backend\db\models.py` for Pydantic model scaffolds.
