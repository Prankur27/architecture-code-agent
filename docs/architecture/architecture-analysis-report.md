# Architecture Analysis Report
## System: Agent Profiles — NICE CXone
**Backend**: `nice-cxone/ms-agent-profiles-apis` @ branch `26.2`
**Frontend**: `nice-cxone/cxone-webapp-agent-profiles` @ branch `master`
**Generated**: March 22, 2026

---

## 1. System Overview

### 1.1 High-Level Description
The **Agent Profiles** system is a multi-tenant SaaS microservice within the **NICE CXone** contact center platform (formerly inContact/ACD). It manages the configuration of agent desktop profiles — named groupings of settings and assignable configurations (such as desktop layouts, tool availability, and behavior parameters) that are assigned to one or more agent teams.

The system exposes a REST API consumed by:
- An **Angular micro-frontend** (`cxone-webapp-agent-profiles`) embedded within the CXone shell application via **Module Federation / Web Components**
- **CXone Agent (CXA)** — an internal CXone consumer of agent profile data via dedicated CXA-specific endpoints (v1 and v2)
- The **CXone Platform** (via `PlatformApiConsumerServiceImpl`) for user permissions resolution and team/agent data fetching

### 1.2 Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend Language | Java 17 |
| Backend Framework | Spring Boot (Jakarta EE, Spring Security OAuth2 Resource Server) |
| Build Tool | Gradle (Kotlin DSL) with Jib (container image build) |
| Frontend Framework | Angular (with `angular.json`, `app.module.ts`) |
| Frontend Build | Webpack (Module Federation), Angular CLI |
| Frontend Package Manager | npm |
| API Style | REST (JSON over HTTPS), versioned at `/agent-profiles/v1` |
| Primary Database | AWS Aurora MySQL 8 (aurora-mysql 3.08.2) — 3-instance cluster |
| Cache | AWS ElastiCache Valkey (Redis-compatible) — Multi-AZ replication group |
| Event Streaming | AWS Kinesis Data Streams (2 streams: `TeamData`, `userManagement`) |
| Feature Store | AWS DynamoDB |
| Secrets | AWS Secrets Manager |
| Configuration | AWS SSM Parameter Store |
| Observability | Prometheus metrics, OTLP traces (OpenTelemetry), Logback structured logging |
| Container Registry | AWS ECR |
| CI/CD (Backend) | GitHub Actions |
| CI/CD (Frontend) | Jenkins (`Jenkinsfile`) |
| Security Scanning | Veracode SAST, BlackDuck SCA (backend); SonarQube (frontend) |

### 1.3 System Boundaries and External Dependencies

| External System | Direction | Purpose |
|----------------|-----------|---------|
| CXone Platform API (`na1.dev.nice-incontact.com`) | Outbound | Fetch user permissions, agent details, team details |
| ACD API (`api-na1.dev.niceincontact.com/inContactAPI`) | Outbound | Legacy ACD integration |
| CXone Auth / JWKS (`cxone.dev.niceincontact.com/auth/jwks`) | Outbound | JWT token validation (JWKS endpoint) |
| AWS Kinesis (`dev-TeamData`, `dev-userManagement` streams) | Inbound (consumer) | Consume team change events and user management events |
| AWS Aurora MySQL | Internal | Primary data store |
| AWS ElastiCache Valkey | Internal | Response cache |
| AWS DynamoDB | Internal | Feature toggles store |
| AWS Secrets Manager | Internal | DB credentials, secrets at rest |
| AWS SSM Parameter Store | Internal | Runtime configuration (FIPS, encryption flags, endpoints) |
| OTLP Collector (`collector-mon.nicecxone-dev.com`) | Outbound | Metrics and traces export |

---

## 2. Backend Architecture

### 2.1 Architecture Style & Patterns

The backend follows a **Layered Monolith / Single Microservice** architecture style:
- **Single deployable unit** packaged as a Docker image via Jib, deployed to ECS (inferred from ECR workflow)
- **Layered architecture**: Controller → Service → DAO/Repository → JPA Entities
- **AOP-based authorization**: `@Authorized` custom annotation processed by `AuthAspect.java` (Spring AOP `@Around` advice)
- **Interface-based service design**: `IAgentProfilesService`, `IPlatformApiConsumerService`, `ICacheResponseService` with `impls/` implementations
- **Dual datasource pattern**: separate write and read replica connections (`database.write.replica.*`, `database.read.replica.*`)
- **Multi-tenancy via schema-per-tenant**: `agentprofiles_schema_tenant_` prefix, `TenantSchema.java` entity
- **Resilience patterns**: Resilience4j Rate Limiter (`cxalimiter`: 75 req/s) and Circuit Breaker (`userhub`: 50% failure threshold, 10s open state)
- **Event-driven consumption**: KCL (Kinesis Client Library) consumers for `TeamData` and `userManagement` streams

### 2.2 Component & Module Structure

