# Single-tenancy mode implementation

_Exported on 4/25/2026 at 10:23:50 GMT+3 from Cursor (3.1.17)_

---

**User**

1. examine system-design-documentation, specially logic that allows to use single-tenancy mode
2. exam foundation-iam-service, foundation-billing-service implementation, and prepare spec doucment to allow overall system works in single tenant mode. in this case users just signup and auto-join to some pre-provisoned default tenant (still, nanoid is always tenant key, even for defalut), no tenant owner here (or exists only like pre-created system user)

---

**Cursor**

I’ll inspect the repository for `system-design-documentation`, `foundation-iam-service`, and `foundation-billing-service`, focusing on current tenancy assumptions and then draft a concrete single-tenant-mode spec that fits the existing architecture. I’m starting by locating these modules and the relevant tenant/user signup flows.

I found all three projects in the workspace. Next I’m scanning for tenant/signup/default-tenant logic and then I’ll draft the spec against what actually exists today.

I’ve confirmed IAM signup is currently hard-wired to create a new tenant and assign `TENANT_OWNER`. I’m now reviewing Billing event consumers and security/tenant-context handling so the spec covers end-to-end behavior.

I’ve gathered enough context: the current docs describe single-tenant behavior, but IAM/Billing code is still effectively multi-tenant-first and owner-centric. I’m now writing a spec in `system-design-documentation` that reconciles the intended behavior with concrete IAM/Billing implementation changes, including default-tenant auto-join and ownerless flow.

Created a new implementation spec at:

- `system-design-documentation/docs/platform/single-tenant-mode-spec.md`

What it covers:

- Current-state analysis of `system-design-documentation`, `foundation-iam-service`, and `foundation-billing-service` against your requested single-tenant behavior.
- Clear target behavior for single mode: signup auto-joins a pre-provisioned default tenant, default tenant key remains NanoID, no customer-facing tenant owner.
- Concrete IAM changes:
  - mode-aware signup flow,
  - default tenant resolver/bootstrap,
  - config extensions (`MULTI`/`SINGLE`),
  - DB/model changes (`is_default` + uniqueness),
  - authority policy shift away from mandatory `TENANT_OWNER`.
- Concrete Billing changes:
  - `tenant.created` handling without mandatory `ownerEmail`,
  - Stripe customer bootstrap fallback strategy for single mode.
- Event contract guidance to make owner fields optional in single mode.
- Migration sequence, rollback guidance, and test plan (IAM, Billing, E2E).
- Open decisions you’ll want to finalize before implementation.

Key finding from the examination: docs already describe single-tenant intent, but IAM/Billing code is still owner-centric and multi-tenant-first, so the spec focuses on closing that gap without breaking multi-tenant behavior.

If you want, I can now turn this spec into an implementation checklist with task-sized tickets per service.

---

**User**

I feel, from coding perspective it should be some strategy pattern, right?

---

**Cursor**

Yes — from coding perspective, this is a great fit for a **strategy pattern**.

Use tenancy mode as the selector, and isolate mode-specific behavior behind interfaces instead of spreading `if (mode == SINGLE)` across services.

Recommended shape:

- `SignupStrategy`
  - `MultiTenantSignupStrategy` (current create-tenant flow)
  - `SingleTenantSignupStrategy` (resolve default tenant + auto-membership)
- `TenantBootstrapStrategy`
  - `MultiTenantBootstrapStrategy` (no-op or existing behavior)
  - `SingleTenantBootstrapStrategy` (ensure default tenant exists/provisioned)
- optional: `TenantAuthorizationPolicy` if authority logic diverges by mode

Then a small factory/resolver:

- `TenancyModeStrategyResolver` picks strategy from `iqkv.tenancy.mode`

Why this is better here:

- Keeps current multi-tenant code mostly intact
- Makes single-tenant path explicit and testable
- Reduces regression risk by avoiding scattered conditionals
- Easier to add future modes/policies

If you want, I can update `single-tenant-mode-spec.md` with a dedicated “Strategy-based implementation blueprint” section and class skeleton names matching your package layout.

---

**User**

update

---

**Cursor**

I’ll update the spec with a concrete strategy-pattern blueprint (interfaces, implementations, resolver, wiring, and testing notes) so it’s directly actionable for the IAM/Billing codebase.

Updated `system-design-documentation/docs/platform/single-tenant-mode-spec.md` with a new section:

- `5.6 Strategy-pattern implementation blueprint (recommended)`

It now includes:

- suggested strategy interfaces (`SignupStrategy`, `TenantBootstrapStrategy`, optional authorization strategy),
- concrete implementations for `MULTI` and `SINGLE`,
- a resolver/factory (`TenancyModeStrategyResolver`),
- service wiring guidance for `UserServiceImpl` and startup bootstrap,
- short Java pseudocode examples,
- strategy-focused test structure.

