# IQ Key Value Platform — Hybrid Tenancy SaaS Foundation

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://makeapullrequest.com)
[![Live Demo](https://img.shields.io/badge/Live-Demo-brightgreen.svg)](https://www.iqkv.site)
[![Website](https://img.shields.io/badge/Website-iqkv.dev-blue)](https://iqkv.dev)

---

K8s-native microservices foundation for building B2B SaaS and single-tenant applications. Unique hybrid tenancy model lets you deploy as multi-tenant SaaS or single-tenant app from the same codebase.

---

## Quick Start

Two ways to get started locally:

### Per-Service Development

Each service includes its own Docker Compose for dependencies (PostgreSQL, RabbitMQ, MailHog, MinIO):

```bash
git clone https://github.com/IQKV/foundation-iam-service.git
cd foundation-iam-service
cp .env.example .env.local
docker compose up -d  # Start infrastructure only
./mvnw spring-boot:run -Pdev
```

### Full Demo Stack

Run the entire platform with one command:

```bash
git clone https://github.com/IQKV/microservices-platform.git
cd microservices-platform
cp .env.example .env
./demo.sh  # Linux/macOS
# or
.\demo.ps1  # Windows
```

Then access:

- Tenant app: http://app.iqkv.local
- Platform admin: http://admin.iqkv.local
- API + Swagger: http://api.iqkv.local

---

## Status: v0.4 — Complete

**v0.4 — Identity Federation, Enterprise SSO & Multi-Gateway Billing** is complete. The strategic deliverable for this release is OAuth2/OIDC identity federation across IAM and both React frontends, with Lemon Squeezy support delivered in parallel as the simpler second billing adapter.

- **v0.1 (Demo Release):** Core microservices (IAM, Gateway, Billing), basic UI auth flows, tenant lifecycle, Stripe integration
- **v0.2 (Administration & Self-Service):** Platform admin UI, audit service, tenant self-service billing, announcements, in-app notifications, token exchange, avatar uploads, refunds API, WebSocket integration, Grafana dashboards
- **v0.3 (Plan Feature Access Control, i18n & Per-Seat Pricing):** Plan-based feature access control, Bulgarian (bg-BG) i18n translations, Spring Boot 4.1 upgrade, tenant user stats, read-only plan catalog, enterprise UI themes, E2E test improvements, SaaS landing kit plan selector, per-seat pricing (`PricingModel` FLAT/PER_SEAT, seat-cap validation, seat adjustment API, seatCount in subscription events, downstream consumer updates)
- **v0.4 (Identity Federation, Enterprise SSO & Multi-Gateway Billing) — complete:** OAuth2/OIDC social login, tenant-scoped enterprise SSO, account linking / unlinking, admin linked-identity remediation, tenant-app callback + security UI, platform-admin OIDC identities tab, Lemon Squeezy adapter, `@ConditionalOnGateway` wiring, gateway-neutral plan catalog config, LS webhook resource + event mapping, `gateway_type` schema columns

For detailed feature breakdowns, see:

- [Per-Seat Payment Implementation Record](docs/roadmap/11-implemented-per-seat-payments.md)
- [Plan Feature Access Control Implementation Summary](docs/roadmap/10-implemented-plan-feature-access-control-summary.md)
- [v0.4 Proposal: Lemon Squeezy Support](docs/roadmap/14-proposal-lemon-squeezy-support.md)
- [v0.4 Proposal: OAuth2 / OIDC Authentication](docs/roadmap/15-proposal-oauth2-oidc.md)
- [v0.4 Features Review (June 2026)](docs/roadmap/16-implemented-features-review.md)
- [Roadmap & Vision](docs/roadmap/vision.md)

---

## Key Features

| Feature                               | Details                                                                                                                                                                                               |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Hybrid Tenancy**                    | Single codebase supports both multi-tenant (B2B) and single-tenant (B2C) deployment modes                                                                                                             |
| **Schema-per-tenant Isolation**       | Each tenant gets its own PostgreSQL schema for complete data separation                                                                                                                               |
| **Production-ready IAM**              | JWT RS256 auth, RBAC, email verification, password reset, token revocation, invitations, plan code in JWT, personal workspace always accessible, OAuth2/OIDC federation, tenant SSO                   |
| **Plan Feature Access Control**       | YAML-based plan config, in-memory registry, JWT plan code propagation, max users quota enforcement, plan feature guard, user entitlements API                                                         |
| **Multi-Gateway Billing Integration** | Stripe and Lemon Squeezy support; subscriptions, invoices, refunds, Customer Portal, webhook handling, plan codes integration, per-seat pricing with seat-cap validation and mid-cycle adjustment API |
| **Centralized Audit Trail**           | Passive event consumption, SPI-based extensibility, admin search API, severity filters                                                                                                                |
| **Reactive API Gateway**              | JWT validation, header sanitization, audit context propagation, plan code header, plan catalog cache, RequiresPlanFeature filter                                                                      |
| **Tenant Self-Service UI**            | React 19 + Mantine SPA for workspace members, social login, enterprise SSO, billing, notifications, plan-based feature access, unified dark sidebar, Bulgarian i18n                                   |
| **Platform Admin UI**                 | Operator interface for user/org/subscription/audit management, platform-authority controls, OIDC identity remediation, enterprise theme, dashboard widgets, member signup trend chart                 |
| **Observability**                     | Prometheus metrics, Grafana dashboards, Loki logging, correlation IDs                                                                                                                                 |
| **Event-driven Architecture**         | RabbitMQ topic exchange for async processing and platform events                                                                                                                                      |
| **Internationalization**              | English and Bulgarian (bg-BG) i18n support, multi-lingual announcements                                                                                                                               |

---

## Services

- **IAM** (`foundation-iam-service`) — registration, JWT auth, OAuth2/OIDC federation, tenant SSO, account linking / unlinking, admin unmerge remediation, organizations, RBAC, invitations, token exchange, avatar uploads, announcements, in-app notifications, plan code caching, tenant user stats, max users quota enforcement, personal workspace handling
- **API Gateway** (`foundation-gateway-service`) — JWT validation, tenant resolution, request routing, audit context propagation, per-tenant monitoring, plan code header, plan catalog cache, RequiresPlanFeature filter
- **Billing** (`foundation-billing-service`) — Multi-gateway integration (Stripe and Lemon Squeezy); subscriptions, invoices, refunds, Customer Portal sessions, YAML-based plan config, PlanFeatureRegistry, internal plans API, user entitlements API, read-only plan catalog, per-seat pricing (`PricingModel` FLAT/PER_SEAT), seat-cap validation, seat adjustment API (`PATCH .../seats`), seatCount in subscription events
- **Audit** (`foundation-audit-service`) — centralized event-driven audit trail with SPI-based extensibility; admin search API, severity filters
- **CMS** (`foundation-cms-service`) — content management microservice with static page management, multi-language support, hierarchical content structure, SEO-friendly metadata, tenant isolation via schema-per-tenant
- **Tenant App** (`foundation-ui-app`) — React + Mantine UI covering auth flows, social login, enterprise SSO, connected accounts, workspace management, billing self-service, notifications, plan-based feature access, unified dark sidebar, Bulgarian i18n
- **Platform Admin** (`foundation-ui-platform-admin`) — Operator interface for global user, organization, plan, subscription, refund, announcement, audit log, and CMS content management, plus platform-authority controls and OIDC identity remediation
- **Landing Kit** (`foundation-ui-saas-landing-kit`) — Production-ready landing page template with integrated auth redirects, plan selector, API plans fetch, theme alignment with platform-admin

---

## Stack

### Backend

- Java 25, Spring Boot 4.1, MyBatis 3.x, PostgreSQL 17
- RabbitMQ (async event bus), Liquibase (migrations), Micrometer (metrics)

### Frontend

- React 19, TypeScript, Mantine UI 9, Mantine Form + Zod, TanStack Router + Query, Vite + SWC, Lingui (i18n), Playwright (E2E)

### Infrastructure

- Kubernetes + Helm, Docker Compose, Prometheus + Grafana, Loki

---

## Docs

| Audience          | Document                                                                                                                                                          |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Founders / CEO    | [Business proposal](docs/business/proposal.md) · [SaaS Metrics](docs/business/saas-metrics.md)                                                                    |
| CTO / Architect   | [Architecture](docs/platform/architecture.md) · [Capabilities](docs/platform/capabilities.md)                                                                     |
| Developers        | [Local Development](docs/platform/local-development.md) · [Backend guidelines](docs/coding-guidelines/backend.md) · [UI guidelines](docs/coding-guidelines/ui.md) |
| Compliance review | [Compliance](docs/business/compliance.md)                                                                                                                         |
| Market & Strategy | [Comparison](docs/business/comparison.md) · [Market Review](docs/business/market-review.md)                                                                       |
| Contributors      | [Roadmap](docs/roadmap/vision.md)                                                                                                                                 |

---

## Coding Guidelines

Backend services (IAM, Gateway, Billing, Audit) are Java 25 + Spring Boot 4. Each service follows a vertical-slice package structure (`com.iqkv.{service}`), uses constructor injection, interface-backed services, MyBatis for data access with Liquibase migrations, and publishes domain events to RabbitMQ. REST APIs are versioned (`/api/v1/`), secured with RS256 JWT, and documented via OpenAPI.

The UI is React 19 + Mantine UI 9, built with Vite + SWC. It follows Feature-Sliced Design — layers are enforced by an architecture test that runs on every CI build. Routing via TanStack Router, server state via TanStack Query, and forms via React Hook Form + Zod or Mantine Form + Zod depending on the app.

---

## License

Apache-2.0. See [LICENSE](LICENSE).
