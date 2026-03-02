# Next Actions: File Inspection Priority

**Date**: 2026-01-25  
**Purpose**: Evidence-first inspection order for APIM integration planning  
**Method**: Read files in priority order to extract API contract details

---

## Top 10 Files to Inspect (In Order)

### 1. `app/backend/app.py` (904 lines)
**Why**: Contains **all backend API routes** - this is the APIM surface  
**What to extract**:
- Complete list of `@app.get/@app.post/@app.put/@app.delete` routes
- Path parameters, query parameters, request body schemas
- Response types and streaming vs. non-streaming endpoints
- Middleware and CORS configuration
- Auth guards or token validation (if any)

**Evidence needed**:
```python
# Extract pattern: @app.{method}("{path}")
# Line numbers: Routes start at line ~248, continue to ~888
```

**Action**:
```bash
rg -n '@app\.(get|post|put|delete)\("([^"]+)"' app/backend/app.py -o | \
  awk -F: '{print $2}' | sort -u > apim-scan-results/endpoint-paths.txt
```

---

### 2. `app/frontend/src/api/api.ts` (490 lines)
**Why**: Contains **all frontend → backend API calls** - validates backend routes  
**What to extract**:
- Function name → endpoint mapping
- HTTP method + path
- Request body structure
- Response parsing (streaming vs. JSON)
- Any custom headers sent

**Evidence needed**:
```typescript
// Pattern: export async function {name}(...) { fetch("{path}") }
// Lines: 23-490
```

**Action**:
```bash
rg -n 'export async function|await fetch\(' app/frontend/src/api/api.ts
```

---

### 3. `app/backend/approaches/chatreadretrieveread.py` (504 lines)
**Why**: Primary RAG approach - shows **backend → Azure service calls**  
**What to extract**:
- Azure SDK client usage (SearchClient, BlobServiceClient, OpenAI)
- Enrichment service HTTP calls (if any)
- Query transformation logic
- Streaming implementation pattern

**Evidence needed**:
```python
# Lines 1-50: Imports and SDK setup
# Lines 50-200: run() method with Azure service calls
# Lines 200-504: Helper methods
```

**Action**:
```bash
rg -n 'search_client\.|openai\.|blob_client\.|requests\.' app/backend/approaches/chatreadretrieveread.py
```

---

### 4. `app/frontend/vite.config.ts` (40 lines)
**Why**: Shows **dev proxy config** - identifies API base URL injection point  
**What to extract**:
- Proxy rules (`/ask`, `/chat` → `http://localhost:5000`)
- Build output directory
- Any env var references

**Evidence already gathered**:
```typescript
server: {
    proxy: {
        "/ask": "http://localhost:5000",
        "/chat": "http://localhost:5000"
    }
}
```

**Implication**: In production, change frontend to call APIM gateway URL

---

### 5. `app/backend/core/` (if exists)
**Why**: Likely contains **shared utilities** like auth middleware, logging, config  
**What to extract**:
- Auth guards or token validation
- Request ID / correlation ID generation
- Logging middleware
- Config loading utilities

**Action**:
```bash
# Check if core/ folder exists
ls app/backend/core/

# If exists, inspect
rg -n 'middleware|auth|token|request_id' app/backend/core/ --type python
```

---

### 6. `app/backend/requirements.txt` (30 lines)
**Why**: Confirms **SDK versions** and **no raw HTTP libraries** (except for enrichment)  
**Evidence already gathered**:
- `fastapi==0.115.0`
- `openai==1.55.3`
- `azure-search-documents==11.5.1`
- `azure-storage-blob==12.23.0`
- `azure-cosmos==4.7.0`
- `azure-identity==1.18.0`

**No raw HTTP libs** like `httpx` or `aiohttp` (except `requests` for enrichment)

---

### 7. `functions/TextEnrichment/__init__.py`
**Why**: Shows **worker → Azure service calls** (indexing pipeline)  
**What to extract**:
- Azure Search indexing API calls
- Queue message handling
- Blob download/upload patterns
- Error handling and retry logic

**Action**:
```bash
rg -n 'search_client\.|queue_client\.|blob_client\.' functions/TextEnrichment/__init__.py
```

---

### 8. `app/frontend/src/pages/Chat.tsx` (or main chat component)
**Why**: Shows **how API calls are invoked** from UI  
**What to extract**:
- `chatApi()` invocation pattern
- Abort signal usage (for cancellation)
- SSE parsing with `ndjson-readablestream`
- Error handling

