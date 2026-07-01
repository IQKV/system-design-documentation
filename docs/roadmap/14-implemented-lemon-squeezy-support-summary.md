# Lemon Squeezy Support - Implementation Summary

**Status:** Implemented with follow-up test hardening still open  
**Implementation Date:** June 29-30, 2026  
**Primary Target:** `foundation-billing-service`

---

## 1. Overview

This document records what was actually implemented from
`14-proposal-lemon-squeezy-support.md`.

The result is a true **multi-gateway billing implementation**. Stripe remains supported,
and Lemon Squeezy is now available behind the same `PaymentGatewayPort` abstraction,
selected at runtime via configuration. The supporting schema, webhook normalization,
application wiring, operator surfaces, and deployment configuration were updated so the
platform can run with either gateway without changing downstream service contracts.

The proposal's core architectural direction was preserved:

- `PaymentGatewayPort` remains the gateway-agnostic seam
- `WebhookProcessingService` continues consuming normalized events
- gateway selection happens by configuration via `iqkv.payment.gateway.type`
- Stripe-specific leakage was removed from plan catalog and runtime data models

What is still intentionally open is the **Phase 6 test-hardening work**:

- dedicated `LemonSqueezyGatewayAdapter` unit coverage
- ArchUnit enforcement for adapter isolation
- end-to-end Lemon Squeezy webhook integration coverage

---

## 2. Delivered Outcome

### 2.1 Runtime Behavior

The billing service now supports:

- `STRIPE` and `LEMON_SQUEEZY` as runtime-selectable `GatewayType` values
- conditional bean wiring via `@ConditionalOnGateway`
- a dedicated Lemon Squeezy REST adapter and webhook endpoint
- read-only variant verification for plan seeding instead of Stripe-style product creation
- Lemon Squeezy customer portal sessions
- Lemon Squeezy subscription lifecycle operations: checkout, update, cancel, pause, reactivate
- Lemon Squeezy refunds via order-based semantics
- normalized webhook ingestion mapped into the existing internal event model

### 2.2 Data Model Changes

The schema was extended so billing data is gateway-aware:

- `billing_settings.gateway_type`
- `subscriptions.gateway_type`
- `subscriptions.external_order_id`
- `plan_catalog.gateway_type`

This makes subscription records, billing settings, and plan catalog entries observable and
operable across both Stripe and Lemon Squeezy.

### 2.3 Configuration Model

The plan catalog no longer lives under Stripe-specific configuration.

Old path:

```yaml
iqkv.billing.stripe.schema.products
```

New path:

```yaml
iqkv.billing.plan-catalog.products
```

Lemon Squeezy runtime configuration was added alongside the existing payment configuration:

```yaml
iqkv:
  payment:
    gateway:
      type: ${PAYMENT_GATEWAY_TYPE:STRIPE}
  lemon-squeezy:
    api-key: ${LEMON_SQUEEZY_API_KEY:}
    store-id: ${LEMON_SQUEEZY_STORE_ID:}
    webhook-secret: ${LEMON_SQUEEZY_WEBHOOK_SECRET:}
    portal-return-url: ${LEMON_SQUEEZY_PORTAL_RETURN_URL:http://localhost:3000/billing}
```

---

## 3. Implemented Phases

### Phase 1 - Stripe Leakage Removal and Gateway Neutrality

Implemented:

- Added `LEMON_SQUEEZY` to `GatewayType`
- Introduced `@ConditionalOnGateway` and `GatewayCondition`
- Annotated `StripeGatewayAdapter` with `@ConditionalOnGateway(GatewayType.STRIPE)`
- Annotated `StripeWebhookRestResource` with `@ConditionalOnGateway(GatewayType.STRIPE)`
- Renamed `StripeProductSchema` to `ProductSchema`
- Flattened `BillingConfigurationProperties` so plan catalog configuration is gateway-neutral
- Updated `application.yml` to use `iqkv.billing.plan-catalog.products`

Outcome:

- the billing service no longer treats Stripe as the structural default everywhere
- the plan catalog is configuration-driven and reusable across gateways
- Stripe and Lemon Squeezy can coexist behind the same application-level contract

### Phase 2 - Lemon Squeezy Configuration

Implemented:

- Added `LemonSqueezyConfigurationProperties`
- Registered configuration binding for Lemon Squeezy
- Added gateway-specific runtime properties and placeholders
- wired deployment-time secrets and values in Helm / CI work completed alongside the service

Outcome:

- production deployments can supply Lemon Squeezy credentials cleanly through environment and Helm values

### Phase 3 - Lemon Squeezy Adapter and Webhooks

Implemented:

- Added `LemonSqueezyRestClientConfig`
- Implemented `LemonSqueezyGatewayAdapter`
- Implemented customer creation
- Implemented checkout session creation
- Implemented subscription update / cancel / pause / reactivate
- Implemented refund creation
- Implemented read-only `syncProduct` verification flow
- Implemented webhook verification and event parsing
- Implemented Lemon Squeezy customer portal session creation
- Added `LemonSqueezyWebhookRestResource`

Outcome:

- Lemon Squeezy is a first-class payment gateway in the same service boundary as Stripe
- the adapter maps provider-specific behavior into the existing port and webhook abstractions

### Phase 4 - Schema Hardening

Implemented:

- Liquibase migration for `billing_settings.gateway_type`
- Liquibase migration for `subscriptions.gateway_type`
- Liquibase migration for `subscriptions.external_order_id`
- Liquibase migration for `plan_catalog.gateway_type`
- master changelog registration

Outcome:

- local persistence reflects gateway origin and LS-specific order semantics
- refunds and reporting can distinguish Stripe and Lemon Squeezy records reliably

### Phase 5 - Application Layer Wiring

Implemented:

- Added `gatewayType()` support across webhook event types
- added `gatewayType` to domain models including `Subscription`, `BillingSettings`, `Plan`, and `UserBillingSettings`
- updated MyBatis mappers and XML mappings for the new columns
- `BillingSeedRunner` now works with `ProductSchema`, uses `externalVariantId`, conditionally updates plans, and persists `gatewayType`
- `WebhookProcessingService` now writes `gatewayType` and `externalOrderId`
- `BillingSettingsService.createBillingSettings()` now sets the active gateway type
- `PlatformModeValidatorImpl` now enforces `defaultContactEmail` when Lemon Squeezy runs in `SINGLE_TENANT`
- `SecurityConfig` now permits the Lemon Squeezy webhook path

Outcome:

- gateway-awareness is carried end-to-end from plan seed to webhook persistence
- Lemon Squeezy's data model differences are handled without leaking into consumers

### Phase 6 - Tests

Not yet completed:

- `LemonSqueezyGatewayAdapterTest`
- ArchUnit rule updates for adapter isolation
- dedicated webhook round-trip integration test

These were explicitly left open at the end of implementation and remain the main follow-up work.

---

## 4. Key Files Added or Significantly Changed

### New or New-for-Feature Files

- `gateway/adapter/lemonsqueezy/LemonSqueezyGatewayAdapter.java`
- `gateway/adapter/lemonsqueezy/LemonSqueezyRestClientConfig.java`
- `webhook/LemonSqueezyWebhookRestResource.java`
- `infrastructure/config/LemonSqueezyConfigurationProperties.java`
- `infrastructure/config/ConditionalOnGateway.java`
- `infrastructure/config/GatewayCondition.java`
- `db/changelog/changes/20260701000000-add-gateway-type-to-billing-settings.xml`
- `db/changelog/changes/20260701000001-add-gateway-type-to-subscriptions.xml`
- `db/changelog/changes/20260701000002-add-external-order-id-to-subscriptions.xml`
- `db/changelog/changes/20260701000003-add-gateway-type-to-plan-catalog.xml`

### Core Refactors

- `gateway/GatewayType.java`
- `config/BillingConfigurationProperties.java`
- `config/ProductSchema.java`
- `config/BillingSeedRunner.java`
- `gateway/PaymentGatewayPort.java`
- `subscription/WebhookProcessingService.java`
- `webhook/StripeWebhookRestResource.java`
- `gateway/adapter/stripe/StripeGatewayAdapter.java`
- `resources/application.yml`
- MyBatis mapper XML files for subscriptions, plans, billing settings, and user billing settings

---

## 5. API and Integration Changes

### Billing Service

Added / enabled:

- `POST /api/v1/billing/webhooks/lemon-squeezy`

Behavioral change:

- internal subscription, invoice, payment-failure, and refund events now carry `gatewayType`
- invoice success handling can persist `externalOrderId`

### Operational / Deployment Alignment

Implementation was also reflected in surrounding platform assets:

- billing README and API documentation updated for Stripe + Lemon Squeezy support
- landing and marketing surfaces updated to present both billing gateways
- system design documentation updated to treat multi-gateway billing as completed

---

## 6. Security and Operational Notes

- Lemon Squeezy webhook verification uses HMAC-SHA256 with the provider secret
- webhook endpoints remain public only for gateway callback traffic
- sensitive overrides in deployment remain expected to use `--set-string`
- `defaultContactEmail` is now enforced for LS in `SINGLE_TENANT` mode to satisfy provider/customer requirements

---

## 7. Proposal Checklist Status

### Completed

- A1-A6: alignment and configuration refactor
- B1-B5: schema migrations
- C1-C9: gateway-aware event/domain model propagation
- D1-D5: mapper updates
- E1-E3: Stripe-side conditional wiring alignment
- F1-F9: Lemon Squeezy adapter, REST client, and webhook resource
- G1-G7: application-layer wiring and runtime hardening

### Still Open

- H1: `LemonSqueezyGatewayAdapterTest`
- H2: ArchUnit adapter-isolation rules
- H3: Lemon Squeezy webhook integration test

---

## 8. Final State

The Lemon Squeezy proposal is **functionally implemented**. The platform now supports
both Stripe and Lemon Squeezy through the same billing architecture, with gateway-aware
schema, application wiring, webhook processing, and operator-facing behavior.

The remaining work is not architectural; it is **test hardening and regression coverage**.
This document therefore supersedes the proposal as the authoritative summary of what was
actually built.
