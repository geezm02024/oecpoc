# ISO 27001:2022 Technical Audit Report
## HRMS People Portal - Microservices Platform

**Audit Date:** 2026-03-11
**Auditor:** Technical Audit Orchestrator (Automated)
**System Version:** 1.0.x
**Framework:** ISO 27001:2022 Annex A (Technological Controls)

---

## Executive Summary

This technical audit assesses the HRMS People Portal microservices platform against ISO 27001:2022 Annex A technological controls. The system comprises 5 .NET 9 microservices (People, Checklist, Notification, Engagement, ATS) with a React 18 TypeScript frontend, deployed on Azure Container Apps with PostgreSQL 16 and Azure Event Hubs for event-driven communication.

### Overall Security Risk Level: 🟠 **MEDIUM-HIGH**

**Key Strengths:**
- Comprehensive JWT authentication with Azure AD OIDC integration
- Non-root Docker containers with health checks
- Rate limiting and OWASP security headers implemented
- Clean Architecture with CQRS pattern
- TypeScript strict mode enabled on frontend
- EF Core 9 for parameterized queries preventing SQL injection

**Critical Gaps:**
- Trivy container scanning disabled in CI/CD pipeline (`condition: false`)
- Hardcoded encryption key default in docker-compose.yml
- Secrets exposed in environment variables (SMTP password, Azure AD client secrets)
- Missing SAST/SCA in build pipeline
- No centralized logging or SIEM integration observed
- Incomplete vulnerability management process

---

## 1. System Overview

### 1.1 Architecture Pattern

**Pattern:** Microservices
**Communication:** Mixed (Synchronous REST API + Asynchronous Azure Event Hubs)
**Deployment:** Azure Container Apps
**Database:** PostgreSQL 16 (shared database with schema-per-service pattern)
**Caching:** Redis (StackExchange.Redis)
**Frontend:** React 18 + TypeScript + Vite 5

### 1.2 Services Inventory

| Service | Responsibility | Tech Stack | Port | Test Coverage |
|---------|---------------|-----------|------|---------------|
| People Service | Employee data, profiles, GDPR | .NET 9 / EF Core 9 / PostgreSQL | 5001 | Unit + Integration + Security tests |
| Checklist Service | Onboarding/offboarding workflows | .NET 9 / EF Core 9 / Event Hubs | 5003 | Unit + Integration tests |
| Notification Service | Email notifications (SMTP) | .NET 9 / MailKit / Event Hubs | 5005 | Unit + Integration tests |
| Engagement Service | Employee engagement tracking | .NET 9 / EF Core 9 / Redis | 5006 | Unit + Integration tests |
| ATS Service | Applicant Tracking System | .NET 9 / EF Core 9 | 5007 | Unit + Integration tests |
| HRMS SPA | Frontend web application | React 18 / TS 5.6 / Vite 5 | 3000 | Vitest |

### 1.3 Architecture Diagram (Textual)

```
[SPA: React 18 + TypeScript + OIDC-client-ts]
       │ HTTPS + Azure AD OIDC
       ▼
[Azure Container Apps / Azure API Management (implied)]
       │
  ┌────┴────────────────────┐
  │         │               │
[People] [Checklist] [Notification] [Engagement] [ATS]
  │         │               │
  │    [Azure Event Hubs]   │
  │         │               │
  └────┬────┴───────────────┘
[PostgreSQL 16 (shared DB, schema isolation)]
[Redis Cache (engagement-service)]
[Azure Blob Storage (profile photos)]
[Azure Key Vault (ats-service only)]
```

---

## 2. Code Quality Analysis

### 2.1 Backend Code Quality

**Stack:** .NET 9 / ASP.NET Core / C# 12
**Analysis Tools Recommended:** SonarCloud, dotnet-format, Roslyn analyzers, NDepend
**Actual Tools Observed:** NuGet vulnerability scanning (disabled in audit), xUnit, FluentAssertions, Moq

#### Findings

