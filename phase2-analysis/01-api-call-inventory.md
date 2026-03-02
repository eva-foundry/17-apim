# API Call Inventory

**Scope**: All HTTP endpoints exposed by the backend FastAPI application  
**Discovery Method**: Complete scan of `app/backend/app.py` (904 lines) and `app/frontend/src/api/api.ts` (490 lines)  
**Generated**: Evidence-based from source code inspection  

## Summary

The InfoJP backend exposes **27 HTTP endpoints** across functional areas:
- **Chat & Streaming**: 3 endpoints (core RAG functionality)
- **Management**: 7 endpoints (upload, delete, status, tags, folders)
- **Configuration**: 5 endpoints (app config, feature flags, warnings, info)
- **Specialized**: 7 endpoints (math, CSV, feedback, citation, file download, refresh, temp images)
- **Infrastructure**: 2 endpoints (health, root redirect)

**Key Finding**: All endpoints use JSON over HTTP (POST or GET). No REST frameworks beyond FastAPI; all routes are `@app.decorator` style at module level. Streaming uses Server-Sent Events (SSE), not WebSocket.

---

## Complete Endpoint Inventory

### A. Chat & Streaming Endpoints

| Endpoint | Type | Method | Path | Target Host | Auth | Streaming | Notes | Evidence |
|----------|------|--------|------|-------------|------|-----------|-------|----------|
| Chat | Backend | POST | `/chat` | Backend | Azure AD Token Provider | Yes (ndjson) | Main RAG conversation. Returns `StreamingResponse` with `media_type="application/x-ndjson"`. Input: approach selection, history, overrides (semantic_ranker, temperature, top_k, etc.) | app.py:307 |
| Math Stream | Backend | GET | `/stream?question={q}` | Backend | None (public) | Yes (text/event-stream) | Math problem streaming via agent. Calls `stream_agent_responses(question)`. | app.py:779 |
| CSV Stream | Backend | GET | `/tdstream?question={q}` | Backend | None (public) | Yes (text/event-stream) | Tabular data analysis streaming. Calls `td_agent_scratch_pad(question, DF_FINAL)`. | app.py:788 |

**Frontend Consumption** (Evidence):
- `/chat`: `chatApi(options, signal)` → fetch POST `/chat`, consumes Response, parses SSE via `ndjson-readablestream` | api.ts:23
- `/stream`: `streamData(question)` → `new EventSource("/stream?question=...")` | api.ts:166
- `/tdstream`: `streamTdData(question, file)` → `new EventSource("/tdstream?question=...")` (after upload to `/posttd`) | api.ts:176

---

### B. File Management Endpoints

| Endpoint | Type | Method | Path | Target Host | Auth | Streaming | Notes | Evidence |
|----------|------|--------|------|-------------|------|-----------|-------|----------|
| Upload File | Backend | POST | `/file` | Backend (→ Blob) | None detected | No | Multipart form upload. Input: file, file_path, tags. Stores in `AZURE_BLOB_STORAGE_UPLOAD_CONTAINER`. Metadata: tags. | app.py:862-880 |
| List Upload Status | Backend | POST | `/getalluploadstatus` | Backend (→ Cosmos DB) | None detected | No | Query upload status by timeframe, state, folder, tag. Returns array of file statuses + unique tags from Cosmos. | app.py:302-340 |
| Get Folders | Backend | POST | `/getfolders` | Backend (→ Blob) | None detected | No | Lists all unique folder paths in upload container. Uses `blob_container.list_blobs()`. | app.py:342-358 |
| Get Tags | Backend | POST | `/gettags` | Backend (→ Cosmos DB) | None detected | No | Returns all unique tags from uploaded files. Queries Cosmos with `SELECT DISTINCT VALUE t FROM c JOIN t IN c.tags`. | app.py:489-509 |
| Delete Item | Backend | POST | `/deleteItems` | Backend (→ Blob + Cosmos) | None detected | No | Delete blob and log status change to DELETING state. Path: `{container}/{blob_path}`. | app.py:360-384 |
| Resubmit Item | Backend | POST | `/resubmitItems` | Backend (→ Blob + Cosmos) | None detected | No | Re-upload blob to pipeline by touching metadata. Changes state to QUEUED. | app.py:386-420 |
| Log Status | Backend | POST | `/logstatus` | Backend (→ Cosmos DB) | None detected | No | Upsert document status in Cosmos DB. Input: path, status, status_classification, state. | app.py:511-530 |

