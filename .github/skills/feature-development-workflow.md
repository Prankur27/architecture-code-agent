# Feature Development Workflow Skill

## Description

You are an orchestrator agent that runs the full feature development pipeline end-to-end — from JIRA requirement to working, tested, documented code. You coordinate five specialized sub-agents in sequence, with mandatory user review and approval gates at every phase before proceeding.

The sub-agents you orchestrate are:
1. **Requirement Analyzer** — fetches and deeply analyzes the JIRA ticket, iterates until requirements are fully clear and agreed
2. **Architecture Analyzer** *(conditional)* — if no C4/flow diagrams exist yet, runs the full documentation pipeline first to bootstrap them
3. **C4 Diagram Generator** *(conditional)* — generates C4 diagrams from scratch if they are absent
4. **Flow Diagram Generator** *(conditional)* — generates flow diagrams from scratch if they are absent
5. **Architecture Design Agent** — produces a detailed technical design using the C4/flow diagrams as context
6. **Implementation Planning Agent** — breaks the design into a fully ordered, file-specific implementation plan
7. **Feature Implementation Agent** — executes every task in the plan, writing production-quality code
8. **C4 Diagram Generator** (reused) + **Flow Diagram Generator** (reused) — updates architecture diagrams to reflect the implemented feature

The workflow enforces **architecture as code**: every feature goes through design → diagrams → implementation → diagram update, ensuring the architecture documentation always stays in sync with the codebase.

---

## Workflow

### ─── PHASE 0: Collect Inputs ───

Before invoking any sub-agent, collect all required inputs from the user in a single interaction.

Present the following prompt and **wait for their response**:

```
I will run the full Feature Development pipeline for you.

This pipeline will execute the following agents in sequence:
  1. Requirement Analyzer     — analyze JIRA ticket + ask clarifying questions
  2. Architecture Design      — produce detailed technical design + ADRs
  3. Implementation Planning  — produce file-specific, ordered task plan
  4. Feature Implementation   — write all code, tests, and migrations
  5. Architecture Update      — update C4 and flow diagrams to reflect changes

Please provide the following inputs:

JIRA:
  • JIRA Epic or Story URL or ID   (e.g., PROJ-123)
  • JIRA Project base URL          (e.g., https://your-org.atlassian.net)

CODEBASE:
  • Backend repository GitHub URL  (required)
  • Backend branch                 (default: master)
  • UI repository GitHub URL       (optional)
  • UI branch                      (default: master)
  • UI subpath                     (optional, e.g., apps/frontend)

EXISTING ARCHITECTURE DOCS (leave blank if not available):
  • Path to Architecture Analysis Report  (default: docs/architecture/architecture-analysis-report.md)
  • Path to C4 Diagrams Report            (default: docs/architecture/c4-diagrams-report.md)
  • Path to Flow Diagrams Report          (default: docs/architecture/flow-diagrams-report.md)

Type your answers and press Send when ready.
```

**⏸ PAUSE — Wait for user to provide inputs.**

Validate inputs:
- JIRA ID/URL and backend repository are required. Ask again if missing.
- Confirm collected inputs in a summary table and ask: *"Are these details correct? Reply 'yes' to proceed."*

**⏸ PAUSE — Wait for user confirmation.**

---

### ─── PHASE 1: Requirement Analysis (Sub-Agent 1) ───

**Invoke Sub-Agent: `requirement-analyzer`**

Pass the following context:
- JIRA ticket URL / ID and project base URL
- Backend repository URL and branch
- UI repository URL and branch (if provided)
- UI subpath (if provided)
- Paths to existing architecture documents (if provided)

Instruct the sub-agent to follow all steps in `.github/agents/requirement-analyzer.md`. The sub-agent must:
1. Fetch and parse the JIRA ticket
2. Load and analyze the codebase/architecture context
3. Perform deep requirement analysis (clarity, completeness, feasibility, security, domain)
4. Present clarifying questions to the user and iterate until all are resolved
5. Produce the Requirements Analysis Document (RAD)
6. Present the RAD for user approval

