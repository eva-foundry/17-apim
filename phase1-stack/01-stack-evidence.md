# Stack Evidence Report

**Date**: 2026-01-25  
**Repo**: `I:\EVA-JP-v1.2\docs\eva-foundation\projects\11-MS-InfoJP\base-platform`  
**Purpose**: Evidence-based tech stack identification for APIM integration planning

---

## Summary

The InfoJP base-platform is a **full-stack RAG application** with:
- **Frontend**: React 18 + TypeScript + Vite
- **Backend**: Python FastAPI + asyncio
- **Workers**: Azure Functions (Python)
- **Enrichment Service**: Python FastAPI + sentence-transformers
- **Streaming**: Server-Sent Events (SSE) via `ndjson-readablestream` + FastAPI `StreamingResponse`

All components use **Azure SDK-based clients** (not raw HTTP) for Azure service access.

---

## A. Frontend Stack Evidence

### Core Framework
**React 18.3.1 + TypeScript + Vite**

**Evidence**:
- `app/frontend/package.json` (lines 17-19):
  ```json
  "react": "18.3.1",
  "react-dom": "18.3.1",
  "typescript": "4.9.5"
  ```

- `app/frontend/vite.config.ts` (lines 1-8):
  ```typescript
  import { defineConfig } from "vite";
  import react from "@vitejs/plugin-react";
  
  export default defineConfig({
      plugins: [react(), nodePolyfills()],
      build: {
          outDir: "../backend/static",
  ```

### UI Framework
**Fluent UI 8.121.8**

**Evidence**:
- `app/frontend/package.json` (lines 12-13):
  ```json
  "@fluentui/react": "8.121.8",
  "@fluentui/react-icons": "2.0.266"
  ```

### API Client Pattern
**Native fetch API + ndjson-readablestream for SSE**

**Evidence**:
- `app/frontend/src/api/api.ts` (line 23):
  ```typescript
  const response = await fetch("/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
  ```

- `app/frontend/package.json` (line 18):
  ```json
  "ndjson-readablestream": "1.2.0"
  ```

### Build & Dev Server
**Vite 5.4.11 with dev proxy**

**Evidence**:
- `app/frontend/vite.config.ts` (lines 20-24):
  ```typescript
  server: {
      proxy: {
          "/ask": "http://localhost:5000",
          "/chat": "http://localhost:5000"
      }
  }
  ```

---

## B. Backend Stack Evidence

### Core Framework
**FastAPI 0.115.0 + Uvicorn 0.30.6**

**Evidence**:
- `app/backend/requirements.txt` (lines 7-8):
  ```
  fastapi==0.115.0
  uvicorn==0.30.6
  ```

- `app/backend/app.py` (lines 15, 197):
  ```python
  from fastapi import FastAPI, File, HTTPException, Request, UploadFile, Form
  from fastapi.responses import RedirectResponse, StreamingResponse
  
  app = FastAPI(
      title="IA Web API",
      description="A Python API to serve as Backend For the Information Assistant Web App",
      version="0.1.0",
      docs_url="/docs",
  )
  ```

### OpenAI Integration
**openai==1.55.3 (Azure OpenAI SDK)**

**Evidence**:
- `app/backend/requirements.txt` (line 18):
  ```
  openai==1.55.3
  ```

- `app/backend/app.py` (lines 16, 27-30):
  ```python
  import openai
  from openai import AsyncAzureOpenAI
  
  # Setup approach with token provider
  openai.api_type = "azure_ad"
  token_provider = get_bearer_token_provider(azure_credential,
                                             f'https://{ENV["AZURE_AI_CREDENTIAL_DOMAIN"]}/.default')
  openai.azure_ad_token_provider = token_provider
  ```

### Azure SDKs (not raw HTTP)
**Evidence**:
- `app/backend/requirements.txt` (lines 1-6):
  ```
  azure-cosmos==4.7.0
  azure-identity==1.18.0
  azure-mgmt-cognitiveservices==13.5.0
  azure-search-documents==11.5.1
  azure-storage-blob==12.23.0
  ```

- `app/backend/app.py` (lines 24-29):
  ```python
  from azure.identity import ManagedIdentityCredential, DefaultAzureCredential, get_bearer_token_provider
  from azure.search.documents import SearchClient
  from azure.storage.blob import BlobServiceClient, ContentSettings
  from azure.mgmt.cognitiveservices import CognitiveServicesManagementClient
  ```

### Authentication Pattern
**Azure Managed Identity + DefaultAzureCredential**

**Evidence**:
- `app/backend/app.py` (lines 129-134):
  ```python
  if ENV["LOCAL_DEBUG"] == "true":
      azure_credential = DefaultAzureCredential(authority=AUTHORITY)
  else:
      azure_credential = ManagedIdentityCredential(authority=AUTHORITY)
  
  token_provider = get_bearer_token_provider(azure_credential, ...)
  ```

### Streaming Implementation
**FastAPI StreamingResponse + async generators**

**Evidence**:
- `app/backend/app.py` (lines 272-293):
  ```python
  @app.post("/chat")
  async def chat(request: Request):
      json_body = await request.json()
      approach = json_body.get("approach")
      impl = chat_approaches.get(Approaches(int(approach)))
      
      r = impl.run(json_body.get("history", []),
                   json_body.get("overrides", {}), ...)
      
      return StreamingResponse(r, media_type="application/x-ndjson")
  ```

---

## C. Enrichment Service Stack Evidence

### Core Framework
**FastAPI 0.115.0 + sentence-transformers 3.1.1**

**Evidence**:
- `app/enrichment/requirements.txt` (lines 7-8, 13):
  ```
  fastapi==0.115.0
  uvicorn==0.30.6
  sentence-transformers==3.1.1
  ```

