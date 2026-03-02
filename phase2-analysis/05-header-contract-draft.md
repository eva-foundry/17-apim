# Header Contract Draft

**Scope**: HTTP headers for governance, cost tracking, request correlation, and authentication  
**Discovery Method**: Backend source code inspection, API patterns, APIM requirements  
**Generated**: Header specification for APIM injection and backend propagation  

---

## Executive Summary

**Current State**: Backend has **NO header extraction or propagation** (all endpoints accept any headers silently)

**Improved Header Contract**: 7 headers with 5-level cost attribution hierarchy

**Header Structure** (Business-Aligned):
1. `X-Client-Id` - Multi-tenant isolation (ESDC-EI, ESDC-OAS, AICOE)
2. `X-Project-Id` - Project-level tracking within client
3. `X-Phase-Id` - SDLC phase (dev, test, staging, prod)
4. `X-Task-Id` - Granular task/operation tracking
5. `X-User-Id` - Individual user accountability
6. `X-Cost-Center` - Financial attribution
7. `X-Environment` - Deployment environment validation
8. `X-Correlation-Id` - Distributed tracing (auto-generated)

**Injection Points**:
- **Level 1 (APIM)**: Generated/injected at APIM gateway (inbound policy)
- **Level 2 (Backend)**: Extracted by new middleware (requires code addition)
- **Level 3 (Cosmos DB)**: Logged with upload/status records (requires new field)

**Business Value**:
- **Multi-Tenant Ready**: Client-level cost isolation (ESDC-EI: $15K, AICOE: $3K)
- **SDLC Phase Tracking**: Separate dev/test/prod costs (Development: $5K, Production: $10K)
- **Task-Level Optimization**: Identify expensive operations (Query-Processing: 67% of costs)
- **Complete Attribution Chain**: Client → Project → Phase → Task → User → Cost Center

---

## Existing HTTP Headers (Observed in Codebase)

### Response Headers Emitted by Backend

| Header | Emitted By | Value | Evidence |
|--------|-----------|-------|----------|
| `Content-Type` | FastAPI/StreamingResponse | `application/x-ndjson` \| `text/event-stream` \| `application/json` | app.py:293, 818 |
| `Transfer-Encoding` | FastAPI (chunked responses) | `chunked` | Implicit in StreamingResponse |
| `Content-Disposition` | File download handler | `inline; filename={blob_name}` | app.py:927 |
| `Cache-Control` | FastAPI default | `no-cache` | Implicit in FastAPI |

### Request Headers Sent by Frontend

| Header | Sent By | Value | Evidence |
|--------|---------|-------|----------|
| `Content-Type` | Frontend fetch() | `application/json` \| `multipart/form-data` | api.ts:28, 260 |
| `User-Agent` | Browser | Browser version | Implicit |
| `Accept` | (none explicit) | Default browser behavior | Not set in api.ts |

### Observed Gaps

| Header | Use Case | Status | Notes |
|--------|----------|--------|-------|
| `Authorization` | Service-to-service auth | Implicit in SDKs | Not in HTTP headers; in SDK token provider |
| `X-Correlation-Id` | Request tracing | ❌ Missing | Not generated or propagated |
| `X-Request-Id` | Request identification | ❌ Missing | Not generated or propagated |
| `User-Id` / `X-User-Id` | User identification | ❌ Missing | No user auth implemented |

---

## Proposed Header Contract

### Category 1: Cost Attribution Headers (5-Level Hierarchy)

These headers enable **complete cost attribution chain** from client to individual user. APIM sets these based on subscription metadata, JWT claims, or URL patterns.

#### 1.1 X-Client-Id (REQUIRED)

| Property | Value |
|----------|-------|
| **Direction** | APIM → Backend (APIM-authoritative) |
| **Format** | Client identifier |
| **Source** | APIM policy (from subscription name or JWT claim) |
| **Backend Action** | Extract and log for multi-tenant cost isolation |
| **Cosmos Logging** | Add `client_id` field to status log |
| **Example** | `X-Client-Id: ESDC-EI` \| `ESDC-OAS` \| `AICOE` |

**Rationale**: **Level 1 cost attribution** - Multi-tenant isolation enables per-client cost tracking

**Business Value**:
- Query: "Show total costs for ESDC-EI client last month"
- Result: "ESDC-EI: $15,000, ESDC-OAS: $8,000, AICOE: $3,000"