**Action**:
```bash
rg -n 'chatApi|fetch|EventSource' app/frontend/src/pages/ --type typescript
```

---

### 9. `.env.template` or `backend.env` (if exists)
**Why**: Documents **required env vars** for APIM gateway config  
**What to extract**:
- `AZURE_OPENAI_ENDPOINT`
- `AZURE_SEARCH_SERVICE_ENDPOINT`
- `AZURE_BLOB_STORAGE_ENDPOINT`
- `COSMOSDB_URL`
- `ENRICHMENT_APPSERVICE_URL`
- `ALLOWED_ORIGINS` (for CORS)

**Action**:
```bash
find . -name "*.env*" -o -name "*.example" -o -name "backend.env"
cat .env.template | grep AZURE
```

---

### 10. `app/backend/approaches/approach.py` (base class)
**Why**: Shows **approach interface** - what all approaches must implement  
**What to extract**:
- `run()` method signature
- Shared utilities (token counting, logging)
- Common Azure SDK initialization patterns

**Action**:
```bash
rg -n 'class Approach|def run\(' app/backend/approaches/approach.py --type python -A 5
```

---

## Inspection Workflow

### Phase 1: Route Inventory (Files 1-2)
**Goal**: Complete mapping of frontend ↔ backend API surface

1. Extract all `@app.{method}("{path}")` from `app.py` → Save to `endpoint-inventory.md`
2. Extract all `fetch("{path}")` from `api.ts` → Map to backend routes
3. Identify mismatches or undocumented endpoints
4. Classify by APIM operation group (chat, upload, status, admin, etc.)

**Output**: Table with columns:
| Frontend Function | Backend Route | Method | Request Body | Response Type | Streaming? |

---

### Phase 2: Azure SDK Usage (Files 3, 7, 10)
**Goal**: Confirm **no HTTP interception needed** for Azure services

1. List all Azure SDK client instantiations in `app.py`
2. Trace SDK usage in approaches (SearchClient, BlobServiceClient, etc.)
3. Verify workers use same SDK pattern
4. Document the **one exception**: Backend → Enrichment HTTP call

**Output**: Confirmation statement:
> "All Azure service access uses typed SDKs. APIM fronts only the backend API (`/chat`, `/upload`, etc.), not downstream Azure services."

---

### Phase 3: Streaming Analysis (Files 1-3, 8)
**Goal**: Design APIM policy for SSE passthrough

1. Document backend `StreamingResponse` usage (`/chat` endpoint)
2. Document frontend `ndjson-readablestream` parsing
3. Test APIM chunked transfer encoding support
4. Design policy to preserve `Transfer-Encoding: chunked`

**Output**: Streaming policy snippet for APIM

---

### Phase 4: Auth & Headers (Files 1, 5, 9)
**Goal**: Identify where to inject cost/governance headers

1. Document current auth pattern (Azure AD token validation?)
2. Identify request context construction points (where headers are assembled)
3. Map env vars to APIM named values
4. Design header propagation plan (X-Run-Id, X-User-Id, etc.)

**Output**: Header contract draft (see `05-header-contract-draft.md`)

---

## Evidence Extraction Template

For each file inspected, document:

```markdown
## File: {path} (lines X-Y)

**Purpose**: {what this file does}

**Key Findings**:
- Finding 1 (line X): {code snippet}
- Finding 2 (line Y): {code snippet}

**APIM Implications**:
- Implication 1
- Implication 2

**Evidence**:
```{language}
{code snippet with line numbers}
```
```

---

## Quick Inspection Commands

### Get File Line Counts
```bash
wc -l app/backend/app.py app/frontend/src/api/api.ts app/backend/approaches/*.py
```

### Preview File Structure
```bash
head -100 app/backend/app.py | rg -n 'import|class|def|@app\.'
```

### Extract Function Signatures
```bash
rg -n '^(async )?def [a-z_]+\(' app/backend/app.py
```

### Extract Route Decorators with Paths
```bash
rg -n '@app\.(get|post|put|delete)\(' app/backend/app.py -A 1
```

---

## After Inspection: Deliverable Checklist

- [ ] `01-api-call-inventory.md` (complete frontend ↔ backend mapping)
- [ ] `02-auth-and-identity.md` (auth flow + middleware)
- [ ] `03-config-and-base-urls.md` (env vars + APIM named values)
- [ ] `04-streaming-analysis.md` (SSE implementation + APIM policy)
- [ ] `05-header-contract-draft.md` (cost/governance headers)

---

**End of File Inspection Priority Guide**
