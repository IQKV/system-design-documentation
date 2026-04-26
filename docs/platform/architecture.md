# Architecture

## Tenancy & Isolation

The platform uses a **Hybrid Tenancy Model** that supports both public SaaS (Multi-Tenant) and internal/enterprise (Single-Tenant) deployments using the same codebase.

### Deployment Archetypes

- **Multi-Tenant (Default):** Every registration creates a new organization with a unique 8-character NanoID key using alphabet `[a-z0-9]`.
- **Single-Tenant:** Tenancy is hidden. All users are automatically joined to a single "Default" tenant created during bootstrapping.

### Isolation Strategy

Isolation is handled via a **PostgreSQL Schema-Per-Tenant Model**:

1. **System Schema (`public`):** Contains platform-wide data (users, tenants, memberships, token denylist)
2. **Tenant Schemas (`t_{tenantKey}`):** Contains tenant-specific business data with complete isolation
3. **MyBatis Interceptor:** Automatically switches PostgreSQL `search_path` based on tenant context
4. **Liquibase Migrations:** Separate changesets for system vs tenant schemas

For details on the hybrid architecture, NanoID resolution, and bootstrapping, see the [Tenancy Deep Dive](./tenancy.md).

---

## Overview

```
  Client ──────────────────▶ API Gateway (Spring Cloud Gateway)
                                  │
                    ┌─────────────┴──────────────┐
                    ▼                            ▼
                   IAM                        Billing
              (Spring Boot)              (Spring Boot)
                    │                            │
                    ▼                            ▼
             PostgreSQL (iam)            PostgreSQL (billing)
            (Schema-per-tenant)          (Stripe integration)

                    └─────────────┬──────────────┘
                                  ▼
                              RabbitMQ
                        (Async tenant provisioning
                         & lifecycle events)

  UI (React + Mantine) ──▶ API Gateway (all requests proxied)
                          (JWT validation & context propagation)
```

---

## Services

### IAM Service

**Core Identity & Access Management with Multi-Tenant Support**

**Authentication & Authorization:**
- User signup with email verification (secure token-based)
- JWT RS256 authentication: access tokens (15 min) + refresh tokens (7 days)
- Password reset via signed email tokens (1h TTL), rate-limited (3 requests per 15min window)
- Brute-force protection: account lockout after 5 failed attempts for 15 minutes
- Token revocation: JTI denylist + global signout timestamp with automatic cleanup
- JWKS endpoint (`/.well-known/jwks.json`) for distributed token validation

**Tenant & Organization Management:**
- Tenant lifecycle: create, suspend, delete, retry provisioning
- Async tenant provisioning via RabbitMQ with ShedLock-guarded reaper for stuck tenants
- Multi-tenant membership: one user can belong to multiple organizations
- RBAC with authorities: `TENANT_OWNER`, `ADMIN`, `MEMBER`
- Cross-tenant user context switching and tenant discovery

**Invitation System:**
- Email invitations with 72h expiring tokens, default authority `MEMBER`
- New users created on accept (email pre-verified)
- Existing users verified by password
- ShedLock-guarded reaper for stale invitation cleanup

**Multi-Mode Support:**
- Pluggable bootstrap strategies for different rollout modes
- Single-tenant: auto-provision default tenant at startup
- Multi-tenant: per-signup tenant creation
- Platform mode consistency validation across services

**Events Published:** `tenant.provisioned`, `tenant.suspended`, `user.invited`, `user.removed`, `tenant.provisioning.failed`

**Tech Stack:** Java 25, Spring Boot 4.0, MyBatis 3.x, PostgreSQL 17, Liquibase, RabbitMQ, JJWT 0.13 (RS256), ShedLock 7.x, Thymeleaf (email templates)

---

### API Gateway Service

**Reactive Gateway with Security & Context Propagation**

**Core Capabilities:**
- Spring Cloud Gateway with WebFlux (reactive, non-blocking)
- JWT validation against IAM JWKS endpoint with authority extraction
- Multi-mode tenant resolution: JWT claims (multi-tenant) vs auto-injection (single-tenant)
- Platform mode guard: validates rollout mode consistency with IAM service
- Header sanitization: prevents client spoofing of `X-User-*` and `X-Tenant-ID` headers

