# Appendix A: Azure SDK Client Inventory

**Scope**: All Azure SDK clients instantiated and used by the backend application  
**Purpose**: Document client initialization, configuration sources, methods called, and implied network endpoints  
**Discovery Method**: Comprehensive grep + source code analysis across app/backend/  
**Critical Finding**: All Azure service calls are SDK-based (not raw HTTP), meaning APIM cannot intercept downstream Azure service traffic. Only ONE raw HTTP call exists (enrichment service, which is internal to deployment).

---

## A1. AsyncAzureOpenAI Client

**Purpose**: Handle all Azure OpenAI API calls (chat completions, embeddings generation)  
**Package**: openai (version 1.55.3)  
**SDK Type**: Official OpenAI SDK with Azure OpenAI support

### Instantiation Locations

| Location | File | Line(s) | Code |
|----------|------|---------|------|
| **chatreadretrieveread** | approaches/chatreadretrieveread.py | 177 | `self.client = AsyncAzureOpenAI(azure_endpoint=openai.api_base, azure_ad_token_provider=azure_ai_token_provider, api_version=openai.api_version)` |
| **gpt_direct_approach** | approaches/gpt_direct_approach.py | 70 | `self.client = AsyncAzureOpenAI(azure_endpoint=openai.api_base, azure_ad_token_provider=azure_openai_token_provider, api_version=openai.api_version)` |
| **compareworkwithweb** | approaches/compareworkwithweb.py | 83 | `self.client = AsyncAzureOpenAI(azure_endpoint=openai.api_base, azure_ad_token_provider=azure_ai_token_provider, api_version=openai.api_version)` |
| **comparewebwithwork** | approaches/comparewebwithwork.py | 92 | `self.client = AsyncAzureOpenAI(azure_endpoint=openai.api_base, azure_ad_token_provider=azure_ai_token_provider, api_version=openai.api_version)` |
| **chatwebretrieveread** | approaches/chatwebretrieveread.py | 111 | `self.client = AsyncAzureOpenAI(azure_endpoint=openai.api_base, azure_ad_token_provider=azure_ai_token_provider, api_version=openai.api_version)` |

### Configuration Sources

| Config | Source | File | Line | Type | Value/Default |
|--------|--------|------|------|------|---|
| **Endpoint URL** | `openai.api_base` (global) | app.py | 96 | Env Var `AZURE_OPENAI_ENDPOINT` | `https://{region}.openai.azure.com/` |
| **Token Provider** | `get_bearer_token_provider()` | app.py | 100 | Credential Provider | `DefaultAzureCredential` or `ManagedIdentityCredential` |
| **Scope/Authority** | `AZURE_AI_CREDENTIAL_DOMAIN` | app.py:100, core/shared_constants.py:52 | Env Var | `cognitiveservices.azure.com` |
| **API Version** | `openai.api_version` | approaches/chatreadretrieveread.py | 152 | Hardcoded | `2024-10-21` |
| **Chat Deployment** | `AZURE_OPENAI_CHATGPT_DEPLOYMENT` | core/shared_constants.py | 34 | Env Var | `gpt-35-turbo-16k` (default) |
| **Chat Model Name** | `AZURE_OPENAI_CHATGPT_MODEL_NAME` | core/shared_constants.py | 35 | Env Var | Resolved at runtime from deployment properties |

### Methods Called

| Method | File | Line(s) | Purpose | HTTP Verb | Endpoint Path |
|--------|------|---------|---------|-----------|---|
| `create()` | chatreadretrieveread.py | 420-460 | Stream chat completions | POST | `/deployments/{deployment}/chat/completions` |
| `create()` (streaming) | chatreadretrieveread.py | 463-495 | Get streaming tokens | POST | `/deployments/{deployment}/chat/completions?stream=true` |

### Network Endpoints Implied

| Endpoint | Hostname Pattern | Port | Notes |
|----------|------------------|------|-------|
| **Azure OpenAI Chat Completions** | `{openai_endpoint}/deployments/{chatgpt_deployment}/chat/completions` | 443 | Resolved from `AZURE_OPENAI_ENDPOINT` env var; token injected by SDK |
| **Azure OpenAI Embeddings** | `{openai_endpoint}/deployments/{embedding_deployment}/embeddings` | 443 | Used in approaches if `USE_AZURE_OPENAI_EMBEDDINGS=true`; not used in current deployment |

