# Implemented Features Review

> Snapshot of what is actually built and shipped across all platform components as of May 2026.
> Sources: `README.md` files from `foundation-iam-service`, `foundation-gateway-service`, `foundation-billing-service`, `foundation-ui-app`, `foundation-ui-platform-admin`, and `foundation-ui-saas-landing-kit`.

---

## Backend Services

### IAM Service (`foundation-iam-service`)

**Tech stack:** Java 25 / Spring Boot 4.1 · MyBatis 3.x · PostgreSQL 17 · Liquibase · RabbitMQ · JJWT 0.13 RS256 · ShedLock 7.x · Thymeleaf · Micrometer + Prometheus

#### Authentication

| Feature                 | Detail                                                                                                                                                |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Self-service signup     | Creates user + tenant in one call; returns `tenantKey`; tenant starts in `PROVISIONING`                                                               |
| Signup status polling   | `GET /auth/signup/status/{tenantKey}` — polls until `ACTIVE`                                                                                          |
| Tenant-scoped sign-in   | `POST /auth/signin` + `X-Tenant-ID`; returns RS256 access + refresh token pair                                                                        |
| Platform admin sign-in  | Separate `POST /auth/admin/signin`; issues platform-scoped token (`tenant_id = null`)                                                                 |
| Token refresh           | `POST /auth/refresh` and `POST /auth/admin/refresh` — rotates both tokens                                                                             |
| Single-session sign-out | `POST /auth/signout` — adds JTI to denylist                                                                                                           |
| Global sign-out         | `POST /auth/signout-all` — sets `last_global_signout_at`; invalidates all sessions                                                                    |
| Token validation        | `POST /auth/validate` — gateway introspection endpoint; checks denylist + global signout                                                              |
| Tenant discovery        | `POST /users/tenants` — credential-gated (no JWT); returns all active memberships                                                                     |
| RS256 JWT issuance      | Access tokens (15 min) and refresh tokens (7 days); claims include `userId`, `username`, `email`, `tenant_id`, `authorities`, `email_verified`, `jti` |
| JWKS endpoint           | `GET /.well-known/jwks.json` — public RSA key for downstream JWT validation                                                                           |

#### Multi-tenancy

| Feature                     | Detail                                                                                                                    |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Multi-tenant mode           | Each signup creates a new tenant; user gets `TENANT_OWNER` authority                                                      |
| Single-tenant mode          | Signup joins the pre-provisioned default tenant with `MEMBER` authority; no tenant creation                               |
| Per-tenant schema isolation | PostgreSQL schema-per-tenant (`t_{tenantKey}`); `MyBatisSchemaInterceptor` sets `search_path` per request                 |
| Tenant provisioning         | Async via RabbitMQ; `TenantProvisioningConsumer` runs Liquibase migrations, sets `ACTIVE`, publishes `tenant.provisioned` |
| Tenant lifecycle            | Status transitions: `PROVISIONING → ACTIVE ↔ SUSPENDED → DELETED`; `PROVISIONING_FAILED` with retry                       |
| Stuck tenant reaper         | ShedLock-guarded job every 5 min; marks stuck `PROVISIONING` tenants as `PROVISIONING_FAILED`                             |
| Platform rollout mode       | `ROLLOUT_MODE` env var (`MULTI_TENANT`                                                                                    | `SINGLE_TENANT`); validated at startup; published via `/actuator/info` |

#### User Management

| Feature                     | Detail                                                                                                                                   |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| User profile (self)         | `GET/PATCH /users/me` — view and update own profile                                                                                      |
| Password change (self)      | `POST /users/me/password` — requires current password; invalidates all sessions                                                          |
| Leave tenant                | `DELETE /users/me` — removes membership from current tenant                                                                              |
| Email verification          | Token-based; `POST /users/email/verify`; resend rate-limited (3/hour)                                                                    |
| Forgot / reset password     | `POST /users/password/forgot` (rate-limited, 3/15 min); `POST /users/password/reset` (1h TTL token); invalidates all sessions on success |
| Brute-force lockout         | Failed login attempts tracked per email; account locked after 5 attempts for 15 minutes                                                  |
| Platform admin self-service | `GET/PATCH /auth/admin/me` — view/update own operator profile; `POST /auth/admin/me/password` — change password                          |

