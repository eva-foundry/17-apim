# ⚠️ CRITICAL FINDINGS: Azure SDK Integration Depth & Refactoring Viability

**Date**: January 25, 2026  
**Investigation Scope**: Complete Azure SDK usage analysis across backend  
**Status**: ✅ **COMPLETE** - DO NOT REFACTOR

---

## Executive Summary

**The codebase has extensive Azure SDK integration (150+ direct SDK calls) that makes refactoring for APIM integration NOT cost-effective.**

- **Refactoring Effort**: 270-410 hours (6-10 weeks)
- **Refactoring Benefit**: Minimal (APIM still cannot intercept SDK calls)
- **Recommended Approach**: Backend middleware + Azure Monitor (1 week, low risk)

---

## 1. Azure SDK Integration Extensiveness

### By SDK Client

| SDK Client | Files | Instances | Method Calls | Severity |
|-----------|-------|-----------|--------------|----------|
| **AsyncAzureOpenAI** | 5 approach files | 5+ | create() | 🔴 Critical |
| **SearchClient** | 3+ files | 6+ | search() | 🔴 Critical |
| **BlobServiceClient** | 10+ files | 20+ | upload/download/delete | 🔴 Critical |
| **CosmosClient** | 6+ files | 50+ | query/upsert/read items | 🔴 Critical |
| **QueueClient** | 2+ files | 4+ | send_message() | 🟡 Medium |

### Files with Deep SDK Integration

**High Integration (>10 SDK calls)**:
- `app/backend/app.py` (2,557 lines) - 40+ SDK calls
  - OpenAI configuration (lines 95-108)
  - Lifespan initialization (lines 430-560)
  - Multiple endpoint handlers (upload, delete, query endpoints)
  
- `app/backend/routers/sessions.py` - 10+ Cosmos calls
- `app/enrichment/app.py` - 15+ Azure SDK calls
- `shared_code/status_log.py` - 8+ Cosmos calls
- `shared_code/user_profile.py` - 5+ Cosmos calls

**Medium Integration (5-10 SDK calls)**:
- `app/backend/approaches/chatreadretrieveread.py` - Search + Blob + Enrichment
- `functions/FileUploadedEtrigger/` - Search + Blob + Queue
- `functions/TextEnrichment/` - Search + Blob + Queue

**Total Integration Points**: 150+ direct SDK method calls

---

## 2. Why Refactoring is NOT Viable

### Problem: SDKs Bypass HTTP Proxies

```
Your Code → CosmosClient SDK → Azure Cosmos DB (Direct Connection)
                ↑
           APIM Cannot Intercept ✗
```

**Critical Limitation**: Azure SDKs:
- Handle authentication internally (bearer tokens, managed identity)
- Establish direct HTTPS connections to Azure services
- Use connection pooling & retry logic at SDK layer
- Do NOT respect HTTP proxy settings for Azure service endpoints

**Result**: APIM cannot:
- ❌ Rate limit downstream Azure service calls
- ❌ Track Azure resource consumption
- ❌ Inject governance headers into Azure SDK calls
- ❌ Intercept or modify Azure SDK communication

### What APIM CAN See

✅ **Frontend → Backend** HTTP calls  
✅ **Enrichment Service** HTTP calls (requests.post, raw HTTP)  
✅ **External APIs** via raw HTTP calls  

❌ **OpenAI SDK calls** (direct connection)  
❌ **Search SDK calls** (direct connection)  
❌ **Blob SDK calls** (direct connection)  
❌ **Cosmos SDK calls** (direct connection)

---

## 3. Refactoring Effort Estimation

### Phase-by-Phase Breakdown

| Phase | Component | Effort | Risk | Timeline | Dependencies |
|-------|-----------|--------|------|----------|--------------|
| **1** | Create Cosmos HTTP abstraction layer | 80-120 hrs | 🔴 High | 2-3 weeks | None |
| **2** | Create Blob Storage HTTP wrapper | 40-60 hrs | 🔴 High | 1-2 weeks | Phase 1 |
| **3** | Rewrite OpenAI token provider | 20-30 hrs | 🟠 Medium | 3-4 days | None |
| **4** | Replace Search SDK with HTTP client | 30-50 hrs | 🟠 Medium | 1 week | None |
| **5** | Update all call sites (50+ locations) | 50-80 hrs | 🔴 High | 2 weeks | Phases 1-4 |
| **6** | Testing & validation (unit, integration, E2E) | 100-150 hrs | 🔴 High | 3 weeks | Phases 1-5 |
| **7** | Deployment & rollback procedures | 20-40 hrs | 🟡 Medium | 1 week | Phases 1-6 |
| **TOTAL** | **Full Refactoring** | **270-410 hours** | 🔴 Very High | **6-10 weeks** | All sequential |

### Risk Assessment

**Technical Risks** (🔴 High):
- Losing SDK retry logic & connection pooling
- Manual authentication token management (expiration, renewal)
- Loss of SDK error handling sophistication
- Breaking changes to business logic