**APIM Visibility**: ❌ **BLOCKED** - Azure OpenAI SDK handles authentication internally via bearer token provider. APIM cannot intercept or route these downstream Azure service calls. Network requests bypass APIM.

---

## A2. SearchClient (Azure Cognitive Search)

**Purpose**: Hybrid vector + keyword search with semantic reranking  
**Package**: azure-search-documents (version 11.5.1)  
**SDK Type**: Official Azure SDK

### Instantiation Locations

| Location | File | Line(s) | Code |
|----------|------|---------|------|
| **App startup (lifespan)** | app.py | 492 | `SearchClient(endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"], index_name=index, credential=AZURE_CREDENTIAL, audience=ENV["AZURE_SEARCH_AUDIENCE"])` |
| **chatreadretrieveread init** | approaches/chatreadretrieveread.py | 156 | `self.search_client = search_client` (passed as constructor arg) |
| **comparewebwithwork init** | approaches/comparewebwithwork.py | 120 | `self.search_client = search_client` (passed as constructor arg) |

### Configuration Sources

| Config | Source | File | Line | Type | Value |
|--------|--------|------|------|------|-------|
| **Endpoint URL** | `AZURE_SEARCH_SERVICE_ENDPOINT` | core/shared_constants.py | 26 | Env Var | `https://{service_name}.search.windows.net` |
| **Index Name** | From `group_items` cache | app.py | 492 | Cosmos DB | Multiple indexes per group/content container |
| **Credential** | `AZURE_CREDENTIAL` | core/shared_constants.py:108 | DefaultAzureCredential/ManagedIdentityCredential | Token provider (implicit) |
| **Audience** | `AZURE_SEARCH_AUDIENCE` | core/shared_constants.py:27 | Env Var | `AzurePublicCloud` (default) |

### Methods Called

| Method | File | Line(s) | Purpose | Query Type |
|--------|------|---------|---------|-----------|
| `search()` | chatreadretrieveread.py | 354-380 | Hybrid search (vector + keyword + semantic) | POST to `/indexes/{index}/docs/search` |
| `search()` with `vector_queries` | chatreadretrieveread.py | 355 | Vector search component | POST (vector payload in request) |
| `search()` with `semantic_configuration_name` | chatreadretrieveread.py | 356 | Semantic reranking | POST |

### Network Endpoints Implied

| Endpoint | Hostname Pattern | Port | Notes |
|----------|------------------|------|-------|
| **Search API** | `{search_endpoint}/indexes/{index_name}/docs/search` | 443 | Resolved from `AZURE_SEARCH_SERVICE_ENDPOINT` env var |
| **Vector Search** | Same endpoint | 443 | Vector embedding sent in request JSON |
| **Semantic Ranking** | Same endpoint | 443 | Semantic re-ranking applied server-side |

**APIM Visibility**: ❌ **BLOCKED** - Azure Cognitive Search SDK internally manages authentication and routing. APIM cannot intercept.

---

## A3. BlobServiceClient (Azure Blob Storage)

**Purpose**: Upload/download documents, persist examples configuration  
**Package**: azure-storage-blob (version 12.23.0)  
**SDK Type**: Official Azure SDK

### Instantiation Locations

| Location | File | Line(s) | Code |
|----------|------|---------|------|
| **App startup (lifespan)** | app.py | 452 | `BlobServiceClient(account_url=ENV["AZURE_BLOB_STORAGE_ENDPOINT"], credential=AZURE_CREDENTIAL)` |
| **Stored in app_clients** | app.py | 453 | `app_clients[AZURE_BLOB_CLIENT_KEY] = blob_client` |
| **Referenced as global** | app.py | 168, 222, etc. | `app_clients[AZURE_BLOB_CLIENT_KEY].get_container_client(...)` |

### Configuration Sources

| Config | Source | File | Line | Type | Value |
|--------|--------|------|------|------|-------|
| **Storage Account URL** | `AZURE_BLOB_STORAGE_ENDPOINT` | core/shared_constants.py:23 | Env Var | `https://{account_name}.blob.core.windows.net` |
| **Credential** | `AZURE_CREDENTIAL` | core/shared_constants.py:108 | DefaultAzureCredential/ManagedIdentityCredential | Token provider (implicit) |
| **Upload Container** | From `group_items` cache | app.py:476 | Cosmos DB | e.g., `proj7-upload`, `proj8-upload` |
| **Content Containers** | From `group_items` cache | app.py:478 | Cosmos DB | e.g., `proj7-content` (after document pipeline processing) |
| **Config Container** | Hardcoded | app.py:117 | String | `config` (for example list storage) |

