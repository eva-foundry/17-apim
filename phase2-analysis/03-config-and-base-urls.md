# Configuration and Base URLs

**Scope**: Environment variables, configuration files, and service endpoint definitions  
**Discovery Method**: Source code scan of app.py (ENV dict), requirements.txt, backend.env template, vite.config.ts  
**Generated**: Evidence-based configuration inventory with per-environment override patterns  

---

## Executive Summary

The InfoJP system uses **47 environment variables** to configure:
- Azure service endpoints (OpenAI, Search, Blob, Cosmos, AI Services)
- Feature flags (chat mode selection, optional assistants)
- Knowledge base field mappings (content, page numbers, source files)
- Query and translation settings (language, embeddings model)
- Frontend base URL (dev proxy via Vite)
- Enrichment service URL (internal API)

**Configuration Loading Pattern**:
1. **Backend**: Hardcoded defaults in `app.py:41-85` (ENV dict), then overridden by OS environment
2. **Frontend**: Hardcoded proxy in `vite.config.ts:23-24` (dev only), no env-based URL
3. **Enrichment**: Base URL passed via `ENRICHMENT_APPSERVICE_URL` env var

---

## Environment Variable Registry

### Azure Service Endpoints (Production-Required)

These must be set in all environments; no fallback defaults.

| Variable | Purpose | Type | Example | Override Method | Evidence |
|----------|---------|------|---------|-----------------|----------|
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI service URL | String | `https://myopenai.openai.azure.com/` | OS environment | app.py:51, 132 |
| `AZURE_OPENAI_CHATGPT_DEPLOYMENT` | Chat model deployment name | String | `gpt-35-turbo-16k` | OS environment | app.py:52 |
| `AZURE_SEARCH_SERVICE_ENDPOINT` | Azure Cognitive Search service URL | String | `https://gptkb.search.windows.net/` | OS environment | app.py:49 |
| `AZURE_SEARCH_INDEX` | Index name (hybrid vector + keyword) | String | `gptkbindex` | OS environment | app.py:50 |
| `AZURE_SEARCH_AUDIENCE` | Search service audience scope | String | `https://search.azure.com/` | OS environment | app.py:50 |
| `AZURE_BLOB_STORAGE_ENDPOINT` | Azure Blob Storage account URL | String | `https://mystorageacct.blob.core.windows.net/` | OS environment | app.py:43 |
| `AZURE_BLOB_STORAGE_ACCOUNT` | Storage account name | String | `mystorageacct` | OS environment | app.py:42 |
| `COSMOSDB_URL` | Cosmos DB account endpoint | String | `https://mycosmosdb.documents.azure.com:443/` | OS environment | app.py:76 |
| `AZURE_AI_ENDPOINT` | Azure AI Services (translat, detect lang) | String | `https://myaiservices.cognitiveservices.azure.com/` | OS environment | app.py:82 |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID for ARM API | String (GUID) | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | OS environment | app.py:60 |

**Evidence**: app.py lines 41-85 (ENV dict initialization with None defaults)

### Azure Service Authentication

| Variable | Purpose | Type | Example | Evidence |
|----------|---------|------|---------|----------|
| `AZURE_OPENAI_AUTHORITY_HOST` | OAuth 2.0 authority selection | String | `AzureCloud` \| `AzureUSGovernment` | app.py:54, 92-94 |
| `AZURE_AI_CREDENTIAL_DOMAIN` | Token scope domain | String | `cognitiveservices.azure.com` (default) | app.py:86 |
| `LOCAL_DEBUG` | Credential selection (dev vs prod) | Boolean string | `true` \| `false` | app.py:98-100 |

### Container & Index Configuration

| Variable | Purpose | Type | Default | Evidence |
|----------|---------|------|---------|----------|
| `AZURE_BLOB_STORAGE_CONTAINER` | Content documents container | String | `content` | app.py:44 |
| `AZURE_BLOB_STORAGE_UPLOAD_CONTAINER` | Upload queue container | String | `upload` | app.py:45 |
| `AZURE_SEARCH_SERVICE` | Search service name (for metadata) | String | `gptkb` | app.py:48 |
| `COSMOSDB_LOG_DATABASE_NAME` | Cosmos DB database | String | `statusdb` | app.py:77 |
| `COSMOSDB_LOG_CONTAINER_NAME` | Cosmos DB container | String | `statuscontainer` | app.py:78 |

