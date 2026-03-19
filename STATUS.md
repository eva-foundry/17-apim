# APIM Integration Project - Status Report

**Project**: MS-InfoJP APIM Integration for Cost Attribution & Governance  
**Last Updated**: 2026-02-06  
**Status**: ?? **Phase 3A Complete - OpenAPI Spec Extracted**

**Progress**: ? Phase 3A OpenAPI Generation Complete (45 minutes - automation-driven)
- ? Phase 2 Complete: All 5 sub-phases finished (API Inventory, RBAC, EnvVars, Streaming, SDK)
- ? **Phase 3A Complete**: OpenAPI spec extracted (39 endpoints, 124.8 KB)
- ? Phase 3B Next: APIM Policy Design (JWT, rate-limit, headers)

**Validation**: ? Phase 3A deliverable verified - all acceptance criteria met
- ? OpenAPI 3.1.0 spec extracted from FastAPI backend
- ? 39 endpoints documented (exceeds Phase 2 estimate of 27)
- ? Degraded mode extraction successful (no Azure dependencies)
- ? Completion report: [PHASE-3A-COMPLETION.md](./PHASE-3A-COMPLETION.md)

---

## Executive Summary

**Phases 1-2 Achievement** [FACT: 00-SUMMARY.md:1-20]:
- ? **212+ hours** of research and analysis complete
- ? **145+ evidence items** documented across 23 files
- ? **8,000+ lines** of technical documentation
- ? **20+ API endpoints** inventoried with evidence
- ? **Critical architectural finding**: SDK refactoring not viable (270-410 hours saved)

**Current State** [FACT: PLAN.md:1-50]:
- Full technical foundation established
- All analysis complete (API inventory, auth patterns, config, streaming, headers)
- Backend middleware approach validated (1 week vs. 6-10 weeks refactoring)
- Ready to proceed with Phase 3 (APIM design) and Phase 4 (middleware implementation)

**Timeline to MVP** [ESTIMATE: PLAN.md:540-570]:
- **Sequential (Option A)**: 5-6 weeks (Phases 3?4?5 in order)
- **Parallel (Option B - RECOMMENDED)**: 3-4 weeks (Phases 3+4 parallel)

---

## Phases 1-2: Analysis Complete ?

### Phase 1: Stack Evidence (40+ hours) [FACT: 00-SUMMARY.md:22-35]

**Deliverables** (3 files in [evidences/](./evidences/)):
1. ? [01-stack-evidence.md](./evidences/01-stack-evidence.md) - Tech stack identification
   - Frontend: React 18.3.1 + TypeScript 4.9.5 + Vite 5.4.11
   - Backend: Python FastAPI 0.115.0 + Azure SDKs
   - Streaming: SSE via ndjson-readablestream
   
2. ? [02-scan-command-plan.md](./evidences/02-scan-command-plan.md) - Ripgrep search patterns
   - 20+ search commands for route discovery, SDK usage, config

3. ? [03-inspection-priority.md](./evidences/03-inspection-priority.md) - File reading order
   - Top 10 files identified: app.py, api.ts, approaches, config

### Phase 2: Comprehensive Analysis (172+ hours) [FACT: 00-SUMMARY.md:36-52]

**Deliverables** (8 files in [docs/apim-scan/](./docs/apim-scan/)):
1. ? [01-api-call-inventory.md](./docs/apim-scan/01-api-call-inventory.md) (322 lines)
   - **20+ HTTP endpoints** documented with file:line evidence
   - **3 streaming endpoints** (/chat, /stream, /tdstream) - SSE implementation
   - **7 file management endpoints** (upload, delete, status, tags, folders)
   - **5 configuration endpoints** (info, warnings, feature flags, titles)
   - **5 specialized endpoints** (math, CSV assistants, citations)

2. ? [02-auth-and-identity.md](./docs/apim-scan/02-auth-and-identity.md) (454 lines)
   - **Critical Gap**: No user authentication currently implemented
   - All 20+ endpoints are publicly accessible (no JWT validation)
   - Service-to-service auth works (Managed Identity ? Azure services)
   - Token flow documented with SDK usage patterns

