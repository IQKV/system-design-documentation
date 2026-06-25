# Proposal: Per-Seat Payment Support

_Status: Draft — v1.0 · June 2026_

---

## 1. Executive Summary

`foundation-billing-service` currently supports **flat-rate subscriptions**: every plan has a
single `priceMinor` value and the checkout session always creates a Stripe line item with
`quantity = 1`.  
This document proposes the minimal, backward-compatible changes required to add
**per-seat (licensed-quantity) pricing** — where the invoice amount equals
`pricePerSeatMinor × seatCount` — while keeping the flat-rate path completely intact.

The proposal is deliberately scoped to a single pricing dimension (seats). Metered / usage-based
billing and tiered pricing are out of scope but are noted in §8.

---

## 2. Current State

### 2.1 Architecture Overview

The service follows hexagonal architecture with a clean separation between:

| Layer                 | Key types                                                                        |
| --------------------- | -------------------------------------------------------------------------------- |
| Domain / plan         | `Plan`, `PlanFeatures`, `PlanFeature`, `PlanFeatureRegistry`, `PlanFeatureGuard` |
| Domain / subscription | `Subscription`, `SubscriptionService`, `SubjectType`, `SubscriptionSubject`      |
| Application config    | `StripeProductSchema`, `BillingConfigurationProperties`, `BillingSeedRunner`     |
| Gateway port          | `PaymentGatewayPort` (Strategy interface)                                        |
| Gateway adapter       | `StripeGatewayAdapter`                                                           |
| Persistence           | MyBatis mappers: `PlanMapper`, `SubscriptionMapper`, Liquibase migrations        |
| Messaging             | `MessagingService`, `WebhookProcessingService`                                   |

Multi-provider readiness is already in place: `GatewayType` enum reserves `PAYPAL`, the active
gateway is selected at runtime via `iqkv.payment.gateway.type`.

### 2.2 Plan Catalog — How It Works Today

Plans are defined entirely in YAML (primary source of truth) and synced to the database and Stripe
at startup by `BillingSeedRunner`:

```
YAML (application-{env}.yml)
  └─▶ BillingConfigurationProperties (StripeProductSchema map)
        ├─▶ PlanFeatureRegistry (in-memory, zero-latency feature lookups)
        └─▶ BillingSeedRunner
              ├─▶ PlanMapper.upsert()   → plan_catalog table
              └─▶ PaymentGatewayPort.syncProduct()
                    └─▶ StripeGatewayAdapter → Stripe Product + Price API
```

`StripeProductSchema` fields relevant to this proposal:

```
planCode, displayName, description, billingPeriod (MONTHLY|ANNUAL),
priceMinor, currency, scope (TENANT|USER), active, trialPeriodDays,
features { maxUsers, maxProjects, Map<String,PlanFeature> features }
```

### 2.3 Subscription Lifecycle

```
Checkout session → Stripe-hosted page → customer.subscription.created webhook
                                          → WebhookProcessingService → subscriptions table
```

The `Subscription` entity already carries:

```java
Long quantity;            // seats — nullable, set from webhook event
String planId;            // Stripe price ID (external)
String subjectType;       // TENANT | USER
String subjectKey;        // tenantKey or userId
```

`CreateCheckoutSessionCommand` and `UpdateSubscriptionCommand` also already have a `quantity`
parameter that is forwarded to Stripe in `StripeGatewayAdapter`. The scaffolding for per-seat
is largely present.

### 2.4 Gap Analysis