```
src/main/java/com/nice/cxone/agentprofiles/
├── App.java                          # Spring Boot entry point
├── AppConfiguration.java             # Bean wiring, HTTP clients, cache configs
├── Logger/
│   └── LogAttributesThreadLocal.java # MDC-style thread-local logging context
├── auth/
│   ├── AuthAspect.java               # AOP: enforces @Authorized permissions
│   ├── AuthService.java              # JWT parsing (Nimbus), permission resolution
│   ├── interfaces/Authorized.java    # Custom auth annotation
│   └── types/AuthRole.java           # Role enum (legacyId-based)
├── common/                           # Shared utilities/base classes
├── configuration/                    # Spring configuration classes
├── controller/
│   ├── AgentProfilesController.java         # Main CRUD REST controller
│   ├── AgentProfilesControllerBase.java     # Base class: exception handling, message source
│   ├── AssignableConfigurationsController.java
│   ├── GetAgentProfileCXAControllerV2.java  # CXA-specific v2 endpoint
│   ├── GetAgentProfileForCXAController.java # CXA-specific v1 endpoint
│   ├── ApiConstants.java                    # URL path and default constants
│   ├── AgentProfileApiErrorCodes.java       # Error code enum
│   └── dto/                                 # Request/Response DTOs
├── jpa/
│   ├── entities/
│   │   ├── AgentProfiles.java               # Core profile entity
│   │   ├── AgentProfileConfiguration.java   # Profile configuration settings
│   │   ├── AgentProfileTeamsMap.java        # Profile-Team many-to-many mapping
│   │   ├── AssignableConfigurations.java    # Available configuration catalog
│   │   ├── AssignableConfigurationsElements.java
│   │   ├── AssignableConfigurationsElementsDependencies.java
│   │   ├── AssignableConfigurationsDependencyElements.java
│   │   ├── MappedCreatedUpdatedEntity.java  # Audit base entity
│   │   └── TenantSchema.java               # Tenant schema registry
│   ├── repositories/                        # Spring Data JPA repositories
│   └── configuration/                       # JPA datasource configuration
├── kinesis/
│   ├── KinesisConsumer.java                 # KCL worker bootstrap
│   ├── TeamDataRecordProcessor.java         # Processes team change events
│   ├── TeamDataRecordProcessorFactory.java
│   ├── UserManagementRecordProcessor.java   # Processes user/agent change events
│   ├── UserManagementRecordProcessorFactory.java
│   ├── TeamEvents.java                      # Team event types enum/model
│   └── UserManagementEvent.java             # User event model
├── service/
│   ├── impls/
│   │   ├── AgentProfilesServiceImpl.java    # Core business logic (35KB — largest file)
│   │   ├── AgentProfilesDaoImpl.java        # DB access layer
│   │   ├── AgentProfileTeamsMapDaoImpl.java # Team-profile mapping DB access
│   │   ├── CacheResponseServiceImpl.java    # Valkey cache read/write
│   │   └── PlatformApiConsumerServiceImpl.java # External CXone Platform API calls
│   └── interfaces/
│       ├── IAgentProfilesService.java
│       ├── IPlatformApiConsumerService.java
│       └── ICacheResponseService.java
├── utils/
│   └── AgentProfileValidationUtils.java    # Input validation
└── value/                                  # Value objects
```

### 2.3 API Surface

Base path: `/agent-profiles/v1`

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| `GET` | `/profiles` | `AGENTPROFILES_VIEW` | List all profiles (paginated: skip, top, orderBy, fields) |
| `GET` | `/profiles/{id}` | `AGENTPROFILES_VIEW` | Get profile by ID |
| `POST` | `/profiles/search` | `AGENTPROFILES_VIEW` | Filter/search profiles |
| `POST` | `/profiles` | `AGENTPROFILES_CREATE` | Create new profile |
| `PUT` | `/profiles/{id}` | `AGENTPROFILES_EDIT` | Update profile |
| `PUT` | `/profiles/status` | `AGENTPROFILES_EDIT` | Bulk status update |
| `GET` | `/profiles/{id}/teams` | `AGENTPROFILES_VIEW` | Get teams assigned to profile |
| `POST` | `/profiles/{id}/teams` | `AGENTPROFILES_CREATE` | Assign teams to profile |
| `DELETE` | `/profiles/{id}/teams` | `AGENTPROFILES_CREATE` | Remove teams from profile |
| `POST` | `/profiles/teams/search` | `AGENTPROFILES_VIEW` | Search teams |
| `GET` | `/assignable-configurations` | (inferred) | List assignable config catalog |
| `GET` | `/cxa/profiles/{id}` | (CXA v1) | Get profile for CXone Agent v1 |
| `GET` | `/cxa/v2/profiles/{id}` | (CXA v2) | Get profile for CXone Agent v2 |

All endpoints apply `@RateLimiter(name="cxalimiter")` — 75 requests/second.

Excluded from auth (public): `/swagger-ui/`, `/v3/api-docs/`, `/probe/healthreport`, `/alive`, `/ready`, `/probe/prometheus`

### 2.4 Data Layer

