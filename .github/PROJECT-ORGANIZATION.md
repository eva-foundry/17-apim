<!-- eva-primed-organization -->

# 17-apim - Organization Standards

**Version**: 1.0.0
**Primed**: 2026-03-10 by agent:copilot
**Status**: Canonical scaffold for folder structure and file placement

---

## Root-Level Rules

Keep root minimal. Use root for governance and entry-point files only:
- `README.md`, `PLAN.md`, `STATUS.md`, `ACCEPTANCE.md`
- `requirements.txt`, `.gitignore`, `Dockerfile` (if applicable)
- `.github/`, `docs/`, `scripts/`, `tests/`, `evidence/`

Move operational outputs out of root:
- Temporary logs -> `artifacts/logs/`
- One-off outputs -> `artifacts/outputs/`
- Historical backups -> `archives/`

---

## Target Folder Structure

```text
docs/
  library/        # Stable reference docs
  sessions/       # Time-bound session notes and reports
  architecture/   # ADRs, diagrams, design patterns

scripts/
  deployment/     # Deploy and environment provisioning
  seed/           # Data/model initialization
  validation/     # Validation and quality checks
  sync/           # External/internal synchronization
  analysis/       # Analysis and reporting scripts
  debug/          # Diagnostics and troubleshooting helpers
  migration/      # Migration and recovery utilities
  admin/          # Administrative scripts
  testing/        # Test helpers and smoke checks

archives/
  logs/
  backups/

artifacts/
  logs/
  outputs/
```

---

## Fractal DPDCA For Reorganization

Apply DPDCA at each granularity level.

1. **DISCOVER**: Inventory current files and classify by type.
2. **PLAN**: Define target destination for each file group.
3. **DO**: Move one file group at a time.
4. **CHECK**: Validate links, script paths, and references after each move.
5. **ACT**: Update docs and standards with actual changes.

---

## Hybrid Paperless Governance (REQUIRED)

This project uses **mandatory hybrid governance** with Data Model API:

**Pre-Flight Requirement**:
- Priming requires EVA Data Model API to be available
- Pre-flight check: `GET https://msub-eva-data-model...canadacentral.../health`
- Priming fails if API unavailable (10-second timeout)

**File-Based (Backward Compatible)**:
- Local PLAN.md, STATUS.md, ACCEPTANCE.md files created
- Traditional git-based workflow preserved
- Works offline after initial prime

**API-First (Mandatory Sync)**:
- Project record synced to EVA Data Model API during prime
- Step 7: Data Model API sync (cannot be skipped)
- Evidence tracked: `API-SYNC:CREATED` | `API-SYNC:EXISTS`

**Priming Behavior**:
1. Pre-flight: Check API availability (REQUIRED)
2. Steps 1-6: Create all governance files
3. Step 7: Sync project record to API (REQUIRED)
4. Evidence: `.eva/prime-evidence.json` with 7-step audit trail

**Why Hybrid**:
- Enables gradual paperless transition without breaking workflows
- Files provide offline reference and git history
- API provides live query/sync for veritas tools
- Future: Full paperless (API-only, files become optional)

**Veritas Integration**:
- `eva sync_repo` reads governance from API + writes MTI back
- `eva get_trust_score` queries API for MTI score
- `eva audit_repo` combines file + API evidence

---

## Placement Rules

1. `SESSION-*.md` belongs in `docs/sessions/`.
2. Ad-hoc scripts (debug, analysis, fixes) belong in `scripts/*/` subfolders.
3. Generated JSON, reports, and command output belong in `artifacts/outputs/`.
4. Historical exports and backups belong in `archives/` with timestamped folders.
5. Root should never accumulate disposable execution artifacts.

---

## Verification Checklist

- [ ] Root contains only approved files/folders
- [ ] `docs/sessions/` contains session reports
- [ ] Scripts are grouped by operational purpose
- [ ] Archive and artifact folders are used for non-source outputs
- [ ] No broken paths after reorganization
