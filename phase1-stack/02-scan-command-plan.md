# Repo Scan Command Plan (ripgrep)

**Date**: 2026-01-25  
**Repo Root**: `I:\EVA-JP-v1.2\docs\eva-foundation\projects\11-MS-InfoJP\base-platform`  
**Tool**: ripgrep (`rg`) for fast code search  
**Purpose**: Produce evidence-based API call inventory for APIM integration

---

## Prerequisites

```powershell
# Ensure ripgrep is installed
winget install BurntSushi.ripgrep.MSVC
# or
choco install ripgrep

# Navigate to repo root
cd "I:\EVA-JP-v1.2\docs\eva-foundation\projects\11-MS-InfoJP\base-platform"
```

---

## A. Frontend → Backend API Calls

### A1. All fetch() Calls
```bash
rg -n "fetch\(" app/frontend/src/ --type typescript
```
**Expected**: ~20-30 matches in `api/api.ts`, possibly page components  
**Purpose**: Identify all outbound HTTP calls from frontend

### A2. EventSource (SSE) Usage
```bash
rg -n "new EventSource\(" app/frontend/src/ --type typescript
```
**Expected**: Matches in `api/api.ts` for streaming endpoints  
**Purpose**: Confirm SSE streaming pattern

### A3. API Base URL Configuration
```bash
rg -n "API_BASE|baseURL|process\.env|import\.meta\.env" app/frontend/src/ --type typescript
```
**Expected**: Env var usage or hardcoded base URL  
**Purpose**: Identify where to inject APIM gateway URL

### A4. All API Method Calls (from api.ts)
```bash
rg -n "export async function" app/frontend/src/api/api.ts
```
**Expected**: ~30+ exported API functions  
**Purpose**: Complete frontend API surface inventory

---

## B. Backend API Route Definitions

### B1. All FastAPI Routes
```bash
rg -n "@app\.(get|post|put|delete|patch)" app/backend/app.py
```
**Expected**: 20-30 route decorators  
**Purpose**: Complete backend API endpoint inventory

### B2. Approach-Specific Routes (if any)
```bash
rg -n "@(app|router)\.(get|post)" app/backend/approaches/ --type python
```
**Expected**: Likely none (routes in `app.py` only)  
**Purpose**: Verify no hidden routes in approach classes

### B3. Route Path Extraction
```bash
rg '@app\.(get|post|put|delete|patch)\("([^"]+)"' app/backend/app.py -o
```
**Expected**: Clean list of endpoint paths  
**Purpose**: Generate OpenAPI path list

---

## C. Backend → Downstream HTTP Calls

### C1. Azure OpenAI SDK Calls
```bash
rg -n "openai\.|AsyncAzureOpenAI|ChatCompletion" app/backend/ --type python
```
**Expected**: Matches in `approaches/*.py` and `app.py`  
**Purpose**: Confirm SDK usage (not raw HTTP)

### C2. Azure Search SDK Calls
```bash
rg -n "SearchClient|search_client\.search" app/backend/ --type python
```
**Expected**: Matches in `app.py` init and `approaches/*.py`  
**Purpose**: Verify SDK-based search (not REST API)

### C3. Azure Blob Storage SDK Calls
```bash
rg -n "BlobServiceClient|blob_client\.|download_blob|upload_blob" app/backend/ --type python
```
**Expected**: Matches in `app.py` and approaches  
**Purpose**: Verify SDK-based storage access

### C4. Azure Cosmos DB SDK Calls
```bash
rg -n "CosmosClient|cosmos_client\.|upsert_document|query_items" app/backend/ --type python
```
**Expected**: Matches in `app.py` and StatusLog usage  
**Purpose**: Verify SDK-based Cosmos access

### C5. Enrichment Service HTTP Calls
```bash
rg -n "ENRICHMENT_APPSERVICE_URL|requests\.(get|post)" app/backend/ --type python
```
**Expected**: Matches in approaches that call enrichment  
**Purpose**: Identify backend → enrichment HTTP calls (rare HTTP usage)

### C6. External HTTP Dependencies
```bash
rg -n "http(s)?://|requests\.|httpx\.|urllib\.request" app/backend/ --type python --glob "!venv/*" --glob "!.venv/*"
```
**Expected**: Minimal matches (mostly SDK usage)  
**Purpose**: Catch any raw HTTP calls

---

## D. Azure Functions (Workers) API Calls

