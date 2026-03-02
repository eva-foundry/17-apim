# Auth and Identity Analysis

**Scope**: Authentication mechanisms, token handling, identity propagation  
**Discovery Method**: Source code inspection of credential initialization, token providers, and request middleware  
**Generated**: Evidence-based from backend configuration and approach implementations  

---

## Executive Summary

The InfoJP backend uses **Azure Managed Identity with Azure AD OAuth 2.0** for service-to-service authentication. **No user authentication is detected in the current codebase**—all endpoints are unauthenticated (open to network traffic). Authentication is **implicit through Azure service SDKs** using bearer tokens.

### Key Findings

- **Service Auth Type**: Azure AD OAuth 2.0 (Bearer Tokens)
- **User Auth Type**: None detected (endpoints are public/unauthenticated)
- **Token Acquisition**: `DefaultAzureCredential` (local dev) or `ManagedIdentityCredential` (production)
- **Token Provider**: `get_bearer_token_provider()` from `azure.identity`
- **Identity Propagation**: Tokens automatically injected by SDKs into Azure service calls
- **User Context**: No user identification in request context (no JWT parsing, no user claims)
- **Session Management**: Stateless; no session cookies or tokens passed to frontend

---

## Authentication Architecture

### 1. Service-to-Service Auth (Backend → Azure Services)

#### A. Credential Initialization (app.py:100-107)

```python
# Lines 95-107: Local debug vs. production credential selection
if ENV["LOCAL_DEBUG"] == "true":
    azure_credential = DefaultAzureCredential(authority=AUTHORITY)
else:
    azure_credential = ManagedIdentityCredential(authority=AUTHORITY)

# Authority selection (Azure Cloud or Government)
if ENV["AZURE_OPENAI_AUTHORITY_HOST"] == "AzureUSGovernment":
    AUTHORITY = AzureAuthorityHosts.AZURE_GOVERNMENT
else:
    AUTHORITY = AzureAuthorityHosts.AZURE_PUBLIC_CLOUD

# Token provider for OpenAI SDK
openai.api_type = "azure_ad"
token_provider = get_bearer_token_provider(
    azure_credential,
    f'https://{ENV["AZURE_AI_CREDENTIAL_DOMAIN"]}/.default'
)
openai.azure_ad_token_provider = token_provider
```

**Evidence**: app.py:95-108

**Inference**: 
- **Local Dev**: Uses `DefaultAzureCredential` which tries multiple sources in order:
  1. Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
  2. Visual Studio Code token cache
  3. Azure CLI token cache
  4. Managed Identity (if running in Azure)
- **Production**: Uses `ManagedIdentityCredential` directly (requires system-assigned or user-assigned MSI)

#### B. Bearer Token Injection (OpenAI Example)

```python
# Lines 143-148: AsyncAzureOpenAI client initialization
self.client = AsyncAzureOpenAI(
    azure_endpoint=openai.api_base,
    azure_ad_token_provider=azure_ai_token_provider,  # Token provider passed here
    api_version=openai.api_version
)

# Token is automatically injected into all calls:
# self.client.chat.completions.create(...) 
#   → Authorization: Bearer <token from provider>
```

**Evidence**: chatreadretrieveread.py:143-148

#### C. SearchClient (Azure Search)

```python
# Lines 156-162: SearchClient initialization
search_client = SearchClient(
    endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"],
    index_name=ENV["AZURE_SEARCH_INDEX"],
    credential=azure_credential,  # Credential object passed
    audience=ENV["AZURE_SEARCH_AUDIENCE"]
)

# Token is automatically obtained and refreshed by SDK
# All search_client.search(...) calls include bearer token
```

**Evidence**: app.py:156-162

#### D. BlobServiceClient (Azure Storage)

```python
# Lines 164-170: BlobServiceClient initialization
blob_client = BlobServiceClient(
    account_url=ENV["AZURE_BLOB_STORAGE_ENDPOINT"],
    credential=azure_credential
)

# Token is automatically obtained for all blob operations
# blob_client.get_container_client(...).download_blob() includes token
```

**Evidence**: app.py:164-170

#### E. CosmosClient (Cosmos DB)