**Implementation Required** (Backend):
```python
@app.middleware("http")
async def extract_correlation_id(request: Request, call_next):
    correlation_id = request.headers.get("X-Correlation-Id", str(uuid.uuid4()))
    request.state.correlation_id = correlation_id
    
    response = await call_next(request)
    response.headers["X-Correlation-Id"] = correlation_id
    return response

# In status logging:
statusLog.upsert_document(
    document_path=path,
    ...,
    correlation_id=request.state.correlation_id  # NEW field
)
```

**Evidence Gap**: Currently not used in app.py; would require middleware addition

---

#### 1.2 X-Project-Id (REQUIRED)

| Property | Value |
|----------|-------|
| **Direction** | APIM → Backend (APIM-authoritative) |
| **Format** | Project identifier within client |
| **Source** | APIM policy (from JWT claim or subscription metadata) |
| **Backend Action** | Extract and log for project-level cost tracking |
| **Cosmos Logging** | Add `project_id` field to status log |
| **Example** | `X-Project-Id: JP-Automation` \| `EVA-Brain` \| `Cost-Analytics` |

**Rationale**: **Level 2 cost attribution** - Project hierarchy under client

**Business Value**:
- Query: "Show ESDC-EI costs broken down by project"
- Result: "JP-Automation: $8K, EVA-Brain: $5K, Cost-Analytics: $2K"

**APIM Configuration**:
```xml
<inbound>
    <set-header name="X-Project-Id" exists-action="skip">
        <value>@(context.Request.Headers.GetValueOrDefault("X-Project-Id", "default"))</value>
    </set-header>
</inbound>
```

**Implementation Required** (Backend):
```python
# In middleware (see 1.1):
project_id = request.headers.get("X-Project-Id", "default")
request.state.project_id = project_id

# In status logging:
statusLog.upsert_document(
    ...,
    project_id=request.state.project_id  # NEW field
)
```

**Evidence Gap**: Currently not used in app.py; would require middleware addition

---

#### 1.3 X-Phase-Id (REQUIRED)

| Property | Value |
|----------|-------|
| **Direction** | APIM → Backend (APIM-authoritative) |
| **Format** | SDLC phase identifier |
| **Source** | APIM policy (from deployment region or environment tag) |
| **Backend Action** | Extract and log for phase-level cost separation |
| **Cosmos Logging** | Add `phase_id` field to status log |
| **Example** | `X-Phase-Id: dev` \| `test` \| `staging` \| `prod` |

**Rationale**: **Level 3 cost attribution** - SDLC phase cost tracking

**Business Value**:
- Query: "Show JP-Automation costs by SDLC phase"
- Result: "Development: $5K, Testing: $2K, Production: $10K"
- Insight: "Production is 2x dev costs - optimize prod queries first"

**APIM Configuration**:
```xml
<inbound>
    <set-header name="X-Phase-Id" exists-action="override">
        <value>@(context.Deployment.ServiceName.Contains("prod") ? "prod" : "dev")</value>
    </set-header>
</inbound>
```

**Implementation Required** (Backend):
```python
# In middleware (see 1.1):
phase_id = request.headers.get("X-Phase-Id", "dev")
request.state.phase_id = phase_id

# In status logging:
statusLog.upsert_document(
    ...,
    phase_id=request.state.phase_id  # NEW field
)
```

**Evidence Gap**: Currently not extracted in app.py; would require middleware addition

---

#### 1.4 X-Task-Id (REQUIRED)

| Property | Value |
|----------|-------|
| **Direction** | APIM → Backend (inferred from URL path) |
| **Format** | Task/operation identifier (lowercase, dash-separated) |
| **Source** | APIM policy (extracted from URL path with 2-segment logic) |
| **Backend Action** | Extract and log for task-level cost optimization |
| **Cosmos Logging** | Add `task_id` field to status log |
| **Example** | `X-Task-Id: chat` \| `sessions-history` \| `file` \| `static-assets` |

**Rationale**: **Level 4 cost attribution** - Identify expensive operations

**Business Value**:
- Query: "Show JP-Automation production costs by task"
- Result: "chat: $6.7K (67%), file: $2K (20%), sessions-history: $0 (13% ops, no AI)"
- Insight: "Chat is 67% of prod costs - optimize vector search first; sessions-history is high-frequency but zero-cost"