This should make the spec implementation-oriented and consistent with your intuition to use strategy pattern.

---

**User**

ok

---

**Cursor**

Great. Next logical step is implementing this in `foundation-iam-service` first (strategy wiring + default-tenant bootstrap), then adapting `foundation-billing-service` event handling for optional owner fields.

If you want, I can start that implementation now.

---

**User**

how this logic affect billing? what is the better approach?

---

**Cursor**

It affects Billing mostly at the **tenant bootstrap contract boundary**, not in core subscription logic.

## Impact On Billing

- **Today Billing assumes owner-centric tenant creation**: `TenantEventConsumer` expects `ownerEmail` on `TENANT_CREATED` and fails if missing.
- In single-tenant mode, your signup flow won’t create tenant/owner per user; there is one pre-provisioned tenant, so owner fields may be absent or system-level.
- Billing still needs exactly one `billing_settings` row per tenant and one Stripe customer per tenant; that model remains valid.
- User signups in single mode should **not** trigger new billing tenant bootstrap events (only membership/user lifecycle).

## Better Approach

Best approach: **make Billing tenant-event handling owner-agnostic and idempotent**.

- Treat `tenant.created` as **tenant identity event**, not owner event.
- Make `ownerEmail` optional in Billing consumer.
- Use fallback billing contact strategy:
  1. event `ownerEmail` (multi mode),
  2. configured default billing email (single mode),
  3. null/blank (if acceptable) and update later via settings UI/API.
- Keep `billing_settings.profile_owner_id` nullable (already compatible with your model).
- Ensure duplicate `tenant.created` stays harmless (`existsByTenantKey` guard is already present).

## Why This Is Better

- One Billing code path supports both multi and single tenancy.
- No fake “tenant owner” is required just to satisfy Billing bootstrap.
- Lower coupling between IAM authority semantics and billing identity.
- Future-proof: if authortities evolve again, Billing contract stays stable.

## Practical recommendation

- **Do not split billing by tenancy strategy first.**
- Keep one consumer, but harden it for optional owner fields.
- If needed later, add a tiny `BillingContactResolver` strategy only for email fallback resolution (not full flow branching).

If you want, I can implement this Billing change set next in `foundation-billing-service` with minimal diff and backward compatibility.

---

**User**

what if in single mode each user could have own billing settings?

---

**Cursor**

That’s possible, but it changes the billing model from **tenant-billing** to **account-billing** (or hybrid), which is a bigger shift.

## What Changes If Each User Has Own Billing Settings In Single Mode

- Current model is `1 tenant -> 1 billing_settings -> 1 Stripe customer`.
- Your proposal becomes `1 tenant -> many user billing profiles` (likely many Stripe customers or payment methods).
- Subscription ownership semantics change: who owns plan/limits if all users share one tenant but pay separately?

## Two Viable Designs

- **A) Keep tenant subscription, add per-user invoicing metadata**
  - Tenant still has one actual subscription.
  - Each user can store billing preferences (name/address/tax/email), mostly for chargeback/internal allocation.
  - Low risk; no entitlement model rewrite.

- **B) True per-user billing in single tenant**
  - Add `user_billing_settings` keyed by `user_id`.
  - Billing/entitlements enforced per user, not per tenant.
  - Requires redesign of subscription checks, feature gating, events, and likely pricing logic.

## Better Approach (Usually)

- If your product access is still tenant-scoped: choose **A**.
- If you truly want users to buy/manage independent plans: choose **B**, but treat it as a separate architecture initiative, not just a single-tenant tweak.

## Recommended Hybrid

- Keep existing `billing_settings` (tenant-level, source of truth for subscription).
- Add optional `user_billing_profiles` for user-specific invoice/contact/tax info.
- Use user profile only when generating user-targeted receipts or finance exports; do not tie core entitlements to it.

If you want, I can extend the spec with a dedicated section: “Single mode with per-user billing profiles” including schema, API, and migration impact.

---

**User**

what if in both variants subscription plans only pre-provisioned? platfrom clients predefining plans before any registration starts, then - user (tenant owner, in case of multi tenant, and every user in single mode always has own billing settings), and plans are not manageable by any user?

---

**Cursor**

That’s a strong model, and it actually simplifies a lot: **catalog-managed billing** (platform-defined plans only, no self-service plan CRUD).

## What this means

- Plans are defined by platform admins **before onboarding**.
- End users cannot create/edit/delete plans.
- Billing UI becomes “select/subscribe/view status,” not “manage plans.”
- Works for both:
  - multi-tenant: tenant owner subscribes tenant,
  - single-tenant: each user subscribes self (with own billing settings).

## Key design decision you need

You now have **two entitlement scopes** in one platform:

- `TENANT` scope (multi mode)
- `USER` scope (single mode)

Best approach: make subscription domain explicitly scoped.

## Recommended model

- `plan_catalog` (platform-owned, pre-provisioned)
  - `plan_code`, `name`, `price`, `billing_period`, `features_json`, `active`, `scope` (`TENANT`/`USER`/`BOTH`)
- `subscriptions`
  - add `subject_type` (`TENANT` or `USER`)
  - add `subject_key` (tenant_key or user_id)
  - unique active subscription per subject
- `billing_settings`
  - keep tenant-level for tenant subscriptions
- `user_billing_settings`
  - add user-level for user subscriptions in single mode

## How to keep services clean

Use strategy/policy here too:

- `SubscriptionSubjectResolver`
  - multi mode -> resolve subject as tenant
  - single mode -> resolve subject as user
- `PlanEligibilityPolicy`
  - validates plan scope against subject type
- `EntitlementEvaluator`
  - checks active subscription by subject

This avoids forking billing service by mode.

## Why this is better than mode-specific hardcoding

- One billing architecture supports both variants.
- Future-proof if you later allow user subscriptions in multi mode (or tenant packs in single mode).
- Clear audit trail and fewer “if single then…” branches.

## Important caveats

- Gateway/auth must put enough context for entitlement checks (`tenant_id`, `user_id`, mode/scope).
- Webhook handling must map Stripe customer/subscription back to correct subject type.
- You need operator APIs (or admin scripts) for catalog lifecycle, since users can’t manage plans.

If you want, I can add a new section to your spec with this exact “pre-provisioned catalog + subject-scoped subscriptions” design and migration steps for current tables.

---

**User**

good. iam, gateway, billing - all core services should know in what runtime they wokrs. it is imposible that gateway works in multi tenanat, but others - in signle, and vise versa, etc.
aio solution, but overall setup (platfrom rollout mode, let's say) possible only in some selected mode

---

**Cursor**

Absolutely — that’s the right constraint.  
You want a **single platform rollout mode** (global runtime contract), not per-service independent mode flags.

## Recommended AIO approach

Define one global setting, e.g. `PLATFORM_ROLLOUT_MODE`, with allowed values:

- `MULTI_TENANT`
- `SINGLE_TENANT`

All core services (`iam`, `gateway`, `billing`) must read and enforce the same value.

## How to enforce consistency

- **Single source of truth**
  - Prefer one shared config source (Helm values, ConfigMap/Secret bundle, or env template) used by all charts.
- **Startup fail-fast**
  - Each service validates required mode-specific config.
  - If invalid/missing -> app fails startup.
- **Cross-service mode handshake**
  - On startup, each service publishes/reads a small “mode declaration” (or checks IAM `/platform/capabilities` endpoint).
  - If mode mismatch detected -> fail readiness / refuse traffic.
- **Readiness gating**
  - Service reports `NOT READY` until mode agreement is confirmed.
- **Operational lock**
  - Mode changes are controlled operation (maintenance/migration), not hot flip in production.

## Minimal contract to add

Shared contract object (logical, not necessarily endpoint-only):

- `platformMode`: `MULTI_TENANT | SINGLE_TENANT`
- `contractVersion`: e.g. `1`
- `issuedAt`
- optional `capabilities` map (signup behavior, invitation availability, billing subject scope)

Gateway should use this contract to drive routing/validation behavior consistently with IAM and Billing.

## Practical deployment rule

- One Helm umbrella/values set should inject the same mode into all core services.
- CI/CD pipeline should validate “all core charts have identical mode value” before deploy.

This gives you the AIO guarantee you described: impossible to run mixed core modes by accident.

If you want, I can now add a dedicated “Global Rollout Mode Contract” section to your spec with concrete config keys, readiness rules, and failure scenarios.

---

**User**

make spec up-to-date

---

**Cursor**

I’ll update the spec now to include the global rollout-mode contract across IAM/Gateway/Billing, plus pre-provisioned plan catalog and subject-scoped subscriptions for both multi and single variants.

Updated `system-design-documentation/docs/platform/single-tenant-mode-spec.md` to reflect your latest direction.

What is now up-to-date:

- Global **AIO rollout-mode contract** for `iam`, `gateway`, `billing`:
  - one shared `iqkv.platform.rollout-mode`,
  - fail readiness on mixed/missing/invalid mode,
  - startup validation + recommended cross-service handshake.
- Billing/domain model updated for **pre-provisioned plan catalog**:
  - plans are platform admin-managed only,
  - no user plan CRUD.
- Subscription model updated to **subject scope**:
  - `TENANT` subject in multi mode,
  - `USER` subject in single mode.
- Added schema/policy direction:
  - `plan_catalog`,
  - `subscriptions.subject_type/subject_key`,
  - `user_billing_settings` for single-mode user billing.
- Test plan expanded for scope-aware entitlements and catalog immutability for non-operators.
- Event contract now includes requirement for `subject_type` and `subject_key` in subscription events.

If you want, I can do one more pass to split this into “MVP phase” vs “Phase 2” so implementation sequencing is clearer for the team.

---

**User**

based on current Implementations, describe in system-design-documentation\docs\platform\single-tenant-mode-spec.md required strict restrctions, gates, components, might be authorities (who knows?:)) for overall platfrom and each microservice