```python
# Lines 153-155: CosmosClient initialization
cosmos_client = CosmosClient(
    url=statusLog._url,
    credential=azure_credential,
    consistency_level='Session'
)

# Token obtained automatically for all queries
# cosmos_client.get_database_client(...).get_container_client(...).query_items(...)
```

**Evidence**: app.py:378-379

---

### 2. User-Facing Authentication

#### HTTP Endpoint Authentication Status

| Endpoint | Auth Required | Mechanism | Evidence |
|----------|---------------|-----------|----------|
| `/chat` (POST) | ❌ NO | None detected | app.py:272 (no auth decorator/middleware) |
| `/stream` (GET) | ❌ NO | None detected | app.py:816 (no auth decorator/middleware) |
| `/file` (POST) | ❌ NO | None detected | app.py:862 (no auth decorator/middleware) |
| `/getalluploadstatus` (POST) | ❌ NO | None detected | app.py:302 (no auth decorator/middleware) |
| `/getfolders` (POST) | ❌ NO | None detected | app.py:342 (no auth decorator/middleware) |
| `/gettags` (POST) | ❌ NO | None detected | app.py:489 (no auth decorator/middleware) |
| `/deleteItems` (POST) | ❌ NO | None detected | app.py:360 (no auth decorator/middleware) |
| `/resubmitItems` (POST) | ❌ NO | None detected | app.py:386 (no auth decorator/middleware) |
| `/logstatus` (POST) | ❌ NO | None detected | app.py:511 (no auth decorator/middleware) |
| `/health` (GET) | ❌ NO | None detected | app.py:261 (standard health check) |
| All GET info/config | ❌ NO | None detected | app.py:544-596 (no auth checks) |

**Inference**: All HTTP endpoints are **publicly accessible** with no authentication requirement. Any caller with network access to the backend can invoke any endpoint.

---

### 3. User Identity in Request Context

#### Search for User/Claims/JWT Parsing

**Searched patterns**:
- `@requires_auth` / `@authenticate`
- `Authorization: Bearer`
- `jwt.decode()`
- `request.user`
- `claims`
- `subject`
- `user_id`

**Result**: ❌ **NOT FOUND**

**Inference**: No user identity is extracted or propagated in request context.

---

### 4. Middleware & Request Guards

#### Search for Request Middleware

```
Searched for:
- @app.middleware
- CORSMiddleware
- AuthenticationMiddleware
- app.add_middleware
```

**Result**: ❌ **NOT FOUND**

**Inference**: No global request middleware is implemented. FastAPI's default CORS behavior applies (reject cross-origin).

**Evidence**: No middleware functions in app.py; no app.add_middleware() calls

---

### 5. Token Scope & Permissions

#### Scope Definition (app.py:103-104)

```python
token_provider = get_bearer_token_provider(
    azure_credential,
    f'https://{ENV["AZURE_AI_CREDENTIAL_DOMAIN"]}/.default'
)
```

**Scope**: `https://cognitiveservices.azure.com/.default`

**Inference**:
- The `.default` scope requests **all permissions** that the Managed Identity has been granted
- Permissions are controlled via **Azure RBAC assignments** (not scopes in the app)
- The backend application principal must have roles assigned:
  - **Azure OpenAI**: `Cognitive Services User` or similar
  - **Azure Search**: `Search Index Data Reader` or similar
  - **Azure Blob**: `Storage Blob Data Reader` or similar
  - **Azure Cosmos DB**: `Cosmos DB Built-in Data Contributor` or similar

**No runtime scope checking** is performed by the application.

---

## Token Flow Diagram