> **Sub-Agent Execution Boundary**: Steps 2–7 of `requirement-analyzer.md` are executed by this sub-agent. The sub-agent handles all Q&A rounds internally and only returns when the RAD is approved.

**⏸ PAUSE — The sub-agent will pause multiple times internally for Q&A. This phase is complete only when the user says 'approved' on the RAD.**

Once the sub-agent returns an approved RAD, inform the user:

```
✅ Phase 1 Complete — Requirements Analysis Document approved.

JIRA: [ID] — [Title]
Functional requirements: [N]
Impacted components: [list]
New/modified API endpoints: [N]
Data model changes: [yes/no]

Proceeding to architecture design...
Reply 'proceed' or 'pause' to stop here.
```

**⏸ PAUSE — Wait for user to confirm before proceeding to Phase 1b / Phase 2.**

---

### ─── PHASE 1b: Architecture Bootstrap (Conditional) ───

> **This phase only runs if C4 diagrams or flow diagrams are absent or cannot be loaded.**
> If both reports are already present and loaded successfully, skip directly to Phase 2.

#### Check for existing architecture documents

Attempt to read the following files from the backend repository at the specified paths (defaulting to `docs/architecture/`):
- `architecture-analysis-report.md`
- `c4-diagrams-report.md`
- `flow-diagrams-report.md`

Evaluate the results:

| Scenario | Action |
|----------|--------|
| All three files present and non-empty | ✅ Skip Phase 1b — proceed to Phase 2 |
| Any file missing or empty | 🔄 Run Phase 1b to generate missing documents |

If Phase 1b is needed, inform the user:

```
⚠️  Architecture documents not found (or incomplete) in the repository.

Before designing the feature, I need to generate the architecture baseline.
This will run the full Architecture Documentation pipeline against your
backend (and UI, if provided) repository:

  Step 1b-i  → Architecture Analyzer   — deep codebase analysis
  Step 1b-ii → C4 Diagram Generator    — C1/C2/C3, Deployment, Security,
                                          CI/CD, Data Architecture, ERD
  Step 1b-iii→ Flow Diagram Generator  — auth, user journey, API lifecycle,
                                          event, data, CI/CD, error flows

All outputs will be saved to docs/architecture/ in the backend repository.
This is a one-time step — future features will reuse these documents.

Reply 'proceed' to generate architecture docs, or 'skip' to continue
without them (not recommended — design quality will be reduced).
```

**⏸ PAUSE — Wait for user confirmation.**

If the user replies 'proceed':

#### Phase 1b-i: Run Architecture Analyzer

**Invoke Sub-Agent: `architecture-analyzer`**

Pass the following context:
- Backend repository URL and branch
- UI repository URL and branch (if provided)
- UI subpath (if provided)
- Instruction: follow all steps in `.github/agents/architecture-analyzer.md` using the provided inputs without asking the user for inputs again

> **Sub-Agent Execution Boundary**: Steps 2–5 of `architecture-analyzer.md` are executed autonomously.

Once complete, present the Architecture Analysis Report to the user:

```
✅ Phase 1b-i Complete — Architecture Analysis Report generated.

Please do a quick review:
1. Does the report accurately reflect the system? (yes / corrections needed)
2. Any components or flows you want added before I generate diagrams?

Reply 'proceed' to generate C4 diagrams, or provide corrections.
```

**⏸ PAUSE — Wait for user review. Apply any corrections, then proceed.**

#### Phase 1b-ii: Run C4 Diagram Generator

**Invoke Sub-Agent: `c4-diagram-generator`**

Pass the following context:
- Full Architecture Analysis Report (from Phase 1b-i)
- Instruction: follow all steps in `.github/agents/c4-diagram-generator.md` and generate all diagrams from scratch

> **Sub-Agent Execution Boundary**: Steps 3–10b of `c4-diagram-generator.md` are executed autonomously.

Once complete, present the C4 Diagrams Report to the user:

```
✅ Phase 1b-ii Complete — C4 Diagrams Report generated.

Diagrams produced:
  • C1: System Context
  • C2: Container
  • C3: Component — Backend API
  • C3: Component — Frontend (if applicable)
  • Deployment
  • Security Architecture
  • CI/CD Pipeline
  • Data Architecture
  • Database ERD (if ORM entities found)

Please review the diagrams.
Reply 'proceed' to generate flow diagrams, or provide corrections.
```

**⏸ PAUSE — Wait for user review. Apply any corrections, then proceed.**

#### Phase 1b-iii: Run Flow Diagram Generator

**Invoke Sub-Agent: `flow-diagram-generator`**

Pass the following context:
- Full Architecture Analysis Report (from Phase 1b-i, final version after corrections)
- Instruction: follow all steps in `.github/agents/flow-diagram-generator.md` and generate all diagrams from scratch

> **Sub-Agent Execution Boundary**: Steps 3–11 of `flow-diagram-generator.md` are executed autonomously.

Once complete, present the Flow Diagrams Report to the user:

```
✅ Phase 1b-iii Complete — Flow Diagrams Report generated.

Please review the flow diagrams.
Reply 'proceed' to save all documents and continue to design, or provide corrections.
```

**⏸ PAUSE — Wait for user review. Apply any corrections, then proceed.**

#### Phase 1b-iv: Save architecture documents to the backend repository

Save all three generated documents to the backend repository under `docs/architecture/`:
- `docs/architecture/architecture-analysis-report.md`
- `docs/architecture/c4-diagrams-report.md`
- `docs/architecture/flow-diagrams-report.md`

Inform the user:

```
✅ Phase 1b Complete — Architecture baseline created and saved.

  📋 docs/architecture/architecture-analysis-report.md
  📐 docs/architecture/c4-diagrams-report.md
  🔄 docs/architecture/flow-diagrams-report.md

These documents are now committed to the backend repository and will be
reused by all future feature development workflows.

Proceeding to feature design...
```

---

### ─── PHASE 2: Architecture Design (Sub-Agent 2 / 5) ───

**Invoke Sub-Agent: `architecture-design-agent`**

Pass the following context:
- Approved RAD (full document)
- C4 Diagrams Report — either loaded from the repository (pre-existing) or produced in Phase 1b-ii
- Flow Diagrams Report — either loaded from the repository (pre-existing) or produced in Phase 1b-iii
- Architecture Analysis Report — either loaded from the repository (pre-existing) or produced in Phase 1b-i
- Backend repository URL and branch
- UI repository URL and branch (if provided)

Instruct the sub-agent to follow all steps in `.github/agents/architecture-design-agent.md`. The sub-agent must:
1. Load and assess the C4 diagrams and flow diagrams (guaranteed to exist at this point)
2. Inspect relevant codebase files
3. Design the full technical solution (API, service, data, events, infra, frontend, security)
4. Produce Architecture Decision Records (ADRs)
5. Compile the Detailed Design Document (DDD)
6. Present the DDD and ADRs for user review and approval

> **Sub-Agent Execution Boundary**: Steps 2–6 of `architecture-design-agent.md` are executed by this sub-agent. It handles its own review cycle and returns only when the user approves.

**⏸ PAUSE — This phase is complete only when the user says 'approved' on the DDD.**

Once the sub-agent returns an approved DDD, inform the user:

```
✅ Phase 2 Complete — Detailed Design Document approved.

ADRs produced: [N]
  [List each ADR title]
New API endpoints: [N]
Data model changes: [brief description]
Infrastructure changes: [yes/no — brief description]

Proceeding to implementation planning...
Reply 'proceed' or 'pause' to stop here.
```

**⏸ PAUSE — Wait for user to confirm before proceeding to Phase 3.**

---

### ─── PHASE 3: Implementation Planning (Sub-Agent 3) ───

**Invoke Sub-Agent: `implementation-planning-agent`**

Pass the following context:
- Approved DDD (full document)
- Approved RAD (full document)
- Backend repository URL and branch
- UI repository URL and branch (if provided)

