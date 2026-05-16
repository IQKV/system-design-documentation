# Capabilities

Status key: ✅ implemented · 🚧 partial · 📋 planned

Repositories: `foundation-iam-service`, `foundation-gateway-service`, `foundation-billing-service`, `foundation-ui-app` (tenant), `foundation-ui-platform-admin` (operator), `foundation-microservice-project-layout` (service template).

---

## IAM

Identity, access, and tenant lifecycle. All auth flows pass through this service.

| Capability           | Notes                                                                                                                                                                                                 | Status |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Signup               | Email/password with self-service tenant creation; email verification required before access; supports both multi-tenant (new tenant per signup) and single-tenant (join default tenant) modes         | ✅     |
| Signup status        | `GET /auth/signup/status/{tenantKey}` — poll tenant provisioning until `ACTIVE` after signup                                                                                                          | ✅     |
| Authentication       | JWT RS256 access token (15 min) + refresh token (7 day); tokens carry user context and tenant membership; JJWT library with custom claims                                                             | ✅     |
| Platform admin auth  | `POST /auth/admin/signin` and `/auth/admin/refresh` — platform-scoped token pair (`tenant_id` null); operator self-service at `/auth/admin/me`                                                        | ✅     |
| Account recovery     | Password reset via signed email token (1h TTL), rate-limited (3 requests per 15min window); Thymeleaf email templates                                                                                 | ✅     |
| Brute-force lockout  | Failed login tracking per email; temporary account lock after 5 attempts for 15 minutes; automatic cleanup of expired lockout records                                                                 | ✅     |
| Token revocation     | JTI denylist (single session) + global signout timestamp; both access and refresh tokens validated against denylist; automatic cleanup of expired denylist entries                                    | ✅     |
| JWKS endpoint        | `/.well-known/jwks.json` — gateway and downstream services validate RS256 tokens locally; public key rotation support                                                                                 | ✅     |
| Organizations        | Create, update, suspend, delete; async provisioning via RabbitMQ with ShedLock-guarded reaper for stuck tenants; automatic retry mechanism for failed provisioning                                    | ✅     |
| RBAC                 | Tenant authorities: `TENANT_OWNER`, `ADMIN`, `MEMBER`; platform authority: `PLATFORM_ADMIN`; per-tenant membership with independent authorities; `@PreAuthorize` on endpoints                         | ✅     |
| Platform admin APIs  | Paginated admin CRUD for users and tenants; cross-tenant invitations; member authority management; force-set password                                                                                 | ✅     |
| Invitations          | Email invite with 72h expiring token; `authority` defaults to `MEMBER` (also `ADMIN`); new users created on accept (email pre-verified); existing users verified by password; ShedLock-guarded reaper | ✅     |
| Multi-org            | One user can belong to multiple organizations with different authorities; tenant discovery by credentials; cross-tenant user context switching                                                        | ✅     |
| Rollout mode         | `MULTI_TENANT` (default) or `SINGLE_TENANT` — configured via `platform.rollout-mode`; single-tenant provisions one default tenant at startup; mode consistency enforced across services               | ✅     |
| Events               | Publishes `tenant.created`, `tenant.provisioned`, `tenant.suspended`, `user.invited`, `user.removed` via RabbitMQ; Billing consumes provisioning events                                               | ✅     |
| Schema isolation     | PostgreSQL schema-per-tenant with `t_` prefix; MyBatis interceptor for automatic schema switching; Liquibase migrations per tenant; identical model in both single and multi-tenant modes             | ✅     |
| Email notifications  | Thymeleaf-rendered transactional emails (verification, password reset, invitations) via SMTP; MailHog for local dev                                                                                   | ✅     |
| Email verification   | Secure token-based email verification; resend capability with rate limiting; verification status tracking                                                                                             | ✅     |
| Token validation     | `POST /auth/validate` — introspection for gateway; validates signature, expiry, denylist, and global signout; returns user context                                                                    | ✅     |
| Tenant discovery     | `POST /users/tenants` — credential-gated discovery of tenant memberships (used by tenant UI sign-in)                                                                                                  | ✅     |
| Scheduled jobs       | ShedLock-protected background jobs: token denylist cleanup, invitation expiry, stuck tenant reaper, email verification cleanup                                                                        | ✅     |
| Bootstrap strategies | Pluggable tenant bootstrap for different rollout modes; default tenant resolution and creation; startup-time tenant provisioning for single-tenant mode                                               | ✅     |
| Observability        | Prometheus metrics, structured JSON logging with correlation IDs, health checks, actuator endpoints; `platform.rollout-mode` on `/actuator/info`                                                      | ✅     |