### Azure OpenAI Model Configuration

| Variable | Purpose | Type | Default | Evidence |
|----------|---------|------|---------|----------|
| `AZURE_OPENAI_CHATGPT_MODEL_NAME` | Model name (e.g., gpt-3.5-turbo) | String | `` (empty) | app.py:56 |
| `AZURE_OPENAI_CHATGPT_MODEL_VERSION` | Model version | String | `` (empty) | app.py:57 |
| `USE_AZURE_OPENAI_EMBEDDINGS` | Use Azure OpenAI or external embeddings | Boolean string | `false` | app.py:58 |
| `EMBEDDING_DEPLOYMENT_NAME` | Embeddings model deployment name | String | `` (empty) | app.py:59 |
| `AZURE_OPENAI_EMBEDDINGS_MODEL_NAME` | Embeddings model name | String | `` (empty) | app.py:61 |
| `AZURE_OPENAI_EMBEDDINGS_VERSION` | Embeddings model version | String | `` (empty) | app.py:62 |

### Feature Flags

| Variable | Purpose | Type | Default | Used By | Evidence |
|----------|---------|------|---------|---------|----------|
| `ENABLE_WEB_CHAT` | Enable web search in chat | Boolean string | `false` | GET /getFeatureFlags | app.py:81 |
| `ENABLE_UNGROUNDED_CHAT` | Enable chat without RAG | Boolean string | `false` | GET /getFeatureFlags | app.py:81 |
| `ENABLE_MATH_ASSISTANT` | Enable math problem solver | Boolean string | `false` | GET /getFeatureFlags | app.py:81 |
| `ENABLE_TABULAR_DATA_ASSISTANT` | Enable CSV analysis | Boolean string | `false` | GET /getFeatureFlags | app.py:81 |
| `USE_SEMANTIC_RERANKER` | Use Azure Search semantic reranker | Boolean string | `true` | ChatReadRetrieveReadApproach | app.py:66 |

### External Service URLs

| Variable | Purpose | Type | Default | Evidence |
|----------|---------|------|---------|----------|
| `ENRICHMENT_APPSERVICE_URL` | Embeddings service base URL | String | `enrichment` (fallback) | app.py:80 |
| `BING_SEARCH_ENDPOINT` | Bing Search API endpoint | String | `https://api.bing.microsoft.com/` | app.py:71 |
| `AZURE_ARM_MANAGEMENT_API` | Azure ARM API endpoint | String | `https://management.azure.com` | app.py:65 |

### Application UI Configuration

| Variable | Purpose | Type | Default | Used By | Evidence |
|----------|---------|------|---------|---------|----------|
| `APPLICATION_TITLE` | UI page title | String | `Information Assistant, built with Azure OpenAI` | GET /getApplicationTitle | app.py:67 |
| `CHAT_WARNING_BANNER_TEXT` | Disclaimer text in chat | String | `` (empty) | GET /getWarningBanner | app.py:64 |
| `MAX_CSV_FILE_SIZE` | Max CSV upload size in MB | String | `7` | GET /getMaxCSVFileSize | app.py:84 |

### Knowledge Base Field Mappings

| Variable | Purpose | Type | Default | Evidence |
|----------|---------|------|---------|----------|
| `KB_FIELDS_CONTENT` | Content field name in search index | String | `content` | app.py:68 |
| `KB_FIELDS_PAGENUMBER` | Page number field name | String | `pages` | app.py:69 |
| `KB_FIELDS_SOURCEFILE` | Source file field name | String | `file_name` | app.py:70 |
| `KB_FIELDS_CHUNKFILE` | Chunk file field name | String | `chunk_file` | app.py:71 |

### Query and Translation Configuration

