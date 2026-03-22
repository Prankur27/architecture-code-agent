# C4 Architecture Diagrams — Agent Profiles (NICE CXone)
Generated from Architecture Analysis Report — March 22, 2026

---

## Diagram Index
1. [C1: System Context Diagram](#1-c1-system-context-diagram)
2. [C2: Container Diagram](#2-c2-container-diagram)
3. [C3: Component Diagram — Backend API](#3-c3-component-diagram--backend-api)
4. [C3: Component Diagram — Frontend Application](#4-c3-component-diagram--frontend-application)
5. [Deployment Diagram](#5-deployment-diagram)
6. [Security Architecture Diagram](#6-security-architecture-diagram)
7. [CI/CD Pipeline Diagram](#7-cicd-pipeline-diagram)
8. [Data Architecture Diagram](#8-data-architecture-diagram)
9. [Database Design (ERD)](#9-database-design-entity-relationship-diagram)

---

## 1. C1: System Context Diagram

```mermaid
C4Context
  title C1: System Context Diagram — Agent Profiles (NICE CXone)

  Person(adminUser, "Contact Center Admin", "Configures agent desktop profiles and assigns them to teams via the CXone Admin UI")
  Person(cxaAgent, "CXone Agent (CXA)", "Reads their assigned agent profile at login to configure their desktop experience")

  System(agentProfilesSystem, "Agent Profiles System", "Manages agent desktop profile configurations. Allows admins to create, update, assign profiles to teams. Exposes profile data to CXone Agent.")

  System_Ext(cxoneShell, "CXone Platform Shell", "NICE CXone web shell application. Hosts micro-frontends. Manages global auth, navigation, tenant context.")
  System_Ext(cxonePlatformAPI, "CXone Platform API", "NICE CXone platform services. Provides user permissions, agent details, team details at v31.0.")
  System_Ext(acdAPI, "ACD API", "Legacy inContact ACD API (api-na1.dev.niceincontact.com). Used for legacy agent/team integrations.")
  System_Ext(cxoneAuth, "CXone Auth / JWKS", "CXone identity provider. Issues and validates JWTs. Exposes JWKS endpoint for token validation.")
  System_Ext(kinesisStreams, "AWS Kinesis Data Streams", "Event streaming platform. Publishes TeamData and userManagement change events consumed by Agent Profiles.")
  System_Ext(otlpCollector, "OTLP Telemetry Collector", "collector-mon.nicecxone-dev.com. Receives OpenTelemetry metrics and traces from the service.")

  Rel(adminUser, cxoneShell, "Uses", "HTTPS/Browser")
  Rel(cxoneShell, agentProfilesSystem, "Embeds and calls", "HTTPS/REST (MFE)")
  Rel(cxaAgent, agentProfilesSystem, "Fetches profile at login", "HTTPS/REST")
  Rel(agentProfilesSystem, cxonePlatformAPI, "Fetches user permissions, agents, teams", "HTTPS/REST")
  Rel(agentProfilesSystem, acdAPI, "Legacy agent/team queries", "HTTPS/REST")
  Rel(agentProfilesSystem, cxoneAuth, "Validates JWT tokens via JWKS", "HTTPS")
  Rel(kinesisStreams, agentProfilesSystem, "Pushes TeamData + userManagement events", "Kinesis/KCL")
  Rel(agentProfilesSystem, otlpCollector, "Exports metrics and traces", "OTLP/HTTPS")
```

**Description**: The Agent Profiles system sits at the centre of the CXone contact centre platform. Contact Centre Admins interact with it exclusively through the CXone Shell (which hosts the Angular micro-frontend). CXone Agents (desktop app) call the system directly to retrieve their profile at login. The system depends on the CXone Platform API for authorization and entity data, consumes team/user lifecycle events from Kinesis, and emits observability data to an OTLP collector.

| Element | Type | Description |
|---------|------|-------------|
| Contact Center Admin | Person | Creates and manages agent profiles and team assignments |
| CXone Agent (CXA) | Person/System | Consumes assigned profile data at session start |
| Agent Profiles System | Software System | Core subject of this documentation |
| CXone Platform Shell | External System | MFE host; provides auth context and navigation |
| CXone Platform API | External System | Source of user permissions and entity data |
| ACD API | External System | Legacy contact centre API |
| CXone Auth / JWKS | External System | JWT issuer and JWKS validator |
| AWS Kinesis Data Streams | External System | Event source for team/user changes |
| OTLP Telemetry Collector | External System | Observability sink |

---

## 2. C2: Container Diagram

```mermaid
C4Container
  title C2: Container Diagram — Agent Profiles System

  Person(adminUser, "Contact Center Admin", "")
  Person(cxaAgent, "CXone Agent", "")

  System_Boundary(agentProfilesSystem, "Agent Profiles System") {

    Container(angularMFE, "Agent Profiles MFE", "Angular 14+ / Web Component / Module Federation", "Admin UI for creating, editing, and managing agent profiles and team assignments. Deployed as a Web Component (agent-profile-app) embedded in the CXone Shell.")

    Container(backendAPI, "Agent Profiles API", "Java 17 / Spring Boot / Gradle / Jib → ECS", "Core REST API. Handles all CRUD operations for agent profiles, team assignments, and assignable configurations. Enforces RBAC via AOP. Exposes CXA-specific endpoints.")

    Container(kinesisConsumer, "Kinesis Event Consumers", "Java 17 / KCL (embedded in API process)", "Two KCL workers consuming TeamData and userManagement Kinesis streams. Syncs team renames/deletes and agent lifecycle changes into the DB.")

    ContainerDb(auroraDB, "Aurora MySQL Cluster", "AWS Aurora MySQL 8.0 (aurora-mysql 3.08.2) — 3 instances", "Primary data store. Stores agent profiles, configurations, team mappings, assignable config catalog. Schema-per-tenant multi-tenancy. Write + read replica endpoints.")

    ContainerDb(valkeyCache, "Valkey Cache", "AWS ElastiCache Valkey (Redis-compatible) — Multi-AZ", "API response cache. Caches profile data and JWKS keys to reduce latency and upstream load. TLS configurable per environment.")

    ContainerDb(dynamoDB, "DynamoDB", "AWS DynamoDB", "Feature toggle store. Consulted at runtime via /config/toggledFeatures endpoint.")

    Container(secretsManager, "Secrets Manager", "AWS Secrets Manager", "Stores DB credentials (username/password). Referenced by CloudFormation at deploy time.")

    Container(ssmParams, "SSM Parameter Store", "AWS SSM", "Stores runtime config: DB endpoints, FIPS flag, encryption-in-transit flag, Valkey host/port.")
  }

  System_Ext(cxoneShell, "CXone Platform Shell", "MFE host application")
  System_Ext(cxonePlatformAPI, "CXone Platform API", "User permissions, agents, teams — v31.0")
  System_Ext(cxoneAuth, "CXone Auth / JWKS", "JWT issuer + JWKS endpoint")
  System_Ext(kinesisStreams, "AWS Kinesis Data Streams", "TeamData + userManagement streams")
  System_Ext(otlpCollector, "OTLP Collector", "Metrics + traces sink")

  Rel(adminUser, cxoneShell, "Uses browser", "HTTPS")
  Rel(cxoneShell, angularMFE, "Loads and embeds MFE", "Module Federation / HTTPS")
  Rel(angularMFE, backendAPI, "REST API calls", "HTTPS/REST — /agent-profiles/v1")
  Rel(cxaAgent, backendAPI, "Fetch agent profile at login", "HTTPS/REST — /agent-profiles/v1/cxa/profiles/{id}")

  Rel(backendAPI, auroraDB, "Read/Write profile data (write replica)", "JDBC/MySQL TLS — port 3306")
  Rel(backendAPI, auroraDB, "Read-only queries (read replica)", "JDBC/MySQL TLS — port 3306")
  Rel(backendAPI, valkeyCache, "Cache get/set for responses + JWKS", "Redis protocol / TLS — port 6379")
  Rel(backendAPI, dynamoDB, "Read feature toggles", "AWS SDK HTTPS")
  Rel(backendAPI, secretsManager, "Reads DB credentials at startup", "AWS SDK HTTPS")
  Rel(backendAPI, ssmParams, "Reads runtime config at startup", "AWS SDK HTTPS")
  Rel(backendAPI, cxonePlatformAPI, "Fetch user permissions, agent/team data", "HTTPS/REST")
  Rel(backendAPI, cxoneAuth, "Validate JWT via JWKS", "HTTPS")
  Rel(backendAPI, otlpCollector, "Export OTLP metrics + traces", "OTLP/HTTPS")

  Rel(kinesisStreams, kinesisConsumer, "TeamData + userManagement events", "Kinesis KCL")
  Rel(kinesisConsumer, auroraDB, "Sync team/user changes to DB", "JDBC/MySQL TLS")
```

**Description**: The backend API and Kinesis consumers run as a single deployable Spring Boot process on ECS Fargate. The Angular micro-frontend is a stateless SPA deployed via CDN/S3, embedded into the CXone Shell via Module Federation as a Web Component. Aurora MySQL provides durable multi-tenant storage; Valkey provides caching; DynamoDB stores feature flags. All secrets and config are externalized to AWS-managed stores.

| Container | Technology | Responsibilities |
|-----------|-----------|-----------------|
| Agent Profiles MFE | Angular, Web Component, Webpack MFE | Admin UI — profile list, create/edit, team assignment |
| Agent Profiles API | Java 17, Spring Boot, ECS Fargate | REST API, RBAC, business logic, external integrations |
| Kinesis Event Consumers | Java 17, KCL (embedded) | Async event consumption for team/user data sync |
| Aurora MySQL Cluster | AWS RDS Aurora MySQL 8 | Primary persistent store (multi-tenant, encrypted) |
| Valkey Cache | AWS ElastiCache Valkey | Response + JWKS caching |
| DynamoDB | AWS DynamoDB | Feature toggles |
| Secrets Manager | AWS Secrets Manager | DB credential storage |
| SSM Parameter Store | AWS SSM | Runtime configuration |

---

## 3. C3: Component Diagram — Backend API

```mermaid
flowchart LR
    %% ── External Systems ───────────────────────────────────────────────
    subgraph EXT["External Systems"]
        direction TB
        CX_AUTH(["CXone Auth\nJWKS Endpoint"])
        CX_PLAT(["CXone Platform API\n/user/permissions\n/agents  /teams"])
        KINESIS_IN(["AWS Kinesis\nTeamData + userManagement"])
    end

    %% ── Controller Layer ────────────────────────────────────────────────
    subgraph CTL["Controller Layer  [Spring @RestController]"]
        direction TB
        C1["AgentProfilesController\n/profiles — CRUD + teams"]
        C2["AssignableConfigurationsController\n/assignable-configurations"]
        C3["GetAgentProfileForCXAController\n/cxa/profiles  (v1)"]
        C4["GetAgentProfileCXAControllerV2\n/cxa/v2/profiles  (v2)"]
    end

    %% ── Auth / AOP Layer ────────────────────────────────────────────────
    subgraph AUTH["Auth Layer  [Spring AOP]"]
        direction TB
        AOP["AuthAspect\n@Around @Authorized"]
        ASVC["AuthService\nJWT parse · claims extract\nNimbus JOSE+JWT"]
        JWKS_C["JWKS Cache\nL1 Caffeine 6-12h\nL2 Persistent 15-30d"]
        BAD["Bad Kid Cache\nCaffeine 5min"]
    end

    %% ── Service Layer ───────────────────────────────────────────────────
    subgraph SVC["Service Layer  [Spring @Service]"]
        direction TB
        AGSVC["AgentProfilesServiceImpl\nCore business logic"]
        CSVC["CacheResponseServiceImpl\nCache-aside read/write/invalidate"]
        PSVC["PlatformApiConsumerServiceImpl\nHTTP client + Circuit Breaker"]
    end

    %% ── Repository Layer ────────────────────────────────────────────────
    subgraph REPO["Repository Layer  [Spring @Repository / JPA]"]
        direction TB
        APDAO["AgentProfilesDaoImpl\nRead/write replica routing"]
        TMDao["AgentProfileTeamsMapDaoImpl\nJunction table access"]
    end

    %% ── Kinesis Consumer ────────────────────────────────────────────────
    subgraph KCL["Kinesis Consumer Layer  [KCL Workers]"]
        direction TB
        KCON["KinesisConsumer\nKCL worker bootstrap"]
        TPROC["TeamDataRecordProcessor\nTeam rename/delete sync"]
        UPROC["UserManagementRecordProcessor\nAgent hire/term/role sync"]
    end

    %% ── Data Stores ─────────────────────────────────────────────────────
    subgraph DATA["Data Stores"]
        direction TB
        AURORA[("Aurora MySQL\nWriter + Reader")]
        VALKEY[("Valkey Cache\nPort 6379")]
        DYNAMO[("DynamoDB\nFeature Toggles")]
    end

    %% ── Edges: Request path ──────────────────────────────────────────────
    C1 -->|"AOP intercept"| AOP
    C2 -->|"AOP intercept"| AOP
    C3 -->|"AOP intercept"| AOP
    C4 -->|"AOP intercept"| AOP

    AOP --> ASVC
    ASVC --> JWKS_C
    ASVC --> BAD
    JWKS_C -->|"cache miss"| CX_AUTH
    ASVC -->|"permissions"| PSVC

    C1 --> AGSVC
    C2 --> AGSVC
    C3 --> AGSVC
    C4 --> AGSVC

    AGSVC --> CSVC
    AGSVC --> APDAO
    AGSVC --> TMDao
    AGSVC --> PSVC

    CSVC <-->|"get/set/del"| VALKEY
    APDAO -->|"JDBC"| AURORA
    TMDao -->|"JDBC"| AURORA
    PSVC -->|"REST"| CX_PLAT
    PSVC -->|"feature flags"| DYNAMO

    %% ── Edges: Kinesis path ───────────────────────────────────────────────
    KINESIS_IN --> KCON
    KCON --> TPROC
    KCON --> UPROC
    TPROC -->|"DB sync"| APDAO
    UPROC -->|"DB sync"| APDAO

    %% ── Colour coding ────────────────────────────────────────────────────
    classDef extStyle  fill:#6c757d,color:#fff,stroke:#495057
    classDef ctlStyle  fill:#0d6efd,color:#fff,stroke:#0a58ca
    classDef authStyle fill:#dc3545,color:#fff,stroke:#b02a37
    classDef svcStyle  fill:#198754,color:#fff,stroke:#146c43
    classDef repoStyle fill:#fd7e14,color:#fff,stroke:#ca6510
    classDef kclStyle  fill:#6f42c1,color:#fff,stroke:#59359a
    classDef dataStyle fill:#0dcaf0,color:#000,stroke:#0aa2c0

    class CX_AUTH,CX_PLAT,KINESIS_IN extStyle
    class C1,C2,C3,C4 ctlStyle
    class AOP,ASVC,JWKS_C,BAD authStyle
    class AGSVC,CSVC,PSVC svcStyle
    class APDAO,TMDao repoStyle
    class KCON,TPROC,UPROC kclStyle
    class AURORA,VALKEY,DYNAMO dataStyle
```

**Description**: The backend API is composed of a thin controller layer (with AOP-injected auth), a thick service layer (`AgentProfilesServiceImpl` orchestrates all business logic), and a repository layer backed by HikariCP-pooled dual datasource (write/read replicas). Two KCL-based Kinesis workers run as background threads within the same process, processing team and user change events asynchronously.

| Component | Responsibility |
|-----------|---------------|
| `AuthAspect` | AOP gateway — all `@Authorized` endpoints pass through here |
| `AuthService` | JWT decode, claims extraction, JWKS cache management |
| `AgentProfilesServiceImpl` | Core business logic orchestrator (35KB) |
| `CacheResponseServiceImpl` | Valkey read/write/invalidate |
| `PlatformApiConsumerServiceImpl` | External HTTP calls to CXone Platform API |
| `AgentProfilesDaoImpl` | Aurora MySQL read/write via JPA |
| `KinesisConsumer` + Processors | Async event-driven DB sync |

---

## 4. C3: Component Diagram — Frontend Application

```mermaid
flowchart TB
    %% ── Shell / Host ─────────────────────────────────────────────────────
    SHELL(["CXone Platform Shell\nModule Federation Host"])

    %% ── Bootstrap Layer ──────────────────────────────────────────────────
    subgraph BOOT["Bootstrap Layer"]
        direction LR
        APP["AppModule\n@NgModule + @angular/elements\nRegisters Web Component"]
        INIT["APP_INITIALIZER Chain\nLocalization · Config · MFE setup"]
        ROUTER["Angular Router\nLazy-loaded routes"]
    end

    %% ── UI Components ────────────────────────────────────────────────────
    subgraph UI["UI Components  [Angular @Component]"]
        direction LR
        COMP_LIST["AgentProfilesComponent\n/agent-profiles\nList · Search · Paginate · Status"]
        COMP_EDIT["CreateEditAgentProfileComponent\n/create-edit-agent-profile\nCreate / Edit form"]
        COMP_DUP["DuplicateDesktopProfileComponent\n/duplicate-desktop-profile"]
        COMP_TEAM["TeamsTabComponent\n/teams-tab\nTeam assignment"]
    end

    %% ── Services ─────────────────────────────────────────────────────────
    subgraph SVCS["Service Layer  [Angular @Injectable]"]
        direction LR
        PSVC["ProfileService\nDomain logic · RxJS observables\ncreate · update · status · teams"]
        ASVC["APIService\nHTTP abstraction\nGET / POST / PUT / DELETE"]
    end

    %% ── Interceptors ─────────────────────────────────────────────────────
    subgraph INTER["HTTP Interceptors  [Angular HTTP_INTERCEPTORS]"]
        direction LR
        REQ["CXOneHttpRequestInterceptor\nInjects Bearer token · tenant headers"]
        RES["CXOneHttpResponseInterceptor\nError normalisation"]
        SPIN["AppSpinnerInterceptor\nGlobal loading state"]
    end

    %% ── Platform Services ────────────────────────────────────────────────
    subgraph PLAT["CXone Platform Services  [@niceltd/cxone-*]"]
        direction LR
        CONFIG["ConfigurationService\nTenant context · env URLs"]
        TENANT["TenantCrossDomainParamsService\nAuth token provider"]
        HTTPU["HttpUtils\nWrapped Angular HttpClient"]
    end

    %% ── Backend ──────────────────────────────────────────────────────────
    BACKEND(["Agent Profiles API\nJava 17 / Spring Boot\n/agent-profiles/v1"])

    %% ── Edges ────────────────────────────────────────────────────────────
    SHELL -->|"Module Federation load"| APP
    APP --> INIT
    APP --> ROUTER
    ROUTER --> COMP_LIST
    ROUTER --> COMP_EDIT
    ROUTER --> COMP_DUP
    ROUTER --> COMP_TEAM

    COMP_LIST --> PSVC
    COMP_EDIT --> PSVC
    COMP_DUP  --> PSVC
    COMP_TEAM --> PSVC

    PSVC --> ASVC
    ASVC --> REQ
    REQ --> RES --> SPIN

    REQ -->|"reads token"| TENANT
    REQ -->|"reads config"| CONFIG
    ASVC --> HTTPU
    HTTPU -->|"HTTPS/REST"| BACKEND

    %% ── Colour coding ────────────────────────────────────────────────────
    classDef shellStyle  fill:#6c757d,color:#fff,stroke:#495057
    classDef bootStyle   fill:#6f42c1,color:#fff,stroke:#59359a
    classDef uiStyle     fill:#0d6efd,color:#fff,stroke:#0a58ca
    classDef svcStyle    fill:#198754,color:#fff,stroke:#146c43
    classDef interStyle  fill:#dc3545,color:#fff,stroke:#b02a37
    classDef platStyle   fill:#fd7e14,color:#fff,stroke:#ca6510
    classDef beStyle     fill:#0dcaf0,color:#000,stroke:#0aa2c0

    class SHELL shellStyle
    class APP,INIT,ROUTER bootStyle
    class COMP_LIST,COMP_EDIT,COMP_DUP,COMP_TEAM uiStyle
    class PSVC,ASVC svcStyle
    class REQ,RES,SPIN interStyle
    class CONFIG,TENANT,HTTPU platStyle
    class BACKEND beStyle
```

**Description**: The Angular MFE is bootstrapped as a Web Component registered via `@angular/elements`. The CXone Shell loads it via Module Federation. The app uses a two-layer service architecture: `ProfileService` for domain logic and `APIService` as an HTTP abstraction. Auth token injection is fully handled by the platform interceptors from `@niceltd/cxone-core-services` — the app never manually handles tokens.

| Component | Responsibility |
|-----------|---------------|
| `AppModule` | Web Component registration, interceptor wiring, MFE bootstrap |
| `ProfileService` | Domain service — all profile API calls with RxJS mapping |
| `APIService` | HTTP wrapper around `HttpUtils` |
| `CXOneHttpRequestInterceptor` | Auto-injects Bearer token + tenant headers |
| `AgentProfilesComponent` | Profile list with pagination/search/sort/status filter |
| `CreateEditAgentProfileComponent` | Create/edit profile form |
| `TeamsTabComponent` | Team assignment UI |

---

## 5. Deployment Diagram

```mermaid
flowchart TB
    %% ── User ─────────────────────────────────────────────────────────────
    USER(["👤 End User Browser\nChrome / Edge"])

    %% ── CDN ──────────────────────────────────────────────────────────────
    subgraph CDN["CDN Layer"]
        CF["CloudFront + S3\nStatic MFE bundle\nHTTPS only"]
    end

    %% ── Network ──────────────────────────────────────────────────────────
    subgraph NET["Network Layer  [CXone Platform VPC]"]
        ALB["Application Load Balancer\nTLS termination\nRoutes /agent-profiles/*"]
    end

    %% ── Compute ──────────────────────────────────────────────────────────
    subgraph COMPUTE["Compute Layer  [ECS Fargate — Private Subnet Az1/Az2]"]
        ECS["Agent Profiles API\nJava 17 / Spring Boot\nApp: 9001  Mgmt: 8080\nImage tagged: github.sha"]
    end

    %% ── Data ─────────────────────────────────────────────────────────────
    subgraph DATALAYER["Data Layer  [Private Subnet — Encrypted]"]
        direction LR
        AW[("Aurora MySQL\nWriter\ndb.r7g.large  :3306")]
        AR[("Aurora MySQL\nReader x2\nAuto-failover")]
        VK[("ElastiCache Valkey\ncache.m7g.xlarge\nMulti-AZ  :6379")]
    end

    %% ── AWS Managed ──────────────────────────────────────────────────────
    subgraph MANAGED["AWS Managed Services"]
        direction LR
        DDB[("DynamoDB\nFeature toggles")]
        SM["Secrets Manager\nDB credentials"]
        SSM["SSM Param Store\nEndpoints + flags"]
        ECR["ECR\n300813158921"]
        KMS["KMS Key\nagentProfiles-kms-key"]
        CW["CloudWatch Alarms\nRDS + Valkey metrics"]
    end

    %% ── Events ───────────────────────────────────────────────────────────
    subgraph EVENTS["Event Streaming  [AWS Kinesis]"]
        direction LR
        KT["dev-TeamData\nstream"]
        KU["dev-userManagement\nstream"]
    end

    %% ── Management ───────────────────────────────────────────────────────
    JUMP["EC2 Jump Server\nManagementCIDR only"]

    %% ── Edges ────────────────────────────────────────────────────────────
    USER -->|"HTTPS"| CF
    USER -->|"HTTPS"| ALB
    CF  -->|"Module Federation"| USER
    ALB -->|"HTTP :9001"| ECS

    ECR  -->|"image pull"| ECS
    ECS  -->|"JDBC write :3306"| AW
    ECS  -->|"JDBC read :3306"| AR
    ECS  -->|"Redis :6379"| VK
    ECS  -->|"AWS SDK"| DDB
    ECS  -->|"startup"| SM
    ECS  -->|"startup"| SSM
    KMS  -->|"encrypts alarms"| CW
    KT   -->|"KCL poll"| ECS
    KU   -->|"KCL poll"| ECS
    AW   -..->|"auto-failover"| AR
    JUMP -->|"MySQL :3306"| AW
    JUMP -->|"Redis :6379"| VK

    %% ── Colour coding ────────────────────────────────────────────────────
    classDef userStyle    fill:#6c757d,color:#fff,stroke:#495057
    classDef cdnStyle     fill:#fd7e14,color:#fff,stroke:#ca6510
    classDef netStyle     fill:#6f42c1,color:#fff,stroke:#59359a
    classDef computeStyle fill:#0d6efd,color:#fff,stroke:#0a58ca
    classDef dataStyle    fill:#0dcaf0,color:#000,stroke:#0aa2c0
    classDef mgmtStyle    fill:#198754,color:#fff,stroke:#146c43
    classDef eventStyle   fill:#dc3545,color:#fff,stroke:#b02a37
    classDef jumpStyle    fill:#ffc107,color:#000,stroke:#d39e00

    class USER userStyle
    class CF cdnStyle
    class ALB netStyle
    class ECS computeStyle
    class AW,AR,VK,DDB dataStyle
    class SM,SSM,ECR,KMS,CW mgmtStyle
    class KT,KU eventStyle
    class JUMP jumpStyle
```

**Deployment Inventory:**

| Layer | Resource | Spec (staging/prod) | Notes |
|-------|----------|--------------------:|-------|
| CDN | CloudFront + S3 | — | MFE static bundle; HTTPS only |
| Network | Shared ALB | Platform-managed | TLS termination; routes `/agent-profiles/*` |
| Compute | ECS Fargate | — | Stateless; ports 9001 / 8080 |
| Database | Aurora MySQL (1 writer + 2 readers) | db.r7g.large | Multi-AZ; 30-day backup; KMS encrypted |
| Cache | ElastiCache Valkey | cache.m7g.xlarge | Multi-AZ; 7-day snapshot |
| Feature store | DynamoDB | On-demand | Feature toggles |
| Secrets | Secrets Manager | — | DB credentials; KMS encrypted |
| Config | SSM Parameter Store | — | Endpoints, FIPS flag, TLS flag |
| Registry | ECR | — | Images tagged by `github.sha` |
| Events | Kinesis (×2) | — | TeamData + userManagement streams |

---

## 6. Security Architecture Diagram

```mermaid
flowchart LR
    %% ── Users ────────────────────────────────────────────────────────────
    subgraph USERS["Users"]
        direction TB
        ADMIN(["Admin\nBrowser"])
        CXA(["CXone Agent\nDesktop"])
    end

    %% ── Frontend / CDN ───────────────────────────────────────────────────
    subgraph FRONT["Frontend / CDN"]
        direction TB
        CF["CloudFront + S3\nHTTPS  TLS 1.2+"]
        SHELL["CXone Shell\nModule Federation host"]
    end

    %% ── Network Perimeter ────────────────────────────────────────────────
    subgraph PERIM["Network Perimeter"]
        direction TB
        WAF["WAF / DDoS\nPlatform-managed"]
        ALB["ALB\nTLS Termination"]
    end

    %% ── Resilience ───────────────────────────────────────────────────────
    subgraph RESIL["Resilience Layer"]
        direction TB
        RL["Rate Limiter\ncxalimiter  75 req/s"]
        CB["Circuit Breaker\nuserhub  50% threshold"]
    end

    %% ── Auth ─────────────────────────────────────────────────────────────
    subgraph AUTHZ["Auth Layer  [AOP]"]
        direction TB
        AOP["@Authorized AOP\nJWT validate + RBAC"]
        JWKS["JWKS Cache\nL1 Caffeine  L2 Persistent"]
        BADKID["Bad Kid Cache\n5 min TTL"]
        PERMS["Platform Permissions\n/user/permissions"]
        CXAUTH(["CXone Auth\nJWKS Endpoint"])
    end

    %% ── Application ──────────────────────────────────────────────────────
    API["Agent Profiles API\n@Authorized  IAM Role"]

    %% ── Data Layer ───────────────────────────────────────────────────────
    subgraph DATALAYER["Data Layer  [Private Subnet — Encrypted]"]
        direction TB
        AURORA[("Aurora MySQL\nKMS at rest  TLS conditional")]
        VALKEY[("Valkey Cache\nAES at rest  TLS conditional")]
        DDB[("DynamoDB\nAWS-managed encryption")]
    end

    %% ── Secrets ──────────────────────────────────────────────────────────
    subgraph SECRETS["Secrets & Config"]
        direction TB
        SM["Secrets Manager\nDB credentials  KMS"]
        SSM["SSM Param Store\nFIPS flag  endpoints"]
        KMS["KMS Key\nSNS + CloudWatch encryption"]
    end

    %% ── CI Security Scanning ─────────────────────────────────────────────
    subgraph SCAN["Security Scanning  [CI/CD]"]
        direction TB
        VER["Veracode SAST\nevery push/PR"]
        BD["BlackDuck SCA\ndependency scan"]
        SQ["SonarQube\nquality gate"]
    end

    %% ── Edges ────────────────────────────────────────────────────────────
    ADMIN -->|"HTTPS"| CF
    ADMIN -->|"HTTPS"| SHELL
    CXA   -->|"HTTPS"| WAF
    SHELL -->|"API via proxy"| WAF
    WAF   --> ALB --> RL --> AOP
    AOP   --> JWKS
    JWKS  -->|"miss"| CXAUTH
    JWKS  --> BADKID
    AOP   --> CB --> PERMS
    AOP   --> API
    API   -->|"read/write"| AURORA
    API   -->|"cache"| VALKEY
    API   -->|"feature flags"| DDB
    API   -->|"startup"| SM
    API   -->|"startup"| SSM
    KMS   -->|"encrypts"| AURORA
    KMS   -->|"encrypts"| VALKEY
    KMS   -->|"encrypts"| SM

    %% ── Colour coding ────────────────────────────────────────────────────
    classDef userStyle   fill:#6c757d,color:#fff,stroke:#495057
    classDef frontStyle  fill:#fd7e14,color:#fff,stroke:#ca6510
    classDef perimStyle  fill:#6f42c1,color:#fff,stroke:#59359a
    classDef resilStyle  fill:#0d6efd,color:#fff,stroke:#0a58ca
    classDef authStyle   fill:#dc3545,color:#fff,stroke:#b02a37
    classDef apiStyle    fill:#198754,color:#fff,stroke:#146c43
    classDef dataStyle   fill:#0dcaf0,color:#000,stroke:#0aa2c0
    classDef secretStyle fill:#ffc107,color:#000,stroke:#d39e00
    classDef scanStyle   fill:#20c997,color:#000,stroke:#1aa179

    class ADMIN,CXA userStyle
    class CF,SHELL frontStyle
    class WAF,ALB perimStyle
    class RL,CB resilStyle
    class AOP,JWKS,BADKID,PERMS,CXAUTH authStyle
    class API apiStyle
    class AURORA,VALKEY,DDB dataStyle
    class SM,SSM,KMS secretStyle
    class VER,BD,SQ scanStyle
```

**Security Controls Summary:**

| Layer | Control |
|-------|---------|
| Transport | TLS 1.2+ on all external traffic; conditional TLS on internal DB/cache (env-controlled) |
| Authentication | JWT Bearer token, validated via CXone JWKS (Nimbus JOSE) |
| Authorization | AOP `@Authorized` — per-endpoint RBAC permissions (`VIEW`/`CREATE`/`EDIT`) |
| Rate Limiting | Resilience4j 75 req/s, instant reject |
| Circuit Breaker | Protects Platform API calls; 50% failure threshold |
| JWKS Caching | 2-tier (L1 Caffeine + L2 persistent) reduces JWKS endpoint exposure |
| Encryption at Rest | Aurora: KMS; Valkey: AES; Secrets Manager: KMS |
| Secrets | AWS Secrets Manager (DB creds); SSM (config); no plaintext in app |
| Scanning | Veracode SAST + BlackDuck SCA (backend, every PR); SonarQube (both) |
| IAM | OIDC-based GitHub Actions assume role; `ServiceAccess-agent-profiles-service-deploy-role` |
| FIPS | Enabled on test and perf environments |

---

## 7. CI/CD Pipeline Diagram

```mermaid
flowchart LR
    %% ── Trigger ──────────────────────────────────────────────────────────
    GIT(["git push / PR\nmaster or 26.x branch"])

    %% ── Backend CI ───────────────────────────────────────────────────────
    subgraph BCI["GitHub Actions — Backend CI  [BuildPushECR.yaml]"]
        direction TB
        CO["Checkout + JDK 17 setup"]
        subgraph TESTS["Integration Tests  [matrix: dev | test | staging]"]
            IT["./gradlew test\n-PincludeIntegrationTests\n-PactiveProfile={env}"]
        end
        SQ["SonarQube scan"]
        SQG{Quality Gate}
        BD["BlackDuck SCA"]
        VER["Veracode SAST"]
        SECG{Security Gate}
        JIB["Gradle + Jib build\nJAR to Docker image"]
        ECR["Push to ECR\nTagged: github.sha"]
    end

    %% ── Infra Deploy ─────────────────────────────────────────────────────
    subgraph INFRA["GitHub Actions — Infra Deploy  [DeployInfra.yaml]"]
        direction TB
        ITRIG(["workflow_dispatch\nSelect: env + stack"])
        IAPPR["Manual Approval Gate\nGitHub Env: approval_needed"]
        IAUTH["OIDC AssumeRole"]
        CFN["CloudFormation deploy\nSelected stack"]
    end

    %% ── Image Promote ────────────────────────────────────────────────────
    subgraph PROMO["GitHub Actions — Image Promotion"]
        direction TB
        PTRIG(["workflow_dispatch\nSelect: image tag + target env"])
        PAPPR["Approval Gate\nProd requires separate workflow"]
        PECR["ECR: copy image\nsource to target account"]
    end

    %% ── Frontend CI ──────────────────────────────────────────────────────
    subgraph FCI["Jenkins — Frontend CI  [Jenkinsfile]"]
        direction TB
        FIN["npm install"]
        FLINT["ESLint + TSLint"]
        FTEST["Unit tests\nKarma / Jasmine  LCOV coverage"]
        FSQ["SonarQube scan"]
        FBUILD["ng build\nwebpack.prod.config.js"]
        FDEP["Deploy to CDN / S3"]
    end

    %% ── Blocked terminals ────────────────────────────────────────────────
    SQBLOCK(["Blocked\nQuality gate"])
    SECBLOCK(["Blocked\nSecurity findings"])

    %% ── Edges: Backend CI ────────────────────────────────────────────────
    GIT --> CO --> IT --> SQ
    SQ  --> SQG
    SQG -->|"Fail"| SQBLOCK
    SQG -->|"Pass"| BD --> VER --> SECG
    SECG -->|"Fail"| SECBLOCK
    SECG -->|"Pass"| JIB --> ECR

    %% ── Edges: Infra ─────────────────────────────────────────────────────
    ITRIG --> IAPPR --> IAUTH --> CFN

    %% ── Edges: Promote ───────────────────────────────────────────────────
    ECR --> PTRIG --> PAPPR --> PECR

    %% ── Edges: Frontend ──────────────────────────────────────────────────
    GIT --> FIN --> FLINT --> FTEST --> FSQ --> FBUILD --> FDEP

    %% ── Colour coding ────────────────────────────────────────────────────
    classDef trigStyle   fill:#6c757d,color:#fff,stroke:#495057
    classDef ciStyle     fill:#0d6efd,color:#fff,stroke:#0a58ca
    classDef secStyle    fill:#dc3545,color:#fff,stroke:#b02a37
    classDef gateStyle   fill:#fd7e14,color:#000,stroke:#ca6510
    classDef buildStyle  fill:#198754,color:#fff,stroke:#146c43
    classDef infraStyle  fill:#6f42c1,color:#fff,stroke:#59359a
    classDef promoStyle  fill:#0dcaf0,color:#000,stroke:#0aa2c0
    classDef feStyle     fill:#ffc107,color:#000,stroke:#d39e00
    classDef blockStyle  fill:#dc3545,color:#fff,stroke:#b02a37

    class GIT trigStyle
    class CO,IT ciStyle
    class SQ,BD,VER secStyle
    class SQG,SECG gateStyle
    class JIB,ECR buildStyle
    class ITRIG,IAPPR,IAUTH,CFN infraStyle
    class PTRIG,PAPPR,PECR promoStyle
    class FIN,FLINT,FTEST,FSQ,FBUILD,FDEP feStyle
    class SQBLOCK,SECBLOCK blockStyle
```

**Pipeline Stage Descriptions:**

| Stage | Tool | Gate? | Notes |
|-------|------|-------|-------|
| Integration Tests (3 envs) | Gradle/JUnit | No | Matrix: dev, test, staging profiles |
| SonarQube Scan | SonarQube | ✅ Quality Gate | Blocks on quality gate failure |
| BlackDuck SCA | BlackDuck | No hard block found | Dependency vulnerability scan |
| Veracode SAST | Veracode | ✅ Security Gate | Static analysis on JAR artifact |
| Jib Build + ECR Push | Gradle Jib | No | Image tagged by `github.sha` |
| Infra Deploy | CloudFormation | ✅ Manual Approval | `approval_needed` GitHub environment |
| Image Promote to Prod | ECR | ✅ Manual Approval | Separate `PromoteImageToProd.yaml` |
| Frontend Lint + Test | ESLint/Karma | No explicit gate | Jenkins-managed |
| Frontend SonarQube | SonarQube | No explicit gate | `sonar-project.properties` |

---

## 8. Data Architecture Diagram

```mermaid
flowchart LR
    subgraph INGRESS["Data Ingress"]
        direction TB
        REST["REST API\nHTTPS/JSON"]
        KIN["Kinesis Streams\nTeamData + userManagement"]
    end

    subgraph APP["Application Layer"]
        direction TB
        SVC["AgentProfilesServiceImpl\nBusiness logic + validation"]
        CACHE["CacheResponseServiceImpl\nCache-aside pattern"]
    end

    subgraph STORES["Data Stores"]
        direction TB
        AW[("Aurora MySQL\nWriter  :3306")]
        AR[("Aurora MySQL\nReader x2  :3306")]
        VK[("Valkey Cache\nPort 6379  Multi-AZ")]
        DDB[("DynamoDB\nFeature toggles")]
    end

    subgraph CONFIG["Config & Secrets  [injected at startup]"]
        direction TB
        SM["Secrets Manager\nDB credentials"]
        SSM["SSM Param Store\nEndpoints + flags"]
    end

    REST  --> SVC
    KIN   --> SVC
    SVC   --> CACHE
    CACHE -->|"cache hit"| REST
    CACHE <-->|"get / set / del"| VK
    CACHE -->|"miss — write"| AW
    CACHE -->|"miss — read"| AR
    SVC   -->|"writes"| AW
    SVC   -->|"reads"| AR
    SVC   -->|"feature flags"| DDB
    SM    -->|"startup"| SVC
    SSM   -->|"startup"| SVC

    classDef inStyle  fill:#6c757d,color:#fff,stroke:#495057
    classDef appStyle fill:#0d6efd,color:#fff,stroke:#0a58ca
    classDef dbStyle  fill:#0dcaf0,color:#000,stroke:#0aa2c0
    classDef cfgStyle fill:#ffc107,color:#000,stroke:#d39e00

    class REST,KIN inStyle
    class SVC,CACHE appStyle
    class AW,AR,VK,DDB dbStyle
    class SM,SSM cfgStyle
```

**Cache Strategy:**
- **Pattern**: Cache-aside — read from Valkey first; on miss, query Aurora and populate cache
- **Write-through invalidation**: All mutations (create / update / delete / status change) purge affected BU-scoped cache keys
- **JWKS L2 cache**: Also stored in Valkey (15–30d TTL), separate from API response cache

**Data Store Inventory:**

| Store | Type | Purpose | Encryption | HA |
|-------|------|---------|-----------|-----|
| Aurora MySQL (writer) | AWS RDS Aurora | All write operations | KMS at rest, TLS conditional | Multi-AZ auto-failover |
| Aurora MySQL (reader ×2) | AWS RDS Aurora | Read / search queries | KMS at rest, TLS conditional | Auto-promoted on writer failure |
| ElastiCache Valkey | Redis-compatible | API response cache + JWKS L2 cache | AES at rest, TLS conditional | Multi-AZ replica |
| DynamoDB | AWS DynamoDB | Feature toggles | AWS-managed | Fully managed |
| Secrets Manager | AWS Secrets | DB credentials | KMS | Fully managed |
| SSM Parameter Store | AWS SSM | Runtime endpoints + FIPS/TLS flags | AWS-managed | Fully managed |

---

---

## 9. Database Design (Entity Relationship Diagram)

Derived from JPA entities in `src/main/java/com/nice/cxone/agentprofiles/jpa/entities/`.
Schema is per-tenant: `agentprofiles_schema_tenant_{tenantId}`.

```mermaid
erDiagram
    agent_profiles {
        BIGINT      id              PK
        BIGINT      bus_id          "BU / tenant identifier"
        VARCHAR     name            "Profile display name (unique per BU)"
        VARCHAR     status          "ACTIVE | INACTIVE"
        TIMESTAMP   created_at
        TIMESTAMP   updated_at
    }

    agent_profile_configuration {
        BIGINT      profile_id      PK,FK
        VARCHAR     config_key      PK
        TEXT        config_value
    }

    agent_profile_teams_map {
        BIGINT      profile_id      PK,FK
        BIGINT      team_id         PK
    }

    assignable_configurations {
        BIGINT      id              PK
        VARCHAR     name
        VARCHAR     type
        TEXT        default_value
    }

    assignable_configurations_elements {
        BIGINT      id              PK
        BIGINT      config_id       FK
        VARCHAR     element_id
        TEXT        value
    }

    assignable_configurations_dependency_elements {
        BIGINT      element_id      PK,FK
        BIGINT      dependency_id   PK
    }

    tenant_schema {
        BIGINT      id              PK
        VARCHAR     tenant_id       "CXone tenantId"
        VARCHAR     schema_name     "agentprofiles_schema_tenant_{tenantId}"
    }

    agent_profiles ||--o{ agent_profile_configuration : "has"
    agent_profiles ||--o{ agent_profile_teams_map     : "assigned to"
    assignable_configurations ||--o{ assignable_configurations_elements : "has elements"
    assignable_configurations_elements ||--o{ assignable_configurations_dependency_elements : "has dependencies"
```

**Table Descriptions:**

| Table | PK | Description |
|-------|-----|-------------|
| `agent_profiles` | `id` | Core profile entity. One profile per named configuration set. `bus_id` provides BU-level tenant isolation. `name` unique per BU. |
| `agent_profile_configuration` | `(profile_id, config_key)` | Key-value settings attached to a profile. Composite PK. Deleted and re-inserted on profile update. |
| `agent_profile_teams_map` | `(profile_id, team_id)` | Junction table — many-to-many between profiles and CXone teams. `team_id` references external CXone team IDs (not a FK to a local table). |
| `assignable_configurations` | `id` | Catalog of available configuration types that can be included in a profile (e.g. desktop widgets, tool availability). |
| `assignable_configurations_elements` | `id` | Specific selectable elements within an assignable configuration (e.g. individual tools/options). |
| `assignable_configurations_dependency_elements` | `(element_id, dependency_id)` | Dependency graph between configuration elements — an element may require another element to be selected. |
| `tenant_schema` | `id` | Registry mapping `tenantId` → schema name for dynamic schema routing. |

**Multi-tenancy Note:**  
The `tenant_schema` table is queried at request time to determine which per-tenant MySQL schema to use (`agentprofiles_schema_tenant_{tenantId}`). This is implemented via Spring's `AbstractRoutingDataSource` pattern — the tenant schema name is resolved from the JWT `tenantId` claim and set as the active datasource route.

**Audit Fields:**  
All mutable tables extend `MappedCreatedUpdatedEntity` which provides `created_at` and `updated_at` timestamps managed by JPA lifecycle callbacks (`@PrePersist`, `@PreUpdate`).

---

## Architecture Decisions & Notes

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| Embedded KCL consumers (in API process) | Simplifies deployment — single ECS task | Kinesis processing tied to API scaling; high API load could starve event processing |
| Schema-per-tenant multi-tenancy | Strong data isolation at DB level | Schema sprawl with many tenants; Flyway migrations must run per-tenant |
| Dual datasource (write/read replicas) | Separates write and read workloads; read scalability | HikariCP pool count effectively doubled (2 × 100 connections) |
| Cache-aside with Valkey | Reduces Aurora read load and latency for repeated queries | Cache invalidation complexity on mutations |
| Custom AOP `@Authorized` + Platform permissions API | Fine-grained RBAC without Spring Security complexity | Synchronous external call on every request — latency risk (mitigated by circuit breaker; caching recommended) |
| Jib for container builds | No Dockerfile maintenance; fast incremental builds | Less control over image layer structure |
| Module Federation MFE | Independent deployability of UI from shell | Increased complexity in build pipeline; version compatibility with shell required |