3. ? [03-config-and-base-urls.md](./docs/apim-scan/03-config-and-base-urls.md)
   - **47 environment variables** documented (32 core + 15 operational: KB field mappings, query/translation settings, Bing search config)
   - Service endpoints mapping
   - APIM configuration requirements

4. ? [04-streaming-analysis.md](./docs/apim-scan/04-streaming-analysis.md)
   - SSE implementation via FastAPI StreamingResponse
   - ndjson format for /chat endpoint
   - text/event-stream for /stream and /tdstream
   - APIM buffering requirements (must disable for real-time)

5. ? [05-header-contract-draft.md](./docs/apim-scan/05-header-contract-draft.md) (714 lines)
   - **7 governance headers** specified:
     - X-Correlation-Id (request tracing)
     - X-Run-Id (batch/run tracking)
     - X-Caller-App (application identification)
     - X-Env (environment: dev/staging/prod)
     - X-Cost-Center (cost allocation - optional)
     - X-Project-Id (project isolation - optional)
     - X-Ingestion-Variant (A/B testing - optional)
   - Injection points: APIM inbound policy ? Backend middleware ? Cosmos DB
   - **Current state**: No headers extracted or propagated

6. ? [APPENDIX-A-Azure-SDK-Clients.md](./docs/apim-scan/APPENDIX-A-Azure-SDK-Clients.md)
   - Detailed SDK usage analysis (AsyncAzureOpenAI, SearchClient, etc.)
   - 150+ SDK integration points identified

7. ? [APPENDIX-SCAN-SUMMARY.md](./docs/apim-scan/APPENDIX-SCAN-SUMMARY.md)
   - Scan execution results
   - Evidence collection methodology

8. ? [INDEX.md](./docs/apim-scan/INDEX.md) + [SUMMARY.md](./docs/apim-scan/SUMMARY.md)
   - Navigation and document index
   - Phase completion status

### Critical Findings [FACT: CRITICAL-FINDINGS-SDK-REFACTORING.md:12-35]

**?? Architecture Decision**: DO NOT refactor Azure SDKs to HTTP wrappers

**The Finding**:
- **150+ direct Azure SDK calls** across 15+ Python files (AsyncAzureOpenAI, SearchClient, BlobServiceClient, CosmosClient)
- **Azure SDKs bypass HTTP proxies** - they handle auth internally and establish direct connections
- **APIM cannot intercept SDK calls** - architectural constraint

**The Cost** [FACT: CRITICAL-FINDINGS-SDK-REFACTORING.md:97-145]:
- Refactoring effort: **270-410 hours** (6-10 weeks)
- Risk: **Very High** (core business logic changes)
- Benefit: **Minimal** (APIM still cannot see SDK calls)
- ROI: **Negative**

**Recommended Solution** [FACT: CRITICAL-FINDINGS-SDK-REFACTORING.md:202-250]:
- ? Backend Middleware + Azure Monitor (20-30 hours, 1 week)
- Add FastAPI middleware to extract APIM headers
- Log governance metadata to Cosmos DB
- Leverage Azure Monitor for SDK call tracking
- Match APIM logs + SDK logs for cost attribution
- **No SDK refactoring required**

**Decision**: Proceed with Backend Middleware approach ? saves 250-380 hours

---

## ?? WHAT TO DO NEXT (Immediate Actions)

### Week of February 10, 2026

**Day 1-2: Team Kickoff & Decision**
1. [ ] **Review validated documentation** with stakeholders
   - Share [ACCURACY-AUDIT-20260204.md](./ACCURACY-AUDIT-20260204.md) - proves 100% accuracy
   - Review [PLAN.md](./PLAN.md) - complete execution roadmap
   - Review [CRITICAL-FINDINGS-SDK-REFACTORING.md](./CRITICAL-FINDINGS-SDK-REFACTORING.md) - saves 250-380 hours