### Methods Called

| Method | File | Line(s) | Purpose | HTTP Verb | Storage Operation |
|--------|------|---------|---------|-----------|-------------------|
| `get_container_client()` | app.py | 476-481 | Get container reference | N/A (local) | Container connection |
| `get_blob_client()` | app.py | 168 | Get blob reference | N/A (local) | Blob connection |
| `upload_blob()` | app.py | 175 | Upload file from /file endpoint | PUT | Blob upload |
| `download_blob()` | app.py | 204 | Download file for /get-file endpoint | GET | Blob download |
| `list_blobs()` | app.py | 215 | List blobs in container (/getfolders) | GET | Container listing |
| `delete_blob()` | app.py | 235 | Delete blob (/deleteItems) | DELETE | Blob deletion |
| `upload_blob()` with overwrite | app.py | 175, 237 | Persist examples config | PUT | Blob upload/overwrite |
| `get_blob_properties()` | app.py | 240 | Check blob ETag for caching | HEAD | Blob metadata |
| `readall()` | app.py | 241 | Read entire blob into memory | GET | Blob download |

### Network Endpoints Implied

| Endpoint | Hostname Pattern | Port | Operation |
|----------|------------------|------|-----------|
| **Blob Upload** | `{storage_endpoint}/{container}/{blob_path}` | 443 | PUT - Upload file |
| **Blob Download** | `{storage_endpoint}/{container}/{blob_path}` | 443 | GET - Download file |
| **Blob Delete** | `{storage_endpoint}/{container}/{blob_path}` | 443 | DELETE - Delete file |
| **Container List** | `{storage_endpoint}/{container}?restype=container&comp=list` | 443 | GET - List blobs |
| **Blob Metadata** | `{storage_endpoint}/{container}/{blob_path}?comp=metadata` | 443 | HEAD - Get properties |

**APIM Visibility**: ❌ **BLOCKED** - Azure Blob Storage SDK internally manages authentication and routing. APIM cannot intercept.

---

## A4. CosmosClient (Azure Cosmos DB)

**Purpose**: Store chat history, status logs, user profiles, RBAC group mappings  
**Package**: azure-cosmos (version 4.7.0)  
**SDK Type**: Official Azure SDK

### Instantiation Locations

| Location | File | Line(s) | Code |
|----------|------|---------|------|
| **App startup (lifespan)** | app.py | 441 | `cosmosdb_client = CosmosClient(url=ENV["COSMOSDB_URL"], credential=AZURE_CREDENTIAL)` |
| **Stored in app_clients** | app.py | 443 | `app_clients[AZURE_COSMOSDB_CLIENT_KEY] = cosmosdb_client` |
| **StatusLog wrapper** | app.py | 450, shared_code/status_log.py:47 | `StatusLog(..., cosmos_client=cosmosdb_client)` |
| **UserProfile wrapper** | app.py | 455 | `UserProfile(..., cosmos_client=cosmosdb_client)` |

### Configuration Sources

| Config | Source | File | Line | Type | Value |
|--------|--------|------|------|------|-------|
| **Endpoint URL** | `COSMOSDB_URL` | core/shared_constants.py:47 | Env Var | `https://{account}.documents.azure.com:443/` |
| **Credential** | `AZURE_CREDENTIAL` | core/shared_constants.py:108 | DefaultAzureCredential/ManagedIdentityCredential | Token provider (implicit) |
| **Consistency Level** | Hardcoded | shared_code/status_log.py:47 | String | `Session` consistency |
| **Log Database** | `COSMOSDB_LOG_DATABASE_NAME` | core/shared_constants.py:49 | Env Var | `statusdb` (default) |
| **Log Container** | `COSMOSDB_LOG_CONTAINER_NAME` | core/shared_constants.py:54 | Env Var | `statuscontainer` (default) |
| **Chat History Database** | `COSMOSDB_URL` | core/shared_constants.py:50 | Env Var | `ChatHistoryDatabase` (inferred from usage) |
| **Chat History Container** | `COSMOSDB_CHATHISTORY_CONTAINER` | core/shared_constants.py:51 | Env Var | `chat_history_session` (default) |