**Primary Store: AWS Aurora MySQL 8**
- Engine: `aurora-mysql 8.0.mysql_aurora.3.08.2`
- 3 DB instances (1 primary writer + 2 read replicas)
- `PubliclyAccessible: false`
- `StorageEncrypted: true` (AWS-managed)
- TLS enforced conditionally via `require_secure_transport=ON` / `tls_version=TLSv1.2` (controlled by `EncryptionInTransitInternalEnabled` SSM flag)
- Credentials stored in AWS Secrets Manager (`{regionCode}-AgentProfilesSecrets`)
- Read/write endpoints published to SSM Parameter Store
- Connection pool: HikariCP (`max-pool: 100`, `min-idle: 50`, `max-lifetime: 2.5M ms`, `keepalive: 5min`)
- Schema migration: Flyway (`classpath:db/migration`)
- Multi-tenant: schema-per-tenant with prefix `agentprofiles_schema_tenant_`

**Cache: AWS ElastiCache Valkey (Redis-compatible)**
- Multi-AZ replication group (2 nodes min, default `cache.m7g.xlarge` for staging/perf)
- `AtRestEncryptionEnabled: true`
- `TransitEncryptionEnabled`: conditional on SSM flag `ENCRYPTION_IN_TRANSIT_INTERNAL`
- Automatic failover enabled
- Snapshot retention: 7 days
- Connection: `redis.host` / `redis.port=6379`
- Used by `CacheResponseServiceImpl` for API response caching

**Feature Store: AWS DynamoDB**
- Single table (provisioned via `agent-profiles-dynamo-db-template.yaml`)
- Used for feature toggles (`feature-toggle.url=/config/toggledFeatures`)

**JWKS Cache (L1/L2)**
- L1 in-memory Caffeine cache (TTL: 6–12 hours, max-size: 10 keys)
- L2 persistent cache: TTL 15–30 days
- Bad Kid cache: Caffeine, TTL 5 min, max-size: 1000

### 2.5 Messaging & Events

**AWS Kinesis Data Streams (consumer)**

| Stream | ARN (default) | Application Name | Processor |
|--------|--------------|-----------------|-----------|
| `dev-TeamData` | `arn:aws:kinesis:us-west-2:934137132601:stream/dev-TeamData` | `ms-agent-profiles-kinesis-teamData` | `TeamDataRecordProcessor` |
| `dev-userManagement` | `arn:aws:kinesis:us-west-2:934137132601:stream/dev-userManagement` | `ms-agent-profiles-kinesis-userManagement` | `UserManagementRecordProcessor` |

- `KinesisConsumer.java` bootstraps KCL workers for both streams
- `TeamDataRecordProcessor`: handles team create/update/delete events — updates team-profile mappings in DB
- `UserManagementRecordProcessor`: handles user/agent lifecycle events (create/delete/update/role change) — updates agent-profile associations
- `TeamEvents.java` and `UserManagementEvent.java` define event schemas

---

## 3. Frontend Architecture

### 3.1 Framework & Structure

- **Framework**: Angular (version inferred as 14+ from `angular.json` and `@angular/elements`)
- **Deployment model**: **Micro-Frontend (MFE)** — the app is compiled as a **Web Component** (`agent-profile-app` custom element) using `@angular/elements` via `createCustomElement` and bootstrapped via `ngDoBootstrap()`
- **Module Federation**: Uses `webpack.config.js` + `cxone-mfe-tools` — the app is a **child MFE** hosted within the CXone shell container app
- **Routing**: Angular Router enabled (`useAngularRouter: true`), MFE activate guard in place
- **Build**: `webpack.config.js` + `webpack.prod.config.js`, Angular CLI

**Module structure:**
```
src/app/
├── agent-profiles/         # Main profiles list view + service
├── create-edit-agent-profile/  # Create/edit profile form
├── duplicate-desktop-profile/  # Duplicate profile flow
├── teams-tab/              # Team assignment tab
├── shared/                 # Shared components, models, utilities
│   └── models/api.constants.ts  # API URL constants
├── app.module.ts           # Root module with Web Component export
├── app-api.service.ts      # HTTP wrapper (GET/POST/PUT/DELETE)
├── app.routes.ts           # Angular route definitions
└── app.component.ts        # Root shell component
src/models/                 # Shared TypeScript models
src/environments/           # Environment-specific config
```

### 3.2 State Management

- **No dedicated state management library** (no Redux/NgRx/Zustand found)
- State is managed **locally within components** and **Angular services** (`ProfileService`, `APIService`)
- `ProfileService` holds `profileData: Profile[]` and `transformedProfiles: Profile[]` in-memory
- Reactive data flow via **RxJS Observables** (`Observable<>`, `map()`, `.pipe()`)

### 3.3 API Integration Patterns

- **API base URL**: `/agent-profiles` (relative path — proxied by shell or reverse proxy to backend at `/agent-profiles/v1`)
- **HTTP client**: `HttpUtils` from `@niceltd/cxone-core-services` (wraps Angular `HttpClient` with platform-level interceptors)
- **Interceptors** (registered in `app.module.ts`):
  - `CXOneHttpRequestInterceptor` — injects auth tokens, tenant headers
  - `CXOneHttpResponseInterceptor` — handles error responses
  - `AppSpinnerInterceptor` — global loading state