### Purpose
**Embedding generation service (separate from backend)**

**Evidence**:
- `app/backend/approaches/chatreadretrieveread.py` (references enrichment endpoint):
  - Backend calls `ENV["ENRICHMENT_APPSERVICE_URL"]` for embeddings
  - Not direct Azure OpenAI calls for embeddings in some configurations

---

## D. Azure Functions (Workers) Stack Evidence

### Runtime
**Azure Functions Python + azure-functions==1.21.0**

**Evidence**:
- `functions/requirements.txt` (line 11):
  ```
  azure-functions==1.21.0
  ```

- `functions/host.json` presence indicates Azure Functions project

### Functions Discovered
**Evidence**: `functions/` directory structure:
- `FileUploadedFunc/` - Blob trigger (starts pipeline)
- `FileFormRecSubmissionPDF/` - OCR processing
- `FileFormRecPollingPDF/` - OCR polling
- `FileLayoutParsingOther/` - Layout parsing
- `TextEnrichment/` - Chunking + embedding + indexing
- `ImageEnrichment/` - Image processing
- `FileDeletion/` - Blob cleanup

### Document Processing SDKs
**Evidence**:
- `functions/requirements.txt` (lines 7-9, 19-22):
  ```
  azure-ai-formrecognizer==3.3.3
  azure-ai-vision-imageanalysis==1.0.0b3
  azure-search-documents==11.5.1
  unstructured[csv,doc,docx,email,html,md,msg,ppt,pptx,text,xlsx,xml] == 0.16.17
  ```

---

## E. Dependency Summary Table

| Component | Language | Framework | Version | Evidence File |
|-----------|----------|-----------|---------|---------------|
| Frontend | TypeScript | React | 18.3.1 | `app/frontend/package.json:17` |
| Frontend Build | Node | Vite | 5.4.11 | `app/frontend/package.json:43` |
| Frontend UI | TypeScript | Fluent UI | 8.121.8 | `app/frontend/package.json:12` |
| Backend API | Python 3.x | FastAPI | 0.115.0 | `app/backend/requirements.txt:7` |
| Backend Server | Python | Uvicorn | 0.30.6 | `app/backend/requirements.txt:27` |
| Enrichment API | Python 3.x | FastAPI | 0.115.0 | `app/enrichment/requirements.txt:7` |
| Workers | Python 3.x | Azure Functions | 1.21.0 | `functions/requirements.txt:11` |
| OpenAI SDK | Python | openai | 1.55.3 | `app/backend/requirements.txt:18` |
| Search SDK | Python | azure-search-documents | 11.5.1 | `app/backend/requirements.txt:4` |
| Storage SDK | Python | azure-storage-blob | 12.23.0 | `app/backend/requirements.txt:5` |
| Cosmos SDK | Python | azure-cosmos | 4.7.0 | `app/backend/requirements.txt:1` |
| Identity SDK | Python | azure-identity | 1.18.0 | `app/backend/requirements.txt:2` |
| Document Intel | Python | azure-ai-formrecognizer | 3.3.3 | `functions/requirements.txt:7` |

---

## F. Critical Architectural Notes for APIM Planning

### 1. **No Raw HTTP for Azure Services**
- All Azure service calls use **typed SDK clients** (SearchClient, BlobServiceClient, CosmosClient, etc.)
- **APIM cannot intercept these** - they use Azure's internal service endpoints
- **Implication**: APIM fronts **only** the backend API endpoints (`/chat`, `/upload`, etc.), not downstream Azure calls

### 2. **Streaming Uses SSE (not WebSocket)**
- Backend: `StreamingResponse(r, media_type="application/x-ndjson")`
- Frontend: `ndjson-readablestream` to parse SSE chunks
- **Implication**: APIM policies must support chunked transfer encoding and SSE passthrough

### 3. **Authentication is Azure AD Token-Based**
- Backend uses `get_bearer_token_provider()` for machine-to-machine auth
- Frontend likely uses OAuth 2.0 PKCE (check auth folder)
- **Implication**: APIM must validate Entra ID tokens or pass them through

### 4. **Frontend Proxies to Backend in Dev**
- `vite.config.ts` proxies `/ask` and `/chat` to `http://localhost:5000`
- **Implication**: In production, change `API_BASE_URL` to APIM gateway

### 5. **Enrichment Service is Separate**
- Backend calls `ENRICHMENT_APPSERVICE_URL` for embedding generation
- **Implication**: If enrichment should also go through APIM, need separate route/backend policy

---

## G. Next Steps for APIM Scan

**Priority order**:
1. Map all backend routes (complete list from `app.py`)
2. Identify frontend API call sites (all `fetch()` calls in `api/api.ts`)
3. Verify auth implementation (check `app/backend/auth/` folder)
4. Document env vars for base URL configuration
5. Analyze streaming implementation details
6. Confirm no direct HTTP calls to Azure (validate SDK usage)

---

## H. Key Files for Evidence Deep Dive

1. `app/backend/app.py` (904 lines) - **All backend routes**
2. `app/frontend/src/api/api.ts` (490 lines) - **All frontend API calls**
3. `app/backend/approaches/*.py` - **RAG logic + Azure SDK usage**
4. `app/frontend/vite.config.ts` - **Dev proxy config**
5. `functions/*/(__init__.py | function.json)` - **Worker triggers + bindings**
6. `app/backend/requirements.txt` - **Backend dependencies**
7. `app/frontend/package.json` - **Frontend dependencies**
8. `functions/requirements.txt` - **Worker dependencies**

---

**End of Stack Evidence Report**