**Context Propagation:**
- Extracts user context from validated JWTs
- Propagates as headers to downstream services:
  - `X-User-ID`, `X-Username`, `X-User-Email`, `X-User-Authorities`, `X-Tenant-ID`
- Correlation ID injection for distributed tracing

**Routing & Security:**
- Path-based routing to upstream services (IAM, Billing)
- Configurable public paths (JWKS, webhooks, health checks, Swagger UI)
- Global CORS configuration with configurable origins and methods
- Rate limiting and request logging with structured output

**Observability:**
- Prometheus metrics and health checks on separate management port
- Swagger UI aggregation from downstream services
- Correlation ID filter for request tracing

**Events Published:** `api.request.metered` (planned)

**Tech Stack:** Java 25, Spring Boot 4.0, Spring Cloud Gateway, Spring Security OAuth2 Resource Server, WebFlux, Micrometer

---

### Billing Service

**Stripe Integration with Multi-Tenant Billing Support**

**Core Features:**
- Complete Stripe integration layer with webhook processing
- Plan catalog management with tenant/user scoped plans
- Multi-mode billing: tenant-scoped (multi-tenant) vs user-scoped (single-tenant)
- Subscription lifecycle management with local caching
- Entitlement evaluation for authorization decisions

**Stripe Integration:**
- Customer provisioning and management
- Webhook processing with signature verification and idempotency
- Outbox pattern for reliable event delivery
- Subscription state caching for fast reads

**Plan Management:**
- Pre-provisioned subscription plans with pricing and features
- Plan eligibility validation based on rollout mode
- CRUD API for plan catalog management
- Feature-based entitlement evaluation

**Events Published:** `subscription.created`, `subscription.cancelled`, `invoice.paid`, `payment.failed`, `billing.settings.updated`

**Tech Stack:** Java 25, Spring Boot 4.0, MyBatis 3.x, PostgreSQL, Stripe Java SDK, RabbitMQ, Jackson (JSON processing)

### Billing Settings

Each tenant/user has a `billing_settings` record — the single source of truth for Stripe customer metadata:

```sql
billing_settings
├── id                UUID PK
├── tenant_key        VARCHAR(8) UNIQUE FK → tenant (multi-tenant)
├── user_id           UUID UNIQUE FK → user (single-tenant)
├── stripe_customer_id VARCHAR
├── billing_email     VARCHAR        -- finance contact, no system access required
├── company_name      VARCHAR
├── billing_address   JSONB          -- street, city, country, postal_code
├── tax_id            VARCHAR        -- VAT/GST number for B2B compliance
├── tax_id_type       VARCHAR        -- Stripe enum: eu_vat, gb_vat, au_abn, etc.
├── currency          VARCHAR(3)     -- ISO 4217, default USD
├── created_at        TIMESTAMP
└── updated_at        TIMESTAMP
```

**Key Design Decisions:**
- Auto-created on tenant provisioning with registration defaults
- Decoupled from IAM users for billing independence
- VAT/GST details flow directly into Stripe invoices
- Supports both tenant-scoped and user-scoped billing models
- Outbox pattern ensures reliable Stripe synchronization

---

## UI Application

**React SPA with Mantine UI Framework**

**Architecture:**
- Single Page Application (SPA) built with React and Mantine UI
- Communicates exclusively through the API Gateway (no direct service access)
- Deployed as static build (Nginx container or CDN)
- JWT-based authentication with automatic token refresh

**Feature Coverage:**
- **Authentication Flows:** Sign up, login, password reset, email verification
- **Organization Management:** Create org, invite members, manage roles, tenant switching
- **Account Settings:** Profile management, password change, user preferences
- **Billing Portal:** Integration with Stripe-hosted dashboard for subscriptions and invoices

**Tenancy Adaptation:**
- Mode detection via IAM actuator endpoint (`/actuator/info`)
- Conditional UI rendering based on rollout mode
- Multi-tenant: shows organization switcher and management features
- Single-tenant: hides tenancy concepts, focuses on workspace features

**Tech Stack:** React 18, Mantine UI, TypeScript, Vite (build tool), Nginx (static hosting)

---

## Data Layer

### PostgreSQL — Database-Per-Service Pattern

Each service owns its own PostgreSQL database with complete data isolation. No shared databases or cross-service table access — inter-service communication flows through APIs or the event bus.