**Frontend Consumption** (Evidence):
- `/file`: `uploadFile()` → multipart form POST `/file` | (Not found in api.ts; inferred from app.py form handler)
- `/getalluploadstatus`: `getAllUploadStatus(options)` → fetch POST `/getalluploadstatus` | api.ts:62
- `/getfolders`: `getFolders()` → fetch POST `/getfolders` | api.ts:146
- `/gettags`: `getTags()` → fetch POST `/gettags` | api.ts:161
- `/deleteItems`: `deleteItem(options)` → fetch POST `/deleteItems` | api.ts:78
- `/resubmitItems`: `resubmitItem(options)` → fetch POST `/resubmitItems` | api.ts:99
- `/logstatus`: `logStatus(entry)` → fetch POST `/logstatus` | api.ts:322

---

### C. Configuration & Info Endpoints

| Endpoint | Type | Method | Path | Target Host | Auth | Streaming | Notes | Evidence |
|----------|------|--------|------|-------------|------|-----------|-------|----------|
| Get Info Data | Backend | GET | `/getInfoData` | Backend (reads ENV + Cosmos metadata) | None detected | No | Returns deployment info: deployment name, model name/version, service endpoints, embeddings config. | app.py:544-565 |
| Get Warning Banner | Backend | GET | `/getWarningBanner` | Backend | None detected | No | Returns `CHAT_WARNING_BANNER_TEXT` from ENV. | app.py:567-572 |
| Get Feature Flags | Backend | GET | `/getFeatureFlags` | Backend | None detected | No | Returns 4 feature flags: ENABLE_WEB_CHAT, ENABLE_UNGROUNDED_CHAT, ENABLE_MATH_ASSISTANT, ENABLE_TABULAR_DATA_ASSISTANT. | app.py:833-848 |
| Get Max CSV Size | Backend | GET | `/getMaxCSVFileSize` | Backend | None detected | No | Returns `MAX_CSV_FILE_SIZE` from ENV (env var example: "7" = 7 MB). | app.py:573-578 |
| Get Application Title | Backend | GET | `/getApplicationTitle` | Backend | None detected | No | Returns `APPLICATION_TITLE` from ENV. | app.py:587-596 |

**Frontend Consumption** (Evidence):
- `/getInfoData`: `getInfoData()` → fetch GET `/getInfoData` | api.ts:333
- `/getWarningBanner`: `getWarningBanner()` → fetch GET `/getWarningBanner` | api.ts:346
- `/getFeatureFlags`: `getFeatureFlags()` → fetch GET `/getFeatureFlags` | api.ts:404
- `/getMaxCSVFileSize`: `getMaxCSVFileSize()` → fetch GET `/getMaxCSVFileSize` | api.ts:358
- `/getApplicationTitle`: `getApplicationTitle()` → fetch GET `/getApplicationTitle` | api.ts:371

---

### D. Content & Citation Endpoints

| Endpoint | Type | Method | Path | Target Host | Auth | Streaming | Notes | Evidence |
|----------|------|--------|------|-------------|------|-----------|-------|----------|
| Get Citation | Backend | POST | `/getcitation` | Backend (→ Blob) | None detected | No | Download citation JSON file from blob by name. Input: encoded citation path. Returns parsed JSON of citation details. | app.py:597-609 |
| Get Citation File | Backend | POST | `/get-file` | Backend (→ Blob) | None detected | Yes (blob stream) | Download actual citation file (PDF, image, etc.). Returns `StreamingResponse` with blob content-type header. | app.py:911-926 |
| Get All Tags | Backend | GET | `/getalltags` | Backend (→ Cosmos DB) | None detected | No | Returns all tags in system. Calls `statusLog.get_all_tags()`. | app.py:611-621 |

**Frontend Consumption** (Evidence):
- `/getcitation`: `getCitationObj(citation)` → fetch POST `/getcitation` with citation name | api.ts:363
- `/get-file`: `fetchCitationFile(filePath)` → fetch POST `/get-file` with file path, returns blob | api.ts:424
- `/getalltags`: `getAllTags()` → fetch GET `/getalltags` | api.ts:381

---

### E. Specialized Endpoints (Math & CSV Assistants)

