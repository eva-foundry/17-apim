# APIM Integration - Quick Reference Card

**Project**: MS-InfoJP APIM Integration  
**Status**: Phase 2 Complete → Phase 3 Ready  
**Updated**: 2026-01-28

---

## 📊 At-a-Glance

| Metric | Value | Source |
|--------|-------|--------|
| **Phase 1-2 Hours** | 212+ hours | [FACT: 00-SUMMARY.md:1] |
| **Phase 3-5 Estimate** | 124-172 hours | [ESTIMATE: PLAN.md:10] |
| **API Endpoints** | 20+ | [FACT: 01-api-call-inventory.md:20] |
| **Streaming Endpoints** | 3 (SSE) | [FACT: 01-api-call-inventory.md:25] |
| **Governance Headers** | 7 | [FACT: 05-header-contract-draft.md:50] |
| **Azure SDK Calls** | 150+ | [FACT: CRITICAL-FINDINGS-SDK-REFACTORING.md:42-58] |
| **Refactoring Saved** | 250-380 hours | [FACT: CRITICAL-FINDINGS-SDK-REFACTORING.md:145-165] |
| **Timeline to MVP** | 3-4 weeks (parallel) | [ESTIMATE: PLAN.md:545-565] |

---

## 🎯 Critical Findings

**⚠️ DO NOT refactor Azure SDKs** [FACT: CRITICAL-FINDINGS-SDK-REFACTORING.md:6-35]
- 150+ SDK calls across 15+ files (deeply integrated)
- APIM cannot intercept SDK calls (SDKs bypass proxies)
- Refactoring cost: 270-410 hours, high risk, minimal benefit
- ✅ **Use Backend Middleware instead** (20-30 hours, low risk)

**⚠️ User Auth Missing** [FACT: 02-auth-and-identity.md:125-145]
- All 20+ endpoints currently public (no JWT validation)
- Must add Entra ID auth in Phase 4B before public exposure
- Requires user → project mapping in Cosmos DB

**⚠️ Streaming Requires Special Handling** [FACT: 04-streaming-analysis.md]
- 3 SSE endpoints: /chat, /stream, /tdstream
- APIM must disable buffering (`buffering=false` policy)
- Test early to avoid breaking real-time responses

---

## 📋 Key Endpoints [FACT: 01-api-call-inventory.md:9-95]

### Chat & Streaming (Core RAG)
```
POST /chat              → Main RAG conversation (ndjson streaming)
GET  /stream?question=  → Math problem streaming (SSE)
GET  /tdstream?question=→ CSV analysis streaming (SSE)
```

### File Management
```
POST /file              → Upload file (multipart)
POST /getalluploadstatus→ List uploads by timeframe/state/folder/tag
POST /getfolders        → List all folders in upload container
POST /gettags           → List all unique tags
POST /deleteItems       → Delete blob + log status
POST /resubmitItems     → Re-upload blob to pipeline
POST /logstatus         → Upsert document status in Cosmos
```

### Configuration & Info
```
GET  /getInfoData       → Deployment info (model, endpoints, config)
GET  /getWarningBanner  → Chat warning banner text
GET  /getFeatureFlags   → Feature flags (web chat, math, tabular)
GET  /getMaxCSVFileSize → CSV size limit
GET  /getApplicationTitle→ Application title
```

### Content & Citations
```
POST /getcitation       → Download citation JSON by name
POST /get-file          → Download citation file (PDF, image)
GET  /getalltags        → Get all tags in system
```

### Specialized (Math & CSV)
```
POST /posttd                         → Upload CSV for tabular data assistant
GET  /process_td_agent_response?q=   → Process CSV question
GET  /getTdAnalysis?q=               → Get CSV analysis trace
GET  /getHint?q=                     → Generate math hint
GET  /process_agent_response?q=      → Process math question
```

### Infrastructure
```
GET  /health            → Liveness probe (status, uptime, version)
GET  /                  → Redirect to /index.html
```

**All endpoints currently public (no auth)** ⚠️

---