| Service | Database             | Contents                                                    |
| ------- | -------------------- | ----------------------------------------------------------- |
| IAM     | `foundation_iam`     | Users, tenants, memberships, authorities, invitations, tokens |
| Billing | `foundation_billing` | Stripe customer refs, subscription cache, webhook logs, plans |

### Schema-Per-Tenant Architecture (IAM Database)

Within `foundation_iam`, each tenant gets a dedicated PostgreSQL schema with automatic routing:

```
foundation_iam/
├── public/                    # Platform registry
│   ├── users                  # Global user accounts
│   ├── tenants               # Tenant registry with status
│   ├── tenant_memberships    # User-tenant relationships
│   ├── token_denylist        # Revoked JWT tokens
│   ├── failed_login_attempts # Brute-force tracking
│   └── shedlock             # Distributed job coordination
├── t_abc12345/              # Tenant-specific schemas
│   ├── tenant_member_authorities  # Per-tenant RBAC
│   ├── invitations               # Tenant invitations
│   └── [business entities]       # Tenant-scoped data
├── t_def67890/
└── t_ghi13579/
```

**Schema Switching Implementation:**
```java
@Intercepts({@Signature(type = StatementHandler.class, method = "prepare")})
public class MyBatisSchemaInterceptor implements Interceptor {
    public Object intercept(Invocation invocation) throws Throwable {
        Connection connection = (Connection) invocation.getArgs()[0];
        String tenantKey = TenantContext.getCurrentTenant();
        String schema = "t_" + tenantKey;
        
        try (PreparedStatement stmt = connection.prepareStatement(
                "SET search_path TO " + schema + ", public")) {
            stmt.execute();
        }
        return invocation.proceed();
    }
}
```

**Migration Strategy:**
- **System Migrations:** `db/changelog/system/db.changelog-master.xml`
- **Tenant Migrations:** `db/changelog/tenant/master.xml`
- **Liquibase Runner:** Automatic schema creation and migration per tenant
- **Cross-Tenant Queries:** Not possible in normal application flow (security by design)

### RabbitMQ — Event-Driven Architecture

Asynchronous communication and tenant provisioning via RabbitMQ with durable queues and dead letter handling:

| Exchange      | Routing Key              | Consumer                | Purpose                           |
| ------------- | ------------------------ | ----------------------- | --------------------------------- |
| `iqkv.events` | `tenant.provisioning.requested` | Provisioning Worker | Create schema, run migrations |
| `iqkv.events` | `tenant.provisioned`     | Billing Service         | Create Stripe customer            |
| `iqkv.events` | `tenant.provisioning.failed` | Monitoring/Alerts   | Handle provisioning failures      |
| `iqkv.events` | `tenant.suspended`       | Billing Service         | Mark billing profile inactive     |
| `iqkv.events` | `user.invited`           | Extensions              | Invitation notifications          |
| `iqkv.events` | `user.removed`           | Extensions              | Membership removal cleanup        |
| `iqkv.events` | `subscription.cancelled` | IAM Service             | Suspend tenant on payment failure |

**Event Processing Patterns:**
- **Idempotent Consumers:** All event handlers are idempotent and safe to retry
- **Dead Letter Queues:** Failed events are routed to DLQ for manual investigation
- **ShedLock Coordination:** Prevents duplicate processing in clustered deployments
- **Outbox Pattern:** Ensures reliable event publishing with transactional guarantees

---

## Tenant Provisioning Flow

### Multi-Tenant Mode (Default)

```
1. POST /api/v1/iam/auth/signup
   ├── Create user account (email verification required)
   ├── Generate 8-char NanoID tenant key
   └── Create tenant record (status: PROVISIONING)
       │
2. Publish tenant.provisioning.requested → RabbitMQ
       │
3. Return HTTP 201 { tenantKey, status: "PROVISIONING" }
       │
4. Async Processing:
   ├── TenantProvisioningConsumer
   │   ├── Create PostgreSQL schema t_{tenantKey}
   │   ├── Run Liquibase tenant migrations
   │   ├── Update tenant status: ACTIVE
   │   └── Publish tenant.provisioned event
   │
   └── Billing Consumer (on tenant.provisioned)
       ├── Create Stripe customer
       ├── Store customer ID in billing_settings
       └── Initialize default billing configuration

5. Client polls GET /api/v1/iam/tenants/{tenantKey} until ACTIVE
```

