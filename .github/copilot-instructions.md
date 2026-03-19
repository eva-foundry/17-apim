<!-- eva-project-authority -->

# GitHub Copilot Instructions -- 17-apim

**Project**: InfoJP APIM front door and governance design  
**Path**: C:\eva-foundry\17-apim\

## Start Here

1. Complete workspace bootstrap first.
2. Query the live project record:
	```powershell
	Invoke-RestMethod "$($session.base)/model/projects/17-apim"
	```
3. Read local docs in this order:
	- README.md
	- PLAN.md
	- STATUS.md
	- ACCEPTANCE.md

## Project Role

This repo defines how APIM becomes the governed ingress for the InfoJP stack. The active planning focus is APIM contract design, policy design, header propagation, deployment sequencing, and cutover safety. Keep work aligned with the phase structure already present in PLAN.md, especially Phases 3 through 5.

## Azure Rules

- This is an Azure project. Load the Azure instruction overlay before Azure-specific design or operational work.
- Prefer Azure-aware tooling and current service guidance over stale local assumptions.
- Treat APIM policy, OpenAPI contract, auth model, diagnostics, and cutover documentation as one governed surface.

## Working Rules

- Preserve the public API contract, policy definitions, and backend-routing assumptions together.
- Do not propose direct frontend-to-backend bypasses when the project goal is governed ingress through APIM.
- Keep required headers, identity propagation, throttling, and observability consistent across docs and code.
- If you add infrastructure artifacts, keep them aligned with the intended deployment and rollback plan.

## Validation

- Validate the specific artifact type you change: OpenAPI, APIM policy XML, IaC, middleware docs, or cutover steps.
- If no executable stack exists for a change, validate by schema, syntax, or documented operational consistency.
- Update STATUS.md whenever phase progress, blockers, or rollout assumptions change.

## Traceability

Use project story tags when traceability is required:

```text
EVA-STORY: F17-03-001
EVA-FEATURE: F17-03
```

Run Project 48 audit tooling before high-risk governance changes.

## Boundaries

- Do not hardcode layer counts or stale API assumptions.
- Do not leave placeholder APIM guidance in place once concrete contract details are known.
- Do not separate policy design from authentication, diagnostics, and cutover implications.