### Methods Called

| Method | File | Line(s) | Purpose | Operation |
|--------|------|---------|---------|-----------|
| `get_database_client()` | shared_code/status_log.py:56 | Get database reference | N/A (local) |
| `get_container_client()` | shared_code/status_log.py:58 | Get container reference | N/A (local) |
| `items.upsert_item()` | app.py:628, shared_code/status_log.py:89 | Insert/update status log | POST/PUT |
| `query_items()` | app.py:614-626 | Query upload status (getalluploadstatus endpoint) | POST (query) |
| `query_items()` with SELECT DISTINCT | app.py:700 | Get unique tags (/gettags endpoint) | POST (query) |
| `read_item()` | routers/sessions.py | Get chat history | GET |
| `query_items()` | routers/sessions.py | List sessions | POST (query) |

### Network Endpoints Implied

| Endpoint | Hostname Pattern | Port | Operation |
|----------|------------------|------|-----------|
| **Cosmos REST API** | `{cosmosdb_url}/dbs/{database}/colls/{container}/docs` | 443 | Document operations |
| **Query Endpoint** | `{cosmosdb_url}/dbs/{database}/colls/{container}/docs` with POST | 443 | Query execution |
| **Upsert** | Same endpoint | 443 | Create/update document |
| **Read** | Same endpoint | 443 | Read document by ID |

**APIM Visibility**: ❌ **BLOCKED** - Azure Cosmos DB SDK internally manages authentication and routing. APIM cannot intercept.

---

## A5. Enrichment Service (Raw HTTP - NOT Azure SDK)

**Purpose**: Generate embeddings for vector search  
**Technology**: Flask backend (separate service, in-process for docker or remote)  
**Protocol**: HTTP (REST)  
**SDK Type**: `requests` library (raw HTTP, not Azure SDK)

### Instantiation Locations

| Location | File | Line(s) | Code |
|----------|------|---------|------|
| **chatreadretrieveread** | approaches/chatreadretrieveread.py | 298-310 | `response = requests.post(url, json=data, headers=headers, timeout=60)` |

### Configuration Sources

| Config | Source | File | Line | Type | Value |
|--------|--------|------|------|------|-------|
| **Service URL Base** | `ENRICHMENT_APPSERVICE_URL` | core/shared_constants.py:64 | Env Var | `http://enrichment` (default) or full URL |
| **Endpoint Path** | Hardcoded | chatreadretrieveread.py:298 | String | `/models/{escaped_target_model}/embed` |
| **Target Model** | `TARGET_EMBEDDINGS_MODEL` | core/shared_constants.py:59 | Env Var | `BAAI/bge-small-en-v1.5` (default) |

### Methods Called

| Method | File | Line(s) | Purpose | HTTP Verb | Content-Type |
|--------|------|---------|---------|-----------|---|
| `requests.post()` | chatreadretrieveread.py:298-310 | POST embedding request | POST | `application/json` |

### Network Endpoints Implied

| Endpoint | Hostname Pattern | Port | Operation |
|----------|------------------|------|-----------|
| **Embeddings API** | `{enrichment_service_url}/models/{model_name}/embed` | 80 or 443 | POST embedding request |
| **Default (docker)** | `http://enrichment/models/...` | 80 | Internal service discovery (docker network) |
| **Remote** | Full URL from env var | Variable | External enrichment service (e.g., App Service) |

**⚠️ CRITICAL - This is the ONLY raw HTTP call in the entire backend**

**Timeout Configuration** (Evidence: chatreadretrieveread.py:310):
- Hardcoded timeout: 60 seconds
- Fallback behavior: If `ENRICHMENT_OPTIONAL=true`, continues with text-only search (no embeddings)

**Error Handling** (Evidence: chatreadretrieveread.py:313-330):
- Status code != 200: Log error, optionally degrade to keyword search
- Exception (timeout/connection): Log error, optionally degrade to keyword search

**APIM Visibility**: ✅ **VISIBLE** - Raw HTTP call via `requests` library. APIM could potentially intercept if enrichment endpoint is exposed through APIM (currently internal deployment service, not exposed).