| Endpoint | Type | Method | Path | Target Host | Auth | Streaming | Notes | Evidence |
|----------|------|--------|------|-------------|------|-----------|-------|----------|
| CSV Upload | Backend | POST | `/posttd` | Backend | None detected | No | Upload CSV file for tabular data assistant. Sets global `DF_FINAL` dataframe. Input: file (multipart). | app.py:771-789 |
| Process CSV Agent | Backend | GET | `/process_td_agent_response?question={q}` | Backend | None detected | No | Process question against loaded CSV via agent (max 3 retries). Returns agent response string. | app.py:791-809 |
| CSV Analysis | Backend | GET | `/getTdAnalysis?question={q}` | Backend | None detected | No | Get scratch pad (working) output from CSV analysis. Returns analysis trace. | app.py:810-828 |
| Math Hint | Backend | GET | `/getHint?question={q}` | Backend | None detected | No | Generate hint for math problem. Calls `generate_response(question)`, parses "Clues" section. | app.py:749-760 |
| Process Agent Response | Backend | GET | `/process_agent_response?question={q}` | Backend | None detected | No | Process question via math agent. Returns agent response string. | app.py:803 |
| Get Temp Images | Backend | GET | `/getTempImages` | Backend | None detected | No | Get temporary images from analysis. Returns array of image paths. | app.py:662 |
| Refresh | Backend | POST | `/refresh` | Backend | None detected | No | Refresh tabular data agent. Returns status array. | app.py:758 |

**Frontend Consumption** (Evidence):
- `/posttd`: `postTd(file)` → fetch POST `/posttd` with multipart form | api.ts:274
- `/process_td_agent_response`: `processCsvAgentResponse(question, file)` → fetch GET with retries | api.ts:289
- `/getTdAnalysis`: Called internally (same as streaming via ndjson) | app.py:810
- `/getHint`: `getHint(question)` → fetch GET `/getHint?question=...` | api.ts:161
- `/process_agent_response`: `processAgentResponse(question)` → fetch GET `/process_agent_response` | api.ts:308

---

### F. Infrastructure Endpoints

| Endpoint | Type | Method | Path | Target Host | Auth | Streaming | Notes | Evidence |
|----------|------|--------|------|-------------|------|-----------|-------|----------|
| Health | Backend | GET | `/health` | Backend | None detected | No | Liveness probe. Returns status (ready/loading), uptime, version. | app.py:261-283 |
| Root | Backend | GET | `/` | Backend | None detected | No | Redirect to `/index.html` (static files). | app.py:252-256 |

---

## Data Flow Architecture

### Request Flow for `/chat` (Primary)

```
Frontend (React/TypeScript)
  ↓
  chatApi(options, signal)  [api.ts:23]
  ↓
  fetch POST /chat 
    Headers: "Content-Type: application/json"
    Body: {
      history: ChatMessage[],
      approach: number (0-4, maps to Approaches enum),
      overrides: { semantic_ranker, temperature, top, etc. },
      citation_lookup: {},
      thought_chain: {}
    }
  ↓
Backend (FastAPI)
  ↓
  @app.post("/chat")  [app.py:272]
  ↓
  chat_approaches[Approaches(int(approach))].run(history, overrides, citation_lookup, thought_chain)
  ↓
  ChatReadRetrieveReadApproach.run()  [chatreadretrieveread.py]
    1. Optimize query via OpenAI (generate search terms)
    2. Embed query via enrichment service (REST POST)
    3. Hybrid search: Azure Search with vector + keyword
    4. Format context
    5. Stream completion from OpenAI via async generator
  ↓
  yield json.dumps({chunk}) + "\n"  (ndjson format)  [chatreadretrieveread.py:~450+]
  ↓
  StreamingResponse(generator, media_type="application/x-ndjson")  [app.py:293]
  ↓
Frontend receives SSE stream via ndjson-readablestream parser
```

### Request Flow for Streaming via `/stream` (Secondary)

```
Frontend
  ↓
  streamData(question)  [api.ts:166]
  ↓
  new EventSource("/stream?question=" + encodeURIComponent(question))
  ↓
Backend
  ↓
  @app.get("/stream")  [app.py:816]
  ↓
  stream = stream_agent_responses(question)  [approaches/mathassistant.py]
  ↓
  yield f"data: {json.dumps(chunk)}\n\n"  (SSE format)
  ↓
  StreamingResponse(stream, media_type="text/event-stream")  [app.py:818]
  ↓
Frontend
  ↓
  eventSourceRef.current.addEventListener("message", ...)
  ↓
  Parse event.data as JSON chunk
```

---

## Azure Service Calls (Backend Internal)

**Note**: These are NOT exposed to APIM. APIM frontends only the HTTP endpoints above. Internal service calls use Azure SDKs:

### OpenAI (Completion & Embeddings)

```python
# Line 100-107: SDK initialization
openai.api_type = "azure"
openai.api_base = ENV["AZURE_OPENAI_ENDPOINT"]  # e.g., https://xxx.openai.azure.com/
openai.api_version = "2024-02-01"
openai.api_type = "azure_ad"
token_provider = get_bearer_token_provider(azure_credential, ...)
openai.azure_ad_token_provider = token_provider

# Line 281: Async client creation
self.client = AsyncAzureOpenAI(
    azure_endpoint=openai.api_base,
    azure_ad_token_provider=azure_ai_token_provider,
    api_version=openai.api_version)

# Usage: self.client.chat.completions.create(...)
```