---

## API Gateway

Entry point for all client traffic. No request reaches IAM or Billing without passing through here.

| Capability            | Notes                                                                                                                                           | Status |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Routing               | Path-based routing to IAM and Billing; Spring Cloud Gateway with WebFlux; Stripe webhooks on public path                                        | ✅     |
| JWT validation        | Validates RS256 tokens against IAM JWKS endpoint; extracts authorities from JWT claims                                                          | ✅     |
| Header sanitization   | Strips client-supplied `X-User-*`, `X-Tenant-ID`, and `X-Organization-ID` before JWT processing — prevents identity spoofing                    | ✅     |
| Context propagation   | Extracts user context from JWT and forwards: `X-User-ID`, `X-Username`, `X-User-Email`, `X-User-Authorities`, `X-Tenant-ID`, `X-Correlation-ID` | ✅     |
| Tenant context filter | `SINGLE_TENANT`: auto-injects `default-tenant-key` when absent; `MULTI_TENANT`: tenant from JWT only                                            | ✅     |
| Public paths          | Configurable public endpoints (JWKS, auth, webhooks, health, Swagger UI); enforced by `SecurityConfig` + `GatewayProperties`                    | ✅     |
| Platform mode guard   | Polls IAM `/actuator/info` every 60s; blocks traffic with 503 on rollout-mode mismatch; fail-open if IAM unreachable                            | ✅     |
| Response security     | `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, `Referrer-Policy`; echoes correlation ID                                       | ✅     |
| CORS                  | Global CORS configuration with configurable origins, methods, and headers                                                                       | ✅     |
| Request logging       | Structured logs with correlation ID filter for request tracing                                                                                  | ✅     |
| Observability         | Prometheus metrics, health checks, and actuator endpoints on separate management port                                                           | ✅     |
| Swagger aggregation   | Aggregates API documentation from downstream services (IAM, Billing) in unified Swagger UI                                                      | ✅     |
| Metering events       | Publishes `api.request.metered` per request                                                                                                     | 📋     |

---

## Billing

Payment gateway abstraction with plan catalog and subscription state. Stripe is the active adapter — subscriptions, invoices, and payment collection are managed on the gateway side; this service maps tenants to customers and syncs webhook events.

| Capability             | Notes                                                                                                                              | Status |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Payment gateway port   | `PaymentGatewayPort` hexagonal abstraction; Stripe adapter implemented; additional gateways via strategy pattern                   | ✅     |
| Stripe integration     | Customer provisioning on `tenant.created` / provisioning events; webhook signature verification; subscription state cached locally | ✅     |
| Plan catalog           | Admin-managed plans (`planCode`, pricing, `featureSet`, `scope` TENANT/USER); tenant read APIs; soft-delete via `active=false`     | ✅     |
| Plan eligibility       | Validates plan `scope` matches rollout mode and subject type at subscription creation                                              | ✅     |
| Billing settings       | Per-tenant `billing_settings`: Stripe customer ID, billing email, company info, tax ID, billing address; syncs to Stripe on PATCH  | ✅     |
| Platform admin APIs    | Admin CRUD for billing settings, subscriptions, and plans under `/admin/*`                                                         | ✅     |
| User billing           | `user_billing_settings` for single-tenant mode; auto-created Stripe customer per user on first access                              | ✅     |
| Subscription cache     | Local cache of Stripe subscription state; updated via webhooks for fast reads without Stripe API calls                             | ✅     |
| Subject resolution     | `SubscriptionSubjectResolver` — TENANT-scoped (multi-tenant) vs USER-scoped (single-tenant)                                        | ✅     |
| Entitlement evaluation | `EntitlementEvaluator` — active subscription + plan `featureSet` for authorization decisions                                       | ✅     |
| Multi-mode support     | Supports both multi-tenant (tenant-scoped) and single-tenant (user-scoped) billing models                                          | ✅     |
| Webhook processing     | Idempotent Stripe webhook handling; subscription created/updated/deleted, invoice paid, payment failed                             | ✅     |
| REST API               | Settings, subscriptions (`/{tenantKey}` and `/me`), plan catalog; JWT + tenant isolation                                           | ✅     |
| Lifecycle events       | Publishes `subscription.created`, `subscription.cancelled`, `invoice.paid`, `payment.failed` via RabbitMQ                          | ✅     |
| Email notifications    | Billing notification types published to RabbitMQ (subscription, trial, invoice, payment events) — consumed by email worker         | ✅     |
| Scheduled jobs         | ShedLock-protected trial-ending (9 AM UTC) and payment-overdue (10 AM UTC) notification jobs                                       | ✅     |
| Tax compliance         | Tax ID/VAT/GST storage and Stripe metadata sync for B2B invoicing                                                                  | ✅     |
| Webhook idempotency    | `webhook_log` table tracks processed events; prevents duplicate processing                                                         | ✅     |
| Email resolution       | Multi-tenant: `billing_settings.billingEmail`; single-tenant: `user_billing_settings.billingEmail` with fallback chain             | ✅     |
| Observability          | Prometheus metrics, structured JSON logging with correlation IDs, health checks, actuator endpoints                                | ✅     |

---

## UI — Tenant app (`foundation-ui-app`)

React 19 + Mantine SPA for workspace members. All requests go through the API Gateway with tenant-scoped JWTs and `X-Tenant-ID`. Static build (Nginx or CDN).

| Screen / flow        | Notes                                                                                                      | Status |
| -------------------- | ---------------------------------------------------------------------------------------------------------- | ------ |
| Sign-in              | Credentials → tenant discovery (`POST /users/tenants`); multi-tenant picker; single-tenant direct sign-in  | ✅     |
| Sign-up              | Self-service registration + tenant creation; polls provisioning until `ACTIVE`                             | ✅     |
| Password reset       | Forgot-password email + token-based reset (`/forgot-password`, `/reset-password`)                          | ✅     |
| Email verification   | Token-based verification from email link (`/verify-email`)                                                 | ✅     |
| Accept invitation    | Public `/invite/:token` — new and existing users                                                           | ✅     |
| Dashboard            | Workspace name, welcome, team member count                                                                 | ✅     |
| Team — members       | Searchable member list                                                                                     | ✅     |
| Team — invitations   | Send, list, revoke pending invitations (`TENANT_OWNER`)                                                    | ✅     |
| My account           | Profile view/edit, change password, organizations and roles                                                | ✅     |
| Session security     | Access token in memory; refresh + tenant key in `sessionStorage`; silent refresh; 30min inactivity timeout | ✅     |
| i18n & theme         | Lingui (English catalog); locale cookie; light/dark theme                                                  | ✅     |
| Billing self-service | Stripe portal / subscription management in tenant UI                                                       | 📋     |
| Tenant settings      | Rename workspace, tenant configuration                                                                     | 📋     |
| Member role editing  | Change member authorities beyond invitation default                                                        | 📋     |

---

## UI — Platform admin (`foundation-ui-platform-admin`)

Separate operator SPA (`PLATFORM_ADMIN` only). Platform-scoped JWT (`tenant_id` null). Deployed internally or behind restricted ingress.

| Screen / flow         | Notes                                                                                           | Status |
| --------------------- | ----------------------------------------------------------------------------------------------- | ------ |
| Admin sign-in         | `POST /auth/admin/signin`; refresh via `/auth/admin/refresh`; `/unauthorized` without authority | ✅     |
| Dashboard             | Parallel count cards: users, organizations, active subscriptions                                | ✅     |
| Users                 | Paginated list; detail with Overview + Organizations tabs; edit profile; set password           | 🚧     |
| Organizations         | Paginated list with status filter; detail with Overview, Members, Billing tabs; edit metadata   | 🚧     |
| Invitations           | Cross-tenant list; propose, edit, revoke                                                        | ✅     |
| Subscriptions         | Global read-only list with search, status filter, sorting                                       | 🚧     |
| Plan catalog          | List, create, edit, delete (deactivate) plans                                                   | ✅     |
| Operator account      | View/edit operator profile; change password                                                     | ✅     |
| Session security      | Access token in memory; refresh in `sessionStorage`; silent refresh; inactivity sign-out        | ✅     |
| i18n                  | Lingui with English catalog; locale switcher UI                                                 | ✅     |
| Platform actions      | Ban/unban, unlock, impersonation                                                                | 📋     |
| System administration | Health, jobs, global audit log                                                                  | 📋     |
| Advanced metrics      | MRR/ARR, growth charts on dashboard                                                             | 📋     |

---

## Infrastructure

| Capability           | Notes                                                                                                                 | Status |
| -------------------- | --------------------------------------------------------------------------------------------------------------------- | ------ |
| Kubernetes           | Deployments with HPA, pod anti-affinity, network policies                                                             | 🚧     |
| Helm                 | Dedicated chart per service; env-specific value files (local/dev/test/staging/production)                             | 🚧     |
| Docker Compose       | Local dev: PostgreSQL, RabbitMQ, MailHog per service; `compose.container.yaml` for runtime stacks                     | ✅     |
| Database-per-service | Each service owns its own PostgreSQL database; no cross-service table access                                          | ✅     |
| Schema-per-tenant    | PostgreSQL schema isolation per tenant within the IAM database; identical model in both single and multi-tenant modes | ✅     |
| Async provisioning   | RabbitMQ event-driven; ShedLock-guarded reaper for tenants stuck in `PROVISIONING`                                    | ✅     |
| Service template     | `foundation-microservice-project-layout` — Spring Boot, MyBatis, Liquibase, RabbitMQ, JWT, quality gates pre-wired    | ✅     |
| Secrets management   | K8s Secrets injected at deploy time via CI pipeline — never committed to source                                       | 🚧     |
| TLS                  | cert-manager integration via Helm ingress values                                                                      | 🚧     |
| Observability        | Prometheus metrics (Micrometer), structured JSON logs (Logstash encoder), correlation ID filter                       | ✅     |
| CI/CD                | Drone pipelines per service: verify → publish artifacts → publish image → deploy → promote                            | 🚧     |

---

## Planned (Post-Demo)

Not in scope for v0.1. Will be delivered as extensions or core additions after the demo is stable.

| Capability                     | Notes                                                      |
| ------------------------------ | ---------------------------------------------------------- |
| SSO / SAML                     | Extension — subscribes to the event bus, not core          |
| Rate limiting                  | Per-tenant and per-user (gateway)                          |
| Tenant resolution by subdomain | Resolve tenant from request subdomain                      |
| Usage-based billing metering   | Metered billing on top of Stripe                           |
| Tenant billing UI              | Self-service billing portal in `foundation-ui-app`         |
| Platform operator actions      | Ban/unlock/impersonation in `foundation-ui-platform-admin` |
| Multi-region                   | Cross-region deployment support                            |
| Managed hosting                | Hosted version of the platform                             |