**Task-Id Extraction Pattern** (see API-ENDPOINT-VERIFICATION.md for full analysis):
- Simple paths: `/chat` → `chat`, `/file` → `file`, `/health` → `health`
- Nested paths: `/sessions/history` → `sessions-history`, `/api/env` → `api-env` (2-segment extraction)
- Static assets: `/assets/logo.png`, `/index.html` → `static-assets` (grouped)

**APIM Configuration** (Improved with 2-Segment Extraction):
```xml
<inbound>
    <set-header name="X-Task-Id" exists-action="skip">
        <value>@{
            var path = context.Request.Url.Path;
            
            // Handle static assets separately
            if (path.Contains("/assets/") || path.EndsWith(".html") || 
                path.EndsWith(".js") || path.EndsWith(".css") || 
                path.EndsWith(".png") || path.EndsWith(".ico")) {
                return "static-assets";
            }
            
            // Extract first 2 segments for nested paths
            var segments = path.Split(new char[] {'/'}, StringSplitOptions.RemoveEmptyEntries);
            if (segments.Length == 0) {
                return "root";
            } else if (segments.Length == 1) {
                return segments[0];
            } else {
                // Combine first 2 segments: /sessions/history → "sessions-history"
                return segments[0] + "-" + segments[1];
            }
        }</value>
    </set-header>
</inbound>
```

**Extraction Examples**:
- `/chat` → `chat`
- `/file` → `file`
- `/sessions/history` → `sessions-history` ✅ (more granular)
- `/sessions/history/page` → `sessions-history` (loses "page", acceptable)
- `/api/env` → `api-env` ✅ (more granular)
- `/assets/logo.png` → `static-assets` ✅ (grouped)
- `/index.html` → `static-assets` ✅ (grouped)

**Implementation Required** (Backend):
```python
# In middleware (see 1.1):
task_id = request.headers.get("X-Task-Id", "unknown")
request.state.task_id = task_id

# In status logging:
statusLog.upsert_document(
    ...,
    task_id=request.state.task_id  # NEW field
)
```

**Evidence Gap**: Currently not extracted in app.py; would require middleware addition

---

#### 1.5 X-User-Id (REQUIRED)

| Property | Value |
|----------|-------|
| **Direction** | APIM → Backend (from authentication) |
| **Format** | User identifier (email or GUID) |
| **Source** | APIM policy (from JWT claim or OAuth token) |
| **Backend Action** | Extract and log for individual user accountability |
| **Cosmos Logging** | Add `user_id` field to status log |
| **Example** | `X-User-Id: marco.presta@hrsdc-rhdcc.gc.ca` |

**Rationale**: **Level 5 cost attribution** - Individual user accountability

**Business Value**:
- Query: "Show Marco's query costs in JP-Automation production last month"
- Result: "marco.presta@hrsdc-rhdcc.gc.ca: $1,200 (3,500 queries, avg $0.34/query)"
- Insight: "Top 3 users account for 80% of costs - provide usage guidelines"

**APIM Configuration**:
```xml
<inbound>
    <set-header name="X-User-Id" exists-action="override">
        <value>@(context.User.Id)</value>
    </set-header>
</inbound>
```

**Implementation Required** (Backend):
```python
# In middleware (see 1.1):
user_id = request.headers.get("X-User-Id", "anonymous")
request.state.user_id = user_id

# In status logging:
statusLog.upsert_document(
    ...,
    user_id=request.state.user_id  # NEW field
)
```

**Evidence Gap**: Currently not extracted in app.py; would require middleware addition

---

### Category 2: Financial Attribution Headers

These headers enable **cost center mapping** for financial reporting.

#### 2.1 X-Cost-Center (REQUIRED)

| Property | Value |
|----------|-------|
| **Direction** | APIM → Backend (APIM-authoritative) |
| **Format** | Cost center code |
| **Source** | APIM policy (from subscription metadata or JWT claim) |
| **Backend Action** | Extract and log for financial reporting |
| **Cosmos Logging** | Add `cost_center` field |
| **Example** | `X-Cost-Center: ESDC-IITB-AI` \| `ESDC-EI-BENEFITS` |

**Rationale**: **Level 6 cost attribution** - Financial reporting and budget tracking

**Business Value**:
- Query: "Show total AI costs by cost center for FY 2024-25"
- Result: "ESDC-IITB-AI: $45K, ESDC-EI-BENEFITS: $18K, ESDC-OAS-ANALYTICS: $12K"
- Insight: "IITB-AI is 60% of total - align with their budget allocation"

