# Appendix Scan Summary

**Completed**: Azure SDK Client Inventory Appendix  
**Date**: January 25, 2026  
**Status**: ✅ COMPLETE - New appendix document created and integrated

---

## What Was Added

### New Document: APPENDIX-A-Azure-SDK-Clients.md

**Location**: `docs/apim-scan/APPENDIX-A-Azure-SDK-Clients.md`  
**Size**: ~1,200 lines  
**Tables**: 10 comprehensive tables  
**Code Blocks**: 8 code examples  
**Evidence Items**: 60+ file:line references

---

## Content Breakdown

### Sections Included

1. **A1: AsyncAzureOpenAI Client**
   - 5 instantiation locations (approaches: chatreadretrieveread, gpt_direct, compareworkwithweb, comparewebwithwork, chatwebretrieveread)
   - Configuration sources (endpoint, token provider, scope, API version, deployment)
   - Methods called (chat completions streaming)
   - Network endpoints (Azure OpenAI REST API)
   - APIM Visibility: ❌ BLOCKED (SDK handles auth internally)

2. **A2: SearchClient (Azure Cognitive Search)**
   - Instantiation: app.py line 492 (lifespan event)
   - Config sources: endpoint, index names from Cosmos, credential
   - Methods: hybrid search, vector queries, semantic reranking
   - Network endpoints: Search REST API with vector payloads
   - APIM Visibility: ❌ BLOCKED (SDK handles auth internally)

3. **A3: BlobServiceClient (Azure Blob Storage)**
   - Instantiation: app.py line 452 (lifespan event)
   - Config sources: storage account endpoint, container names
   - Methods: upload, download, delete, list, properties
   - Network endpoints: Blob REST API (PUT/GET/DELETE)
   - APIM Visibility: ❌ BLOCKED (SDK handles auth internally)

4. **A4: CosmosClient (Azure Cosmos DB)**
   - Instantiation: app.py line 441 (lifespan event)
   - Config sources: endpoint, consistency level, database/container names
   - Methods: upsert items, query items, read items
   - Network endpoints: Cosmos REST API (/dbs/{db}/colls/{coll}/docs)
   - APIM Visibility: ❌ BLOCKED (SDK handles auth internally)

5. **A5: Enrichment Service (Raw HTTP)**
   - Instantiation: chatreadretrieveread.py line 298-310 (requests.post)
   - Config sources: service URL, model name, timeout (60s)
   - Method: POST with embedding request
   - Network endpoints: HTTP to enrichment service
   - ⚠️ ONLY raw HTTP call in entire backend
   - APIM Visibility: ✅ VISIBLE (if exposed through APIM)

6. **A6: Summary Table**
   - All 5 SDK clients compared
   - Package versions, client types, endpoints, auth methods
   - APIM visibility for each

7. **A7: Critical Security Implications**
   - Authentication architecture (Managed Identity + token provider)
   - APIM gateway limitations (cannot intercept SDK calls)
   - Cost tracking implications (must be done in backend, not APIM)

8. **A8: Configuration Checklist**
   - 10-item pre-deployment checklist
   - Environment variables to configure
   - RBAC roles to assign
   - Private endpoint setup

---

## Key Discoveries

### Critical Finding 1: APIM Cannot Intercept Azure SDK Calls

**Evidence**:
- All 4 Azure SDKs (OpenAI, Search, Blob, Cosmos) handle authentication internally
- SDKs bypass HTTP proxies and connect directly to Azure service endpoints
- Network routing is: Client → APIM → Backend App → Azure Services
  - APIM sees: Backend HTTP requests to endpoints
  - APIM does NOT see: Backend app → Azure service calls
- Only the raw HTTP call (enrichment service via requests.post) is visible to APIM

**Implication for Cost Tracking**:
- Cannot rely on APIM to track costs per caller
- Must implement cost tracking in backend middleware
- Must use correlation IDs to link APIM requests to Azure operations
- Cosmos DB logging must include governance headers for attribution

### Critical Finding 2: Only ONE Raw HTTP Call in Backend

**Location**: chatreadretrieveread.py:298-310

```python
response = requests.post(url, json=data, headers=headers, timeout=60)
```

**Purpose**: Generate embeddings for vector search  
**Service**: Enrichment service (Flask backend)  
**Visibility to APIM**: ✅ Yes (if exposed through APIM)

All other Azure service calls use SDKs, which bypass APIM.

### Critical Finding 3: Credential Management is Solid