```
┌─────────────────────────────────────────┐
│ Backend App Startup (app.py:95-108)     │
├─────────────────────────────────────────┤
│ 1. Determine credential source:          │
│    - LOCAL_DEBUG=true → DefaultAzureCredential
│    - else → ManagedIdentityCredential    │
│                                          │
│ 2. Select authority:                     │
│    - If AZURE_GOV → Azure Government     │
│    - Else → Azure Public Cloud           │
│                                          │
│ 3. Create token provider:                │
│    get_bearer_token_provider(            │
│      credential,                         │
│      scope=cognitiveservices.azure.com   │
│    )                                     │
│                                          │
│ 4. Pass to SDK clients:                  │
│    - AsyncAzureOpenAI(token_provider) │
│    - SearchClient(credential)            │
│    - BlobServiceClient(credential)       │
│    - CosmosClient(credential)            │
└─────────────────────────────────────────┘
                    ↓
        ┌──────────────────────┐
        │ HTTP Request Received │ (No auth header required)
        │ POST /chat ...       │
        └──────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ Route Handler (app.py:272-293)          │
├─────────────────────────────────────────┤
│ @app.post("/chat")                      │
│ async def chat(request: Request):       │
│   # NO user/identity extraction         │
│   # NO permission checks                │
│   # Proceed directly to approach.run()  │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ Approach.run() (chatreadretrieveread)   │
├─────────────────────────────────────────┤
│ 1. Call OpenAI SDK:                     │
│    await self.client.chat.completions.create(...)
│    → SDK checks if token is valid       │
│    → If expired, token_provider fetches new token
│    → Bearer token automatically injected
│    → Azure OpenAI validates token scope │
│                                          │
│ 2. Call Search SDK:                     │
│    self.search_client.search(...)       │
│    → SDK fetches token from credential  │
│    → Injected as Authorization header   │
│    → Azure Search validates token       │
│                                          │
│ 3. Call Blob SDK:                       │
│    blob_container.download_blob(...)    │
│    → SDK fetches token                  │
│    → Injected and validated             │
│                                          │
│ 4. Call Cosmos SDK:                     │
│    container.query_items(...)           │
│    → SDK fetches token                  │
│    → Injected and validated             │
└─────────────────────────────────────────┘
                    ↓
        ┌──────────────────────┐
        │ Streaming Response    │
        │ Yielded as ndjson    │
        └──────────────────────┘
```

---

## APIM Integration Points

### For User Authentication (Frontend ↔ APIM ↔ Backend)

**Current State**: ❌ **No user authentication implemented**

**Required Steps to Add User Auth**:

1. **Frontend**: Add OAuth 2.0 PKCE flow with Azure AD
   - Request auth token from Azure AD
   - Pass `Authorization: Bearer <user_token>` in all requests

2. **APIM Policy**: Add JWT validation
   - Policy: `<validate-jwt>` inbound
   - Check token signature, issuer, audience
   - Extract claims (user ID, email, roles)
   - Forward claims as headers to backend

3. **Backend Middleware** (NEW - not currently implemented):
   ```python
   from fastapi import Depends, HTTPException
   from fastapi.security import HTTPBearer, HTTPAuthCredentials
   
   security = HTTPBearer()
   
   async def verify_user(credentials: HTTPAuthCredentials = Depends(security)):
       token = credentials.credentials
       try:
           # Decode JWT without verification (APIM validated it)
           payload = jwt.decode(token, options={"verify_signature": False})
           user_id = payload.get("sub")
           return user_id
       except:
           raise HTTPException(status_code=401, detail="Invalid token")
   
   @app.post("/chat")
   async def chat(request: Request, user_id: str = Depends(verify_user)):
       # Now user_id is available
       # Log to Cosmos DB with user context
   ```

### For Service-to-Service Auth (APIM ↔ Backend)

**Current State**: ✅ **Implicit via Azure AD**

**Configuration**:
- Backend Managed Identity must have RBAC role in Azure OpenAI, Search, Blob, Cosmos
- APIM can optionally add Managed Identity to enforce mutual authentication
- **No code changes required**—SDKs handle token exchange

### For Cost/Governance Headers (APIM ↔ Backend)

**Required Backend Middleware** (NEW):
```python
from fastapi import Request, FastAPI

@app.middleware("http")
async def extract_governance_headers(request: Request, call_next):
    """Extract governance headers and include in logs"""
    
    run_id = request.headers.get("X-Run-Id", "unknown")
    correlation_id = request.headers.get("X-Correlation-Id", "unknown")
    caller_app = request.headers.get("X-Caller-App", "unknown")
    cost_center = request.headers.get("X-Cost-Center", "unknown")
    project_id = request.headers.get("X-Project-Id", "unknown")
    
    # Store in request state for later logging
    request.state.run_id = run_id
    request.state.correlation_id = correlation_id
    request.state.caller_app = caller_app
    request.state.cost_center = cost_center
    request.state.project_id = project_id
    
    response = await call_next(request)
    
    # Add governance headers to response
    response.headers["X-Run-Id"] = run_id
    response.headers["X-Correlation-Id"] = correlation_id
    
    return response
```