- **Token handling**: Managed by `@niceltd/cxone-client-platform-services` (`TenantCrossDomainParamsService`, `ConfigurationService`) — tokens are not manually stored; platform services handle injection
- **API calls in `ProfileService`**:
  - `POST /v1/profiles/search` — paginated/filtered profile listing
  - `GET /v1/profiles/{id}` — fetch single profile
  - `GET /v1/assignable-configurations` — fetch config catalog
  - `PUT /v1/profiles/status` — status update
  - `POST /v1/profiles` — create profile
  - `PUT /v1/profiles/{id}` — update profile
  - `POST /v1/profiles/{id}/teams` — assign teams

### 3.4 UI-Backend Contract

- **API constants** defined in `src/app/shared/models/api.constants.ts`
- **Models**: `Profile`, `AgentProfilesResponse`, `AgentProfileApi` in `agent-profiles.model.ts`
- **No OpenAPI spec** found in either repo (gap — no shared contract file)
- **No GraphQL** — pure REST
- **SonarQube** configured via `sonar-project.properties`:
  - Project key: `com.nice.cxone-webapp-agent-profiles`
  - Sources: `src/app`
  - Test inclusions: `**/*.spec.ts`
  - Coverage: LCOV at `target/reports/coverage/**/lcov.info`

---

## 4. Cloud Infrastructure

### 4.1 AWS Resources & Topology

All infrastructure is deployed to **AWS region `us-west-2`** across 4 environments:

| Environment | Profile | AWS Account ID | CIDR |
|-------------|---------|---------------|------|
| Dev | `ic-dev` | `300813158921` | `10.9.48.0/21` |
| Test | `ic-test` | `265671366761` | `10.9.64.0/21` |
| Perf | `ic-perf` | `150598861634` | `10.9.112.0/21` |
| Staging | `ic-staging` | `545209810301` | `10.9.80.0/21` |

**CloudFormation Stacks:**

| Stack Name | Template | Resources Created |
|-----------|----------|------------------|
| `{env}-agentprofiles-rds-aurora` | `agent-profiles-rds-aurora-template.yaml` | Aurora MySQL cluster + 3 instances + security group + subnet group + SSM params for endpoints |
| `{env}-agentprofiles-valkey-cluster` | `agent-profiles-valkey-cache-template.yaml` | ElastiCache Valkey replication group (Multi-AZ) + security group + subnet group + SSM params |
| `{env}-agentprofiles-jump-server` | `agent-profiles-jump-server-template.yaml` | EC2 jump server for DB/cache management access |
| `{env}-agentprofiles-dynamo-db` | `agent-profiles-dynamo-db-template.yaml` | DynamoDB table (feature toggles) |
| `{env}-agentprofiles-secrets` | `agent-profiles-secret-template.yaml` | AWS Secrets Manager secret for DB credentials |
| `{env}-agentprofiles-ssm-params` | `agent-profiles-ssm-parameter-template.yaml` | SSM parameters: `FIPS_ENABLED`, `ENCRYPTION_IN_TRANSIT_INTERNAL` |
| `{env}-agentprofiles-kms-key` | `agent-profiles-kms-key-template.yaml` | KMS key (alias: `agentProfiles-kms-key`) for SNS/CloudWatch encryption |
| `{env}-agentprofiles-cloudwatch-alarms` | `agent-profiles-cloudwatch-template.yaml` | CloudWatch alarms for RDS + Valkey metrics |

**Network**: VPC imported via `CoreNetwork-Vpc`. Subnets imported via `CoreNetwork-Az1CoreSubnet` and `CoreNetwork-Az2CoreSubnet` — multi-AZ by design.

### 4.2 Networking & Security Groups

- **RDS Security Group** (`DBSecurityGroup`): Allows inbound TCP 3306 from `PrimaryCIDR` and `ManagementCIDR` (jump server IP/32)
- **Valkey Security Group** (`SecurityGroup`): Allows inbound TCP 6379 from `PrimaryCIDR` and `ManagementCIDR`
- Both SGs: open egress (`0.0.0.0/0`)
- `PubliclyAccessible: false` on all RDS instances
- VPC is externally managed (`CoreNetwork-Vpc` cross-stack import) — networking is inherited from platform core
- No WAF configuration found in CloudFormation (may be at platform/ALB level upstream)

### 4.3 Deployment Architecture

- Backend application container image built via **Jib** (no Dockerfile needed, direct to ECR)
- Image tag = `${{ github.sha }}`
- Deployed to ECS (inferred from ECR usage and NICE CXone platform patterns)
- ECR account: `300813158921` (dev, `us-west-2`)
- Deploy role: `ServiceAccess-agent-profiles-service-deploy-role` (OIDC-assumed via GitHub Actions)
- Application port: `9001` (app); `8080` (management/actuator)
- Frontend deployed via Jenkins pipeline (`Jenkinsfile`) to static hosting (CDN/S3 inferred from MFE architecture)

### 4.4 Auto-scaling & High Availability

