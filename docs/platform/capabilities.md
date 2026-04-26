# Capabilities

Status key: ✅ implemented · 🚧 in progress · 📋 planned

---

## IAM

Identity, access, and tenant lifecycle. All auth flows pass through this service.

| Capability           | Notes                                                                                                                                                                                                       | Status |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Signup               | Email/password with self-service tenant creation; email verification required before access; supports both multi-tenant (new tenant per signup) and single-tenant (join default tenant) modes               | ✅     |
| Authentication       | JWT RS256 access token (15 min) + refresh token (7 day); tokens carry user context and tenant membership; JJWT library with custom claims                                                                   | ✅     |
| Account recovery     | Password reset via signed email token (1h TTL), rate-limited (3 requests per 15min window); Thymeleaf email templates with i18n support                                                                     | ✅     |
| Brute-force lockout  | Failed login tracking per email; temporary account lock after 5 attempts for 15 minutes; automatic cleanup of expired lockout records                                                                       | ✅     |
| Token revocation     | JTI denylist (single session) + global signout timestamp; both access and refresh tokens validated against denylist; automatic cleanup of expired denylist entries                                          | ✅     |
| JWKS endpoint        | `/.well-known/jwks.json` — gateway and downstream services validate RS256 tokens locally; public key rotation support                                                                                       | ✅     |
| Organizations        | Create, update, suspend, delete; async provisioning via RabbitMQ with ShedLock-guarded reaper for stuck tenants; automatic retry mechanism for failed provisioning                                          | ✅     |
| RBAC                 | Authorities: `TENANT_OWNER`, `ADMIN`, `MEMBER`; per-tenant membership with independent roles across organizations; authority-based endpoint protection                                                      | ✅     |
| Invitations          | Email invite with 72h expiring token; `authority` defaults to `MEMBER`; new users created on accept (email pre-verified); existing users verified by password; ShedLock-guarded reaper expires stale tokens | ✅     |
| Multi-org            | One user can belong to multiple organizations with different authorities; tenant discovery by credentials; cross-tenant user context switching                                                              | ✅     |
| Rollout mode         | `MULTI_TENANT` (default) or `SINGLE_TENANT` — configured via `platform.rolloutMode`; single-tenant provisions one default tenant at startup; mode consistency enforced across services                      | ✅     |
| Events               | Publishes `tenant.provisioned`, `tenant.suspended`, `user.invited`, `user.removed` via RabbitMQ for async processing; event-driven architecture for cross-service coordination                              | ✅     |
| Schema isolation     | PostgreSQL schema-per-tenant with `t_` prefix; MyBatis interceptor for automatic schema switching; Liquibase migrations per tenant; identical model in both single and multi-tenant modes                   | ✅     |
| Email notifications  | Thymeleaf-rendered transactional emails (verification, password reset, invitations) via SMTP with i18n support; configurable templates and localization                                                     | ✅     |
| Email verification   | Secure token-based email verification; resend capability with rate limiting; verification status tracking; required before account activation                                                               | ✅     |
| Token validation     | Introspection endpoint for gateway; validates token signature, expiry, denylist status, and global signout timestamp; returns user context for downstream services                                          | ✅     |
| Tenant discovery     | Endpoint to discover user's tenant memberships by credentials; supports multi-tenant user workflows; returns tenant keys and authorities                                                                    | ✅     |
| Scheduled jobs       | ShedLock-protected background jobs: token denylist cleanup, invitation expiry, stuck tenant reaper, email verification cleanup; distributed-safe execution                                                  | ✅     |
| Bootstrap strategies | Pluggable tenant bootstrap for different rollout modes; default tenant resolution and creation; startup-time tenant provisioning for single-tenant mode                                                     | ✅     |
| Observability        | Prometheus metrics, structured JSON logging with correlation IDs, health checks, actuator endpoints; comprehensive monitoring and debugging capabilities                                                    | ✅     |

---

## API Gateway

Entry point for all client traffic. No request reaches IAM or Billing without passing through here.

