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
- Login / logout, JWT access + refresh tokens
- Password reset via signed email token
- Organization management (create, suspend, delete)
- Member invitations, role assignment (owner / admin / member / viewer)
- Multi-org membership

Publishes: `tenant.provisioned`, `tenant.suspended`, `user.invited`, `user.removed`

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
├── public/          # platform registry (orgs, plans)
├── tenant_acme/
├── tenant_globex/
└── tenant_initech/
```

API Gateway sets `search_path` per request based on resolved tenant context. Cross-tenant queries are not possible in normal application flow.

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
1. POST /register
       │
2. IAM creates user + org  (status: PENDING)
       │
3. Publishes tenant.provisioned → RabbitMQ
       │
4. Returns HTTP 202
       │
       ├── Provisioning worker
       │     create schema
       │     run migrations
       │     seed defaults
       │     set org status: ACTIVE
       │
       └── Billing worker
             create Stripe customer
```

Workers retry with exponential backoff on failure.

---

## Infrastructure as Code

```
helm/
├── iam/           # standalone — includes PostgreSQL sub-chart
├── api-gateway/   # standalone — no stateful dependencies
├── billing/       # standalone — includes Stripe webhook config
└── ui/            # standalone — Nginx serving static React build
```

Deploy individually:

```bash
helm install iqscaffold-iam-service ./helm/iam -f iam-values.yaml
helm install iqscaffold-gateway-service ./helm/api-gateway -f gateway-values.yaml
helm install iqscaffold-billing-service ./helm/billing -f billing-values.yaml
helm install iqscaffold-ui-service ./helm/ui -f ui-values.yaml
```

Shared dependencies (PostgreSQL, RabbitMQ) are managed separately. Point each service at existing instances via connection string values.

---

## Extension Model

Core services publish to a versioned RabbitMQ exchange. Extensions subscribe without modifying core code.

```
  IAM, Gateway, Billing ──▶ platform exchange ──▶ core workers
                                                ──▶ extensions (SAML, analytics, etc.)
```

The event schema is the public API. Core internals can change freely as long as the schema is stable.