2. [ ] **Make execution timeline decision**
   - **Option A (Sequential)**: 5-6 weeks, lower risk, clear handoffs
   - **Option B (Parallel - RECOMMENDED)**: 3-4 weeks, slightly higher risk, faster delivery

3. [ ] **Confirm team composition** (8-12 people needed)
   - Architect (1 FTE)
   - Backend Developers (2-3 FTE)
   - DevOps Engineers (2 FTE)
   - QA Engineer (1 FTE)
   - APIM Specialist (0.5 FTE - optional)

**Day 3-5: Phase 3A Kickoff (OpenAPI Spec)**
1. [ ] **Start backend locally**
   ```powershell
   cd I:\eva-foundation\11-MS-InfoJP\base-platform\app\backend
   python app.py
   ```

2. [ ] **Download OpenAPI spec**
   - Navigate to http://localhost:5000/docs
   - Download JSON: http://localhost:5000/openapi.json
   - Save as `I:\eva-foundation\17-apim\09-openapi-spec.json`

3. [ ] **Refine OpenAPI spec** (see PLAN.md Phase 3A)
   - Add `operationId` to each endpoint (required by APIM)
   - Add security scheme (Bearer JWT for Entra ID)
   - Mark streaming endpoints with `x-apim-policy: handle-streaming`
   - Normalize response schemas

**Week 2: Phase 3B (APIM Policies) + Phase 4A (Middleware)**
- Run in parallel if Option B selected
- See [PLAN.md](./PLAN.md) for detailed task breakdown

---

## Phases 3-5: Planned Execution [RECOMMENDATION: PLAN.md:1-42]

### Phase 3: APIM Design & Setup (44-60 hours, 2-3 weeks)

**Deliverables**:
1. [ ] **09-openapi-spec.yaml** (Phase 3A: 8-12 hrs)
   - Extract from FastAPI /docs endpoint
   - Add operationId, security scheme (Bearer JWT)
   - Mark streaming endpoints (SSE)
   - Validate 20+ endpoints from inventory

2. [ ] **10-apim-policies.xml** + **10-apim-policies.md** (Phase 3B: 12-16 hrs)
   - JWT validation policy (Entra ID)
   - Rate limiting (100 req/min per user, 10k/min per app)
   - Header injection (X-Correlation-Id, X-Caller-App, X-Env, etc.)
   - Streaming handling (buffering=false for SSE)
   - CORS policy
   - Request/response logging (Application Insights)

3. [ ] **11-deployment-plan.md** (Phase 3C: 12-16 hrs)
   - APIM resource creation (tier, region, capacity)
   - API import and policy assignment
   - Traffic cutover strategy (3 options: config switch, DNS, dual-run)
   - Rollback procedures
   - Go-live checklist

**Owner**: Architect, Backend Developer, DevOps, APIM Specialist (4 people)

### Phase 4: Backend Middleware & Header Propagation (40-56 hours, 2 weeks)

**Deliverables**:
1. [ ] **Governance Logging Middleware** (Phase 4A: 8-12 hrs)
   - FastAPI middleware extracts headers
   - Log to Cosmos DB `governance_requests` collection
   - Store in `request.state` for route handlers
   - Correlation ID in response headers

2. [ ] **Authorization Enforcement** (Phase 4B: 8-12 hrs)
   - JWT validation (Entra ID `oid` extraction)
   - User ? project mapping (Cosmos DB lookup)
   - Route guards (@require_auth, @require_project_access)
   - 401/403 responses for unauthorized access

3. [ ] **Cosmos DB Schema Extension** (Phase 4C: 4-6 hrs)
   - Create collections: `sandbox_users`, `projects`, `user_projects`, `governance_requests`
   - Add indexes (timestamp, user_id, project_id, run_id, variant)
   - TTL on governance_requests (90 days)

4. [ ] **Cost Attribution Queries** (Phase 4D: 8-12 hrs)
   - KQL queries for cost reports
   - Run comparison (Variant A vs B)
   - Project usage dashboards
   - OpenAI token tracking

**Owner**: Backend Developer (2), DevOps, Data Analyst (4 people)

