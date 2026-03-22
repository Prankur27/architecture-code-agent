# Architecture Code Agent

A set of GitHub Copilot agents that automatically analyze software repositories and generate comprehensive architecture documentation, C4 diagrams, and flow diagrams.

---

## Overview

This project contains **three specialized AI agents** that work together to produce a full architectural view of your system:

| Agent | File | Purpose |
|-------|------|---------|
| 🔍 **Architecture Analyzer** | `.github/agents/architecture-analyzer.md` | Analyzes backend and UI codebases, CloudFormation, CI/CD, security tools, and NFRs |
| 📐 **C4 Diagram Generator** | `.github/agents/c4-diagram-generator.md` | Generates C4 model diagrams (Context, Container, Component, Deployment, Security, CI/CD, Data) |
| 🔄 **Flow Diagram Generator** | `.github/agents/flow-diagram-generator.md` | Generates sequence and flow diagrams for all system flows |

---

## Prerequisites

- **GitHub Copilot** with agent/chat mode enabled in VS Code or GitHub.com
- Access to the backend and (optionally) UI repositories you want to analyze

---

## Quick Start — Run the Full Workflow (Recommended)

Instead of running each agent manually, use the **Architecture Documentation Workflow Skill** to orchestrate all three agents automatically in sequence with guided pauses for your input and review.

```
Use .github/skills/architecture-documentation-workflow.md
```

The skill will:
1. Collect all inputs (repos, branches, UI subpath) in one go
2. Run the **Architecture Analyzer** sub-agent → pause for your review
3. Run the **C4 Diagram Generator** sub-agent → pause for your review
4. Run the **Flow Diagram Generator** sub-agent → pause for your review
5. Offer to save all outputs as markdown files in your repository

**Pause points**: The workflow stops at 6 checkpoints to ask for your input or confirmation before proceeding, ensuring you stay in control throughout.

---

## Usage — Running Agents Individually

If you prefer to run each agent independently instead of using the workflow skill:

### Step 1 — Run the Architecture Analyzer Agent

This is the **first agent** and must be run before the others. It performs deep analysis of your codebase.

1. Open GitHub Copilot Chat
2. Reference the agent file:
   ```
   @workspace /agent .github/agents/architecture-analyzer.md
   ```
   Or in GitHub Copilot agent mode, attach the file and prompt:
   ```
   Use the instructions in .github/agents/architecture-analyzer.md to analyze my system.
   ```
3. When prompted, provide:
   - **Backend repository path or URL** *(required)*
   - **UI repository path or URL** *(optional)*
   - **UI subpath** *(optional)* — e.g., `apps/dashboard` if the UI repo is a monorepo with multiple projects. Only this subpath will be analyzed.

4. The agent will produce an **Architecture Analysis Report** in markdown covering:
   - System architecture & patterns
   - Cloud infrastructure (CloudFormation / AWS resources)
   - CI/CD pipelines (GitHub Actions)
   - Security posture (Veracode SAST/SCA, SonarQube, secrets management, IAM)
   - Non-functional requirements (performance, scalability, observability, DR)
   - Data flows and integration points
   - Architecture risks and recommendations

5. **Save the output** — you will need it for the next two agents.

---

### Step 2 — Run the C4 Diagram Generator Agent

This agent takes the Architecture Analysis Report and produces structured C4 model diagrams.

1. Open GitHub Copilot Chat
2. Reference the agent file:
   ```
   Use the instructions in .github/agents/c4-diagram-generator.md
   ```
3. Paste or attach the **Architecture Analysis Report** from Step 1 as context.
4. The agent will generate a **C4 Diagrams Report** containing:
   - **L1 System Context Diagram** — system in its environment with users and external systems
   - **L2 Container Diagram** — all deployable units (apps, services, databases, brokers)
   - **L3 Component Diagrams** — internals of the backend API and frontend application
   - **Deployment Diagram** — AWS infrastructure topology (VPC, subnets, ECS, RDS, Lambda, etc.)
   - **Security Architecture Diagram** — auth flows, encryption, WAF, IAM, secrets
   - **CI/CD Pipeline Diagram** — build → test → scan → deploy pipeline
   - **Data Architecture Diagram** — data stores, ownership, replication, encryption

5. All diagrams are in **Mermaid** format and can be rendered in:
   - GitHub Markdown (automatically rendered)
   - VS Code with the Mermaid Preview extension
   - Confluence with the Mermaid plugin
   - [mermaid.live](https://mermaid.live)

---

### Step 3 — Run the Flow Diagram Generator Agent

This agent takes the Architecture Analysis Report and generates detailed flow and sequence diagrams for all system flows.

1. Open GitHub Copilot Chat
2. Reference the agent file:
   ```
   Use the instructions in .github/agents/flow-diagram-generator.md
   ```
3. Paste or attach the **Architecture Analysis Report** from Step 1 as context.
4. The agent will generate a **Flow Diagrams Report** containing:
   - **Authentication & Authorization flows** (login, token refresh, RBAC)
   - **User journey flows** for each key business scenario (end-to-end UI → API → DB)
   - **API request lifecycle** — how requests traverse middleware, services, and data layers
   - **Event-driven / async flows** — message publishing, consumption, retry, DLQ
   - **Data flows** — ingestion, transformation, export
   - **CI/CD & deployment flows** — from commit to production
   - **Infrastructure provisioning flows** — CloudFormation stack lifecycle
   - **Error & failure handling flows** — circuit breaker, retry with backoff, DLQ processing

---

## Example Prompts

### For the Architecture Analyzer
```
Use .github/agents/architecture-analyzer.md instructions.

Backend repo: https://github.com/my-org/my-backend
Backend branch: main
UI repo: https://github.com/my-org/my-monorepo
UI branch: develop
UI subpath: apps/customer-portal

Analyze the full architecture of this system.
```

### For the C4 Diagram Generator
```
Use .github/agents/c4-diagram-generator.md instructions.

Here is the Architecture Analysis Report:
[paste report from Step 1]

Generate all C4 diagrams for this system.
```

### For the Flow Diagram Generator
```
Use .github/agents/flow-diagram-generator.md instructions.

Here is the Architecture Analysis Report:
[paste report from Step 1]

Generate all flow diagrams for this system.
```

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
- ✅ Dependency scanning (Dependabot, Snyk, OWASP)
- ✅ Authentication & Authorization (JWT, OAuth2/OIDC, Cognito, Auth0)
- ✅ Secrets management (Secrets Manager, Parameter Store, Vault)
- ✅ Network security (security groups, WAF, TLS, CORS)
- ✅ Data encryption (KMS, at-rest, in-transit)
- ✅ Performance (caching, connection pooling, async patterns)
- ✅ Scalability (auto-scaling, serverless, horizontal scaling)
- ✅ Observability (logging, metrics, tracing, alerting)
- ✅ Disaster recovery (backup, RTO/RPO, multi-region)
- ✅ UI code (framework, state management, API integration, security)

---

## Agent & Skill Files

```
.github/
├── agents/
│   ├── architecture-analyzer.md    # Agent 1: Codebase analysis
│   ├── c4-diagram-generator.md     # Agent 2: C4 model diagrams
│   └── flow-diagram-generator.md   # Agent 3: Flow & sequence diagrams
└── skills/
    └── architecture-documentation-workflow.md  # Orchestrates all 3 agents in sequence
```

---

## Output Formats

All agents produce **Markdown** output with embedded **Mermaid** diagrams.

To render Mermaid diagrams locally in VS Code, install the [Mermaid Preview](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid) extension.
