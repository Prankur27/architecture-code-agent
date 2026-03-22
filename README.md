# Architecture Code Agent

A set of GitHub Copilot agents that automatically analyze software repositories, generate comprehensive architecture documentation, and drive feature development end-to-end — from JIRA ticket to working, tested, documented code.

---

## Overview

This project contains **two pipelines** built from specialized AI agents:

### Pipeline 1 — Architecture Documentation
Analyzes an existing codebase and generates C4 diagrams and flow diagrams.

| Agent | File | Purpose |
|-------|------|---------|
| 🔍 **Architecture Analyzer** | `.github/agents/architecture-analyzer.md` | Analyzes backend and UI codebases, CloudFormation, CI/CD, security tools, and NFRs |
| 📐 **C4 Diagram Generator** | `.github/agents/c4-diagram-generator.md` | Generates C4 model diagrams (C1–C3, Deployment, Security, CI/CD, Data, ERD) |
| 🔄 **Flow Diagram Generator** | `.github/agents/flow-diagram-generator.md` | Generates sequence and flow diagrams for all system flows |

### Pipeline 2 — Feature Development (Architecture as Code)
Takes a JIRA ticket and drives it all the way to implemented, tested code with updated architecture diagrams.

| Agent | File | Purpose |
|-------|------|---------|
| 📋 **Requirement Analyzer** | `.github/agents/requirement-analyzer.md` | Fetches JIRA ticket, deeply analyzes requirements, iterates Q&A until agreed |
| 🏗️ **Architecture Design Agent** | `.github/agents/architecture-design-agent.md` | Produces detailed technical design (API, service, data, infra) + ADRs |
| 🗂️ **Implementation Planning Agent** | `.github/agents/implementation-planning-agent.md` | Breaks design into file-specific, ordered task plan |
| 💻 **Feature Implementation Agent** | `.github/agents/feature-implementation-agent.md` | Executes every task, writes production-quality code + tests |

---

## Prerequisites

- **GitHub Copilot** with agent/chat mode enabled in VS Code or GitHub.com
- Access to the backend and (optionally) UI repositories you want to analyze or develop against
- For Pipeline 2: JIRA access (URL + credentials) for the project containing your epics/stories

---

## Quick Start — Pipeline 1: Architecture Documentation

Use the **Architecture Documentation Workflow Skill** to orchestrate all three documentation agents in sequence.

```
Use .github/skills/architecture-documentation-workflow.md
```

The skill will:
1. Collect all inputs (repos, branches, UI subpath) in one go
2. Run the **Architecture Analyzer** → pause for your review
3. Run the **C4 Diagram Generator** → pause for your review
4. Run the **Flow Diagram Generator** → pause for your review
5. Offer to save all outputs as markdown files in your repository

**Pause points**: Stops at 6 checkpoints to ask for your input or confirmation.

---

## Quick Start — Pipeline 2: Feature Development

Use the **Feature Development Workflow Skill** to drive a JIRA ticket all the way through to implementation and updated architecture diagrams.

```
Use .github/skills/feature-development-workflow.md
```

The skill will:
1. Collect JIRA ticket ID, repo URLs, and architecture doc paths
2. Run the **Requirement Analyzer** — fetches JIRA, analyzes against codebase, iterates Q&A → RAD approved
3. Run the **Architecture Design Agent** — designs the solution using existing C4/flow diagrams → DDD + ADRs approved
4. Run the **Implementation Planning Agent** — produces ordered, file-specific task plan → Plan approved
5. Run the **Feature Implementation Agent** — writes all code and tests task by task
6. Run the **C4 Diagram Generator** — updates existing diagrams with new components/flows
7. Run the **Flow Diagram Generator** — adds new flow diagrams for new user journeys

**Architecture as Code**: Every feature goes through design → approval → code → diagram update. Architecture docs are always kept in sync.

**Pause points**: Stops at 14 checkpoints — every approval gate requires your explicit 'approved' before proceeding.

---

## Usage — Running Agents Individually

If you prefer to run each agent independently instead of using the workflow skills:

### Pipeline 1 Agents

#### Step 1 — Run the Architecture Analyzer Agent

1. Open GitHub Copilot Chat and reference the agent:
   ```
   Use the instructions in .github/agents/architecture-analyzer.md
   ```
2. Provide: backend repo URL, branch, UI repo URL (optional), UI branch, UI subpath (optional)
3. The agent produces an **Architecture Analysis Report** covering architecture, infrastructure, CI/CD, security, NFRs, and data flows.
4. **Save the output** — required as input for Steps 2 and 3.

#### Step 2 — Run the C4 Diagram Generator Agent

1. Reference the agent and attach the Architecture Analysis Report:
   ```
   Use the instructions in .github/agents/c4-diagram-generator.md
   [paste Architecture Analysis Report]
   ```
2. Produces a **C4 Diagrams Report** with: C1 System Context, C2 Container, C3 Component (Backend + Frontend), Deployment, Security, CI/CD, Data Architecture, and ERD diagrams.