---

## A6. Summary Table: All Azure SDK Clients

| SDK | Package | Version | Client | Instantiation | Endpoint | Auth Method | APIM Visible? |
|-----|---------|---------|--------|---|---|---|---|
| OpenAI | openai | 1.55.3 | AsyncAzureOpenAI | approaches (all) | `AZURE_OPENAI_ENDPOINT` | Bearer Token (SDK) | ❌ No |
| Cognitive Search | azure-search-documents | 11.5.1 | SearchClient | app.py:492 | `AZURE_SEARCH_SERVICE_ENDPOINT` | Bearer Token (SDK) | ❌ No |
| Blob Storage | azure-storage-blob | 12.23.0 | BlobServiceClient | app.py:452 | `AZURE_BLOB_STORAGE_ENDPOINT` | Bearer Token (SDK) | ❌ No |
| Cosmos DB | azure-cosmos | 4.7.0 | CosmosClient | app.py:441 | `COSMOSDB_URL` | Bearer Token (SDK) | ❌ No |
| Embeddings | requests (raw HTTP) | - | requests.post() | chatreadretrieveread.py:298 | `ENRICHMENT_APPSERVICE_URL` | None (internal) | ✅ Yes* |

*Enrichment service is currently internal; visible to APIM only if exposed through it

---

## A7. Critical Security Implications

### Authentication Architecture

**Current State**:
- All Azure services authenticate via `DefaultAzureCredential` → `ManagedIdentityCredential` (production) or `DefaultAzureCredential` (local dev)
- No API Keys stored in environment or code
- Tokens obtained at startup and refreshed by SDKs automatically
- Service-to-service authentication only; no user authentication layer

**Evidence**:
```python
# core/shared_constants.py:102-108
if ENV["LOCAL_DEBUG"] == "true":
    AZURE_CREDENTIAL = DefaultAzureCredential(authority=AUTHORITY)
else:
    AZURE_CREDENTIAL = ManagedIdentityCredential(authority=AUTHORITY)
```

### APIM Gateway Limitations

**Key Finding**: APIM cannot intercept Azure SDK calls because:
1. SDKs handle authentication internally (bearer tokens injected by SDK)
2. SDKs bypass HTTP proxies for Azure service endpoints (direct connection)
3. Network routing: Client → APIM → Backend App → Azure Services
   - APIM sees: Backend HTTP requests from app
   - APIM cannot see: Backend app → Azure Services calls
4. Only raw HTTP calls (enrichment service) are visible to APIM

**Implication for Cost Tracking**: To implement cost tracking per caller, you must:
- NOT rely on APIM to track Azure SDK calls
- Add logging in the application to record which caller triggered which Azure service calls
- Use correlation IDs to link APIM requests to Azure service calls
- Implement via backend middleware (see 05-header-contract-draft.md)

---

## A8. Configuration Checklist for Deployment

Before APIM integration, ensure:

- [ ] `AZURE_OPENAI_ENDPOINT` set to Azure OpenAI resource URL
- [ ] `AZURE_OPENAI_CHATGPT_DEPLOYMENT` set to deployment name (e.g., gpt-4o)
- [ ] `AZURE_SEARCH_SERVICE_ENDPOINT` set to Cognitive Search URL
- [ ] `AZURE_BLOB_STORAGE_ENDPOINT` set to blob storage URL
- [ ] `COSMOSDB_URL` set to Cosmos DB endpoint
- [ ] `ENRICHMENT_APPSERVICE_URL` set to embeddings service URL
- [ ] `AZURE_AI_CREDENTIAL_DOMAIN` set correctly (usually `cognitiveservices.azure.com`)
- [ ] Managed Identity assigned proper RBAC roles on all Azure services
- [ ] Private endpoints configured in production (all Azure services behind private links)
- [ ] `LOCAL_DEBUG` set to `false` in production (forces ManagedIdentityCredential)

---

## References

- [01-api-call-inventory.md](01-api-call-inventory.md) - HTTP endpoint inventory
- [02-auth-and-identity.md](02-auth-and-identity.md) - Service authentication patterns
- [05-header-contract-draft.md](05-header-contract-draft.md) - Governance header strategy and middleware insertion points
- [INDEX.md](INDEX.md) - Navigation hub for all documents