---

**Cursor**

I’ll add a new section that is explicitly grounded in the current code paths, defining strict platform-wide and per-service restrictions/gates/components/authorities for rollout-mode safety and tenant isolation.

Updated `system-design-documentation/docs/platform/single-tenant-mode-spec.md` with a new strict governance section based on current implementations:

- Added `## 14. Strict Restrictions, Gates, Components, and Authorities (based on current implementations)`

It now defines:

- platform-wide hard restrictions and gates (single rollout mode, fail-fast, readiness rules),
- required governance components (`PlatformModeValidator`, capabilities provider/consumer, health indicator),
- IAM-specific restrictions/gates/components aligned with current filter/auth/signup behavior,
- Gateway-specific restrictions/gates/components for mode consistency and subject resolution,
- Billing-specific restrictions/gates/components aligned with current `TenantEventConsumer` and tenant context enforcement,
- authority boundaries (recommended platform-level authorities + strict separation from tenant-scoped authortities),
- non-negotiable invariants for runtime safety.

If you want, I can next convert section 14 into a concise “Implementation Checklist” with owner-by-service and acceptance criteria per gate.

---

**User**

go on

---

**Cursor**

I’ll continue by adding a practical implementation checklist section with per-service ownership, acceptance criteria, and rollout order so section 14 becomes execution-ready.

Extended the spec with an execution-ready section:

- Added `## 15. Implementation Checklist (service owners, gates, acceptance criteria)`

What’s included:

- per-service ownership and tasks for Platform/Core, IAM, Gateway, Billing, Security, DB, QA,
- strict acceptance criteria for each area,
- migration checklist and integration/release gates,
- suggested delivery sequence,
- overall definition of done aligned with your AIO rollout-mode requirement.

If you want, I can continue with `## 16. Endpoint & Authority Matrix` (explicit endpoint-by-endpoint permissions for both modes), which usually helps implementation and QA a lot.

---

**User**

is should be in tenancy package, according to existing design approached?

---

**Cursor**

Yes — for this codebase, most of this should live under the **`tenancy` package** (or `tenancy` + `platform` split), based on current design.

Best-fit placement:

- `tenancy`
  - `TenancyMode` enum
  - `TenancyConfigurationProperties` (extended)
  - `TenancyModeStrategyResolver`
  - `SignupStrategy` + mode implementations
  - `TenantBootstrapStrategy` + mode implementations
  - `DefaultTenantResolver`
  - `PlatformModeValidator` (if only tenancy-related)

- Keep in existing domain packages:
  - `user` package: user orchestration entrypoint (`UserServiceImpl`) delegates to strategy
  - `billing`/`subscription` packages: billing domain logic (subject resolution can be tenancy-aware but belongs near subscription domain)
  - `infrastructure.config`: bean wiring only (no business decisions)

If you want strict DDD cleanliness, a nice pattern is:

- `tenancy` = mode rules and tenant resolution
- `platform` (new package) = cross-service rollout contract (`platform.rollout-mode`, capability handshake)

But if you prefer minimal refactor and current style, putting it in `tenancy` is absolutely consistent.

---

**User**

yap

---

**Cursor**

I’ll update the spec to explicitly state package placement conventions (primarily `tenancy`) so implementation aligns with existing structure.

Updated the spec with package-placement guidance aligned to your current architecture:

- Added `## 16. Package placement (aligned with current design approach)`

It now explicitly states:

- default placement in `tenancy` for mode/runtime behavior,
- what stays in existing domain packages (`user`, `tenant`, `subscription`, `settings`),
- what should remain in `infrastructure.config` (wiring only),
- per-service placement guidance for IAM, Gateway, and Billing,
- optional future `platform` package only if rollout contract scope grows beyond tenancy.

This should make implementation decisions much clearer for the team.

---

**User**

PLATFORM_BILLING_OPERATOR is too much

---

**Cursor**

Agreed — that’s over-segmented for now. I’ll simplify the spec to a single platform-level authority and remove the extra billing-specific operator authority.

Updated — removed `PLATFORM_BILLING_OPERATOR` and standardized on just:

- `PLATFORM_ADMIN`

Also adjusted the implementation checklist task to reflect a single platform-level authority model.
