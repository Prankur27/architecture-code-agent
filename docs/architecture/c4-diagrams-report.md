# C4 Architecture Diagrams — Agent Profiles (NICE CXone)
Generated from Architecture Analysis Report — March 22, 2026

---

## Diagram Index
1. [L1: System Context Diagram](#1-l1-system-context-diagram)
2. [L2: Container Diagram](#2-l2-container-diagram)
3. [L3: Component Diagram — Backend API](#3-l3-component-diagram--backend-api)
4. [L3: Component Diagram — Frontend Application](#4-l3-component-diagram--frontend-application)
5. [Deployment Diagram](#5-deployment-diagram)
6. [Security Architecture Diagram](#6-security-architecture-diagram)
7. [CI/CD Pipeline Diagram](#7-cicd-pipeline-diagram)
8. [Data Architecture Diagram](#8-data-architecture-diagram)
9. [Database Design (ERD)](#9-database-design-entity-relationship-diagram)

---

## 1. L1: System Context Diagram

```mermaid
C4Context
  title System Context Diagram — Agent Profiles (NICE CXone)

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

## 2. L2: Container Diagram

```mermaid
C4Container
  title Container Diagram — Agent Profiles System

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

## 3. L3: Component Diagram — Backend API

```mermaid
C4Component
  title Component Diagram — Agent Profiles API (Backend)

  Container_Boundary(backendAPI, "Agent Profiles API — Spring Boot Application") {

    Component(appEntry, "App.java", "Spring Boot @SpringBootApplication", "Application entry point. Initialises Spring context.")
    Component(appConfig, "AppConfiguration.java", "Spring @Configuration", "Wires all beans: HTTP clients, cache, datasource routing, Resilience4j, OTLP, Kinesis.")

    Component(agentProfilesController, "AgentProfilesController", "Spring @RestController — /profiles", "Handles CRUD endpoints for agent profiles and team assignments. Applies @Authorized and @RateLimiter.")
    Component(assignableConfigController, "AssignableConfigurationsController", "Spring @RestController — /assignable-configurations", "Exposes the catalog of assignable configuration items.")
    Component(cxaControllerV1, "GetAgentProfileForCXAController", "Spring @RestController — /cxa/profiles", "CXone Agent v1 profile fetch endpoint.")
    Component(cxaControllerV2, "GetAgentProfileCXAControllerV2", "Spring @RestController — /cxa/v2/profiles", "CXone Agent v2 profile fetch endpoint.")

    Component(authAspect, "AuthAspect", "Spring AOP @Aspect @Around @Authorized", "Intercepts all @Authorized controller methods. Extracts JWT, calls AuthService, checks required permissions. Throws 401/403 on failure.")
    Component(authService, "AuthService", "Spring @Service", "Parses JWT (Nimbus JOSE), extracts userId/buId/tenantId/role from claims. Calls PlatformApiConsumerService to resolve user permissions.")

    Component(agentProfilesService, "AgentProfilesServiceImpl", "Spring @Service — IAgentProfilesService", "Core business logic: create/update/delete profiles, manage team assignments, status updates, search/filter. Orchestrates DAO + Cache + Platform calls.")
    Component(cacheService, "CacheResponseServiceImpl", "Spring @Service — ICacheResponseService", "Reads and writes API responses to/from Valkey. Key-based invalidation on mutations.")
    Component(platformService, "PlatformApiConsumerServiceImpl", "Spring @Service — IPlatformApiConsumerService", "HTTP client to CXone Platform API. Fetches user permissions (/user/permissions), agent details (/agents/), team details (/teams). Circuit breaker protected.")

    Component(agentProfilesDao, "AgentProfilesDaoImpl", "Spring @Repository / JPA", "DB access for AgentProfiles entity. Supports read/write replica routing. Flyway-managed schema.")
    Component(teamsMapDao, "AgentProfileTeamsMapDaoImpl", "Spring @Repository / JPA", "DB access for AgentProfileTeamsMap junction table.")

    Component(kinesisConsumer, "KinesisConsumer", "KCL Worker Bootstrap", "Starts two KCL workers for TeamData and userManagement streams.")
    Component(teamDataProcessor, "TeamDataRecordProcessor", "KCL IRecordProcessor", "Processes team create/rename/delete events. Updates AgentProfileTeamsMap in DB.")
    Component(userMgmtProcessor, "UserManagementRecordProcessor", "KCL IRecordProcessor", "Processes agent hire/terminate/role-change events.")

    Component(jpaEntities, "JPA Entities", "Jakarta @Entity classes", "AgentProfiles, AgentProfileConfiguration, AgentProfileTeamsMap, AssignableConfigurations, TenantSchema, MappedCreatedUpdatedEntity")
    Component(dtos, "DTOs", "POJO / @Valid annotated", "AgentProfileRequest, AgentProfileStatusChangeRequest, AgentProfileTeamsMapRequestDTO, GetAgentProfilePayloadDTO, SuccessResponseDTO")
    Component(validationUtils, "AgentProfileValidationUtils", "Utility class", "Input validation for request params and profile request bodies.")
    Component(logContext, "LogAttributesThreadLocal", "MDC ThreadLocal", "Injects correlation IDs and tenant context into log records.")
  }

  ContainerDb(auroraDB, "Aurora MySQL", "", "")
  ContainerDb(valkeyCache, "Valkey Cache", "", "")
  ContainerDb(dynamoDB, "DynamoDB", "", "")
  System_Ext(cxonePlatformAPI, "CXone Platform API", "")
  System_Ext(cxoneAuth, "CXone Auth / JWKS", "")
  System_Ext(kinesisStreams, "AWS Kinesis", "")

  Rel(agentProfilesController, authAspect, "Intercepted by (AOP)")
  Rel(authAspect, authService, "Delegates JWT validation + permission check")
  Rel(authService, valkeyCache, "L1/L2 JWKS cache lookup")
  Rel(authService, cxoneAuth, "Fetch JWKS on cache miss", "HTTPS")
  Rel(authService, platformService, "Fetch user permissions")

  Rel(agentProfilesController, agentProfilesService, "Delegates business operations")
  Rel(agentProfilesController, validationUtils, "Validates request params")
  Rel(agentProfilesService, cacheService, "Read/write response cache")
  Rel(agentProfilesService, agentProfilesDao, "DB reads and writes")
  Rel(agentProfilesService, teamsMapDao, "Team-profile mapping reads/writes")
  Rel(agentProfilesService, platformService, "Fetch agent/team data")
  Rel(cacheService, valkeyCache, "Redis get/set/del", "Redis protocol")
  Rel(agentProfilesDao, auroraDB, "JDBC queries", "MySQL/TLS")
  Rel(teamsMapDao, auroraDB, "JDBC queries", "MySQL/TLS")
  Rel(platformService, cxonePlatformAPI, "REST calls", "HTTPS")
  Rel(platformService, dynamoDB, "Feature toggle reads", "AWS SDK")

  Rel(kinesisConsumer, kinesisStreams, "Polls shards", "KCL")
  Rel(kinesisConsumer, teamDataProcessor, "Dispatches team records")
  Rel(kinesisConsumer, userMgmtProcessor, "Dispatches user records")
  Rel(teamDataProcessor, agentProfilesDao, "Sync team changes to DB")
  Rel(userMgmtProcessor, agentProfilesDao, "Sync user changes to DB")
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

## 4. L3: Component Diagram — Frontend Application

```mermaid
C4Component
  title Component Diagram — Agent Profiles MFE (Angular)

  Container_Boundary(angularMFE, "Agent Profiles MFE — Angular Web Component") {

    Component(appModule, "AppModule / bootstrap", "Angular @NgModule + @angular/elements", "Root module. Registers agent-profile-app as a Web Component. Registers HTTP interceptors. Configures app initializer chain.")

    Component(appInitializer, "App Initializer Chain", "APP_INITIALIZER factory", "Runs on app bootstrap: LocalizationInitializer (i18next), ConfigurationService, WebAppInitializerService (MFE setup, hide nav flags, mfeActivateGuard).")

    Component(appRoutes, "AppRoutes / AppRoutingModule", "Angular Router", "Defines lazy-loaded routes: agent-profiles list, create/edit profile, duplicate profile, teams tab.")

    Component(agentProfilesComponent, "AgentProfilesComponent", "Angular @Component — /agent-profiles", "Main profile list view. Handles pagination, search, status filter, sort, activate/deactivate actions. Consumes ProfileService.")
    Component(createEditComponent, "CreateEditAgentProfileComponent", "Angular @Component — /create-edit-agent-profile", "Form for creating and editing an agent profile. Binds to assignable configuration catalog.")
    Component(duplicateComponent, "DuplicateDesktopProfileComponent", "Angular @Component — /duplicate-desktop-profile", "Workflow for duplicating an existing profile.")
    Component(teamsTabComponent, "TeamsTabComponent", "Angular @Component — /teams-tab", "Team assignment tab. Lists and modifies teams assigned to a profile.")

    Component(profileService, "ProfileService", "Angular @Injectable service", "Business logic + state for profile operations. Wraps APIService with domain methods: getProfilesData, getDesktopProfileById, createDesktopProfile, updateDesktopProfile, updateProfileStatus, updateDesktopProfileAssignedTeams.")
    Component(apiService, "APIService", "Angular @Injectable service", "HTTP abstraction. Wraps HttpUtils with typed GET/POST/PUT/DELETE to /agent-profiles base URL.")
    Component(apiConstants, "API_CONSTANTS", "TypeScript constants module", "Centralises all backend API URL builder functions.")

    Component(httpInterceptors, "HTTP Interceptors", "Angular HTTP_INTERCEPTORS", "CXOneHttpRequestInterceptor (injects auth token + tenant headers), CXOneHttpResponseInterceptor (error handling), AppSpinnerInterceptor (loading state).")

    Component(platformServices, "CXone Platform Services", "@niceltd/cxone-core-services + cxone-client-platform-services", "ConfigurationService, TenantCrossDomainParamsService, HttpUtils. Provides auth tokens, tenant context, localized HTTP client.")

    Component(sharedModels, "Shared Models", "TypeScript interfaces / src/models", "Profile, AgentProfilesResponse, AgentProfileApi, and other domain types.")

    Component(i18n, "i18n / Localization", "angular-i18next + @niceltd/sol/translation", "Internationalisation support. Locale initialised at app startup via LocalizationInitializer.")
  }

  ContainerDb(backendAPI, "Agent Profiles API", "Java 17 / Spring Boot", "")
  System_Ext(cxoneShell, "CXone Platform Shell", "MFE host — provides auth, tenant context, navigation")

  Rel(cxoneShell, appModule, "Loads Web Component via Module Federation", "HTTPS")
  Rel(appModule, appInitializer, "Runs on bootstrap")
  Rel(appModule, appRoutes, "Activates routing")
  Rel(appRoutes, agentProfilesComponent, "Lazy loads on /agent-profiles")
  Rel(appRoutes, createEditComponent, "Lazy loads on /create-edit-agent-profile")
  Rel(appRoutes, duplicateComponent, "Lazy loads on /duplicate-desktop-profile")
  Rel(appRoutes, teamsTabComponent, "Lazy loads on /teams-tab")
  Rel(agentProfilesComponent, profileService, "Calls for data")
  Rel(createEditComponent, profileService, "Calls for create/update/config")
  Rel(teamsTabComponent, profileService, "Calls for team assignment")
  Rel(profileService, apiService, "Delegates HTTP calls")
  Rel(apiService, httpInterceptors, "Passes through interceptor chain")
  Rel(httpInterceptors, platformServices, "Reads auth token + tenant params")
  Rel(httpInterceptors, backendAPI, "Authenticated REST calls", "HTTPS/REST")
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
    subgraph BROWSER["👤 End User Browser (Chrome / Edge)"]
        MFE_BUNDLE["Agent Profiles MFE\nAngular Web Component\n(loaded from CDN)"]
    end

    subgraph CDN["☁️ CDN Layer"]
        CF["CloudFront + S3\nStatic MFE bundle\nHTTPS only"]
    end

    subgraph AWS["AWS Cloud — us-west-2"]
        subgraph VPC["CoreNetwork-VPC (Shared Platform VPC)"]
            ALB["Application Load Balancer\nTLS termination\nRoutes /agent-profiles/*"]

            subgraph PRIVATE["Private Subnets — Az1 + Az2"]
                ECS["ECS Fargate\nAgent Profiles API\nJava 17 / Spring Boot\nPort: 9001 (app) 8080 (mgmt)\nImage: ECR — github.sha tag"]

                subgraph DATA["Data Layer"]
                    AURORA_W["Aurora MySQL Writer\ndb.r7g.large\nPort 3306 / TLS"]
                    AURORA_R1["Aurora MySQL Reader 1\ndb.r7g.large\nAuto-failover"]
                    AURORA_R2["Aurora MySQL Reader 2\ndb.r7g.large"]
                end

                subgraph CACHE["Cache Layer"]
                    VALKEY["ElastiCache Valkey\ncache.m7g.xlarge\nMulti-AZ / Port 6379"]
                end

                JUMP["EC2 Jump Server\nManagementCIDR only"]
            end
        end

        subgraph AWS_MANAGED["AWS Managed Services"]
            DYNAMO["DynamoDB\nFeature toggle table"]
            SM["Secrets Manager\n{env}-AgentProfilesSecrets"]
            SSM["SSM Parameter Store\nEndpoints + flags"]
            ECR["ECR\n300813158921"]
            KMS["KMS Key\nagentProfiles-kms-key"]
            CW["CloudWatch Alarms\nRDS + Valkey metrics"]
        end

        subgraph KINESIS["Event Streaming"]
            KTEAM["Kinesis: dev-TeamData"]
            KUSER["Kinesis: dev-userManagement"]
        end
    end

    BROWSER -- HTTPS --> CF
    BROWSER -- HTTPS --> ALB
    CF -- "Module Federation" --> MFE_BUNDLE
    ALB -- "HTTP :9001" --> ECS

    ECS -- "JDBC write :3306" --> AURORA_W
    ECS -- "JDBC read :3306" --> AURORA_R1
    ECS -- "Redis :6379" --> VALKEY
    ECS -- "AWS SDK" --> DYNAMO
    ECS -- "startup" --> SM
    ECS -- "startup" --> SSM
    ECR -- "image pull" --> ECS
    KMS -- "encrypts" --> CW

    KTEAM -- "KCL poll" --> ECS
    KUSER -- "KCL poll" --> ECS

    AURORA_W -. "auto-failover" .-> AURORA_R1
    AURORA_R1 -. "replica" .-> AURORA_R2
    VALKEY -. "Multi-AZ replica" .-> VALKEY

    JUMP -- "MySQL :3306" --> AURORA_W
    JUMP -- "Redis :6379" --> VALKEY
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
flowchart TD
    subgraph "Internet / Browser"
        User["👤 Admin User / CXone Agent"]
    end

    subgraph "CXone Platform Layer"
        Shell["CXone Shell\n[TLS 1.2+]"]
        Auth["CXone Auth / JWKS\n[JWT Issuer]"]
    end

    subgraph "CDN Layer"
        CF["CloudFront + S3\n[TLS 1.2+]\n[HTTPS only]"]
    end

    subgraph "Network Perimeter"
        WAF["WAF / DDoS Protection\n[Platform-managed — upstream of ALB]"]
        ALB["Application Load Balancer\n[TLS Termination]\n[HTTPS → HTTP internal]"]
    end

    subgraph "Application Layer  [Private Subnet]"
        API["Agent Profiles API\n[IAM Role: ServiceAccess-agent-profiles-service-deploy-role]\n[JWT validated on every request]\n[@Authorized AOP]"]
        CB["Circuit Breaker\n[resilience4j — userhub]\n[50% failure threshold]"]
        RL["Rate Limiter\n[resilience4j — cxalimiter]\n[75 req/s]"]
    end

    subgraph "Auth Flow"
        JWKS_CACHE["JWKS Cache\n[L1: Caffeine 6-12h]\n[L2: Persistent 15-30d]"]
        BAD_KID["Bad Kid Cache\n[Caffeine 5min / 1000 entries]"]
        PERMS["Platform Permissions API\n[/user/permissions]"]
    end

    subgraph "Data Layer  [Private Subnet — Encrypted]"
        Aurora["Aurora MySQL\n[KMS encrypted at rest]\n[TLS 1.2 in transit — conditional]\n[Not publicly accessible]\n[30d backup retention]"]
        Valkey["Valkey Cache\n[At-rest encrypted]\n[TLS in transit — conditional]\n[Multi-AZ]"]
        DDB["DynamoDB\n[AWS-managed encryption]"]
    end

    subgraph "Secrets & Config  [AWS Managed]"
        SM["Secrets Manager\n[DB credentials]\n[KMS encrypted]"]
        SSM["SSM Parameter Store\n[FIPS flag, TLS flag, endpoints]"]
        KMS["KMS Key\n[alias: agentProfiles-kms-key]\n[Key rotation on FIPS envs]\n[SNS + CloudWatch encryption]"]
    end

    subgraph "Security Scanning  [CI/CD Pipeline]"
        Veracode["Veracode SAST\n[every push/PR]\n[artifact: ms-agent-profiles-apis-1.0-SNAPSHOT]"]
        BlackDuck["BlackDuck SCA\n[dependency scan on every build]"]
        Sonar["SonarQube\n[backend + frontend]\n[quality gate on build]"]
    end

    User -->|HTTPS| Shell
    User -->|HTTPS| CF
    Shell -->|Loads MFE via Module Federation| CF
    Shell -->|API calls via proxy| WAF
    WAF --> ALB
    ALB -->|HTTP :9001| RL
    RL --> API
    API -->|AOP intercept| JWKS_CACHE
    JWKS_CACHE -->|cache miss| Auth
    JWKS_CACHE --> BAD_KID
    API -->|permission lookup| CB
    CB --> PERMS
    API -->|read/write| Aurora
    API -->|cache| Valkey
    API -->|feature flags| DDB
    API -->|startup| SM
    API -->|startup| SSM
    KMS -->|encrypts| Aurora
    KMS -->|encrypts| Valkey
    KMS -->|encrypts| SM

    style Veracode fill:#ff9900,color:#000
    style BlackDuck fill:#ff9900,color:#000
    style Sonar fill:#4e9a06,color:#fff
    style KMS fill:#d62728,color:#fff
    style SM fill:#d62728,color:#fff
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
flowchart TD
    subgraph "Developer Workflow"
        DEV["Developer Push / PR\nto master or versioned branch\n(e.g. 26.2, 26.2.0)"]
    end

    subgraph "GitHub Actions — BackendCI  [BuildPushECR.yaml]"
        CHECKOUT["1. Checkout Code\n(actions/checkout@v4)"]
        JDK["2. Setup JDK 17\n(Zulu distribution)"]
        GRADLE_PERM["3. chmod +x gradlew"]
        
        subgraph "Integration Tests  [matrix: dev | test | staging]"
            INT_TEST["4. ./gradlew test\n-PincludeIntegrationTests\n-PactiveProfile={env}"]
            UPLOAD_XML["5. Upload Test Results XML\n(30-day retention)"]
            UPLOAD_HTML["6. Upload Test Report HTML\n(30-day retention)"]
        end

        subgraph "Reusable Build Workflow  [acddevops-reusable-workflows]"
            SONAR["7. SonarQube Scan\n✅ enable-sonar-scan: true"]
            SQ_GATE{Quality Gate}
            BLACKDUCK["8. BlackDuck SCA\n✅ enable-blackduck-scan: true\n(dependency scan)"]
            VERACODE["9. Veracode SAST\n✅ enable-veracode-scan: true\nartifact: ms-agent-profiles-apis-1.0-SNAPSHOT"]
            SEC_GATE{Security Gate}
            JIB_BUILD["10. Gradle Build + Jib\n./gradlew build\nJAR: ms-agent-profiles-apis-1.0-SNAPSHOT.jar\nImage tag: git SHA"]
            ECR_PUSH["11. Push Docker Image to ECR\n300813158921.dkr.ecr.us-west-2.amazonaws.com\nTagged: github.sha"]
        end
    end

    subgraph "GitHub Actions — Infra Deploy  [DeployInfra.yaml]"
        INFRA_TRIGGER["workflow_dispatch\nInputs: profile (dev/test/perf/staging)\nStack (rds-aurora / valkey / secrets / etc.)"]
        ENV_APPROVAL["Environment: approval_needed\n⏸ Manual Approval Gate"]
        AWS_AUTH["OIDC → assume\nServiceAccess-agent-profiles-service-deploy-role"]
        CFN_DEPLOY["aws-cloudformation-github-deploy\nDeploys selected CFN stack\nno-fail-on-empty-changeset: true"]
    end

    subgraph "GitHub Actions — Image Promote  [PromoteImage.yaml / PromoteImageToProd.yaml]"
        PROMOTE_TRIGGER["workflow_dispatch\nPromote image between environments"]
        PROD_APPROVAL["⏸ Approval Gate for Production"]
        ECR_PROMOTE["Promote image tag in ECR\nfrom staging → prod account"]
    end

    subgraph "Jenkins — Frontend CI  [Jenkinsfile]"
        FE_CHECKOUT["1. Checkout cxone-webapp-agent-profiles@master"]
        FE_INSTALL["2. npm install"]
        FE_LINT["3. ESLint + TSLint"]
        FE_TEST["4. Unit Tests (Karma/Jasmine)\nCoverage: LCOV → target/reports/coverage"]
        FE_SONAR["5. SonarQube Scan\nsonar-project.properties"]
        FE_BUILD["6. ng build / webpack --config webpack.prod.config.js"]
        FE_DEPLOY["7. Deploy to CDN / S3\n(pipeline.properties)"]
    end

    DEV --> CHECKOUT
    CHECKOUT --> JDK --> GRADLE_PERM --> INT_TEST
    INT_TEST --> UPLOAD_XML & UPLOAD_HTML
    UPLOAD_XML & UPLOAD_HTML --> SONAR
    SONAR --> SQ_GATE
    SQ_GATE -->|Fail| SQFAIL["❌ Build blocked\nQuality gate failed"]
    SQ_GATE -->|Pass| BLACKDUCK --> VERACODE
    VERACODE --> SEC_GATE
    SEC_GATE -->|Fail| SECFAIL["❌ Build blocked\nSecurity findings"]
    SEC_GATE -->|Pass| JIB_BUILD --> ECR_PUSH

    INFRA_TRIGGER --> ENV_APPROVAL --> AWS_AUTH --> CFN_DEPLOY

    PROMOTE_TRIGGER --> PROD_APPROVAL --> ECR_PROMOTE

    DEV --> FE_CHECKOUT --> FE_INSTALL --> FE_LINT --> FE_TEST --> FE_SONAR --> FE_BUILD --> FE_DEPLOY
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
        REST["REST API"]
        KIN["Kinesis Streams\n(TeamData + userManagement)"]
    end

    subgraph APP["Application Layer"]
        SVC["AgentProfilesServiceImpl"]
        CACHE["CacheResponseServiceImpl\n(cache-aside)"]
    end

    subgraph STORES["Data Stores"]
        direction TB
        AUR_W[("Aurora MySQL\nWriter\nPort 3306")]
        AUR_R[("Aurora MySQL\nReader x2\nPort 3306")]
        VALKEY[("ElastiCache Valkey\nPort 6379\nMulti-AZ")]
        DDB[("DynamoDB\nFeature Toggles")]
    end

    subgraph CONFIG["Config & Secrets (startup)"]
        SM["Secrets Manager\nDB credentials"]
        SSM["SSM Param Store\nEndpoints + flags"]
    end

    REST --> SVC
    KIN --> SVC
    SVC --> CACHE
    CACHE -->|"cache hit"| REST
    CACHE <-->|"get / set / del"| VALKEY
    CACHE -->|"cache miss — write"| AUR_W
    CACHE -->|"cache miss — read"| AUR_R
    SVC -->|"writes"| AUR_W
    SVC -->|"reads"| AUR_R
    SVC -->|"feature flags"| DDB
    SM -->|"injected at startup"| SVC
    SSM -->|"injected at startup"| SVC
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