## 🔐 Governance Headers [FACT: 05-header-contract-draft.md:50-120]

### Required Headers (Propagate End-to-End)

| Header | Source | Purpose | Example |
|--------|--------|---------|---------|
| **X-Correlation-Id** | APIM generates | Request tracing | `550e8400-e29b-41d4-a716-446655440000` |
| **X-Run-Id** | APIM generates | Batch/run tracking | `batch-2024-01-25-001` |
| **X-Caller-App** | APIM sets | Application ID | `infojp-web` |
| **X-Env** | APIM sets | Environment | `dev`, `staging`, `prod` |
| **X-User-Id** | APIM from JWT | User identity (Entra OID) | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| **X-Project-Id** | Client sends | Project isolation | `proj-infojp-001` |
| **X-Cost-Center** | APIM/backend | Cost allocation (optional) | `CC-12345` |
| **X-Ingestion-Variant** | Client sends | A/B testing (optional) | `control`, `variant-a` |

### Injection Flow
```
Client → APIM (generate/inject) → Backend Middleware (extract) → Cosmos DB (log) → Response (echo)
```

### Backend Implementation Required [RECOMMENDATION: PLAN.md:151-250]
```python
@app.middleware("http")
async def extract_governance_headers(request: Request, call_next):
    correlation_id = request.headers.get("X-Correlation-Id", str(uuid.uuid4()))
    user_id = request.headers.get("X-User-Id", "anonymous")
    project_id = request.headers.get("X-Project-Id", "unknown")
    
    request.state.correlation_id = correlation_id
    request.state.user_id = user_id
    request.state.project_id = project_id
    
    # Log to Cosmos governance_requests
    await log_governance_entry(correlation_id, user_id, project_id, ...)
    
    response = await call_next(request)
    response.headers["X-Correlation-Id"] = correlation_id
    return response
```

---

## 🏗️ Architecture

### Current (No APIM) [FACT: 01-api-call-inventory.md, 02-auth-and-identity.md]
```
Frontend → Backend (public endpoints) → Azure Services (SDK direct)
           ❌ No auth
           ❌ No headers
           ❌ No cost tracking
```

### Target (With APIM) [RECOMMENDATION: PLAN.md:1-667]
```
Frontend → APIM (JWT + policies) → Backend (middleware + auth) → Azure Services (SDK)
           ✅ JWT validation            ✅ Extract headers           ↓
           ✅ Rate limiting             ✅ Authorization          Azure Monitor
           ✅ Header injection          ✅ Log to Cosmos          (SDK diagnostics)
                                                                     ↓
                                                                 Cost Reports
                                                                 (match logs)
```

**Key Insight**: APIM fronts backend API only. Azure SDKs bypass APIM (direct Azure connections).

---

## 📅 Timeline [ESTIMATE: PLAN.md:540-600]

### Phase 3: APIM Design (2-3 weeks, 44-60 hours)
- [ ] Week 1: OpenAPI spec (3A: 1-2 days)
- [ ] Week 1-2: APIM policies (3B: 3-4 days)
- [ ] Week 2: Deployment plan (3C: 3-4 days)

### Phase 4: Backend Middleware (2 weeks, 40-56 hours, parallel with Phase 3)
- [ ] Week 1: Governance logging middleware (4A: 2-3 days)
- [ ] Week 1: Authorization enforcement (4B: 2-3 days)
- [ ] Week 1: Cosmos DB schema (4C: 1 day)
- [ ] Week 2: Cost attribution queries (4D: 2-3 days)

### Phase 5: Testing & Deployment (1 week, 40-56 hours)
- [ ] Week 3 Mon-Tue: Integration tests (5A: 2-3 days)
- [ ] Week 3 Wed: Load tests (5B: 1-2 days)
- [ ] Week 3 Thu: UAT (5C: 2-3 days)
- [ ] Week 3 Fri: Go-live cutover (5D: 1-2 days)

**Parallel Execution (Recommended)**: 3-4 weeks total (Phases 3+4 overlap)

---