#### Step 3 — Run the Flow Diagram Generator Agent

1. Reference the agent and attach the Architecture Analysis Report:
   ```
   Use the instructions in .github/agents/flow-diagram-generator.md
   [paste Architecture Analysis Report]
   ```
2. Produces a **Flow Diagrams Report** with: auth flows, user journeys, API lifecycle, event-driven flows, data flows, CI/CD flows, and error handling flows.

---

### Pipeline 2 Agents

#### Step 1 — Requirement Analyzer
```
Use .github/agents/requirement-analyzer.md

JIRA ticket: PROJ-123
JIRA URL: https://your-org.atlassian.net
Backend repo: https://github.com/org/backend
Backend branch: main
Architecture docs: docs/architecture/
```
Fetches the JIRA ticket, analyzes requirements against the codebase, asks clarifying questions in rounds, and produces an approved Requirements Analysis Document (RAD).

#### Step 2 — Architecture Design Agent
```
Use .github/agents/architecture-design-agent.md
[paste approved RAD]
[attach C4 diagrams report path]
[attach flow diagrams report path]
```
Designs the full technical solution (API, service, data, events, infra, frontend) and produces ADRs.

#### Step 3 — Implementation Planning Agent
```
Use .github/agents/implementation-planning-agent.md
[paste approved DDD]
[paste approved RAD]
```
Produces an ordered, file-specific task plan with complexity estimates and a testing plan.

#### Step 4 — Feature Implementation Agent
```
Use .github/agents/feature-implementation-agent.md
[paste approved Implementation Plan]
[paste approved DDD]
[paste approved RAD]
```
Executes every task in the plan, writes production-quality code and tests, and produces an implementation report.

---

## Agent & Skill Files

```
.github/
├── agents/
│   │
│   │── Pipeline 1: Architecture Documentation ────────────────
│   ├── architecture-analyzer.md           # Codebase analysis
│   ├── c4-diagram-generator.md            # C4 model diagrams (C1/C2/C3 + supplementary)
│   ├── flow-diagram-generator.md          # Flow & sequence diagrams
│   │
│   └── Pipeline 2: Feature Development ───────────────────────
│       ├── requirement-analyzer.md        # JIRA fetch + requirement Q&A → RAD
│       ├── architecture-design-agent.md   # Technical design + ADRs → DDD
│       ├── implementation-planning-agent.md  # Ordered task plan → Implementation Plan
│       └── feature-implementation-agent.md   # Code + tests → Implementation Report
│
└── skills/
    ├── architecture-documentation-workflow.md  # Orchestrates Pipeline 1 (3 agents)
    └── feature-development-workflow.md         # Orchestrates Pipeline 2 (5 agents + diagram updates)
```

---

## Diagram Standards

All generated Mermaid diagrams follow these conventions:

| Convention | Rule |
|------------|------|
| **C1 System Context** | `C4Context` format |
| **C2 Container** | `C4Container` format |
| **C3 Component** | `flowchart LR` with `classDef` colour coding |
| **Deployment** | `flowchart TB` flat zones (no nested `C4Deployment`) |
| **Security / CI/CD / Data** | `flowchart LR` with named subgraph zones + `classDef` |
| **ERD** | `erDiagram` format |
| **Sequence diagrams** | `sequenceDiagram` with `autonumber` |
| **Colour coding** | 10-colour standard palette via `classDef` (never inline `style`) |

To render Mermaid diagrams locally in VS Code, install the [Mermaid Preview](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid) extension.

---

## What the Agents Analyze

### Architecture Analyzer covers:
- ✅ Backend code structure (controllers, services, repositories, models)
- ✅ API surface (REST, GraphQL, gRPC, WebSocket)
- ✅ Databases, caches, message brokers
- ✅ CloudFormation YAML templates (all AWS resources, VPC, IAM, networking)
- ✅ GitHub Actions workflows (CI/CD pipeline stages)
- ✅ Veracode configuration (SAST, SCA, pipeline scans)
- ✅ SonarQube / SonarCloud (quality gates, rules)
- ✅ Authentication & Authorization (JWT, OAuth2/OIDC, Cognito, Auth0)
- ✅ Secrets management (Secrets Manager, Parameter Store, Vault)
- ✅ Network security (security groups, WAF, TLS, CORS)
- ✅ Data encryption (KMS, at-rest, in-transit)
- ✅ Performance, scalability, observability, and disaster recovery
- ✅ UI code (framework, state management, API integration, security)

### Requirement Analyzer covers:
- ✅ JIRA epic/story fetch with all fields, acceptance criteria, child stories
- ✅ Deep requirement analysis (clarity, completeness, feasibility, security, domain)
- ✅ Iterative Q&A rounds until all gaps are resolved
- ✅ Impacted component mapping against actual codebase
- ✅ Proposed API contract and data model changes


## Output Formats

All agents produce **Markdown** output with embedded **Mermaid** diagrams.

To render Mermaid diagrams locally in VS Code, install the [Mermaid Preview](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid) extension.
