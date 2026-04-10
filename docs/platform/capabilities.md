# Capabilities

Status key: ✅ implemented · 🚧 in progress · 📋 planned

---

## IAM

Identity, access, and tenant lifecycle. All auth flows pass through this service.

| Capability          | Notes                                                                              | Status |
| ------------------- | ---------------------------------------------------------------------------------- | ------ |
| Registration        | Email/password, email verification required before access                          | 🚧     |
| Authentication      | JWT RS256 access token (15 min) + refresh token (7 day)                            | 🚧     |
| Account recovery    | Password reset via signed email token, rate-limited                                | 🚧     |
| Brute-force lockout | Failed login tracking per email; temporary account lock                            | 🚧     |
| Token revocation    | JTI denylist (single session) + global signout timestamp                           | 🚧     |
| JWKS endpoint       | `/.well-known/jwks.json` — downstream services validate locally                    | 🚧     |
| Organizations       | Create, update, suspend, delete; async provisioning via RabbitMQ                   | 🚧     |
| RBAC                | Roles: `TENANT_OWNER`, `ADMIN`, `MEMBER`                                           | 🚧     |
| Invitations         | Email invite with expiring token                                                   | 🚧     |
| Multi-org           | One user can belong to multiple organizations with independent roles               | 🚧     |
| Events              | Publishes `tenant.provisioned`, `tenant.suspended`, `user.invited`, `user.removed` | 🚧     |

---

## API Gateway

Entry point for all client traffic. No request reaches IAM or Billing without passing through here.

| Capability        | Notes                                                    | Status |
| ----------------- | -------------------------------------------------------- | ------ |
| Routing           | Path-based and header-based routing to upstream services | 🚧     |
| JWT validation    | Validates RS256 token on every request                   | 🚧     |
| Tenant resolution | Resolved from token claim or request header              | 🚧     |
| Request logging   | Structured logs: tenant, latency, upstream, status code  | 🚧     |
| Metering events   | Publishes `api.request.metered` per request              | 📋     |

---

## Billing

Stripe Connect wrapper. No custom billing logic — subscriptions, invoices, and the dashboard are managed on Stripe's side.

| Capability        | Notes                                                                                        | Status |
| ----------------- | -------------------------------------------------------------------------------------------- | ------ |
| Stripe customer   | Created per tenant automatically on `tenant.provisioned`                                     | 🚧     |
| Billing settings  | 1:1 per tenant — owns Stripe customer metadata, decoupled from IAM users                    | 🚧     |
| Billing email     | Separate contact for finance dept; no system account required                                | 🚧     |
| Tax ID / VAT/GST  | Stored in `billing_settings`, synced to Stripe for compliant B2B invoices                   | 🚧     |
| Subscriptions     | Managed in Stripe Dashboard; no custom subscription logic                                    | 🚧     |
| Invoices          | Generated and hosted by Stripe                                                               | 🚧     |
| Webhooks          | Processed idempotently; duplicate delivery is safe                                           | 🚧     |
| Lifecycle events  | Publishes `subscription.created`, `subscription.cancelled`, `invoice.paid`, `payment.failed` | 🚧     |

---

## UI

React + Mantine SPA. All requests go through the API Gateway — no direct access to backend services. Deployed as a static build (Nginx or CDN).

| Screen             | Notes                                                      | Status |
| ------------------ | ---------------------------------------------------------- | ------ |
| Sign up / login    | Email/password, email verification flow                    | 🚧     |
| Password reset     | Token-based recovery flow                                  | 🚧     |
| Dashboard          | Org overview, active members, subscription status          | 🚧     |
| Organization setup | Create org, invite members, assign roles                   | 🚧     |
| Member management  | List members, change roles, revoke access                  | 🚧     |
| Account settings   | Profile, password change                                   | 🚧     |
| Billing portal     | Link to Stripe-hosted dashboard for subscriptions/invoices | 🚧     |

---

## Infrastructure

| Capability           | Notes                                                                                           | Status |
| -------------------- | ----------------------------------------------------------------------------------------------- | ------ |
| Kubernetes           | Deployments with HPA, pod anti-affinity, network policies                                       | 🚧     |
| Helm                 | Dedicated chart per service; env-specific value files (local/dev/test/staging/production)       | 🚧     |
| Docker Compose       | Local dev environment with PostgreSQL, RabbitMQ, MailHog                                        | 🚧     |
| Database-per-service | Each service owns its own PostgreSQL database; no cross-service table access                    | ✅     |
| Schema-per-tenant    | PostgreSQL schema isolation per tenant within the IAM database                                  | ✅     |
| Async provisioning   | RabbitMQ event-driven; ShedLock-guarded reaper for tenants stuck in `PROVISIONING`              | 🚧     |
| Secrets management   | K8s Secrets injected at deploy time via CI pipeline — never committed to source                 | 🚧     |
| TLS                  | cert-manager integration via Helm ingress values                                                | 🚧     |
| Observability        | Prometheus metrics (Micrometer), structured JSON logs (Logstash encoder), correlation ID filter | 🚧     |
| CI/CD                | Drone pipelines per service: verify → publish artifacts → publish image → deploy → promote      | 🚧     |

---

## Planned (Post-Demo)

Not in scope for v0.1. Will be delivered as extensions or core additions after the demo is stable.

| Capability                     | Notes                                             |
| ------------------------------ | ------------------------------------------------- |
| SSO / SAML                     | Extension — subscribes to the event bus, not core |
| Rate limiting                  | Per-tenant and per-user                           |
| Tenant resolution by subdomain | Resolve tenant from request subdomain             |
| Usage-based billing metering   | Metered billing on top of Stripe                  |
| Admin panel                    | Platform-level tenant management UI               |
| Multi-region                   | Cross-region deployment support                   |
| Managed hosting                | Hosted version of the platform                    |
