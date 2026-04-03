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

---

## Gap

No open-source, infrastructure-first, language-agnostic option exists between indie boilerplates and Entando. All indie-tier tools ship application code only — K8s, schema isolation, async provisioning, and a real IAM/Gateway/Billing split are left to the team.