**APIM Configuration**:
```xml
<inbound>
    <set-header name="X-Cost-Center" exists-action="skip">
        <value>@(context.Subscription.Name.Split('-')[1])</value>
    </set-header>
</inbound>
```

**Backend Usage**:
```python
cost_center = request.headers.get("X-Cost-Center", "UNALLOCATED")

# Log with Cosmos DB for downstream cost analysis
cosmos_log_entry["cost_center"] = cost_center
```

**Evidence Gap**: No cost tracking in current implementation

---

#### 2.2 X-Environment (REQUIRED)

| Property | Value |
|----------|-------|
| **Direction** | APIM → Backend (APIM-authoritative) |
| **Format** | Deployment environment identifier |
| **Source** | APIM policy (from deployment region or tag) |
| **Backend Action** | Extract and validate environment consistency |
| **Cosmos Logging** | Add `environment` field |
| **Example** | `X-Environment: dev` \| `test` \| `staging` \| `prod` |

**Rationale**: Environment validation and cost separation by deployment tier

**Business Value**:
- Query: "Show costs by environment to validate dev vs prod separation"
- Result: "Production: $18K, Staging: $4K, Development: $3K, Test: $1K"
- Insight: "Dev+Test+Staging = $8K (31% of prod) - acceptable overhead"

**APIM Configuration**:
```xml
<inbound>
    <set-header name="X-Environment" exists-action="override">
        <value>@(context.Deployment.ServiceName.Contains("prod") ? "prod" : "dev")</value>
    </set-header>
</inbound>
```

**Backend Usage**:
```python
environment = request.headers.get("X-Environment", "unknown")
request.state.environment = environment

# Validate environment consistency:
if ENV.get("ENVIRONMENT") != environment:
    log.warning(f"Environment mismatch: backend={ENV.get('ENVIRONMENT')}, header={environment}")
```

**Evidence Gap**: Currently uses ENV["LOCAL_DEBUG"] but not header-based validation

---

### Category 3: Distributed Tracing Headers

These headers enable **request correlation** across distributed systems.

#### 3.1 X-Correlation-Id (REQUIRED)

| Property | Value |
|----------|-------|
| **Direction** | Client → APIM → Backend → Response (echo) |
| **Format** | UUID v4 or trace ID |
| **Source** | Client generates (if not present, APIM generates) |
| **Backend Action** | Extract, log, and echo in response |
| **Cosmos Logging** | Add `correlation_id` field |
| **Example** | `X-Correlation-Id: 550e8400-e29b-41d4-a716-446655440000` |

**Rationale**: Distributed tracing across APIM → Backend → Azure services

**APIM Configuration**:
```xml
<inbound>
    <set-header name="X-Correlation-Id" exists-action="skip">
        <value>@(context.Request.Headers.GetValueOrDefault("X-Correlation-Id", Guid.NewGuid().ToString()))</value>
    </set-header>
</inbound>
```

**Implementation Required** (Backend):
```python
# In middleware (see 1.1):
correlation_id = request.headers.get("X-Correlation-Id", str(uuid.uuid4()))
request.state.correlation_id = correlation_id

# Echo in response:
response.headers["X-Correlation-Id"] = correlation_id

# In status logging:
statusLog.upsert_document(
    ...,
    correlation_id=request.state.correlation_id  # NEW field
)
```

**Evidence Gap**: Currently not used in app.py; would require middleware addition



---

## Detailed Insertion Points

### Insertion Point 1: HTTP Request Entry (APIM Inbound Policy)