### D1. Function Triggers and Bindings
```bash
rg -n "def main\(|@app\.route|@app\.function_name" functions/ --type python
```
**Expected**: Matches in each function's `__init__.py`  
**Purpose**: Identify worker entry points

### D2. Azure SDK Usage in Functions
```bash
rg -n "from azure\.|import azure\." functions/ --type python
```
**Expected**: Many matches (Form Recognizer, Search, Blob, Queue)  
**Purpose**: Confirm SDK-based access in workers

### D3. Queue and Blob Triggers
```bash
rg -n "QueueTrigger|BlobTrigger|TimerTrigger" functions/ --type python
```
**Expected**: Matches in `function.json` or decorators  
**Purpose**: Understand worker pipeline flow

---

## E. Streaming Implementation Details

### E1. Backend Streaming Response
```bash
rg -n "StreamingResponse|async def.*yield" app/backend/ --type python
```
**Expected**: Matches in `app.py` `/chat` endpoint and approaches  
**Purpose**: Confirm SSE streaming pattern

### E2. Frontend SSE Parsing
```bash
rg -n "ndjson|EventSource|text/event-stream" app/frontend/src/ --type typescript
```
**Expected**: Matches in `api/api.ts`  
**Purpose**: Confirm client-side SSE handling

### E3. Chunked Transfer Encoding
```bash
rg -n "Transfer-Encoding|chunked|stream" app/backend/ --type python
```
**Expected**: Implicit in FastAPI StreamingResponse  
**Purpose**: Verify APIM must support chunked responses

---

## F. Configuration & Environment Variables

### F1. Backend Env Var Loading
```bash
rg -n "os\.environ|os\.getenv|ENV\[" app/backend/app.py --type python -A 1
```
**Expected**: 50+ matches (all Azure service config)  
**Purpose**: Identify required env vars for APIM routing

### F2. Azure Service Endpoints
```bash
rg -n "AZURE_OPENAI_ENDPOINT|AZURE_SEARCH_.*ENDPOINT|AZURE_BLOB_STORAGE_ENDPOINT|COSMOSDB_URL|ENRICHMENT_APPSERVICE_URL" app/backend/ --type python
```
**Expected**: Matches in `app.py` and approaches  
**Purpose**: Map downstream service endpoints

### F3. Frontend Env Var Usage
```bash
rg -n "process\.env|import\.meta\.env" app/frontend/src/ --type typescript
```
**Expected**: API base URL and feature flags  
**Purpose**: Identify frontend config injection points

### F4. .env File Templates
```bash
rg -n "AZURE_|OPENAI|SEARCH|BLOB|COSMOS" --glob "*.env*" --glob "*.example" .
```
**Expected**: Matches in `.env.template` or similar  
**Purpose**: Document required env vars

---

## G. Authentication & Identity

### G1. Azure Identity Usage
```bash
rg -n "DefaultAzureCredential|ManagedIdentityCredential|get_bearer_token_provider" app/backend/ --type python
```
**Expected**: Matches in `app.py` credential initialization  
**Purpose**: Verify token-based auth pattern

### G2. JWT/Token Validation
```bash
rg -n "jwt|JWT|token|@require|@authenticate" app/backend/ --type python
```
**Expected**: Possible matches in auth middleware  
**Purpose**: Identify existing auth guards

### G3. Auth Middleware/Guards
```bash
rg -n "middleware|Depends\(|@require|verify_token" app/backend/ --type python
```
**Expected**: Possible matches in routers or decorators  
**Purpose**: Understand auth enforcement points

---

## H. Special Searches for APIM Planning

### H1. CORS Configuration
```bash
rg -n "CORS|cors|Access-Control-Allow" app/backend/ --type python
```
**Expected**: CORS middleware setup in `app.py`  
**Purpose**: Understand current origin restrictions

### H2. Rate Limiting (if any)
```bash
rg -n "rate_limit|RateLimit|throttle|Throttle" app/backend/ --type python
```
**Expected**: Likely none (APIM will handle)  
**Purpose**: Check if rate limiting exists

### H3. Request ID/Correlation ID
```bash
rg -n "request_id|correlation_id|x-request-id|x-correlation-id" app/backend/ --type python -i
```
**Expected**: Possible matches in logging  
**Purpose**: Identify existing header contract

