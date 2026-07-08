# Per-Seat Payment Support — Implementation Record

_Status: Implemented · June 2026_

---

## 1. Overview

This document records what was actually built, file by file. It supersedes the proposal
(`proposal-per-seat-payments.md`) as the authoritative reference for the per-seat feature.

The implementation adds **`PER_SEAT` pricing** to `foundation-billing-service` and propagates
the change to all downstream consumer services. Flat-rate plans are completely unaffected —
the default for every existing and new plan is `FLAT` unless explicitly overridden.

---

## 2. New Files

### `plan/PricingModel.java`

```
com.iqkv.foundation.billingservice.plan.PricingModel
```

Enum with two values:

| Value      | Stripe line item       | `priceMinor` meaning              |
| ---------- | ---------------------- | --------------------------------- |
| `FLAT`     | `quantity = 1` always  | Total price per billing period    |
| `PER_SEAT` | `quantity = seatCount` | Price per seat per billing period |

Both modes use a Stripe `UNIT_AMOUNT` recurring price. The distinction is enforced in
`SubscriptionService`, not at the Stripe API level.

---

### `shared/exception/SeatLimitExceededException.java`

Maps to **HTTP 422 Unprocessable Entity**. Carries `planCode`, `requested`, and `limit` so
the UI can render a targeted upgrade prompt. Thrown by `SubscriptionService.validateSeatCount`
when `requestedSeats > plan.features.maxUsers` (and `maxUsers > 0`).

---

### `db/changelog/changes/20260625120000-add-pricing-model-to-plan-catalog.xml`

Liquibase migration:

```sql
ALTER TABLE plan_catalog
  ADD COLUMN pricing_model VARCHAR(16) NOT NULL DEFAULT 'FLAT';

ALTER TABLE plan_catalog
  ADD CONSTRAINT chk_plan_catalog_pricing_model
  CHECK (pricing_model IN ('FLAT', 'PER_SEAT'));
```

All pre-existing rows get `pricing_model = 'FLAT'` automatically. No data migration needed.
Registered in `db.changelog-master.xml`.

---

## 3. Modified Files

### `plan/Plan.java`

- Added `String pricingModel` field
- Added `pricingModel` parameter to the all-args constructor
- Added `getPricingModel()` / `setPricingModel()` accessors

---

### `infrastructure/config/StripeProductSchema.java`

- Added `PricingModel pricingModel` record component (optional — `null` is valid for legacy YAML)
- Added `effectivePricingModel()` helper: returns `pricingModel` when present, `PricingModel.FLAT` otherwise
- Added import for `PricingModel`

This is the YAML binding class. Omitting `pricingModel` in YAML is safe — `effectivePricingModel()`
returns `FLAT` and existing plan definitions require no changes.

---

### `infrastructure/config/BillingSeedRunner.java`

Added one line in `syncProduct()`:

```java
plan.setPricingModel(schema.effectivePricingModel().name());
```

Runs at startup. Persists `pricing_model` to `plan_catalog` for every plan defined in YAML.

---

### `plan/PlanFeatureRegistry.java`

**Significant rewrite.** Key changes:

- New inner record:
  ```java
  public record PlanCatalogEntry(String planCode, PlanEntitlement entitlement, PricingModel pricingModel) {}
  ```
- New `pricingRegistry` map (`planCode → PricingModel`) populated from `StripeProductSchema.effectivePricingModel()`
- New `pricingModelForPlan(String planCode)` — O(1) lookup, falls back to `FLAT`
- New `allEntries()` — returns `Map<String, PlanCatalogEntry>` bundling features + pricingModel; used by the internal plans endpoint
- `all()` retained unchanged for backward compatibility

---

### `plan/PlanInternalRestResource.java`

Both response records updated:

**`PlanCatalogEntry`** (service-to-service endpoint `GET /api/v1/billing/internal/plans`):

```java
public record PlanCatalogEntry(String planCode, PlanEntitlement entitlement, PricingModel pricingModel) {}
```

`listPlanCatalog()` now uses `planFeatureRegistry.allEntries()`.

**`PublicPlanEntry`** (pricing page endpoint `GET /api/v1/billing/internal/plans/public`):

```java
public record PublicPlanEntry(
    String planCode, String displayName, String description,
    String billingPeriod, Integer priceMinor, String currency,
    PlanEntitlement features, String scope, Boolean active,
    PricingModel pricingModel   // NEW
) {}
```

`listPublicPlans()` passes `schema.effectivePricingModel()`.

---

### `mappers/PlanMapper.xml`

`pricing_model` added to:

- `resultMap` — maps `pricing_model` column → `Plan.pricingModel` field
- All five `SELECT` statements (`findById`, `findByPlanCode`, `findByExternalPriceId`, `findAllActive`, `findAll`)
- `INSERT` — `COALESCE(#{pricingModel}, 'FLAT')`
- `UPDATE` — `pricing_model = COALESCE(#{pricingModel}, 'FLAT')`

---

### `subscription/SubscriptionDtos.java`

Added:

```java
public record AdjustSeatsRequest(
    @NotNull @Min(1) Long seatCount,
    String prorationBehavior   // nullable; defaults to "create_prorations" in service
) {}
```

---

### `subscription/SubscriptionService.java`

**Most significant change.** Added:

**New dependency:** `PlanFeatureRegistry planFeatureRegistry` (constructor-injected).

**Private helpers:**

```java
// Returns 1L for FLAT plans (ignoring caller input).
// Returns requested (≥1) for PER_SEAT plans.
private long resolveEffectiveQuantity(Plan plan, Long requested)

// Throws SeatLimitExceededException if requestedSeats > plan.maxUsers (when maxUsers > 0).
private void validateSeatCount(Plan plan, long requestedSeats)
```

**Checkout routing** — applied to both `createCheckoutSession` and `createCheckoutSessionForSubject`:

```java
final long effectiveQuantity = resolveEffectiveQuantity(plan, request.quantity());
if (PricingModel.PER_SEAT.name().equals(plan.getPricingModel())) {
    validateSeatCount(plan, effectiveQuantity);
}
// effectiveQuantity passed to CreateCheckoutSessionCommand (was: request.quantity())
```

**New method `adjustSeats`:**

- Verifies tenant ownership
- Resolves plan from `subscription.planId` via `planMapper.findByExternalPriceId`
- Rejects with `IllegalStateException` if plan is not `PER_SEAT`
- Calls `validateSeatCount`
- Delegates to `paymentGatewayPort.updateSubscription` with `quantity = seatCount`, `priceId = null`
- Default proration: `"create_prorations"` (overridable per request)
- Records metric `billing_seat_adjustments_total`

---

### `subscription/SubscriptionRestResource.java`

New endpoint:

```
PATCH /api/v1/billing/subscriptions/{tenantKey}/{externalSubscriptionId}/seats
```

- Authority: `TENANT_OWNER` or `ADMIN`
- Tenant ownership enforced via `enforceOwnership`
- Body: `AdjustSeatsRequest` (validated with `@Valid`)
- Returns `204 No Content` — gateway webhook writes the updated `quantity` back to the local cache
- HTTP 422 on seat limit exceeded, 404 on subscription not found, 403 on tenant mismatch

---

### `infrastructure/messaging/SubscriptionEvent.java`

- Added `Long seatCount` field (nullable — `null` for `FLAT` plans)
- Updated all-args constructor from 7 to 8 parameters
- Added `getSeatCount()` / `setSeatCount()` accessors

---

### `infrastructure/messaging/MessagingService.java`

All internal `SubscriptionEvent` constructions updated to 8-arg form.

New overloads with explicit `seatCount`:

```java
publishSubscriptionCreated(..., String planCode, Long seatCount)
publishSubscriptionUpdated(..., String planCode, Long seatCount)
```

Old 5-arg overloads (`planCode` only, no `seatCount`) marked `@Deprecated` and delegate to the
new 6-arg versions with `seatCount = null`. This preserves compile-time compatibility with any
caller that has not yet been updated.

---

### `resources/application.yml`

Schema comment block updated to document `pricingModel`:

- Explains `FLAT` vs `PER_SEAT` semantics
- Clarifies `priceMinor` interpretation under each mode
- Notes `maxUsers` doubles as seat ceiling for `PER_SEAT` plans
- Two inline examples: a `FLAT` plan and a `PER_SEAT` plan

---

### `resources/application-{local,sit,uat,prd}.yml`

All existing plan definitions updated to include `pricingModel: "FLAT"` explicitly.
`application-local.yml` and `application-sit.yml` include a commented-out
`pro-monthly-per-seat` sample plan for easy test activation.

---

## 4. Downstream Consumer Updates

### `foundation-iam-service` — `plan/` package

| File                    | Change                                                                                                                                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `PlanEntitlement.java`  | Added `String pricingModel` record component; added compact constructor with defensive `features` map copy; added `has(String code)` method; added `isPerSeat()` helper; `NONE` sentinel passes `null` |
| `PlanFeatureGuard.java` | `hasFeature()` now delegates to `features.has()` instead of duplicating the inline map lookup                                                                                                          |

`PlanResolver` and `PlanCatalogRestTemplateConfig` — no changes needed.