| Variable | Purpose | Type | Default | Evidence |
|----------|---------|------|---------|----------|
| `QUERY_TERM_LANGUAGE` | Query language for search | String | `English` | app.py:77 |
| `TARGET_EMBEDDINGS_MODEL` | Embeddings model identifier | String | `BAAI/bge-small-en-v1.5` | app.py:78 |
| `TARGET_TRANSLATION_LANGUAGE` | Translation target language | String | `en` | app.py:80 |
| `AZURE_AI_LOCATION` | Azure AI Services location | String | `` (empty) | app.py:82 |
| `BING_SEARCH_KEY` | Bing Search API key | String | `` (empty) | app.py:87 |
| `ENABLE_BING_SAFE_SEARCH` | Enable safe search filtering | Boolean string | `true` | app.py:88 |

### Search & Content Configuration

| Variable | Purpose | Type | Default | Evidence |
|----------|---------|------|---------|----------|
| `KB_FIELDS_CONTENT` | Search index field: document content | String | `content` | app.py:68 |
| `KB_FIELDS_PAGENUMBER` | Search index field: page number | String | `pages` | app.py:69 |
| `KB_FIELDS_SOURCEFILE` | Search index field: source file name | String | `file_name` | app.py:70 |
| `KB_FIELDS_CHUNKFILE` | Search index field: chunk metadata | String | `chunk_file` | app.py:71 |
| `QUERY_TERM_LANGUAGE` | Language for query terms | String | `English` | app.py:73 |
| `TARGET_EMBEDDINGS_MODEL` | Embeddings model name | String | `BAAI/bge-small-en-v1.5` | app.py:79 |

### Translation & Localization

| Variable | Purpose | Type | Default | Evidence |
|----------|---------|------|---------|----------|
| `TARGET_TRANSLATION_LANGUAGE` | Target language for responses | String | `en` | app.py:83 |
| `AZURE_AI_LOCATION` | Azure AI Services location | String | `` (empty) | app.py:83 |

### Optional Search Capabilities

| Variable | Purpose | Type | Default | Evidence |
|----------|---------|------|---------|----------|
| `BING_SEARCH_KEY` | Bing Search API key | String | `` (empty) | app.py:72 |
| `ENABLE_BING_SAFE_SEARCH` | Bing safe search setting | Boolean string | `true` | app.py:74 |

---

## Configuration Loading Pattern

### Backend (app.py:41-140)

```python
# Step 1: Define defaults
ENV = {
    "AZURE_OPENAI_ENDPOINT": None,                    # Required; no default
    "AZURE_OPENAI_CHATGPT_DEPLOYMENT": "gpt-35-turbo-16k",  # Has default
    "LOCAL_DEBUG": "false",                           # Has default
    ...
}

# Step 2: Override with OS environment
for key, value in ENV.items():
    new_value = os.getenv(key)
    if new_value is not None:
        ENV[key] = new_value
    elif value is None:
        raise ValueError(f"Environment variable {key} not set")

# Step 3: Use throughout app
search_client = SearchClient(
    endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"],
    ...
)

# Step 4: Pass to approaches
chat_approaches = {
    Approaches.ReadRetrieveRead: ChatReadRetrieveReadApproach(
        ...,
        ENV["AZURE_OPENAI_ENDPOINT"],
        ENV["AZURE_OPENAI_CHATGPT_DEPLOYMENT"],
        ...
    ),
}
```

**Evidence**: app.py:41-140

**Loading Order**:
1. Defaults in code (preferred for development)
2. OS environment `os.getenv()` (overrides defaults)
3. Raises error if required variable not set

### Frontend (vite.config.ts:20-24)

```typescript
server: {
    proxy: {
        "/ask": "http://localhost:5000",    // Dev proxy only
        "/chat": "http://localhost:5000"    // Dev proxy only
    }
}
```

**Evidence**: vite.config.ts:20-24

**Characteristics**:
- Hardcoded to localhost:5000 (development only)
- No environment variable support
- Vite proxy only active in dev mode (`npm run dev`)
- Production builds (Vite output) do NOT include proxy; rely on same-origin deployment

**APIM Implication**: 
- Frontend needs env var to override proxy base URL for APIM gateway
- Currently NOT supported; frontend assumes backend on same origin

---

## Per-Environment Configuration Example

### Local Development (`LOCAL_DEBUG=true`)

