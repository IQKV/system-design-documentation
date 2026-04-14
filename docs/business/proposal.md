# Business Proposal

## Problem

Building B2B SaaS requires solving infrastructure before building product: auth, multi-tenancy, billing, API management, async provisioning. Most teams either skip it and accumulate technical debt, or spend 4–6 months on it before writing a line of product code.

Existing boilerplates (ShipFast, MakerKit, SaaS Pegasus) ship application scaffolding only. Infrastructure is left to the team.

---

## What This Is

Three open-source microservices covering the infrastructure layer:

- IAM — identity, access, organizations
- API Gateway — routing, auth enforcement, rate limiting
- Billing — Stripe Connect wrapper

Deployed via Helm to any Kubernetes cluster. PostgreSQL with schema-per-tenant. RabbitMQ for async provisioning.

Tenancy mode is a deploy-time configuration. Multi-tenant by default; single-tenant by provisioning one default tenant at startup — same codebase, same schema model, no code changes required.

---

## Scope at This Stage

This is early-stage software. Current scope:

- Core tenant lifecycle (register → provision → suspend → delete)
- JWT-based auth with org/role model
- Stripe Connect integration (no custom billing logic)
- React + Mantine UI covering auth, org management, and billing portal
- Helm charts for K8s deployment + Docker Compose for local dev

Out of scope for now: SSO/SAML, usage-based billing, multi-region, managed hosting.

---

## Target Users

Engineering teams (3–15 people) building B2B SaaS who need a deployable infrastructure baseline and have Kubernetes experience. Works equally well for single-product deployments (single-tenant mode) and multi-customer platforms (multi-tenant mode). Not aimed at solo founders or no-code users.

---

## Business Model

Open Core. The three services are Apache-2.0. The event bus (RabbitMQ) is the extension boundary — paid or community extensions subscribe to lifecycle events without modifying core services.

Potential revenue paths (not active yet):

- Paid extensions (SAML, analytics, automation)
- Managed hosting
- Enterprise support contracts

---

## Scaling Path

| Stage      | Setup                                                                                 |
| ---------- | ------------------------------------------------------------------------------------- |
| Early      | Single cluster, shared PostgreSQL, schema-per-tenant                                  |
| Growth     | Read replicas, HPA per service                                                        |
| Enterprise | Tenant migrated to dedicated PostgreSQL instance — config change only, no code change |