### Phase 5: Testing & Deployment (40-56 hours, 1 week)

**Deliverables**:
1. [ ] **Integration Testing** (Phase 5A: 12-16 hrs)
   - All 20+ endpoints routed through APIM
   - Header propagation end-to-end
   - Streaming endpoints (SSE) work without buffering
   - Authorization tests (401/403 responses)

2. [ ] **Load Testing** (Phase 5B: 8-12 hrs)
   - Performance baseline vs. with APIM+middleware
   - P95 latency <2s, throughput ?100 req/sec
   - Error rate <0.1%

3. [ ] **UAT** (Phase 5C: 12-16 hrs)
   - AICOE team sign-off
   - Finance validates cost report accuracy
   - Security approves auth model
   - Support team trained

4. [ ] **Go-Live Cutover** (Phase 5D: 8-12 hrs)
   - Deploy APIM + policies
   - Deploy backend middleware
   - Traffic cutover (DNS recommended)
   - Monitoring dashboards operational
   - On-call support ready

**Owner**: QA, PM, DevOps, Backend Developer, On-Call (5+ people)

---

## Timeline & Sequencing [ESTIMATE: PLAN.md:540-600]

### Option A: Sequential (Safer, 5-6 weeks)
```
Week 1-2: Phase 3 (APIM Design)
Week 3-4: Phase 4 (Backend Middleware)
Week 5:   Phase 5 (Testing & Deployment)
```
**Risk**: Low | **Complexity**: Simple | **Team**: 3-4 people per phase

### Option B: Parallel - RECOMMENDED (Faster, 3-4 weeks)
```
Week 1-2: Phase 3 + Phase 4 (parallel)
  - Phase 3B finishes Week 1 ? Phase 4A uses finalized header list
  - Phase 3A/3C and Phase 4C/4D run independently
Week 3:   Phase 5 (Testing & Deployment)
```
**Risk**: Medium | **Complexity**: Coordination required | **Team**: 8-12 people

**Recommendation**: **Option B** - saves 2-3 weeks with acceptable risk

---

## Critical Path & Dependencies [ESTIMATE: PLAN.md:605-635]

**Blocking Path**:
1. Phase 3A (OpenAPI spec) ? Phase 3B (APIM policies) ? Phase 4A (middleware expects header list)
2. Phase 4C (Cosmos schema) ? Phase 4A (middleware writes to DB)
3. Phase 4A + 4B + 4D ? Phase 5 (testing requires all components)

**High-Risk Areas**:
1. **User Auth** (Phase 4B) - Currently missing entirely. Adding JWT validation requires thorough testing.
2. **Streaming through APIM** (Phase 3B + 5A) - SSE must not be buffered. Test early.
3. **Cosmos Throughput** (Phase 4C) - Every request logs. Start with 10k RU, monitor.
4. **Traffic Cutover** (Phase 5D) - All users affected. DNS-based for easy rollback.

---

## Key Metrics & Success Criteria [FACT: PLAN.md:650-667]

### MVP Definition [FACT: README.md:400-450]
1. ? All UI calls routed through APIM gateway
2. ? Headers propagated and stored (X-Correlation-Id, X-Run-Id, X-Project-Id, X-User-Id, etc.)
3. ? Cost attribution reports (OpenAI tokens, Search queries, Storage usage per run/variant)
4. ? Authorization enforced (user can only call assigned projects)
5. ? Governance metadata persisted to Cosmos DB

### Technical Acceptance
- [ ] OpenAPI spec validated by APIM (imports without errors)
- [ ] All APIM policies tested with Entra ID tokens
- [ ] 20+ endpoints routed correctly (inventory verified)
- [ ] Streaming endpoints (SSE) pass through without buffering
- [ ] Headers extracted, logged, queryable from Cosmos
- [ ] Cost queries produce expected results (?10% of Azure bill)
- [ ] Authorization working (401/403 responses, project access enforced)

