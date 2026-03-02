# AI-Optimized Folder Structure - Preview

**Date**: February 6, 2026  
**Purpose**: Show final structure before housekeeping execution

---

## Final Structure

```
17-apim/
├── README.md                              # Master overview (676 lines)
├── STATUS.md                              # Current status (404 lines)
├── INDEX.md                               # Navigation guide (235 lines)
├── QUICK-REFERENCE.md                     # Daily lookup
├── PLAN.md                                # Execution roadmap (Phases 3-5)
├── CRITICAL-FINDINGS-SDK-REFACTORING.md   # Critical decision (saves 270-410 hrs)
├── 09-openapi-spec.json                   # OpenAPI 3.1.0 spec (39 endpoints, 124.8 KB)
│
├── phase1-stack/                          # [3 files] Stack discovery (40 hours)
│   ├── 01-stack-evidence.md
│   ├── 02-scan-command-plan.md
│   └── 03-inspection-priority.md
│
├── phase2-analysis/                       # [20 files] Comprehensive analysis (172 hours)
│   ├── 01-api-call-inventory.md          # 39 endpoints
│   ├── 02-auth-and-identity.md           # Auth gap identified
│   ├── 03-config-and-base-urls.md        # 47 env vars
│   ├── 04-streaming-analysis.md          # SSE + ndjson
│   ├── 05-header-contract-draft.md       # 7 governance headers
│   ├── APPENDIX-A-Azure-SDK-Clients.md   # 150+ SDK calls
│   ├── APPENDIX-SCAN-SUMMARY.md
│   ├── INDEX.md
│   ├── SUMMARY.md
│   │
│   ├── eva-jp-specific/                   # [8 files] EVA-JP-v1.2 deep-dive
│   │   ├── 01-PHASE1-STACK-EVIDENCE.md
│   │   ├── 02-PHASE2A-API-INVENTORY.md
│   │   ├── 03-PHASE2B-RBAC-AUTH-FLOW.md
│   │   ├── 04-PHASE2C-ENVIRONMENT-VARIABLES.md
│   │   ├── 05-PHASE2D-STREAMING-ANALYSIS.md
│   │   ├── 06-PHASE2E-SDK-INTEGRATION.md
│   │   ├── PHASE2D-COMPLETION-REPORT.md
│   │   └── PHASE2E-COMPLETION-REPORT.md
│   │
│   └── verification/                      # [3 files] Phase 2 verification
│       ├── API-ENDPOINT-VERIFICATION.md
│       ├── ENDPOINT-VERIFICATION-SUMMARY.md
│       └── HEADER-UPDATE-SUMMARY.md
│
├── phase3-deliverables/                   # [3 files] APIM policies + deployment
│   ├── 10-apim-policies.xml              # Complete policy suite (531 lines)
│   ├── 10-apim-policies.md               # Policy documentation
│   └── 11-deployment-plan.md             # Deployment + cutover strategy
│
├── phase3-validation/                     # [4 files] Phase 3 validation
│   ├── PHASE-3-COMPLETE.md
│   ├── PHASE-3A-COMPLETION.md            # OpenAPI extraction (45 mins)
│   ├── 07-PHASE3-VALIDATION-REPORT.md
│   └── PHASE3-COMPLETION-REPORT.md
│
├── phase4-design/                         # [1 file] Future work
│   └── 08-CONFIGURATION-AS-DATA-DESIGN.md  # Multi-project config system (1348 lines)
│
├── diagrams/                              # [5 files] Architecture diagrams
│   ├── 01-current-architecture.md
│   ├── 02-target-apim-architecture.md
│   ├── 03-header-flow.md
│   ├── 04-authentication-flow.md
│   └── README.md
│
└── archive/                               # [10 files] Superseded documentation
    ├── ARCHIVE-INDEX.md                   # [NEW] Archive explanation
    ├── completion-reports/                # [7 files] Superseded by STATUS.md
    │   ├── 00-SUMMARY.md
    │   ├── 00-NEXT-STEPS.md
    │   ├── COMPLETION-FINDINGS-DOCUMENTED.md
    │   ├── COMPLETION-REPORT-SDK-SCAN.md
    │   ├── COMPREHENSIVE-APIM-GUIDE-COMPLETE.md
    │   ├── IMPLEMENTATION-COMPLETE.md
    │   └── REORGANIZATION-SUMMARY.md
    │
    └── superseded-docs/                   # [3 files] Old versions
        ├── README-REORGANIZED.md
        ├── START-HERE-CRITICAL-FINDINGS.md
        └── ACCURACY-AUDIT-20260204.md
```

