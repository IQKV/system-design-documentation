# Competitive Comparison

## Feature Matrix

| Capability         | [ShipFast](https://shipfa.st) | [MakerKit](https://makerkit.dev) | [Laravel Spark](https://spark.laravel.com) | [SaaS Pegasus](https://saaspegasus.com) | [Django SaaS BP](https://github.com/apptension/saas-boilerplate) | [Entando](https://entando.com) | This platform           |
| ------------------ | ----------------------------- | -------------------------------- | ------------------------------------------ | --------------------------------------- | ---------------------------------------------------------------- | ------------------------------ | ----------------------- |
| Price              | $199–299                      | $299                             | $99/yr                                     | $249                                    | Free/paid                                                        | Enterprise                     | Free / OSS              |
| License            | Proprietary                   | Proprietary                      | Proprietary                                | Proprietary                             | MIT                                                              | Proprietary                    | Apache-2.0              |
| Multi-tenancy      | None                          | Basic org                        | Basic                                      | Teams                                   | None                                                             | Yes                            | Schema-per-tenant       |
| Data isolation     | Shared DB                     | Shared DB                        | Shared DB                                  | Shared DB                               | Shared DB                                                        | Logical                        | Schema-level            |
| Infrastructure     | None                          | None                             | None                                       | None                                    | None                                                             | K8s (opinionated)              | K8s + Helm              |
| Microservices      | No                            | No                               | No                                         | No                                      | No                                                               | Micro-frontend                 | IAM + Gateway + Billing |
| Async provisioning | No                            | No                               | No                                         | No                                      | No                                                               | No                             | RabbitMQ                |
| API Gateway        | No                            | No                               | No                                         | No                                      | No                                                               | No                             | Yes                     |
| Billing            | Stripe wrapper                | Stripe wrapper                   | Stripe wrapper                             | Stripe wrapper                          | Stripe wrapper                                                   | No                             | Stripe Connect          |
| Self-hosted        | Partial                       | Partial                          | Partial                                    | Yes                                     | Yes                                                              | Partial                        | Yes                     |
| Vendor lock-in     | Vercel/Supabase               | Supabase                         | Laravel Cloud                              | Heroku/DO                               | AWS                                                              | Entando Cloud                  | None                    |

---

## Notes per Competitor

**[ShipFast](https://shipfa.st) / [MakerKit](https://makerkit.dev) / Larafast / SaaSLaunchpad** — Next.js monoliths, one-time purchase. No infrastructure, no schema isolation. Aimed at indie hackers and solo founders.

**[Laravel Starter Kits](https://laravel.com/starter-kits) / [Laravel Jetstream](https://jetstream.laravel.com) / [Laravel Spark](https://spark.laravel.com)** — Laravel monoliths. The base starter kits ship with Inertia (React/Vue/Svelte) or Livewire stacks and optional WorkOS AuthKit. Jetstream adds teams, roles, and API tokens. Spark layers Stripe/Paddle subscription billing on top. No K8s, no schema isolation.

**[SaaS Pegasus](https://saaspegasus.com)** — Django/Python, solid docs, teams model. Monolithic, no infrastructure layer.

**[Django SaaS Boilerplate (Apptension)](https://github.com/apptension/saas-boilerplate)** — Open-source, closest to infrastructure-aware in the indie tier. AWS-specific (Cognito, SES) — cloud lock-in. No K8s, no schema-per-tenant.

**[Entando](https://entando.com)** — Enterprise micro-frontend platform on K8s. Solves composable portal composition, not SaaS tenant provisioning. Java-based, vendor-managed, high TCO.

**This platform wins when:**

- The team needs real data isolation — every customer's data lives in its own PostgreSQL schema, not mixed in a shared table with a `tenant_id` column
- Self-hosting is a requirement — full Kubernetes deployment, no dependency on Vercel, Supabase, AWS, or any vendor
- The product needs to grow beyond a monolith — IAM, Gateway, and Billing are separate services; add your own without touching the core
- The team wants to avoid paying for infrastructure they'll outgrow — Apache-2.0, free forever, extend via the event bus
- Enterprise customers are in the picture — schema isolation, RBAC, and async provisioning are table stakes for B2B sales; most boilerplates can't offer them at all

---

## Gap

No open-source, infrastructure-first, language-agnostic option exists between indie boilerplates and Entando. All indie-tier tools ship application code only — K8s, schema isolation, async provisioning, and a real IAM/Gateway/Billing split are left to the team.

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
