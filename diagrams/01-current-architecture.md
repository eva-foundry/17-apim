# Current Architecture (No APIM)

**Status**: Evidence-based diagram from Phase 2 analysis  
**Sources**: [01-api-call-inventory.md](../docs/apim-scan/01-api-call-inventory.md), [02-auth-and-identity.md](../docs/apim-scan/02-auth-and-identity.md)

---

## Architecture Diagram

```mermaid
graph TB
    subgraph "Frontend (React + TypeScript)"
        UI[React UI<br/>Chat.tsx]
        API[API Client<br/>api.ts]
    end
    
    subgraph "Backend (FastAPI)"
        APP[app.py<br/>20+ endpoints]
        MIDDLEWARE[No Auth Middleware<br/>❌ All endpoints public]
        APPROACHES[Approaches<br/>chatreadretrieveread.py]
    end
    
    subgraph "Azure Services (Private Endpoints)"
        OPENAI[Azure OpenAI<br/>AsyncAzureOpenAI SDK]
        SEARCH[Azure Search<br/>SearchClient SDK]
        BLOB[Blob Storage<br/>BlobServiceClient SDK]
        COSMOS[Cosmos DB<br/>CosmosClient SDK]
    end
    
    subgraph "Enrichment Service"
        ENRICH[enrichment/app.py<br/>Flask API]
    end
    
    UI -->|fetch POST /chat| API
    API -->|HTTP + JSON| APP
    APP -->|No auth check| MIDDLEWARE
    MIDDLEWARE -->|Route to approach| APPROACHES
    
    APPROACHES -->|SDK call<br/>❌ APIM cannot intercept| OPENAI
    APPROACHES -->|SDK call<br/>❌ APIM cannot intercept| SEARCH
    APPROACHES -->|SDK call<br/>❌ APIM cannot intercept| BLOB
    APPROACHES -->|SDK call<br/>❌ APIM cannot intercept| COSMOS
    APPROACHES -->|HTTP POST<br/>✅ Can intercept| ENRICH
    
    style MIDDLEWARE fill:#ffcccc
    style OPENAI fill:#e6f2ff
    style SEARCH fill:#e6f2ff
    style BLOB fill:#e6f2ff
    style COSMOS fill:#e6f2ff
    style ENRICH fill:#ccffcc
```

---

## Key Observations [FACT]

### Security Gaps
- ❌ **No user authentication** - All endpoints publicly accessible [02-auth-and-identity.md:120-160]
- ❌ **No authorization** - No role-based access control
- ❌ **No rate limiting** - Vulnerable to abuse
- ❌ **No request logging** - Limited observability

### SDK Integration Pattern
- ✅ **Service-to-service auth works** - Managed Identity → Azure services [02-auth-and-identity.md:40-120]
- ❌ **APIM cannot intercept SDK calls** - SDKs bypass HTTP proxies [CRITICAL-FINDINGS-SDK-REFACTORING.md:45-80]
- ✅ **Enrichment service is HTTP** - Can be fronted by APIM [01-api-call-inventory.md:180-220]

### API Surface
- **20+ HTTP endpoints** documented [01-api-call-inventory.md:20-250]
- **3 streaming endpoints** using SSE [04-streaming-analysis.md]
- **7 file management endpoints** [01-api-call-inventory.md:60-120]

---

## Current Request Flow

```
1. User → Browser → React UI
2. React UI → fetch() → api.ts
3. api.ts → HTTP POST /chat → FastAPI app.py
4. app.py → No auth check → route handler
5. Route handler → Approach class (e.g., chatreadretrieveread.py)
6. Approach → Azure SDK calls:
   - AsyncAzureOpenAI.create() → Azure OpenAI
   - SearchClient.search() → Azure Cognitive Search
   - BlobServiceClient.download_blob() → Azure Blob Storage
   - CosmosClient.upsert_item() → Azure Cosmos DB
7. Approach → HTTP POST → Enrichment Service (embeddings)
8. Response streams back to user (SSE/ndjson)
```

**Problem**: No governance layer between steps 3-4. All requests accepted without validation.

---

## Evidence References

- **[FACT: 01-api-call-inventory.md:23-56]** - Chat endpoint flow
- **[FACT: 01-api-call-inventory.md:180-220]** - Azure SDK usage patterns
- **[FACT: 02-auth-and-identity.md:125-145]** - No user auth detected
- **[FACT: CRITICAL-FINDINGS-SDK-REFACTORING.md:65-85]** - SDK bypass APIM constraint
