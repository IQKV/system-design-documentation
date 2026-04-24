# Architecture

## Tenancy & Isolation

The platform uses a **Hybrid Tenancy Model** that supports both public SaaS (Multi-Tenant) and internal/enterprise (Single-Tenant) deployments using the same codebase.

### Deployment Archetypes

- **Multi-Tenant (Default):** Every registration creates a new organization with a unique NanoID key.
- **Single-Tenant:** Tenancy is hidden. All users are automatically joined to a single "Default" tenant created during bootstrapping.

### Isolation Strategy

Isolation is handled via a **Tiered Model**:

1. **Logical (Standard):** Dedicated PostgreSQL schemas on a shared instance.
2. **Physical (Enterprise):** Dedicated PostgreSQL instances.

For details on the hybrid architecture, NanoID resolution, and bootstrapping, see the [Tenancy Deep Dive](./tenancy.md).

---

## Overview

```
  Client ──────────────────▶ API Gateway
                                  │
                    ┌─────────────┴──────────────┐
                    ▼                            ▼
                   IAM                        Billing
                    │                            │
                    ▼                            ▼
             PostgreSQL (iam)            PostgreSQL (billing)

                    └─────────────┬──────────────┘
                                  ▼
                              RabbitMQ
                           (lifecycle events)

  UI (React + Mantine) ──▶ API Gateway (all requests proxied)
```

---

## Services

### IAM

- User registration with email verification
- Login / logout, JWT RS256 access (15 min) + refresh (7 day) tokens
- Password reset via signed email token, brute-force lockout
- Tenant lifecycle (create, suspend, delete, retry provisioning)
- Member invitations, role assignment (`TENANT_OWNER` / `ADMIN` / `MEMBER`)
- Multi-tenant membership — one user, multiple tenants
- Token revocation: JTI denylist + global signout timestamp
- JWKS endpoint (`/.well-known/jwks.json`) for downstream token validation

Publishes: `tenant.provisioned`, `tenant.suspended`, `user.invited`, `user.removed`

Stack: Java 25, Spring Boot 4.0, MyBatis (no JPA), PostgreSQL, Liquibase, RabbitMQ, JJWT, ShedLock

---

### API Gateway

- Routes requests to IAM and Billing
- Validates JWT on every request
- Resolves tenant from token claim or header
- Structured request logging

Publishes: `api.request.metered`

---

### Billing

Stripe Connect wrapper. No custom billing logic — subscriptions, invoices, and the dashboard are managed on Stripe's side.

- Creates Stripe customer per tenant on `tenant.provisioned`
- Handles Stripe webhooks (idempotent)
- Stripe customer ID and subscription ID stored in `billing_settings`

Publishes: `subscription.created`, `subscription.cancelled`, `invoice.paid`, `payment.failed`, `billing.settings.updated`

### Billing Settings

Each tenant has a `billing_settings` record (1:1) — the single source of truth for Stripe customer metadata. Decouples billing identity from IAM users.

```
billing_settings
├── id                UUID PK
├── tenant_key        VARCHAR(21) UNIQUE FK → tenant
├── stripe_customer_id VARCHAR
├── billing_email     VARCHAR        -- finance dept contact, no system access required
├── company_name      VARCHAR
├── billing_address   JSONB          -- street, city, country, postal_code
├── tax_id            VARCHAR        -- VAT/GST number
├── tax_id_type       VARCHAR        -- Stripe enum: eu_vat, gb_vat, au_abn, etc.
├── currency          VARCHAR(3)     -- ISO 4217, default USD
├── profile_owner_id  BIGINT NULL FK → users(id)  -- optional, null by default
├── created_at        TIMESTAMP
└── updated_at        TIMESTAMP
```

**Key decisions:**

- Created automatically on `tenant.provisioned` with defaults from registration data
- Any update syncs to Stripe via `CustomerUpdateParams` (name, email, address, tax ID) — outbox pattern ensures delivery
- Owner/CEO changes in IAM do not affect billing identity
- `billing_email` allows finance teams to receive invoices without a system account
- VAT/GST details flow directly into Stripe invoices — required for B2B tax compliance
- `profile_owner_id` is nullable — billing settings are fully decoupled from users by default; optionally points to a user who "owns" the billing profile (e.g. CFO with a system account)

---

## UI

React SPA built with Mantine UI. Communicates exclusively through the API Gateway.

**Covers:**

- Auth flows — sign up, login, password reset, email verification
- Organization management — create org, invite members, manage roles
- Account settings — profile, password change
- Billing portal — links to Stripe-hosted dashboard for subscription and invoice management

Deployed as a static build (Nginx container or CDN). No direct database or service access.

---

## Data Layer

### PostgreSQL — Database-Per-Service