### Single-Tenant Mode (Bootstrap)

When `iqkv.platform.rollout-mode: SINGLE_TENANT`, IAM runs the same provisioning flow at application startup:

```
1. Application Startup (ApplicationReadyEvent)
       │
2. SingleTenantBootstrapStrategy.bootstrap()
   ├── Check if default tenant exists
   ├── If not: create default tenant (status: PROVISIONING)
   └── Run same async provisioning flow
       │
3. Default tenant becomes ACTIVE
       │
4. All user signups join the default tenant with MEMBER authority
   (TENANT_OWNER authority reserved for initial admin)
```

**Failure Handling:**
- **Exponential Backoff:** Workers retry with exponential backoff on failure
- **Stuck Tenant Reaper:** ShedLock-guarded job cleans up tenants stuck in `PROVISIONING` (configurable timeout: 10 minutes)
- **Manual Retry:** Owners can trigger `POST /tenants/{tenantKey}/retry-provisioning` for `PROVISIONING_FAILED` tenants
- **Monitoring:** Prometheus metrics track provisioning success/failure rates

---

## Security Architecture

### JWT-Based Authentication

**Token Structure:**
- **Algorithm:** RS256 (asymmetric signing)
- **Access Token:** 15-minute expiry with user context and authorities
- **Refresh Token:** 7-day expiry for token rotation
- **Claims:** `userId`, `username`, `email`, `tenant_id`, `authorities`, `email_verified`

**Token Lifecycle:**
```
1. Login → Generate token pair (access + refresh)
2. API requests → Validate access token via JWKS
3. Token expiry → Use refresh token to get new pair
4. Logout → Add JTI to denylist
5. Global logout → Update user.last_global_signout_at
```

**Validation Chain:**
1. **Signature Verification:** Against IAM JWKS endpoint
2. **Expiry Check:** Token not expired
3. **Denylist Check:** JTI not in revocation list
4. **Global Signout Check:** Token issued after last global signout
5. **Tenant Context:** Validate tenant membership and authorities

### Multi-Tenant Security

**Tenant Isolation:**
- **Database Level:** PostgreSQL schema isolation with `search_path` switching
- **Application Level:** Tenant context validation on every request
- **API Level:** Gateway strips and re-injects tenant headers to prevent spoofing

**Authorization Model:**
```
User → TenantMembership → Authorities (per tenant)
     ↓
   TENANT_OWNER: Full tenant management
   ADMIN: User management, invitations
   MEMBER: Basic access
```

**Cross-Tenant Protection:**
- **Context Validation:** Every database operation validates tenant context
- **Header Sanitization:** Gateway prevents client-supplied tenant headers
- **Schema Isolation:** Database-level isolation prevents cross-tenant queries

---

## Infrastructure as Code

### Helm Chart Architecture

Each service has a dedicated Helm chart with environment-specific configurations:

```
helm-charts/IQKV/
├── foundation-iam-service/
│   ├── Chart.yaml
│   ├── values.yaml                    # Default values
│   ├── values-local.yaml             # Local development
│   ├── values-sit.yaml               # System integration testing
│   ├── values-uat.yaml               # User acceptance testing
│   └── values-prd.yaml               # Production
├── foundation-gateway-service/
├── foundation-billing-service/
└── foundation-ui-mantine-app-portal/
```

### Deployment Strategy

**Service Independence:**
- Each service deploys independently with its own release cycle
- Shared infrastructure (PostgreSQL, RabbitMQ) managed separately
- Configuration via Helm values and Kubernetes secrets

**Example Deployment:**
```bash
# Deploy IAM service to production
helm upgrade --install foundation-iam-service ./foundation-iam-service \
  --values ./values.yaml \
  --values ./values-prd.yaml \
  --set secrets.database.password=$PG_PASSWORD \
  --set secrets.rabbitmq.password=$RMQ_PASSWORD \
  --set secrets.jwt.privateKey=$JWT_PRIVATE_KEY \
  --namespace iqkv-prd \
  --atomic --wait --timeout=10m
```

**Configuration Management:**
- **Secrets:** Kubernetes secrets for sensitive data (passwords, keys)
- **ConfigMaps:** Non-sensitive configuration (URLs, timeouts, feature flags)
- **Environment Variables:** Runtime configuration injection
- **Helm Values:** Environment-specific overrides