#### Admin — User Management (`PLATFORM_ADMIN`)

| Endpoint                            | Description                                       |
| ----------------------------------- | ------------------------------------------------- |
| `GET /admin/users`                  | Paginated, filterable user list                   |
| `GET /admin/users/count`            | Total user count                                  |
| `GET /admin/users/{id}`             | Get user by UUID                                  |
| `POST /admin/users`                 | Create user with random temporary password        |
| `PUT /admin/users/{id}`             | Full user update                                  |
| `PATCH /admin/users/{id}`           | Partial user update                               |
| `DELETE /admin/users/{id}`          | Delete user and all memberships (cascade)         |
| `GET /admin/users/{id}/authorities` | Get user platform authorities                     |
| `PUT /admin/users/{id}/authorities` | Replace user platform authorities                 |
| `GET /admin/users/{id}/memberships` | Get user tenant memberships                       |
| `POST /admin/users/{id}/password`   | Force-set user password; invalidates all sessions |

#### Admin — Tenant Management (`PLATFORM_ADMIN`)

| Endpoint                                                      | Description                           |
| ------------------------------------------------------------- | ------------------------------------- |
| `GET /admin/tenants`                                          | Paginated, filterable tenant list     |
| `GET /admin/tenants/count`                                    | Total tenant count                    |
| `GET /admin/tenants/{tenantKey}`                              | Get tenant by key                     |
| `PUT /admin/tenants/{tenantKey}`                              | Rename tenant                         |
| `PATCH /admin/tenants/{tenantKey}`                            | Partially update tenant               |
| `DELETE /admin/tenants/{tenantKey}`                           | Delete tenant and all associated data |
| `GET /admin/tenants/{tenantKey}/members`                      | Paginated member list                 |
| `GET /admin/tenants/{tenantKey}/members/count`                | Total member count                    |
| `GET /admin/tenants/{tenantKey}/members/{userId}/authorities` | Get member's tenant authorities       |
| `PUT /admin/tenants/{tenantKey}/members/{userId}/authorities` | Replace member's tenant authorities   |

#### Invitations

| Feature                     | Detail                                                                                                                                      |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Send invitation             | `POST /tenants/{tenantKey}/invitations` — `TENANT_OWNER` or `ADMIN`; 72h TTL; default authority `MEMBER`                                    |
| List pending invitations    | `GET /tenants/{tenantKey}/invitations`                                                                                                      |
| Revoke invitation           | `DELETE /tenants/{tenantKey}/invitations/{id}`                                                                                              |
| Preview invitation          | `GET /invitations/{token}` — public; returns tenant name, authority, expiry, `requiresSignup` flag                                          |
| Accept invitation           | `POST /invitations/{token}/accept` — public; new users created with email pre-verified; existing users verify by password; returns JWT pair |
| Invitation reaper           | ShedLock-guarded hourly job; bulk-expires stale `PENDING` invitations                                                                       |
| Admin invitation management | `GET/POST/DELETE /admin/invitations` — cross-tenant list, propose, revoke (`PLATFORM_ADMIN`)                                                |

#### Observability & Infrastructure

- Prometheus metrics: `auth.success`, `auth.failure`, `tenant.created` counters; `auth.duration` timer
- Structured JSON logging with MDC correlation ID
- Health probes: `/actuator/health` includes `PlatformModeHealthIndicator`
- `/actuator/info` publishes canonical `platform.rollout-mode` (consumed by Gateway)
- ShedLock on all scheduled jobs (denylist cleanup, token reapers, stuck tenant reaper)

---

### Gateway Service (`foundation-gateway-service`)

**Tech stack:** Java 25 / Spring Boot 4.1 · Spring Cloud Gateway (WebFlux) · Spring Security OAuth2 Resource Server · Micrometer + Prometheus

#### Filter Chain