```xml
<!-- apim-policy.xml: Inbound Processing (Improved 8-Header Structure) -->
<policies>
    <inbound>
        <!-- COST ATTRIBUTION HIERARCHY (Levels 1-6) -->
        
        <!-- Level 1: Client-level isolation -->
        <set-header name="X-Client-Id" exists-action="override">
            <value>@(context.Subscription.Name.Split('-')[0])</value>
        </set-header>
        
        <!-- Level 2: Project within client -->
        <set-header name="X-Project-Id" exists-action="skip">
            <value>@(context.Request.Headers.GetValueOrDefault("X-Project-Id", "default"))</value>
        </set-header>
        
        <!-- Level 3: SDLC phase -->
        <set-header name="X-Phase-Id" exists-action="override">
            <value>@(context.Deployment.ServiceName.Contains("prod") ? "prod" : "dev")</value>
        </set-header>
        
        <!-- Level 4: Task/operation -->
        <set-header name="X-Task-Id" exists-action="skip">
            <value>@(context.Request.Url.Path.Split('/')[1])</value>
        </set-header>
        
        <!-- Level 5: User accountability -->
        <set-header name="X-User-Id" exists-action="override">
            <value>@(context.User.Id)</value>
        </set-header>
        
        <!-- Level 6: Cost center -->
        <set-header name="X-Cost-Center" exists-action="skip">
            <value>@(context.Subscription.Name.Split('-')[1])</value>
        </set-header>
        
        <!-- OPERATIONAL HEADERS -->
        
        <!-- Environment validation -->
        <set-header name="X-Environment" exists-action="override">
            <value>@(context.Deployment.ServiceName.Contains("prod") ? "prod" : "dev")</value>
        </set-header>
        
        <!-- Distributed tracing -->
        <set-header name="X-Correlation-Id" exists-action="skip">
            <value>@(context.Request.Headers.GetValueOrDefault("X-Correlation-Id", Guid.NewGuid().ToString()))</value>
        </set-header>
    </inbound>
</policies>
```

---

### Insertion Point 2: Backend Middleware (NEW - Code Addition Required)

```python
# app.py: Add new middleware function
from fastapi import Request
import uuid

@app.middleware("http")
async def extract_cost_attribution_headers(request: Request, call_next):
    """
    Extract APIM cost attribution headers (5-level hierarchy) and operational headers.
    These headers enable complete cost tracking from client to individual user.
    """
    
    # COST ATTRIBUTION HIERARCHY (Levels 1-6)
    client_id = request.headers.get("X-Client-Id", "UNKNOWN")
    project_id = request.headers.get("X-Project-Id", "default")
    phase_id = request.headers.get("X-Phase-Id", "dev")
    task_id = request.headers.get("X-Task-Id", "unknown")
    user_id = request.headers.get("X-User-Id", "anonymous")
    cost_center = request.headers.get("X-Cost-Center", "UNALLOCATED")
    
    # OPERATIONAL HEADERS
    environment = request.headers.get("X-Environment", ENV.get("LOCAL_DEBUG", "false") == "true" and "dev" or "unknown")
    correlation_id = request.headers.get("X-Correlation-Id", str(uuid.uuid4()))
    
    # Store in request state for handlers to access
    request.state.client_id = client_id
    request.state.project_id = project_id
    request.state.phase_id = phase_id
    request.state.task_id = task_id
    request.state.user_id = user_id
    request.state.cost_center = cost_center
    request.state.environment = environment
    request.state.correlation_id = correlation_id
    
    # Process request
    response = await call_next(request)
    
    # Echo correlation headers in response
    response.headers["X-Correlation-Id"] = correlation_id
    
    # Log request with 5-level attribution
    log.info(
        f"Request: {request.method} {request.url.path} | "
        f"Client={client_id} | Project={project_id} | Phase={phase_id} | "
        f"Task={task_id} | User={user_id} | CC={cost_center} | "
        f"Env={environment} | Correlation={correlation_id}"
    )
    
    return response
```

**Placement**: After imports, before route definitions (around line 250)

**Evidence Location**: Would need new code block (not currently present)

---

### Insertion Point 3: Cosmos DB Status Logging (NEW - Schema Update Required)

```python
# In statusLog.upsert_document() calls throughout app.py
# Add new fields for 5-level cost attribution + operational headers

statusLog.upsert_document(
    document_path=path,
    status=status,
    status_classification=status_classification,
    state=state,
    # COST ATTRIBUTION HIERARCHY (Levels 1-6)
    client_id=request.state.client_id,        # Level 1
    project_id=request.state.project_id,      # Level 2
    phase_id=request.state.phase_id,          # Level 3
    task_id=request.state.task_id,            # Level 4
    user_id=request.state.user_id,            # Level 5
    cost_center=request.state.cost_center,    # Level 6
    # OPERATIONAL HEADERS
    environment=request.state.environment,
    correlation_id=request.state.correlation_id,
)
```

**Affected Routes**:
- POST `/file` (upload) → line 879
- POST `/deleteItems` → line 382
- POST `/resubmitItems` → line 418
- POST `/logstatus` → line 527