- **Aurora**: 3-instance cluster (1 write + 2 read replicas), `BackupRetentionPeriod: 30 days`, deletion protection configurable
- **Valkey**: Multi-AZ replication group, `AutomaticFailoverEnabled: true`, snapshot retention 7 days
- **HikariCP**: Pool max 100 connections with keepalive pings for RDS idle disconnect prevention
- **Resilience4j Circuit Breaker** (`userhub`): sliding window 10, failure threshold 50%, 10s open state
- **Rate limiter**: 75 req/s per instance

---

## 5. CI/CD Pipeline

### 5.1 Build & Test Pipeline (`BuildPushECR.yaml`)

Triggered on: PR or push to `master`, `[2-9][3-9].[0-9]`, `[2-9][3-9].[0-9].[0-9]` branches

**Stage 1: Integration Tests** (matrix: dev, test, staging environments)
1. Checkout code
2. Setup JDK 17 (Zulu distribution)
3. `./gradlew test -PincludeIntegrationTests -PactiveProfile={env}`
4. Upload test results XML (30-day retention)
5. Upload HTML test report (30-day retention)

**Stage 2: Build** (reusable workflow `acddevops-reusable-workflows/code-build-and-deploy.yml@v0`)
- Language: Java/Gradle with Jib
- Sonar scan: ✅ `enable-sonar-scan: true`
- BlackDuck SCA scan: ✅ `enable-blackduck-scan: true` (docker scan disabled)
- Veracode SAST scan: ✅ `enable-veracode-scan: true` (artifact: `ms-agent-profiles-apis-1.0-SNAPSHOT`)
- Build artifact: `ms-agent-profiles-apis-1.0-SNAPSHOT.jar`
- Push to ECR: ✅ on any branch matching `.*`

### 5.2 Deployment Pipeline

**Non-prod (`DeployInfra.yaml`)** — Manual `workflow_dispatch`:
- Requires environment `approval_needed` (approval gate enforced by GitHub Environment)
- Deploys individual CloudFormation stacks selectable by input
- Environments: `ic-dev`, `ic-test`, `ic-perf`, `ic-staging`
- OIDC auth to AWS using `ServiceAccess-agent-profiles-service-deploy-role`

**Production (`ProdDeployInfra.yaml` + `PromoteImageToProd.yaml`)**:
- Separate workflow for prod stack deployment
- `PromoteImageToProd.yaml` — image promotion from staging ECR to prod ECR
- `PromoteImage.yaml` — cross-environment image promotion (non-prod)

**Frontend CI/CD**: Jenkins-based (`Jenkinsfile`), pipeline properties in `pipeline.properties`

### 5.3 Environment Strategy

| Environment | Branch Pattern | FIPS | Encryption In Transit | Cache Node | DB Instance |
|-------------|---------------|------|----------------------|-----------|-------------|
| Dev (`ic-dev`) | any branch | ❌ | ❌ | `cache.t2.micro` | `db.t3.medium` |
| Test (`ic-test`) | any branch | ✅ | ✅ | `cache.t2.micro` | `db.t3.medium` |
| Perf (`ic-perf`) | any branch | ✅ | ✅ | `cache.m7g.xlarge` | `db.r7g.large` |
| Staging (`ic-staging`) | any branch | ❌ | ✅ | `cache.m7g.xlarge` | `db.r7g.large` |
| Production | (separate workflow) | (inferred ✅) | ✅ | (prod sizing) | (prod sizing) |

---

## 6. Security Analysis

### 6.1 Static Analysis

**Backend:**
- **Veracode SAST**: Configured in `BuildPushECR.yaml` via `enable-veracode-scan: true`, artifact `ms-agent-profiles-apis-1.0-SNAPSHOT` — runs on every push/PR to versioned branches
- **BlackDuck SCA**: Configured (`enable-blackduck-scan: true`) — docker scan disabled but dependency scan active
- **SonarQube**: `enable-sonar-scan: true` in reusable workflow

**Frontend:**
- **SonarQube**: Configured via `sonar-project.properties` (`com.nice.cxone-webapp-agent-profiles`)
  - Sources: `src/app`, Tests: `**/*.spec.ts`
  - Coverage: LCOV reports from `target/reports/coverage`
- **No Veracode** configured in the UI repo (Jenkins pipeline — coverage unknown)

### 6.2 Authentication & Authorization

**JWT-based Auth with Custom AOP Enforcement:**

1. All API requests must carry a **Bearer JWT token** in the `Authorization` header
2. `AppConfiguration` configures Spring Security OAuth2 Resource Server with JWKS URI (`CX_JWKS_URI`, default: `https://cxone.dev.niceincontact.com/auth/jwks`)
3. `AuthService.parseClaims()` parses the JWT using **Nimbus JOSE+JWT** library
4. `AuthService.getUserPermissions()` calls `PlatformApiConsumerServiceImpl.getUserPermissionsFromToken()` — fetches the user's permissions from the CXone Platform API
5. `AuthAspect.java` — Spring AOP `@Around` advice intercepts all `@Authorized` annotated controller methods, extracts JWT from the request, validates claims, calls `AuthService`, and checks required `authPermissions[]`
6. **Permission model**: Fine-grained — `AGENTPROFILES_VIEW`, `AGENTPROFILES_CREATE`, `AGENTPROFILES_EDIT`
7. **Role extraction**: `AuthService.getUserRoleFromClaims()` extracts `role.legacyId` from JWT claims
8. **Tenant isolation**: `getBUIdFromClaims()` extracts `icBUId`; `getTenantIdFromClaims()` extracts `tenantId` — used for schema-per-tenant DB routing
9. **JWKS caching**: Two-tier Caffeine L1 (6–12h TTL) + persistent L2 (15–30 days TTL); bad kid (invalid key ID) cache (5 min, 1000 entries)