### Operational Acceptance
- [ ] APIM instance deployed, scaled, monitored
- [ ] Rollback procedures tested and documented
- [ ] Cost dashboard operational
- [ ] Performance: P95 latency <2s, throughput ?100 req/sec
- [ ] Error rate <0.1% within 1 hour post-cutover

### Stakeholder Acceptance
- [ ] AICOE team sign-off (UAT passed)
- [ ] Finance validates cost report accuracy
- [ ] Security approves JWT validation + authorization
- [ ] Support team trained on runbook

---

## Team Composition [FACT: PLAN.md:600-630]

**Recommended Team** (8-12 people):
- Architect (1 FTE) - Design APIM policies, header contract
- Backend Developers (2 FTE) - Implement middleware, authorization
- DevOps Engineer (1 FTE) - APIM deployment, Cosmos provisioning
- APIM Specialist (0.5 FTE) - Policy expertise, streaming issues
- QA/Test Engineer (1 FTE) - Integration + load testing
- Data Analyst (0.5 FTE) - Cost queries, dashboard
- Security Engineer (0.5 FTE) - JWT validation, authorization review
- Product Manager (0.5 FTE) - UAT coordination
- On-Call Support (rotating) - Post-cutover monitoring

---

## Next Steps (Immediate) [FACT: 00-NEXT-STEPS.md:1-218]

### This Week (Week of Jan 27)
1. [ ] **Distribute PLAN.md** to team and stakeholders
2. [ ] **Decision**: Sequential (Option A) or Parallel (Option B)?
3. [ ] **Confirm team roles** (8-12 people, assign names)
4. [ ] **Address critical gaps**:
   - User auth gap (Phase 4B spike)
   - Streaming test (APIM SSE validation)
   - Cosmos RU estimation (10k RU starting point)

### Next Week (Week of Feb 3) - Phase 3A Kickoff
1. [ ] **OpenAPI Spec Generation** (1-2 days)
   - Start backend locally (http://localhost:5000/docs)
   - Download OpenAPI JSON
   - Add operationId, security scheme, streaming annotations
   - Deliverable: [09-openapi-spec.yaml](./09-openapi-spec.yaml)

2. [ ] **Cosmos Schema Creation** (1 day, parallel)
   - Create collections: sandbox_users, projects, user_projects, governance_requests
   - Add indexes, configure TTL
   - Deliverable: Collections in sandbox Cosmos DB

---

## Document Index

**Critical - Read First**:
- ?? [CRITICAL-FINDINGS-SDK-REFACTORING.md](./CRITICAL-FINDINGS-SDK-REFACTORING.md) - Architecture decision
- ?? [PLAN.md](./PLAN.md) - Master execution plan (Phases 3-5)
- ?? [README.md](./README.md) - 5-phase APIM process, MVP definition

**Phase 1-2 Deliverables**:
- [evidences/](./evidences/) - Stack evidence, scan commands, inspection priorities (3 files)
- [docs/apim-scan/](./docs/apim-scan/) - Complete analysis (8 files, 8,000+ lines)

**Phase 3-5 Planning**:
- [00-NEXT-STEPS.md](./00-NEXT-STEPS.md) - Immediate action items
- [COMPLETION-FINDINGS-DOCUMENTED.md](./COMPLETION-FINDINGS-DOCUMENTED.md) - Phase 2 completion summary

---

## Questions & Support

**Questions about findings?** See [CRITICAL-FINDINGS-SDK-REFACTORING.md](./CRITICAL-FINDINGS-SDK-REFACTORING.md)  
**Questions about PLAN?** See [PLAN.md Executive Summary](./PLAN.md#executive-summary)  
**Questions about current state?** See [00-SUMMARY.md](./00-SUMMARY.md)

---

**Last Updated**: 2026-01-28 | **Status**: Ready for Phase 3 Execution


---

## 2026-03-03 -- Re-primed by agent:copilot

<!-- eva-primed-status -->

Data model: GET http://localhost:8010/model/projects/17-apim
29-foundry agents: C:\eva-foundry\eva-foundation\29-foundry\agents\
48-eva-veritas: run audit_repo MCP tool