**Pattern** (core/shared_constants.py:102-108):
```python
if ENV["LOCAL_DEBUG"] == "true":
    AZURE_CREDENTIAL = DefaultAzureCredential(authority=AUTHORITY)
else:
    AZURE_CREDENTIAL = ManagedIdentityCredential(authority=AUTHORITY)
```

**Findings**:
- ✅ No API keys in code or environment
- ✅ Tokens obtained at startup and refreshed automatically by SDKs
- ✅ Different credential types per environment (dev vs prod)
- ✅ Tokens scoped to `https://cognitiveservices.azure.com/.default`

---

## Updated Documentation Index

### Changes to INDEX.md

1. **Added new section**: "6. Azure SDK Client Inventory"
   - Full description of appendix
   - Key findings summary
   - Critical APIM implication (cannot intercept SDK calls)
   - Evidence references

2. **Updated metadata table**: 
   - Old: 5 documents, ~5,000 lines, 145+ evidence items
   - New: 6 documents, ~7,200 lines, 205+ evidence items

---

## Evidence Matrix

### File Evidence (Instantiation Locations)

| SDK | File | Line(s) | Evidence |
|-----|------|---------|----------|
| AsyncAzureOpenAI | approaches/chatreadretrieveread.py | 177 | `self.client = AsyncAzureOpenAI(...)` |
| AsyncAzureOpenAI | approaches/gpt_direct_approach.py | 70 | `self.client = AsyncAzureOpenAI(...)` |
| AsyncAzureOpenAI | approaches/compareworkwithweb.py | 83 | `self.client = AsyncAzureOpenAI(...)` |
| AsyncAzureOpenAI | approaches/comparewebwithwork.py | 92 | `self.client = AsyncAzureOpenAI(...)` |
| AsyncAzureOpenAI | approaches/chatwebretrieveread.py | 111 | `self.client = AsyncAzureOpenAI(...)` |
| SearchClient | app.py | 492 | `SearchClient(endpoint=..., index_name=..., credential=...)` |
| BlobServiceClient | app.py | 452 | `BlobServiceClient(account_url=..., credential=...)` |
| CosmosClient | app.py | 441 | `CosmosClient(url=..., credential=...)` |
| requests.post() | approaches/chatreadretrieveread.py | 298-310 | `response = requests.post(url, json=data, headers=headers, timeout=60)` |

### Configuration Evidence (Endpoint Sources)

| SDK | Config Var | File | Line | Default Value |
|-----|-----------|------|------|---|
| OpenAI | `AZURE_OPENAI_ENDPOINT` | core/shared_constants.py | - | (required) |
| Search | `AZURE_SEARCH_SERVICE_ENDPOINT` | core/shared_constants.py | 26 | (required) |
| Blob | `AZURE_BLOB_STORAGE_ENDPOINT` | core/shared_constants.py | 23 | (required) |
| Cosmos | `COSMOSDB_URL` | core/shared_constants.py | 47 | (required) |
| Enrichment | `ENRICHMENT_APPSERVICE_URL` | core/shared_constants.py | 64 | `http://enrichment` |

### Method Call Evidence (What Operations Each SDK Performs)

| SDK | Method | Operation | File | Line |
|-----|--------|-----------|------|------|
| OpenAI | `create()` | Chat completions | chatreadretrieveread.py | 420-495 |
| Search | `search()` | Hybrid search | chatreadretrieveread.py | 354-380 |
| Blob | `upload_blob()` | File upload | app.py | 175 |
| Blob | `download_blob()` | File download | app.py | 204 |
| Blob | `delete_blob()` | File delete | app.py | 235 |
| Cosmos | `upsert_item()` | Update status log | app.py | 628 |
| Cosmos | `query_items()` | Get upload status | app.py | 614-626 |
| Enrichment | `requests.post()` | Generate embeddings | chatreadretrieveread.py | 298-310 |

---

## Network Endpoints Summary

### Azure SDK Endpoints (SDKs Handle Auth)

| Service | Endpoint Pattern | Port | Auth Method | APIM Visible? |
|---------|------------------|------|-------------|---|
| **Azure OpenAI** | `{endpoint}/deployments/{deployment}/chat/completions` | 443 | Bearer token (SDK) | ❌ No |
| **Cognitive Search** | `{endpoint}/indexes/{index}/docs/search` | 443 | Bearer token (SDK) | ❌ No |
| **Blob Storage** | `{endpoint}/{container}/{blob}` | 443 | Bearer token (SDK) | ❌ No |
| **Cosmos DB** | `{endpoint}/dbs/{db}/colls/{coll}/docs` | 443 | Bearer token (SDK) | ❌ No |