| Order           | Filter                         | Responsibility                                                                                           |
| --------------- | ------------------------------ | -------------------------------------------------------------------------------------------------------- |
| `-200`          | `CorrelationIdFilter`          | Generate or propagate `X-Correlation-ID`; store in MDC                                                   |
| `-190`          | `HeaderSanitizationFilter`     | Strip `X-User-*`, `X-Tenant-ID`, `X-Organization-ID` from incoming requests (prevents identity spoofing) |
| Spring Security | JWT validation                 | RS256 signature validation via JWKS from IAM                                                             |
| `-100`          | `JwtContextPropagationFilter`  | Extract claims from validated JWT; set downstream headers                                                |
| `-50`           | `TenantContextFilter`          | Multi-tenant: pass `X-Tenant-ID` from JWT; single-tenant: inject default tenant key if absent            |
| `MIN+1`         | `ResponseTransformationFilter` | Add security response headers; echo `X-Correlation-ID` to client                                         |

#### Downstream Headers (set after JWT validation)

`X-User-ID` · `X-Username` · `X-User-Email` · `X-User-Authorities` · `X-Tenant-ID` · `X-Correlation-ID`

#### Security Response Headers

`X-Content-Type-Options: nosniff` · `X-Frame-Options: DENY` · `X-XSS-Protection: 1; mode=block` · `Referrer-Policy: strict-origin-when-cross-origin`

#### Routes

| Route              | URI                                     | Auth                                                        |
| ------------------ | --------------------------------------- | ----------------------------------------------------------- |
| `iam-jwks`         | `/.well-known/**` → IAM                 | Public                                                      |
| `iam-api`          | `/api/v1/iam/**` → IAM                  | Configurable (public paths via `iqkv.gateway.public-paths`) |
| `billing-webhooks` | `/api/v1/billing/webhooks/**` → Billing | Public (Stripe signature)                                   |
| `billing-api`      | `/api/v1/billing/**` → Billing          | Protected (JWT required)                                    |

#### Platform Mode Guard

- `PlatformModeGuardFilter` polls IAM `/actuator/info` every 60s for `platform.rollout-mode`
- Mismatch → sets `ReadinessState.REFUSING_TRAFFIC`, returns 503 on all requests
- IAM unreachable → fail-open (logs warning, allows traffic)
- Aggregated Swagger UI at `/swagger-ui.html` proxies downstream `/api-docs` endpoints

---

### Billing Service (`foundation-billing-service`)

**Tech stack:** Java 25 / Spring Boot 4.1 · MyBatis 3.x · PostgreSQL 17 · Liquibase · RabbitMQ · Stripe Java SDK · ShedLock 7.x · Micrometer + Prometheus

#### Billing Settings

| Endpoint                                             | Auth             | Description                               |
| ---------------------------------------------------- | ---------------- | ----------------------------------------- |
| `GET /settings/{tenantKey}`                          | `TENANT_OWNER`   | Get billing settings                      |
| `PATCH /settings/{tenantKey}`                        | `TENANT_OWNER`   | Update billing settings (syncs to Stripe) |
| `GET /admin/tenants/{tenantKey}/billing-settings`    | `PLATFORM_ADMIN` | Get billing settings (admin)              |
| `POST /admin/tenants/{tenantKey}/billing-settings`   | `PLATFORM_ADMIN` | Create billing settings                   |
| `PUT /admin/tenants/{tenantKey}/billing-settings`    | `PLATFORM_ADMIN` | Replace billing settings                  |
| `PATCH /admin/tenants/{tenantKey}/billing-settings`  | `PLATFORM_ADMIN` | Partially update billing settings         |
| `DELETE /admin/tenants/{tenantKey}/billing-settings` | `PLATFORM_ADMIN` | Delete billing settings                   |

Billing settings fields: `externalCustomerId` (Stripe `cus_xxx`), `billingEmail`, `companyName`, `billingAddress` (JSONB), `taxId`, `taxIdType`, `currency`

#### Subscriptions

| Endpoint                                | Auth                       | Description                             |
| --------------------------------------- | -------------------------- | --------------------------------------- |
| `GET /subscriptions/{tenantKey}/active` | `TENANT_OWNER`             | Active subscription for tenant          |
| `GET /subscriptions/{tenantKey}`        | `TENANT_OWNER`             | All subscriptions for tenant            |
| `GET /subscriptions/me/active`          | `TENANT_OWNER` or `MEMBER` | Active subscription for current subject |
| `GET /subscriptions/me`                 | `TENANT_OWNER` or `MEMBER` | All subscriptions for current subject   |
| `GET /admin/subscriptions`              | `PLATFORM_ADMIN`           | Global paginated subscription list      |
| `GET /admin/subscriptions/count`        | `PLATFORM_ADMIN`           | Total subscription count                |
| `GET /admin/subscriptions/{id}`         | `PLATFORM_ADMIN`           | Get subscription by ID                  |
| `PATCH /admin/subscriptions/{id}`       | `PLATFORM_ADMIN`           | Partially update subscription           |
| `DELETE /admin/subscriptions/{id}`      | `PLATFORM_ADMIN`           | Delete subscription                     |

