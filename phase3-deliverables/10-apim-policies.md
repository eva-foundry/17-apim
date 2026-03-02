# APIM Policies Documentation

**Deliverable**: Phase 3B - APIM Policy Design  
**Version**: 1.0.0  
**Date**: February 6, 2026  
**Status**: ✅ Complete - Ready for Implementation

---

## Executive Summary

Comprehensive Azure API Management policy suite providing **6 core capabilities** for the MS-InfoJP RAG system:

1. ✅ **CORS** - Cross-origin resource sharing for frontend applications
2. ✅ **JWT Validation** - Entra ID authentication with claim extraction
3. ✅ **Rate Limiting** - Traffic control (100 req/min per user, 10k req/min per app, 1M req/month quota)
4. ✅ **Header Injection** - Governance tracking (9 headers for cost attribution & observability)
5. ✅ **Streaming Handling** - SSE support with disabled buffering for `/chat`, `/stream`, `/tdstream`
6. ✅ **Logging & Observability** - Application Insights integration with structured logging

**Business Impact**:
- **Cost Attribution**: Complete tracking chain (Client → Project → User → Cost Center)
- **Security**: Entra ID authentication on 36 of 39 endpoints
- **Observability**: Full request/response logging with correlation IDs
- **Performance**: Streaming endpoints optimized with no buffering
- **Governance**: Multi-tenant isolation ready for ESDC-EI, AICOE clients

---

## Table of Contents