**Frontend:**
- Auth tokens managed by `@niceltd/cxone-client-platform-services` (`TenantCrossDomainParamsService`)
- `CXOneHttpRequestInterceptor` automatically injects tokens on all API calls
- No manual token storage (localStorage/sessionStorage) — platform-managed

### 6.3 Secrets & Configuration Management

| Secret/Config | Storage | Access Method |
|--------------|---------|--------------|
| DB Username/Password | AWS Secrets Manager (`{env}-AgentProfilesSecrets`) | CloudFormation `{{resolve:secretsmanager:...}}` at stack deploy time |
| DB Endpoints | AWS SSM Parameter Store | App reads at startup via `CX_*` env vars |
| FIPS Enable Flag | SSM `/agent-profiles/FIPS_ENABLED` | CloudFormation SSM ref |
| Encryption In Transit Flag | SSM `/agent-profiles/ENCRYPTION_IN_TRANSIT_INTERNAL` | CloudFormation SSM ref + app env var |
| Kinesis Stream ARNs | Environment variables (`AP_TEAMDATASTREAM_ARN`, `AP_USERMANAGEMENTSTREAM_ARN`) | App config |
| WFO Assume Role ARN | Env var `AP_WFOASSUMEROLE_ARN` | App config |
| JWKS URI | Env var `CX_JWKS_URI` | App config |
| CXOne Base URLs | Env vars `CXONEBaseUrl`, `ACD_API_BASE_URL` | App config |

**Gaps identified:**
- `dynamodb.accesskey` and `dynamodb.secretKey` properties exist in `application.properties` with empty values — DynamoDB access appears to use IAM role (correct), but presence of plaintext key fields is a risk if accidentally populated

### 6.4 Network Security

- All DB and cache resources use **private subnets** (`CoreNetwork-Az1/Az2CoreSubnet`)
- `PubliclyAccessible: false` on all RDS instances
- Security groups restrict inbound to CIDR ranges only (no open internet access)
- Jump server exists for management access with controlled CIDR (`ManagementCIDR`)
- TLS in transit: configurable via `EncryptionInTransitInternalEnabled` SSM flag
  - **Dev**: TLS disabled — ⚠️ risk
  - **Test, Perf, Staging**: TLS enabled
- FIPS mode: enabled on `ic-test` and `ic-perf` only; **not on ic-dev or ic-staging** — ⚠️ gap in staging
- No WAF found in CloudFormation templates (may be upstream at platform level)
- RDS egress rule is `0.0.0.0/0` (all traffic) — ⚠️ overly permissive egress

### 6.5 Data Security & Encryption

| Data | Encryption |
|------|-----------|
| RDS storage | ✅ `StorageEncrypted: true` (AWS-managed) |
| RDS in-transit | ✅ Conditional TLSv1.2 (SSM-controlled) |
| Valkey storage | ✅ `AtRestEncryptionEnabled: true` |
| Valkey in-transit | ✅ Conditional `TransitEncryptionEnabled` |
| Secrets Manager | ✅ AWS-managed encryption |
| KMS Key | ✅ Custom KMS key (`agentProfiles-kms-key`) for SNS/CloudWatch, key rotation on FIPS |
| JWT tokens | ✅ Signed JWTs (Nimbus JOSE validation) |

### 6.6 Identified Security Risks & Recommendations

| # | Risk | Severity | Recommendation |
|---|------|----------|---------------|
| 1 | Dev environment has no TLS in transit for DB/Valkey | Medium | Enable TLS in dev; use consistent security baseline |
| 2 | FIPS not enabled in `ic-staging` | Medium | Enable FIPS on staging to match production security posture |
| 3 | RDS security group allows `IpProtocol: '-1'` (all protocols) from `PrimaryCIDR` — in addition to TCP 3306 | Medium | Restrict to TCP 3306 only; remove protocol `-1` rule |
| 4 | RDS egress group: `CidrIp: 0.0.0.0/0` on all protocols | Medium | Restrict egress to necessary destinations only |
| 5 | KMS key policy: `Action: 'kms:*'` for `Principal: AWS: '*'` | High | Restrict to specific IAM roles/accounts; wildcard grants full key control |
| 6 | DynamoDB access key fields (`dynamodb.accesskey`) in `application.properties` | Low | Remove fields; enforce IAM role-only access |
| 7 | No OpenAPI contract between UI and backend | Low | Generate and version an OpenAPI spec to prevent drift |
| 8 | Veracode not configured for the UI (frontend) repo | Medium | Add Veracode pipeline scan to Jenkins `Jenkinsfile` for the Angular app |