| Concern                             | Current State                                                                            | Gap                                                                                                                                                                                                                                               |
| ----------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pricing model flag**              | No `pricingModel` field on `Plan` or `StripeProductSchema`                               | Cannot distinguish flat from per-seat at runtime                                                                                                                                                                                                  |
| **Stripe price type**               | `syncProduct` always creates `UNIT_AMOUNT` recurring prices (flat per-interval)          | Per-seat needs the same `UNIT_AMOUNT` but with variable quantity; Stripe handles this automatically — the Stripe price creation itself does not differ, but the _per-seat price_ must be stored separately from the flat price to avoid ambiguity |
| **pricePerSeatMinor**               | `Plan.priceMinor` is the only price field                                                | Flat plans use total price; per-seat plans need price-per-seat; they cannot share the same field without confusion                                                                                                                                |
| **Seat validation at checkout**     | `SubscriptionService.createCheckoutSession` passes `request.quantity()` straight through | No check that `quantity ≥ 1`, or that `quantity ≤ maxUsers` (when maxUsers > 0)                                                                                                                                                                   |
| **Seat change endpoint**            | Existing `updateSubscription` accepts `quantity`; no dedicated seat-change path          | Self-service seat adjustments need proration guidance and seat-cap enforcement                                                                                                                                                                    |
| **YAML / config schema**            | `StripeProductSchema` has no `pricingModel` field                                        | BillingSeedRunner cannot differentiate plan types                                                                                                                                                                                                 |
| **DB schema**                       | `plan_catalog` has no `pricing_model` or `price_per_seat_minor` column                   | Gateway sync and runtime cannot persist/read per-seat data                                                                                                                                                                                        |
| **Entitlement propagation**         | `MessagingService.publishSubscriptionCreated/Updated` does not include `seatCount`       | Downstream IAM service has no reliable way to enforce seat limits on user-invite actions                                                                                                                                                          |
| **PlanFeatures.maxUsers semantics** | Already `0 = unlimited, N = cap` — well-defined                                          | No gap; this field becomes the seat cap for per-seat plans                                                                                                                                                                                        |

---

## 3. Proposed Design

### 3.1 Core Principle: Two Pricing Modes, One Stripe Price Type

Stripe does not have a separate "per-seat" price type. Both flat and per-seat plans use
`UNIT_AMOUNT` recurring prices. The difference is behavioral:

- **Flat (`FLAT`)**: checkout line item uses `quantity = 1`; the price already represents the
  full plan cost.
- **Per-seat (`PER_SEAT`)**: checkout line item uses `quantity = seatCount`; the price
  represents the cost per seat per billing period.

This means `syncProduct` in `StripeGatewayAdapter` requires no fundamental change in Stripe API
calls — but the service layer must route the right `priceMinor` to the Stripe price creation
(flat total vs. per-seat unit price), and must enforce seat validation before creating the checkout
session.

### 3.2 New `PricingModel` Enum

```java
// package com.iqkv.foundation.billingservice.plan
public enum PricingModel {
    FLAT,       // fixed price per billing period regardless of seat count
    PER_SEAT    // price multiplied by quantity (seat count) per billing period
}
```

### 3.3 Changes to `StripeProductSchema` (YAML binding)

Add one new optional field with a `FLAT` default so all existing YAML files continue to work
without modification:

```java
public record StripeProductSchema(
    @NotBlank String planCode,
    @NotBlank String displayName,
    String description,
    @NotBlank String billingPeriod,
    @NotNull @Positive Integer priceMinor,   // flat total OR per-seat unit price depending on pricingModel
    @NotBlank String currency,
    PlanFeatures features,
    @NotBlank String scope,
    Boolean active,
    Integer trialPeriodDays,
    PricingModel pricingModel               // NEW — defaults to FLAT when absent
) {
    public PricingModel effectivePricingModel() {
        return pricingModel != null ? pricingModel : PricingModel.FLAT;
    }
}
```

**Semantics of `priceMinor` under the new model:**

| `pricingModel` | Meaning of `priceMinor`            | Example                  |
| -------------- | ---------------------------------- | ------------------------ |
| `FLAT`         | Total price for the billing period | `1500` = $15.00/month    |
| `PER_SEAT`     | Price per seat per billing period  | `500` = $5.00/seat/month |

No rename is needed; the field meaning is clear from context.

### 3.4 Changes to `Plan` (domain entity)

```java
// New fields
private String pricingModel;    // "FLAT" | "PER_SEAT" — persisted to plan_catalog
```

`priceMinor` stays as-is; its semantics follow `pricingModel` as described above.
`PlanFeatures.maxUsers` (already `0 = unlimited, N = cap`) becomes the seat ceiling for
`PER_SEAT` plans; no new field is needed.

### 3.5 Database Migration

One new column on `plan_catalog`:

```xml
<!-- db/changelog/changes/20260625120000-add-pricing-model-to-plan-catalog.xml -->
<changeSet id="20260625120000-add-pricing-model" author="foundation-team">
  <addColumn tableName="plan_catalog">
    <column name="pricing_model" type="VARCHAR(16)" defaultValue="FLAT">
      <constraints nullable="false"/>
    </column>
  </addColumn>
</changeSet>
```