### H4. Custom Headers
```bash
rg -n "headers\[|request\.headers|\.get\(\"X-|\.get\(\"x-" app/backend/ --type python
```
**Expected**: Matches where headers are read/set  
**Purpose**: Document existing header usage

---

## I. Azure-Specific Dependencies

### I1. Azure OpenAI References
```bash
rg -n "openai\.azure|azure\.ai\.openai|chat\.completions|embeddings" app/backend/ --type python
```
**Expected**: Widespread in approaches  
**Purpose**: Confirm Azure OpenAI SDK pattern

### I2. Azure Cognitive Search
```bash
rg -n "azure\.search|search\.windows\.net|VectorizedQuery" app/backend/ --type python
```
**Expected**: Matches in approaches and `app.py`  
**Purpose**: Verify search SDK usage

### I3. Azure Blob Storage
```bash
rg -n "blob\.core\.windows\.net|BlobServiceClient|azure\.storage\.blob" app/backend/ --type python
```
**Expected**: Matches in `app.py` and approaches  
**Purpose**: Verify blob SDK usage

### I4. Azure Cosmos DB
```bash
rg -n "cosmos|CosmosClient|documents\.azure\.com" app/backend/ --type python
```
**Expected**: Matches in `app.py` and StatusLog  
**Purpose**: Verify Cosmos SDK usage

### I5. Azure Key Vault (if used)
```bash
rg -n "KeyVaultSecret|SecretClient|vault\.azure\.net" app/backend/ --type python
```
**Expected**: Possible matches in credential loading  
**Purpose**: Check if secrets fetched at runtime

---

## J. Output Redirection & Analysis

### Save All Results
```powershell
# Create output directory
mkdir -p apim-scan-results

# Run all scans and save
rg -n "@app\.(get|post)" app/backend/app.py > apim-scan-results/backend-routes.txt
rg -n "fetch\(" app/frontend/src/ --type typescript > apim-scan-results/frontend-fetch.txt
rg -n "AZURE_.*ENDPOINT|.*_URL" app/backend/ --type python > apim-scan-results/azure-endpoints.txt
rg -n "StreamingResponse|EventSource" app/ --type python --type typescript > apim-scan-results/streaming.txt
```

### Analyze Results
```powershell
# Count routes
(Get-Content apim-scan-results/backend-routes.txt | Measure-Object -Line).Lines

# Extract unique endpoints
rg '@app\.(get|post|put|delete)\("([^"]+)"' app/backend/app.py -o | Sort-Object -Unique
```

---

## K. Quick One-Liner for Complete Scan

```bash
# Full backend route inventory with line numbers
rg -n '@app\.(get|post|put|delete|patch)\("([^"]+)"' app/backend/app.py --color=always | tee apim-scan-results/routes-full.txt

# Full frontend API calls
rg -n 'await fetch\(|new EventSource\(' app/frontend/src/api/api.ts --color=always | tee apim-scan-results/api-calls-full.txt

# All Azure service references
rg -n 'azure\.(search|storage|cosmos|identity|openai|ai)' app/backend/ --type python | tee apim-scan-results/azure-sdks.txt
```

---

## L. Expected Output Summary

| Scan Target | Command | Expected Matches | Output File |
|-------------|---------|------------------|-------------|
| Backend routes | `rg -n "@app\.(get\|post)" app/backend/app.py` | ~25-30 | `backend-routes.txt` |
| Frontend API calls | `rg -n "fetch\(" app/frontend/src/` | ~30-40 | `frontend-fetch.txt` |
| Azure endpoints | `rg -n "AZURE_.*ENDPOINT"` | ~10-15 | `azure-endpoints.txt` |
| Streaming code | `rg -n "StreamingResponse"` | ~5-10 | `streaming.txt` |
| Auth code | `rg -n "DefaultAzureCredential"` | ~5-10 | `auth.txt` |
| Enrichment calls | `rg -n "ENRICHMENT_"` | ~3-5 | `enrichment.txt` |

---

## M. Next Steps After Running Scans

1. **Parse route output** → Generate OpenAPI spec skeleton
2. **Map fetch calls to routes** → Create frontend → backend call matrix
3. **Verify SDK usage** → Confirm no HTTP interception needed for Azure services
4. **Document env vars** → Create APIM environment variable mapping
5. **Analyze streaming** → Design APIM policy for SSE passthrough
6. **Identify auth points** → Plan APIM token validation policy

---

**End of Repo Scan Command Plan**