---

## AI Agent Navigation Benefits

### Before Housekeeping
```
[AI reads root] → 26 files (overwhelming)
[AI needs evidence] → Search through 3 different folders
[AI needs analysis] → docs/apim-scan/ (not obvious)
[AI needs EVA-JP details] → eva-jp-v12-analysis/ (unclear purpose)
```

### After Housekeeping
```
[AI reads root] → 7 files (navigation clear)
[AI needs Phase 1] → phase1-stack/ (obvious)
[AI needs Phase 2] → phase2-analysis/ (obvious)
[AI needs Phase 3] → phase3-deliverables/ or phase3-validation/ (clear split)
[AI needs Phase 4] → phase4-design/ (obvious)
```

**Discovery Speed**: 3-4 list_dir calls vs 8-10 previously

---

## Folder Purpose Summary

| Folder | Files | Purpose | When AI Uses |
|--------|-------|---------|-------------|
| **Root** | 7 | Navigation + critical decisions | Every session start |
| **phase1-stack** | 3 | Tech stack discovery | Understanding system architecture |
| **phase2-analysis** | 20 | API/Auth/Config/Streaming/SDK | Understanding current state |
| **phase3-deliverables** | 3 | OpenAPI spec + APIM policies | Implementing APIM |
| **phase3-validation** | 4 | Validation reports | Checking Phase 3 completion |
| **phase4-design** | 1 | Configuration system design | Planning Phase 4 work |
| **diagrams** | 5 | Visual architecture | Understanding flows |
| **archive** | 10 | Historical docs | Rarely (audit only) |

---

## Metrics

### File Distribution
- **Total Files**: ~50 markdown files
- **Root Level**: 7 files (down from 26)
- **Active Work**: 31 files in phase folders
- **Archived**: 10 files
- **Diagrams**: 5 files
- **Binary**: 1 file (OpenAPI JSON)

### Depth Reduction
- **Before**: 4 levels deep (docs/apim-scan/verification/file.md)
- **After**: 3 levels max (phase2-analysis/verification/file.md)
- **Benefit**: Fewer list_dir calls for AI navigation

### Semantic Clarity
- **Before**: "evidences", "docs/apim-scan", "eva-jp-v12-analysis" (unclear)
- **After**: "phase1-stack", "phase2-analysis", "phase3-deliverables" (obvious)
- **Benefit**: AI knows purpose without reading files

---

## Cross-Reference Updates Required

After housekeeping, update these files:

### 1. README.md
- Update folder structure section (lines ~250-350)
- Update document links

### 2. INDEX.md
- Update "I need to..." navigation paths (lines ~100-130)
- Update deliverables paths

### 3. STATUS.md
- Update deliverables paths (lines ~80-120)
- Update evidence references

### 4. PLAN.md
- Update Phase 3 deliverable paths
- Update Phase 2 evidence references

---

## Execution Instructions

### Dry Run (Recommended First)
```powershell
cd I:\eva-foundation\17-apim
.\HOUSEKEEPING-SCRIPT.ps1 -DryRun
```

### Real Execution
```powershell
.\HOUSEKEEPING-SCRIPT.ps1
```

### Validation
```powershell
# Check new structure
tree /F /A

# Verify git history preserved
git log --follow phase1-stack/01-stack-evidence.md

# Check for broken links (manual)
# Update INDEX.md, README.md, STATUS.md paths
```

---

## Rollback Plan

If housekeeping causes issues:

```powershell
# Show last commit before housekeeping
git log -1

# Revert housekeeping
git reset --hard HEAD~1

# Or revert specific files
git checkout HEAD~1 -- <file-path>
```

---

## Decision Checkpoint

**Review this preview before executing.**

Questions:
1. ✅ Root limited to 7 navigation files?
2. ✅ Phase-based folder names clear?
3. ✅ Archive strategy acceptable?
4. ✅ Git history preservation required?

**If approved**: Run `.\HOUSEKEEPING-SCRIPT.ps1 -DryRun` to preview changes