### `foundation-cms-service` — `plan/` package

| File                   | Change                                                                                            |
| ---------------------- | ------------------------------------------------------------------------------------------------- |
| `PlanEntitlement.java` | Added `String pricingModel` record component; updated `NONE` sentinel; added `isPerSeat()` helper |

### `foundation-microservice-project-layout` — `plan/` package

| File                   | Change                      |
| ---------------------- | --------------------------- |
| `PlanEntitlement.java` | Same changes as cms-service |

### `foundation-ui-app` — billing API types

| File                                           | Change                                                                                                                    |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `src/shared/api/billing.ts`                    | Added `PricingModel` union type; added `pricingModel?: PricingModel \| null` to `Plan` and `PlanEntitlement` interfaces   |
| `src/features/manage-billing/ui/plan-card.tsx` | `entitlement` fallback object includes `pricingModel: null`; price label renders `/ seat / {period}` for `PER_SEAT` plans |

---

## 5. What Was Not Changed

| Component                                  | Reason                                                                                                                  |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `StripeGatewayAdapter.syncProduct`         | Stripe price creation is identical for both modes (`UNIT_AMOUNT` recurring). No change needed.                          |
| `PaymentGatewayPort`                       | `updateSubscription` already accepts `quantity`. No new methods required.                                               |
| `WebhookProcessingService`                 | Already maps `event.quantity()` → `subscription.quantity`. Per-seat quantity arrives from Stripe webhook automatically. |
| `PlanEntitlement` record (billing-service) | `pricingModel` belongs on `StripeProductSchema` and `Plan`, not on the feature-entitlement record.                      |
| `EntitlementEvaluator`                     | Entitlement checks are feature-flag based; pricing mode does not affect them.                                           |

---

## 6. Downstream IAM Changes (Deferred)

The proposal identified a second IAM-side gap: for `PER_SEAT` plans, IAM currently enforces
`maxUsers` (plan ceiling) at invite-accept time, but should enforce the _purchased seat count_
(`Subscription.quantity`) instead. This requires:

1. Adding `seatCount` to `SubscriptionEvent` in IAM ✅ (field exists in billing `SubscriptionEvent`; IAM's local copy needs updating)
2. Adding `activeSeatCount` column to IAM `tenants` table
3. Caching `seatCount` in `SubscriptionEventConsumer` alongside `planCode`
4. Updating the three quota-check sites in IAM (`InvitationServiceImpl`, `TenantServiceImpl`, `SingleTenantSignupStrategy`) to use `min(activeSeatCount, maxUsers)` as the effective limit

This work is tracked separately and does not block the billing-service deployment.

---

## 7. How to Add a PER_SEAT Plan

1. Add the plan definition to the appropriate `application-{env}.yml` under
   `iqkv.billing.stripe.schema.products`:

```yaml
pro-monthly-per-seat:
  planCode: "pro-monthly-per-seat"
  displayName: "Pro Monthly (per seat)"
  description: "Billed per active user seat. $5 per seat per month."
  billingPeriod: "MONTHLY"
  priceMinor: 500 # $5.00 per seat per month
  currency: "USD"
  scope: "TENANT"
  active: true
  trialPeriodDays: 14
  pricingModel: "PER_SEAT"
  features:
    maxUsers: 200 # seat ceiling; 0 = unlimited
    maxProjects: 0
    features:
      priority_support:
        title: "Priority Support"
        value: "true"
        description: "Access to priority support channel"
      advanced_analytics:
        title: "Advanced Analytics"
        value: "true"
        description: "Access to detailed analytics dashboard"
```

2. Restart the service. `BillingSeedRunner` will:
   - Upsert the plan row in `plan_catalog` with `pricing_model = 'PER_SEAT'`
   - Create (or verify) the Stripe Product and `UNIT_AMOUNT` recurring Price
   - Load the plan into `PlanFeatureRegistry` with `PricingModel.PER_SEAT`

3. At checkout, pass `quantity` in the request body:

```
POST /api/v1/billing/subscriptions/{tenantKey}/checkout
{
  "planCode": "pro-monthly-per-seat",
  "quantity": 15,
  "successUrl": "...",
  "cancelUrl": "..."
}
```

`SubscriptionService` validates `15 ≤ maxUsers (200)` and creates the Stripe checkout session
with `lineItem.quantity = 15`. Invoice = $5.00 × 15 = $75.00/month.

4. To adjust seats mid-cycle:

```
PATCH /api/v1/billing/subscriptions/{tenantKey}/{subId}/seats
{
  "seatCount": 20,
  "prorationBehavior": "create_prorations"
}
```