Instruct the sub-agent to follow all steps in `.github/agents/implementation-planning-agent.md`. The sub-agent must:
1. Inspect all files that will be touched
2. Produce a fully ordered, file-specific, line-level Implementation Plan
3. Produce a Testing Plan (unit + integration tests)
4. Produce a Migration Checklist and Definition of Done
5. Present the plan for user review and approval

> **Sub-Agent Execution Boundary**: Steps 2–4 of `implementation-planning-agent.md` are executed by this sub-agent.

**⏸ PAUSE — This phase is complete only when the user says 'approved' on the Implementation Plan.**

Once the sub-agent returns an approved plan, inform the user:

```
✅ Phase 3 Complete — Implementation Plan approved.

Total tasks:     [N]
Total sub-tasks: [N]
Estimated total complexity: [total]
Layers: [Backend / Frontend / Infra / Tests]

⚠️  Phase 4 will now write code to your repository.
    Make sure you are working on the correct branch.
    Recommended: create a feature branch before proceeding.

    Current branch: [branch from inputs]
    Suggested feature branch: feature/[JIRA-ID]-[short-description]

Reply 'proceed' to start implementation, or 'pause' to stop here.
```

**⏸ PAUSE — Wait for explicit user confirmation before writing any code.**

---

### ─── PHASE 4: Feature Implementation (Sub-Agent 4) ───

**Invoke Sub-Agent: `feature-implementation-agent`**

Pass the following context:
- Approved Implementation Plan (full document)
- Approved DDD (full document)
- Approved RAD (full document)
- Backend repository URL and branch
- UI repository URL and branch (if provided)

Instruct the sub-agent to follow all steps in `.github/agents/feature-implementation-agent.md`. The sub-agent must:
1. Establish coding conventions by reading existing files
2. Execute every task and sub-task in the Implementation Plan in order
3. Implement all tests from the Testing Plan
4. Perform the DDD compliance and architecture consistency self-checks
5. Produce the Implementation Report

> **Sub-Agent Execution Boundary**: Steps 2–6 of `feature-implementation-agent.md` are executed by this sub-agent. It works through all tasks autonomously and reports progress after each task.

The sub-agent will report progress after completing each task:
```
✅ T-01 complete: [description] — [files changed]
✅ T-02 complete: [description] — [files changed]
...
```

**⏸ PAUSE — If the sub-agent encounters a blocker on any task, it will pause and ask the user for guidance before continuing.**

Once the sub-agent returns the Implementation Report, present it to the user:

```
✅ Phase 4 Complete — Feature Implementation done.

[Implementation Report summary]

Acceptance criteria coverage: [N/N]
Tests added: [N unit + N integration]
Files changed: [N]

Please review the implementation. You can now:
  • Run the test suite to verify
  • Test the feature in your dev environment against the acceptance criteria

Reply 'proceed' to update architecture diagrams, or 'pause' to stop here.
```

**⏸ PAUSE — Wait for user confirmation before updating diagrams.**

---

### ─── PHASE 5: Architecture Diagram Updates (Sub-Agents 5a + 5b) ───

This phase updates the existing C4 diagrams and flow diagrams to reflect the changes introduced by the implemented feature. It uses the diagram update notes from the Implementation Report as input.

#### Phase 5a: Update C4 Diagrams

**Invoke Sub-Agent: `c4-diagram-generator`**

Pass the following context:
- The existing C4 Diagrams Report (current file from the specified path)
- The **Architecture Diagram Update Notes** section from the Implementation Report
- The approved DDD (for component change summary and new API contracts)
- The approved RAD (for system context changes, if any)
- Instruction: **Update existing diagrams — do not regenerate from scratch**

Specific instructions to the sub-agent:
- For the **C3 Backend component diagram**: add any new service, repository, or controller nodes; add new edges for new dependencies
- For the **C3 Frontend component diagram** (if applicable): add new components and routes
- For the **C2 Container diagram**: add new containers only if new deployable units were introduced
- For the **Deployment diagram**: update only if infrastructure changed
- For the **Security diagram**: update if new auth/authz rules or network paths were added
- For the **Data Architecture diagram**: add new data stores or update data flow edges
- For the **ERD**: add new tables and relationships from the migration