The `NOT NULL` with `DEFAULT 'FLAT'` means all existing rows automatically become `FLAT` plans —
no data migration needed.

### 3.6 Changes to `BillingSeedRunner`

`syncProduct` already calls `plan.setPriceMinor(schema.priceMinor())`. Add:

```java
plan.setPricingModel(schema.effectivePricingModel().name());
```

No change to `StripeGatewayAdapter.syncProduct()` is required. Stripe does not differentiate
flat vs. per-seat at the price object level; both are `UNIT_AMOUNT` recurring prices. The
service-side `pricingModel` flag governs how `quantity` is set at checkout time.

### 3.7 Changes to `SubscriptionService`

#### 3.7.1 Seat Validation Helper

```java
private void validateSeatCount(Plan plan, long requestedSeats) {
    if (requestedSeats < 1) {
        throw new IllegalArgumentException("Seat count must be at least 1");
    }
    PlanFeatures features = planFeatureRegistry.forPlan(plan.getPlanCode());
    int maxUsers = features.maxUsers();
    if (maxUsers > 0 && requestedSeats > maxUsers) {
        throw new SeatLimitExceededException(plan.getPlanCode(), requestedSeats, maxUsers);
    }
}
```

`SeatLimitExceededException` — a new domain exception that maps to `HTTP 422 Unprocessable Entity`.

#### 3.7.2 `createCheckoutSession` / `createCheckoutSessionForSubject`

```java
// Determine effective quantity for the Stripe line item
long effectiveQuantity = resolveEffectiveQuantity(plan, request.quantity());

// Validate only for PER_SEAT plans
if (PricingModel.PER_SEAT.name().equals(plan.getPricingModel())) {
    validateSeatCount(plan, effectiveQuantity);
}

var command = new CreateCheckoutSessionCommand(
    externalCustomerId, plan.getExternalPriceId(),
    request.successUrl(), request.cancelUrl(),
    trialDays,
    effectiveQuantity,          // was: request.quantity()
    request.allowPromotionCodes(),
    metadata
);
```

```java
private long resolveEffectiveQuantity(Plan plan, Long requested) {
    if (PricingModel.PER_SEAT.name().equals(plan.getPricingModel())) {
        return requested != null && requested > 0 ? requested : 1L;
    }
    return 1L; // flat plans always use quantity=1
}
```

This keeps flat plans untouched — they continue to pass `quantity = 1` regardless of what the
caller sends.

#### 3.7.3 Seat-Adjustment Endpoint (new self-service operation)

Rather than overloading the existing `updateSubscription` (which is used for plan upgrades),
expose a dedicated seat-adjustment operation. This makes intent explicit and allows different
proration defaults:

```java
// SubscriptionService
public void adjustSeats(String tenantKey,
                        String externalSubscriptionId,
                        AdjustSeatsRequest request) {

    Subscription subscription = subscriptionMapper
        .findByExternalSubscriptionId(externalSubscriptionId)
        .orElseThrow(() -> new ResourceNotFoundException("Subscription not found: " + externalSubscriptionId));

    if (!tenantKey.equals(subscription.getTenantKey())) {
        throw new TenantContextMismatchException(...);
    }

    Plan plan = planMapper.findByExternalPriceId(subscription.getPlanId())
        .orElseThrow(() -> new ResourceNotFoundException("Plan not found for priceId: " + subscription.getPlanId()));

    if (!PricingModel.PER_SEAT.name().equals(plan.getPricingModel())) {
        throw new IllegalStateException("Plan " + plan.getPlanCode() + " is not a per-seat plan");
    }

    validateSeatCount(plan, request.seatCount());

    paymentGatewayPort.updateSubscription(new UpdateSubscriptionCommand(
        externalSubscriptionId,
        null,                              // no plan change
        request.seatCount(),
        request.prorationBehavior() != null
            ? request.prorationBehavior()
            : "create_prorations",         // safe default: prorate immediately
        Map.of("tenantKey", tenantKey)
    ));

    // The webhook handler will pick up the quantity update and persist it.
    log.info("Seat adjustment initiated: tenantKey={}, subscription={}, seats={}",
        tenantKey, externalSubscriptionId, request.seatCount());
}
```

**REST endpoint:**

```
PATCH /api/v1/billing/subscriptions/{tenantKey}/{subscriptionId}/seats
Body: { "seatCount": 15, "prorationBehavior": "create_prorations" }
```

