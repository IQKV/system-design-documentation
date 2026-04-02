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

| Capability       | Notes                                             |
| ---------------- | ------------------------------------------------- |
| Registration     | Email/password, email verification required       |
| Authentication   | JWT access token + refresh token                  |
| Account recovery | Password reset via signed email token             |
| Organizations    | Create, update, suspend, delete                   |
| RBAC             | Roles: owner, admin, member, viewer               |
| Invitations      | Email invite with expiring token                  |
| Multi-org        | User can belong to multiple organizations         |
| Events           | Auth and membership changes published to RabbitMQ |

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

| Capability           | Notes                                                       |
| -------------------- | ----------------------------------------------------------- |
| Kubernetes           | Deployments with HPA                                        |
| Helm                 | Dedicated chart per service (iam, api-gateway, billing, ui) |
| Docker Compose       | Local dev environment                                       |
| Database-per-service | Each service has its own PostgreSQL database by default     |
| Schema-per-tenant    | PostgreSQL schema isolation per tenant, within IAM database |
| Async provisioning   | RabbitMQ event-driven                                       |
| Secrets              | K8s Secrets; external secret manager compatible             |
| TLS                  | cert-manager integration                                    |
| Observability        | Prometheus metrics, JSON logs, OpenTelemetry                |