## ✅ MVP Success Criteria [FACT: README.md:400-450, PLAN.md:650]

1. ✅ All UI calls routed through APIM gateway
2. ✅ Headers propagated end-to-end (X-Correlation-Id, X-Run-Id, X-Project-Id, X-User-Id)
3. ✅ Cost attribution reports operational (OpenAI tokens, Search queries per run/variant)
4. ✅ Authorization enforced (user can only call assigned projects)
5. ✅ Governance metadata persisted to Cosmos DB

**Technical**:
- OpenAPI spec imports to APIM without errors
- JWT validation working (Entra ID tokens)
- Streaming endpoints (SSE) work without buffering
- Cost queries accurate (±10% of Azure bill)

**Operational**:
- P95 latency <2s, throughput ≥100 req/sec
- Error rate <0.1% within 1 hour post-cutover
- Rollback procedures validated

---

## 🚨 Risk Mitigation [ESTIMATE: PLAN.md:605-635]

| Risk | Severity | Mitigation |
|------|----------|------------|
| **User Auth Missing** | 🔴 HIGH | Phase 4B (Authorization) - thorough UAT with test users |
| **Streaming Buffering** | 🟡 MEDIUM | Explicit `buffering=false` APIM policy, load testing |
| **Cosmos Throughput** | 🟡 MEDIUM | Start with 10k RU, monitor, scale as needed |
| **Traffic Cutover** | 🟡 MEDIUM | DNS-based cutover (easy rollback), on-call ready |

---

## 📚 Key Documents (Read Order)

1. **[CRITICAL-FINDINGS-SDK-REFACTORING.md](./CRITICAL-FINDINGS-SDK-REFACTORING.md)** ⚠️ START HERE
   - Architecture decision: Why backend middleware, not SDK refactoring

2. **[PLAN.md](./PLAN.md)** 📋 MASTER EXECUTION PLAN
   - Phases 3-5 detailed breakdown (750+ lines)

3. **[README.md](./README.md)** 📖 OVERVIEW
   - 5-phase APIM process, MVP definition

4. **[docs/apim-scan/01-api-call-inventory.md](./docs/apim-scan/01-api-call-inventory.md)** 📊 API REFERENCE
   - 20+ endpoints with evidence

5. **[docs/apim-scan/02-auth-and-identity.md](./docs/apim-scan/02-auth-and-identity.md)** 🔐 AUTH ANALYSIS
   - Current: No user auth, all endpoints public

6. **[docs/apim-scan/05-header-contract-draft.md](./docs/apim-scan/05-header-contract-draft.md)** 📋 HEADER SPEC
   - 7 governance headers, injection points

---

## 🔧 Quick Commands

### Start Backend Locally
```bash
cd app/backend
.venv/Scripts/Activate.ps1  # Windows
python app.py                # Runs on :5000
```

### Access OpenAPI Spec
```bash
# Browser: http://localhost:5000/docs
# Download JSON:
curl http://localhost:5000/openapi.json > openapi.json
```

### Ripgrep Searches (from repo root)
```bash
# Find all routes
rg -n '@app\.(get|post|put|delete)\("' app/backend/

# Find all frontend API calls
rg -n 'export async function|await fetch\(' app/frontend/src/api/

# Find Azure SDK usage
rg -n 'from azure\.|import azure\.' app/backend/ --type python
```

---

## 📞 Next Steps

**This Week** (Week of Jan 27):
1. [ ] Review PLAN.md with team
2. [ ] Decide: Sequential (5-6 weeks) or Parallel (3-4 weeks)?
3. [ ] Assign team roles (8-12 people)
4. [ ] Schedule Phase 3A kickoff (OpenAPI spec)

**Next Week** (Week of Feb 3):
1. [ ] Generate OpenAPI spec from /docs endpoint
2. [ ] Create Cosmos DB collections (sandbox_users, projects, etc.)
3. [ ] Begin APIM policy design

---

**Questions?** See [STATUS.md](./STATUS.md) or [PLAN.md](./PLAN.md) for full details.
