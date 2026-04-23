# Competitive Comparison

## Feature Matrix

The original six competitors plus notable additions discovered since the initial analysis. Grouped by tier for readability.

### Indie / Solo-founder tier

| Capability         | [ShipFast](https://shipfa.st) | [MakerKit](https://makerkit.dev) | [Supastarter](https://supastarter.dev) | [Next.js SaaS BP](https://github.com/ixartz/SaaS-Boilerplate) | [BoxyHQ](https://github.com/boxyhq/saas-starter-kit) | This platform              |
| ------------------ | ----------------------------- | -------------------------------- | -------------------------------------- | ------------------------------------------------------------- | ---------------------------------------------------- | -------------------------- |
| Price              | $199–299                      | $299                             | ~$349                                  | Free / OSS                                                    | Free / OSS                                           | Free / OSS                 |
| License            | Proprietary                   | Proprietary                      | Proprietary                            | MIT                                                           | Apache-2.0                                           | Apache-2.0                 |
| Stack              | Next.js                       | Next.js / Remix                  | Next.js / Nuxt                         | Next.js                                                       | Next.js                                              | Java 25 + React            |
| Multi-tenancy      | None                          | Basic org                        | Organizations                          | Organizations                                                 | Teams                                                | Schema-per-tenant (hybrid) |
| Data isolation     | Shared DB                     | Shared DB                        | Shared DB                              | Shared DB                                                     | Shared DB                                            | Schema-level               |
| Infrastructure     | None                          | None                             | None                                   | None                                                          | None                                                 | K8s + Helm                 |
| Microservices      | No                            | No                               | No                                     | No                                                            | No                                                   | IAM + Gateway + Billing    |
| Async provisioning | No                            | No                               | No                                     | No                                                            | No                                                   | RabbitMQ                   |
| API Gateway        | No                            | No                               | No                                     | No                                                            | No                                                   | Yes                        |
| Billing            | Stripe wrapper                | Stripe wrapper                   | Stripe wrapper                         | Stripe wrapper                                                | Stripe wrapper                                       | Stripe Connect             |
| SSO / SAML         | No                            | No                               | No                                     | No                                                            | Yes (Jackson)                                        | Planned (extension)        |
| Audit logs         | No                            | No                               | No                                     | No                                                            | Yes                                                  | Via event bus              |
| Self-hosted        | Partial                       | Partial                          | Partial                                | Yes                                                           | Yes                                                  | Yes                        |
| Vendor lock-in     | Vercel/Supabase               | Supabase                         | Supabase                               | None                                                          | None                                                 | None                       |

### Framework-specific tier

| Capability         | [Laravel Spark](https://spark.laravel.com) | [SaaSykit Tenancy](https://saasykit.com/multi-tenancy) | [SaaS Pegasus](https://saaspegasus.com) | [Django SaaS BP](https://github.com/apptension/saas-boilerplate) | [BlazorPlate](https://blazorplate.net) | This platform              |
| ------------------ | ------------------------------------------ | ------------------------------------------------------ | --------------------------------------- | ---------------------------------------------------------------- | -------------------------------------- | -------------------------- |
| Price              | $99/yr                                     | ~$299                                                  | $249                                    | Free/paid                                                        | ~$499                                  | Free / OSS                 |
| License            | Proprietary                                | Proprietary                                            | Proprietary                             | MIT                                                              | Proprietary                            | Apache-2.0                 |
| Stack              | Laravel / PHP                              | Laravel / PHP                                          | Django / Python                         | Django / Python                                                  | .NET 10 / Blazor WASM                  | Java 25 + React            |
| Multi-tenancy      | Basic                                      | Schema or DB per tenant                                | Teams                                   | None                                                             | Dedicated DB or shared                 | Schema-per-tenant (hybrid) |
| Data isolation     | Shared DB                                  | Schema-level or DB-level                               | Shared DB                               | Shared DB                                                        | DB-level or shared                     | Schema-level               |
| Infrastructure     | None                                       | None                                                   | None                                    | None                                                             | None                                   | K8s + Helm                 |
| Microservices      | No                                         | No                                                     | No                                      | No                                                               | No                                     | IAM + Gateway + Billing    |
| Async provisioning | No                                         | No                                                     | No                                      | No                                                               | No                                     | RabbitMQ                   |
| API Gateway        | No                                         | No                                                     | No                                      | No                                                               | No                                     | Yes                        |
| Billing            | Stripe/Paddle                              | Stripe/Paddle/Lemon Squeezy                            | Stripe wrapper                          | Stripe wrapper                                                   | Stripe wrapper                         | Stripe Connect             |
| Self-hosted        | Partial                                    | Yes                                                    | Yes                                     | Yes                                                              | Yes                                    | Yes                        |
| Vendor lock-in     | Laravel Cloud                              | None                                                   | Heroku/DO                               | AWS                                                              | None                                   | None                       |

### Enterprise tier

| Capability         | [Entando](https://entando.com) | This platform              |
| ------------------ | ------------------------------ | -------------------------- |
| Price              | Enterprise                     | Free / OSS                 |
| License            | Proprietary                    | Apache-2.0                 |
| Stack              | Java / Angular                 | Java 25 + React            |
| Multi-tenancy      | Yes                            | Schema-per-tenant (hybrid) |
| Data isolation     | Logical                        | Schema-level               |
| Infrastructure     | K8s (opinionated)              | K8s + Helm                 |
| Microservices      | Micro-frontend                 | IAM + Gateway + Billing    |
| Async provisioning | No                             | RabbitMQ                   |
| API Gateway        | No                             | Yes                        |
| Billing            | No                             | Stripe Connect             |
| Self-hosted        | Partial                        | Yes                        |
| Vendor lock-in     | Entando Cloud                  | None                       |

---

## Notes per Competitor

**[ShipFast](https://shipfa.st) / [MakerKit](https://makerkit.dev) / Larafast / SaaSLaunchpad** — Next.js monoliths, one-time purchase. No infrastructure, no schema isolation. Aimed at indie hackers and solo founders.

**[Supastarter](https://supastarter.dev)** — Next.js and Nuxt variants, ~$349 one-time. Multi-tenancy via organizations, but shared DB with `tenant_id` filtering. Supabase-dependent for auth and database. No infrastructure layer.

**[Next.js SaaS Boilerplate (ixartz)](https://github.com/ixartz/SaaS-Boilerplate)** — Free, MIT-licensed, actively maintained. Next.js + Tailwind + Shadcn + DrizzleORM. Organizations, RBAC, i18n, Stripe. Shared DB. No infrastructure. Closest free OSS option in the indie tier.

**[BoxyHQ Enterprise SaaS Starter Kit](https://github.com/boxyhq/saas-starter-kit)** — Free, Apache-2.0. Next.js, teams, SAML SSO via Jackson, directory sync, audit logs, webhooks. The most enterprise-feature-complete free boilerplate in the Next.js space. Still a monolith, shared DB, no infrastructure.

**[Laravel Starter Kits](https://laravel.com/starter-kits) / [Laravel Jetstream](https://jetstream.laravel.com) / [Laravel Spark](https://spark.laravel.com)** — Laravel monoliths. The base starter kits ship with Inertia (React/Vue/Svelte) or Livewire stacks and optional WorkOS AuthKit. Jetstream adds teams, roles, and API tokens. Spark layers Stripe/Paddle subscription billing on top. No K8s, no schema isolation.

**[SaaSykit Tenancy](https://saasykit.com/multi-tenancy)** — Laravel, ~$299. The most serious multi-tenancy option in the Laravel ecosystem — uses `stancl/tenancy` to support schema-per-tenant or database-per-tenant isolation. Seat-based billing with Stripe, Paddle, and Lemon Squeezy. No infrastructure layer, no K8s, no async provisioning. Closest competitor on data isolation, but application-only.

**[SaaS Pegasus](https://saaspegasus.com)** — Django/Python, solid docs, teams model. Monolithic, no infrastructure layer.

**[Django SaaS Boilerplate (Apptension)](https://github.com/apptension/saas-boilerplate)** — Open-source, closest to infrastructure-aware in the indie tier. AWS-specific (Cognito, SES) — cloud lock-in. No K8s, no schema-per-tenant.

**[BlazorPlate](https://blazorplate.net)** — .NET 10 + Blazor WASM, ~$499. Supports dedicated DB per tenant, shared DB, or single-tenant mode — switchable without code changes. Clean Architecture, CQRS via MediatR, real-time notifications. No K8s, no async provisioning, no API Gateway. Interesting for .NET teams that need isolation options, but proprietary and no infrastructure story.

**[Entando](https://entando.com)** — Enterprise micro-frontend platform on K8s. Solves composable portal composition, not SaaS tenant provisioning. Java-based, vendor-managed, high TCO.

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
- **Audit logs** — BoxyHQ includes structured audit logs. This platform emits events to RabbitMQ which can feed an audit log, but there is no built-in audit log UI or storage.
- **Billing flexibility** — SaaSykit Tenancy supports Stripe, Paddle, and Lemon Squeezy. This platform is Stripe Connect only.
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
2. **Which company do you belong to?** — organizations, teams, roles
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