---

## 7. Non-Functional Requirements

### 7.1 Performance & Caching

- **Rate limiting**: Resilience4j `cxalimiter` — 75 req/s, 0ms timeout (instant reject over limit), per-instance
- **Response caching**: `CacheResponseServiceImpl` — reads/writes to Valkey (Redis-compatible) for API responses
- **JWKS caching**: 2-tier (Caffeine L1 + persistent L2) to avoid repeated external JWKS calls
- **HikariCP connection pool**: max 100 connections, keepalive pings every 5 minutes, `SELECT 1` validation
- **Pagination**: All list endpoints support `skip`, `top`, `orderBy`, `fields` parameters

### 7.2 Scalability

- Backend is a **horizontally scalable** stateless service (session state in Valkey, not in-process)
- ECS deployment (inferred) allows auto-scaling based on CPU/memory
- Dual datasource (write + read replica) supports read scale-out
- Kinesis consumers use KCL — supports shard-based parallel processing

### 7.3 Availability & Reliability

- **Circuit Breaker** (`userhub`): Protects calls to the CXone Platform API — 50% failure threshold, 10s wait in open state
- **Aurora Multi-AZ**: 3-instance cluster with automatic failover
- **Valkey Multi-AZ**: Replication group with `AutomaticFailoverEnabled: true`
- **Health probes**: Liveness (`/alive`), Readiness (`/ready`), and detailed health (`/probe/healthreport`) via Spring Actuator
- **Resilience4j health indicator**: Rate limiter and circuit breaker states exposed to health endpoint

### 7.4 Observability & Monitoring

| Concern | Implementation |
|---------|---------------|
| Logging | Logback (`logback.xml`), SLF4J, `LogAttributesThreadLocal` for MDC context |
| Metrics | Prometheus endpoint at `/probe/prometheus`; OTLP metrics export to `collector-mon.nicecxone-dev.com` (`CX_OTLP_METRICS_URL`) |
| Tracing | OpenTelemetry (OTLP traces) to `collector-mon.nicecxone-dev.com` (`CX_OTLP_TRACES_URL`) |
| Alerting | CloudWatch Alarms (separate CFN stack `cloudwatch-alarms`) for RDS and Valkey metrics, KMS-encrypted SNS notifications |
| Rate limiter health | Exposed via Spring Actuator health endpoint |
| Circuit breaker health | Exposed via Spring Actuator health endpoint |

### 7.5 Disaster Recovery

- **RDS backup**: 30-day snapshot retention, automated backups
- **Valkey snapshots**: 7-day retention, daily window `05:00-09:00 UTC`
- **DB deletion protection**: Configurable (`DBDeletionProtection` parameter, default `false`) — ⚠️ should be `true` in prod
- **Multi-AZ failover**: Both Aurora and Valkey have automatic failover
- No explicit multi-region DR configuration found (single `us-west-2` region per account)

---

## 8. Data Flows

### 8.1 Key Business Flows

1. **Get All Agent Profiles (Paginated List)** — UI calls `POST /v1/profiles/search` with filter/pagination payload → `AgentProfilesController` → `AgentProfilesServiceImpl` → check Valkey cache → on miss, query Aurora read replica → return paginated `AgentProfilesResponse`

2. **Create Agent Profile** — UI submits form → `POST /v1/profiles` → `@Authorized(AGENTPROFILES_CREATE)` AOP check → `AgentProfilesServiceImpl.createAgentProfile()` → validate uniqueness (no duplicate name) → write to Aurora → invalidate Valkey cache → return created profile

3. **Update Agent Profile** — `PUT /v1/profiles/{id}` → validate → `AgentProfilesServiceImpl.updateAgentProfile()` → update Aurora → cache invalidation

4. **Update Profile Status (Bulk)** — `PUT /v1/profiles/status` → `AgentProfilesServiceImpl.updateAgentProfileStatus()` → returns `SuccessResponseDTO(true/false)` with `304 NOT_MODIFIED` if no change

5. **Assign Teams to Profile** — `POST /v1/profiles/{id}/teams` → `AgentProfilesServiceImpl.assignTeamsToAgentProfile()` → validate team IDs → write to `AgentProfileTeamsMap` junction table → update cache

6. **Remove Teams from Profile** — `DELETE /v1/profiles/{id}/teams` → `AgentProfileTeamsMapDaoImpl` → delete records from junction table

7. **Get Profile for CXA (v1/v2)** — Internal CXone Agent consumers fetch profile data via dedicated endpoints (`GetAgentProfileForCXAController`, `GetAgentProfileCXAControllerV2`) — likely lighter payload than full admin API

8. **Team Data Change Event** — Kinesis `dev-TeamData` stream record → `KinesisConsumer` → `TeamDataRecordProcessor` → parse `TeamEvents` → update team references in DB (rename/delete team syncing)