Each service owns its own PostgreSQL database. No shared database, no cross-service table access — inter-service data flows through the API or the event bus.

| Service | Database             | Contents                                              |
| ------- | -------------------- | ----------------------------------------------------- |
| IAM     | `foundation_iam`     | Users, organizations, memberships, roles, invitations |
| Billing | `foundation_billing` | Stripe customer refs, subscription IDs, webhook log   |

### Schema-Per-Tenant (within IAM database)

Within `foundation_iam`, each tenant gets a dedicated PostgreSQL schema:

```
foundation_iam/
├── public/          # platform registry (users, tenants, token_denylist, failed_logins, shedlock)
├── tenant_V1StGXR8_Z5j/     # per-tenant: members, authorities, tenant-scoped data
├── tenant_K9pL2mN7qR4s/
└── tenant_A3bC5dE7fG9h/
```

The `MyBatisSchemaInterceptor` rewrites `search_path` per request based on the resolved tenant context from the JWT claim. Cross-tenant queries are not possible in normal application flow.

To migrate a tenant to a dedicated database instance: dump schema → restore → update connection string in registry. No code changes.

### RabbitMQ — Event Bus

| Exchange      | Routing key              | Consumer            | Purpose                         |
| ------------- | ------------------------ | ------------------- | ------------------------------- |
| `iqkv.events` | `tenant.created`         | Billing             | Create payment gateway customer |
| `iqkv.events` | `tenant.created`         | Provisioning worker | Create schema, run migrations   |
| `iqkv.events` | `tenant.updated`         | Provisioning worker | Schema provisioning succeeded   |
| `iqkv.events` | `tenant.suspended`       | Billing             | Mark billing profile inactive   |
| `iqkv.events` | `user.removed`           | (extensions)        | Membership removed              |
| `iqkv.events` | `subscription.cancelled` | IAM                 | Suspend tenant                  |

---

## Tenant Provisioning Flow

### Multi-tenant (default)

```
1. POST /api/v1/iam/auth/signup
       │
2. IAM creates user + tenant  (status: PROVISIONING)
       │
3. Publishes tenant.provisioned → RabbitMQ (platform exchange)
       │
4. Returns HTTP 202  { tenantKey, status: "PROVISIONING" }
       │
       ├── Schema provisioning worker
       │     create tenant schema
       │     run Liquibase migrations
       │     seed defaults
       │     set tenant status: ACTIVE
       │
       └── Billing worker
             create Stripe customer
             store customer ID
```

### Single-tenant (deploy-time)

When `tenancy.mode: single`, IAM runs the same provisioning flow at application startup for the configured default tenant. Registration endpoint is disabled. All subsequent users are invited into the single tenant by the owner.

```
1. Application startup
       │
2. IAM checks if default tenant exists
       │
3. If not: creates tenant + owner account (status: PROVISIONING)
       │     → same async flow as multi-tenant
       │
4. If yes: no-op — idempotent startup
```

Workers retry with exponential backoff on failure. A ShedLock-guarded reaper job cleans up tenants stuck in `PROVISIONING` beyond a configurable timeout. Owners can manually trigger `POST /tenants/{tenantKey}/retry-provisioning` for `PROVISIONING_FAILED` tenants.

---

## Infrastructure as Code

Each service has a dedicated Helm chart. Shared infrastructure (PostgreSQL, RabbitMQ) is managed separately via `KnowHowDevOps/helm-charts/KnowHowDevOps/foundation-infra`.

```
KnowHowDevOps/helm-charts/IQKV/
├── foundation-iam-service/
├── foundation-gateway-service/
├── foundation-billing-service/
├── foundation-ui-mantine-app-portal/
```

Each chart ships environment-specific value files: `values.yaml` (defaults), `values-local.yaml`, `values-sit.yaml`, `values-uat.yaml`, `values-test.yaml`, `values-prd.yaml`.

Deploy individually — point each service at existing infrastructure instances via connection string values:

```bash
helm upgrade --install foundation-iam-service ./foundation-iam-service \
  --values ./values.yaml --values ./values-prd.yaml \
  --set infraServices.postgresql.password=$PG_PASSWORD \
  --set infraServices.rabbitmq.password=$RMQ_PASSWORD \
  --namespace iqkv-prd-env --atomic --wait
```

CI/CD pipelines (Drone) handle deployments automatically. See `KnowHowDevOps/homelab-operations-pipeline/IQKV/` for pipeline definitions per service.

---

## Extension Model

Core services publish to a versioned RabbitMQ exchange. Extensions subscribe without modifying core code.

```
  IAM, Gateway, Billing ──▶ platform exchange ──▶ core workers
                                                ──▶ extensions (SAML, analytics, etc.)
```

The event schema is the public API. Core internals can change freely as long as the schema is stable.
