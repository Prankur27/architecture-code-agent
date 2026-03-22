# Flow & Sequence Diagrams — Agent Profiles (NICE CXone)
Generated from Architecture Analysis Report — March 22, 2026

---

## Diagram Index
1. [Authentication & JWT Validation Flow](#1-authentication--jwt-validation-flow)
2. [RBAC Permission Check Flow](#2-rbac-permission-check-flow)
3. [Get Agent Profiles List (Cache-Aside)](#3-get-agent-profiles-list-cache-aside)
4. [Create Agent Profile Flow](#4-create-agent-profile-flow)
5. [Update Agent Profile Flow](#5-update-agent-profile-flow)
6. [Update Profile Status (Bulk) Flow](#6-update-profile-status-bulk-flow)
7. [Assign Teams to Profile Flow](#7-assign-teams-to-profile-flow)
8. [CXone Agent (CXA) Profile Fetch Flow](#8-cxone-agent-cxa-profile-fetch-flow)
9. [Kinesis TeamData Event Processing Flow](#9-kinesis-teamdata-event-processing-flow)
10. [Kinesis User Management Event Processing Flow](#10-kinesis-user-management-event-processing-flow)
11. [DynamoDB Feature Toggle Lookup Flow](#11-dynamodb-feature-toggle-lookup-flow)
12. [Backend CI/CD Pipeline Flow](#12-backend-cicd-pipeline-flow)
13. [Infrastructure Provisioning (CloudFormation) Flow](#13-infrastructure-provisioning-cloudformation-flow)
14. [Error Handling & Resilience Flow](#14-error-handling--resilience-flow)
15. [Frontend Bootstrap & MFE Load Flow](#15-frontend-bootstrap--mfe-load-flow)

---

## 1. Authentication & JWT Validation Flow

Covers the end-to-end JWT validation path including the 2-tier JWKS cache (Caffeine L1 + persistent L2) and bad-kid tracking.

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client (Browser / CXA)
    participant ALB as ALB / Platform Proxy
    participant API as Agent Profiles API
    participant AOP as AuthAspect (AOP)
    participant AuthSvc as AuthService
    participant CaffeineL1 as Caffeine L1 Cache<br/>(in-process, 6–12h TTL)
    participant PersistL2 as Persistent L2 Cache<br/>(15–30d TTL)
    participant JWKS as CXone JWKS Endpoint<br/>(cxone.dev.niceincontact.com)
    participant BadKid as Bad Kid Cache<br/>(Caffeine 5min/1000)

    Client->>ALB: HTTP request with Authorization: Bearer {JWT}
    ALB->>API: Forward to port 9001
    Note over API: Spring Security validates Bearer token structure

    API->>AOP: @Around @Authorized intercepts controller method
    AOP->>AuthSvc: parseClaims(jwt)
    AuthSvc->>AuthSvc: Extract header.kid (Key ID) from JWT

    AuthSvc->>BadKid: Check bad kid cache for this kid
    alt kid is in bad kid cache
        BadKid-->>AuthSvc: HIT — kid previously marked invalid
        AuthSvc-->>AOP: throw UnauthorizedException
        AOP-->>Client: 401 Unauthorized
    else kid not in bad kid cache
        BadKid-->>AuthSvc: MISS

        AuthSvc->>CaffeineL1: Lookup JWKS key by kid
        alt L1 cache HIT
            CaffeineL1-->>AuthSvc: Return RSAPublicKey
        else L1 cache MISS
            AuthSvc->>PersistL2: Lookup JWKS key by kid
            alt L2 cache HIT
                PersistL2-->>AuthSvc: Return RSAPublicKey
                AuthSvc->>CaffeineL1: Populate L1 cache
            else L2 cache MISS
                AuthSvc->>JWKS: GET /auth/jwks
                JWKS-->>AuthSvc: Return JWKS (all public keys)
                AuthSvc->>AuthSvc: Find key matching kid
                alt key found
                    AuthSvc->>PersistL2: Store key (15–30d TTL)
                    AuthSvc->>CaffeineL1: Store key (6–12h TTL)
                else key not found
                    AuthSvc->>BadKid: Mark kid as bad (5min TTL)
                    AuthSvc-->>AOP: throw UnauthorizedException
                    AOP-->>Client: 401 Unauthorized
                end
            end
        end

        AuthSvc->>AuthSvc: Nimbus JOSE: verify JWT signature with RSAPublicKey
        AuthSvc->>AuthSvc: Verify expiry (exp), issuer (iss), audience (aud)
        alt JWT invalid / expired
            AuthSvc-->>AOP: throw UnauthorizedException
            AOP-->>Client: 401 Unauthorized
        else JWT valid
            AuthSvc->>AuthSvc: Extract claims: userId, icBUId, tenantId, role.legacyId
            AuthSvc-->>AOP: Return parsed claims
        end
    end
```

**Key Points:**
- Two-tier JWKS caching dramatically reduces calls to the external CXone JWKS endpoint
- Bad Kid cache (5-min TTL) prevents thundering herd on unknown/rotated key IDs
- Nimbus JOSE+JWT performs full cryptographic signature verification on every request

---

## 2. RBAC Permission Check Flow

Covers the permission resolution path after JWT validation — the call to CXone Platform API and the RBAC gate.

```mermaid
sequenceDiagram
    autonumber
    participant AOP as AuthAspect (AOP)
    participant AuthSvc as AuthService
    participant PlatformSvc as PlatformApiConsumerServiceImpl
    participant CB as Circuit Breaker<br/>(Resilience4j: userhub<br/>50% failure / 10s open)
    participant PlatformAPI as CXone Platform API<br/>(/user/permissions)
    participant Controller as Target Controller Method

    Note over AOP: Claims already parsed (see Flow 1)

    AOP->>AuthSvc: getUserPermissions(userId, buId, jwt)
    AuthSvc->>PlatformSvc: getUserPermissionsFromToken(jwt)
    PlatformSvc->>CB: Execute with circuit breaker guard

    alt Circuit Breaker OPEN (too many recent failures)
        CB-->>PlatformSvc: CircuitBreakerOpenException
        PlatformSvc-->>AuthSvc: Service unavailable
        AuthSvc-->>AOP: throw ServiceUnavailableException
        AOP-->>Client: 503 Service Unavailable
    else Circuit Breaker CLOSED / HALF-OPEN
        CB->>PlatformAPI: GET /user/permissions (Authorization: Bearer JWT)
        alt Platform API success
            PlatformAPI-->>CB: 200 OK — {permissions: ["AGENTPROFILES_VIEW", ...]}
            CB-->>PlatformSvc: Return permissions list
            PlatformSvc-->>AuthSvc: Return List<String> permissions
        else Platform API error (4xx/5xx/timeout)
            CB->>CB: Increment failure count
            alt Failure rate ≥ 50%
                CB->>CB: Transition to OPEN state (10s)
            end
            CB-->>PlatformSvc: throw exception
            PlatformSvc-->>AuthSvc: throw exception
            AuthSvc-->>AOP: throw ServiceUnavailableException
            AOP-->>Client: 503 Service Unavailable
        end
    end

    AuthSvc-->>AOP: Return permissions list
    AOP->>AOP: Check required permissions from @Authorized(authPermissions={...})
    alt User has required permission
        AOP->>Controller: Proceed with controller method (ProceedingJoinPoint.proceed())
    else User missing required permission
        AOP-->>Client: 403 Forbidden
    end
```

**Key Points:**
- Every API call triggers a synchronous permission check against CXone Platform API
- Circuit breaker (`userhub`) prevents cascading failure when Platform API is degraded
- Recommendation (from Risk #9): Cache permissions in Valkey with ~30s TTL to reduce Platform API load

---

## 3. Get Agent Profiles List (Cache-Aside)

The standard cache-aside pattern for the paginated profile list query.

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Contact Center Admin
    participant MFE as Angular MFE<br/>(ProfileService)
    participant API as Agent Profiles API<br/>(AgentProfilesController)
    participant AOP as AuthAspect + RateLimiter
    participant Svc as AgentProfilesServiceImpl
    participant CacheSvc as CacheResponseServiceImpl
    participant Valkey as Valkey Cache<br/>(ElastiCache)
    participant DAO as AgentProfilesDaoImpl
    participant Aurora as Aurora MySQL<br/>(Read Replica)

    Admin->>MFE: Opens Agent Profiles page
    MFE->>MFE: ProfileService.getProfilesData(filter, pagination)
    MFE->>API: POST /agent-profiles/v1/profiles/search\n{buId, status, skip, top, orderBy, fields}

    API->>AOP: @RateLimiter(cxalimiter) — check rate (75 req/s)
    alt Over rate limit
        AOP-->>MFE: 429 Too Many Requests
    else Within rate limit
        AOP->>AOP: @Authorized(AGENTPROFILES_VIEW) — JWT + permission check (see Flows 1-2)
        alt Auth fails
            AOP-->>MFE: 401 / 403
        else Auth passes
            AOP->>Svc: searchAgentProfiles(buId, filter, pagination)

            Svc->>CacheSvc: getCachedResponse(cacheKey)
            Note over CacheSvc: Cache key = {buId}:{filter_hash}:{pagination_hash}
            CacheSvc->>Valkey: GET {cacheKey}

            alt Valkey cache HIT
                Valkey-->>CacheSvc: Return cached JSON
                CacheSvc-->>Svc: Return AgentProfilesResponse
                Svc-->>API: Return result (cache hit)
                API-->>MFE: 200 OK — AgentProfilesResponse\n{profiles: [...], total: N}
                Note over MFE: Renders profile list table
            else Valkey cache MISS
                Valkey-->>CacheSvc: nil
                CacheSvc-->>Svc: null
                Svc->>DAO: findAllByBuId(buId, filter, pageable)
                DAO->>Aurora: SELECT * FROM agent_profiles\nWHERE bus_id = ? AND status = ?\nLIMIT ? OFFSET ?
                Aurora-->>DAO: ResultSet (profile rows)
                DAO-->>Svc: List<AgentProfiles>
                Svc->>Svc: Map entities → AgentProfilesResponse DTO
                Svc->>CacheSvc: cacheResponse(cacheKey, response)
                CacheSvc->>Valkey: SET {cacheKey} {json} EX {ttl}
                Valkey-->>CacheSvc: OK
                Svc-->>API: Return AgentProfilesResponse
                API-->>MFE: 200 OK — AgentProfilesResponse
                Note over MFE: Renders profile list table
            end
        end
    end
```

---

## 4. Create Agent Profile Flow

Full lifecycle of creating a new agent profile from the admin UI.

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Contact Center Admin
    participant MFE as Angular MFE<br/>(CreateEditAgentProfileComponent)
    participant ProfileSvc as ProfileService (Angular)
    participant Interceptor as CXOneHttpRequestInterceptor
    participant API as Agent Profiles API<br/>(AgentProfilesController)
    participant AOP as AuthAspect + RateLimiter
    participant Svc as AgentProfilesServiceImpl
    participant Validator as AgentProfileValidationUtils
    participant DAO as AgentProfilesDaoImpl
    participant AuroraWrite as Aurora MySQL<br/>(Writer Endpoint)
    participant CacheSvc as CacheResponseServiceImpl
    participant Valkey as Valkey Cache

    Admin->>MFE: Fills in profile form (name, configurations, teams)
    Admin->>MFE: Clicks "Save" / "Create"
    MFE->>ProfileSvc: createDesktopProfile(profileRequest)
    ProfileSvc->>Interceptor: POST /agent-profiles/v1/profiles
    Interceptor->>Interceptor: Inject Authorization: Bearer {JWT}\nInject X-Tenant-Id, X-BU-Id headers
    Interceptor->>API: POST /v1/profiles {AgentProfileRequest}

    API->>AOP: @RateLimiter check
    AOP->>AOP: @Authorized(AGENTPROFILES_CREATE) — Auth + permission check

    AOP->>Svc: createAgentProfile(buId, tenantId, request)
    Svc->>Validator: validateAgentProfileRequest(request)
    alt Validation fails (missing required fields, invalid format)
        Validator-->>Svc: ValidationException
        Svc-->>API: throw BadRequestException
        API-->>MFE: 400 Bad Request {error details}
        MFE-->>Admin: Show validation error message
    else Validation passes
        Svc->>DAO: existsByNameAndBuId(name, buId)
        DAO->>AuroraWrite: SELECT COUNT(*) WHERE name = ? AND bus_id = ?
        AuroraWrite-->>DAO: count

        alt Profile name already exists for this BU
            DAO-->>Svc: true
            Svc-->>API: throw ConflictException (duplicate name)
            API-->>MFE: 409 Conflict
            MFE-->>Admin: "Profile name already exists"
        else Name is unique
            DAO-->>Svc: false
            Svc->>DAO: save(agentProfile)
            DAO->>AuroraWrite: INSERT INTO agent_profiles (...)\nSET created_at = NOW(), updated_at = NOW()
            AuroraWrite-->>DAO: Generated ID
            Note over DAO: Also saves AgentProfileConfiguration rows
            DAO-->>Svc: Saved AgentProfiles entity (with ID)

            Svc->>CacheSvc: invalidateCache(buId)
            CacheSvc->>Valkey: DEL {buId}:* (pattern delete for this BU)
            Valkey-->>CacheSvc: OK

            Svc-->>API: Return created AgentProfileResponse
            API-->>MFE: 201 Created {profile}
            MFE-->>Admin: Show success toast + navigate to profile list
        end
    end
```

---

## 5. Update Agent Profile Flow

Update an existing profile including configuration changes.

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Contact Center Admin
    participant MFE as Angular MFE<br/>(CreateEditAgentProfileComponent)
    participant API as Agent Profiles API
    participant AOP as AuthAspect
    participant Svc as AgentProfilesServiceImpl
    participant DAO as AgentProfilesDaoImpl
    participant AuroraWrite as Aurora MySQL Writer
    participant AuroraRead as Aurora MySQL Reader
    participant CacheSvc as CacheResponseServiceImpl
    participant Valkey as Valkey Cache

    Admin->>MFE: Opens existing profile for editing
    MFE->>API: GET /v1/profiles/{id}
    Note over API,AuroraRead: Fetch existing profile (cache-aside — see Flow 3 pattern)
    API-->>MFE: 200 OK {existing profile data}
    MFE-->>Admin: Display populated edit form

    Admin->>MFE: Modifies fields (name, configurations)
    Admin->>MFE: Clicks "Update"
    MFE->>API: PUT /v1/profiles/{id} {AgentProfileRequest}

    API->>AOP: @RateLimiter + @Authorized(AGENTPROFILES_EDIT)

    AOP->>Svc: updateAgentProfile(id, buId, tenantId, request)
    Svc->>DAO: findByIdAndBuId(id, buId)
    DAO->>AuroraRead: SELECT * FROM agent_profiles WHERE id = ? AND bus_id = ?
    AuroraRead-->>DAO: AgentProfiles entity

    alt Profile not found
        DAO-->>Svc: null / empty
        Svc-->>API: throw NotFoundException
        API-->>MFE: 404 Not Found
    else Profile found
        Svc->>Svc: Validate update request (name uniqueness check if name changed)
        Svc->>DAO: Check duplicate name (if name changed)
        DAO->>AuroraRead: SELECT COUNT(*) WHERE name = ? AND bus_id = ? AND id != ?
        AuroraRead-->>DAO: count

        alt Duplicate name
            Svc-->>API: throw ConflictException
            API-->>MFE: 409 Conflict
        else No conflict
            Svc->>DAO: Update profile entity fields
            DAO->>AuroraWrite: UPDATE agent_profiles SET name = ?, updated_at = NOW() WHERE id = ?
            Note over DAO,AuroraWrite: Also deletes + re-inserts AgentProfileConfiguration rows
            AuroraWrite-->>DAO: rows affected

            Svc->>CacheSvc: invalidateCache(buId, profileId)
            CacheSvc->>Valkey: DEL {buId}:* + DEL {buId}:{profileId}:*
            Valkey-->>CacheSvc: OK

            Svc-->>API: Return updated AgentProfileResponse
            API-->>MFE: 200 OK {updated profile}
            MFE-->>Admin: Show success toast
        end
    end
```

---

## 6. Update Profile Status (Bulk) Flow

Activate or deactivate multiple profiles in one call.

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Contact Center Admin
    participant MFE as Angular MFE<br/>(AgentProfilesComponent)
    participant API as Agent Profiles API
    participant AOP as AuthAspect
    participant Svc as AgentProfilesServiceImpl
    participant DAO as AgentProfilesDaoImpl
    participant AuroraWrite as Aurora MySQL Writer
    participant CacheSvc as CacheResponseServiceImpl
    participant Valkey as Valkey Cache

    Admin->>MFE: Selects profile(s) → clicks "Activate" or "Deactivate"
    MFE->>API: PUT /v1/profiles/status\n{profileIds: [id1, id2], status: "ACTIVE"|"INACTIVE", buId}

    API->>AOP: @RateLimiter + @Authorized(AGENTPROFILES_EDIT)

    AOP->>Svc: updateAgentProfileStatus(buId, statusChangeRequest)
    Svc->>DAO: findAllByIdsAndBuId(profileIds, buId)
    DAO->>AuroraWrite: SELECT * FROM agent_profiles WHERE id IN (?) AND bus_id = ?
    AuroraWrite-->>DAO: List<AgentProfiles>

    alt No profiles found or IDs do not belong to this BU
        Svc-->>API: Return SuccessResponseDTO(false)
        API-->>MFE: 304 Not Modified
        MFE-->>Admin: "No changes made"
    else Profiles found
        loop For each profile
            Svc->>Svc: Check if status already equals requested status
            alt Status already matches
                Note over Svc: Skip this profile
            else Status differs
                Svc->>DAO: updateStatus(id, newStatus, updatedAt)
                DAO->>AuroraWrite: UPDATE agent_profiles\nSET status = ?, updated_at = NOW()\nWHERE id = ?
                AuroraWrite-->>DAO: OK
            end
        end

        Svc->>CacheSvc: invalidateCache(buId)
        CacheSvc->>Valkey: DEL {buId}:*
        Valkey-->>CacheSvc: OK

        Svc-->>API: Return SuccessResponseDTO(true)
        API-->>MFE: 200 OK {success: true}
        MFE-->>Admin: Status badge updated in list
    end
```

---

## 7. Assign Teams to Profile Flow

Full lifecycle of assigning teams to an existing agent profile.

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Contact Center Admin
    participant MFE as Angular MFE<br/>(TeamsTabComponent)
    participant API as Agent Profiles API
    participant AOP as AuthAspect
    participant Svc as AgentProfilesServiceImpl
    participant PlatformSvc as PlatformApiConsumerServiceImpl
    participant CB as Circuit Breaker<br/>(userhub)
    participant PlatformAPI as CXone Platform API<br/>(/services/v31.0/teams)
    participant TeamsDAO as AgentProfileTeamsMapDaoImpl
    participant AuroraWrite as Aurora MySQL Writer
    participant CacheSvc as CacheResponseServiceImpl
    participant Valkey as Valkey Cache

    Admin->>MFE: Opens Teams tab on profile
    MFE->>API: GET /v1/profiles/{id}/teams
    Note over API: Returns currently assigned teams (cache-aside)
    API-->>MFE: 200 OK {assignedTeams: [...]}

    Admin->>MFE: Selects additional teams from picker
    MFE->>API: POST /v1/profiles/teams/search\n{searchTerm, buId}
    Note over API: Search teams via PlatformApiConsumerService

    API->>PlatformSvc: searchTeams(buId, searchTerm)
    PlatformSvc->>CB: Execute
    CB->>PlatformAPI: GET /services/v31.0/teams?searchTerm={term}
    PlatformAPI-->>CB: 200 OK {teams: [...]}
    CB-->>PlatformSvc: teams list
    PlatformSvc-->>API: teams list
    API-->>MFE: 200 OK {teams: [...]}
    MFE-->>Admin: Display team picker with search results

    Admin->>MFE: Confirms team selection → "Assign"
    MFE->>API: POST /v1/profiles/{id}/teams\n{teamIds: [t1, t2, t3], buId}

    API->>AOP: @RateLimiter + @Authorized(AGENTPROFILES_CREATE)
    AOP->>Svc: assignTeamsToAgentProfile(profileId, buId, teamIds)

    Svc->>Svc: Validate profileId exists and belongs to buId
    Svc->>TeamsDAO: getCurrentTeamIds(profileId)
    TeamsDAO->>AuroraWrite: SELECT team_id FROM agent_profile_teams_map WHERE profile_id = ?
    AuroraWrite-->>TeamsDAO: existing team IDs

    Svc->>Svc: Compute diff: new teams to add, existing teams to remove

    Svc->>TeamsDAO: deleteTeamsFromProfile(profileId, removedTeamIds)
    TeamsDAO->>AuroraWrite: DELETE FROM agent_profile_teams_map\nWHERE profile_id = ? AND team_id IN (?)
    AuroraWrite-->>TeamsDAO: OK

    Svc->>TeamsDAO: saveAll(newTeamMappings)
    TeamsDAO->>AuroraWrite: INSERT INTO agent_profile_teams_map\n(profile_id, team_id) VALUES (?, ?) × N
    AuroraWrite-->>TeamsDAO: OK

    Svc->>CacheSvc: invalidateCache(buId, profileId)
    CacheSvc->>Valkey: DEL {buId}:{profileId}:teams + {buId}:*
    Valkey-->>CacheSvc: OK

    Svc-->>API: Return updated team assignment response
    API-->>MFE: 200 OK {assignedTeams: [...updated...]}
    MFE-->>Admin: Teams tab refreshes with updated list
```

---

## 8. CXone Agent (CXA) Profile Fetch Flow

The lightweight profile fetch called by the CXone Agent desktop app at session start.

```mermaid
sequenceDiagram
    autonumber
    participant CXA as CXone Agent App<br/>(CXA Desktop)
    participant API as Agent Profiles API
    participant AOP as AuthAspect + RateLimiter
    participant CXAController as GetAgentProfileForCXAController (v1)<br/>GetAgentProfileCXAControllerV2 (v2)
    participant Svc as AgentProfilesServiceImpl
    participant CacheSvc as CacheResponseServiceImpl
    participant Valkey as Valkey Cache
    participant DAO as AgentProfilesDaoImpl
    participant TeamsDAO as AgentProfileTeamsMapDaoImpl
    participant Aurora as Aurora MySQL Reader

    Note over CXA: Agent logs into CXone desktop app
    CXA->>API: GET /agent-profiles/v1/cxa/profiles/{agentId}\n(or /cxa/v2/profiles/{agentId})\nAuthorization: Bearer {CXA JWT}

    API->>AOP: @RateLimiter(cxalimiter — 75 req/s)
    Note over AOP: CXA endpoints may have lighter auth requirements<br/>(dedicated CXA JWT claims path)
    AOP->>CXAController: Route to v1 or v2 controller

    CXAController->>Svc: getAgentProfileForCXA(agentId, buId)

    Svc->>CacheSvc: getCachedResponse("cxa:{buId}:{agentId}")
    CacheSvc->>Valkey: GET cxa:{buId}:{agentId}

    alt Cache HIT
        Valkey-->>CacheSvc: Cached CXA profile payload
        CacheSvc-->>Svc: Return payload
        Svc-->>CXAController: CXA profile response
        CXAController-->>CXA: 200 OK {profile, configurations}
        Note over CXA: Desktop configured with profile settings
    else Cache MISS
        Valkey-->>CacheSvc: nil
        Svc->>DAO: findActiveProfileForAgent(agentId, buId)
        Note over DAO: Looks up profile assigned to agent's team
        DAO->>TeamsDAO: getProfileIdByTeamId(agentTeamId)
        TeamsDAO->>Aurora: SELECT profile_id FROM agent_profile_teams_map\nWHERE team_id = ? LIMIT 1
        Aurora-->>TeamsDAO: profileId

        DAO->>Aurora: SELECT ap.*, apc.* FROM agent_profiles ap\nJOIN agent_profile_configuration apc ON apc.profile_id = ap.id\nWHERE ap.id = ? AND ap.status = 'ACTIVE'
        Aurora-->>DAO: Profile + configurations

        Svc->>Svc: Build GetAgentProfilePayloadDTO (v1/v2 format)
        Svc->>CacheSvc: cacheResponse("cxa:{buId}:{agentId}", payload)
        CacheSvc->>Valkey: SET cxa:{buId}:{agentId} {json} EX {ttl}

        Svc-->>CXAController: CXA profile payload
        CXAController-->>CXA: 200 OK {profile, configurations}
    end
```

---

## 9. Kinesis TeamData Event Processing Flow

How team lifecycle changes (rename, delete) propagate from the CXone platform into Agent Profiles.

```mermaid
sequenceDiagram
    autonumber
    participant PlatformTeam as CXone Platform<br/>(Team Service)
    participant Kinesis as Kinesis Stream<br/>(dev-TeamData)
    participant KCL as KinesisConsumer<br/>(KCL Worker — embedded in API)
    participant Processor as TeamDataRecordProcessor
    participant TeamEvents as TeamEvents (event parser)
    participant Svc as AgentProfilesServiceImpl
    participant TeamsDAO as AgentProfileTeamsMapDaoImpl
    participant AuroraWrite as Aurora MySQL Writer
    participant CacheSvc as CacheResponseServiceImpl
    participant Valkey as Valkey Cache

    Note over PlatformTeam: Admin renames or deletes a team in CXone
    PlatformTeam->>Kinesis: PutRecord({teamId, eventType: TEAM_RENAMED|TEAM_DELETED,\nnewName, buId, tenantId})

    loop KCL polling (continuous)
        KCL->>Kinesis: GetRecords (shard polling)
        Kinesis-->>KCL: List<KinesisRecord>
        KCL->>Processor: processRecords(records)

        loop For each record
            Processor->>TeamEvents: deserialize(record.getData())
            TeamEvents-->>Processor: TeamEvent{type, teamId, buId, tenantId, ...}

            alt eventType = TEAM_RENAMED
                Processor->>Svc: updateTeamNameInProfiles(teamId, newName, buId)
                Svc->>TeamsDAO: findProfilesByTeamId(teamId)
                TeamsDAO->>AuroraWrite: SELECT profile_id FROM agent_profile_teams_map WHERE team_id = ?
                AuroraWrite-->>TeamsDAO: List<profileId>

                loop For each affected profile
                    Svc->>AuroraWrite: UPDATE team metadata for profile
                end

                Svc->>CacheSvc: invalidateCacheForBU(buId)
                CacheSvc->>Valkey: DEL {buId}:*
                Valkey-->>CacheSvc: OK

            else eventType = TEAM_DELETED
                Processor->>Svc: removeTeamFromAllProfiles(teamId, buId)
                Svc->>TeamsDAO: deleteByTeamId(teamId)
                TeamsDAO->>AuroraWrite: DELETE FROM agent_profile_teams_map WHERE team_id = ?
                AuroraWrite-->>TeamsDAO: rows affected

                Svc->>CacheSvc: invalidateCacheForBU(buId)
                CacheSvc->>Valkey: DEL {buId}:*
                Valkey-->>CacheSvc: OK

            else eventType = TEAM_CREATED
                Note over Processor: No action needed — team assignments handled via REST API
            end

            Processor->>KCL: Checkpoint sequence number
            KCL->>Kinesis: Update shard checkpoint
        end
    end
```

**Key Points:**
- KCL workers run as background threads within the same Spring Boot process as the REST API
- Checkpoint after each batch prevents reprocessing on restart
- Cache invalidation on team events ensures profiles list reflects current team names

---

## 10. Kinesis User Management Event Processing Flow

How agent hire/terminate/role changes propagate from the CXone platform into Agent Profiles.

```mermaid
sequenceDiagram
    autonumber
    participant PlatformHR as CXone Platform<br/>(User/HR Service)
    participant Kinesis as Kinesis Stream<br/>(dev-userManagement)
    participant KCL as KinesisConsumer<br/>(KCL Worker — embedded in API)
    participant Processor as UserManagementRecordProcessor
    participant UserEvent as UserManagementEvent (parser)
    participant Svc as AgentProfilesServiceImpl
    participant DAO as AgentProfilesDaoImpl
    participant AuroraWrite as Aurora MySQL Writer
    participant CacheSvc as CacheResponseServiceImpl
    participant Valkey as Valkey Cache

    Note over PlatformHR: Agent onboarding, offboarding, or role change occurs
    PlatformHR->>Kinesis: PutRecord({userId, eventType: USER_HIRED|USER_TERMINATED|ROLE_CHANGED,\nagentId, buId, tenantId, teamId})

    loop KCL polling (continuous)
        KCL->>Kinesis: GetRecords (shard polling)
        Kinesis-->>KCL: List<KinesisRecord>
        KCL->>Processor: processRecords(records)

        loop For each record
            Processor->>UserEvent: deserialize(record.getData())
            UserEvent-->>Processor: UserManagementEvent{type, userId, agentId, buId, teamId}

            alt eventType = USER_HIRED / AGENT_CREATED
                Processor->>Svc: handleNewAgent(agentId, teamId, buId, tenantId)
                Note over Svc: Agent may inherit team's default profile
                Svc->>DAO: findDefaultProfileForTeam(teamId, buId)
                DAO->>AuroraWrite: Lookup default profile for the team
                AuroraWrite-->>DAO: profileId (if any)

                alt Default profile exists for team
                    Svc->>DAO: createAgentProfileAssociation(agentId, profileId)
                    DAO->>AuroraWrite: INSERT INTO agent_profile_assignments...
                end

            else eventType = USER_TERMINATED / AGENT_DELETED
                Processor->>Svc: handleAgentRemoved(agentId, buId, tenantId)
                Svc->>DAO: removeAgentFromProfiles(agentId)
                DAO->>AuroraWrite: DELETE agent profile associations for agentId

            else eventType = ROLE_CHANGED / TEAM_MOVED
                Processor->>Svc: handleAgentRoleOrTeamChange(agentId, newTeamId, buId)
                Svc->>DAO: reassignAgentProfile(agentId, newTeamId, buId)
                DAO->>AuroraWrite: UPDATE agent profile assignment to new team's default
            end

            Svc->>CacheSvc: invalidateCacheForAgent(buId, agentId)
            CacheSvc->>Valkey: DEL cxa:{buId}:{agentId}
            Valkey-->>CacheSvc: OK

            Processor->>KCL: Checkpoint sequence number
        end
    end
```

---

## 11. DynamoDB Feature Toggle Lookup Flow

How the API resolves feature flags at runtime from DynamoDB.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client Request
    participant API as Agent Profiles API
    participant Svc as AgentProfilesServiceImpl<br/>(or any service with feature check)
    participant PlatformSvc as PlatformApiConsumerServiceImpl
    participant DDB as AWS DynamoDB<br/>(Feature Toggle Table)

    Note over Client,API: During request processing, a code path checks a feature flag
    Client->>API: Any API request
    API->>Svc: Handle business operation

    Svc->>PlatformSvc: isFeatureEnabled(featureName, buId)
    PlatformSvc->>DDB: GetItem({TableName: feature-toggles, Key: {featureName: {S: "..."}}})\nvia AWS SDK (IAM role-based auth)

    alt DynamoDB returns item
        DDB-->>PlatformSvc: {featureName, enabled: true|false, scope: "BU"|"GLOBAL", metadata: {...}}
        PlatformSvc->>PlatformSvc: Check scope — global flag or BU-specific?
        alt Global flag AND enabled = true
            PlatformSvc-->>Svc: true
        else BU-specific AND buId matches AND enabled = true
            PlatformSvc-->>Svc: true
        else
            PlatformSvc-->>Svc: false
        end
    else Item not found (feature unknown)
        DDB-->>PlatformSvc: null / empty
        PlatformSvc-->>Svc: false (default disabled)
    end

    alt Feature enabled
        Svc->>Svc: Execute new feature code path
    else Feature disabled
        Svc->>Svc: Execute stable/legacy code path
    end

    Svc-->>API: Result
    API-->>Client: Response
```

---

## 12. Backend CI/CD Pipeline Flow

End-to-end from developer commit to production image.

```mermaid
flowchart TD
    subgraph "Developer Action"
        GIT_PUSH["git push / PR opened\nto master or 26.x branch"]
    end

    subgraph "GitHub Actions — BuildPushECR.yaml"
        direction TB
        CHECKOUT["1. actions/checkout@v4"]
        JDK["2. Setup JDK 17 (Zulu)"]
        
        subgraph "Integration Test Matrix  [dev | test | staging]"
            INT_DEV["./gradlew test\n-PactiveProfile=dev"]
            INT_TEST["./gradlew test\n-PactiveProfile=test"]
            INT_STAGING["./gradlew test\n-PactiveProfile=staging"]
        end
        
        JUNIT_UPLOAD["Upload JUnit XML + HTML\n(30-day retention)"]

        subgraph "Reusable Build Workflow  [acddevops-reusable-workflows]"
            SONAR["SonarQube SAST\n(enable-sonar-scan: true)"]
            SONAR_GATE{Quality Gate}
            BLACKDUCK["BlackDuck SCA\n(dependency scan)"]
            VERACODE["Veracode SAST\n(artifact: ms-agent-profiles-apis-1.0-SNAPSHOT)"]
            SEC_GATE{Security Gate}
            JIB["./gradlew build\n+ Jib container build\nImage tag: github.sha"]
            ECR["Push to ECR\n300813158921.dkr.ecr.us-west-2.amazonaws.com"]
        end
    end

    subgraph "Manual Promotion — PromoteImage.yaml"
        PROMOTE_INPUT["workflow_dispatch\nSelect: source env → target env\nSelect: image tag"]
        PROMOTE_APPROVE["⏸ GitHub Environment Approval\n(required for higher envs)"]
        ECR_PROMOTE["ECR: copy image tag\nsource account → target account"]
    end

    subgraph "Infrastructure Deploy — DeployInfra.yaml"
        INFRA_INPUT["workflow_dispatch\nSelect: stack name\nSelect: environment"]
        INFRA_APPROVE["⏸ GitHub Environment: approval_needed"]
        OIDC_AUTH["OIDC → AssumeRole\nServiceAccess-agent-profiles-service-deploy-role"]
        CFN_DEPLOY["aws-cloudformation-github-deploy\nUpdate selected stack\n(no-fail-on-empty-changeset)"]
        SSM_UPDATED["SSM params updated\n(endpoints, flags)"]
    end

    subgraph "Production Deploy — ProdDeployInfra.yaml + PromoteImageToProd.yaml"
        PROD_APPROVE["⏸ Production Approval Gate\n(separate workflow)"]
        PROD_ECR["Promote image: staging ECR → prod ECR"]
        PROD_CFN["Deploy prod CloudFormation stacks"]
    end

    GIT_PUSH --> CHECKOUT --> JDK
    JDK --> INT_DEV & INT_TEST & INT_STAGING
    INT_DEV & INT_TEST & INT_STAGING --> JUNIT_UPLOAD
    JUNIT_UPLOAD --> SONAR
    SONAR --> SONAR_GATE
    SONAR_GATE -->|Fail ❌| BLOCKED_SONAR["Build blocked\n(quality gate)"]
    SONAR_GATE -->|Pass ✅| BLACKDUCK --> VERACODE
    VERACODE --> SEC_GATE
    SEC_GATE -->|Fail ❌| BLOCKED_VER["Build blocked\n(security findings)"]
    SEC_GATE -->|Pass ✅| JIB --> ECR

    ECR --> PROMOTE_INPUT --> PROMOTE_APPROVE --> ECR_PROMOTE
    ECR --> INFRA_INPUT --> INFRA_APPROVE --> OIDC_AUTH --> CFN_DEPLOY --> SSM_UPDATED
    ECR_PROMOTE --> PROD_APPROVE --> PROD_ECR --> PROD_CFN

    style BLOCKED_SONAR fill:#d62728,color:#fff
    style BLOCKED_VER fill:#d62728,color:#fff
    style SONAR_GATE fill:#ff9900,color:#000
    style SEC_GATE fill:#ff9900,color:#000
    style INFRA_APPROVE fill:#1f77b4,color:#fff
    style PROMOTE_APPROVE fill:#1f77b4,color:#fff
    style PROD_APPROVE fill:#d62728,color:#fff
```

---

## 13. Infrastructure Provisioning (CloudFormation) Flow

How a new environment is fully provisioned using the 8 CloudFormation stacks, showing dependency order.

```mermaid
flowchart TD
    subgraph "Pre-requisites  [Platform-managed, imported via cross-stack refs]"
        VPC["CoreNetwork-Vpc\n(existing)"]
        AZ1["CoreNetwork-Az1CoreSubnet\n(existing)"]
        AZ2["CoreNetwork-Az2CoreSubnet\n(existing)"]
    end

    subgraph "Phase 1 — Secrets & Config  [No dependencies]"
        KMS["Stack 1: {env}-agentprofiles-kms-key\nTemplate: agent-profiles-kms-key-template.yaml\nOutputs: KmsKeyArn\nResources: KMS Key + Alias"]
        SECRETS["Stack 2: {env}-agentprofiles-secrets\nTemplate: agent-profiles-secret-template.yaml\nResources: SecretsManager secret\n{dbUsername, dbPassword}"]
        SSM_PARAMS["Stack 3: {env}-agentprofiles-ssm-params\nTemplate: agent-profiles-ssm-parameter-template.yaml\nResources: FIPS_ENABLED, ENCRYPTION_IN_TRANSIT_INTERNAL"]
    end

    subgraph "Phase 2 — Data Stores  [Depends on Phase 1 + VPC]"
        AURORA["Stack 4: {env}-agentprofiles-rds-aurora\nTemplate: agent-profiles-rds-aurora-template.yaml\nResources: AuroraCluster + 3 DBInstances\n+ DBSubnetGroup + SecurityGroup\nOutputs → SSM: writeEndpoint, readEndpoint"]
        VALKEY["Stack 5: {env}-agentprofiles-valkey-cluster\nTemplate: agent-profiles-valkey-cache-template.yaml\nResources: Valkey ReplicationGroup\n+ SubnetGroup + SecurityGroup\nOutputs → SSM: host, port"]
        DYNAMO["Stack 6: {env}-agentprofiles-dynamo-db\nTemplate: agent-profiles-dynamo-db-template.yaml\nResources: DynamoDB Table (feature toggles)"]
    end

    subgraph "Phase 3 — Observability & Management  [Depends on Phase 2]"
        CW["Stack 7: {env}-agentprofiles-cloudwatch-alarms\nTemplate: agent-profiles-cloudwatch-template.yaml\nResources: CloudWatch Alarms (RDS + Valkey)\n+ SNS Topic (KMS-encrypted)"]
        JUMP["Stack 8: {env}-agentprofiles-jump-server\nTemplate: agent-profiles-jump-server-template.yaml\nResources: EC2 Jump Server\n(ManagementCIDR access only)"]
    end

    subgraph "Phase 4 — Application  [Deployed separately via ECS platform]"
        ECS["ECS Task Definition + Service\n(Platform-managed, not in CFN repo)\nImage: 300813158921.dkr.ecr.us-west-2.amazonaws.com:SHA\nEnv vars: CX_DB_WRITE_ENDPOINT, CX_DB_READ_ENDPOINT\nCX_REDIS_HOST, CX_REDIS_PORT\nAP_TEAMDATASTREAM_ARN, AP_USERMANAGEMENTSTREAM_ARN"]
    end

    VPC & AZ1 & AZ2 --> AURORA
    VPC & AZ1 & AZ2 --> VALKEY
    SECRETS --> AURORA
    SSM_PARAMS --> AURORA
    SSM_PARAMS --> VALKEY
    KMS --> CW

    AURORA --> CW
    VALKEY --> CW
    AURORA --> JUMP
    VALKEY --> JUMP

    AURORA --> ECS
    VALKEY --> ECS
    DYNAMO --> ECS
    SECRETS --> ECS
    SSM_PARAMS --> ECS

    style ECS fill:#2ca02c,color:#fff
    style KMS fill:#d62728,color:#fff
    style SECRETS fill:#d62728,color:#fff
```

---

## 14. Error Handling & Resilience Flow

How the system handles and propagates errors at each layer.

```mermaid
flowchart TD
    subgraph "Rate Limit Layer"
        RL_CHECK{"Rate Limiter\n(cxalimiter)\n75 req/s"}
        RL_REJECT["429 Too Many Requests\nX-RateLimit-Remaining: 0"]
    end

    subgraph "Auth Layer"
        JWT_CHECK{"JWT Valid?\nSignature + Expiry"}
        JWKS_FAIL["401 Unauthorized\n{code: INVALID_TOKEN}"]
        PERM_CHECK{"User has\nrequired permission?"}
        PERM_FAIL["403 Forbidden\n{code: INSUFFICIENT_PERMISSIONS}"]
        CB_CHECK{"Circuit Breaker\n(userhub) open?"}
        CB_FAIL["503 Service Unavailable\n{code: PLATFORM_UNAVAILABLE}"]
    end

    subgraph "Business Logic Layer"
        VALIDATION{"Request\nvalidation OK?"}
        VAL_FAIL["400 Bad Request\n{code: VALIDATION_ERROR,\ndetails: [field errors]}"]
        NOT_FOUND{"Resource\nexists?"}
        NF_FAIL["404 Not Found\n{code: RESOURCE_NOT_FOUND}"]
        CONFLICT{"Name\nunique?"}
        CONFLICT_FAIL["409 Conflict\n{code: DUPLICATE_PROFILE_NAME}"]
    end

    subgraph "Database Layer"
        DB_TRY["JDBC / JPA operation\non Aurora MySQL"]
        DB_TIMEOUT{"Connection\ntimeout?"}
        DB_TO_FAIL["504 Gateway Timeout\n(HikariCP pool exhausted)"]
        DB_RETRY["HikariCP keepalive\n(SELECT 1 every 5min)\nAuto-reconnect on idle drop"]
    end

    subgraph "Kinesis Layer"
        KCL_RETRY["KCL automatic retry\non processRecords failure"]
        KCL_CHECKPOINT["Checkpoint only on success\n(prevents data loss on crash)"]
        KCL_DLQ["Poison pill handling:\nLog + skip after N retries"]
    end

    subgraph "External API Layer"
        PLATFORM_CALL["Call CXone Platform API"]
        TIMEOUT_CHECK{"Timeout /\n5xx?"}
        CB_OPEN["Circuit Breaker\nopens after 50% failures\n(10s wait duration)"]
        CB_HALF_OPEN["HALF-OPEN: probe\n1 test call allowed"]
        CB_CLOSE["Circuit Breaker\ncloses on success"]
    end

    subgraph "Observability"
        LOG["Logback MDC log\n(correlationId, buId, tenantId)"]
        METRICS["Prometheus counter\n+ OTLP trace span"]
        CW_ALARM["CloudWatch Alarm\n→ SNS notification"]
    end

    Request["Incoming API Request"] --> RL_CHECK
    RL_CHECK -->|Over limit| RL_REJECT
    RL_CHECK -->|OK| JWT_CHECK
    JWT_CHECK -->|Invalid| JWKS_FAIL
    JWT_CHECK -->|Valid| CB_CHECK
    CB_CHECK -->|OPEN| CB_FAIL
    CB_CHECK -->|CLOSED| PERM_CHECK
    PERM_CHECK -->|Denied| PERM_FAIL
    PERM_CHECK -->|Granted| VALIDATION
    VALIDATION -->|Invalid| VAL_FAIL
    VALIDATION -->|Valid| NOT_FOUND
    NOT_FOUND -->|Not found| NF_FAIL
    NOT_FOUND -->|Found| CONFLICT
    CONFLICT -->|Conflict| CONFLICT_FAIL
    CONFLICT -->|OK| DB_TRY
    DB_TRY --> DB_TIMEOUT
    DB_TIMEOUT -->|Timeout| DB_TO_FAIL
    DB_TIMEOUT -->|OK| DB_RETRY
    DB_RETRY --> Success["200/201 Response"]

    PLATFORM_CALL --> TIMEOUT_CHECK
    TIMEOUT_CHECK -->|Fail| CB_OPEN --> CB_HALF_OPEN --> CB_CLOSE
    TIMEOUT_CHECK -->|OK| CB_CLOSE

    VAL_FAIL & NF_FAIL & CONFLICT_FAIL & JWKS_FAIL & PERM_FAIL & CB_FAIL & DB_TO_FAIL --> LOG
    LOG --> METRICS
    DB_TO_FAIL --> CW_ALARM

    KCL_RETRY --> KCL_CHECKPOINT
    KCL_RETRY -->|After N retries| KCL_DLQ

    style RL_REJECT fill:#ff9900,color:#000
    style JWKS_FAIL fill:#d62728,color:#fff
    style PERM_FAIL fill:#d62728,color:#fff
    style CB_FAIL fill:#d62728,color:#fff
    style VAL_FAIL fill:#ff9900,color:#000
    style NF_FAIL fill:#ff9900,color:#000
    style CONFLICT_FAIL fill:#ff9900,color:#000
    style DB_TO_FAIL fill:#d62728,color:#fff
    style Success fill:#2ca02c,color:#fff
```

---

## 15. Frontend Bootstrap & MFE Load Flow

How the Angular micro-frontend is loaded, initialized, and connected to the backend when an admin navigates to Agent Profiles in CXone.

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Contact Center Admin
    participant Browser as Browser
    participant Shell as CXone Shell App<br/>(Host MFE Container)
    participant CF as CloudFront / S3<br/>(MFE bundle CDN)
    participant ModFed as Webpack Module Federation
    participant AppModule as AppModule (Angular bootstrap)
    participant AppInit as APP_INITIALIZER chain
    participant LocalInit as LocalizationInitializer<br/>(i18next / @niceltd/sol/translation)
    participant WebInit as WebAppInitializerService<br/>(cxone-mfe-tools)
    participant ConfigSvc as ConfigurationService<br/>(@niceltd/cxone-client-platform-services)
    participant Router as Angular Router
    participant ProfilesComp as AgentProfilesComponent
    participant ProfileSvc as ProfileService
    participant Interceptor as CXOneHttpRequestInterceptor
    participant API as Agent Profiles API

    Admin->>Browser: Navigate to Agent Profiles section in CXone
    Browser->>Shell: Load CXone Shell (already cached / hydrated)
    Shell->>Shell: Route matches /agent-profiles → MFE zone
    Shell->>ModFed: Resolve remote: agent-profile-mfe (webpack Module Federation)
    ModFed->>CF: GET /remoteEntry.js (Module Federation manifest)
    CF-->>ModFed: remoteEntry.js (chunk hashes, module map)
    ModFed->>CF: GET /main.{hash}.js, /polyfills.{hash}.js, /styles.{hash}.css
    CF-->>ModFed: Angular MFE bundle

    ModFed->>AppModule: Bootstrap Angular application
    AppModule->>AppModule: Register Web Component: customElements.define("agent-profile-app", ...)
    AppModule->>AppModule: Register HTTP interceptors:\n- CXOneHttpRequestInterceptor\n- CXOneHttpResponseInterceptor\n- AppSpinnerInterceptor

    AppModule->>AppInit: Run APP_INITIALIZER chain (in order)
    AppInit->>LocalInit: Initialize i18next localization
    LocalInit->>CF: GET /assets/i18n/{locale}.json
    CF-->>LocalInit: Locale strings
    LocalInit-->>AppInit: Localization ready

    AppInit->>ConfigSvc: Load platform configuration (tenant, env URLs)
    ConfigSvc->>Shell: Read config injected by shell (postMessage / shared context)
    Shell-->>ConfigSvc: {tenantId, buId, platformBaseUrl, environment}
    ConfigSvc-->>AppInit: Config ready

    AppInit->>WebInit: WebAppInitializerService.initialize()
    WebInit->>Shell: Set MFE navigation flags (hide CXone nav chrome)
    WebInit->>Router: Activate mfeActivateGuard
    WebInit-->>AppInit: MFE context ready

    AppInit-->>AppModule: All initializers complete
    AppModule->>Router: Bootstrap router at /agent-profiles

    Router->>ProfilesComp: Lazy-load AgentProfilesComponent (route match)
    ProfilesComp->>ProfileSvc: getProfilesData(defaultFilter, defaultPagination)
    ProfileSvc->>Interceptor: POST /agent-profiles/v1/profiles/search
    Interceptor->>Interceptor: Inject Authorization: Bearer {token from ConfigurationService}\nInject X-Tenant-Id: {tenantId}\nInject X-BU-Id: {buId}
    Interceptor->>API: POST /agent-profiles/v1/profiles/search\n{buId, status: null, skip: 0, top: 25}

    API-->>Interceptor: 200 OK {profiles: [...], total: N}
    Interceptor->>ProfileSvc: AgentProfilesResponse
    ProfileSvc->>ProfileSvc: Map response to component view models
    ProfileSvc-->>ProfilesComp: profiles list data
    ProfilesComp->>Browser: Render profiles table
    Browser-->>Admin: Agent Profiles list visible and interactive
```

---

## Flow Summary

| # | Flow | Trigger | Key Components | Latency Profile |
|---|------|---------|----------------|-----------------|
| 1 | JWT Validation | Every API request | AuthAspect → AuthService → Caffeine L1 → Persistent L2 → JWKS | Fast (cache hits ~1ms; miss ~100ms) |
| 2 | RBAC Permission Check | Every @Authorized request | AuthAspect → PlatformSvc → Circuit Breaker → Platform API | ~50–200ms (network call) |
| 3 | Get Profiles List | Admin UI load | ProfileService → Cache → Aurora Read | ~5ms (cache hit) / ~50ms (DB miss) |
| 4 | Create Profile | Admin form submit | Controller → Service → DB Write → Cache Invalidate | ~100–200ms |
| 5 | Update Profile | Admin edit save | Controller → Service → DB Read+Write → Cache Invalidate | ~100–200ms |
| 6 | Update Status (Bulk) | Admin status toggle | Controller → Service → DB Write Loop → Cache Invalidate | ~50–500ms (depends on batch size) |
| 7 | Assign Teams | Admin teams tab | Controller → Service → Platform API → DB Write → Cache Invalidate | ~200–500ms |
| 8 | CXA Profile Fetch | Agent desktop login | CXAController → Cache → DB Read | ~5ms (cache hit) / ~50ms (DB miss) |
| 9 | Kinesis TeamData | Team lifecycle event | KCL → TeamDataRecordProcessor → DB Write → Cache Invalidate | Async (~seconds from event publish) |
| 10 | Kinesis UserMgmt | Agent lifecycle event | KCL → UserMgmtRecordProcessor → DB Write → Cache Invalidate | Async (~seconds from event publish) |
| 11 | Feature Toggle | Business logic branch | Service → DynamoDB GetItem | ~5–20ms |
| 12 | Backend CI/CD | git push / PR | GitHub Actions → Gradle → Sonar → Veracode → Jib → ECR | ~15–30 min total |
| 13 | Infra Provisioning | Manual workflow_dispatch | GitHub Actions → OIDC → CloudFormation (8 stacks) | ~20–40 min total |
| 14 | Error Handling | Any error path | Resilience4j → Error response → Logback MDC + OTLP | Inline with request |
| 15 | MFE Bootstrap | Admin navigates to Agent Profiles | Shell → Module Federation → Angular Bootstrap → API | ~1–3s initial load |