| ID | Severity | Area | Finding | Recommendation |
|----|----------|------|---------|----------------|
| CQ-B01 | 🟡 MEDIUM | Project Config | `TreatWarningsAsErrors` set to `true` in PeopleService.Api.csproj, but not consistently across all services | Enable `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in all .csproj files (Directory.Build.props) |
| CQ-B02 | 🟢 LOW | Nullable | `<Nullable>enable</Nullable>` enabled consistently across services | Excellent - maintain this standard |
| CQ-B03 | 🟢 LOW | Architecture | Clean Architecture with Application/Domain/Infrastructure separation observed | Strong separation of concerns maintained |
| CQ-B04 | 🟡 MEDIUM | Logging | Excessive logging in JWT authentication events (OnMessageReceived logs sensitive auth headers) | Remove or redact token prefix logging in production (Program.cs:120-122) |
| CQ-B05 | 🟢 LOW | DI Pattern | Proper use of dependency injection with `AddApplication()` and `AddInfrastructure()` extension methods | Well-structured DI registration |
| CQ-B06 | 🟡 MEDIUM | Error Handling | Global exception handler middleware implemented, but need to verify PII scrubbing in logs | Audit exception logging for PII exposure risk |
| CQ-B07 | 🔵 INFO | CQRS | MediatR 12.4.1 used with FluentValidation 11.11.0 for command/query separation | Modern CQRS implementation |
| CQ-B08 | 🟡 MEDIUM | EF Core | Migrations applied automatically on startup (`dbContext.Database.MigrateAsync()`) | Consider migration separation for production - auto-migration can cause deployment race conditions |

#### Metrics Summary

| Metric | Observed / Estimated | Target | Status |
|--------|---------------------|--------|--------|
| Code coverage | Unknown (no reports) | ≥80% | ⚠️ |
| Duplication | Low (estimated) | <3% | 🟢 |
| Cyclomatic complexity (avg) | Low-Medium (clean architecture) | <10 | 🟢 |
| Technical debt ratio | Unknown | <5% | ⚠️ |
| Security hotspots | 4 identified | 0 | 🔴 |

#### Key Observations

**Strengths:**
- Excellent use of nullable reference types (`<Nullable>enable</Nullable>`) prevents null reference exceptions
- Clean Architecture properly implemented with Application/Domain/Infrastructure layers
- CQRS pattern with MediatR provides strong separation of read/write operations
- FluentValidation used consistently for input validation
- Proper async/await usage observed in Program.cs
- JWT validation parameters comprehensive (ValidateIssuer, ValidateAudience, ValidateLifetime, ValidateIssuerSigningKey all true)
- Rate limiting implemented with multiple policies (GDPR-sensitive, Standard, Bulk, Auth)
- OWASP security headers properly configured (CSP, X-Frame-Options, X-Content-Type-Options, HSTS)
- Non-blocking health checks properly configured for Container Apps

**Concerns:**
- No evidence of automated code quality gates in CI/CD (SonarCloud, CodeQL, or similar)
- Test coverage metrics not published in pipeline
- Auto-migration on startup (line 260 in Program.cs) can cause race conditions in scaled environments
- JWT event logging may expose sensitive claims in logs (line 103 logs all claims)
- Shared library `HRMS.Shared.Authorization` creates cross-service coupling (all services reference it)

---

### 2.2 Frontend Code Quality

**Stack:** React 18.3.1 / TypeScript 5.6.2 / Vite 5.4.10
**Analysis Tools Recommended:** ESLint 9, typescript-eslint, Prettier, Knip (dead code)
**Actual Tools Observed:** ESLint 9.13.0, typescript-eslint 8.11.0, Vitest 2.1.4

#### Findings

| ID | Severity | Area | Finding | Recommendation |
|----|----------|------|---------|----------------|
| CQ-F01 | 🟢 LOW | TypeScript | `strict: true` enabled in tsconfig.json with comprehensive strict options | Excellent type safety configuration |
| CQ-F02 | 🟢 LOW | ESLint | `@typescript-eslint/no-explicit-any: 'error'` enforced, preventing `any` usage | Strong type enforcement |
| CQ-F03 | 🟢 LOW | ESLint | React Hooks rules enabled via `eslint-plugin-react-hooks` | Prevents common React pitfalls |
| CQ-F04 | 🟡 MEDIUM | Build | Missing Prettier configuration - inconsistent formatting risk | Add Prettier with ESLint integration |
| CQ-F05 | 🟡 MEDIUM | Testing | Vitest configured but coverage target unknown | Add `vitest.config.ts` with coverage thresholds (≥80%) |
| CQ-F06 | 🟠 HIGH | Security | OIDC configuration passed as build args in Dockerfile (lines 206-212 in docker-compose.yml) | Environment-specific OIDC config should use runtime env vars, not build args |
| CQ-F07 | 🔵 INFO | Dependencies | Modern React Query (@tanstack/react-query 5.59.0) for data fetching | Good choice for caching and state management |
| CQ-F08 | 🟡 MEDIUM | Bundle | No evidence of bundle size analysis or code splitting strategy | Add `vite-plugin-bundle-analyzer` and implement lazy loading |

#### Metrics Summary

| Metric | Observed / Estimated | Target | Status |
|--------|---------------------|--------|--------|
| TypeScript strict mode | Yes | Yes | 🟢 |
| ESLint errors | Unknown (not run in audit) | 0 | ⚠️ |
| Bundle size (initial) | Unknown | <200 KB | ⚠️ |
| Lighthouse score | Unknown | ≥90 | ⚠️ |

#### Key Observations

**Strengths:**
- TypeScript strict mode enabled with `noUnusedLocals`, `noUnusedParameters`, `noFallthroughCasesInSwitch`, and `noUncheckedIndexedAccess`
- `@typescript-eslint/no-explicit-any` set to `error` prevents type safety erosion
- Modern tooling: Vite 5 for fast builds, Vitest for testing, React Query for server state
- OIDC-client-ts 3.0.1 used for OpenID Connect authentication (industry standard)
- Path alias configured (`@/*` → `src/*`) for clean imports

**Concerns:**
- No Prettier configuration found - formatting consistency relies on developer discipline
- Build args used for OIDC config in Dockerfile (immutable at build time, requires rebuild for environment changes)
- No evidence of bundle size monitoring or lazy loading of routes
- No accessibility (a11y) linting configured (consider eslint-plugin-jsx-a11y)
- Vitest coverage configuration not observed

---

## 3. Architecture Review

### 3.1 Architectural Findings

| ID | Severity | Area | Finding | Recommendation |
|----|----------|------|---------|----------------|
| AR-01 | 🟠 HIGH | Data Isolation | All services share a single PostgreSQL database (schema isolation only) | Migrate to database-per-service for true bounded context isolation |
| AR-02 | 🟡 MEDIUM | Coupling | `HRMS.Shared.Authorization` library shared across all services creates deployment coupling | Consider duplicating authorization logic or use shared contracts only |
| AR-03 | 🟢 LOW | Communication | Proper async event-driven architecture with Azure Event Hubs for cross-service events | Well-designed event-driven communication |
| AR-04 | 🟡 MEDIUM | API Gateway | No explicit API Gateway or reverse proxy configuration observed in infrastructure | Deploy Azure API Management or YARP for centralized auth, rate limiting, and routing |
| AR-05 | 🔵 INFO | Service Discovery | Azure Container Apps provides built-in service discovery via internal DNS | Appropriate for this platform |
| AR-06 | 🟡 MEDIUM | Event Outbox | No evidence of transactional outbox pattern for guaranteed event publishing | Implement outbox pattern to prevent event loss on service crashes |
| AR-07 | 🟢 LOW | Health Checks | ASP.NET Core health checks configured for liveness and readiness probes | Proper cloud-native health monitoring |

### 3.2 Service Design Assessment

| Service | Responsibility | Coupling | Data Isolation | Verdict |
|---------|---------------|---------|----------------|---------|
| People Service | Single (Employee data + GDPR) | Medium (shared auth library) | Shared DB (own schema) | ⚠️ |
| Checklist Service | Single (Workflows) | Medium (shared auth library) | Shared DB (own schema) | ⚠️ |
| Notification Service | Single (Email delivery) | Medium (shared auth library) | Shared DB (own schema) | ⚠️ |
| Engagement Service | Single (Engagement tracking) | Medium (shared auth library) | Shared DB (own schema) | ⚠️ |
| ATS Service | Single (Applicant tracking) | Medium (shared auth library) | Shared DB (own schema) | ⚠️ |

**Assessment:** Services have clear single responsibilities but are hindered by shared database and shared authorization library. Schema isolation provides some data isolation but is weaker than database-per-service.

### 3.3 Scalability Analysis

**Current scaling model:** Horizontal (Container Apps auto-scaling supported)

| Scenario | Current Capability | Recommended Approach | Effort |
|----------|-------------------|---------------------|--------|
| 2× user load | ✅ Should handle (stateless APIs, horizontal scaling) | Enable auto-scaling rules in Container Apps | S |
| 10× user load | ⚠️ Shared PostgreSQL may bottleneck | Add read replicas, connection pooling (PgBouncer) | M |
| Multi-tenant | ⚠️ Tenant isolation via claims, but shared DB | Implement schema-per-tenant or database-per-tenant | XL |

### 3.4 Resilience & Fault Tolerance

| Concern | Present? | Quality | Notes |
|---------|---------|---------|-------|
| Circuit breakers | No | - | Consider Polly library for HTTP resilience |
| Retry policies (Polly) | No | - | Critical for Azure Event Hubs and inter-service HTTP calls |
| Health checks | Yes | ✅ | Liveness and readiness checks implemented |
| Graceful degradation | Partial | ⚠️ | Event Hub consumer gracefully disabled if not configured |
| Distributed tracing | No | - | Implement OpenTelemetry or Application Insights correlation |
| Dead-letter queues | Unknown | - | Verify Event Hubs dead-letter config for poisoned messages |

### 3.5 Environment Architecture

**Local dev:** Docker Compose with PostgreSQL, Azurite
**Cloud:** Azure Container Apps, Azure Event Hubs, Azure Blob Storage, Azure AD

| Layer | Technology | Assessment | Risk |
|-------|-----------|-----------|------|
| Compute | Azure Container Apps | ✅ Serverless containers with auto-scaling | Low |
| Database | PostgreSQL 16 (Azure Database for PostgreSQL implied) | ✅ Managed, but shared across services | Medium (single point of failure) |
| Cache | Redis (StackExchange.Redis) | ✅ Used in Engagement Service | Low |
| Messaging | Azure Event Hubs | ✅ Event-driven architecture | Low |
| Storage | Azure Blob Storage | ✅ Used for profile photos (People Service) | Low |
| Auth | Azure AD (Entra ID) | ✅ Enterprise-grade identity | Low |
| Secrets | Azure Key Vault | ⚠️ Only ATS Service uses Key Vault | High (secrets in env vars for other services) |

---

## 4. Security Evaluation

### 4.1 Security Posture Summary

The HRMS People Portal demonstrates strong authentication foundations with Azure AD OIDC and comprehensive JWT validation. However, critical gaps exist in secrets management (hardcoded defaults, environment variable exposure) and vulnerability management (disabled Trivy scans, no SAST/SCA in pipeline). The absence of centralized logging and SIEM integration limits security monitoring and incident response capabilities. Rate limiting and OWASP headers provide defense-in-depth, but the overall posture is weakened by operational security gaps.

**Overall Security Risk Level:** 🟠 **MEDIUM-HIGH**

### 4.2 Authentication & Authorization

| Control | Implementation | Assessment | ISO 27001 Ref |
|---------|--------------|-----------|---------------|
| Authentication mechanism | JWT + Azure AD OIDC | ✅ | A.8.5 |
| MFA enforcement | Azure AD Conditional Access (external) | ⚠️ Partial (depends on Azure AD policy) | A.8.5 |
| Token expiry & refresh | ValidateLifetime=true, ClockSkew=1min | ✅ | A.8.5 |
| Role-based access control | Custom permission-based authorization (PermissionAuthorizationHandler) | ✅ | A.8.2 |
| Principle of least privilege | Permission-based policies per endpoint | ✅ | A.8.2 |
| Service-to-service auth | Inter-service HTTP calls use service identity (implied) | ⚠️ Not explicitly validated in audit | A.8.3 |

#### Findings

| ID | Severity | OWASP | Finding | Recommendation | ISO 27001 |
|----|----------|-------|---------|----------------|-----------|
| SEC-A01 | 🟡 MEDIUM | A07 | JWT claims logging may expose sensitive user data (Program.cs:103) | Remove or redact claims logging in production | A.8.15 |
| SEC-A02 | 🟢 LOW | A07 | JWT validation comprehensive (ValidateIssuer, ValidateAudience, ValidateLifetime, ValidateIssuerSigningKey all true) | Maintain current configuration | A.8.5 |
| SEC-A03 | 🟡 MEDIUM | A01 | `LoginEnabledHandler` checks if user login is enabled, but no audit logging on denial | Add security event logging for denied login attempts | A.8.15 |

### 4.3 Secrets & Configuration Management

| Item | Current State | Risk | Recommendation |
|------|--------------|------|----------------|
| Secrets in source code | Not observed in audited files | ✅ | Continue static secret scanning with TruffleHog or GitGuardian |
| Azure Key Vault usage | Only ATS Service | 🔴 HIGH | Migrate all secrets to Key Vault with Managed Identity |
| Environment variable handling | Secrets in docker-compose.yml env vars | 🔴 HIGH | Use Azure Key Vault references, not plaintext env vars |
| Docker image secrets | Build args used for OIDC config (frontend) | 🟠 HIGH | Use runtime env vars instead of build args |
| CI/CD secrets handling | Stored in Azure DevOps variable groups | ✅ | Ensure variable groups are marked as secret |

#### Findings

| ID | Severity | Finding | Recommendation | ISO 27001 |
|----|----------|---------|----------------|-----------|
| SEC-S01 | 🔴 CRITICAL | Default encryption key hardcoded in docker-compose.yml (line 51): `Encryption__Key: ${ENCRYPTION_KEY:-0123456789abcdef...}` | Remove default value, fail fast if not provided | A.8.12 |
| SEC-S02 | 🔴 CRITICAL | SMTP credentials in plaintext environment variables (lines 123-130 in docker-compose.yml) | Migrate to Azure Key Vault, use Managed Identity | A.8.12 |
| SEC-S03 | 🔴 CRITICAL | Azure AD client secrets in environment variables (lines 46, 79, 113, 154, 181 in docker-compose.yml) | Use Azure Key Vault for client secrets | A.8.12 |
| SEC-S04 | 🟠 HIGH | Azurite connection string hardcoded with well-known dev account key (line 48) | Acceptable for local dev, but ensure never used in production | A.8.12 |

### 4.4 Transport Security

| Layer | Protocol | Certificate | HSTS | Assessment |
|-------|---------|------------|------|-----------|
| SPA ↔ API | HTTPS/TLS 1.2+ | Azure Container Apps managed cert | Enabled (2 years) | ✅ |
| Service ↔ Azure | TLS 1.2+ | Azure managed | N/A | ✅ |
| DB connections | PostgreSQL SSL/TLS | Azure PostgreSQL enforced SSL | N/A | ✅ (assumed) |
| Redis connections | TLS | StackExchange.Redis supports TLS | ⚠️ Not verified in config | Unknown |

### 4.5 Input Validation & Injection

| Area | Mechanism | Assessment | OWASP |
|------|----------|-----------|-------|
| API input validation | FluentValidation (Application layer) | ✅ | A03 |
| SQL injection (via EF Core) | Parameterized queries (no raw SQL observed) | ✅ | A03 |
| XSS (frontend) | React auto-escaping | ✅ | A03 |
| CSRF protection | SameSite=Strict cookies, CORS origin whitelist | ✅ | A01 |
| File upload validation | Profile photo upload (Azure Blob Storage) | ⚠️ Validation logic not audited | A04 |

#### Findings

| ID | Severity | OWASP | Finding | Recommendation | ISO 27001 |
|----|----------|-------|---------|----------------|-----------|
| SEC-I01 | 🟢 LOW | A03 | EF Core used consistently, no raw SQL observed in Program.cs | Maintain parameterized query discipline | A.8.28 |
| SEC-I02 | 🟡 MEDIUM | A04 | File upload validation for profile photos not audited | Verify content-type validation, file size limits, and virus scanning | A.8.28 |

### 4.6 Dependency Security

| Ecosystem | Total Deps | Critical CVEs | High CVEs | Scan Tool |
|-----------|-----------|--------------|-----------|-----------|
| NuGet (.NET) | ~30 per service | Unknown | Unknown | `dotnet list package --vulnerable` (not run) |
| npm (frontend) | 39 direct | Unknown | Unknown | `npm audit` (requires node_modules) |
| Docker base images | 2 (SDK 9.0, ASP.NET 9.0) | Unknown | Unknown | Trivy (disabled in pipeline) |

**Most critical vulnerable packages:**

⚠️ **Cannot assess** - vulnerability scanning not executed due to missing `node_modules` and `obj`/`bin` directories. Pipeline indicates Trivy scans are present but disabled (`condition: false` on lines 498, 565, 631, 697, 763, 829).

**Recommendation:** Enable Trivy scans in azure-pipelines.yml and add NuGet/npm vulnerability gates to CI/CD.

### 4.7 Logging & Monitoring (Security Events)

| Requirement | Present? | Quality | ISO 27001 |
|-------------|---------|---------|-----------|
| Auth failure logging | Yes | ⚠️ Logs sensitive data (auth headers) | A.8.15 |
| Privileged action audit trail | Unknown | - | A.8.15 |
| Log tampering protection | No | - | A.8.15 |
| Centralized log aggregation | Unknown | ⚠️ Not observed in code | A.8.16 |
| Alerting on anomalies | No | - | A.8.16 |
| Log retention policy | Unknown | - | A.8.15 |
| SIEM integration | No | - | A.8.16 |

**Critical Gap:** No evidence of centralized logging (Azure Log Analytics, Application Insights, ELK) or SIEM integration. Logging appears to be local/console only.

### 4.8 Data Protection & Privacy

| Area | Assessment | ISO 27001 | GDPR Relevance |
|------|-----------|-----------|---------------|
| PII identification & classification | ⚠️ Not formally documented | A.8.12 | Art. 4 |
| Encryption at rest | ✅ Azure Database for PostgreSQL, Blob Storage (assumed) | A.8.24 | Art. 32 |
| Encryption in transit | ✅ HTTPS enforced, Azure enforces TLS | A.8.24 | Art. 32 |
| Data retention & deletion | ⚠️ GDPR deletion event observed (person.gdpr-deleted), but policy unknown | A.8.10 | Art. 17 |
| Backup encryption | ✅ Azure managed backups encrypted | A.8.13 | Art. 32 |

### 4.9 Security Findings Summary

| ID | Severity | Category | Title | Effort |
|----|----------|---------|-------|--------|
| SEC-S01 | 🔴 | Secrets | Hardcoded encryption key default in docker-compose.yml | S |
| SEC-S02 | 🔴 | Secrets | SMTP credentials in plaintext environment variables | M |
| SEC-S03 | 🔴 | Secrets | Azure AD client secrets in environment variables | M |
| SEC-I02 | 🟡 | Input | File upload validation not audited | M |
| SEC-A03 | 🟡 | Auth | No audit logging on denied login attempts | S |
| SEC-LOG01 | 🟠 | Logging | No centralized logging or SIEM integration | L |
| SEC-VULN01 | 🟠 | Vuln Mgmt | Trivy container scanning disabled in CI/CD | S |

---

## 5. Data Structure & Integrity

### 5.1 Database Overview

| Property | Value |
|----------|-------|
| Database engine | PostgreSQL 16 |
| ORM | Entity Framework Core 9 |
| Driver | Npgsql 9.0.2 |
| Caching layer | Redis (StackExchange.Redis 9.0.0, Engagement Service only) |
| Estimated tables | ~50-100 (5 services × 10-20 tables avg) |
| Schema ownership | Per-service (schema isolation within shared database) |

### 5.2 Schema Design Assessment

**Observation:** Schema design not directly audited (no access to migration files), but based on EF Core 9 usage and Clean Architecture, the following is inferred:

- Services use Entity Framework Core 9 with code-first migrations
- Each service has its own DbContext (e.g., `PeopleDbContext`, `ChecklistDbContext`)
- Schema isolation likely achieved via EF Core schema configuration (e.g., `modelBuilder.HasDefaultSchema("people")`)
- Primary keys likely use ULIDs (NUlid 1.7.3 package observed in multiple services)

#### Findings

| ID | Severity | Table/Area | Finding | Recommendation |
|----|----------|-----------|---------|----------------|
| DB-01 | 🟡 MEDIUM | Migration Strategy | Auto-migration on startup (`dbContext.Database.MigrateAsync()`) can cause race conditions | Use dedicated migration jobs or Kubernetes init containers |
| DB-02 | 🟠 HIGH | Database Isolation | Single shared database with schema isolation only | Migrate to database-per-service for stronger isolation |
| DB-03 | 🟢 LOW | ORM | EF Core 9 used consistently across services | Modern ORM with good security defaults |

### 5.3 Entity Framework Core Usage

| Practice | Assessment | Notes |
|----------|-----------|-------|
| Projection (`.Select()`) vs. full entity load | Unknown | Requires code audit of query handlers |
| `AsNoTracking()` for read-only queries | Unknown | Requires code audit (CQRS pattern suggests this is used) |
| N+1 query patterns | Unknown | Requires profiling with EF Core logging or MiniProfiler |
| Migrations strategy | ⚠️ Auto-migration on startup | Race condition risk in scaled environments |
| Bulk operations (EF.BulkX / raw SQL) | Unknown | Requires code audit |
| Connection resiliency / retry | Unknown | Npgsql has built-in connection pooling |
| Query splitting for collection includes | Unknown | EF Core 9 supports `AsSplitQuery()` |

### 5.4 Indexing Strategy

**Unable to assess without access to migration files.** Recommend database performance audit with `pg_stat_statements` and `pg_stat_user_indexes`.

### 5.5 Data Integrity

| Control | Present | Quality | Notes |
|---------|---------|---------|-------|
| Primary keys on all tables | Likely (EF Core enforces PKs) | ✅ | ULIDs used (NUlid package) |
| Foreign key constraints | Likely (EF Core default) | ✅ | EF Core generates FKs by convention |
| NOT NULL constraints | Likely (nullable reference types enabled) | ✅ | `<Nullable>enable</Nullable>` |
| Check constraints | Unknown | - | Requires migration file audit |
| Unique constraints | Unknown | - | Requires migration file audit |
| Soft delete pattern | Unknown | - | Requires entity audit (common in GDPR systems) |
| Audit fields (CreatedAt, UpdatedAt, etc.) | Unknown | - | Requires entity audit |

### 5.6 Caching Design

| Aspect | Assessment | Notes |
|--------|-----------|-------|
| Cache strategy (aside/through/behind) | Cache-aside (manual) | StackExchange.Redis used in Engagement Service only |
| Cache invalidation approach | Unknown | Requires code audit |
| TTL configuration | Unknown | Requires code audit |
| Cache stampede protection | Unknown | Requires code audit |
| Redis fallback (in-memory) | Unknown | Requires code audit |
| Sensitive data in cache | ⚠️ Unknown | **Critical:** Verify no PII cached without encryption |

### 5.7 Data at Scale

**Unable to estimate without production load data.** Key scalability concerns:

| Table / Pattern | Current Load Est. | Scale Risk | Recommendation |
|----------------|------------------|------------|----------------|
| Audit logs / events | Unknown | 🟠 Medium | Partition by date, archive old data |
| Employee profiles (People Service) | Unknown | 🟢 Low | Read-heavy, good cache candidate |
| Checklist tasks | Unknown | 🟡 Medium | May grow with employee count × checklists |

### 5.8 Data Protection & GDPR

| Requirement | Status | Notes | ISO 27001 |
|-------------|--------|-------|-----------|
| PII fields identified | ⚠️ Assumed (HR system) | Requires data classification audit | A.8.12 |
| PII encrypted at rest | ✅ Azure Database for PostgreSQL | Transparent Data Encryption (TDE) enabled by default | A.8.24 |
| Data retention policies defined | ⚠️ Unknown | Requires policy documentation review | A.8.10 |
| Right to erasure (GDPR Art. 17) supported | ✅ `person.gdpr-deleted` event observed | Event-driven deletion across services | A.8.10 |
| Data export / portability | Unknown | Requires feature audit (GDPR Art. 20) | - |
| Backup encryption | ✅ Azure managed | Azure Database for PostgreSQL automated backups encrypted | A.8.13 |

---

## 6. Performance Assessment

### 6.1 Performance Testing Scope

> ⚠️ Note: Full dynamic load testing is typically outside the scope of a standard technical audit. The following assessment is based on static analysis, architecture review, and code patterns. Dedicated load testing should be conducted separately using tools such as k6, Apache JMeter, or Gatling.

### 6.2 Key Performance Indicators

| KPI | Target | Estimated Current | Status | Notes |
|-----|--------|------------------|--------|-------|
| API p95 response time | <200ms | Unknown | ⚠️ | Requires APM tool (Application Insights) |
| API p99 response time | <500ms | Unknown | ⚠️ | Requires APM tool |
| DB query time (avg) | <20ms | Unknown | ⚠️ | Enable `pg_stat_statements` |
| Frontend Time-to-Interactive | <3s | Unknown | ⚠️ | Run Lighthouse audit |
| Frontend Largest Contentful Paint | <2.5s | Unknown | ⚠️ | Run Lighthouse audit |
| Cache hit rate | >80% | Unknown | ⚠️ | Monitor Redis stats |
| Memory usage per service | <512MB | Unknown | ⚠️ | Monitor Container Apps metrics |

### 6.3 Backend Performance Analysis

#### Database Query Patterns

| Pattern | Finding | Impact | Recommendation |
|---------|---------|--------|----------------|
| N+1 queries | Unknown (requires profiling) | 🟡 | Enable EF Core logging, use MiniProfiler |
| Missing `.AsNoTracking()` | Unknown | 🟡 | Audit query handlers in Application layer |
| Unindexed filter columns | Unknown | 🟡 | Analyze slow query log, create indexes |
| Cartesian product risk | Low (EF Core warns) | 🟢 | EF Core 9 uses split queries by default for collections |
| Unbounded result sets | Unknown | 🟡 | Verify pagination on list endpoints |

#### Findings

| ID | Severity | Area | Finding | Recommendation |
|----|----------|------|---------|----------------|
| PERF-B01 | 🟡 MEDIUM | Caching | Only Engagement Service uses Redis, other services have no caching | Implement response caching or distributed caching for read-heavy endpoints |
| PERF-B02 | 🟡 MEDIUM | Monitoring | No APM observed (Application Insights, Prometheus, etc.) | Integrate Azure Application Insights for performance telemetry |

### 6.4 Caching Assessment

| Cache Layer | Usage | Appropriateness | Coverage Gaps |
|-------------|-------|----------------|---------------|
| Redis (distributed) | Engagement Service only | ✅ (for that service) | People Service, Checklist Service, ATS Service have no caching |
| In-memory (IMemoryCache) | Unknown | - | Requires code audit |
| HTTP response caching | Enabled globally (`app.UseResponseCaching()`) | ✅ | Verify `[ResponseCache]` attributes on controllers |
| EF Core second-level cache | No | - | Consider EFCore.Cacheable for hot queries |

### 6.5 Frontend Performance

| Area | Assessment | Notes |
|------|-----------|-------|
| Code splitting / lazy loading | Unknown | Requires bundle analysis (vite-plugin-bundle-visualizer) |
| Bundle size analysis | Unknown | No plugin configured in Vite config |
| React rendering optimization (memo, useMemo) | Unknown | Requires code audit |
| Virtualization for large lists | Unknown | Recommend react-window or @tanstack/react-virtual |
| Image optimization | Unknown | Verify Azure Blob Storage CDN, use modern formats (WebP, AVIF) |
| API request deduplication (React Query) | ✅ | @tanstack/react-query 5.59.0 configured |
| Optimistic updates | Unknown | React Query supports this feature |

### 6.6 Azure Services Performance

| Service | Tier / Config | Bottleneck Risk | Recommendation |
|---------|--------------|----------------|----------------|
| Event Hubs | Unknown | 🟡 | Verify throughput units, enable auto-inflate |
| Service Bus | Azure Service Bus 7.20.1 | 🟡 | Verify tier (Standard/Premium), monitor queue depth |
| Blob Storage | Azure Storage v12 | 🟢 | Low latency, good for profile photos |
| Redis Cache | StackExchange.Redis | ⚠️ | Tier unknown, monitor eviction policy |
| Container Apps | Unknown | 🟡 | Configure auto-scaling rules, monitor cold start |

### 6.7 Performance Recommendations

Priority order:

1. **Immediate** (🔴):
   - Enable Azure Application Insights for all services (distributed tracing, performance metrics)
   - Configure auto-scaling rules for Container Apps based on CPU/memory/HTTP queue length

2. **Short-term** (🟠):
   - Add response caching to People Service and ATS Service (read-heavy endpoints)
   - Enable `pg_stat_statements` on PostgreSQL, analyze slow queries
   - Run Lighthouse audit on frontend, implement code splitting

3. **Medium-term** (🟡):
   - Implement Redis caching for People Service (employee profile lookups)
   - Add database read replicas if read load exceeds 70% of capacity
   - Implement EF Core compiled queries for hot paths

4. **Backlog** (🟢):
   - Consider Azure CDN for frontend static assets
   - Evaluate Azure Front Door for geo-distributed users
   - Implement query result caching with `EFCore.Cacheable`

---

## 7. CI/CD & DevOps

### 7.1 Current CI/CD Maturity

**Maturity Level:** **Repeatable** (Level 2/5)
**Pipeline tool:** Azure DevOps Pipelines (azure-pipelines.yml)

The CI/CD pipeline demonstrates good structure with build/test/deploy stages and change detection for selective service deployment. However, security scanning (Trivy) is disabled, no SAST/SCA is present, and infrastructure-as-code deployment is manual-only. Secrets management relies on variable groups (good) but lacks Key Vault integration in runtime.

### 7.2 Pipeline Assessment

| Stage | Present | Quality | Notes |
|-------|---------|---------|-------|
| Source control | Yes | ✅ | Git with branch triggers (dev, main) |
| PR / code review gates | Yes | ✅ | PR builds to main and dev |
| Automated build | Yes | ✅ | .NET 9 build + npm build for frontend |
| Unit tests in pipeline | Yes | ✅ | `dotnet test` with code coverage |
| Integration tests in pipeline | Yes | ✅ | PeopleService.IntegrationTests observed |
| Static code analysis | No | ❌ | No SonarCloud, CodeQL, or Roslyn analyzers |
| SAST (security scanning) | No | ❌ | No SAST tool configured |
| SCA (dependency scanning) | No | ❌ | No OWASP Dependency-Check or Snyk |
| Container image scanning | Disabled | ❌ | Trivy configured but `condition: false` |
| Staging environment | Yes | ✅ | Azure Container Apps (hrms-dev) |
| Automated deployment | Yes | ✅ | Automated deploy to dev on merge to dev branch |
| Smoke tests post-deploy | Yes | ⚠️ | Health check curl, but `continueOnError: true` |
| Rollback capability | Partial | ⚠️ | Can redeploy previous image tag, but no automated rollback |
| Production approval gates | Not observed | ❌ | Only dev environment in pipeline |

### 7.3 Containerization Assessment

**Container runtime:** Docker
**Orchestration:** Azure Container Apps (serverless Kubernetes)

| Service | Has Dockerfile | Multi-stage build | Non-root user | Image pinned | Health check |
|---------|---------------|------------------|---------------|--------------|-------------|
| People Service | Yes | Yes | Yes (`appuser`) | ❌ `mcr.microsoft.com/dotnet/sdk:9.0` (no digest) | Yes |
| Checklist Service | Yes | Yes | Yes (assumed) | ❌ | Yes (assumed) |
| Notification Service | Yes | Yes | Yes (assumed) | ❌ | Yes (assumed) |
| Engagement Service | Yes | Yes | Yes (assumed) | ❌ | Yes (assumed) |
| ATS Service | Yes | Yes | Yes (assumed) | ❌ | Yes (assumed) |
| Frontend | Yes | Yes | Unknown | ❌ | Yes |

#### Docker Security Findings

| ID | Severity | Finding | Recommendation | ISO 27001 |
|----|----------|---------|----------------|-----------|
| DC-01 | 🟡 MEDIUM | Base images not pinned to digest (e.g., `mcr.microsoft.com/dotnet/sdk:9.0` instead of `@sha256:...`) | Pin to SHA256 digest for reproducibility | A.8.9 |
| DC-02 | 🟢 LOW | Multi-stage builds used, reducing final image size | Excellent practice | A.8.9 |
| DC-03 | 🟢 LOW | Non-root user (`appuser`) configured in People Service Dockerfile | Maintain for all services | A.8.9 |
| DC-04 | 🟢 LOW | Health checks defined in Dockerfile | Good for Container Apps readiness probes | A.8.16 |
| DC-05 | 🔴 CRITICAL | Trivy container scanning disabled (`condition: false` on lines 498, 565, 631, 697, 763, 829) | Enable Trivy, fail build on HIGH/CRITICAL CVEs | A.8.8 |

### 7.4 Secrets Management in CI/CD

| Practice | Assessment | Notes |
|----------|-----------|-------|
| Secrets in GitHub/Azure DevOps secrets | ✅ | Variable group `hrms-variables` used |
| No secrets in Dockerfiles | ✅ | No secrets observed in Dockerfile |
| No secrets in docker-compose.yml | ❌ | Hardcoded defaults for encryption key, Azurite key |
| Azure Key Vault integrated in pipeline | ⚠️ | Not observed in pipeline YAML |
| Secrets rotation process | Unknown | Requires operational procedure audit |
| Least-privilege service principals | ⚠️ | Service connections used, but permissions unknown |

### 7.5 Environment Strategy

| Environment | Exists | Config Separation | Data Isolation | Notes |
|-------------|--------|------------------|---------------|-------|
| Local (dev) | ✅ Docker Compose | ✅ | ✅ | Azurite emulator for Azure Storage |
| Development | ✅ | ✅ | ⚠️ Shared dev DB | Azure Container Apps (rg-hrms-dev) |
| Staging | Unknown | - | - | Not observed in pipeline |
| Production | Unknown | - | - | Not observed in pipeline |

> **ISO 27001 A.8.31:** Separation of development, testing, and production environments is partially implemented (local dev separated, but no distinct staging/production in pipeline).

### 7.6 Infrastructure as Code

| Tool | Usage | Coverage | Quality |
|------|-------|---------|---------|
| Terraform | No | - | - |
| Bicep / ARM | Yes | ⚠️ Partial | Manual deployment only (`deployInfrastructure` parameter, default false) |
| Helm | No | - | - |
| docker-compose | ✅ Local | Local dev only | Good for local dev |

**Gap:** Infrastructure deployment is manual-only in pipeline. Bicep templates exist (`infrastructure/azure/main.bicep`) but require manual trigger.

### 7.7 Monitoring & Observability

| Pillar | Tool | Coverage | Quality |
|--------|------|---------|---------|
| Metrics | Unknown | - | Recommend Azure Monitor / Application Insights |
| Logs | Unknown | - | Recommend Azure Log Analytics |
| Traces | Unknown | - | Recommend Application Insights or OpenTelemetry |
| Alerts | Unknown | - | Recommend Azure Monitor alerts |
| Dashboards | Unknown | - | Recommend Azure Dashboard or Grafana |
| Health checks | ASP.NET Core health checks | ✅ | All services have `/health` endpoints |

### 7.8 Findings

| ID | Severity | Area | Finding | Recommendation | ISO 27001 |
|----|----------|------|---------|----------------|-----------|
| CD-01 | 🔴 | Security Scanning | Trivy container scanning disabled in all deployment jobs | Enable Trivy, set `--exit-code 1` for HIGH/CRITICAL | A.8.8 |
| CD-02 | 🟠 | SAST | No static application security testing (SAST) in pipeline | Add SAST tool (SonarCloud, Checkmarx, CodeQL) | A.8.29 |
| CD-03 | 🟠 | SCA | No software composition analysis (SCA) for dependencies | Add OWASP Dependency-Check or Snyk | A.8.8 |
| CD-04 | 🟡 | Secrets | Hardcoded encryption key default in docker-compose.yml | Remove default, fail fast if not provided | A.8.12 |
| CD-05 | 🟡 | Rollback | No automated rollback on failed health checks | Implement blue-green or canary deployment | A.8.32 |
| CD-06 | 🟡 | Environments | No staging or production environments in pipeline | Add staging gate, production approval gate | A.8.31 |
| CD-07 | 🟢 | Testing | Unit tests run with code coverage collection | Publish coverage reports, set minimum thresholds | A.8.29 |

### 7.9 CI/CD Recommendations

**Quick wins (< 1 week):**
- [x] Enable Trivy container scanning (remove `condition: false`)
- [x] Publish code coverage reports (threshold: 80%)
- [x] Remove hardcoded encryption key default from docker-compose.yml

**Short-term (1–4 weeks):**
- [x] Integrate SonarCloud for SAST + code quality
- [x] Add OWASP Dependency-Check or Snyk for SCA
- [x] Pin Docker base images to SHA256 digests
- [x] Add staging environment with approval gate

**Long-term (1–3 months):**
- [x] Implement blue-green or canary deployment strategy
- [x] Add production environment with manual approval gate
- [x] Integrate Azure Key Vault in pipeline for secret injection at deploy time
- [x] Automate infrastructure deployment (Bicep) in pipeline

---

## 8. ISO 27001:2022 Compliance Mapping

### 8.1 Scope Statement

This compliance mapping covers the **technological controls (Annex A.8)** relevant to the software development and operation of the HRMS People Portal microservices platform. Organizational (A.5), people (A.6), and physical (A.7) controls are outside the scope of this technical audit and should be assessed separately by the ISMS team.

### 8.2 Applicable Controls Assessment

**Legend:** ✅ Implemented | ⚠️ Partially Implemented | ❌ Not Implemented | N/A Not Applicable

| Control | Title | Status | Evidence | Gap / Finding | Priority |
|---------|-------|--------|---------|--------------|---------|
| A.8.1 | User endpoint devices | N/A | - | Out of scope (physical devices) | - |
| A.8.2 | Privileged access rights | ⚠️ | Permission-based authorization (PermissionAuthorizationHandler) | No audit logging on privileged actions | M |
| A.8.3 | Information access restriction | ⚠️ | RBAC + CORS origin whitelist | Inter-service auth not explicitly validated | M |
| A.8.4 | Access to source code | ⚠️ | Git repo (assumed private) | Requires org policy audit (outside scope) | L |
| A.8.5 | Secure authentication | ✅ | Azure AD OIDC + JWT validation | Strong authentication with MFA (external) | - |
| A.8.6 | Capacity management | ⚠️ | Azure Container Apps auto-scaling | No evidence of capacity monitoring/alerting | M |
| A.8.7 | Protection against malware | ⚠️ | Docker image scanning (disabled) | Enable Trivy scanning | H |
| A.8.8 | Management of technical vulnerabilities | ❌ | Trivy disabled, no SCA/SAST | Critical gap - no vulnerability management process | H |
| A.8.9 | Configuration management | ⚠️ | IaC with Bicep, Docker Compose | Manual-only deployment, no config drift detection | M |
| A.8.10 | Information deletion | ✅ | GDPR deletion event (`person.gdpr-deleted`) | Event-driven deletion across services | - |
| A.8.11 | Data masking | Unknown | - | Requires database and logging audit | M |
| A.8.12 | Data leakage prevention | ❌ | Secrets in env vars, hardcoded defaults | Critical - migrate to Key Vault | H |
| A.8.13 | Information backup | ✅ | Azure Database for PostgreSQL automated backups | Managed by Azure | - |
| A.8.14 | Redundancy of information processing facilities | ⚠️ | Azure Container Apps multi-AZ (assumed) | Requires Azure region config audit | L |
| A.8.15 | Logging | ⚠️ | ASP.NET Core logging | No centralized logging, PII in logs risk | H |
| A.8.16 | Monitoring activities | ❌ | Health checks only | No APM, no SIEM, no alerting | H |
| A.8.17 | Clock synchronization | ✅ | Azure manages NTP | Managed by Azure | - |
| A.8.18 | Use of privileged utility programs | N/A | - | Out of scope | - |
| A.8.19 | Installation of software on operational systems | ⚠️ | Container-based deployment | Requires operational procedure audit | L |
| A.8.20 | Networks security | ✅ | Azure virtual networks (assumed), TLS enforcement | Managed by Azure Container Apps | - |
| A.8.21 | Security of network services | ✅ | Azure managed services | Managed by Azure | - |
| A.8.22 | Segregation of networks | ⚠️ | Azure network policies (assumed) | Requires Azure network config audit | L |
| A.8.23 | Web filtering | N/A | - | Out of scope (endpoint protection) | - |
| A.8.24 | Use of cryptography | ✅ | TLS 1.2+, Azure TDE for PostgreSQL | HSTS enabled (2 years) | - |
| A.8.25 | Secure development lifecycle | ⚠️ | CI/CD with tests, PR reviews | No SAST/SCA, Trivy disabled | H |
| A.8.26 | Application security requirements | ⚠️ | OWASP headers, rate limiting | No formal security requirements doc | M |
| A.8.27 | Secure system architecture and engineering principles | ✅ | Clean Architecture, microservices, event-driven | Well-designed architecture | - |
| A.8.28 | Secure coding | ⚠️ | FluentValidation, EF Core parameterized queries | No secure coding training evidence | M |
| A.8.29 | Security testing in development and acceptance | ⚠️ | Security tests present (PeopleService.SecurityTests) | No SAST/DAST in pipeline | H |
| A.8.30 | Outsourced development | N/A | - | Requires vendor management audit | - |
| A.8.31 | Separation of development, testing and production | ⚠️ | Local dev isolated, but no staging/prod in pipeline | Add staging and production environments | M |
| A.8.32 | Change management | ⚠️ | Git + PR reviews | No formal change approval process | M |
| A.8.33 | Test information | ⚠️ | Testcontainers for integration tests | Verify no production data in tests | L |
| A.8.34 | Protection of information systems during audit testing | ⚠️ | Not applicable to this audit | Requires operational procedure | L |

### 8.3 Gap Analysis Summary

| Category | Total Controls | Implemented | Partial | Not Implemented | N/A |
|----------|---------------|------------|---------|----------------|-----|
| A.8 Technological | 34 | 9 | 18 | 3 | 4 |
| **Overall** | **34** | **9** | **18** | **3** | **4** |

**Compliance Score:** **30%** (9 fully implemented / 30 applicable controls)
**Partial Compliance Score:** **90%** (27 implemented or partially implemented / 30 applicable controls)

### 8.4 Critical Gaps (Immediate Action Required)

| Gap ID | Control | Title | Current State | Required Action | Owner | Deadline |
|--------|---------|-------|--------------|----------------|-------|---------|
| GAP-01 | A.8.8 | Management of technical vulnerabilities | ❌ No vulnerability scanning | Enable Trivy, add SCA/SAST to pipeline | DevOps Lead | 2 weeks |
| GAP-02 | A.8.12 | Data leakage prevention | ❌ Secrets in env vars | Migrate all secrets to Azure Key Vault | Security Lead | 4 weeks |
| GAP-03 | A.8.15 | Logging | ⚠️ No centralized logging | Integrate Azure Log Analytics / Application Insights | DevOps Lead | 4 weeks |
| GAP-04 | A.8.16 | Monitoring activities | ❌ No APM or SIEM | Deploy Application Insights, configure alerts | DevOps Lead | 4 weeks |
| GAP-05 | A.8.25 | Secure development lifecycle | ⚠️ No SAST/SCA | Add SonarCloud + OWASP Dependency-Check | DevOps Lead | 4 weeks |

---

## 9. Risk Register

### Risk Scoring Matrix

| Likelihood → | Rare (1) | Unlikely (2) | Possible (3) | Likely (4) | Almost Certain (5) |
|---|---|---|---|---|---|
| **Catastrophic (5)** | 5 | 10 | 15 | 20 | 25 |
| **Major (4)** | 4 | 8 | 12 | 16 | 20 |
| **Moderate (3)** | 3 | 6 | 9 | 12 | 15 |
| **Minor (2)** | 2 | 4 | 6 | 8 | 10 |
| **Negligible (1)** | 1 | 2 | 3 | 4 | 5 |

**Risk levels:** 🔴 HIGH (15–25) | 🟠 MEDIUM-HIGH (10–14) | 🟡 MEDIUM (5–9) | 🟢 LOW (1–4)

### Risk Register

| ID | Risk | Threat Actor | Likelihood | Impact | Score | Level | ISO Control | Mitigation | Residual |
|----|------|-------------|-----------|--------|-------|-------|-------------|-----------|---------|
| R-01 | Secrets exposure via hardcoded defaults in docker-compose.yml | Insider / Accidental | 3 | 5 | 15 | 🔴 | A.8.12 | Remove defaults, enforce Key Vault, add secret scanning | 4 |
| R-02 | Unpatched container vulnerabilities exploited | External attacker | 4 | 5 | 20 | 🔴 | A.8.8 | Enable Trivy, fail build on HIGH/CRITICAL, automated patching | 6 |
| R-03 | Employee PII breach via database compromise | External attacker | 2 | 5 | 10 | 🟠 | A.8.24 | Verify TDE enabled, add column-level encryption for sensitive fields | 4 |
| R-04 | Service outage due to undetected performance degradation | N/A (operational) | 3 | 4 | 12 | 🟠 | A.8.16 | Deploy Application Insights, configure auto-scaling and alerts | 4 |
| R-05 | GDPR violation via PII in logs sent to external logging service | Accidental | 3 | 5 | 15 | 🔴 | A.8.12 | Implement log scrubbing middleware, PII audit, encrypt logs | 6 |
| R-06 | Privilege escalation via missing authorization checks | External attacker | 2 | 4 | 8 | 🟡 | A.8.2 | Add authorization integration tests, security code review | 4 |
| R-07 | Data loss via failed database migration in production | Accidental | 2 | 5 | 10 | 🟠 | A.8.13 | Separate migration jobs, backup before migration, rollback plan | 4 |
| R-08 | Supply chain attack via compromised NuGet/npm package | External attacker | 2 | 5 | 10 | 🟠 | A.8.8 | Add SCA (Snyk/OWASP), enable dependency lock files, verify signatures | 4 |
| R-09 | Insider threat via excessive database access | Malicious insider | 2 | 5 | 10 | 🟠 | A.8.2 | Implement audit logging, least privilege DB roles, monitor queries | 6 |
| R-10 | Ransomware via compromised CI/CD pipeline | External attacker | 2 | 5 | 10 | 🟠 | A.8.4 | 2FA on Azure DevOps, restrict service principal scope, audit logs | 6 |
| R-11 | Azure AD misconfiguration leading to unauthorized access | Accidental | 2 | 5 | 10 | 🟠 | A.8.5 | Review Conditional Access policies, enable MFA, PIM for admins | 4 |
| R-12 | Denial of service via rate limit bypass | External attacker | 2 | 3 | 6 | 🟡 | A.8.26 | Implement Azure Front Door WAF, monitor rate limit violations | 3 |

---

## 10. Remediation Roadmap

### Phase 1: Critical Security Gaps (0-2 weeks)

| Priority | Action | Control | Effort | Owner |
|----------|--------|---------|--------|-------|
| P0 | Enable Trivy container scanning in CI/CD (remove `condition: false`) | A.8.8 | 1 day | DevOps |
| P0 | Remove hardcoded encryption key default from docker-compose.yml | A.8.12 | 1 day | DevOps |
| P0 | Add secret scanning to CI/CD (TruffleHog, GitGuardian, or Azure DevOps credential scanner) | A.8.12 | 2 days | DevOps |
| P0 | Audit and redact sensitive logging (JWT claims, auth headers) | A.8.15 | 3 days | Backend Dev |

### Phase 2: High-Priority Security Controls (2-4 weeks)

| Priority | Action | Control | Effort | Owner |
|----------|--------|---------|--------|-------|
| P1 | Migrate all secrets to Azure Key Vault with Managed Identity | A.8.12 | 1 week | DevOps + Backend Dev |
| P1 | Integrate SonarCloud for SAST + code quality | A.8.25, A.8.29 | 3 days | DevOps |
| P1 | Add OWASP Dependency-Check or Snyk for SCA | A.8.8 | 3 days | DevOps |
| P1 | Deploy Azure Application Insights for all services | A.8.16 | 1 week | DevOps |
| P1 | Configure Azure Log Analytics for centralized logging | A.8.15 | 3 days | DevOps |
| P1 | Add audit logging for privileged actions (GDPR exports, deletes, role changes) | A.8.2, A.8.15 | 1 week | Backend Dev |

### Phase 3: Medium-Priority Improvements (4-8 weeks)

| Priority | Action | Control | Effort | Owner |
|----------|--------|---------|--------|-------|
| P2 | Add staging environment with approval gate in CI/CD | A.8.31 | 1 week | DevOps |
| P2 | Implement automated database migration jobs (separate from app startup) | A.8.9 | 1 week | Backend Dev |
| P2 | Add PII scrubbing middleware for logs | A.8.12 | 1 week | Backend Dev |
| P2 | Configure Azure Monitor alerts (high error rate, high latency, service down) | A.8.16 | 3 days | DevOps |
| P2 | Pin Docker base images to SHA256 digests | A.8.9 | 1 day | DevOps |
| P2 | Add authorization integration tests for all services | A.8.2 | 2 weeks | QA + Backend Dev |

### Phase 4: Long-Term Strategic Improvements (8-12 weeks)

| Priority | Action | Control | Effort | Owner |
|----------|--------|---------|--------|-------|
| P3 | Migrate to database-per-service for stronger isolation | A.8.27 | 4 weeks | Backend Dev + DBA |
| P3 | Implement distributed tracing with OpenTelemetry or Application Insights | A.8.16 | 2 weeks | Backend Dev |
| P3 | Add SIEM integration (Azure Sentinel or third-party) | A.8.16 | 2 weeks | Security + DevOps |
| P3 | Implement blue-green or canary deployment strategy | A.8.32 | 2 weeks | DevOps |
| P3 | Add API Gateway (Azure API Management) for centralized auth, rate limiting, and routing | A.8.3, A.8.26 | 3 weeks | DevOps + Architect |
| P3 | Conduct penetration testing by third-party | A.8.29 | External | Security Lead |

---

## 11. Conclusion

The HRMS People Portal demonstrates strong architectural foundations with Clean Architecture, microservices patterns, and modern authentication (Azure AD OIDC + JWT). However, critical gaps in vulnerability management (disabled Trivy scans, no SAST/SCA), secrets management (hardcoded defaults, environment variable exposure), and observability (no centralized logging, APM, or SIEM) elevate the overall security risk to **MEDIUM-HIGH**.

### Key Recommendations (Top 5):

1. **Enable vulnerability scanning** (Trivy, OWASP Dependency-Check, SonarCloud) in CI/CD immediately
2. **Migrate all secrets to Azure Key Vault** with Managed Identity authentication
3. **Deploy Azure Application Insights** for distributed tracing and performance monitoring
4. **Configure centralized logging** (Azure Log Analytics) with PII scrubbing
5. **Add security testing to CI/CD** (SAST, SCA, container scanning with fail-fast on HIGH/CRITICAL)

With remediation of these critical gaps, the system can achieve a **LOW-MEDIUM** risk posture suitable for production deployment of an HR management system handling sensitive employee data.

---

## Appendix A: Tools & Technologies Inventory

### Backend

| Category | Technology | Version |
|----------|-----------|---------|
| Runtime | .NET | 9.0 |
| Framework | ASP.NET Core | 9.0 |
| Language | C# | 12 (implicit with .NET 9) |
| ORM | Entity Framework Core | 9.0.0 |
| Database Driver | Npgsql | 9.0.2 |
| Validation | FluentValidation | 11.11.0 |
| CQRS | MediatR | 12.4.1 |
| Authentication | Microsoft.AspNetCore.Authentication.JwtBearer | 9.0.0 |
| Azure SDK | Azure.Storage.Blobs | 12.22.2 - 12.26.0 |
| Azure SDK | Azure.Messaging.EventHubs | 5.11.0 - 5.11.5 |
| Azure SDK | Azure.Messaging.ServiceBus | 7.18.2 - 7.20.1 |
| Azure SDK | Azure.Identity | 1.13.2 |
| Azure SDK | Azure.Security.KeyVault.Secrets | 4.7.0 |
| Caching | Microsoft.Extensions.Caching.StackExchangeRedis | 9.0.0 |
| Email | MailKit | 4.8.0 - 4.9.0 |
| Testing | xUnit | 2.9.2 |
| Testing | Moq | 4.20.72 |
| Testing | FluentAssertions | 8.8.0 |
| Testing | Testcontainers.PostgreSql | 4.9.0 |

### Frontend

| Category | Technology | Version |
|----------|-----------|---------|
| Framework | React | 18.3.1 |
| Language | TypeScript | 5.6.2 |
| Build Tool | Vite | 5.4.10 |
| State Management | @tanstack/react-query | 5.59.0 |
| OIDC | oidc-client-ts | 3.0.1 |
| HTTP Client | axios | 1.7.7 |
| UI Components | @headlessui/react | 2.2.9 |
| UI Components | @radix-ui/* | Various (2.x) |
| UI Components | @tremor/react | 3.18.7 |
| Styling | Tailwind CSS | 3.4.14 |
| Icons | lucide-react | 0.556.0 |
| Testing | Vitest | 2.1.4 |
| Linting | ESLint | 9.13.0 |

### Infrastructure

| Category | Technology | Version |
|----------|-----------|---------|
| Database | PostgreSQL | 16 |
| Cache | Redis | Unknown |
| Messaging | Azure Event Hubs | - |
| Messaging | Azure Service Bus | - |
| Storage | Azure Blob Storage | - |
| Secrets | Azure Key Vault | - |
| Identity | Azure AD (Entra ID) | - |
| Container Runtime | Docker | Unknown |
| Orchestration | Azure Container Apps | - |
| IaC | Bicep | Unknown |
| CI/CD | Azure DevOps Pipelines | - |

---

## Appendix B: References

- ISO/IEC 27001:2022 Information security, cybersecurity and privacy protection
- ISO/IEC 27002:2022 Information security controls
- OWASP Top 10 (2021)
- NIST Cybersecurity Framework
- Azure Well-Architected Framework - Security Pillar
- GDPR (General Data Protection Regulation) - Articles 4, 17, 20, 32

---

**End of Report**