Subject resolution: multi-tenant → `TENANT / tenantKey`; single-tenant → `USER / userId`

#### Plan Catalog

| Endpoint                         | Auth              | Description                                   |
| -------------------------------- | ----------------- | --------------------------------------------- |
| `GET /plans`                     | Any authenticated | List all active plans                         |
| `GET /plans/{planCode}`          | Any authenticated | Get plan by code                              |
| `GET /admin/plans`               | `PLATFORM_ADMIN`  | List all plans (including inactive)           |
| `GET /admin/plans/{planCode}`    | `PLATFORM_ADMIN`  | Get plan by code                              |
| `POST /admin/plans`              | `PLATFORM_ADMIN`  | Create plan                                   |
| `PUT /admin/plans/{planCode}`    | `PLATFORM_ADMIN`  | Replace plan                                  |
| `PATCH /admin/plans/{planCode}`  | `PLATFORM_ADMIN`  | Partially update plan                         |
| `DELETE /admin/plans/{planCode}` | `PLATFORM_ADMIN`  | Deactivate plan (soft-delete, `active=false`) |

Plan fields: `planCode`, `displayName`, `billingPeriod` (MONTHLY|ANNUAL), `priceMinor` (cents), `currency`, `scope` (TENANT|USER), `featureSet` (JSON), `active`

#### Stripe Webhook Processing

- `POST /webhooks/stripe` — public; verifies `Stripe-Signature`; idempotent (deduplicates via `webhook_log`)
- Handles: `customer.subscription.created/updated/deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`
- Publishes: `subscription.created`, `subscription.cancelled`, `invoice.paid`, `payment.failed` to platform event bus

#### Event-Driven Integrations

- Consumes `tenant.provisioned` → creates Stripe customer; stores `external_customer_id` in `billing_settings`
- Consumes `tenant.suspended` → sends `ACCOUNT_SUSPENDED` notification
- Consumes `user.removed` / `user.deleted` → clears `profileOwnerId` on billing settings
- Publishes `subscription.cancelled` → IAM suspends tenant via `SubscriptionEventConsumer`

#### Scheduled Jobs

- Daily 9AM UTC: trial-ending notifications (2–3 days before expiry)
- Daily 10AM UTC: payment-overdue notifications for `past_due` subscriptions
- Both guarded by ShedLock

#### Payment Gateway Abstraction

- `PaymentGatewayPort` interface decouples business logic from gateway SDKs
- Stripe is the active implementation (`PAYMENT_GATEWAY_TYPE=STRIPE`)
- Strategy pattern allows future gateway additions without business logic changes

---

## Frontend Applications

### Tenant App (`foundation-ui-app`)

**Tech stack:** React 19 · TypeScript 6 · Vite 8 (SWC) · Mantine UI 9 · TanStack Router (file-based) · TanStack Query · Zustand · React Hook Form + Zod · Lingui 6 · Axios · Vitest + Playwright · OxLint / OxFmt

**Architecture:** Feature-Sliced Design (`app → processes → pages → widgets → features → shared`); boundary tests via `pnpm test:arch`

#### Implemented Features