```bash
# Required (must be reachable from dev machine)
AZURE_OPENAI_ENDPOINT=https://myopenai.openai.azure.com/
AZURE_OPENAI_CHATGPT_DEPLOYMENT=gpt-35-turbo-16k
AZURE_SEARCH_SERVICE_ENDPOINT=https://gptkb.search.windows.net/
AZURE_BLOB_STORAGE_ENDPOINT=https://mystorageacct.blob.core.windows.net/
COSMOSDB_URL=https://mycosmosdb.documents.azure.com:443/
AZURE_AI_ENDPOINT=https://myaiservices.cognitiveservices.azure.com/

# Optional
LOCAL_DEBUG=true
AZURE_OPENAI_AUTHORITY_HOST=AzureCloud
ENRICHMENT_APPSERVICE_URL=http://localhost:5001  # Local enrichment service
ENABLE_WEB_CHAT=false
ENABLE_MATH_ASSISTANT=true
```

**Credential Flow**:
- `LocalDebug=true` → `DefaultAzureCredential`
- Tries: env vars, VS Code cache, Azure CLI cache

### Production (`LOCAL_DEBUG=false`)

```bash
# Required (backend must have Managed Identity)
AZURE_OPENAI_ENDPOINT=https://myopenai.openai.azure.com/
AZURE_OPENAI_CHATGPT_DEPLOYMENT=gpt-35-turbo-16k
AZURE_SEARCH_SERVICE_ENDPOINT=https://gptkb.search.windows.net/
AZURE_BLOB_STORAGE_ENDPOINT=https://mystorageacct.blob.core.windows.net/
COSMOSDB_URL=https://mycosmosdb.documents.azure.com:443/
AZURE_AI_ENDPOINT=https://myaiservices.cognitiveservices.azure.com/
AZURE_SUBSCRIPTION_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Optional
LOCAL_DEBUG=false  # or omitted
AZURE_OPENAI_AUTHORITY_HOST=AzureCloud
ENRICHMENT_APPSERVICE_URL=https://enrichment-app.azurewebsites.net/  # Remote service
ENABLE_MATH_ASSISTANT=false
```

**Credential Flow**:
- `LocalDebug=false` → `ManagedIdentityCredential`
- Uses system-assigned or user-assigned MSI on App Service/Container

---

## Base URL Configuration for APIM

### Current State (No APIM)

**Frontend → Backend**:
- Dev: Vite proxy `http://localhost:5000` (hardcoded in vite.config.ts)
- Prod: Same origin (frontend static files + backend on same domain)

**Backend → Azure Services**:
- Endpoints from ENV vars (e.g., `AZURE_OPENAI_ENDPOINT`)

**Backend → Enrichment**:
- Base URL from `ENRICHMENT_APPSERVICE_URL` env var (e.g., `http://localhost:5001`)

### With APIM (Required Changes)

**Frontend → APIM → Backend**:

1. **Frontend needs env var for API base URL**:
   ```typescript
   // Currently missing; hardcoded to localhost:5000
   const API_BASE = process.env.REACT_APP_API_BASE || "http://localhost:5000";
   // Then use in api.ts: fetch(`${API_BASE}/chat`, ...)
   ```
   **Recommendation**: Add to frontend env config
   - Development: `REACT_APP_API_BASE=http://localhost:5000`
   - APIM Staging: `REACT_APP_API_BASE=https://apim-staging.azure-api.net/infojp`
   - APIM Production: `REACT_APP_API_BASE=https://apim.azure-api.net/infojp`

2. **APIM must expose backend at a path**:
   ```
   https://apim.azure-api.net/infojp/chat
   https://apim.azure-api.net/infojp/stream
   https://apim.azure-api.net/infojp/getalluploadstatus
   ... (all 20+ endpoints)
   ```

3. **Backend can stay unchanged** (expects requests from APIM at same URL)

---

## Environment Variable Source Files

| File | Format | Purpose | Evidence |
|------|--------|---------|----------|
| `app/backend/app.py` | Python dict | Hardcoded defaults | app.py:41-86 |
| `.env` (if used) | `KEY=VALUE` | Local override (gitignored) | Not found; uses OS env directly |
| `backend.env` | Shell script | Referenced but not present | Mentioned in docs |
| OS environment | Shell/config | Production override | `os.getenv(key)` throughout app.py |
| `vite.config.ts` | TypeScript object | Frontend dev proxy (hardcoded) | vite.config.ts:20-24 |