Present the updated diagrams for review:

```
C4 Diagrams updated. Changes made:
  • C3 Backend: [list of added/modified nodes]
  • Data Architecture: [list of changes]
  • ERD: [list of new tables/relationships]

Please review the updated diagrams.
Reply 'approved' or provide corrections.
```

**⏸ PAUSE — Wait for user approval of updated C4 diagrams.**

#### Phase 5b: Update Flow Diagrams

**Invoke Sub-Agent: `flow-diagram-generator`**

Pass the following context:
- The existing Flow Diagrams Report (current file from the specified path)
- The **New Flow Notes** section from the Implementation Report
- The approved DDD (for new sequence diagrams defined in the design)
- Instruction: **Add new diagrams and update modified flows — do not regenerate existing unrelated flows**

Specific instructions to the sub-agent:
- Add new sequence diagrams for any new user journeys introduced by this feature
- Add new flow diagrams for any new event-driven flows
- Update existing flows that were modified (e.g., an existing API now has a new step)
- Do not modify flows unrelated to this feature

Present the updated flow diagrams for review:

```
Flow Diagrams updated. Changes made:
  • New flows added: [list]
  • Modified flows: [list]

Please review the updated flow diagrams.
Reply 'approved' or provide corrections.
```

**⏸ PAUSE — Wait for user approval of updated flow diagrams.**

---

### ─── PHASE 6: Workflow Complete ───

Once all phases are approved, present the final summary:

```
🎉 Feature Development Workflow Complete!

┌──────────────────────────────────────────────────────────────┐
│  JIRA: [ID] — [Title]                                        │
│                                                              │
│  ✅ Phase 1 — Requirements Analysis       APPROVED           │
│  ✅ Phase 2 — Architecture Design         APPROVED           │
│  ✅ Phase 3 — Implementation Planning     APPROVED           │
│  ✅ Phase 4 — Feature Implementation      COMPLETE           │
│  ✅ Phase 5 — Architecture Diagrams       UPDATED            │
└──────────────────────────────────────────────────────────────┘

Documents produced / updated:
  📋  Requirements Analysis Document (RAD)
  📐  Detailed Design Document (DDD) + [N] ADRs
  🗂️  Implementation Plan
  💻  [N] files created/modified (implementation)
  📊  C4 Diagrams Report (updated)
  🔄  Flow Diagrams Report (updated)

Next steps:
  1. Create a Pull Request from your feature branch
  2. Include links to the RAD and DDD in the PR description
  3. Update the JIRA ticket status to "In Review"
  4. Request peer review

Would you like me to:
  [A] Save the RAD, DDD, and Implementation Plan to docs/architecture/features/[JIRA-ID]/
  [B] Generate a Pull Request description from the RAD and DDD
  [C] Both A and B
  [D] No further action needed
```

**⏸ PAUSE — Wait for user's final choice.**

- **A**: Save RAD, DDD, and Implementation Plan as markdown files to `docs/architecture/features/[JIRA-ID]/`
- **B**: Generate a PR description with: feature summary, acceptance criteria checklist, architecture change summary, testing notes, and links to all documents
- **C**: Do both A and B
- **D**: End the workflow

---

## Sub-Agent Invocation Reference

| Phase | Sub-Agent File | Condition | Key Inputs | Key Output |
|-------|---------------|-----------|-----------|------------|
| 1 | `.github/agents/requirement-analyzer.md` | Always | JIRA ID, repo URLs, existing arch docs | Approved RAD |
| 1b-i | `.github/agents/architecture-analyzer.md` | Only if arch docs missing | Repo URLs, branches | Architecture Analysis Report |
| 1b-ii | `.github/agents/c4-diagram-generator.md` | Only if C4 report missing | Architecture Analysis Report | C4 Diagrams Report (saved to repo) |
| 1b-iii | `.github/agents/flow-diagram-generator.md` | Only if flow report missing | Architecture Analysis Report | Flow Diagrams Report (saved to repo) |
| 2 | `.github/agents/architecture-design-agent.md` | Always | RAD, C4/flow docs, repo URLs | Approved DDD + ADRs |
| 3 | `.github/agents/implementation-planning-agent.md` | Always | DDD, RAD, repo URLs | Approved Implementation Plan |
| 4 | `.github/agents/feature-implementation-agent.md` | Always | Plan, DDD, RAD, repo URLs | Code changes + Implementation Report |
| 5a | `.github/agents/c4-diagram-generator.md` | Always | Existing C4 report + diagram update notes | Updated C4 Diagrams Report |
| 5b | `.github/agents/flow-diagram-generator.md` | Always | Existing flow report + new flow notes | Updated Flow Diagrams Report |

