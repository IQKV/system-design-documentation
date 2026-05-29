# SaaS Platform

K8s-native microservices foundation for multi-tenant B2B SaaS. Three open-source services — IAM, API Gateway, Billing — with schema-per-tenant PostgreSQL isolation and event-driven tenant provisioning via RabbitMQ.

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://makeapullrequest.com)
[![Live Demo](https://img.shields.io/badge/Live-Demo-brightgreen.svg)](https://www.iqkv.site)

---

## Status: v0.2 — Administration & Self-Service

The platform has completed its initial demo phase (v0.1) and is currently focusing on **Administration & Self-Service (v0.2)**.

- **Completed:** Core microservices (IAM, Gateway, Billing), basic UI auth flows, tenant lifecycle, and Stripe integration.
- **In Progress:** Platform administration UI, cross-tenant management tools, and tenant billing self-service.

For a detailed breakdown of what's built and what's next, see the [Implemented Features Review](docs/roadmap/06-implemented-features-review-may.md) and [Roadmap](docs/roadmap/vision.md).

---

## Services

- **IAM** (`foundation-iam-service`) — registration, JWT auth, account recovery, organizations, RBAC, invitations
- **API Gateway** (`foundation-gateway-service`) — JWT validation, tenant resolution, request routing
- **Billing** (`foundation-billing-service`) — Stripe wrapper integration; subscriptions and invoices managed on Stripe's side
- **Tenant App** (`foundation-ui-app`) — React + Mantine UI covering auth flows, workspace management, and billing
- **Platform Admin** (`foundation-ui-platform-admin`) — Operator interface for global user, organization, and plan management
- **Landing Kit** (`foundation-ui-saas-landing-kit`) — Production-ready landing page template with integrated auth redirects

---

## Stack

- Kubernetes + Helm
- PostgreSQL (schema-per-tenant)
- RabbitMQ (async event bus)
- Docker Compose for local development

---

## Quick Start

```bash
# Local (multi-tenant by default)
docker compose up

# Single-tenant mode — provision one default tenant at startup
# Set in your values file:
#   platform.rolloutMode: "SINGLE_TENANT"
#   platform.defaultTenantKey: "my-org"
#   platform.defaultTenantName: "My Organization"
docker compose up

# Cluster (deploy each service independently)
helm install foundation-iam-service ./helm/iam -f iam-values.yaml
helm install foundation-gateway-service ./helm/api-gateway -f gateway-values.yaml
helm install foundation-billing-service ./helm/billing -f billing-values.yaml
helm install foundation-ui-service ./helm/ui -f ui-values.yaml
```

---

## Docs

| Audience          | Document                                                                                                                                    |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Founders / CEO    | [Business proposal](docs/business/proposal.md) · [SaaS Metrics](docs/business/saas-metrics.md)                                              |
| CTO / Architect   | [Architecture](docs/platform/architecture.md) · [Capabilities](docs/platform/capabilities.md)                                               |
| Compliance review | [Compliance](docs/business/compliance.md)                                                                                                   |
| Market & Strategy | [Comparison](docs/business/comparison.md) · [Market Review](docs/business/market-review.md)                                                 |
| Contributors      | [Roadmap](docs/roadmap/vision.md) · [Backend guidelines](docs/coding-guidelines/backend.md) · [UI guidelines](docs/coding-guidelines/ui.md) |

---

## Coding Guidelines

Backend services (IAM, Gateway, Billing) are Java 25 + Spring Boot. Each service follows a vertical-slice package structure (`com.iqkv.{service}`), uses constructor injection, interface-backed services, MyBatis for data access with Liquibase migrations, and publishes domain events to RabbitMQ. REST APIs are versioned (`/api/v1/`), secured with RS256 JWT, and documented via OpenAPI.

The UI is React 19 + Mantine v8, built with Vite + SWC. It follows Feature-Sliced Design — layers are enforced by an architecture test that runs on every CI build. Routing via TanStack Router, server state via TanStack Query, forms via React Hook Form + Zod.

Full conventions in [backend guidelines](docs/coding-guidelines/backend.md) and [UI guidelines](docs/coding-guidelines/ui.md).

---

## License

Apache-2.0. See [LICENSE](LICENSE).
