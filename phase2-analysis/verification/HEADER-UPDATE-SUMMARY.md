# APIM Header Contract Update Summary

**Date**: February 3, 2026  
**Updated By**: AI Assistant (with user guidance)  
**Document Updated**: `05-header-contract-draft.md`

---

## What Changed

### From: Original 7-Header Structure (Generic)

1. X-Correlation-Id (tracing)
2. X-Run-Id (batch tracking)
3. X-Caller-App (app identification)
4. X-Env (environment)
5. X-Cost-Center (cost tracking)
6. X-Project-Id (project isolation)
7. X-Ingestion-Variant (A/B testing)

**Problem**: Limited business value, no cost attribution hierarchy, missing multi-tenant support

---

### To: Improved 8-Header Structure (Business-Aligned)

**COST ATTRIBUTION HIERARCHY (5 Levels)**:
1. **X-Client-Id** - Multi-tenant isolation (ESDC-EI, ESDC-OAS, AICOE)
2. **X-Project-Id** - Project-level tracking within client (JP-Automation, EVA-Brain)
3. **X-Phase-Id** - SDLC phase tracking (dev, test, staging, prod)
4. **X-Task-Id** - Granular operation tracking (Query-Processing, Document-Upload)
5. **X-User-Id** - Individual user accountability (email@hrsdc-rhdcc.gc.ca)
6. **X-Cost-Center** - Financial attribution (ESDC-IITB-AI)

**OPERATIONAL**:
7. **X-Environment** - Deployment validation (dev/test/staging/prod)
8. **X-Correlation-Id** - Distributed tracing (UUID)

---

## Business Value Improvements

### Multi-Tenant Ready
**Before**: No client-level separation  
**After**: Query: "Show costs by client" → "ESDC-EI: $15K, ESDC-OAS: $8K, AICOE: $3K"

### Project Hierarchy
**Before**: Flat project tracking  
**After**: Query: "Show ESDC-EI projects" → "JP-Automation: $8K, EVA-Brain: $5K, Cost-Analytics: $2K"

### SDLC Phase Tracking
**Before**: No phase separation  
**After**: Query: "Show JP-Automation by phase" → "Development: $5K, Testing: $2K, Production: $10K"

### Task-Level Optimization
**Before**: No operation granularity  
**After**: Query: "Show JP-Automation prod tasks" → "Query-Processing: $6.7K (67%), Document-Upload: $2K (20%)"

### User Accountability
**Before**: No user tracking  
**After**: Query: "Show Marco's costs" → "marco.presta@...: $1.2K (3,500 queries, avg $0.34/query)"

---

## Complete Attribution Chain Example

**Query**: "Show JP-Automation Query-Processing costs for Marco in Production last month"

**Result**:
```json
{
  "client_id": "ESDC-EI",
  "project_id": "JP-Automation",
  "phase_id": "prod",
  "task_id": "Query-Processing",
  "user_id": "marco.presta@hrsdc-rhdcc.gc.ca",
  "cost_center": "ESDC-IITB-AI",
  "environment": "prod",
  "total_cost": "$1,200",
  "operations": 3500,
  "avg_cost_per_op": "$0.34"
}
```

---

## Technical Changes

### APIM Inbound Policy
**Updated**: 8-header injection with cost attribution hierarchy  
**Source**: Subscription name parsing, JWT claims, URL path extraction, deployment region

### Backend Middleware
**Updated**: `extract_cost_attribution_headers()` function  
**Purpose**: Extract 8 headers, log with 5-level attribution chain

### Cosmos DB Schema
**Added 8 Fields**:
- `client_id`, `project_id`, `phase_id`, `task_id`, `user_id`, `cost_center`, `environment`, `correlation_id`

**Enables**:
- 5-level cost rollup queries (Client → Project → Phase → Task → User)
- Financial reporting by cost center
- Environment cost separation
- Complete audit trail with correlation IDs

---

## SQL Query Examples Added

1. **Level 1 - Client Rollup**: Total costs by client
2. **Level 2 - Project Comparison**: Projects within a client
3. **Level 3 - SDLC Phase Costs**: Dev/test/prod separation
4. **Level 4 - Task Optimization**: Expensive operations identification
5. **Level 5 - User Accountability**: Individual user costs
6. **Complete Attribution**: All 5 levels in single query

---

## Implementation Impact

### Time Estimate
- Middleware implementation: **15-30 minutes**
- Cosmos schema update: **30 minutes**
- Route handler updates (4 routes): **1 hour**
- APIM policy deployment: **15 minutes**
- End-to-end testing: **1 hour**

**Total**: **3-4 hours** for complete 5-level cost attribution

### Business Impact
**Before Update**: ZERO cost attribution capability  
**After Update**: Complete 5-level cost hierarchy with actionable insights

---

## Files Modified

1. **I:\eva-foundation\17-apim\docs\apim-scan\05-header-contract-draft.md**
   - Executive Summary: Updated with 8-header structure and business value
   - Category 1: Replaced with 5-level cost attribution hierarchy
   - Category 2: Updated with financial attribution headers
   - Category 3: Simplified to distributed tracing only
   - APIM Policy Example: Complete 8-header implementation
   - Backend Middleware: Updated with cost attribution extraction
   - Cosmos DB Schema: Added 8 new fields with SQL query examples
   - Header Propagation Diagram: Updated with 5-level flow
   - Implementation Checklist: Updated for 8-header structure
   - Separation of Concerns: Simplified to APIM-authoritative vs tracing
   - Example Flow: Complete end-to-end with 8 headers

---

## Next Steps

1. ✅ **Header contract documentation updated** (COMPLETE)
2. ⏳ **Phase 3A**: Generate OpenAPI specification from FastAPI
3. ⏳ **Phase 3B**: Deploy APIM policies with 8-header structure
4. ⏳ **Phase 4A**: Implement backend middleware
5. ⏳ **Phase 4C**: Update Cosmos DB schema
6. ⏳ **Phase 4D**: Update route handlers with cost attribution logging

---

## References

- **Updated Document**: `I:\eva-foundation\17-apim\docs\apim-scan\05-header-contract-draft.md`
- **Original Proposal**: User input (February 3, 2026)
- **Implementation Plan**: `I:\eva-foundation\17-apim\PLAN.md` (Phases 3-5)
- **Project Status**: `I:\eva-foundation\17-apim\STATUS.md`

---

**Validation**: Header contract now aligns with enterprise cost attribution requirements and provides complete traceability from client to individual user.