### Raw HTTP Endpoints (Visible to APIM)

| Service | Endpoint Pattern | Port | Auth Method | APIM Visible? |
|---------|------------------|------|-------------|---|
| **Enrichment** | `{enrichment_url}/models/{model}/embed` | 80/443 | None (internal) | ✅ Yes* |

*Only visible if enrichment service is exposed through APIM

---

## APIM Integration Recommendations

### What APIM CAN Do

✅ **Track frontend → backend HTTP calls**
- Measure latency, throughput, errors
- Validate JWT tokens (when implemented)
- Rate limit by subscription/API key
- Log request/response headers

✅ **Inject governance headers**
- Set X-Correlation-Id
- Set X-Run-Id
- Set X-Caller-App
- Set X-Env
- Set X-Cost-Center (optional)

✅ **Route to enrichment service** (if exposed through APIM)
- Apply rate limiting to `/models/*/embed`
- Log enrichment service usage
- Implement circuit breaker

### What APIM CANNOT Do

❌ **Track backend → Azure service calls**
- No visibility into OpenAI API usage
- No visibility into Cognitive Search operations
- No visibility into Blob Storage operations
- No visibility into Cosmos DB operations

❌ **Attribute costs per caller** for Azure services
- SDKs bypass APIM for downstream calls
- Must implement cost tracking in backend middleware
- Must use correlation IDs to link backend operations to callers

### Recommendation: Backend Middleware Pattern

**Instead of relying on APIM**, implement middleware in backend:

1. **Extract governance headers from HTTP request**
   - X-Correlation-Id, X-Run-Id, X-Caller-App, X-Env, X-Cost-Center

2. **Store headers in request.state**
   - Available to all downstream code

3. **Include headers in Cosmos DB logging**
   - Status logs include caller attribution
   - Chat history logs include correlation IDs
   - Enable cost tracking by examining Cosmos data

4. **Use correlation IDs in logs**
   - All Azure SDK calls can be tagged with correlation ID in logging
   - Links APIM requests to backend operations

See [05-header-contract-draft.md](docs/apim-scan/05-header-contract-draft.md) Insertion Points 2-3 for implementation details.

---

## Next Steps

### Phase 3: APIM Design (Use This Appendix For...)

1. **OpenAPI Spec Generation** (06-openapi-spec.yaml)
   - Reference all endpoints from 01-api-call-inventory.md
   - Reference timeout settings from 04-streaming-analysis.md
   - Mark endpoints that DO NOT require auth (current) vs WILL require auth (after JWT added)

2. **APIM Policy Development** (07-apim-policies.xml)
   - Reference governance headers from 05-header-contract-draft.md
   - Use timeout values from 04-streaming-analysis.md
   - Implement rate limiting for enrichment endpoint (only visible raw HTTP call)

3. **Deployment Plan** (08-deployment-plan.md)
   - Reference backend code changes from 05-header-contract-draft.md Insertion Points 2-3
   - Reference configuration from 03-config-and-base-urls.md
   - Reference security hardening from 02-auth-and-identity.md

4. **Security Hardening** (09-security-hardening.md)
   - Implement user authentication (currently missing)
   - Reference token validation patterns from this appendix
   - Implement authorization checks

---

## Document Cross-References

**This appendix is referenced by:**
- [INDEX.md](INDEX.md) - Section 6, Metadata table
- [01-api-call-inventory.md](01-api-call-inventory.md) - APIM Implications section
- [02-auth-and-identity.md](02-auth-and-identity.md) - Credential architecture section
- [03-config-and-base-urls.md](03-config-and-base-urls.md) - Service endpoint mapping section
- [05-header-contract-draft.md](05-header-contract-draft.md) - APIM integration points section

**This appendix references:**
- [core/shared_constants.py](../../base-platform/app/backend/core/shared_constants.py) - Configuration definitions
- [app.py](../../base-platform/app/backend/app.py) - Client initialization and lifespan
- [approaches/chatreadretrieveread.py](../../base-platform/app/backend/approaches/chatreadretrieveread.py) - Client usage patterns

---

**Generated**: 2026-01-25  
**Scope**: Azure SDK client inventory and APIM implications  
**Method**: Evidence-first source code inspection  
**Coverage**: 9 Python files, 32 environment variables, 5 Azure SDKs, 1 raw HTTP call