### CI/CD Pipeline Integration

**Pipeline Stages:**
1. **Build & Test:** Maven build, unit tests, integration tests
2. **Security Scan:** Dependency vulnerability scanning
3. **Image Build:** Docker multi-stage builds with layer caching
4. **Deploy to Staging:** Automated deployment for testing
5. **Production Deploy:** Manual approval gate with automated rollout
6. **Health Checks:** Post-deployment validation and monitoring

**Pipeline Configuration:**
```yaml
# .drone.yml example
kind: pipeline
name: foundation-iam-service

steps:
- name: test
  image: maven:3.9-eclipse-temurin-25
  commands:
  - mvn clean verify -Dcheckstyle.skip=false

- name: build-image
  image: plugins/docker
  settings:
    repo: iqkv/foundation-iam-service
    tags: [latest, ${DRONE_COMMIT_SHA:0:8}]

- name: deploy-staging
  image: alpine/helm:latest
  commands:
  - helm upgrade --install foundation-iam-service ./charts/foundation-iam-service
```

---

## Observability & Operations

### Monitoring Stack

**Metrics Collection:**
- **Micrometer + Prometheus:** Application metrics (JVM, HTTP, custom business metrics)
- **Grafana Dashboards:** Pre-configured dashboards for each service
- **Alerting:** Prometheus AlertManager with Slack/email notifications

**Key Metrics:**
- **Authentication:** Login success/failure rates, token validation latency
- **Tenant Provisioning:** Provisioning success rate, time to active, stuck tenant count
- **API Gateway:** Request throughput, response times, error rates by service
- **Database:** Connection pool usage, query performance, schema count

**Logging Strategy:**
- **Structured JSON Logs:** Logstash encoder for consistent log format
- **Correlation IDs:** Request tracing across service boundaries
- **Log Aggregation:** Centralized logging with ELK stack or similar
- **Log Levels:** Configurable per service and package

### Health Checks & Readiness

**Spring Boot Actuator:**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
```

**Health Check Hierarchy:**
- **Liveness:** Service is running and not deadlocked
- **Readiness:** Service can handle traffic (database connected, dependencies available)
- **Custom Checks:** Platform mode consistency, tenant provisioning capacity

**Operational Endpoints:**
- `GET /actuator/health` - Kubernetes liveness/readiness probes
- `GET /actuator/info` - Service version, build info, platform mode
- `GET /actuator/metrics` - Application metrics
- `GET /actuator/prometheus` - Prometheus scrape endpoint

---

## Extension Model & Future Architecture

### Event-Driven Extensions

Core services publish to a versioned RabbitMQ exchange, enabling extensions without core code modifications:

```
Core Services (IAM, Gateway, Billing)
    ↓ (publishes events)
Platform Exchange (iqkv.events)
    ↓ (routes to)
├── Core Workers (tenant provisioning, billing sync)
└── Extensions (SAML SSO, analytics, audit logging, etc.)
```

**Extension Patterns:**
- **Event Subscribers:** React to platform events without modifying core services
- **API Extensions:** Additional REST endpoints via separate services
- **UI Extensions:** Micro-frontend architecture for additional features
- **Webhook Extensions:** External system integrations via webhook consumers

**Versioning Strategy:**
- **Event Schema Versioning:** Backward-compatible event schema evolution
- **API Versioning:** Semantic versioning for REST APIs
- **Database Migrations:** Forward-only Liquibase migrations
- **Service Contracts:** OpenAPI specifications for service interfaces

### Scalability Considerations

**Horizontal Scaling:**
- **Stateless Services:** All services are stateless and horizontally scalable
- **Database Scaling:** Read replicas, connection pooling, query optimization
- **Message Queue Scaling:** RabbitMQ clustering for high availability
- **Caching Strategy:** Redis for session storage and frequently accessed data

**Performance Optimization:**
- **Connection Pooling:** HikariCP for database connections
- **Async Processing:** Non-blocking I/O with WebFlux where appropriate
- **Batch Processing:** Bulk operations for data-intensive tasks
- **CDN Integration:** Static asset delivery via CDN

The event schema serves as the public API contract, allowing core internals to evolve freely while maintaining extension compatibility.