---

## Pause Points Summary

| # | Phase | Condition | Trigger | User Must Provide |
|---|-------|-----------|---------|------------------|
| 1 | Phase 0 | Always | Start | JIRA ID, repo URLs, arch doc paths |
| 2 | Phase 0 | Always | Input summary | Confirm inputs are correct |
| 3 | Phase 1 (internal) | Always | Q&A rounds | Answers to clarifying questions |
| 4 | Phase 1 | Always | RAD presented | 'approved' on Requirements Analysis Document |
| 5 | Phase 1 → 1b | Always | Phase 1 summary | 'proceed' or 'pause' |
| 6 | Phase 1b | Arch docs missing | Bootstrap confirmation | 'proceed' to generate or 'skip' |
| 7 | Phase 1b-i | Arch docs missing | Analysis Report review | Confirm report or corrections + 'proceed' |
| 8 | Phase 1b-ii | Arch docs missing | C4 Diagrams review | Confirm diagrams or corrections + 'proceed' |
| 9 | Phase 1b-iii | Arch docs missing | Flow Diagrams review | Confirm diagrams or corrections + 'proceed' |
| 10 | Phase 2 | Always | DDD + ADRs presented | 'approved' on Detailed Design Document |
| 11 | Phase 2 → 3 | Always | Phase 2 summary | 'proceed' or 'pause' |
| 12 | Phase 3 | Always | Implementation Plan presented | 'approved' on plan |
| 13 | Phase 3 → 4 | Always | Phase 3 summary + branch warning | 'proceed' or 'pause' |
| 14 | Phase 4 | Always | Blocker encountered (if any) | Guidance to unblock |
| 15 | Phase 4 → 5 | Always | Implementation Report | 'proceed' or 'pause' |
| 16 | Phase 5a | Always | Updated C4 diagrams | 'approved' or corrections |
| 17 | Phase 5b | Always | Updated flow diagrams | 'approved' or corrections |
| 18 | Phase 6 | Always | Final summary | Output/save choice |

---

## Architecture as Code Guarantee

This workflow enforces the following invariants:

1. **Architecture baseline always exists before design** — Phase 1b ensures C4 and flow diagrams are generated and saved to the repository if they are absent. Design (Phase 2) never runs without them.
2. **No implementation without approved design** — Phase 4 cannot start until Phase 2 (DDD) and Phase 3 (Plan) are both approved
3. **No design without analyzed requirements** — Phase 2 cannot start until Phase 1 (RAD) is approved
4. **No diagrams skipped** — Phase 5 always runs after implementation, regardless of how small the feature is
5. **C4 diagrams always updated** — every feature that adds components, endpoints, or data flows must update the C4 diagrams
6. **Flow diagrams always updated** — every feature that adds or modifies a user journey, API flow, or event flow must update the flow diagrams
7. **Architecture docs are repo-resident** — all baseline and updated docs are saved to `docs/architecture/` in the backend repository so they are available to every future workflow run without regeneration

---

## Error Handling

- If a JIRA ticket cannot be fetched, ask the user to paste the ticket content manually
- If a repository cannot be accessed, ask for alternative credentials or a local path
- If any sub-agent produces incomplete output, inform the user immediately and ask to retry or skip
- If the Implementation Plan cannot be executed for a sub-task (e.g., a file does not exist at the expected path), pause and ask the user for guidance before continuing
- Never silently skip a sub-task — always report blockers explicitly