**Operational Risks** (🔴 High):
- Regression testing across 15+ files
- Performance degradation without SDK optimizations
- Maintenance burden (staying in sync with Azure service APIs)
- Difficulty hiring developers familiar with custom wrappers

**Schedule Risks** (🔴 High):
- Estimate likely 15-25% under-budgeted
- Dependencies between phases block parallelization
- Testing discovered issues extend timeline by 2-3 weeks

---

## 4. Cost-Benefit Analysis

### Refactoring Path

| Aspect | Cost | Benefit | ROI |
|--------|------|---------|-----|
| Engineering Time | 270-410 hours | Ability to see Azure SDK calls in APIM | 🔴 Negative |
| Risk to Business | Very High | APIM can still NOT intercept | 🔴 Negative |
| Maintenance Burden | Long-term | Custom HTTP wrappers need updates | 🔴 Negative |
| Timeline Impact | 6-10 weeks | Delays APIM integration by 2 months | 🔴 Negative |

### Recommended Alternative Path

| Aspect | Cost | Benefit | ROI |
|--------|------|---------|-----|
| Engineering Time | 20-30 hours | Full cost tracking via middleware + monitoring | 🟢 Positive |
| Risk to Business | Low | No changes to critical business logic | 🟢 Positive |
| Maintenance Burden | Minimal | Simple middleware pattern | 🟢 Positive |
| Timeline Impact | 1 week | APIM integration on schedule | 🟢 Positive |

---

## 5. Recommended Architecture: Backend Middleware Approach

### ✅ Option A: Backend Middleware Logging (RECOMMENDED)

**Implementation**: Add FastAPI middleware to capture governance headers and log to Application Insights + Cosmos DB

**Effort**: 20-30 hours | **Timeline**: 1 week | **Risk**: Low

**Pattern**:
```python
@app.middleware("http")
async def governance_logging_middleware(request: Request, call_next):
    # 1. Extract APIM-injected headers
    correlation_id = request.headers.get("X-Correlation-ID", str(uuid.uuid4()))
    caller_app = request.headers.get("X-Caller-App", "unknown")
    caller_env = request.headers.get("X-Env", "unknown")
    
    # 2. Store in request state for later use
    request.state.correlation_id = correlation_id
    request.state.caller_app = caller_app
    
    # 3. Process request
    response = await call_next(request)
    
    # 4. Log governance metadata to Cosmos DB
    # This captures caller identity + timestamp + endpoint
    # Later, match against Azure service bills to attribute costs
    
    return response
```

**Flow**:
```
Frontend → APIM (injects headers) → Backend Middleware → Business Logic → Azure Services
                                        ↓
                                    Log to Cosmos DB
                                    Log to Application Insights
```

**Cosmos DB Schema** (for cost tracking):
```json
{
  "id": "request-abc123",
  "timestamp": "2026-01-25T14:30:00Z",
  "correlation_id": "abc123",
  "caller_app": "frontend-react",
  "caller_env": "production",
  "endpoint": "/chat",
  "method": "POST",
  "status_code": 200,
  "response_time_ms": 1234,
  "user_id": "user@company.com",
  "project_id": "proj-789",
  "request_size_bytes": 512,
  "response_size_bytes": 2048
}
```

**Cost Attribution Later** (via Application Insights):
```kusto
AppDependencies  // Azure SDK calls logged by SDK monitoring
| where timestamp >= ago(1d)
| where client_Type == "Other"  // Azure services
| summarize cost = sum(duration) by name  // Estimate based on duration
| join kind=inner (
    requests  // Join with request metadata from Cosmos
    | where timestamp >= ago(1d)
    | project correlation_id, caller_app, caller_env
  ) on correlation_id
```

### ✅ Option B: Azure Monitor Integration (COST-EFFECTIVE)

**Implementation**: Leverage existing Azure Monitor setup + Application Insights for SDK call tracking

**Effort**: 10-15 hours | **Timeline**: 3-4 days | **Risk**: Very Low

**Already Partially Implemented**:
- `app/backend/app.py:3` - `configure_azure_monitor()` already called
- `app/backend/app.py:413` - OpenTelemetry tracing configured
- Just needs configuration refinement

**Implementation**:
```python
# app/backend/app.py (already done, just needs correlation ID propagation)
from azure.monitor.opentelemetry import configure_azure_monitor

configure_azure_monitor(
    connection_string=ENV["APPLICATIONINSIGHTS_CONNECTION_STRING"],
    logger_name="EVA_BACKEND",
    enable_live_metrics=True,
)

# Add correlation ID propagation to OpenTelemetry
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

@app.middleware("http")
async def add_otel_correlation(request: Request, call_next):
    correlation_id = request.headers.get("X-Correlation-ID")
    with tracer.start_as_current_span("http_request") as span:
        span.set_attribute("correlation.id", correlation_id)
        response = await call_next(request)
    return response
```