| Area                      | Status            | Detail                                                                                                                    |
| ------------------------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Sign-in                   | ✅ Done           | Two-step: credentials → tenant discovery; multi-tenant users pick workspace; single-tenant signs in directly              |
| Sign-up                   | ✅ Done           | Self-service registration + tenant creation; polls provisioning status until `ACTIVE`                                     |
| Forgot / reset password   | ✅ Done           | Email flow + token-based reset (`/forgot-password`, `/reset-password`)                                                    |
| Email verification        | ✅ Done           | Token-based page (`/verify-email?token=…`)                                                                                |
| Accept invitation         | ✅ Done           | `/invite/:token` — works for new and existing users                                                                       |
| Dashboard                 | ✅ Done (basic)   | Workspace name, welcome message, team member count                                                                        |
| Team — member list        | ✅ Done           | Searchable member list                                                                                                    |
| Team — invitations        | ✅ Done           | Send, list, revoke (TENANT_OWNER only)                                                                                    |
| Profile & change password | ✅ Done           | View/edit name, change password, organizations and roles                                                                  |
| Session security          | ✅ Done           | Access token in memory; refresh token + tenant key in `sessionStorage`; silent refresh on 401; 30-min inactivity sign-out |
| Light/dark theme          | ✅ Done           | Persisted via Zustand                                                                                                     |
| i18n                      | ✅ Done (en only) | Lingui 6 PO catalogs; locale cookie; `Accept-Language` on API requests                                                    |

#### Not Yet Implemented

- Billing and subscription self-service
- Tenant/workspace settings (rename, configuration)
- Member role editing beyond invitation authority
- Additional locales (infrastructure ready; only `en` compiled)

#### Routes

`/sign-in` · `/signup` · `/forgot-password` · `/reset-password` · `/verify-email` · `/invite/:token` · `/` (dashboard) · `/team` · `/account` · `/unauthorized` · `/404` · `/500`

#### Authorization Model

| Role           | Capabilities                                         |
| -------------- | ---------------------------------------------------- |
| `TENANT_OWNER` | Invite members, view/revoke pending invitations      |
| `ADMIN`        | Invitable role; no extra UI beyond member list today |
| `MEMBER`       | Dashboard, team member list, own account             |

---

### Platform Admin (`foundation-ui-platform-admin`)

**Tech stack:** React 19 · TypeScript · Vite + SWC · Mantine UI 9 · mantine-datatable · TanStack Router · TanStack Query · Zustand · Lingui · Zod + Mantine Form · Vitest + Playwright · OxLint / OxFmt

**Architecture:** FSD-style layers (`app → processes → pages → features → shared`); boundary tests via `pnpm test:arch`

#### Implemented Features

| Area                | Status            | Detail                                                                                                                      |
| ------------------- | ----------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Sign-in & session   | ✅ Done           | `PLATFORM_ADMIN` credentials; access token in memory; refresh token in `sessionStorage`; silent refresh; inactivity timeout |
| Dashboard           | ✅ Done           | Parallel count cards: total users, organizations, active subscriptions (per-card loading/error states)                      |
| User list           | ✅ Done           | Paginated, sortable, filterable                                                                                             |
| User detail         | 🔶 Partial        | Overview tab + Organizations tab; edit profile; set password                                                                |
| Organization list   | ✅ Done           | Paginated with status filter                                                                                                |
| Organization detail | 🔶 Partial        | Overview tab, Members tab, Billing settings tab; edit metadata                                                              |
| Invitations         | ✅ Done           | List with filters; propose, edit, revoke                                                                                    |
| Subscriptions       | 🔶 Partial        | Read-only global list with search, status filter, sorting                                                                   |
| Plan catalog        | ✅ Done           | List; create plan; plan detail with edit and delete                                                                         |
| Operator account    | ✅ Done           | View/edit profile; change password                                                                                          |
| i18n                | ✅ Done (en only) | Lingui; locale switcher UI                                                                                                  |
| Runtime config      | ✅ Done           | Override `VITE_*` via `public/config.js` without rebuild                                                                    |

#### Not Yet Implemented

- Platform actions: ban/unban, unlock, impersonation
- Subscription lifecycle mutations (change plan, cancel, reactivate, apply discount)
- System health / background jobs monitoring
- Global audit log
- Advanced dashboard metrics (MRR/ARR, growth charts, trends)
- Multi-tab user/org detail views (auth history, billing, activity, notes)

#### Routes

`/sign-in` · `/unauthorized` · `/admin` (dashboard) · `/admin/users` · `/admin/users/:userId` · `/admin/organizations` · `/admin/organizations/:tenantKey` · `/admin/organizations/:tenantKey/members` · `/admin/organizations/:tenantKey/billing` · `/admin/invitations` · `/admin/subscriptions` · `/admin/plans` · `/admin/plans/:planCode` · `/admin/account`

---

### SaaS Landing Kit (`foundation-ui-saas-landing-kit`)

