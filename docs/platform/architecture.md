# Architecture

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

Stack: Java 21, Spring Boot 3.4, MyBatis (no JPA), PostgreSQL, Liquibase, RabbitMQ, JJWT, ShedLock

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
- Stores Stripe customer ID and subscription ID per tenant

Publishes: `subscription.created`, `subscription.cancelled`, `invoice.paid`, `payment.failed`

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
| IAM     | `iqscaffold_iam`     | Users, organizations, memberships, roles, invitations |
| Billing | `iqscaffold_billing` | Stripe customer refs, subscription IDs, webhook log   |

### Schema-Per-Tenant (within IAM database)

Within `iqscaffold_iam`, each tenant gets a dedicated PostgreSQL schema:

```
iqscaffold_iam/
├── public/          # platform registry (users, tenants, token_denylist, failed_logins, shedlock)
├── tenant_acme/     # per-tenant: members, authorities, tenant-scoped data
├── tenant_globex/
└── tenant_initech/
```

The `MyBatisSchemaInterceptor` rewrites `search_path` per request based on the resolved tenant context from the JWT claim. Cross-tenant queries are not possible in normal application flow.

To migrate a tenant to a dedicated database instance: dump schema → restore → update connection string in registry. No code changes.

### RabbitMQ — Event Bus

| Exchange   | Routing key              | Consumer            | Purpose                       |
| ---------- | ------------------------ | ------------------- | ----------------------------- |
| `platform` | `tenant.provisioned`     | Billing             | Create Stripe customer        |
| `platform` | `tenant.provisioned`     | Provisioning worker | Create schema, run migrations |
| `platform` | `subscription.cancelled` | IAM                 | Suspend org access            |

---

## Tenant Provisioning Flow

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