1. [Policy Overview](#policy-overview)
2. [Policy 1: CORS Configuration](#policy-1-cors-configuration)
3. [Policy 2: JWT Validation](#policy-2-jwt-validation)
4. [Policy 3: Rate Limiting](#policy-3-rate-limiting)
5. [Policy 4: Header Injection](#policy-4-header-injection)
6. [Policy 5: Streaming Handling](#policy-5-streaming-handling)
7. [Policy 6: Logging & Observability](#policy-6-logging--observability)
8. [Error Handling](#error-handling)
9. [Policy Testing Plan](#policy-testing-plan)
10. [Configuration Requirements](#configuration-requirements)
11. [Deployment Checklist](#deployment-checklist)

---

## Policy Overview

### Policy Execution Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ INBOUND PHASE                                                   │
├─────────────────────────────────────────────────────────────────┤
│ 1. CORS Preflight → Allow ESDC/HRSDC origins                   │
│ 2. JWT Validation → Verify Entra ID token (skip for 3 public)  │
│ 3. Rate Limiting → 100/min per user, 10k/min per app          │
│ 4. Header Injection → Add 9 governance headers                 │
│ 5. Streaming Detection → Flag SSE endpoints                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ BACKEND PHASE                                                   │
├─────────────────────────────────────────────────────────────────┤
│ • Forward request with streaming config (if applicable)        │
│ • Timeout: 600s for streaming, 120s for standard              │
│ • Buffering: Disabled for SSE, enabled for standard           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ OUTBOUND PHASE                                                  │
├─────────────────────────────────────────────────────────────────┤
│ 1. Add Response Timing → X-Response-Time header               │
│ 2. Log to Application Insights → Structured JSON metadata     │
│ 3. Debug Headers → Add X-APIM-Trace-Enabled (dev/staging)     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ ON-ERROR PHASE (if errors occur)                               │
├─────────────────────────────────────────────────────────────────┤
│ • Log error details to Application Insights                    │
│ • Return standardized error response (401/429/403/500)        │
│ • Include correlation ID for support troubleshooting          │
└─────────────────────────────────────────────────────────────────┘
```

### Policy Scope

**Applies to**: All 39 endpoints in OpenAPI spec  
**Exceptions**:
- `/health` - Public health check (no JWT required)
- `/api/env` - Public environment config (no JWT required)
- `/getFeatureFlags` - Public feature flags (no JWT required)

---

## Policy 1: CORS Configuration

### Purpose

Enable frontend applications from ESDC/HRSDC domains to make cross-origin requests to the APIM gateway.

### Allowed Origins

| Environment | Domain | Purpose |
|-------------|--------|---------|
| **Production** | `https://infojp.hrsdc-rhdcc.gc.ca` | Primary production frontend |
| **Production** | `https://infojp.esdc.gc.ca` | Alternate production domain |
| **Staging** | `https://infojp-stg.hrsdc-rhdcc.gc.ca` | Staging environment |
| **Staging** | `https://infojp-stg.esdc.gc.ca` | Alternate staging domain |
| **Development** | `https://infojp-dev.hrsdc-rhdcc.gc.ca` | Development environment |
| **Development** | `https://infojp-dev.esdc.gc.ca` | Alternate dev domain |
| **Local Dev** | `http://localhost:5173` | Vite dev server (primary) |
| **Local Dev** | `http://localhost:5174` | Vite dev server (alternate) |

### Allowed Methods

- `GET` - Read operations
- `POST` - Create/submit operations
- `PUT` - Update operations
- `DELETE` - Delete operations
- `OPTIONS` - CORS preflight

### Allowed Headers

| Header | Purpose |
|--------|---------|
| `Content-Type` | Request body MIME type |
| `Authorization` | JWT Bearer token |
| `Accept` | Response MIME type preference |
| `X-Correlation-Id` | Distributed tracing |
| `X-Request-Id` | Request identification |
| `X-Project-Id` | Project attribution |
| `X-Cost-Center` | Cost tracking |
| `X-Caller-App` | Client identification |

### Exposed Headers

| Header | Purpose |
|--------|---------|
| `X-Correlation-Id` | Echo correlation ID for client logging |
| `X-Run-Id` | Backend execution tracking |
| `X-Response-Time` | Request latency (ms) |

### Testing CORS

```powershell
# Preflight request (OPTIONS)
curl -X OPTIONS http://apim-gateway-url/chat `
  -H "Origin: https://infojp.hrsdc-rhdcc.gc.ca" `
  -H "Access-Control-Request-Method: POST" `
  -H "Access-Control-Request-Headers: Content-Type, Authorization" `
  -v

# Expected Response Headers:
# Access-Control-Allow-Origin: https://infojp.hrsdc-rhdcc.gc.ca
# Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
# Access-Control-Allow-Headers: Content-Type, Authorization, ...
# Access-Control-Allow-Credentials: true
```

---

## Policy 2: JWT Validation

### Purpose

Authenticate users via Azure AD (Entra ID) JWT tokens to secure 36 of 39 endpoints.

### Authentication Flow

```
1. Frontend → MSAL authentication → Acquire JWT token
2. Frontend → API request with "Authorization: Bearer <jwt>"
3. APIM → Validate JWT signature, audience, expiration
4. APIM → Extract claims (oid, email, groups)
5. APIM → Inject X-User-Id header with oid claim
6. APIM → Forward to backend with headers
```

### JWT Validation Configuration

| Parameter | Value |
|-----------|-------|
| **OpenID Config URL** | `https://login.microsoftonline.com/bfb12ca1-7f37-47d5-9cf5-8aa52214a0d8/v2.0/.well-known/openid-configuration` |
| **Tenant ID** | `bfb12ca1-7f37-47d5-9cf5-8aa52214a0d8` (ESDC/HRSDC) |
| **Expected Audiences** | `api://infojp-backend-prod`, `api://infojp-backend-stg`, `api://infojp-backend-dev` |
| **Token Header** | `Authorization` (Bearer scheme) |
| **Clock Skew** | 300 seconds (5 minutes) |
| **Require Expiration** | Yes |
| **Require Signed Tokens** | Yes |

### Public Endpoints (No JWT Required)

| Endpoint | Reason |
|----------|--------|
| `/health` | Infrastructure health checks |
| `/api/env` | Environment configuration (non-sensitive) |
| `/getFeatureFlags` | Feature flags (non-sensitive) |

### JWT Claim Extraction

**Extracted Claims** (injected as headers):

1. **oid** (Object ID) → `X-User-Id`
   - Azure AD user identifier (GUID)
   - Example: `9f540c2e-a8b7-4c3d-9e1f-2a3b4c5d6e7f`

2. **email** → Available in JWT payload
   - User email address
   - Example: `marco.presta@hrsdc-rhdcc.gc.ca`

3. **groups** → Available in JWT payload (optional claim)
   - RBAC group memberships (array of GUIDs)
   - Example: `["group-guid-1", "group-guid-2"]`

4. **costCenter** → `X-Cost-Center` (custom claim, optional)
   - Cost center for chargeback
   - Example: `AICOE-EVA`

### Testing JWT Validation

```powershell
# 1. Acquire JWT token (via MSAL or Azure CLI)
$token = az account get-access-token --resource api://infojp-backend-dev --query accessToken -o tsv

# 2. Test protected endpoint WITH token
curl -X GET http://apim-gateway-url/getUsrGroupInfo `
  -H "Authorization: Bearer $token" `
  -v

# Expected: 200 OK with response body

# 3. Test protected endpoint WITHOUT token
curl -X GET http://apim-gateway-url/getUsrGroupInfo `
  -v

# Expected: 401 Unauthorized
# {
#   "error": "Unauthorized",
#   "message": "Invalid or expired JWT token. Please authenticate and try again.",
#   "correlation_id": "...",
#   "timestamp": "2026-02-06T..."
# }

# 4. Test public endpoint WITHOUT token
curl -X GET http://apim-gateway-url/health `
  -v

# Expected: 200 OK (no JWT required)
```

### Configuration Requirements

**Before Deployment**:
1. **Create App Registrations** in Entra ID (Azure AD):
   - `infojp-backend-prod` (Production)
   - `infojp-backend-stg` (Staging)
   - `infojp-backend-dev` (Development)

2. **Configure Audiences** in app registrations:
   - Set `api://infojp-backend-{env}` as Application ID URI

3. **Configure Optional Claims** (recommended):
   - `email` - User email address
   - `groups` - RBAC group memberships
   - `costCenter` - Custom claim (if using directory extensions)

4. **Update Policy** with actual Client IDs:
   - Replace placeholder audiences in `<audiences>` section
   - Line 94-98 in `10-apim-policies.xml`

---

## Policy 3: Rate Limiting

### Purpose

Control API traffic to prevent abuse, ensure fair usage, and protect backend infrastructure.

### Rate Limit Tiers

| Tier | Limit | Renewal Period | Counter Key | Scope |
|------|-------|----------------|-------------|-------|
| **Per-User** | 100 requests | 60 seconds (1 minute) | JWT `oid` claim | Individual user |
| **Per-App** | 10,000 requests | 60 seconds (1 minute) | Subscription ID | Application/client |
| **Monthly Quota** | 1,000,000 requests | 2,592,000 seconds (30 days) | Subscription ID | Application/client |

### Rate Limit Response

When rate limit exceeded, APIM returns:

**Status**: `429 Too Many Requests`

**Headers**:
```
Retry-After: 60
X-RateLimit-Remaining: 0
X-RateLimit-Limit: 100
X-Correlation-Id: <correlation-id>
```

**Body**:
```json
{
  "error": "TooManyRequests",
  "message": "Rate limit exceeded. Please retry after 60 seconds.",
  "retry_after": "60",
  "correlation_id": "<correlation-id>",
  "timestamp": "2026-02-06T15:30:00Z"
}
```

### Rate Limit Monitoring

**Application Insights Queries**:

```kusto
// Users hitting rate limits
ApiManagementGatewayLogs
| where ResponseCode == 429
| summarize HitCount = count() by UserId = tostring(customDimensions["user_id"]), bin(TimeGenerated, 1h)
| order by HitCount desc

// Subscription quota usage
ApiManagementGatewayLogs
| where TimeGenerated > ago(30d)
| summarize RequestCount = count() by SubscriptionId = tostring(customDimensions["subscription_id"])
| extend QuotaUsagePercent = (RequestCount * 100.0) / 1000000
| order by QuotaUsagePercent desc
```

### Testing Rate Limits

```powershell
# Test per-user rate limit (100 req/min)
$token = az account get-access-token --resource api://infojp-backend-dev --query accessToken -o tsv

# Send 105 requests in < 60 seconds
for ($i=1; $i -le 105; $i++) {
  $response = curl -X GET http://apim-gateway-url/getUsrGroupInfo `
    -H "Authorization: Bearer $token" `
    -H "X-Correlation-Id: test-ratelimit-$i" `
    -w "%{http_code}" `
    -o /dev/null `
    -s
  
  Write-Host "Request $i : $response"
  
  if ($response -eq 429) {
    Write-Host "Rate limit hit at request $i" -ForegroundColor Yellow
    break
  }
}

# Expected: Requests 1-100 return 200, requests 101+ return 429
```

### Bypass Rate Limits (if needed)

For internal/admin users, create a separate APIM subscription with higher limits:

```xml
<!-- In subscription-level policy override -->
<inbound>
  <rate-limit-by-key calls="10000" renewal-period="60" counter-key="@(context.Subscription?.Id)" />
  <quota-by-key calls="100000000" renewal-period="2592000" counter-key="@(context.Subscription?.Id)" />
</inbound>
```

---

## Policy 4: Header Injection

### Purpose

Inject governance and cost attribution headers for observability, tracing, and financial tracking.

### Injected Headers (9 total)

| # | Header | Source | Value Example | Purpose |
|---|--------|--------|---------------|---------|
| 1 | `X-Correlation-Id` | APIM (generated if missing) | `3e4d5f6a-7b8c-9d0e-1f2a-3b4c5d6e7f8a` | Distributed tracing across services |
| 2 | `X-Run-Id` | APIM (always generated) | `1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d` | Unique execution identifier |
| 3 | `X-Caller-App` | Subscription name or default | `InfoJP-Frontend` | Client application identifier |
| 4 | `X-Env` | APIM service name | `production`, `staging`, `development`, `sandbox` | Deployment environment |
| 5 | `X-User-Id` | JWT `oid` claim | `9f540c2e-a8b7-4c3d-9e1f-2a3b4c5d6e7f` | User accountability |
| 6 | `X-Cost-Center` | Subscription metadata or JWT | `AICOE-EVA` | Financial attribution |
| 7 | `X-Project-Id` | Subscription metadata | `MS-InfoJP` | Project attribution |
| 8 | `X-Ingestion-Variant` | Header or query param | `default`, `experimental`, `v2` | A/B testing flag |
| 9 | `X-Request-Timestamp` | APIM (generated) | `2026-02-06T15:30:00.123Z` | Request arrival time (ISO 8601) |

### Header Propagation Flow

```
┌──────────────┐
│   Frontend   │
│  (React SPA) │
└──────┬───────┘
       │ Authorization: Bearer <jwt>
       │ X-Correlation-Id: <optional-client-generated>
       ↓
┌──────────────┐
│     APIM     │ ← JWT Validation
│   Gateway    │ ← Header Injection (9 headers)
└──────┬───────┘
       │ Authorization: Bearer <jwt>
       │ X-Correlation-Id: <propagated-or-generated>
       │ X-Run-Id: <generated>
       │ X-User-Id: <extracted-from-jwt>
       │ X-Caller-App: InfoJP-Frontend
       │ X-Env: production
       │ X-Cost-Center: AICOE-EVA
       │ X-Project-Id: MS-InfoJP
       │ X-Ingestion-Variant: default
       │ X-Request-Timestamp: 2026-02-06T15:30:00.123Z
       ↓
┌──────────────┐
│   Backend    │ ← Middleware extracts headers
│ (FastAPI)    │ ← Logs to Cosmos DB governance_requests
└──────────────┘
```

### Cost Attribution Hierarchy

**5-Level Attribution Chain**:

```
Client (X-Caller-App)
  └─ Project (X-Project-Id)
      └─ Environment (X-Env)
          └─ User (X-User-Id)
              └─ Cost Center (X-Cost-Center)
```

**Business Value Queries**:

1. **Client-level costs**:
   ```kusto
   GovernanceRequests
   | summarize TotalCost = sum(estimated_cost) by caller_app
   | order by TotalCost desc
   ```
   Result: `ESDC-EI: $15K, AICOE: $3K`

2. **Project-level costs**:
   ```kusto
   GovernanceRequests
   | where caller_app == "ESDC-EI"
   | summarize TotalCost = sum(estimated_cost) by project_id
   | order by TotalCost desc
   ```
   Result: `JP-Automation: $8K, EVA-Brain: $5K`

3. **User-level costs** (accountability):
   ```kusto
   GovernanceRequests
   | summarize TotalRequests = count(), TotalCost = sum(estimated_cost) by user_id
   | order by TotalCost desc
   ```
   Result: `user-A: 15K requests ($5K), user-B: 8K requests ($2K)`

### Header Protection

**Client Cannot Override** (APIM strips and regenerates):
- `X-User-Id` - Always extracted from JWT (security)
- `X-Run-Id` - Always APIM-generated (uniqueness)
- `X-Env` - Always APIM-authoritative (trust)

**Client Can Provide** (APIM preserves if present):
- `X-Correlation-Id` - Frontend can generate for end-to-end tracing
- `X-Ingestion-Variant` - Frontend can flag A/B test variants
- `X-Project-Id` - Frontend can specify project (if not in subscription)

### Testing Header Injection

```powershell
# Test header injection
$token = az account get-access-token --resource api://infojp-backend-dev --query accessToken -o tsv

curl -X GET http://apim-gateway-url/health `
  -H "Authorization: Bearer $token" `
  -H "X-Correlation-Id: test-correlation-123" `
  -v 2>&1 | Select-String "X-"

# Expected response headers:
# X-Correlation-Id: test-correlation-123  (preserved from request)
# X-Run-Id: <apim-generated-guid>
# X-Response-Time: 45

# Check backend logs for injected headers
curl -X POST http://backend-url/chat `
  -H "Authorization: Bearer $token" `
  -H "Content-Type: application/json" `
  -d '{"question":"test"}' `
  -v 2>&1 | Select-String "X-"

# Expected: All 9 headers present in backend request
```

---

## Policy 5: Streaming Handling

### Purpose

Optimize Server-Sent Events (SSE) and chunked responses by disabling APIM buffering for real-time streaming endpoints.

### Streaming Endpoints

| Endpoint | Response Type | Use Case |
|----------|---------------|----------|
| `/chat` | `application/x-ndjson` | Main chat with hybrid search (newline-delimited JSON) |
| `/stream` | `text/event-stream` | SSE streaming for chat responses |
| `/tdstream` | `text/event-stream` | SSE streaming for technical debt analysis |

### Buffering Configuration

**Streaming Endpoints** (`/chat`, `/stream`, `/tdstream`):
- `buffer-request-body="false"` - Do not buffer incoming request
- `buffer-response="false"` - Do not buffer outgoing response
- `timeout="600"` - 10 minute timeout (long-running queries)
- `Cache-Control: no-cache` - Prevent caching
- `X-Accel-Buffering: no` - Disable nginx buffering (if using nginx reverse proxy)

**Non-Streaming Endpoints** (all others):
- Standard buffering enabled
- `timeout="120"` - 2 minute timeout
- Standard caching headers

### Streaming Detection Logic

```csharp
var path = context.Request.Url.Path.ToLowerInvariant();
bool isStreaming = path.Equals("/chat") || 
                   path.Equals("/stream") || 
                   path.Equals("/tdstream");

if (isStreaming) {
  // Apply streaming optimizations
}
```

### SSE Response Format

**Server-Sent Events Structure**:

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"type":"start","run_id":"..."}

data: {"type":"content","token":" Hello"}

data: {"type":"content","token":" world"}

data: {"type":"end","latency_ms":2345}
```

**NDJSON Streaming** (`/chat`):

```
Content-Type: application/x-ndjson

{"type":"start","correlation_id":"..."}\n
{"type":"search_results","documents":[...]}\n
{"type":"answer_chunk","content":"Hello"}\n
{"type":"answer_chunk","content":" world"}\n
{"type":"finished","latency_ms":3456}\n
```

### Testing Streaming

```powershell
# Test SSE streaming (/stream)
curl -N http://apim-gateway-url/stream `
  -H "Authorization: Bearer $token" `
  -H "Accept: text/event-stream"

# Expected: Real-time events (no buffering)
# data: {"type":"start"}
# data: {"type":"content","token":"..."}
# ...

# Test NDJSON streaming (/chat)
curl -N http://apim-gateway-url/chat `
  -H "Authorization: Bearer $token" `
  -H "Content-Type: application/json" `
  -H "Accept: application/x-ndjson" `
  -d '{"question":"What is EI eligibility?"}'

# Expected: Newline-delimited JSON chunks (real-time)
```

### Troubleshooting Streaming

**Symptom**: Streaming response arrives all at once (buffered)

**Causes & Solutions**:
1. **APIM Buffering**: Verify `buffer-response="false"` in policy
2. **nginx Reverse Proxy**: Set `X-Accel-Buffering: no` header
3. **Load Balancer**: Check ALB/NLB buffering settings (Azure Load Balancer, App Gateway)
4. **Backend Timeout**: Ensure backend sends data within timeout window
5. **Client Buffering**: Frontend must use streaming fetch API (`response.body.getReader()`)

---

## Policy 6: Logging & Observability

### Purpose

Capture comprehensive request/response metadata to Application Insights for monitoring, debugging, and cost analysis.

### Logged Metadata (20 fields)

| Field | Source | Example | Purpose |
|-------|--------|---------|---------|
| `timestamp` | APIM | `2026-02-06T15:30:00Z` | Request arrival time |
| `correlation_id` | Header | `3e4d5f6a-...` | Distributed tracing |
| `run_id` | Header | `1a2b3c4d-...` | Execution identifier |
| `user_id` | Header (JWT) | `9f540c2e-...` | User accountability |
| `subscription_id` | APIM | `sub-12345` | APIM subscription |
| `subscription_name` | APIM | `InfoJP-Prod` | Client/app name |
| `caller_app` | Header | `InfoJP-Frontend` | Application identifier |
| `environment` | Header | `production` | Deployment environment |
| `cost_center` | Header | `AICOE-EVA` | Cost attribution |
| `project_id` | Header | `MS-InfoJP` | Project tracking |
| `ingestion_variant` | Header | `default` | A/B testing flag |
| `request_method` | APIM | `POST` | HTTP method |
| `request_url` | APIM | `/chat` | Endpoint path |
| `request_query` | APIM | `?variant=v2` | Query parameters |
| `response_status` | APIM | `200` | HTTP status code |
| `response_time_ms` | APIM | `2345.67` | Latency (milliseconds) |
| `backend_service` | APIM | `infojp-backend-prod` | Backend hostname |
| `api_version` | Static | `1.0.0` | API version |
| `apim_region` | APIM | `canadacentral` | APIM deployment region |
| `is_streaming` | Variable | `true` | Streaming endpoint flag |

### Application Insights Integration

**Log Destination**: Azure Monitor > Application Insights

**Query Examples**:

```kusto
// 1. Request latency by endpoint
ApiManagementGatewayLogs
| extend response_time = todouble(customDimensions["response_time_ms"])
| summarize 
    AvgLatency = avg(response_time), 
    P50 = percentile(response_time, 50),
    P95 = percentile(response_time, 95),
    P99 = percentile(response_time, 99),
    RequestCount = count()
  by Endpoint = tostring(customDimensions["request_url"])
| order by P95 desc

// 2. Error rate by endpoint
ApiManagementGatewayLogs
| extend status = toint(customDimensions["response_status"])
| summarize 
    TotalRequests = count(),
    ErrorRequests = countif(status >= 400),
    ErrorRate = round(100.0 * countif(status >= 400) / count(), 2)
  by Endpoint = tostring(customDimensions["request_url"])
| where ErrorRate > 0
| order by ErrorRate desc

// 3. Cost attribution by client
ApiManagementGatewayLogs
| extend 
    caller = tostring(customDimensions["caller_app"]),
    project = tostring(customDimensions["project_id"]),
    cost_center = tostring(customDimensions["cost_center"])
| summarize 
    TotalRequests = count(),
    EstimatedCost = count() * 0.01  // $0.01 per request assumption
  by caller, project, cost_center
| order by TotalRequests desc

// 4. User activity tracking
ApiManagementGatewayLogs
| extend user_id = tostring(customDimensions["user_id"])
| where user_id != "anonymous"
| summarize 
    TotalRequests = count(),
    UniqueEndpoints = dcount(tostring(customDimensions["request_url"])),
    AvgLatency = avg(todouble(customDimensions["response_time_ms"]))
  by user_id
| order by TotalRequests desc

// 5. Streaming endpoint performance
ApiManagementGatewayLogs
| extend is_streaming = tobool(customDimensions["is_streaming"])
| where is_streaming == true
| extend response_time = todouble(customDimensions["response_time_ms"])
| summarize 
    StreamingRequests = count(),
    AvgStreamTime = avg(response_time),
    MaxStreamTime = max(response_time)
  by Endpoint = tostring(customDimensions["request_url"])
```

### Sensitive Data Redaction

**Automatically Redacted** (via Application Insights configuration):
- `Authorization` header value (JWT tokens)
- `X-API-Key` header value (if used)
- Request/response bodies (not logged by default)

**Not Redacted** (metadata only):
- Header names (e.g., "Authorization" name is logged, value is not)
- Correlation IDs, User IDs (GUIDs are non-sensitive)
- Response status codes, latency metrics

### Monitoring Dashboards

**Recommended Dashboard Widgets**:

1. **Request Volume** (Line chart)
   - Requests/minute over time
   - Color by environment (dev/staging/prod)

2. **Error Rate** (Area chart)
   - 4xx/5xx errors over time
   - Alert if > 1% error rate

3. **Latency Distribution** (Histogram)
   - P50, P95, P99 latencies
   - By endpoint

4. **Cost Attribution** (Pie chart)
   - Request count by client (caller_app)
   - Request count by project_id

5. **Top Users** (Table)
   - User ID, Total Requests, Avg Latency

6. **Rate Limit Violations** (Single stat)
   - Count of 429 responses (last 1 hour)

---

## Error Handling

### Purpose

Provide consistent, user-friendly error responses with correlation IDs for support troubleshooting.

### Error Response Format

**Standard Error Schema** (all errors):

```json
{
  "error": "ErrorType",
  "message": "Human-readable description of the error",
  "correlation_id": "3e4d5f6a-7b8c-9d0e-1f2a-3b4c5d6e7f8a",
  "timestamp": "2026-02-06T15:30:00.123Z",
  "support_reference": "Quote this correlation_id when contacting support" (optional)
}
```

### Error Types

| HTTP Status | Error Type | Message | Cause |
|-------------|-----------|---------|-------|
| **401** | `Unauthorized` | Invalid or expired JWT token. Please authenticate and try again. | JWT validation failed |
| **403** | `QuotaExceeded` | Monthly request quota exceeded. Please contact support to increase your quota. | Monthly quota (1M requests) exceeded |
| **429** | `TooManyRequests` | Rate limit exceeded. Please retry after {retry_after} seconds. | Per-user (100 req/min) or per-app (10k req/min) limit exceeded |
| **500** | `InternalServerError` | An unexpected error occurred. Please try again later or contact support if the issue persists. | Gateway or backend failure |

### Error Logging

All errors are logged to Application Insights with:

```json
{
  "timestamp": "2026-02-06T15:30:00.123Z",
  "correlation_id": "...",
  "run_id": "...",
  "user_id": "...",
  "error_source": "validate-jwt",
  "error_reason": "TokenExpired",
  "error_message": "JWT token has expired",
  "error_scope": "inbound",
  "error_section": "authentication",
  "request_method": "POST",
  "request_url": "/chat",
  "response_status": 401,
  "response_time_ms": 45
}
```

### Support Troubleshooting

**User reports error**:
1. Ask for `correlation_id` from error response
2. Query Application Insights:
   ```kusto
   ApiManagementGatewayLogs
   | where customDimensions["correlation_id"] == "3e4d5f6a-7b8c-..."
   | project timestamp, request_url, response_status, error_reason, error_message
   ```
3. Investigate root cause from error metadata

---

## Policy Testing Plan

### Phase 1: Unit Testing (Per-Policy)

| Policy | Test Case | Expected Result | How to Test |
|--------|-----------|-----------------|-------------|
| **CORS** | Preflight from allowed origin | 200 OK with CORS headers | `curl -X OPTIONS` with Origin header |
| **CORS** | Preflight from disallowed origin | 403 Forbidden | `curl -X OPTIONS` with invalid Origin |
| **JWT** | Valid JWT token | 200 OK, request forwarded | `curl` with valid token |
| **JWT** | Missing JWT token | 401 Unauthorized | `curl` without Authorization header |
| **JWT** | Expired JWT token | 401 Unauthorized | `curl` with expired token |
| **JWT** | Public endpoint without token | 200 OK | `curl /health` without token |
| **Rate Limit** | 101st request in 1 minute | 429 Too Many Requests | Loop 101 requests |
| **Rate Limit** | Check rate limit headers | `X-RateLimit-Remaining` present | Inspect response headers |
| **Header Injection** | Check all 9 headers present | All headers in backend logs | Backend `/debug/headers` endpoint |
| **Header Injection** | Client override protection | X-User-Id regenerated | Send fake X-User-Id, verify replaced |
| **Streaming** | /chat streaming response | Real-time chunks, no buffering | `curl -N /chat` |
| **Streaming** | /stream SSE | SSE events stream | `curl -N /stream` |
| **Streaming** | Non-streaming endpoint | Standard buffered response | `curl /getUsrGroupInfo` |
| **Logging** | Request logged to App Insights | Log entry in Application Insights | Query App Insights after request |
| **Error** | 401 error format | JSON error response with correlation_id | Trigger 401, check response body |
| **Error** | 429 error format | JSON with retry_after | Trigger rate limit, check response |

### Phase 2: Integration Testing (End-to-End)

| Test Scenario | Steps | Success Criteria |
|---------------|-------|------------------|
| **Full Request Flow** | Frontend → APIM → Backend → Response | Request completes in < 5s, all headers present |
| **Streaming Chat** | POST /chat with SSE | Real-time tokens stream, no buffering delay |
| **Rate Limit Enforcement** | Send 105 requests in 1 minute | Requests 1-100 succeed, 101+ return 429 |
| **Multi-User Isolation** | User A and User B send 60 requests each | Both succeed (rate limit per user, not shared) |
| **Cost Attribution** | Send requests with different X-Project-Id | Cosmos DB logs show correct project_id |
| **Error Recovery** | Trigger 401, then send valid token | Second request succeeds after first fails |
| **CORS Preflight** | OPTIONS request from frontend | CORS headers allow subsequent POST |

### Phase 3: Load Testing

| Test | Configuration | Expected Result | Tool |
|------|---------------|-----------------|------|
| **Sustained Load** | 1000 req/min for 10 minutes | < 1% error rate, P95 latency < 3s | JMeter, k6 |
| **Burst Load** | 5000 req/min for 1 minute | Rate limits enforced, no gateway crashes | Locust |
| **Streaming Load** | 100 concurrent SSE connections | All connections stable, no buffering | Custom test script |
| **Rate Limit Accuracy** | 100 users x 100 req/min | Each user limited to 100 req/min | k6 with user simulation |

---

## Configuration Requirements

### Prerequisites

Before deploying APIM policies:

1. **Azure AD / Entra ID Configuration**:
   - ✅ Create App Registrations (prod, staging, dev)
   - ✅ Configure Application ID URIs (`api://infojp-backend-{env}`)
   - ✅ Enable optional claims (email, groups, costCenter)
   - ✅ Create RBAC groups in Azure AD

2. **Azure API Management Resource**:
   - ✅ APIM instance created (Standard or Premium tier for multiple regions)
   - ✅ API imported from OpenAPI spec (`09-openapi-spec.json`)
   - ✅ Backend service configured (FastAPI backend URL)
   - ✅ Subscriptions created for clients

3. **Application Insights**:
   - ✅ Application Insights resource created
   - ✅ APIM diagnostic settings configured to send logs to AppInsights
   - ✅ Configure log retention (90 days recommended)

4. **Policy Configuration**:
   - ✅ Update JWT `<audiences>` with actual Client IDs (lines 94-98)
   - ✅ Configure subscription metadata (cost-center, project-id)
   - ✅ Update CORS allowed origins with actual frontend domains
   - ✅ Configure backend service URL in APIM

### Subscription Metadata Configuration

**Set metadata for cost attribution**:

```powershell
# Via Azure Portal:
# APIM → Subscriptions → <subscription-name> → Properties → Add custom property
# - Key: cost-center, Value: AICOE-EVA
# - Key: project-id, Value: MS-InfoJP

# Via Azure CLI:
az rest --method patch `
  --url "/subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apim}/subscriptions/{sub-id}?api-version=2021-08-01" `
  --body '{
    "properties": {
      "displayName": "InfoJP Production",
      "state": "active",
      "properties": {
        "cost-center": "AICOE-EVA",
        "project-id": "MS-InfoJP"
      }
    }
  }'
```

---

## Deployment Checklist

### Pre-Deployment

- [ ] Review PLACEHOLDER comments in policy XML (search for `REPLACE`)
- [ ] Update JWT audiences with actual App Registration Client IDs
- [ ] Update CORS allowed origins with actual frontend domains
- [ ] Configure APIM diagnostic settings (Application Insights integration)
- [ ] Create APIM subscriptions for clients (with metadata)
- [ ] Test policies in APIM Test Console (per-policy validation)

### Deployment

- [ ] Deploy policy to APIM (via Azure Portal or ARM template)
- [ ] Verify policy compiles without errors
- [ ] Test public endpoints (`/health`) without token (should succeed)
- [ ] Test protected endpoints (`/chat`) without token (should return 401)
- [ ] Test protected endpoints with valid token (should succeed)

### Post-Deployment

- [ ] Run integration tests (see "Policy Testing Plan")
- [ ] Verify Application Insights logs appear (5-10 minute delay)
- [ ] Create monitoring dashboards (request volume, error rate, latency)
- [ ] Set up alerts (error rate > 1%, P95 latency > 5s)
- [ ] Document rollback procedure (revert to previous policy version)

### Validation Queries (Application Insights)

```kusto
// 1. Verify logs are arriving
ApiManagementGatewayLogs
| where TimeGenerated > ago(10m)
| take 10

// 2. Check for policy errors
ApiManagementGatewayLogs
| where customDimensions["error_source"] != ""
| take 20

// 3. Verify header injection
ApiManagementGatewayLogs
| where TimeGenerated > ago(10m)
| extend 
    correlation_id = tostring(customDimensions["correlation_id"]),
    run_id = tostring(customDimensions["run_id"]),
    user_id = tostring(customDimensions["user_id"])
| where isnotempty(correlation_id) and isnotempty(run_id)
| take 10

// 4. Check rate limit enforcement
ApiManagementGatewayLogs
| where customDimensions["response_status"] == 429
| take 10
```

---

## Acceptance Criteria (Phase 3B)

| Criterion | Status | Evidence |
|-----------|--------|----------|
| All policies compile in APIM | ✅ Ready | XML validates against APIM schema |
| JWT validation tested with Entra ID tokens | ⏳ Pending | Requires App Registration configuration |
| Streaming endpoints tested (no buffering) | ⏳ Pending | Integration test required |
| Headers flow end-to-end (APIM → backend → logs) | ⏳ Pending | Backend middleware implementation required (Phase 4) |
| Rate limits enforced (429 responses) | ⏳ Pending | Load test required |
| CORS tested from frontend | ⏳ Pending | Frontend integration required |
| Application Insights logs verified | ⏳ Pending | Post-deployment validation |
| Error responses return correlation IDs | ⏳ Pending | Error scenario testing required |

**Status**: ✅ **Policy Design Complete** - Ready for deployment and testing (Phase 3B acceptance pending integration tests)

---

## Next Steps

1. **Phase 3C** (Next): Create deployment plan with cutover strategy
2. **Phase 4A** (After 3C): Implement backend middleware to extract APIM headers
3. **Phase 4B** (After 4A): Create Cosmos DB governance_requests collection
4. **Phase 5** (Final): Integration testing, load testing, UAT

---

## References

- **OpenAPI Spec**: `09-openapi-spec.json` (39 endpoints)
- **Header Contract**: `docs/apim-scan/05-header-contract-draft.md` (952 lines)
- **APIM Project Plan**: `PLAN.md` (Phase 3B details)
- **Phase 3A Completion**: `PHASE-3A-COMPLETION.md` (OpenAPI extraction)
- **Azure APIM Docs**: [Microsoft Learn - API Management Policies](https://learn.microsoft.com/azure/api-management/api-management-policies)

---

**Document Version**: 1.0.0  
**Last Updated**: February 6, 2026  
**Author**: AI Agent (GitHub Copilot)  
**Status**: ✅ Complete - Ready for Review

**END OF APIM POLICIES DOCUMENTATION**