**Cosmos DB Schema Update**:
```json
{
  "id": "doc-path-1",
  "document_path": "upload/file.pdf",
  "status": "Processing",
  "state": "PROCESSING",
  "timestamp": "2024-01-25T10:30:00Z",
  
  // COST ATTRIBUTION HIERARCHY
  "client_id": "ESDC-EI",
  "project_id": "JP-Automation",
  "phase_id": "prod",
  "task_id": "Query-Processing",
  "user_id": "marco.presta@hrsdc-rhdcc.gc.ca",
  "cost_center": "ESDC-IITB-AI",
  
  // OPERATIONAL HEADERS
  "environment": "prod",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Cost Query Examples** (SQL on Cosmos DB):

```sql
-- Level 1: Client rollup
SELECT c.client_id, 
       SUM(c.openai_tokens) * 0.0015 AS estimated_cost,
       COUNT(1) AS operations
FROM c
WHERE c.timestamp >= '2024-01-01'
GROUP BY c.client_id
-- Result: ESDC-EI: $15K, ESDC-OAS: $8K, AICOE: $3K

-- Level 2: Project comparison within client
SELECT c.project_id, 
       SUM(c.openai_tokens) * 0.0015 AS estimated_cost
FROM c
WHERE c.client_id = 'ESDC-EI'
GROUP BY c.project_id
ORDER BY estimated_cost DESC
-- Result: JP-Automation: $8K, EVA-Brain: $5K, Cost-Analytics: $2K

-- Level 3: SDLC phase cost separation
SELECT c.phase_id, 
       SUM(c.openai_tokens) * 0.0015 AS estimated_cost
FROM c
WHERE c.client_id = 'ESDC-EI' AND c.project_id = 'JP-Automation'
GROUP BY c.phase_id
-- Result: Development: $5K, Testing: $2K, Production: $10K

-- Level 4: Task-level optimization
SELECT c.task_id, 
       SUM(c.openai_tokens) * 0.0015 AS estimated_cost,
       (SUM(c.openai_tokens) * 0.0015) / 
       (SELECT SUM(c2.openai_tokens) * 0.0015 FROM c c2 
        WHERE c2.project_id = 'JP-Automation' AND c2.phase_id = 'prod') AS pct_of_total
FROM c
WHERE c.project_id = 'JP-Automation' AND c.phase_id = 'prod'
GROUP BY c.task_id
ORDER BY estimated_cost DESC
-- Result: Query-Processing: $6.7K (67%), Document-Upload: $2K (20%), User-Auth: $1.3K (13%)

-- Level 5: User accountability
SELECT c.user_id, 
       SUM(c.openai_tokens) * 0.0015 AS estimated_cost,
       COUNT(1) AS queries
FROM c
WHERE c.project_id = 'JP-Automation' AND c.phase_id = 'prod' AND c.task_id = 'Query-Processing'
GROUP BY c.user_id
ORDER BY estimated_cost DESC
-- Result: marco.presta@...: $1.2K (3,500 queries), user2@...: $800 (2,100 queries)

-- Complete attribution chain (all 5 levels)
SELECT TOP 10
       c.client_id,
       c.project_id,
       c.phase_id,
       c.task_id,
       c.user_id,
       SUM(c.openai_tokens) * 0.0015 AS estimated_cost,
       COUNT(1) AS operations
FROM c
WHERE c.timestamp >= '2024-01-01' AND c.timestamp < '2024-02-01'
GROUP BY c.client_id, c.project_id, c.phase_id, c.task_id, c.user_id
ORDER BY estimated_cost DESC
-- Shows: "ESDC-EI | JP-Automation | prod | Query-Processing | marco.presta@... | $1,200 | 3,500"
```

---

## Header Propagation Diagram

```
┌─────────────┐
│ External    │
│ Client      │
│ (Optional:  │
│  X-Project- │
│  Id from    │
│  JWT claim) │
└──────┬──────┘
       │
       │ Request (possibly with X-Project-Id)
       ↓
┌─────────────────────────────────────────────────┐
│ APIM Gateway (Inbound Policy)                   │
│ ─────────────────────────────────────────────── │
│ COST ATTRIBUTION (5 Levels):                    │
│ SET X-Client-Id = subscription prefix           │
│ SET X-Project-Id = JWT or "default"             │
│ SET X-Phase-Id = deployment region              │
│ SET X-Task-Id = URL path extraction             │
│ SET X-User-Id = JWT user claim                  │
│ SET X-Cost-Center = subscription suffix         │
│                                                  │
│ OPERATIONAL:                                     │
│ SET X-Environment = deployment name             │
│ IF X-Correlation-Id missing:                    │
│   → Generate UUID                               │
└──────┬──────────────────────────────────────────┘
       │
       │ Request + 8 Headers
       ↓