**Tracking SDKs Automatically**:
- OpenAI SDK calls logged to Application Insights
- Search SDK calls logged to Application Insights
- Blob SDK calls logged to Application Insights
- Cosmos SDK calls logged to Application Insights
- No code changes needed (SDK monitoring is automatic)

---

## 6. APIM Integration Architecture (Recommended)

```
┌─────────────────────────────────────────────────────────────┐
│  Frontend (React)                                           │
│  - Sends requests to APIM                                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  API Management Gateway (APIM)                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Inbound Policies:                                    │  │
│  │ - Extract authentication                             │  │
│  │ - Generate correlation ID                            │  │
│  │ - Inject governance headers:                         │  │
│  │   X-Correlation-ID                                   │  │
│  │   X-Caller-App                                       │  │
│  │   X-Caller-User                                      │  │
│  │   X-Env                                              │  │
│  │   X-Cost-Center                                      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Backend FastAPI (app/backend/app.py)                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Middleware Layer:                                    │  │
│  │ - Extract governance headers from request           │  │
│  │ - Log to Cosmos DB (governance.requests container)  │  │
│  │ - Add correlation_id to OpenTelemetry span          │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Request Handlers:                                    │  │
│  │ - /chat (POST)                                       │  │
│  │ - /upload (POST)                                     │  │
│  │ - /documents (GET/DELETE)                            │  │
│  │ - etc.                                               │  │
│  │                                                      │  │
│  │ (Call SDKs - NOT visible to APIM)                   │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Azure Services (SDKs - Direct Connections)                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │ Azure OpenAI │ │ Cognitive    │ │ Search       │         │
│  │              │ │ Services     │ │              │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │ Blob Storage │ │ Cosmos DB    │ │ Queue        │         │
│  │              │ │              │ │ Storage      │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Monitoring & Cost Tracking                                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │ Cosmos DB    │ │ Application  │ │ Azure Cost   │         │
│  │ Request Logs │ │ Insights     │ │ Management   │         │
│  │              │ │ (SDK calls)  │ │              │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Implementation Roadmap (Recommended)

### Phase 1: Setup (Days 1-2) - 2 person-days
- [ ] Update Cosmos DB schema (add governance.requests container)
- [ ] Configure APIM policies (inbound, add headers)
- [ ] Configure APIM named values (environment variables)

### Phase 2: Middleware (Days 3-5) - 3 person-days
- [ ] Create governance logging middleware
- [ ] Test header extraction & propagation
- [ ] Validate Cosmos DB logging

### Phase 3: Monitoring (Days 6-7) - 2 person-days
- [ ] Configure Application Insights correlation ID propagation
- [ ] Create cost attribution queries (Kusto)
- [ ] Validate SDK call tracking

### Phase 4: Testing & Validation (Days 8-10) - 3 person-days
- [ ] End-to-end test with APIM
- [ ] Validate cost tracking accuracy
- [ ] Performance testing (baseline vs. with middleware)

**Total Timeline**: ~2 weeks  
**Total Effort**: ~10 person-days  
**Risk Level**: Low

---

## 8. Critical Implementation Notes

### ⚠️ Do NOT

❌ Attempt to refactor Azure SDKs to raw HTTP  
❌ Try to make SDKs respect HTTP proxies (not possible)  
❌ Expect APIM to intercept downstream Azure calls  
❌ Replace SDKs with `httpx` or `requests` (loses optimization)

### ✅ Do

✅ Use middleware to capture governance metadata  
✅ Log all relevant context to Cosmos DB  
✅ Leverage Application Insights for SDK monitoring  
✅ Match APIM logs + SDK logs for cost attribution  
✅ Keep SDKs for their optimizations (pooling, retry, auth)

### 🎯 Success Metrics

- Correlation ID flows end-to-end (frontend → APIM → backend → Cosmos)
- All SDK calls visible in Application Insights
- Cost can be attributed per caller via middleware logs
- APIM policies enforce governance headers
- No impact to business logic or performance

---

## 9. Decision: Recommended Path Forward

**RECOMMENDATION: ✅ Backend Middleware Approach (Option A)**

**Rationale**:
1. **Minimal refactoring** - Only add middleware, no SDK changes
2. **Low risk** - Simple middleware pattern, easy to test
3. **Fast delivery** - 1 week vs. 6-10 weeks
4. **Cost tracking works** - Governance logs + SDK monitoring
5. **APIM integration unblocked** - Can proceed immediately

**Alternative if cost is critical**: Option B (Azure Monitor only) - 3-4 days, even lower risk.

**Do NOT use**: Full refactoring - not cost-effective, high risk, unclear benefit.

---

## References

- **APIM Planning**: See [INDEX.md](./INDEX.md) for full documentation index
- **SDK Details**: See [APPENDIX-A-Azure-SDK-Clients.md](./docs/APPENDIX-A-Azure-SDK-Clients.md)
- **Architecture**: See [COMPLETION-REPORT-SDK-SCAN.md](./COMPLETION-REPORT-SDK-SCAN.md)

---

**Status**: ✅ Investigation Complete - Ready for Architecture Review & Implementation Planning
