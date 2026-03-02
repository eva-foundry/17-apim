# APIM Architecture & Flow Diagrams

**Purpose**: Visual representations of current and target architectures  
**Status**: Complete - ready for use in presentations and documentation  
**Last Updated**: 2026-01-28

---

## Diagram Index

### 1. [Current Architecture (No APIM)](./01-current-architecture.md)
**What it shows**: Current system state with no API Management layer  
**Key findings**:
- No user authentication (all endpoints public)
- Azure SDKs bypass any potential HTTP proxy
- No governance headers or cost tracking

**Use this when**: Explaining the problem we're solving

---

### 2. [Target APIM Architecture](./02-target-apim-architecture.md)
**What it shows**: Recommended design with APIM + backend middleware  
**Key components**:
- APIM gateway with JWT validation and policies
- Backend governance middleware for header extraction
- Cost attribution via Cosmos DB + Azure Monitor

**Use this when**: Planning implementation or presenting the solution

---

### 3. [Header Flow Sequence](./03-header-flow.md)
**What it shows**: End-to-end header propagation from client to Cosmos DB  
**Key steps**:
1. APIM validates JWT and injects headers
2. Backend middleware extracts and logs to Cosmos
3. Response echoes correlation ID back to client

**Use this when**: Implementing middleware or debugging header issues

---

### 4. [Authentication Flow](./04-authentication-flow.md)
**What it shows**: User authentication + authorization implementation  
**Key steps**:
1. User logs in with Entra ID
2. APIM validates JWT token
3. Backend checks user → project mapping
4. Authorization enforced (401/403 responses)

**Use this when**: Implementing Phase 4B (Authorization) or security review

---

## How to Use These Diagrams

### In VS Code
Open any `.md` file and VS Code will render the Mermaid diagrams automatically with the Markdown preview.

### In Presentations
Copy the Mermaid code and paste into:
- [Mermaid Live Editor](https://mermaid.live) for PNG/SVG export
- PowerPoint with Mermaid add-in
- Confluence/Notion (both support Mermaid)

### In Documentation
These diagrams are referenced throughout:
- [PLAN.md](../PLAN.md) - References target architecture
- [README.md](../README.md) - Repository structure includes diagrams/
- [STATUS.md](../STATUS.md) - Links to authentication flow
- [QUICK-REFERENCE.md](../QUICK-REFERENCE.md) - Architecture at-a-glance

---

## Evidence Sources

All diagrams are based on factual analysis from Phase 2:
- [01-api-call-inventory.md](../docs/apim-scan/01-api-call-inventory.md) - 20+ endpoints
- [02-auth-and-identity.md](../docs/apim-scan/02-auth-and-identity.md) - Current: no user auth
- [05-header-contract-draft.md](../docs/apim-scan/05-header-contract-draft.md) - 7 headers
- [CRITICAL-FINDINGS-SDK-REFACTORING.md](../CRITICAL-FINDINGS-SDK-REFACTORING.md) - SDK constraint

Each diagram includes evidence references with file:line citations.

---

## Diagram Format

All diagrams use **Mermaid** syntax for:
- ✅ Version control friendly (plain text)
- ✅ Easy to update (just edit the markdown)
- ✅ Renders in GitHub, VS Code, Confluence
- ✅ Export to PNG/SVG for presentations