┌─────────────────────────────────────────────────┐
│ Backend App (Middleware)                        │
│ ─────────────────────────────────────────────── │
│ @app.middleware("http")                         │
│ EXTRACT 8 headers                               │
│ STORE in request.state                          │
│ LOG attribution chain                           │
│ PASS to route handler                           │
└──────┬──────────────────────────────────────────┘
       │
       │ Route Handler
       ↓
┌─────────────────────────────────────────────────┐
│ Route Handler (e.g., POST /file)                │
│ ─────────────────────────────────────────────── │
│ USE request.state.client_id                     │
│ USE request.state.project_id                    │
│ USE request.state.phase_id                      │
│ USE request.state.task_id                       │
│ USE request.state.user_id                       │
│ USE request.state.cost_center                   │
│ LOG to Cosmos DB with full attribution          │
└──────┬──────────────────────────────────────────┘
       │
       │ Cosmos DB insert
       ↓
┌─────────────────────────────────────────────────┐
│ Cosmos DB (statusdb/statuscontainer)            │
│ ─────────────────────────────────────────────── │
│ {                                               │
│   "document_path": "...",                       │
│   "client_id": "ESDC-EI",                       │
│   "project_id": "JP-Automation",                │
│   "phase_id": "prod",                           │
│   "task_id": "Query-Processing",                │
│   "user_id": "marco.presta@...",                │
│   "cost_center": "ESDC-IITB-AI",                │
│   "environment": "prod",                        │
│   "correlation_id": "UUID",                     │
│   ...                                           │
│ }                                               │
└──────┬──────────────────────────────────────────┘
       │
       │ Response
       ↓
┌─────────────────────────────────────────────────┐
│ Backend Response (Outbound)                     │
│ ─────────────────────────────────────────────── │
│ ECHO X-Correlation-Id                           │
│ (Middleware automatically)                      │
└──────┬──────────────────────────────────────────┘
       │
       │ Response + X-Correlation-Id
       ↓
┌─────────────────────────────────────────────────┐
│ APIM Gateway (Outbound Policy)                  │
│ ─────────────────────────────────────────────── │
│ (Optional: massage response headers)            │
│ PASS through: X-Correlation-Id                  │
└──────┬──────────────────────────────────────────┘
       │
       │ Response
       ↓
┌─────────────────┐
│ Client receives │
│ response with   │
│ correlation ID  │
│ for logging     │
└─────────────────┘
```

---

## Backend Code Implementation Checklist

- [ ] **Add middleware** (see Insertion Point 2)
  - [ ] Import uuid, Request
  - [ ] Define `extract_cost_attribution_headers()` function
  - [ ] Call `@app.middleware("http")` decorator
  - [ ] Place before route definitions (~line 250)

- [ ] **Update status logging** (see Insertion Point 3)
  - [ ] Add 8 header fields to `statusLog.upsert_document()` calls
  - [ ] Update in POST `/file` handler (line ~879)
  - [ ] Update in POST `/deleteItems` handler (line ~382)
  - [ ] Update in POST `/resubmitItems` handler (line ~418)
  - [ ] Update in POST `/logstatus` handler (line ~527)

- [ ] **Update Cosmos DB schema** (optional but recommended)
  - [ ] Add indexing policy for `client_id`, `project_id`, `phase_id`, `task_id`, `user_id`
  - [ ] Add indexing policy for `cost_center`, `environment`, `correlation_id`
  - [ ] Update any queries to include new fields

- [ ] **Test header propagation**
  - [ ] POST request with `X-Client-Id` header from APIM
  - [ ] Verify `X-Correlation-Id` echoed in response
  - [ ] Verify Cosmos DB document contains all 8 fields
  - [ ] Test cost query examples (Level 1-5 attribution)

---

## Separation of Concerns

### Client-Sent Headers (External)

| Header | Sent By | APIM Role | Backend Role |
|--------|---------|-----------|--------------|
| `X-Correlation-Id` | Client (or APIM generates) | Pass through | Extract, echo |
| `X-Cost-Center` | Client (JWT claim) | Extract, set header | Extract, log |
| `X-Project-Id` | Client (JWT claim) | Extract, set header | Extract, log |
| `X-Ingestion-Variant` | Client (optional) | Pass through | Extract, log |

### APIM-Authoritative Headers

| Header | Set By | Backend Role |
|--------|--------|--------------|
| `X-Run-Id` | APIM (inbound policy) | Extract, log, echo |
| `X-Caller-App` | APIM (from subscription) | Extract, log |
| `X-Env` | APIM (from deployment) | Extract, log |

**Key Rule**: Backend should **NOT override** APIM-authoritative headers; only extract and log them.

---

## Example Request/Response Flow

### Request

```http
POST https://apim.azure-api.net/infojp/file HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
X-Correlation-Id: 550e8400-e29b-41d4-a716-446655440000
X-Project-Id: JP-Automation

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="document.pdf"