### 3.8 Entitlement Propagation

`MessagingService.publishSubscriptionCreated` and `publishSubscriptionUpdated` currently include
`planCode` but not `seatCount`. Downstream IAM needs the seat count to enforce user-invite limits
at runtime without calling back to the billing service.

Add `seatCount` (nullable `Long`) to the existing subscription event payload:

```java
// In MessagingService, enrich the existing subscription.created / subscription.updated events:
{
  "tenantKey": "acme",
  "externalSubscriptionId": "sub_xxx",
  "subjectType": "TENANT",
  "subjectKey": "acme",
  "planCode": "pro-monthly-per-seat",
  "seatCount": 15            // NEW — null for FLAT plans
}
```

No schema break: `seatCount` is additive. Existing consumers that do not read it are unaffected.

### 3.9 `PlanFeatureRegistry` — No Change Required

`PlanFeatureRegistry` is already loaded from `StripeProductSchema.features` at startup and provides
`O(1)` lookups of `PlanFeatures.maxUsers`. The new `pricingModel` field on the schema is needed
only in `BillingSeedRunner` and `SubscriptionService`; it does not need to be stored in the
registry.

If callers ever need to know whether a plan is per-seat (e.g., for pricing-page rendering), the
`/internal/plans` endpoint can be extended to include `pricingModel` from the plan catalog — that
is a read-only addition to the existing response DTO.

---

## 4. YAML Configuration for a Per-Seat Plan

Example `application-sit.yml` addition. Existing flat plans require no changes:

```yaml
iqkv:
  billing:
    stripe:
      schema:
        products:
          # --- existing flat plans unchanged ---
          basic-monthly:
            planCode: "basic-monthly"
            # ...unchanged...

          # --- new per-seat plan ---
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
            pricingModel: "PER_SEAT" # NEW
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

---

## 5. Complete Changeset Summary

| #   | What changes                                                                             | Why                                           |
| --- | ---------------------------------------------------------------------------------------- | --------------------------------------------- |
| 1   | New `PricingModel` enum (`FLAT` / `PER_SEAT`) in `plan` package                          | Central type; avoids magic strings everywhere |
| 2   | `StripeProductSchema` — add `pricingModel` field (optional, default `FLAT`)              | YAML binding; backward compatible             |
| 3   | `Plan` — add `String pricingModel` field                                                 | Domain entity carries pricing mode            |
| 4   | `PlanMapper` — add `pricing_model` to SELECT/INSERT/UPDATE                               | Persistence of new field                      |
| 5   | Liquibase migration — `ADD COLUMN pricing_model VARCHAR(16) DEFAULT 'FLAT'`              | DB schema                                     |
| 6   | `BillingSeedRunner.syncProduct` — set `plan.pricingModel`                                | Seeds pricing model from YAML                 |
| 7   | `SubscriptionService` — `resolveEffectiveQuantity` + `validateSeatCount` helpers         | Core seat logic                               |
| 8   | `SubscriptionService.createCheckoutSession[ForSubject]` — use `resolveEffectiveQuantity` | Checkout quantity routing                     |
| 9   | `SubscriptionService.adjustSeats` + `AdjustSeatsRequest` record                          | Dedicated seat-change operation               |
| 10  | `SubscriptionRestResource` (or new `SeatRestResource`) — `PATCH .../seats`               | REST endpoint                                 |
| 11  | New `SeatLimitExceededException` → `HTTP 422`                                            | Domain exception for cap violations           |
| 12  | `MessagingService` — add `seatCount` to subscription event payloads                      | IAM / downstream integration                  |
| 13  | `StripeProductSchema` Javadoc update, existing YAML comments updated                     | Documentation                                 |

**`StripeGatewayAdapter.syncProduct` — no change.** Stripe price creation is identical for both
modes; both use `UNIT_AMOUNT`. The service layer controls quantity at checkout time.

**`PaymentGatewayPort` — no new methods.** `updateSubscription` already accepts a `quantity`
parameter. The adapter already passes it to Stripe.

**`PlanFeatureRegistry` — no change.** `maxUsers` is the seat ceiling; no new registry field needed.

**`WebhookProcessingService` — no change.** `toSubscription()` already maps `event.quantity()` to
`subscription.quantity`; the webhook path is already seat-aware.

---

## 6. Backward Compatibility

- All existing plans omit `pricingModel` → `effectivePricingModel()` returns `FLAT` → behavior
  identical to today.
- `plan_catalog` rows created before the migration get `pricing_model = 'FLAT'` by default.
- Checkout sessions for flat plans still pass `quantity = 1` (enforced by `resolveEffectiveQuantity`).
- No existing REST API contract changes; the new `PATCH .../seats` endpoint is additive.
- `seatCount: null` in messaging events is valid for consumers that already handle optional fields.

---

## 7. Open Design Decisions

### 7.1 Where to put `pricingModel` in `PlanFeatureRegistry`

**Option A** — Keep `pricingModel` only in `plan_catalog` (DB), read it through `PlanMapper`
in `SubscriptionService`.  
Pros: registry stays focused on feature entitlements. Cons: one extra DB read per checkout.

**Option B** — Load `pricingModel` into a lightweight companion map in `PlanFeatureRegistry`
(or a new `PlanMetadataRegistry`).  
Pros: zero-latency like feature lookups; consistent pattern. Cons: small scope creep in the registry.

**Recommendation:** Option A for the first implementation. The checkout path is not hot enough
to justify a second in-memory map. Revisit if a `PlanMetadataRegistry` becomes useful for other
metadata (e.g., tier ranking for upgrade validation).

### 7.2 Proration behaviour on seat adjustments

The proposed default is `"create_prorations"` (immediate credit/charge). This is Stripe's
recommended default for mid-cycle seat changes. Teams that want to defer to end-of-period
can pass `"none"` or `"always_invoice"` in the request body.

### 7.3 Minimum seat count

The proposal enforces `quantity ≥ 1`. Some products enforce a higher minimum (e.g., 3 seats).
This can be added as a `minUsers` field on `PlanFeatures` in a follow-up without touching the
core per-seat logic.

### 7.4 Overage handling

The current model hard-caps seats at `maxUsers` (rejects the checkout). An alternative is to allow
overage and bill at a higher per-seat rate. This requires tiered Stripe pricing and is out of scope
here.

---

## 8. Out of Scope

The following are explicitly deferred:

- **Metered / usage-based billing** — Stripe `metered` price aggregation (report usage via API;
  invoice created at period end). Requires a new `METERED` `PricingModel` variant and a
  usage-reporting endpoint/job.
- **Tiered pricing** — volume discounts at different seat thresholds (Stripe `tiers`).
- **Per-project or other quantity dimensions** — `maxProjects` is present in `PlanFeatures` but
  billing by project count is not proposed here.
- **Add-on line items** — multiple Stripe subscription items (e.g., base fee + per-seat add-on).
  The current data model assumes one price per subscription.
- **Seat-usage reconciliation** — reporting how many of the purchased seats are actually occupied;
  requires IAM → billing event feed.

---

## 9. Phased Implementation Plan

### Phase 1 — Model & Schema (no behavior change, fully backward compatible)

1. Add `PricingModel` enum.
2. Add `pricingModel` to `StripeProductSchema` and `Plan`.
3. Create Liquibase migration.
4. Update `PlanMapper` (column in SELECT / INSERT / UPDATE).
5. Update `BillingSeedRunner.syncProduct` to persist `pricingModel`.
6. Update internal plans endpoint response DTO to expose `pricingModel`.
7. Deploy and verify existing plans still work.

### Phase 2 — Checkout Routing

1. Implement `resolveEffectiveQuantity` in `SubscriptionService`.
2. Implement `validateSeatCount` + `SeatLimitExceededException`.
3. Apply both to `createCheckoutSession` and `createCheckoutSessionForSubject`.
4. Add `pro-monthly-per-seat` plan to a non-production environment YAML.
5. Manual end-to-end test: checkout with `quantity = 5` on a per-seat plan.

### Phase 3 — Seat Adjustment API

1. Add `AdjustSeatsRequest` record.
2. Implement `SubscriptionService.adjustSeats`.
3. Expose `PATCH /api/v1/billing/subscriptions/{tenantKey}/{subscriptionId}/seats`.
4. Verify proration invoices are generated in Stripe test mode.

### Phase 4 — Downstream Integration

1. Add `seatCount` field to subscription messaging events.
2. Coordinate with IAM service to consume `seatCount` for user-invite enforcement.
3. Update public `/internal/plans` response to include `pricingModel`.