9. **User Management Event** — Kinesis `dev-userManagement` stream → `UserManagementRecordProcessor` → parse `UserManagementEvent` → update agent-profile associations (agent hire/term/role change)

10. **Get Assignable Configurations Catalog** — `GET /v1/assignable-configurations` → `AssignableConfigurationsController` → return full catalog of configurable items available for agent profile assignment

### 8.2 Integration Flows

1. **Permission Check** — On every authenticated request, `AuthService.getUserPermissions()` calls CXone Platform API (`/user/permissions`) to fetch the calling user's permissions — result used by `AuthAspect` for RBAC enforcement

2. **Agent Details Fetch** — `PlatformApiConsumerServiceImpl` calls `/services/v31.0/agents/{id}` on `CXONEBaseUrl` to fetch agent metadata

3. **Team Details Fetch** — `PlatformApiConsumerServiceImpl` calls `/services/v31.0/teams` to fetch team metadata for profile-team assignment flows

4. **GA Redirection** — `GA_REDIRECTION_URL` used for redirect to global admin portal

---

## 9. Cross-Cutting Concerns

### 9.1 Shared Libraries & Contracts

**Backend:**
- Reusable GitHub Actions workflow: `inContact/acddevops-reusable-workflows/.github/workflows/code-build-and-deploy.yml@v0` (standardizes CI across all NICE microservices)
- `CODEOWNERS` file in `.github/CODEOWNERS` — team ownership defined
- No inter-service SDK/shared library found (platform integration via HTTP only)

**Frontend:**
- `@niceltd/cxone-core-services` — platform services: `HttpUtils`, `AppInitializerFactory`, `CXOneHttpRequestInterceptor`, `ConfigurationService`, `WebAppInitializerService`
- `@niceltd/cxone-client-platform-services` — auth/tenant: `TenantCrossDomainParamsService`, `ConfigurationService`
- `@niceltd/cxone-domain-components/app-spinner` — spinner interceptor
- `@niceltd/sol/translation` — i18n locale provider
- `cxone-mfe-tools` — Module Federation host/shell integration
- `angular-i18next` — internationalization support

### 9.2 Feature Flags & Experimentation

- Feature toggles stored in **DynamoDB** — endpoint `/config/toggledFeatures`
- `feature-toggle.url` property configures the toggle API
- No A/B testing framework identified

### 9.3 Compliance & Regulatory Considerations

- **FIPS 140-2**: Enabled in `ic-test` and `ic-perf` — indicates compliance requirements (government/regulated customers)
- **Encryption in transit**: Environment-controlled, required in test/perf/staging
- **Key rotation**: KMS key rotation enabled when FIPS is active
- **Audit trail**: `MappedCreatedUpdatedEntity` base entity provides `createdAt`/`updatedAt` audit timestamps on all JPA entities
- **Multi-tenancy**: Schema-per-tenant isolation (`agentprofiles_schema_tenant_{tenantId}`) — strong data segregation

---

## 10. Architecture Risks & Recommendations

| # | Risk | Severity | Area | Recommendation |
|---|------|----------|------|---------------|
| 1 | KMS key policy grants `kms:*` to `Principal: AWS: '*'` | **High** | Security | Restrict to specific IAM principals; wildcard allows any AWS account identity to perform all KMS operations |
| 2 | No OpenAPI/Swagger spec committed to repo | **Medium** | API Contract | Generate `openapi.yaml` from Spring Boot (springdoc) and commit to repo; share with UI team |
| 3 | Dev environment has no encryption in transit (DB/Cache) | **Medium** | Security | Standardize security baseline across all environments |
| 4 | FIPS not enabled on `ic-staging` | **Medium** | Security/Compliance | Staging should match production security posture for pre-prod validation |
| 5 | RDS `DBDeletionProtection: false` (default) | **Medium** | DR | Set to `true` for staging and production |
| 6 | Overly broad RDS security group: `IpProtocol: '-1'` rule + unrestricted egress | **Medium** | Security | Tighten to TCP 3306 inbound only; restrict egress to specific destinations |
| 7 | Veracode not configured for UI (frontend) repo | **Medium** | Security | Add Veracode pipeline scan to Jenkins frontend pipeline |
| 8 | `AgentProfilesServiceImpl.java` is 35KB — very large service class | **Medium** | Maintainability | Decompose into focused sub-services (ProfileCRUDService, TeamAssignmentService, StatusService) |
| 9 | Permission check on every API call makes synchronous call to CXone Platform API | **Medium** | Performance | Cache user permissions in Valkey with short TTL (e.g., 30s) to reduce latency on hot paths |
| 10 | No explicit multi-region DR configuration | **Low** | Availability | Document RTO/RPO targets; evaluate cross-region Aurora read replica or global tables |
| 11 | DynamoDB access key fields in `application.properties` | **Low** | Security | Remove legacy credential fields; enforce IAM role-based access only |
| 12 | No API versioning beyond base path `/v1` on most endpoints | **Low** | API | Implement proper API versioning strategy for backward compatibility |

---

*This report was generated by the Architecture Analyzer Agent. Use it as input for the C4 Diagram Generator Agent and Flow Diagram Generator Agent.*