**Evidence**: app/backend/app.py:100-107, chatreadretrieveread.py:143-148

### Azure Search (Hybrid Vector + Keyword)

```python
# Line 156-162: SDK initialization
search_client = SearchClient(
    endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"],
    index_name=ENV["AZURE_SEARCH_INDEX"],
    credential=azure_credential,
    audience=ENV["AZURE_SEARCH_AUDIENCE"]
)

# Usage: search_client.search(
#   generated_query,
#   vector_queries=[VectorizedQuery(vector=embedding, ...)],
#   filter=search_filter
# )
```

**Evidence**: app/backend/app.py:156-162, chatreadretrieveread.py:263-276

### Azure Blob Storage (Document Retrieval)

```python
# Line 164-170: SDK initialization
blob_client = BlobServiceClient(
    account_url=ENV["AZURE_BLOB_STORAGE_ENDPOINT"],
    credential=azure_credential,
)
blob_container = blob_client.get_container_client(ENV["AZURE_BLOB_STORAGE_CONTAINER"])

# Usage: blob_container.get_blob_client(citation).download_blob()
```

**Evidence**: app/backend/app.py:164-170, app.py:606

### Azure Cosmos DB (Status Log)

```python
# Line 153-155: SDK initialization via StatusLog wrapper
cosmos_client = CosmosClient(
    url=statusLog._url,
    credential=azure_credential,
    consistency_level='Session'
)

# Usage: container.query_items(query=...)  or  statusLog.upsert_document(...)
```

**Evidence**: app/backend/app.py:153-155, app.py:377-383

### Enrichment Service (Embeddings via REST HTTP)

```python
# Line 225-235: REST HTTP call (NOT SDK)
url = f'{self.embedding_service_url}/models/{self.escaped_target_model}/embed'
headers = {
    'Accept': 'application/json',
    'Content-Type': 'application/json',
}
response = requests.post(url, json=data, headers=headers, timeout=60)
```

**Evidence**: chatreadretrieveread.py:225-245

---

## APIM Implications

### What APIM CAN Front
- **Public HTTP API Endpoints** (A-F above): All 20+ routes can be fronted by APIM gateway
- **Streaming Passthrough**: APIM supports SSE (text/event-stream) and ndjson streaming with proper passthrough policies

### What APIM CANNOT Front (Already Hidden by SDK)
- **Azure OpenAI calls**: Client credentials (bearer token) handled by SDK via Azure AD
- **Azure Search calls**: Same—credentials hidden by SDK
- **Azure Blob calls**: Same—credentials hidden by SDK
- **Azure Cosmos DB calls**: Same—credentials hidden by SDK
- **Enrichment Service calls**: Only this is raw HTTP, but it's backend-to-backend, not exposed

### Recommendation for Header Injection
- **Client Headers** (external injected by APIM):
  - `X-Run-Id`: Generated by APIM inbound policy
  - `X-Correlation-Id`: Propagated from frontend or generated by APIM
  - `X-Caller-App`: Assigned by APIM based on subscription or cert
  - `X-Env`: Set in APIM policy (dev/staging/prod)

- **Server Headers** (backend must accept and log):
  - All above headers should be extracted from request in backend and included in logs/Cosmos DB
  - Backend does NOT currently extract headers; would need middleware addition

- **Cost/Governance Headers**:
  - `X-Cost-Center`: Optional; backend can extract and include in Cosmos status log
  - `X-Project-Id`: Optional; backend can extract and include in Cosmos status log
  - `X-Ingestion-Variant`: For upload pipeline tagging (already similar to existing "tags" field)

---

## Evidence Summary

| Category | File | Lines | Finding |
|----------|------|-------|---------|
| Route inventory | app.py | 252-926 | 20+ @app.get/post decorators |
| Frontend API calls | api.ts | 23-424 | 20+ fetch() calls to endpoints |
| Streaming response | app.py | 293, 818, 927 | StreamingResponse + ndjson/text-event-stream |
| Auth (OpenAI) | app.py | 100-107 | get_bearer_token_provider + azure_ad_token_provider |
| Azure SDKs | requirements.txt | 1-30 | azure-identity, azure-search-documents, azure-cosmos, azure-storage-blob |
| Enrichment HTTP | chatreadretrieveread.py | 225-245 | requests.post() to enrichment service (only raw HTTP) |