| Capability          | Notes                                                                                                                                    | Status |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Routing             | Path-based routing to upstream services (IAM, Billing); Spring Cloud Gateway with WebFlux                                                | ✅     |
| JWT validation      | Validates RS256 tokens against IAM JWKS endpoint; extracts authorities from JWT claims                                                   | ✅     |
| Tenant resolution   | Multi-mode: JWT claim in multi-tenant, auto-inject default tenant in single-tenant; enforces mode consistency with IAM service           | ✅     |
| Context propagation | Extracts user context from JWT and propagates as headers: `X-User-ID`, `X-Username`, `X-User-Email`, `X-User-Authorities`, `X-Tenant-ID` | ✅     |
| Header sanitization | Strips client-supplied `X-User-*` and `X-Tenant-ID` headers to prevent spoofing; runs before JWT context propagation                     | ✅     |
| Public paths        | Configurable public endpoints (JWKS, webhooks, health checks, Swagger UI); bypasses authentication                                       | ✅     |
| Platform mode guard | Validates rollout mode consistency with IAM service; blocks traffic on mismatch with 503 Service Unavailable                             | ✅     |
| CORS                | Global CORS configuration with configurable origins, methods, and headers                                                                | ✅     |
| Request logging     | Structured logs with correlation ID filter for request tracing                                                                           | ✅     |
| Observability       | Prometheus metrics, health checks, and actuator endpoints on separate management port                                                    | ✅     |
| Swagger aggregation | Aggregates API documentation from downstream services (IAM, Billing) in unified Swagger UI                                               | ✅     |
| Metering events     | Publishes `api.request.metered` per request                                                                                              | 📋     |

---

## Billing

Stripe integration layer with plan catalog and subscription management. Handles tenant-to-customer mapping, webhook processing, and entitlement evaluation.

| Capability         | Notes                                                                                                     | Status |
| ------------------ | --------------------------------------------------------------------------------------------------------- | ------ |
| Plan catalog       | Pre-provisioned subscription plans with pricing, features, and scope (TENANT/USER); CRUD via REST API     | ✅     |
| Plan eligibility   | Validates plan scope matches rollout mode (tenant vs user scoped plans)                                   | ✅     |
| Billing settings   | Per-tenant settings: Stripe customer ID, billing email, company info, tax ID, billing address             | ✅     |
| User billing       | Per-user billing settings for single-tenant mode; auto-created on first access                            | ✅     |
| Subscription cache | Local cache of Stripe subscription state; updated via webhooks for fast reads                             | ✅     |
| Entitlement eval   | Evaluates active subscriptions and plan features for authorization decisions                              | ✅     |
| Stripe integration | Customer provisioning, webhook processing with signature verification, outbox pattern for reliability     | ✅     |
| Multi-mode support | Supports both multi-tenant (tenant-scoped) and single-tenant (user-scoped) billing models                 | ✅     |
| Webhook processing | Idempotent Stripe webhook handling; processes subscription lifecycle events safely                        | ✅     |
| REST API           | Complete API for plans, settings, subscriptions with JWT auth and tenant isolation                        | ✅     |
| Lifecycle events   | Publishes `subscription.created`, `subscription.cancelled`, `invoice.paid`, `payment.failed` via RabbitMQ | ✅     |
| Tax compliance     | Tax ID/VAT/GST storage and Stripe sync for B2B invoicing                                                  | 🚧     |

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

| Capability           | Notes                                                                                                                 | Status |
| -------------------- | --------------------------------------------------------------------------------------------------------------------- | ------ |
| Kubernetes           | Deployments with HPA, pod anti-affinity, network policies                                                             | 🚧     |
| Helm                 | Dedicated chart per service; env-specific value files (local/dev/test/staging/production)                             | 🚧     |
| Docker Compose       | Local dev environment with PostgreSQL, RabbitMQ, MailHog                                                              | 🚧     |
| Database-per-service | Each service owns its own PostgreSQL database; no cross-service table access                                          | ✅     |
| Schema-per-tenant    | PostgreSQL schema isolation per tenant within the IAM database; identical model in both single and multi-tenant modes | ✅     |
| Async provisioning   | RabbitMQ event-driven; ShedLock-guarded reaper for tenants stuck in `PROVISIONING`                                    | 🚧     |
| Secrets management   | K8s Secrets injected at deploy time via CI pipeline — never committed to source                                       | 🚧     |
| TLS                  | cert-manager integration via Helm ingress values                                                                      | 🚧     |
| Observability        | Prometheus metrics (Micrometer), structured JSON logs (Logstash encoder), correlation ID filter                       | 🚧     |
| CI/CD                | Drone pipelines per service: verify → publish artifacts → publish image → deploy → promote                            | 🚧     |

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