**Tech stack:** Astro · React · Tailwind CSS · shadcn/ui · Zustand · TypeScript

#### Implemented Features

| Feature               | Detail                                                                                                   |
| --------------------- | -------------------------------------------------------------------------------------------------------- |
| Static pages          | Home, Features, Pricing, About                                                                           |
| Responsive layout     | Mobile-first; `BaseLayout.astro` base template                                                           |
| Auth-aware navigation | `TopNav.tsx` shows Login/Sign Up when unauthenticated; user menu with avatar + logout when authenticated |
| Auth state            | Zustand store with `localStorage` persistence                                                            |
| React islands         | Partial hydration for interactive components                                                             |
| Code quality          | OxLint, OxFmt, Husky pre-commit hooks, commitlint                                                        |

---

## Cross-Cutting Platform Capabilities

### Security

| Capability                   | Implementation                                                                                      |
| ---------------------------- | --------------------------------------------------------------------------------------------------- |
| RS256 JWT                    | JJWT 0.13; private key in IAM; public key distributed via JWKS                                      |
| Token revocation             | JTI denylist (per-signout) + `last_global_signout_at` (global signout)                              |
| Identity spoofing prevention | Gateway strips all `X-User-*` / `X-Tenant-ID` headers before JWT propagation                        |
| Brute-force protection       | Per-email failed attempt tracking; 5-attempt threshold; 15-min lockout                              |
| Password policy              | Lowercase + uppercase + digit + special char; 8–128 chars                                           |
| Security response headers    | `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, `Referrer-Policy` on all responses |

### Messaging (RabbitMQ)

| Event                           | Publisher | Consumer(s)                            |
| ------------------------------- | --------- | -------------------------------------- |
| `tenant.created`                | IAM       | Billing (create Stripe customer)       |
| `tenant.provisioned`            | IAM       | Billing (send activation notification) |
| `tenant.provisioning_failed`    | IAM       | —                                      |
| `tenant.suspended`              | IAM       | Billing (send suspension notification) |
| `tenant.deleted`                | IAM       | —                                      |
| `user.created`                  | IAM       | —                                      |
| `user.invited`                  | IAM       | —                                      |
| `user.removed` / `user.deleted` | IAM       | Billing (clear `profileOwnerId`)       |
| `subscription.created`          | Billing   | —                                      |
| `subscription.cancelled`        | Billing   | IAM (suspend tenant)                   |
| `invoice.paid`                  | Billing   | —                                      |
| `payment.failed`                | Billing   | —                                      |

All queues: 24h TTL + dead-letter exchange (`iqkv.dlx`). Topic exchange: `iqkv.events`.

### Observability

- Prometheus metrics on all three services (`/actuator/prometheus`)
- Structured JSON logging with Logstash encoder + MDC correlation ID
- Grafana dashboards (bundled in `docker/grafana/`) for each service
- Health probes (`/actuator/health`) with custom indicators (`PlatformModeHealthIndicator`)
- Swagger UI on all backend services; Gateway aggregates downstream specs

### Quality Tooling (all repos)

- Checkstyle (Java) · JaCoCo coverage gate · ArchUnit architecture tests
- OxLint + OxFmt (TypeScript/React) · Stylelint · Knip (unused exports)
- Husky pre-commit hooks · commitlint · Dependabot
- 8 issue templates · 20 issue labels per repo

---

## What Is Not Yet Implemented

### Backend

- User ban/unban/unlock/impersonation endpoints
- Tenant suspend/unsuspend/transfer-ownership via `PLATFORM_ADMIN` (only `TENANT_OWNER` paths exist today)
- Subscription lifecycle mutations via admin API (change plan, cancel, reactivate, apply discount, extend trial)
- Platform metrics dashboard endpoint (`/admin/dashboard/metrics`)
- Global audit log
- System health / background job management endpoints
- GDPR data export endpoints

### Frontend — Tenant App

- Billing and subscription self-service UI
- Tenant/workspace settings page
- Member role editing UI
- Additional locales beyond English

### Frontend — Platform Admin

- Platform action modals (ban, unlock, impersonation)
- Subscription mutation UI
- System administration section (health, jobs, audit log)
- Advanced dashboard metrics (MRR/ARR, growth charts)
- Additional user/org detail tabs (auth history, billing, activity, notes)