---

## Environment Variables

### Authentication-Related Variables

| Variable | Purpose | Example | Evidence |
|----------|---------|---------|----------|
| `AZURE_OPENAI_AUTHORITY_HOST` | OAuth 2.0 authority selection | "AzureCloud" or "AzureUSGovernment" | app.py:92-94, 1:98 |
| `AZURE_OPENAI_ENDPOINT` | Token scope service | "https://myopenai.openai.azure.com/" | app.py:51, 103 |
| `AZURE_AI_CREDENTIAL_DOMAIN` | Token scope domain | "cognitiveservices.azure.com" (default line 86) | app.py:86, 103 |
| `LOCAL_DEBUG` | Credential selection | "true" (dev) or "false" (prod) | app.py:98-100 |

### Service Endpoint Variables

| Variable | Service | Example |
|----------|---------|---------|
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI | "https://xxx.openai.azure.com/" |
| `AZURE_SEARCH_SERVICE_ENDPOINT` | Azure Search | "https://xxx.search.windows.net/" |
| `AZURE_BLOB_STORAGE_ENDPOINT` | Azure Blob | "https://xxx.blob.core.windows.net/" |
| `COSMOSDB_URL` | Cosmos DB | "https://xxx.documents.azure.com:443/" |
| `ENRICHMENT_APPSERVICE_URL` | Enrichment Service (HTTP) | "http://localhost:5001" or remote URL |

---

## Security Posture Summary

| Aspect | Status | Risk | Notes |
|--------|--------|------|-------|
| **Service Auth** | ✅ Implemented | 🟢 Low | Azure Managed Identity with bearer tokens. RBAC controls permissions. |
| **User Auth** | ❌ NOT Implemented | 🔴 HIGH | All endpoints are public. Anyone with network access can invoke. |
| **Token Refresh** | ✅ Automatic | 🟢 Low | SDK handles token expiration and refresh. |
| **Token Validation** | ✅ Enforced by Azure | 🟢 Low | Azure services validate token signature and scope. |
| **Endpoint Authorization** | ❌ NOT Implemented | 🔴 HIGH | No role-based access control. All routes equally accessible. |
| **Request Logging** | ⚠️ Partial | 🟡 Medium | Cosmos DB logs file uploads and status, but no user context. |
| **CORS** | ❌ NOT Configured | 🔴 HIGH | Default FastAPI behavior = all origins rejected; check actual config. |

---

## Recommendations for APIM Integration

1. **Add User Authentication** before exposing to public
   - Implement OAuth 2.0 PKCE in frontend
   - Add APIM JWT validation policy
   - Add backend middleware to extract user identity

2. **Add Request Logging Middleware**
   - Capture X-Run-Id, X-Correlation-Id, X-Caller-App headers
   - Log to Cosmos DB with each request
   - Correlate logs across services

3. **Add Rate Limiting Policy**
   - APIM `<rate-limit-by-key>` policy per API key or user
   - Prevent abuse of expensive OpenAI calls

4. **Enforce HTTPS**
   - Ensure APIM requires HTTPS
   - Backend should also enforce HTTPS (proxy setup)

5. **Add CORS Policy**
   - Explicitly configure allowed origins in APIM
   - Remove default open CORS if present

---

## Evidence Summary

| Finding | File | Lines | Details |
|---------|------|-------|---------|
| Service auth setup | app.py | 95-108 | DefaultAzureCredential/ManagedIdentityCredential selection |
| Bearer token provider | app.py | 103-104 | get_bearer_token_provider with cognitiveservices scope |
| OpenAI client init | chatreadretrieveread.py | 143-148 | AsyncAzureOpenAI with token_provider |
| Search client init | app.py | 156-162 | SearchClient with credential object |
| No user auth in /chat | app.py | 272 | @app.post("/chat") with no auth decorator |
| No middleware | app.py | 1-904 | No app.add_middleware() calls, no @app.middleware definitions |
| No JWT parsing | app.py, requirements.txt | all | PyJWT in requirements but never imported in app.py |
| No user context | app.py | 272-293 | No request.user, no user extraction from claims |