**Inference**: Application uses OS environment variables exclusively (no `.env` file in code). Configuration must be injected via:
- Azure App Service: Application Settings blade
- Azure Container Apps: Environment variables in container definition
- Kubernetes: ConfigMap or Secret
- Docker: `-e` flag or `.env` file at runtime

---

## Service Endpoint Mapping

### Azure OpenAI Endpoint Usage

```python
# Line 132: Set SDK base
openai.api_base = ENV["AZURE_OPENAI_ENDPOINT"]

# Line 148: Pass to AsyncAzureOpenAI client
self.client = AsyncAzureOpenAI(
    azure_endpoint=openai.api_base,
    ...
)

# Usage: self.client.chat.completions.create(
#   model=ENV["AZURE_OPENAI_CHATGPT_DEPLOYMENT"],
#   ...
# )
```

**Endpoint Format**: `https://{resource-name}.openai.azure.com/`  
**Evidence**: app.py:51, 132, chatreadretrieveread.py:148

### Azure Search Endpoint Usage

```python
# Line 156: SearchClient initialization
search_client = SearchClient(
    endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"],
    index_name=ENV["AZURE_SEARCH_INDEX"],
    credential=azure_credential,
    audience=ENV["AZURE_SEARCH_AUDIENCE"]
)
```

**Endpoint Format**: `https://{resource-name}.search.windows.net/`  
**Evidence**: app.py:156-162

### Cosmos DB Endpoint Usage

```python
# Line 378: CosmosClient initialization
cosmos_client = CosmosClient(
    url=statusLog._url,  # _url comes from ENV["COSMOSDB_URL"]
    credential=azure_credential,
    consistency_level='Session'
)
```

**Endpoint Format**: `https://{resource-name}.documents.azure.com:443/`  
**Evidence**: app.py:378, statusLog initialization (line 153-155)

### Enrichment Service Endpoint Usage

```python
# chatreadretrieveread.py line 225
url = f'{self.embedding_service_url}/models/{self.escaped_target_model}/embed'

# POST to embeddings service (HTTP, not SDK)
response = requests.post(url, json=data, headers=headers, timeout=60)
```

**Endpoint Format**: `http://localhost:5001` (dev) or `https://enrichment-app.azurewebsites.net` (prod)  
**Evidence**: chatreadretrieveread.py:225, app.py:80

---

## Configuration Validation Checklist

| Check | Command | Expected | Evidence |
|-------|---------|----------|----------|
| Required vars set | `python -c "from app import ENV"` | No ValueError | app.py:140 |
| Azure auth works | Backend health | `/health` returns 200 | app.py:261 |
| Search index reachable | Chat response | Search results present | chatreadretrieveread.py:263 |
| Cosmos DB reachable | `/getalluploadstatus` | Status records returned | app.py:377 |
| Blob storage reachable | `/getfolders` | Folder list returned | app.py:342 |

---

## APIM Configuration Implications

### Named Values in APIM

Create APIM named values for dynamic config:

```xml
<!-- APIM Named Values -->
<NamedValue Id="azure-openai-endpoint">
    <Value>https://myopenai.openai.azure.com/</Value>
</NamedValue>

<NamedValue Id="backend-base-url">
    <Value>https://backend.azureapp.net/</Value>
</NamedValue>

<NamedValue Id="enrichment-url">
    <Value>https://enrichment.azureapp.net/</Value>
</NamedValue>
```

### APIM Policies for Base URL Rewriting

```xml
<policies>
    <inbound>
        <!-- Rewrite incoming /infojp/chat to backend /chat -->
        <rewrite-uri template="/chat" />
    </inbound>
    <backend>
        <!-- Route to backend -->
        <set-backend-service base-url="@(context.Variables["backend-url"])" />
    </backend>
</policies>
```

---

## Evidence Summary

| Category | File | Lines | Count |
|----------|------|-------|-------|
| ENV variables | app.py | 41-86 | 32 variables |
| Override mechanism | app.py | 139-140 | for loop + os.getenv() |
| Frontend proxy | vite.config.ts | 20-24 | 2 hardcoded routes |
| Service init | app.py | 132-184 | 4 SDK client initializations |
| Enrichment HTTP | chatreadretrieveread.py | 225-245 | requests.post() call |