[PDF file content]
------WebKitFormBoundary
Content-Disposition: form-data; name="file_path"

documents/report.pdf
------WebKitFormBoundary--
```

### APIM Inbound Policy Processing

```
X-Client-Id: ESDC-EI  (APIM sets from subscription "ESDC-EI-Prod")
X-Project-Id: JP-Automation  (passed through from request)
X-Phase-Id: prod  (APIM sets from deployment region)
X-Task-Id: file  (APIM extracts from URL path /file)
X-User-Id: marco.presta@hrsdc-rhdcc.gc.ca  (APIM sets from JWT)
X-Cost-Center: ESDC-IITB-AI  (APIM sets from subscription suffix)
X-Environment: prod  (APIM sets from deployment name)
X-Correlation-Id: 550e8400-e29b-41d4-a716-446655440000  (passed through)
```

### Backend Middleware Processing

```python
request.state.client_id = "ESDC-EI"
request.state.project_id = "JP-Automation"
request.state.phase_id = "prod"
request.state.task_id = "file"
request.state.user_id = "marco.presta@hrsdc-rhdcc.gc.ca"
request.state.cost_center = "ESDC-IITB-AI"
request.state.environment = "prod"
request.state.correlation_id = "550e8400-e29b-41d4-a716-446655440000"
```

### Cosmos DB Log Entry

```json
{
  "id": "doc-20240125-001",
  "document_path": "upload/documents/report.pdf",
  "status": "Processing",
  "state": "QUEUED",
  "timestamp": "2024-01-25T10:30:00Z",
  
  "client_id": "ESDC-EI",
  "project_id": "JP-Automation",
  "phase_id": "prod",
  "task_id": "file",
  "user_id": "marco.presta@hrsdc-rhdcc.gc.ca",
  "cost_center": "ESDC-IITB-AI",
  "environment": "prod",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Correlation-Id: 550e8400-e29b-41d4-a716-446655440000

{
  "message": "File 'report.pdf' uploaded successfully"
}
```

---

## Evidence Summary

| Category | Status | File | Lines | Notes |
|----------|--------|------|-------|-------|
| **Current Headers** | ✅ | app.py | 272-293, 911-927 | Content-Type, Transfer-Encoding, Content-Disposition set |
| **Cost Attribution Headers** | ❌ | app.py | all | NOT extracted, NOT logged - requires middleware (8 headers: client_id, project_id, phase_id, task_id, user_id, cost_center, environment, correlation_id) |
| **Middleware** | ❌ | app.py | 1-904 | No middleware defined; no header extraction for 5-level cost attribution |
| **Cosmos Logging** | ⚠️ Partial | app.py | 380-385, 416-420, 525-530, 879 | Logs path/status but not cost attribution fields (missing 8 fields) |
| **Response Headers** | ⚠️ Partial | app.py | 293, 818, 927 | Some response headers set but no correlation ID echo |

**Business Impact**: Without these headers, **ZERO cost attribution** is possible. Current logging only tracks document paths, not which client/project/phase/task/user incurred costs.

**Next Steps**: 
1. Implement middleware (Insertion Point 2) - **15-30 minutes**
2. Update Cosmos schema with 8 new fields - **30 minutes**
3. Update 4 route handlers with new fields - **1 hour**
4. Deploy APIM policy (Insertion Point 1) - **15 minutes**
5. Test end-to-end attribution - **1 hour**

**Total Implementation Time**: **3-4 hours** for complete 5-level cost attribution
