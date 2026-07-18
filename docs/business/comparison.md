# Competitive Comparison

## Feature Matrix

The original six competitors plus notable additions discovered since the initial analysis. Grouped by tier for readability.

### Indie / Solo-founder tier

| Capability         | [ShipFast](https://shipfa.st) | [MakerKit](https://makerkit.dev) | [VibeReady](https://betalist.com/startups/vibeready) | [Supastarter](https://supastarter.dev) | [Next.js SaaS BP](https://github.com/ixartz/SaaS-Boilerplate) | [BoxyHQ](https://github.com/boxyhq/saas-starter-kit) | This platform              |
| ------------------ | ----------------------------- | -------------------------------- | ---------------------------------------------------- | -------------------------------------- | ------------------------------------------------------------- | ---------------------------------------------------- | -------------------------- |
| Price              | $199–299                      | $299                             | ~$49                                                 | ~$349                                  | Free / OSS                                                    | Free / OSS                                           | Free / OSS                 |
| License            | Proprietary                   | Proprietary                      | Proprietary                                          | Proprietary                            | MIT                                                           | Apache-2.0                                           | Apache-2.0                 |
| Stack              | Next.js                       | Next.js / Remix                  | Next.js                                              | Next.js / Nuxt                         | Next.js                                                       | Next.js                                              | Java 25 + React            |
| AI tooling         | No                            | No                               | Built-in (LLM context, RAG assistant)                | No                                     | No                                                            | No                                                   | No (uses standard tools)   |
| Multi-tenancy      | None                          | Basic org                        | Org-level                                            | Organizations                          | Organizations                                                 | Teams                                                | Schema-per-tenant (hybrid) |
| Data isolation     | Shared DB                     | Shared DB                        | Shared DB                                            | Shared DB                              | Shared DB                                                     | Shared DB                                            | Schema-level               |
| Admin portal       | No                            | Partial                          | Super admin portal                                   | Partial                                | No                                                            | Yes                                                  | Dedicated SPA              |
| Feature flags      | No                            | No                               | Yes (built-in)                                       | No                                     | No                                                            | No                                                   | YAML plan features         |
| Infrastructure     | None                          | None                             | None                                                 | None                                   | None                                                          | None                                                 | K8s + Helm                 |
| Microservices      | No                            | No                               | No                                                   | No                                     | No                                                            | No                                                   | IAM + Gateway + Billing    |
| Async provisioning | No                            | No                               | No                                                   | No                                     | No                                                            | No                                                   | RabbitMQ                   |
| API Gateway        | No                            | No                               | No                                                   | No                                     | No                                                            | No                                                   | Yes                        |
| Billing            | Stripe wrapper                | Stripe wrapper                   | Stripe wrapper                                       | Stripe wrapper                         | Stripe wrapper                                                | Stripe wrapper                                       | Stripe Wrapper             |
| SSO / SAML         | No                            | No                               | No                                                   | No                                     | No                                                            | Yes (Jackson)                                        | Planned (extension)        |
| Audit logs         | No                            | No                               | No                                                   | No                                     | No                                                            | Yes                                                  | Centralized service        |
| Self-hosted        | Partial                       | Partial                          | Yes                                                  | Partial                                | Yes                                                           | Yes                                                  | Yes                        |
| Vendor lock-in     | Vercel/Supabase               | Supabase                         | None noted                                           | Supabase                               | None                                                          | None                                                 | None                       |

### Framework-specific tier

| Capability         | [Laravel Spark](https://spark.laravel.com) | [SaaSykit Tenancy](https://saasykit.com/multi-tenancy) | [SaaS Pegasus](https://saaspegasus.com) | [Django SaaS BP](https://github.com/apptension/saas-boilerplate) | [BlazorPlate](https://blazorplate.net) | [AppointMe](https://appointme.dev)  | This platform              |
| ------------------ | ------------------------------------------ | ------------------------------------------------------ | --------------------------------------- | ---------------------------------------------------------------- | -------------------------------------- | ----------------------------------- | -------------------------- |
| Price              | $99/yr                                     | ~$299                                                  | $249                                    | Free/paid                                                        | ~$499                                  | Free (MIT) / Paid PRO/Solo/Team     | Free / OSS                 |
| License            | Proprietary                                | Proprietary                                            | Proprietary                             | MIT                                                              | Proprietary                            | MIT (Free) / Commercial (PRO)       | Apache-2.0                 |
| Stack              | Laravel / PHP                              | Laravel / PHP                                          | Django / Python                         | Django / Python                                                  | .NET 10 / Blazor WASM                  | .NET 10 + React 19                  | Java 25 + React            |
| Multi-tenancy      | Basic                                      | Schema or DB per tenant                                | Teams                                   | None                                                             | Dedicated DB or shared                 | Shared DB (tenant filtering)        | Schema-per-tenant (hybrid) |
| Data isolation     | Shared DB                                  | Schema-level or DB-level                               | Shared DB                               | Shared DB                                                        | DB-level or shared                     | Shared DB                           | Schema-level               |
| Infrastructure     | None                                       | None                                                   | None                                    | None                                                             | None                                   | .NET Aspire + Terraform (Azure/AWS) | K8s + Helm                 |
| Microservices      | No                                         | No                                                     | No                                      | No                                                               | No                                     | No (modular monolith)               | IAM + Gateway + Billing    |
| Async provisioning | No                                         | No                                                     | No                                      | No                                                               | No                                     | No (Wolverine async messaging)      | RabbitMQ                   |
| API Gateway        | No                                         | No                                                     | No                                      | No                                                               | No                                     | No                                  | Yes                        |
| Billing            | Stripe/Paddle                              | Stripe/Paddle/Lemon Squeezy                            | Stripe wrapper                          | Stripe wrapper                                                   | Stripe wrapper                         | Stripe (PRO only)                   | Stripe Connect             |
| Self-hosted        | Partial                                    | Yes                                                    | Yes                                     | Yes                                                              | Yes                                    | Yes                                 | Yes                        |
| Vendor lock-in     | Laravel Cloud                              | None                                                   | Heroku/DO                               | AWS                                                              | None                                   | None                                | None                       |

### Enterprise tier

| Capability         | [Liferay DXP](https://liferay.com) | [dotCMS](https://dotcms.com) | [Entando](https://entando.com) | [Helical Insight](https://www.helicalinsight.com) | [Spree Commerce](https://spreecommerce.org) | This platform              |
| ------------------ | ---------------------------------- | ---------------------------- | ------------------------------ | ------------------------------------------------- | ------------------------------------------- | -------------------------- |
| Price              | Enterprise                         | Enterprise / Free            | Enterprise                     | Enterprise / Community                            | Free / OSS                                  | Free / OSS                 |
| License            | Proprietary / LGPL                 | Proprietary / GPL            | Proprietary                    | Proprietary / Apache-2.0                          | BSD-3-Clause                                | Apache-2.0                 |
| Stack              | Java / OSGi                        | Java / OSGi                  | Java / Angular                 | Java / React                                      | Ruby on Rails / React                       | Java 25 + React            |
| Multi-tenancy      | Virtual Instances                  | Sites / Hosts                | Yes                            | Organization-level                                | Storefront-based                            | Schema-per-tenant (hybrid) |
| Data isolation     | Logical / DB-level                 | Logical                      | Logical                        | Logical / Schema-level                            | Logical                                     | Schema-level               |
| Infrastructure     | K8s / Cloud                        | K8s / Cloud                  | K8s (opinionated)              | VM / Docker                                       | K8s / Cloud                                 | K8s + Helm                 |
| Microservices      | OSGi Bundles                       | OSGi Bundles                 | Micro-frontend                 | No (Monolithic)                                   | Headless API                                | IAM + Gateway + Billing    |
| Async provisioning | Yes                                | Yes                          | No                             | No                                                | No                                          | RabbitMQ                   |
| API Gateway        | No (uses reverse proxy)            | No                           | No                             | No                                                | No                                          | Yes                        |
| Billing            | No                                 | No                           | No                             | No                                                | Yes (Payment Gateways)                      | Stripe Wrapper             |
| Self-hosted        | Yes                                | Yes                          | Partial                        | Yes                                               | Yes                                         | Yes                        |
| Vendor lock-in     | High (Liferay Cloud)               | Medium                       | Entando Cloud                  | Low                                               | Low                                         | None                       |

### Cloud-Native & Platform Engineering Tier

| Capability         | [Kubermatic](https://kubermatic.com) | [Platform Stack (Argo/Crossplane/Backstage)](https://backstage.io) | This platform              |
| ------------------ | ------------------------------------ | ------------------------------------------------------------------ | -------------------------- |
| Price              | Enterprise / Free                    | Free / OSS                                                         | Free / OSS                 |
| License            | Apache-2.0                           | Apache-2.0                                                         | Apache-2.0                 |
| Focus              | Multi-cluster K8s mgmt               | Internal Developer Portal (IDP)                                    | SaaS Product Foundation    |
| Multi-tenancy      | Project/Cluster isolation            | Namespace / Resource isolation                                     | Schema-per-tenant (hybrid) |
| Data isolation     | Infrastructure-level                 | Infrastructure-level                                               | Schema-level               |
| Infrastructure     | K8s Operator-driven                  | GitOps (Argo) + IaC (Crossplane)                                   | K8s + Helm                 |
| Microservices      | No                                   | No (Infrastructure only)                                           | IAM + Gateway + Billing    |
| Async provisioning | Yes                                  | Yes (via ArgoCD/Crossplane)                                        | RabbitMQ                   |
| API Gateway        | No                                   | No                                                                 | Yes                        |
| Billing            | No                                   | No                                                                 | Stripe Connect             |
| Self-hosted        | Yes                                  | Yes                                                                | Yes                        |
| Vendor lock-in     | None                                 | Low (Cloud provider via Crossplane)                                | None                       |

---

## Notes per Competitor

**[ShipFast](https://shipfa.st) / [MakerKit](https://makerkit.dev) / Larafast / SaaSLaunchpad** — Next.js monoliths, one-time purchase. No infrastructure, no schema isolation. Aimed at indie hackers and solo founders.

**[VibeReady](https://betalist.com/startups/vibeready)** — AI-native Next.js SaaS starter kit, ~$49 one-time. Its core differentiator is LLM-friendly architecture: structured context routing, quality gates, and documentation formatted for AI coding tools (Claude Code, Cursor, Windsurf, Copilot) so AI-generated code stays consistent as the codebase grows. Includes auth, Stripe billing, org-level multi-tenancy (shared DB), a super admin portal, feature flags, and a built-in AI assistant with RAG. It is a monolith with no infrastructure layer, no schema isolation, and no microservices — but it targets a specific use case (solo founders building AI-native products with AI coding tools) that the other indie starters don't address. Proprietary license.

**[Supastarter](https://supastarter.dev)** — Next.js and Nuxt variants, ~$349 one-time. Multi-tenancy via organizations, but shared DB with `tenant_id` filtering. Supabase-dependent for auth and database. No infrastructure layer.

**[Next.js SaaS Boilerplate (ixartz)](https://github.com/ixartz/SaaS-Boilerplate)** — Free, MIT-licensed, actively maintained. Next.js + Tailwind + Shadcn + DrizzleORM. Organizations, RBAC, i18n, Stripe. Shared DB. No infrastructure. Closest free OSS option in the indie tier.

**[BoxyHQ Enterprise SaaS Starter Kit](https://github.com/boxyhq/saas-starter-kit)** — Free, Apache-2.0. Next.js, teams, SAML SSO via Jackson, directory sync, audit logs, webhooks. The most enterprise-feature-complete free boilerplate in the Next.js space. Still a monolith, shared DB, no infrastructure.

**[Laravel Starter Kits](https://laravel.com/starter-kits) / [Laravel Jetstream](https://jetstream.laravel.com) / [Laravel Spark](https://spark.laravel.com)** — Laravel monoliths. The base starter kits ship with Inertia (React/Vue/Svelte) or Livewire stacks and optional WorkOS AuthKit. Jetstream adds teams, authortities, and API tokens. Spark layers Stripe/Paddle subscription billing on top. No K8s, no schema isolation.

**[SaaSykit Tenancy](https://saasykit.com/multi-tenancy)** — Laravel, ~$299. The most serious multi-tenancy option in the Laravel ecosystem — uses `stancl/tenancy` to support schema-per-tenant or database-per-tenant isolation. Seat-based billing with Stripe, Paddle, and Lemon Squeezy. No infrastructure layer, no K8s, no async provisioning. Closest competitor on data isolation, but application-only.

**[SaaS Pegasus](https://saaspegasus.com)** — Django/Python, solid docs, teams model. Monolithic, no infrastructure layer.

**[Django SaaS Boilerplate (Apptension)](https://github.com/apptension/saas-boilerplate)** — Open-source, closest to infrastructure-aware in the indie tier. AWS-specific (Cognito, SES) — cloud lock-in. No K8s, no schema-per-tenant.

**[BlazorPlate](https://blazorplate.net)** — .NET 10 + Blazor WASM, ~$499. Supports dedicated DB per tenant, shared DB, or single-tenant mode — switchable without code changes. Clean Architecture, CQRS via MediatR, real-time notifications. No K8s, no async provisioning, no API Gateway. Interesting for .NET teams that need isolation options, but proprietary and no infrastructure story.

**[AppointMe](https://appointme.dev)** — .NET 10 + React 19 modular monolith SaaS template, free (MIT) with paid PRO/Solo/Team tiers. Includes multi-tenancy (shared DB with tenant filtering), OAuth2/OIDC auth (Keycloak/Entra), CQRS/DDD, vertical slice architecture, .NET Aspire orchestration, audit trail, transactional outbox, structured logging/OpenTelemetry, CI/CD templates, and Terraform for Azure/AWS. Billing is PRO-only. No microservices or API Gateway, and no schema-level data isolation. Strong option for .NET teams that want a production-ready modular monolith foundation.

**[Entando](https://entando.com)** — Enterprise micro-frontend platform on K8s. Solves composable portal composition, not SaaS tenant provisioning. Java-based, vendor-managed, high TCO.

**[Liferay DXP](https://liferay.com)** — The "heavyweight" champion of Java portals. Highly flexible through OSGi, supports complex multi-tenancy (Virtual Instances). However, it is a massive monolith with high complexity and licensing costs. IQKV provides the multi-tenancy power without the OSGi/DXP overhead.

**[dotCMS](https://dotcms.com)** — Java-based hybrid CMS with multi-site/multi-tenant support. Like Liferay, it's focused on content delivery rather than SaaS application plumbing. Great for websites, overkill for products.

**[Helical Insight](https://www.helicalinsight.com)** — Java-based Open Source Business Intelligence (BI) platform. Offers organization-level multi-tenancy and support for schema-level isolation. While it solves the analytics layer, it doesn't provide the SaaS application foundation (IAM/Gateway/Billing) that this platform offers.

**[Spree Commerce](https://spreecommerce.org)** — Ruby on Rails headless e-commerce platform. While it has multi-store/multi-tenant capabilities, it is highly specialized for commerce. This platform is a more general-purpose foundation for any B2B SaaS product.

**[Kubermatic Kubernetes Platform (KKP)](https://kubermatic.com)** — Focuses on the "Infrastructure as a Product" layer. Excellent for managing multi-cluster/multi-tenant K8s, but stops at the cluster/namespace level. IQKV picks up where KKP leaves off by providing the _application_ layer tenancy (IAM, Billing).

**[ArgoCD + Crossplane + Backstage](https://backstage.io)** — The modern "Platform Engineering" gold standard. This stack creates an Internal Developer Platform (IDP). Crossplane manages cloud resources, ArgoCD handles GitOps, and Backstage provides the UI. While IQKV uses some of these patterns (K8s, async events), its goal is to be the _product_ foundation for external customers, whereas this stack is for _internal_ developer enablement.

---

## This platform wins when

- The team needs real data isolation — every customer's data lives in its own PostgreSQL schema, not mixed in a shared table with a `tenant_id` column
- The product starts single-tenant and needs to grow — deploy with one default tenant provisioned at startup; add more tenants later without any code or schema changes
- Self-hosting is a requirement — full Kubernetes deployment, no dependency on Vercel, Supabase, AWS, or any vendor
- The product needs to grow beyond a monolith — IAM, Gateway, and Billing are separate services; add your own without touching the core
- The team wants to avoid paying for infrastructure they'll outgrow — Apache-2.0, free forever, extend via the event bus
- Enterprise customers are in the picture — schema isolation, RBAC, and async provisioning are table stakes for B2B sales; most boilerplates can't offer them at all

---

## Honest gaps vs. the competition

- **SSO / SAML** — BoxyHQ ships it out of the box. This platform has it on the roadmap as a paid extension. Teams that need SAML on day one should look at BoxyHQ or add Jackson themselves.
- **Audit logs** — BoxyHQ includes structured audit logs. This platform ships a centralized Audit Service that passively consumes events and provides an admin search API with severity filtering.
- **Billing flexibility** — SaaSykit Tenancy supports Stripe, Paddle, and Lemon Squeezy. This platform is Stripe only.
- **AI tooling ergonomics** — VibeReady ships LLM-friendly documentation, context routing, and quality gates specifically designed for AI-assisted development (Claude Code, Cursor, Windsurf). This platform is built for human engineers with standard tooling; teams building with AI coding agents at scale will find VibeReady's structured context layer more immediately productive.
- **Language** — The backend is Java 25. Teams on Node, Python, PHP, or .NET will need to treat the services as black-box infrastructure and build their product services in their own language on top of the event bus. That is by design, but it is a real onboarding cost.
- **Maturity** — ShipFast, MakerKit, SaaS Pegasus, and SaaSykit all have paying customers, community, and battle-tested codebases. This platform is early-stage.

---

## Gap

No open-source, infrastructure-first, language-agnostic option exists between indie boilerplates and Entando. All indie-tier tools ship application code only — K8s, schema isolation, async provisioning, and a real IAM/Gateway/Billing split are left to the team. BoxyHQ comes closest on enterprise features but is still a Next.js monolith with shared DB. SaaSykit Tenancy comes closest on data isolation but has no infrastructure layer.

---

## Why Build This Anyway

**The simple version:** imagine you want to open a restaurant. You could buy a recipe book (that's ShipFast or MakerKit). But a recipe book doesn't give you the kitchen, the plumbing, the health permits, or the cash register. You still have to figure all of that out yourself. This project is the kitchen — the part every restaurant needs before they can cook anything.

Every team building a B2B product has to solve the same four problems before they can write a single line of their actual product:

1. **Who are you?** — login, accounts, password reset
2. **Which company do you belong to?** — organizations, teams, authortities
3. **Are you paying?** — subscriptions, invoices, billing
4. **Can you talk to the system?** — a front door that checks all of the above on every request

Most boilerplates give you a rough sketch of these things baked into one big app. That works fine if you're a solo founder building a simple tool. But the moment a real company wants to use your product — a company with 50 employees, a security team, and a procurement process — that sketch falls apart. They'll ask: can each customer's data be kept completely separate? Can we self-host this? Can we plug in our own login system? The answer with most boilerplates is "not really."

This project answers yes to all of those questions, out of the box, for free.

**Why free and open source?** Because the infrastructure layer is not the product — it's the foundation. Giving it away builds trust, invites contributions, and creates a community. The money comes from what sits on top.

**What sits on top:**

- **Extensions** — things like single sign-on (SSO/SAML), usage tracking, automation workflows, and analytics. These are paid add-ons that plug into the platform without touching the core. Any developer in the community can build and sell one.
- **Managed hosting** — some teams don't want to run Kubernetes themselves. A hosted version of the platform, fully managed, is a natural paid offering.
- **Enterprise support** — larger companies will pay for guaranteed response times, custom integrations, and hands-on help getting set up.
- **Expertise** — consulting and implementation services for teams that want the platform tailored to their specific setup.

**Why does this make sense when there are already so many competitors?**

Because none of them are playing the same game. The indie boilerplates (ShipFast, MakerKit, Pegasus) are selling to solo founders who want to ship fast and don't care about data isolation or Kubernetes. Entando is selling to large enterprises with six-figure budgets and a Java team. There is nothing in between that is open, self-hostable, infrastructure-complete, and language-agnostic.

The teams who need this — small engineering teams (3–15 people) building real B2B products for real business customers — currently have two options: spend 4–6 months building this themselves, or pay Entando-level prices. This project is the third option.
