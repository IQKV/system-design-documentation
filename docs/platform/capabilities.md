# Capabilities

## UI

React + Mantine UI. All requests go through the API Gateway.

| Screen             | Notes                                     |
| ------------------ | ----------------------------------------- |
| Sign up / login    | Email/password, email verification flow   |
| Password reset     | Token-based recovery flow                 |
| Organization setup | Create org, invite members, assign roles  |
| Member management  | List members, change roles, revoke access |
| Account settings   | Profile, password change                  |
| Billing portal     | Link to Stripe-hosted dashboard           |

Deployed as a static build (Nginx or CDN). No direct access to backend services.

---

| Capability          | Notes                                                               |
| ------------------- | ------------------------------------------------------------------- |
| Registration        | Email/password, email verification required                         |
| Authentication      | JWT RS256 access token (15 min) + refresh token (7 day)             |
| Account recovery    | Password reset via signed email token, rate-limited                 |
| Brute-force lockout | Failed login tracking per email; temporary account lock             |
| Token revocation    | JTI denylist (single session) + global signout timestamp            |
| JWKS endpoint       | `/.well-known/jwks.json` — downstream services validate locally     |
| Organizations       | Create, update, suspend, delete; async provisioning via RabbitMQ    |
| RBAC                | Roles: `TENANT_OWNER`, `ADMIN`, `MEMBER`                            |
| Invitations         | Email invite with expiring token                                    |
| Multi-org           | User can belong to multiple organizations with different roles each |
| Events              | Auth and membership changes published to RabbitMQ                   |

## API Gateway

| Capability        | Notes                                            |
| ----------------- | ------------------------------------------------ |
| Routing           | Path-based and header-based to upstream services |
| Auth              | JWT validation on every request                  |
| Tenant resolution | From token claim or header                       |
| Logging           | Structured logs: tenant, latency, status         |

## Billing

Stripe Connect only. No custom billing logic.

| Capability       | Notes                                                            |
| ---------------- | ---------------------------------------------------------------- |
| Stripe customer  | Created per tenant on provisioning                               |
| Subscriptions    | Managed in Stripe Dashboard                                      |
| Invoices         | Generated and hosted by Stripe                                   |
| Webhooks         | Processed idempotently                                           |
| Lifecycle events | `subscription.cancelled`, `payment.failed` published to RabbitMQ |

## Infrastructure

| Capability           | Notes                                                                                           |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| Kubernetes           | Deployments with HPA, pod anti-affinity, network policies                                       |
| Helm                 | Dedicated chart per service; env-specific value files (dev/staging/production)                  |
| Docker Compose       | Local dev environment with PostgreSQL, RabbitMQ, MailHog                                        |
| Database-per-service | Each service has its own PostgreSQL database                                                    |
| Schema-per-tenant    | PostgreSQL schema isolation per tenant within IAM database                                      |
| Async provisioning   | RabbitMQ event-driven; ShedLock-guarded reaper for stuck tenants                                |
| Secrets              | K8s Secrets; injected at deploy time via Drone pipeline — never committed                       |
| TLS                  | cert-manager integration via Helm ingress values                                                |
| Observability        | Prometheus metrics (Micrometer), structured JSON logs (Logstash encoder), correlation ID filter |
| CI/CD                | Drone pipelines per service: verify → publish artifacts → publish image → deploy → promote      |
