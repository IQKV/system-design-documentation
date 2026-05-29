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

## Status: v0.2 — Completed

The platform has completed the **Administration & Self-Service (v0.2)** milestone, delivering a fully manageable production-ready SaaS foundation.

- **v0.1 (Demo Release):** Core microservices (IAM, Gateway, Billing), basic UI auth flows, tenant lifecycle, Stripe integration
- **v0.2 (Administration & Self-Service):** Platform admin UI, audit service, tenant self-service billing, announcements, in-app notifications, token exchange, avatar uploads, refunds API, WebSocket integration, Grafana dashboards

For detailed feature breakdowns, see:

- [Latest Features Review (v0.2 Completion)](docs/roadmap/09-implemented-features-review.md)
- [Previous Review (May 2026)](docs/roadmap/06-implemented-features-review-may.md)
- [Roadmap & Vision](docs/roadmap/vision.md)

---

## Key Features

| Feature                         | Details                                                                                   |
| ------------------------------- | ----------------------------------------------------------------------------------------- |
| **Hybrid Tenancy**              | Single codebase supports both multi-tenant (B2B) and single-tenant (B2C) deployment modes |
| **Schema-per-tenant Isolation** | Each tenant gets its own PostgreSQL schema for complete data separation                   |
| **Production-ready IAM**        | JWT RS256 auth, RBAC, email verification, password reset, token revocation, invitations   |
| **Stripe Billing Integration**  | Subscriptions, invoices, refunds, Customer Portal, webhook handling                       |
| **Centralized Audit Trail**     | Passive event consumption, SPI-based extensibility, admin search API                      |
| **Reactive API Gateway**        | JWT validation, header sanitization, audit context propagation, per-tenant metrics        |
| **Tenant Self-Service UI**      | React 19 + Mantine SPA for workspace members, billing, notifications                      |
| **Platform Admin UI**           | Operator interface for user/org management, audit logs, announcements, refunds            |
| **Observability**               | Prometheus metrics, Grafana dashboards, Loki logging, correlation IDs                     |
| **Event-driven Architecture**   | RabbitMQ topic exchange for async processing and platform events                          |

---

## Services

- **IAM** (`foundation-iam-service`) — registration, JWT auth, account recovery, organizations, RBAC, invitations, token exchange, avatar uploads, announcements, in-app notifications
- **API Gateway** (`foundation-gateway-service`) — JWT validation, tenant resolution, request routing, audit context propagation, per-tenant monitoring
- **Billing** (`foundation-billing-service`) — Stripe integration; subscriptions, invoices, refunds, Customer Portal sessions
- **Audit** (`foundation-audit-service`) — centralized event-driven audit trail with SPI-based extensibility; admin search API
- **Tenant App** (`foundation-ui-app`) — React + Mantine UI covering auth flows, workspace management, billing self-service, notifications
- **Platform Admin** (`foundation-ui-platform-admin`) — Operator interface for global user, organization, plan, subscription, refund, announcement, and audit log management
- **Landing Kit** (`foundation-ui-saas-landing-kit`) — Production-ready landing page template with integrated auth redirects

---

## Stack

### Backend

- Java 25, Spring Boot 4.0, MyBatis 3.x, PostgreSQL 17
- RabbitMQ (async event bus), Liquibase (migrations), Micrometer (metrics)

### Frontend

- React 19, TypeScript, Mantine UI 8, TanStack Router + Query, Vite + SWC

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

The UI is React 19 + Mantine v8, built with Vite + SWC. It follows Feature-Sliced Design — layers are enforced by an architecture test that runs on every CI build. Routing via TanStack Router, server state via TanStack Query, forms via React Hook Form + Zod.

---

## License

Apache-2.0. See [LICENSE](LICENSE).
