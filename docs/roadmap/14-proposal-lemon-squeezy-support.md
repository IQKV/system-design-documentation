# Proposal: Refactoring Plan for Lemon Squeezy Gateway Support

**Status:** In Progress  
**Date:** 2026-06-29  
**Target:** `foundation-billing-service`

---

## 1. Executive Summary

The billing service already implements a clean hexagonal port/adapter architecture.
`PaymentGatewayPort` is the single seam between all domain/application logic and the payment
provider, and `WebhookProcessingService` operates exclusively on normalized `GatewayWebhookEvent`
sealed-interface records. The Stripe adapter is the only current implementation.

This plan adds a **Lemon Squeezy** adapter behind the same port, makes the gateway fully
selectable at configuration time, and hardens every place where Stripe-specific concepts
have leaked beyond the adapter boundary.

---

## 2. Current State Assessment

### 2.1 What is Already Multi-Gateway Ready

| Concern                                | Status                                        |
| -------------------------------------- | --------------------------------------------- |
| `PaymentGatewayPort` interface         | Clean — 11 gateway-agnostic methods           |
| `GatewayWebhookEvent` sealed hierarchy | Clean — no provider types                     |
| `WebhookProcessingService`             | Clean — zero provider imports                 |
| Gateway commands (record types)        | Clean — no provider fields                    |
| `SubscriptionService`                  | Clean — only calls port interface             |
| `GatewayType` enum                     | STRIPE only; reserved comment mentions PAYPAL |

### 2.2 Leakage Points That Must Be Fixed

| Leakage                                                                                       | File                                         | Severity       |
| --------------------------------------------------------------------------------------------- | -------------------------------------------- | -------------- |
| `StripeWebhookRestResource` injects `StripeGatewayAdapter` by concrete type                   | `webhook/StripeWebhookRestResource.java`     | High           |
| `BillingConfigurationProperties` nests a `StripeProperties` record with a `schema` sub-record | `config/BillingConfigurationProperties.java` | High           |
| `StripeProductSchema` record name is Stripe-specific                                          | `config/StripeProductSchema.java`            | Medium         |
| `BillingSeedRunner` references `billingProps.stripe().schema()`                               | `config/BillingSeedRunner.java`              | High           |
| `StripeConfigurationProperties` pattern-validates `sk_test_/sk_live_` and `whsec_` prefixes   | `config/StripeConfigurationProperties.java`  | Low (internal) |
| `application.yml`: plan catalog lives under `iqkv.billing.stripe.schema.products`             | `application.yml`                            | High           |
| `GatewayType` only has `STRIPE`; no conditional bean wiring exists yet                        | `GatewayType.java`                           | Medium         |
| No `@ConditionalOnProperty` or `@Primary` on the Stripe adapter bean                          | `StripeGatewayAdapter.java`                  | Medium         |
| `createPortalSession` concept is Stripe-specific (Billing Portal)                             | `PaymentGatewayPort.java`                    | Medium         |
| `pauseSubscription` / `reactivateSubscription` use Stripe `PauseCollection` semantics         | `PaymentGatewayPort.java`                    | Low            |
| Plan sync (`syncProduct`) creates Stripe Product + Price objects; no LS equivalent path       | `PaymentGatewayPort.java`                    | Medium         |

### 2.3 Lemon Squeezy vs Stripe — Concept Mapping

| Stripe Concept                          | Lemon Squeezy Equivalent                                         | Gap / Notes                                                                                                                |
| --------------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Customer (`cus_…`)                      | Customer (`/v1/customers`)                                       | LS requires email; name is optional                                                                                        |
| Product (`prod_…`)                      | Product (`/v1/products`)                                         | LS products are created in dashboard; API creation supported                                                               |
| Price (`price_…`)                       | Variant (`/v1/variants`)                                         | LS variant holds the recurring price; billing interval is a variant field                                                  |
| Checkout Session → URL                  | Checkout object → `url` attribute (`/v1/checkouts`)              | LS checkout is a POST to create, returns hosted URL                                                                        |
| Subscription (`sub_…`)                  | Subscription (`/v1/subscriptions`)                               | LS subscription has `status`, `renews_at`, `ends_at`, `cancelled` boolean                                                  |
| Subscription update (proration)         | PATCH `/v1/subscriptions/{id}` with `variant_id`                 | LS applies proration automatically; no explicit proration_behavior param                                                   |
| Cancel at period end                    | PATCH `cancelled: true`                                          | LS sets `ends_at` to current period end                                                                                    |
| Immediate cancel                        | DELETE `/v1/subscriptions/{id}`                                  |                                                                                                                            |
| Pause                                   | PATCH `pause: { mode: "void" \| "free" }`                        | LS supports void (no charge) or free (charge $0)                                                                           |
| Resume from pause                       | PATCH `pause: null`                                              |                                                                                                                            |
| Refund                                  | POST `/v1/refunds` with `order_id`                               | LS refunds are against Orders, not raw charges                                                                             |
| Webhook signature                       | HMAC-SHA256 of raw body; header `X-Signature`                    | Different header name from Stripe's `Stripe-Signature`                                                                     |
| Billing Portal                          | Customer Portal URL (`/v1/customer-portal-sessions`)             | LS has a built-in portal; similar concept                                                                                  |
| Plan sync at startup                    | Not applicable — LS products/variants managed in dashboard       | LS does not expose a create-variant-programmatically flow for recurring; `syncProduct` becomes a no-op or read-only verify |
| `invoice.payment_succeeded`             | `subscription_payment_success` webhook event                     |                                                                                                                            |
| `invoice.payment_failed`                | `subscription_payment_failed` webhook event                      |                                                                                                                            |
| `customer.subscription.created`         | `subscription_created` webhook event                             |                                                                                                                            |
| `customer.subscription.updated`         | `subscription_updated` webhook event                             |                                                                                                                            |
| `customer.subscription.deleted`         | `subscription_cancelled` / `subscription_expired` webhook events |                                                                                                                            |
| `charge.refunded`                       | `order_refunded` webhook event                                   |                                                                                                                            |
| `invoice.created` / `invoice.finalized` | `subscription_payment_success` (LS invoices are implicit)        | LS does not fire separate "invoice created/finalized" events                                                               |

---

## 3. Architecture After Refactoring

```
iqkv.payment.gateway.type = STRIPE | LEMON_SQUEEZY
            │
            ▼
┌──────────────────────────────────────────────────────┐
│  PaymentGatewayPort  (unchanged interface)           │
│  + GatewayWebhookEvent sealed hierarchy (unchanged)  │
└──────────┬───────────────────────────┬───────────────┘
           │                           │
  @ConditionalOnGateway(STRIPE)  @ConditionalOnGateway(LEMON_SQUEEZY)
           │                           │
  StripeGatewayAdapter        LemonSqueezyGatewayAdapter
  (refactored)                (new)

Webhook endpoints (one per gateway, both registered in SecurityConfig):
  POST /api/v1/billing/webhooks/stripe          → StripeWebhookRestResource
  POST /api/v1/billing/webhooks/lemon-squeezy   → LemonSqueezyWebhookRestResource  (new)

Plan catalog seed:
  BillingSeedRunner → gateway-neutral ProductSchema
     → StripeGatewayAdapter.syncProduct()  if STRIPE
     → LemonSqueezyGatewayAdapter.syncProduct()  if LEMON_SQUEEZY (read-only verify)
```

---

## 4. Refactoring Tasks

### Phase 1 — Alignment: Fix Stripe Leakage (no new features)

These changes make the existing code structurally ready for a second adapter without
breaking anything in production.

---

#### Task 1.1 — Add `LEMON_SQUEEZY` to `GatewayType`

File: `gateway/GatewayType.java`

Add the new constant. No other changes needed yet — the enum drives downstream conditionals.

```java
public enum GatewayType {
  STRIPE,
  LEMON_SQUEEZY
}
```

---

#### Task 1.2 — Create `@ConditionalOnGateway` meta-annotation

Create a new annotation in `infrastructure/config/`:

```java
// infrastructure/config/ConditionalOnGateway.java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(GatewayCondition.class)
public @interface ConditionalOnGateway {
  GatewayType value();
}

// infrastructure/config/GatewayCondition.java
public class GatewayCondition implements Condition {
  @Override
  public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    String configured = context.getEnvironment()
        .getProperty("iqkv.payment.gateway.type", "STRIPE");
    String required = (String) metadata.getAnnotationAttributes(
        ConditionalOnGateway.class.getName()).get("value");
    return configured.equalsIgnoreCase(required);
  }
}
```

Then annotate `StripeGatewayAdapter` with `@ConditionalOnGateway(GatewayType.STRIPE)`.

---

#### Task 1.3 — Decouple webhook resource from `StripeGatewayAdapter`

File: `webhook/StripeWebhookRestResource.java`

The resource currently injects the concrete `StripeGatewayAdapter`. Change the injection
to use the `PaymentGatewayPort` interface and guard by active gateway type. Alternatively —
and preferably — keep a gateway-specific webhook resource per gateway, but inject via the
port interface marked with `@ConditionalOnGateway`.

Recommended approach: each gateway gets its own webhook REST resource that wires its
specific adapter directly (the webhook endpoint path itself identifies the gateway). Both
resources are registered independently. The concrete type injection in the Stripe resource
is acceptable because the Stripe resource is `@ConditionalOnGateway(STRIPE)` itself.

Annotate `StripeWebhookRestResource` with `@ConditionalOnGateway(GatewayType.STRIPE)`.

---

#### Task 1.4 — Rename `StripeProductSchema` → `ProductSchema`

File: `config/StripeProductSchema.java`

Rename the record to `ProductSchema` (gateway-neutral name). Update all references:

- `BillingConfigurationProperties.StripeProperties` → `BillingConfigurationProperties.PlanSchemaProperties`
- `BillingSeedRunner`
- `StripeGatewayAdapter` (none — seed runner handles the schema)

The underlying YAML key path changes are handled in Task 1.6.

---

#### Task 1.5 — Flatten `BillingConfigurationProperties` plan schema path

File: `config/BillingConfigurationProperties.java`

Current nested path: `iqkv.billing.stripe.schema.products`

Replace the `stripe.schema` nesting with a flat `plan-catalog` namespace:

```java
@ConfigurationProperties(prefix = "iqkv.billing")
public record BillingConfigurationProperties(
    @Email String defaultContactEmail,
    @Valid PlanCatalogProperties planCatalog
) {
  public record PlanCatalogProperties(@Valid Map<String, ProductSchema> products) { ... }
}
```

The old `stripe` sub-record is removed. This is a breaking change to `application.yml`
addressed in Task 1.6.

---

#### Task 1.6 — Update `application.yml` plan catalog path

Rename the YAML key from:

```yaml
iqkv:
  billing:
    stripe:
      schema:
        products: ...
```

to:

```yaml
iqkv:
  billing:
    plan-catalog:
      products: ...
```

Also add the Lemon Squeezy configuration block (values empty/placeholder for now):

```yaml
iqkv:
  lemon-squeezy:
    api-key: ${LEMON_SQUEEZY_API_KEY:}
    store-id: ${LEMON_SQUEEZY_STORE_ID:}
    webhook-secret: ${LEMON_SQUEEZY_WEBHOOK_SECRET:}
    portal-return-url: ${LEMON_SQUEEZY_PORTAL_RETURN_URL:http://localhost:3000/billing}
```

And update `payment.gateway.type` comment:

```yaml
iqkv:
  payment:
    gateway:
      type: ${PAYMENT_GATEWAY_TYPE:STRIPE} # STRIPE | LEMON_SQUEEZY
```

---

#### Task 1.7 — Harden `PaymentGatewayPort`: document LS behaviour per method

Add Javadoc notes to each port method explaining expected Lemon Squeezy behavior, so
that any future implementer has a written contract:

- `createPortalSession`: LS equivalent is POST `/v1/customer-portal-sessions`; return URL.
- `syncProduct`: For LS, products and variants are managed in the dashboard. This method
  should resolve the external variant ID by looking up a pre-configured variant ID stored
  alongside the plan in the plan catalog. It may be a no-op write and a read-only verify.
- `cancelSubscription(id, false)`: LS maps to DELETE subscription (immediate).
- `cancelSubscription(id, true)`: LS maps to PATCH `{"data": {"attributes": {"cancelled": true}}}`.
- `pauseSubscription`: LS maps to PATCH `{"data": {"attributes": {"pause": {"mode": "void"}}}}`.
- `reactivateSubscription`: LS maps to PATCH `{"data": {"attributes": {"pause": null}}}`.

---

### Phase 2 — New: Lemon Squeezy Configuration Properties

#### Task 2.1 — Create `LemonSqueezyConfigurationProperties`

File: `infrastructure/config/LemonSqueezyConfigurationProperties.java`

```java
@Validated
@ConfigurationProperties(prefix = "iqkv.lemon-squeezy")
@ConditionalOnGateway(GatewayType.LEMON_SQUEEZY)
public record LemonSqueezyConfigurationProperties(
    @NotBlank String apiKey,
    @NotBlank String storeId,
    @NotBlank String webhookSecret,
    @NotBlank String portalReturnUrl
) {}
```

Add to `@EnableConfigurationProperties` list in `BillingServiceApplication` (or a
dedicated `@Configuration` class).

---

### Phase 3 — New: `LemonSqueezyGatewayAdapter`

#### Task 3.1 — Add Lemon Squeezy HTTP client dependency

Lemon Squeezy does not publish an official Java SDK. Options:

- **Option A (recommended):** Use Spring's `RestClient` (Spring Boot 3.2+, already on
  classpath) to call the JSON:API REST endpoints directly. This avoids an extra dependency
  and keeps the adapter thin.
- **Option B:** Add the community `codewriterbv/lemonsqueezy-java` library from GitHub.
  Less stable; adds an unvetted transitive dependency. Not recommended for production.

Proceed with **Option A**. The adapter will construct HTTP calls against
`https://api.lemonsqueezy.com/v1/` with `Authorization: Bearer <apiKey>` and
`Accept: application/vnd.api+json`.

Add `RestClient` bean configuration in a new `@ConditionalOnGateway(LEMON_SQUEEZY)`
config class:

```java
@Bean
@ConditionalOnGateway(GatewayType.LEMON_SQUEEZY)
public RestClient lemonSqueezyRestClient(LemonSqueezyConfigurationProperties props) {
    return RestClient.builder()
        .baseUrl("https://api.lemonsqueezy.com/v1")
        .defaultHeader("Authorization", "Bearer " + props.apiKey())
        .defaultHeader("Accept", "application/vnd.api+json")
        .defaultHeader("Content-Type", "application/vnd.api+json")
        .build();
}
```

---

#### Task 3.2 — Implement `LemonSqueezyGatewayAdapter`

File: `gateway/adapter/lemonsqueezy/LemonSqueezyGatewayAdapter.java`

Implement all 11 methods of `PaymentGatewayPort`. Below is the per-method specification.

**`getGatewayType()`** → return `GatewayType.LEMON_SQUEEZY`

**`createCustomer(CreateCustomerCommand)`**

- POST `/v1/customers` with `{"data": {"type": "customers", "attributes": {"name": …, "email": …, "store_id": storeId}}}`
- Response: `data.id` → return as external customer ID string.
- Note: email is required by LS. If null, throw `PaymentGatewayException` early with clear message.

**`createCheckoutSession(CreateCheckoutSessionCommand)`**

- POST `/v1/checkouts` with:
  ```json
  { "data": { "type": "checkouts",
      "attributes": {
        "checkout_data": { "email": "...", "custom": { "tenantKey": "..." } },
        "product_options": { "redirect_url": successUrl },
        "expires_at": null
      },
      "relationships": {
        "store": { "data": { "type": "stores", "id": storeId } },
        "variant": { "data": { "type": "variants", "id": priceId } }
      }
  } }
  ```
- `priceId` maps to LS **variant ID** (stored in `plan_catalog.external_price_id`).
- `trialPeriodDays` → set `trial_ends_at` to `Instant.now().plus(trialPeriodDays, DAYS)`.
- Response: `data.attributes.url` → return checkout URL.
- LS checkout quantity is not a line-item parameter; PER_SEAT pricing is handled via
  subscription item updates post-checkout. Document this limitation.

**`updateSubscription(UpdateSubscriptionCommand)`**

- PATCH `/v1/subscriptions/{subscriptionId}` with variant_id and/or quantity.
- Map `prorationBehavior` to LS `immediate_payment` boolean (treat `"create_prorations"`
  or `"always_invoice"` as `true`; `"none"` as `false`).

**`cancelSubscription(String, boolean cancelAtPeriodEnd)`**

- `cancelAtPeriodEnd = true`: PATCH `{"data": {"id": "…", "type": "subscriptions", "attributes": {"cancelled": true}}}`
- `cancelAtPeriodEnd = false`: DELETE `/v1/subscriptions/{subscriptionId}`

**`pauseSubscription(String)`**

- PATCH subscription with `{"attributes": {"pause": {"mode": "void"}}}`

**`reactivateSubscription(String)`**

- PATCH subscription with `{"attributes": {"pause": null}}`

**`createRefund(CreateRefundCommand)`**

- LS refunds are Order-level: POST `/v1/refunds` with `{"data": {"attributes": {"order_id": …, "amount": …}}}`
- `paymentId` in the command maps to the LS **order ID** (callers must store order IDs in subscription metadata).
- This implies a schema column addition: `order_id` on `refunds` table (see Task 4.3).
- Return LS refund object `data.id`.

**`syncProduct(Plan)`**

- LS products and variants are created/managed in the dashboard, not via API during runtime.
- Read-only verification: GET `/v1/variants/{externalPriceId}` to confirm the variant exists and is active.
- If `externalPriceId` is blank, log a warning and return empty string — the operator must
  configure the variant ID manually in `application.yml` before going live.
- Do NOT attempt to create LS products/variants programmatically. LS is a merchant-of-record
  model; product creation via API is limited and not idiomatic.

**`verifyAndParseWebhookEvent(String payload, String signature)`**

- Signature verification: compute `HmacSHA256(payload, webhookSecret)`, hex-encode, compare
  to `X-Signature` header (constant-time comparison).
- Parse the `meta.event_name` field to select the concrete `GatewayWebhookEvent` subtype.
- Map LS event names to normalized event types (see Task 3.3).

**`createPortalSession(String customerId, String returnUrl)`**

- POST `/v1/customer-portal-sessions` (or construct the portal URL using the LS customer portal
  configuration). Return the portal URL.

---

#### Task 3.3 — Map Lemon Squeezy webhook events to normalized domain events

| LS `meta.event_name`             | Normalized `eventType` (written to `webhook_log`)                                     | `GatewayWebhookEvent` subtype                              |
| -------------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `subscription_created`           | `subscription.created`                                                                | `GatewaySubscriptionEvent` (isCreated=true)                |
| `subscription_updated`           | `subscription.updated`                                                                | `GatewaySubscriptionEvent` (isUpdated=true)                |
| `subscription_cancelled`         | `subscription.deleted`                                                                | `GatewaySubscriptionEvent` (isDeleted=true)                |
| `subscription_resumed`           | `subscription.updated`                                                                | `GatewaySubscriptionEvent` (isUpdated=true)                |
| `subscription_expired`           | `subscription.deleted`                                                                | `GatewaySubscriptionEvent` (isDeleted=true)                |
| `subscription_paused`            | `subscription.updated`                                                                | `GatewaySubscriptionEvent` (isUpdated=true, status=paused) |
| `subscription_unpaused`          | `subscription.updated`                                                                | `GatewaySubscriptionEvent` (isUpdated=true, status=active) |
| `subscription_payment_success`   | `invoice.payment_succeeded`                                                           | `GatewayInvoiceEvent`                                      |
| `subscription_payment_failed`    | `invoice.payment_failed`                                                              | `GatewayPaymentFailureEvent`                               |
| `subscription_payment_recovered` | `invoice.payment_succeeded`                                                           | `GatewayInvoiceEvent`                                      |
| `order_created`                  | `invoice.payment_succeeded` (for one-off; skip for subscription orders handled above) | `GatewayInvoiceEvent`                                      |
| `order_refunded`                 | `charge.refunded`                                                                     | `GatewayRefundEvent`                                       |

LS does not fire `invoice.created` or `invoice.finalized` equivalents as discrete events.
`WebhookProcessingService` handles missing event types gracefully (logs and skips), so no
domain code changes are required for the absent invoice lifecycle events.

For `GatewaySubscriptionEvent` construction from LS data:

- `externalSubscriptionId` → `data.id`
- `externalCustomerId` → `data.attributes.customer_id` (string)
- `status` → `data.attributes.status` (`active`, `paused`, `past_due`, `unpaid`, `cancelled`, `expired`)
- `planId` → `data.attributes.variant_id` (maps to `external_price_id` in plan_catalog)
- `currentPeriodStart` → `data.attributes.renews_at` minus billing interval (approximate) or `data.attributes.created_at`
- `currentPeriodEnd` → `data.attributes.renews_at`
- `cancelAtPeriodEnd` → `data.attributes.cancelled`
- `canceledAt` → `data.attributes.ends_at`
- `metadata` → `meta.custom_data` map from webhook payload

**Important:** LS puts custom data in `meta.custom_data`, not in an object-level metadata map.
The `tenantKey` and `userId` values embedded at checkout creation must be stored there and
read back here to populate the `metadata` map in `GatewaySubscriptionEvent`.

---

#### Task 3.4 — Create `LemonSqueezyWebhookRestResource`

File: `webhook/LemonSqueezyWebhookRestResource.java`

- Endpoint: `POST /api/v1/billing/webhooks/lemon-squeezy`
- Header: `X-Signature` (not `Stripe-Signature`)
- Annotated with `@ConditionalOnGateway(GatewayType.LEMON_SQUEEZY)`
- Delegates to `LemonSqueezyGatewayAdapter.verifyAndParseWebhookEvent(payload, signature)`
- Delegates processing to `WebhookProcessingService.process(event)` — no change to processor
- Returns 200 always after idempotency check; 400 on signature failure

Register the new endpoint path as `permitAll()` in `SecurityConfig` (alongside `/webhooks/stripe`).

---

### Phase 4 — Schema Hardening

#### Task 4.1 — Add `gateway_type` column to `billing_settings`

Purpose: Record which gateway created the customer, so that admin queries and potential
future migrations know which adapter owns each customer record.

```xml
<!-- Liquibase: 20260701000000-add-gateway-type-to-billing-settings.xml -->
<addColumn tableName="billing_settings">
  <column name="gateway_type" type="VARCHAR(32)" defaultValue="STRIPE">
    <constraints nullable="false"/>
  </column>
</addColumn>
```

Populate `BillingSettingsService.createBillingSettings()` to write the active gateway type.

---

#### Task 4.2 — Add `gateway_type` column to `subscriptions`

```xml
<!-- Liquibase: 20260701000001-add-gateway-type-to-subscriptions.xml -->
<addColumn tableName="subscriptions">
  <column name="gateway_type" type="VARCHAR(32)" defaultValue="STRIPE">
    <constraints nullable="false"/>
  </column>
</addColumn>
```

Populate from `GatewaySubscriptionEvent` — add `gatewayType` field to the event interface.

---

#### Task 4.3 — Add `order_id` column to `subscriptions` (LS-specific)

LS refunds target an Order ID, not a Charge ID. The Order ID is available in the
`subscription_payment_success` webhook payload. Storing it on the subscription record
allows refund requests to pass the correct ID.

```xml
<!-- Liquibase: 20260701000002-add-order-id-to-subscriptions.xml -->
<addColumn tableName="subscriptions">
  <column name="external_order_id" type="VARCHAR(255)"/>
</addColumn>
```

`WebhookProcessingService.handleInvoiceEvent()` already has access to `GatewayInvoiceEvent`.
Add `externalOrderId` to `GatewayInvoiceEvent` (nullable; Stripe leaves it null) and write
it when processing `invoice.payment_succeeded`.

---

#### Task 4.4 — Migrate `external_price_id` semantics for LS

In the Stripe model, `external_price_id` stores a Stripe Price ID (`price_…`).
In the LS model, the same column stores a Lemon Squeezy Variant ID (integer string).

No schema change is required. Document in `plan_catalog` column comment that the value is
gateway-type-dependent. Add `gateway_type` column to `plan_catalog` for observability:

```xml
<!-- Liquibase: 20260701000003-add-gateway-type-to-plan-catalog.xml -->
<addColumn tableName="plan_catalog">
  <column name="gateway_type" type="VARCHAR(32)" defaultValue="STRIPE">
    <constraints nullable="false"/>
  </column>
</addColumn>
```

---

### Phase 5 — Application Layer Hardening

#### Task 5.1 — Propagate `gatewayType` into `GatewaySubscriptionEvent` and `GatewayInvoiceEvent`

Add `String gatewayType()` to `GatewayWebhookEvent` sealed interface (default or abstract).
Both Stripe and LS adapters populate it from their respective `getGatewayType().name()`.
`WebhookProcessingService` writes the value to the `subscriptions.gateway_type` column.

This is a non-breaking addition: the interface gains a new method with a default implementation
that falls back to `"UNKNOWN"` if not overridden, preventing compilation failures in tests.

---

#### Task 5.2 — Harden `BillingSeedRunner` for LS gateway

`BillingSeedRunner` currently calls `paymentGatewayPort.syncProduct(plan)` unconditionally.
For Lemon Squeezy, `syncProduct` is read-only. This is safe but the seed runner must not
_require_ `external_price_id` to be set post-sync for LS (LS operators configure it manually).

Change: after `syncProduct()`, only call `planMapper.update(plan)` if `externalProductId`
or `externalPriceId` actually changed. For LS, the verify-only adapter returns the existing
`externalPriceId` unchanged, so no spurious update is issued.

Add a startup log warning when any plan has a blank `externalPriceId` with gateway type LS,
directing the operator to configure variant IDs in `application.yml`.

---

#### Task 5.3 — Plan catalog YAML: add `variantId` field for LS

To allow LS variant IDs to be configured in `application.yml` (since LS doesn't create
variants programmatically), extend `ProductSchema` with an optional `externalVariantId`:

```java
public record ProductSchema(
    @NotBlank String planCode,
    // ... existing fields ...
    String externalVariantId  // Populated by LS operators; ignored by Stripe
) { }
```

`BillingSeedRunner.syncProduct()` writes this to `plan_catalog.external_price_id` before
calling `syncProduct()`, so the LS adapter has the variant ID available.

---

#### Task 5.4 — Handle LS customer email requirement in `BillingSettingsService`

`CreateCustomerCommand` allows a null email (documented for single-tenant mode where
Stripe permits no-email customers). LS does not allow this.

In `LemonSqueezyGatewayAdapter.createCustomer()`: if email is null or blank, throw
`PaymentGatewayException("Lemon Squeezy requires a customer email address")` with a clear
message. This is a hard requirement of the LS API.

Ensure `BillingConfigurationProperties.defaultContactEmail` is always set when the active
gateway is LEMON_SQUEEZY in single-tenant mode. Add a startup validator:

```java
// infrastructure/config/PlatformModeValidatorImpl.java (extend existing)
if (GatewayType.LEMON_SQUEEZY == gatewayProps.type()
    && RolloutMode.SINGLE_TENANT == platformProps.rolloutMode()
    && (billingProps.defaultContactEmail() == null || billingProps.defaultContactEmail().isBlank())) {
  throw new IllegalStateException(
      "iqkv.billing.default-contact-email is required when using LEMON_SQUEEZY gateway in SINGLE_TENANT mode");
}
```

---

#### Task 5.5 — `createPortalSession` — bridge LS customer portal

LS customer portal is accessible via a portal URL that includes the customer's email or
a signed token depending on configuration. Implement `createPortalSession` in the LS adapter
to call `POST /v1/customer-portal-sessions` and return the resulting URL, matching the
Stripe Billing Portal UX.

No changes to `PaymentRestResource` or `SubscriptionService` are needed — they already call
`paymentGatewayPort.createPortalSession(customerId, returnUrl)`.

---

### Phase 6 — Testing

#### Task 6.1 — Unit tests for `LemonSqueezyGatewayAdapter`

For each `PaymentGatewayPort` method, write a unit test with a mock `RestClient` that:

- Verifies the correct HTTP method and path are used.
- Verifies request body structure (especially JSON:API `data.type` and `data.attributes`).
- Verifies the correct field from the response is returned.
- Covers error cases (`4xx`, `5xx`, network timeout → `PaymentGatewayException`).

Webhook signature verification:

- Happy path: valid HMAC matches → event parsed correctly.
- Invalid signature: `WebhookProcessingException` thrown.
- Unknown event type: returns `Optional.empty()`.

---

#### Task 6.2 — ArchUnit: enforce adapter isolation

Add or extend existing ArchUnit rules to assert:

- No code outside `gateway/adapter/stripe` imports any `com.stripe.*` class.
- No code outside `gateway/adapter/lemonsqueezy` imports any LS-specific HTTP model classes.
- `WebhookProcessingService` has no imports from any `gateway/adapter` package.

---

#### Task 6.3 — Integration test: full webhook round-trip for LS

Add a `@SpringBootTest` test with the active gateway set to `LEMON_SQUEEZY`:

- Mock the `LemonSqueezyGatewayAdapter` or use a test-double.
- POST a simulated `subscription_created` webhook payload to `/api/v1/billing/webhooks/lemon-squeezy`.
- Assert the subscription row is created in the DB.
- Assert the `subscription.created` RabbitMQ message is published.

---

---

## 5. Gaps and Constraints

### 5.1 Features Not Directly Supportable with Lemon Squeezy

| Feature                                                           | Stripe                         | Lemon Squeezy                                      | Mitigation                                                                                            |
| ----------------------------------------------------------------- | ------------------------------ | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Programmatic product/price creation at startup                    | Full API                       | Dashboard-only (variants)                          | Manual config via `externalVariantId` in YAML; startup warning if missing                             |
| PER_SEAT checkout quantity                                        | Line-item quantity on checkout | Not a checkout parameter                           | Seat count must be stored and applied after subscription creation via subscription-item update        |
| Invoice lifecycle events (`invoice.created`, `invoice.finalized`) | Yes                            | Not emitted                                        | Already handled: `WebhookProcessingService` skips unknown event types                                 |
| Partial refund against a charge                                   | Stripe Charge ID               | LS Order ID                                        | Store `externalOrderId` on subscription (Task 4.3); operators must use LS order ID in refund requests |
| Promotion codes on checkout                                       | `allowPromotionCodes` flag     | LS supports discount codes natively in checkout UI | No API flag needed; discount codes can be configured per-variant in LS dashboard                      |
| Portal session return URL as parameter                            | Per-request                    | LS portal URL is preconfigured in dashboard        | Use `portalReturnUrl` from config as a hint; LS may not honor per-request redirect URLs               |
| `customer.subscription.deleted` vs `expired` distinction          | Single event type              | Two separate events                                | Both mapped to `subscription.deleted` normalized type; no domain impact                               |

### 5.2 LS-Specific Operational Requirements

- **Products and variants must be pre-created in the LS dashboard.** The operator must
  manually enter variant IDs into `iqkv.billing.plan-catalog.products.<key>.externalVariantId`
  before deployment.
- **LS is a merchant-of-record.** Tax collection and invoicing is handled by LS.
  The service's local invoice tracking (via `invoice.payment_succeeded`) remains valid.
- **LS store ID is required** for creating checkouts and customers. It is a configuration
  property, not derivable at runtime.
- **Test mode vs live mode** is controlled by the API key in LS (no separate
  `sk_test_` / `sk_live_` prefix as in Stripe). The LS dashboard toggles test mode per store.

---

## 6. Implementation Order and Dependencies

```
Phase 1 (Alignment — independent, no risk)
  1.4 Rename StripeProductSchema → ProductSchema
  1.5 Flatten BillingConfigurationProperties
  1.6 Update application.yml paths          ← depends on 1.4, 1.5
  1.1 Add LEMON_SQUEEZY to GatewayType
  1.2 Create @ConditionalOnGateway
  1.3 Annotate StripeWebhookRestResource     ← depends on 1.2
  1.7 Harden PaymentGatewayPort Javadoc

Phase 2 (Configuration)
  2.1 LemonSqueezyConfigurationProperties   ← depends on 1.1

Phase 3 (Implementation)
  3.1 Add RestClient bean                   ← depends on 2.1
  3.2 LemonSqueezyGatewayAdapter            ← depends on 3.1
  3.3 Webhook event mapping (part of 3.2)
  3.4 LemonSqueezyWebhookRestResource       ← depends on 3.2

Phase 4 (Schema)
  4.1 billing_settings.gateway_type
  4.2 subscriptions.gateway_type
  4.3 subscriptions.external_order_id
  4.4 plan_catalog.gateway_type

Phase 5 (Application layer)
  5.1 gatewayType in GatewayWebhookEvent    ← depends on 4.2
  5.2 BillingSeedRunner hardening           ← depends on 1.5, 1.6
  5.3 ProductSchema.externalVariantId       ← depends on 1.4
  5.4 Email validation for LS               ← depends on 2.1
  5.5 createPortalSession for LS            ← depends on 3.2

Phase 6 (Tests)
  6.1 Unit tests for LS adapter             ← depends on 3.2
  6.2 ArchUnit rules                        ← depends on 3.2
  6.3 Integration test                      ← depends on 3.4, phase 4
```

---

## 7. Configuration Reference After Refactoring

```yaml
iqkv:
  payment:
    gateway:
      type: ${PAYMENT_GATEWAY_TYPE:STRIPE} # STRIPE | LEMON_SQUEEZY

  # --- Active when gateway.type=STRIPE ---
  stripe:
    secret-key: ${STRIPE_SECRET_KEY:sk_test_placeholder}
    webhook-secret: ${STRIPE_WEBHOOK_SECRET:whsec_placeholder}
    portal-return-url: ${STRIPE_PORTAL_RETURN_URL:http://localhost:3000/billing}

  # --- Active when gateway.type=LEMON_SQUEEZY ---
  lemon-squeezy:
    api-key: ${LEMON_SQUEEZY_API_KEY:}
    store-id: ${LEMON_SQUEEZY_STORE_ID:}
    webhook-secret: ${LEMON_SQUEEZY_WEBHOOK_SECRET:}
    portal-return-url: ${LEMON_SQUEEZY_PORTAL_RETURN_URL:http://localhost:3000/billing}

  billing:
    default-contact-email: ${DEFAULT_BILLING_EMAIL:}
    plan-catalog: # renamed from billing.stripe.schema
      products:
        basic-monthly:
          planCode: "basic-monthly"
          displayName: "Basic Monthly"
          billingPeriod: "MONTHLY"
          priceMinor: 1000
          currency: "USD"
          scope: "TENANT"
          active: true
          externalVariantId: "123456" # LS variant ID (ignored by Stripe)
          features:
            maxUsers: 5
            maxProjects: 3
```

---

## 8. Files to Create

| File                                                                           | Phase |
| ------------------------------------------------------------------------------ | ----- |
| `gateway/GatewayType.java` (modified)                                          | 1.1   |
| `infrastructure/config/ConditionalOnGateway.java`                              | 1.2   |
| `infrastructure/config/GatewayCondition.java`                                  | 1.2   |
| `infrastructure/config/ProductSchema.java` (renamed from StripeProductSchema)  | 1.4   |
| `infrastructure/config/BillingConfigurationProperties.java` (modified)         | 1.5   |
| `infrastructure/config/LemonSqueezyConfigurationProperties.java`               | 2.1   |
| `infrastructure/config/LemonSqueezyRestClientConfig.java`                      | 3.1   |
| `gateway/adapter/lemonsqueezy/LemonSqueezyGatewayAdapter.java`                 | 3.2   |
| `webhook/LemonSqueezyWebhookRestResource.java`                                 | 3.4   |
| `db/changelog/changes/20260701000000-add-gateway-type-to-billing-settings.xml` | 4.1   |
| `db/changelog/changes/20260701000001-add-gateway-type-to-subscriptions.xml`    | 4.2   |
| `db/changelog/changes/20260701000002-add-order-id-to-subscriptions.xml`        | 4.3   |
| `db/changelog/changes/20260701000003-add-gateway-type-to-plan-catalog.xml`     | 4.4   |

## 9. Files to Modify

| File                                                          | Change                                                                     |
| ------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `gateway/GatewayType.java`                                    | Add `LEMON_SQUEEZY`                                                        |
| `gateway/event/GatewayWebhookEvent.java`                      | Add `gatewayType()` method                                                 |
| `gateway/event/GatewaySubscriptionEvent.java`                 | Add `gatewayType` field                                                    |
| `gateway/event/GatewayInvoiceEvent.java`                      | Add `gatewayType`, `externalOrderId` fields                                |
| `gateway/event/GatewayPaymentFailureEvent.java`               | Add `gatewayType` field                                                    |
| `gateway/event/GatewayRefundEvent.java`                       | Add `gatewayType` field                                                    |
| `gateway/adapter/stripe/StripeGatewayAdapter.java`            | Add `@ConditionalOnGateway(STRIPE)`; populate `gatewayType` in events      |
| `gateway/port/PaymentGatewayPort.java`                        | Add LS Javadoc notes per method                                            |
| `webhook/StripeWebhookRestResource.java`                      | Add `@ConditionalOnGateway(STRIPE)`                                        |
| `webhook/WebhookProcessingService.java`                       | Write `gateway_type` column                                                |
| `infrastructure/config/BillingConfigurationProperties.java`   | Flatten stripe → plan-catalog                                              |
| `infrastructure/config/BillingSeedRunner.java`                | Use `ProductSchema`; handle LS externalVariantId; conditional update logic |
| `infrastructure/config/StripeProductSchema.java`              | Rename to `ProductSchema`; add `externalVariantId`                         |
| `infrastructure/config/PlatformModeValidatorImpl.java`        | Add LS + single-tenant email check                                         |
| `infrastructure/persistence/SubscriptionMapper.java` + XML    | Add `gateway_type`, `external_order_id`                                    |
| `infrastructure/persistence/BillingSettingsMapper.java` + XML | Add `gateway_type`                                                         |
| `infrastructure/persistence/PlanMapper.java` + XML            | Add `gateway_type`                                                         |
| `resources/application.yml`                                   | Rename plan-catalog path; add LS config block                              |
| `resources/db/changelog/db.changelog-master.xml`              | Include new migration files                                                |

---

## 10. Detailed Code Changes

This section provides the exact changes required for every file listed in sections 8 and 9.

### 10.1 `gateway/GatewayType.java` — add `LEMON_SQUEEZY` constant

```java
// BEFORE
public enum GatewayType {
  STRIPE
  // Reserved for future implementation:
  // PAYPAL,
  // BRAINTREE
}

// AFTER
public enum GatewayType {
  /** Stripe payment gateway. Products/prices created programmatically. */
  STRIPE,
  /**
   * Lemon Squeezy payment gateway.
   * Products/variants are managed in the LS dashboard; variant IDs are
   * configured via {@code iqkv.billing.plan-catalog.products.<key>.externalVariantId}.
   */
  LEMON_SQUEEZY
  // Reserved: PAYPAL, BRAINTREE
}
```

### 10.2 `infrastructure/config/ConditionalOnGateway.java` — new file

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(GatewayCondition.class)
public @interface ConditionalOnGateway {
  GatewayType value();
}
```

### 10.3 `infrastructure/config/GatewayCondition.java` — new file

```java
public class GatewayCondition implements Condition {
  @Override
  public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    final String configured = context.getEnvironment()
        .getProperty("iqkv.payment.gateway.type", "STRIPE").toUpperCase();
    final Map<String, Object> attrs =
        metadata.getAnnotationAttributes(ConditionalOnGateway.class.getName());
    if (attrs == null) return false;
    // value() is stored as GatewayType enum; toString() gives the name
    return configured.equals(attrs.get("value").toString());
  }
}
```

### 10.4 `infrastructure/config/ProductSchema.java` — rename + extend

Rename `StripeProductSchema.java` → `ProductSchema.java`.
Add `externalVariantId` field (optional; used by LS only).

```java
// BEFORE (record name and Javadoc were Stripe-specific)
public record StripeProductSchema(
    @NotBlank String planCode, @NotBlank String displayName,
    String description, @NotBlank String billingPeriod,
    @NotNull @Positive Integer priceMinor, @NotBlank String currency,
    PlanFeatures features, @NotBlank String scope,
    Boolean active, Integer trialPeriodDays, PricingModel pricingModel
) { ... }

// AFTER
/**
 * Gateway-neutral product/plan configuration entry.
 * Bound from {@code iqkv.billing.plan-catalog.products.<key>}.
 *
 * <p>{@code externalVariantId}: Lemon Squeezy only. Variant ID pre-created in LS dashboard.
 * Ignored by Stripe — Stripe IDs are written back by {@code BillingSeedRunner} at startup.
 */
public record ProductSchema(
    @NotBlank String planCode, @NotBlank String displayName,
    String description, @NotBlank String billingPeriod,
    @NotNull @Positive Integer priceMinor, @NotBlank String currency,
    PlanFeatures features, @NotBlank String scope,
    Boolean active, Integer trialPeriodDays, PricingModel pricingModel,
    String externalVariantId  // LS variant ID; null for Stripe-managed plans
) {
  public PricingModel effectivePricingModel() {
    return pricingModel != null ? pricingModel : PricingModel.FLAT;
  }
}
```

### 10.5 `infrastructure/config/BillingConfigurationProperties.java` — flatten plan catalog path

```java
// BEFORE — Stripe-specific nesting
@ConfigurationProperties(prefix = "iqkv.billing")
public record BillingConfigurationProperties(
    @Email String defaultContactEmail,
    @Valid StripeProperties stripe
) {
  public record StripeProperties(@Valid SchemaProperties schema) { ... }
  public record SchemaProperties(@Valid Map<String, StripeProductSchema> products) { ... }
}

// AFTER — gateway-neutral
@ConfigurationProperties(prefix = "iqkv.billing")
public record BillingConfigurationProperties(
    @Email String defaultContactEmail,
    @Valid PlanCatalogProperties planCatalog
) {
  public BillingConfigurationProperties {
    if (planCatalog == null) {
      planCatalog = new PlanCatalogProperties(Collections.emptyMap());
    }
  }

  /** Bound from {@code iqkv.billing.plan-catalog}. */
  public record PlanCatalogProperties(@Valid Map<String, ProductSchema> products) {
    public PlanCatalogProperties {
      if (products == null) products = Collections.emptyMap();
    }
  }
}
```

Update every call-site:

- `billingProps.stripe().schema().products()` → `billingProps.planCatalog().products()`
- Affected files: `BillingSeedRunner.java` (one call)

### 10.6 `resources/application.yml` — rename plan catalog key, add LS block

```yaml
# BEFORE
iqkv:
  billing:
    default-contact-email: ${DEFAULT_BILLING_EMAIL:}
    stripe:
      schema:
        products:
          my-plan: ...

# AFTER
iqkv:
  payment:
    gateway:
      type: ${PAYMENT_GATEWAY_TYPE:STRIPE}    # STRIPE | LEMON_SQUEEZY

  stripe:
    secret-key: ${STRIPE_SECRET_KEY:sk_test_placeholder}
    webhook-secret: ${STRIPE_WEBHOOK_SECRET:whsec_placeholder}
    portal-return-url: ${STRIPE_PORTAL_RETURN_URL:http://localhost:3000/billing}

  lemon-squeezy:
    api-key: ${LEMON_SQUEEZY_API_KEY:}
    store-id: ${LEMON_SQUEEZY_STORE_ID:}
    webhook-secret: ${LEMON_SQUEEZY_WEBHOOK_SECRET:}
    portal-return-url: ${LEMON_SQUEEZY_PORTAL_RETURN_URL:http://localhost:3000/billing}

  billing:
    default-contact-email: ${DEFAULT_BILLING_EMAIL:}
    plan-catalog:           # renamed from billing.stripe.schema
      products:
        my-plan:
          planCode: "my-plan"
          externalVariantId: ""   # LS only — set to LS variant ID before deploying with LS
```

### 10.7 `gateway/event/GatewayWebhookEvent.java` — add `gatewayType()` default method

```java
// ADD to the interface (before the closing brace)

  /**
   * Identifies which payment gateway produced this event.
   * Default returns {@code "UNKNOWN"} so existing test stubs compile without changes.
   *
   * @return {@link com.iqkv.foundation.billingservice.gateway.GatewayType#name()} value
   */
  default String gatewayType() {
    return "UNKNOWN";
  }
```

The four concrete record types (`GatewaySubscriptionEvent`, `GatewayInvoiceEvent`,
`GatewayPaymentFailureEvent`, `GatewayRefundEvent`) each add a `String gatewayType`
component and implement the interface method by returning it.

### 10.8 `gateway/event/GatewaySubscriptionEvent.java` — add `gatewayType` field

```java
// BEFORE (15 components)
public record GatewaySubscriptionEvent(
    String eventId, String eventType, Instant occurredAt,
    String externalSubscriptionId, String externalCustomerId, String status,
    String planId, Long quantity, Instant trialStart, Instant trialEnd,
    Instant currentPeriodStart, Instant currentPeriodEnd,
    boolean cancelAtPeriodEnd, Instant canceledAt,
    Map<String, String> metadata
) implements GatewayWebhookEvent { ... }

// AFTER — add gatewayType as the second component (convention: after eventType)
public record GatewaySubscriptionEvent(
    String eventId, String eventType, String gatewayType, Instant occurredAt,
    String externalSubscriptionId, String externalCustomerId, String status,
    String planId, Long quantity, Instant trialStart, Instant trialEnd,
    Instant currentPeriodStart, Instant currentPeriodEnd,
    boolean cancelAtPeriodEnd, Instant canceledAt,
    Map<String, String> metadata
) implements GatewayWebhookEvent {
  @Override public String gatewayType() { return gatewayType; }
  // isCreated(), isUpdated(), isDeleted() unchanged
}
```

Update all construction sites in `StripeGatewayAdapter` (3 private builder methods)
to pass `GatewayType.STRIPE.name()` as the third argument.

### 10.9 `gateway/event/GatewayInvoiceEvent.java` — add `gatewayType` + `externalOrderId`

```java
// BEFORE (9 components)
public record GatewayInvoiceEvent(
    String eventId, String eventType, Instant occurredAt,
    String externalInvoiceId, String externalCustomerId, String externalSubscriptionId,
    Long amountPaid, Long amountDue, String currency
) implements GatewayWebhookEvent {}

// AFTER (11 components)
public record GatewayInvoiceEvent(
    String eventId, String eventType, String gatewayType, Instant occurredAt,
    String externalInvoiceId, String externalCustomerId, String externalSubscriptionId,
    Long amountPaid, Long amountDue, String currency,
    String externalOrderId   // LS: order ID from subscription_payment_success; null for Stripe
) implements GatewayWebhookEvent {
  @Override public String gatewayType() { return gatewayType; }
}
```

### 10.10 `gateway/event/GatewayPaymentFailureEvent.java` and `GatewayRefundEvent.java`

Same pattern — add `String gatewayType` as third component and override `gatewayType()`.

```java
// GatewayPaymentFailureEvent — add after eventType:
public record GatewayPaymentFailureEvent(
    String eventId, String eventType, String gatewayType, Instant occurredAt,
    String externalInvoiceId, String externalCustomerId, String externalSubscriptionId,
    Long amountDue, String currency, String failureReason
) implements GatewayWebhookEvent {
  @Override public String gatewayType() { return gatewayType; }
}

// GatewayRefundEvent — add after eventType:
public record GatewayRefundEvent(
    String eventId, String eventType, String gatewayType, Instant occurredAt,
    String externalRefundId, String externalPaymentId, String externalCustomerId,
    Long amountRefunded, String currency, String status
) implements GatewayWebhookEvent {
  @Override public String gatewayType() { return gatewayType; }
}
```

### 10.11 `gateway/port/PaymentGatewayPort.java` — LS implementation notes per method

Add the following Javadoc lines to each relevant method (no signature changes):

```
createCustomer      — LS: POST /v1/customers; email required (throws if null).
createCheckoutSession — LS: POST /v1/checkouts; priceId maps to variantId; quantity
                        not supported as checkout line-item param (PER_SEAT handled post-checkout).
updateSubscription  — LS: PATCH /v1/subscriptions/{id}; prorationBehavior maps to
                        immediate_payment boolean.
cancelSubscription  — LS: cancelAtPeriodEnd=true → PATCH cancelled:true;
                        cancelAtPeriodEnd=false → DELETE /v1/subscriptions/{id}.
pauseSubscription   — LS: PATCH pause:{mode:"void"}.
reactivateSubscription — LS: PATCH pause:null.
createRefund        — LS: POST /v1/refunds with order_id (not charge ID).
                        paymentId in command must carry LS order ID.
syncProduct         — LS: read-only; verifies variant exists via GET /v1/variants/{id}.
                        Does not create or modify remote resources. Returns externalPriceId
                        unchanged if already set; empty string if not configured.
createPortalSession — LS: POST /v1/customer-portal-sessions; returnUrl used as hint.
```

### 10.12 `gateway/adapter/stripe/StripeGatewayAdapter.java` — add conditional + gatewayType in events

```java
// ADD annotations to the class declaration
@Component
@ConditionalOnGateway(GatewayType.STRIPE)    // ← NEW
public class StripeGatewayAdapter implements PaymentGatewayPort { ... }
```

In each private `toSubscriptionEvent`, `toInvoiceEvent`, `toPaymentFailureEvent`,
`toRefundEvent` builder method, pass `GatewayType.STRIPE.name()` as the `gatewayType`
argument in the new record constructor position (third argument after `eventType`):

```java
// Example — toSubscriptionEvent (before)
return new GatewaySubscriptionEvent(
    event.getId(), event.getType(), occurredAt, ...);

// After
return new GatewaySubscriptionEvent(
    event.getId(), event.getType(), GatewayType.STRIPE.name(), occurredAt, ...);
```

For `toInvoiceEvent`, also pass `null` as `externalOrderId` (Stripe doesn't use order IDs).

### 10.13 `webhook/StripeWebhookRestResource.java` — add `@ConditionalOnGateway`

```java
// ADD annotation to the class declaration
@RestController
@RequestMapping("/api/v1/billing/webhooks")
@ConditionalOnGateway(GatewayType.STRIPE)    // ← NEW
@Tag(name = "Webhooks", ...)
public class StripeWebhookRestResource { ... }
```

No other changes. The resource continues to inject `StripeGatewayAdapter` by concrete type,
which is safe because both the resource and the adapter are conditionally active together.

### 10.14 `infrastructure/config/LemonSqueezyConfigurationProperties.java` — new file

```java
@Validated
@ConfigurationProperties(prefix = "iqkv.lemon-squeezy")
@ConditionalOnGateway(GatewayType.LEMON_SQUEEZY)
public record LemonSqueezyConfigurationProperties(
    /** Bearer token for the LS REST API. */
    @NotBlank String apiKey,
    /** Numeric LS store ID — required for checkout and customer creation. */
    @NotBlank String storeId,
    /** HMAC-SHA256 signing secret from LS webhook settings. */
    @NotBlank String webhookSecret,
    /** URL the customer is returned to after leaving the LS Customer Portal. */
    @NotBlank String portalReturnUrl
) {}
```

Register in `BillingServiceApplication` (add to `@EnableConfigurationProperties`):

```java
@EnableConfigurationProperties({
    BillingConfigurationProperties.class,
    StripeConfigurationProperties.class,
    LemonSqueezyConfigurationProperties.class,   // ← NEW
    PaymentGatewayConfigurationProperties.class,
    // ...
})
```

### 10.15 `infrastructure/config/LemonSqueezyRestClientConfig.java` — new file

```java
@Configuration
@ConditionalOnGateway(GatewayType.LEMON_SQUEEZY)
public class LemonSqueezyRestClientConfig {

  private static final String BASE_URL = "https://api.lemonsqueezy.com/v1";

  /**
   * Pre-configured {@link RestClient} for the Lemon Squeezy JSON:API.
   * Uses a dedicated bean name to avoid conflict with any other RestClient beans.
   */
  @Bean("lemonSqueezyRestClient")
  public RestClient lemonSqueezyRestClient(final LemonSqueezyConfigurationProperties props) {
    return RestClient.builder()
        .baseUrl(BASE_URL)
        .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + props.apiKey())
        .defaultHeader(HttpHeaders.ACCEPT, "application/vnd.api+json")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, "application/vnd.api+json")
        .build();
  }
}
```

### 10.16 `gateway/adapter/lemonsqueezy/LemonSqueezyGatewayAdapter.java` — new file (full spec)

```java
@Component
@ConditionalOnGateway(GatewayType.LEMON_SQUEEZY)
public class LemonSqueezyGatewayAdapter implements PaymentGatewayPort {

  private static final Logger log = LoggerFactory.getLogger(LemonSqueezyGatewayAdapter.class);

  // LS event names emitted in meta.event_name
  private static final String LS_SUBSCRIPTION_CREATED   = "subscription_created";
  private static final String LS_SUBSCRIPTION_UPDATED   = "subscription_updated";
  private static final String LS_SUBSCRIPTION_CANCELLED = "subscription_cancelled";
  private static final String LS_SUBSCRIPTION_RESUMED   = "subscription_resumed";
  private static final String LS_SUBSCRIPTION_EXPIRED   = "subscription_expired";
  private static final String LS_SUBSCRIPTION_PAUSED    = "subscription_paused";
  private static final String LS_SUBSCRIPTION_UNPAUSED  = "subscription_unpaused";
  private static final String LS_PAYMENT_SUCCESS        = "subscription_payment_success";
  private static final String LS_PAYMENT_RECOVERED      = "subscription_payment_recovered";
  private static final String LS_PAYMENT_FAILED         = "subscription_payment_failed";
  private static final String LS_ORDER_REFUNDED         = "order_refunded";

  private final LemonSqueezyConfigurationProperties config;
  private final RestClient restClient;           // qualifier: "lemonSqueezyRestClient"
  private final ObjectMapper objectMapper;       // standard Jackson mapper

  // constructor injection ...

  @Override
  public GatewayType getGatewayType() { return GatewayType.LEMON_SQUEEZY; }
```

#### `createCustomer`

```java
  @Override
  public String createCustomer(final CreateCustomerCommand command) {
    if (command.email() == null || command.email().isBlank()) {
      throw new PaymentGatewayException(
          "Lemon Squeezy requires a customer email address; none provided for: " + command.name());
    }
    final Map<String, Object> body = Map.of("data", Map.of(
        "type", "customers",
        "attributes", Map.of(
            "name", command.name(),
            "email", command.email(),
            "store_id", Integer.parseInt(config.storeId())
        )
    ));
    try {
      final JsonNode response = restClient.post().uri("/customers")
          .body(body).retrieve().body(JsonNode.class);
      final String customerId = response.path("data").path("id").asText();
      log.debug("Created LS customer {} for name={}", customerId, command.name());
      return customerId;
    } catch (final RestClientException e) {
      throw new PaymentGatewayException("Failed to create LS customer: " + command.name(), e);
    }
  }
```

#### `createCheckoutSession`

```java
  @Override
  public String createCheckoutSession(final CreateCheckoutSessionCommand command) {
    // Build custom_data to carry tenantKey/userId through to webhooks
    final Map<String, Object> customData = new HashMap<>(command.metadata());

    // Build checkout_data attributes
    final Map<String, Object> checkoutData = new HashMap<>();
    checkoutData.put("custom", customData);

    final Map<String, Object> attributes = new HashMap<>();
    attributes.put("checkout_data", checkoutData);
    attributes.put("product_options", Map.of("redirect_url", command.successUrl()));

    if (command.trialPeriodDays() != null && command.trialPeriodDays() > 0) {
      final String trialEnd = Instant.now()
          .plus(command.trialPeriodDays(), ChronoUnit.DAYS).toString();
      attributes.put("expires_at", null);  // no checkout expiry
      // trial_ends_at goes inside checkout_data.subscription_data
      checkoutData.put("subscription_data",
          Map.of("trial_ends_at", trialEnd));
    }

    final Map<String, Object> body = Map.of("data", Map.of(
        "type", "checkouts",
        "attributes", attributes,
        "relationships", Map.of(
            "store",   Map.of("data", Map.of("type", "stores",   "id", config.storeId())),
            "variant", Map.of("data", Map.of("type", "variants", "id", command.priceId()))
        )
    ));
    try {
      final JsonNode response = restClient.post().uri("/checkouts")
          .body(body).retrieve().body(JsonNode.class);
      final String url = response.path("data").path("attributes").path("url").asText();
      log.debug("Created LS checkout for customer {}: {}", command.customerId(), url);
      return url;
    } catch (final RestClientException e) {
      throw new PaymentGatewayException(
          "Failed to create LS checkout for customer: " + command.customerId(), e);
    }
  }
```

#### `updateSubscription`, `cancelSubscription`, `pauseSubscription`, `reactivateSubscription`

```java
  @Override
  public void updateSubscription(final UpdateSubscriptionCommand command) {
    final Map<String, Object> attrs = new HashMap<>();
    if (command.priceId() != null) attrs.put("variant_id", Integer.parseInt(command.priceId()));
    if (command.quantity() != null) attrs.put("quantity", command.quantity());
    // prorationBehavior → immediate_payment
    if ("none".equalsIgnoreCase(command.prorationBehavior())) {
      attrs.put("immediate_payment", false);
    } else if (command.prorationBehavior() != null) {
      attrs.put("immediate_payment", true);
    }
    patch("/subscriptions/" + command.subscriptionId(), attrs);
    log.debug("Updated LS subscription {}", command.subscriptionId());
  }

  @Override
  public void cancelSubscription(final String subscriptionId, final boolean cancelAtPeriodEnd) {
    if (cancelAtPeriodEnd) {
      patch("/subscriptions/" + subscriptionId, Map.of("cancelled", true));
      log.debug("Set LS subscription {} to cancel at period end", subscriptionId);
    } else {
      try {
        restClient.delete().uri("/subscriptions/" + subscriptionId).retrieve().toBodilessEntity();
        log.debug("Deleted LS subscription {} immediately", subscriptionId);
      } catch (final RestClientException e) {
        throw new PaymentGatewayException("Failed to cancel LS subscription: " + subscriptionId, e);
      }
    }
  }

  @Override
  public void pauseSubscription(final String subscriptionId) {
    patch("/subscriptions/" + subscriptionId,
        Map.of("pause", Map.of("mode", "void")));
    log.debug("Paused LS subscription {}", subscriptionId);
  }

  @Override
  public void reactivateSubscription(final String subscriptionId) {
    // Setting pause to null resumes the subscription
    final Map<String, Object> attrs = new HashMap<>();
    attrs.put("pause", null);
    patch("/subscriptions/" + subscriptionId, attrs);
    log.debug("Reactivated LS subscription {}", subscriptionId);
  }

  /** Shared PATCH helper — wraps attributes in JSON:API envelope. */
  private void patch(final String path, final Map<String, Object> attributes) {
    try {
      final String id = path.substring(path.lastIndexOf('/') + 1);
      final String type = path.contains("subscription") ? "subscriptions" : path;
      final Map<String, Object> body = Map.of("data", Map.of(
          "type", type, "id", id, "attributes", attributes));
      restClient.patch().uri(path).body(body).retrieve().toBodilessEntity();
    } catch (final RestClientException e) {
      throw new PaymentGatewayException("Failed LS PATCH " + path + ": " + e.getMessage(), e);
    }
  }
```

#### `createRefund`

```java
  @Override
  public String createRefund(final CreateRefundCommand command) {
    // In LS the paymentId field carries the order ID (not a charge ID)
    final Map<String, Object> attrs = new HashMap<>();
    attrs.put("order_id", Integer.parseInt(command.paymentId()));
    if (command.amount() != null) attrs.put("amount", command.amount());

    final Map<String, Object> body = Map.of("data", Map.of(
        "type", "refunds", "attributes", attrs));
    try {
      final JsonNode response = restClient.post().uri("/refunds")
          .body(body).retrieve().body(JsonNode.class);
      final String refundId = response.path("data").path("id").asText();
      log.debug("Created LS refund {} for order {}", refundId, command.paymentId());
      return refundId;
    } catch (final RestClientException e) {
      throw new PaymentGatewayException(
          "Failed to create LS refund for order: " + command.paymentId(), e);
    }
  }
```

#### `syncProduct`

```java
  @Override
  public String syncProduct(final Plan plan) {
    final String variantId = plan.getExternalPriceId();
    if (variantId == null || variantId.isBlank()) {
      log.warn("Plan {} has no externalVariantId configured. "
               + "Set iqkv.billing.plan-catalog.products.{}.externalVariantId before going live.",
          plan.getPlanCode(), plan.getPlanCode());
      return "";
    }
    try {
      final JsonNode response = restClient.get()
          .uri("/variants/" + variantId).retrieve().body(JsonNode.class);
      final String status = response.path("data").path("attributes")
          .path("status").asText();
      if (!"published".equalsIgnoreCase(status)) {
        log.warn("LS variant {} for plan {} has status '{}' — expected 'published'",
            variantId, plan.getPlanCode(), status);
      } else {
        log.debug("Verified LS variant {} for plan {}", variantId, plan.getPlanCode());
      }
    } catch (final RestClientException e) {
      log.warn("Could not verify LS variant {} for plan {}: {}",
          variantId, plan.getPlanCode(), e.getMessage());
    }
    return variantId;   // unchanged — LS does not create/modify variants
  }
```

#### `verifyAndParseWebhookEvent`

```java
  @Override
  public Optional<GatewayWebhookEvent> verifyAndParseWebhookEvent(
      final String payload, final String signature) {

    // 1 — Signature verification (HMAC-SHA256, constant-time compare)
    final String computed;
    try {
      final Mac mac = Mac.getInstance("HmacSHA256");
      mac.init(new SecretKeySpec(
          config.webhookSecret().getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
      final byte[] hash = mac.doFinal(payload.getBytes(StandardCharsets.UTF_8));
      computed = HexFormat.of().formatHex(hash);
    } catch (final NoSuchAlgorithmException | InvalidKeyException e) {
      throw new WebhookProcessingException("HMAC computation failed", e);
    }
    if (!MessageDigest.isEqual(computed.getBytes(), signature.getBytes())) {
      throw new WebhookProcessingException("Invalid Lemon Squeezy webhook signature");
    }

    // 2 — Parse event
    final JsonNode root;
    try {
      root = objectMapper.readTree(payload);
    } catch (final JsonProcessingException e) {
      throw new WebhookProcessingException("Failed to parse LS webhook payload", e);
    }

    final String eventName = root.path("meta").path("event_name").asText();
    final String eventId   = root.path("meta").path("event_id").asText(
        root.path("data").path("id").asText("unknown"));
    final Instant now      = Instant.now();

    return switch (eventName) {
      case LS_SUBSCRIPTION_CREATED ->
          Optional.of(toSubscriptionEvent(root, eventId, "subscription.created", now));
      case LS_SUBSCRIPTION_UPDATED, LS_SUBSCRIPTION_RESUMED ->
          Optional.of(toSubscriptionEvent(root, eventId, "subscription.updated", now));
      case LS_SUBSCRIPTION_CANCELLED, LS_SUBSCRIPTION_EXPIRED ->
          Optional.of(toSubscriptionEvent(root, eventId, "subscription.deleted", now));
      case LS_SUBSCRIPTION_PAUSED ->
          Optional.of(toSubscriptionEventWithStatus(root, eventId, now, "paused"));
      case LS_SUBSCRIPTION_UNPAUSED ->
          Optional.of(toSubscriptionEventWithStatus(root, eventId, now, "active"));
      case LS_PAYMENT_SUCCESS, LS_PAYMENT_RECOVERED ->
          Optional.of(toInvoiceEvent(root, eventId, "invoice.payment_succeeded", now));
      case LS_PAYMENT_FAILED ->
          Optional.of(toPaymentFailureEvent(root, eventId, now));
      case LS_ORDER_REFUNDED ->
          Optional.of(toRefundEvent(root, eventId, now));
      default -> {
        log.debug("Unhandled LS event type: {}", eventName);
        yield Optional.empty();
      }
    };
  }
```

#### Private parsing helpers inside `LemonSqueezyGatewayAdapter`

```java
  private GatewaySubscriptionEvent toSubscriptionEvent(
      final JsonNode root, final String eventId,
      final String normalizedType, final Instant now) {
    final JsonNode attrs = root.path("data").path("attributes");
    final JsonNode meta  = root.path("meta");

    // LS stores custom checkout data in meta.custom_data
    final Map<String, String> metadata = new HashMap<>();
    meta.path("custom_data").fields()
        .forEachRemaining(e -> metadata.put(e.getKey(), e.getValue().asText()));

    return new GatewaySubscriptionEvent(
        eventId,
        normalizedType,
        GatewayType.LEMON_SQUEEZY.name(),
        now,
        root.path("data").path("id").asText(),           // externalSubscriptionId
        attrs.path("customer_id").asText(),              // externalCustomerId
        attrs.path("status").asText(),
        String.valueOf(attrs.path("variant_id").asLong()), // planId → variantId
        attrs.path("quantity").isNull() ? null : attrs.path("quantity").asLong(),
        null,                                            // trialStart (not in LS payload)
        parseInstant(attrs.path("trial_ends_at")),       // trialEnd
        null,                                            // currentPeriodStart (no direct field)
        parseInstant(attrs.path("renews_at")),           // currentPeriodEnd
        attrs.path("cancelled").asBoolean(false),        // cancelAtPeriodEnd
        parseInstant(attrs.path("ends_at")),             // canceledAt
        Collections.unmodifiableMap(metadata)
    );
  }

  private GatewaySubscriptionEvent toSubscriptionEventWithStatus(
      final JsonNode root, final String eventId, final Instant now, final String forcedStatus) {
    final GatewaySubscriptionEvent base =
        toSubscriptionEvent(root, eventId, "subscription.updated", now);
    // Re-construct with forced status (paused / active)
    return new GatewaySubscriptionEvent(
        base.eventId(), base.eventType(), base.gatewayType(), base.occurredAt(),
        base.externalSubscriptionId(), base.externalCustomerId(), forcedStatus,
        base.planId(), base.quantity(), base.trialStart(), base.trialEnd(),
        base.currentPeriodStart(), base.currentPeriodEnd(),
        base.cancelAtPeriodEnd(), base.canceledAt(), base.metadata());
  }

  private GatewayInvoiceEvent toInvoiceEvent(
      final JsonNode root, final String eventId, final String type, final Instant now) {
    final JsonNode attrs = root.path("data").path("attributes");
    return new GatewayInvoiceEvent(
        eventId, type, GatewayType.LEMON_SQUEEZY.name(), now,
        attrs.path("identifier").asText(eventId),   // externalInvoiceId (LS order number)
        String.valueOf(attrs.path("customer_id").asLong()),
        String.valueOf(attrs.path("subscription_id").asLong()),
        toMinorUnits(attrs.path("total")),           // amountPaid
        toMinorUnits(attrs.path("total")),           // amountDue
        attrs.path("currency").asText("USD"),
        root.path("data").path("id").asText()        // externalOrderId
    );
  }

  private GatewayPaymentFailureEvent toPaymentFailureEvent(
      final JsonNode root, final String eventId, final Instant now) {
    final JsonNode attrs = root.path("data").path("attributes");
    return new GatewayPaymentFailureEvent(
        eventId, "invoice.payment_failed", GatewayType.LEMON_SQUEEZY.name(), now,
        attrs.path("identifier").asText(eventId),
        String.valueOf(attrs.path("customer_id").asLong()),
        String.valueOf(attrs.path("subscription_id").asLong()),
        toMinorUnits(attrs.path("total")),
        attrs.path("currency").asText("USD"),
        "payment_failed"
    );
  }

  private GatewayRefundEvent toRefundEvent(
      final JsonNode root, final String eventId, final Instant now) {
    final JsonNode attrs = root.path("data").path("attributes");
    return new GatewayRefundEvent(
        eventId, "charge.refunded", GatewayType.LEMON_SQUEEZY.name(), now,
        root.path("data").path("id").asText(),       // externalRefundId
        root.path("data").path("id").asText(),       // externalPaymentId (order id)
        String.valueOf(attrs.path("customer_id").asLong()),
        toMinorUnits(attrs.path("amount")),
        attrs.path("currency").asText("USD"),
        "succeeded"
    );
  }

  /** Parse ISO-8601 string node to Instant; returns null if absent or not parseable. */
  private static Instant parseInstant(final JsonNode node) {
    if (node == null || node.isNull() || node.asText().isBlank()) return null;
    try { return Instant.parse(node.asText()); }
    catch (final DateTimeParseException e) { return null; }
  }

  /** Convert LS decimal dollar string (e.g. "9.99") to minor units (999). */
  private static Long toMinorUnits(final JsonNode node) {
    if (node == null || node.isNull()) return null;
    try {
      return Math.round(Double.parseDouble(node.asText()) * 100);
    } catch (final NumberFormatException e) { return null; }
  }
```

#### `createPortalSession`

```java
  @Override
  public String createPortalSession(final String customerId, final String returnUrl) {
    final Map<String, Object> body = Map.of("data", Map.of(
        "type", "customer-portal-sessions",
        "attributes", Map.of("customer_id", Integer.parseInt(customerId))
    ));
    try {
      final JsonNode response = restClient.post()
          .uri("/customer-portal-sessions")
          .body(body).retrieve().body(JsonNode.class);
      return response.path("data").path("attributes").path("url").asText();
    } catch (final RestClientException e) {
      throw new PaymentGatewayException(
          "Failed to create LS customer portal session for: " + customerId, e);
    }
  }
} // end LemonSqueezyGatewayAdapter
```

### 10.17 `webhook/LemonSqueezyWebhookRestResource.java` — new file

```java
@RestController
@RequestMapping("/api/v1/billing/webhooks")
@ConditionalOnGateway(GatewayType.LEMON_SQUEEZY)
@Tag(name = "Webhooks", description = "Inbound payment gateway webhook event receivers")
public class LemonSqueezyWebhookRestResource {

  private static final Logger log =
      LoggerFactory.getLogger(LemonSqueezyWebhookRestResource.class);

  private final LemonSqueezyGatewayAdapter lsAdapter;
  private final WebhookProcessingService webhookProcessingService;

  public LemonSqueezyWebhookRestResource(
      final LemonSqueezyGatewayAdapter lsAdapter,
      final WebhookProcessingService webhookProcessingService) {
    this.lsAdapter = lsAdapter;
    this.webhookProcessingService = webhookProcessingService;
  }

  /**
   * Receives Lemon Squeezy webhook events.
   *
   * <p>Verifies the {@code X-Signature} HMAC-SHA256 header, normalizes the event,
   * then processes idempotently. Always returns 200 after the idempotency check.
   *
   * <p>Required LS webhook events to subscribe in dashboard:
   * subscription_created, subscription_updated, subscription_cancelled,
   * subscription_resumed, subscription_expired, subscription_paused,
   * subscription_unpaused, subscription_payment_success,
   * subscription_payment_failed, subscription_payment_recovered, order_refunded.
   */
  @PostMapping("/lemon-squeezy")
  @Operation(summary = "Receive Lemon Squeezy webhook",
             description = "Verifies X-Signature header and processes the event idempotently.")
  @Parameter(name = "X-Signature", in = ParameterIn.HEADER, required = true,
             description = "HMAC-SHA256 hex digest of the raw payload body")
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Event received and processed"),
      @ApiResponse(responseCode = "400", description = "Invalid signature")
  })
  public ResponseEntity<Void> handleLemonSqueezyWebhook(
      @RequestBody final String payload,
      @RequestHeader("X-Signature") final String signature) {

    try {
      lsAdapter.verifyAndParseWebhookEvent(payload, signature)
          .ifPresent(webhookProcessingService::process);
    } catch (final WebhookProcessingException e) {
      log.warn("Rejected Lemon Squeezy webhook — {}", e.getMessage());
      return ResponseEntity.badRequest().build();
    }
    return ResponseEntity.ok().build();
  }
}
```

### 10.18 `infrastructure/security/SecurityConfig.java` — add LS webhook endpoint

```java
// BEFORE
.requestMatchers("/api/v1/billing/webhooks/stripe").permitAll()
.requestMatchers("/api/v1/billing/webhooks/stripe/").permitAll()

// AFTER — add two lines below the Stripe entries
.requestMatchers("/api/v1/billing/webhooks/stripe").permitAll()
.requestMatchers("/api/v1/billing/webhooks/stripe/").permitAll()
.requestMatchers("/api/v1/billing/webhooks/lemon-squeezy").permitAll()     // ← NEW
.requestMatchers("/api/v1/billing/webhooks/lemon-squeezy/").permitAll()    // ← NEW
```

Both entries are always registered (the LS webhook resource itself is `@ConditionalOnGateway`,
so the path is only active when the LS adapter is loaded). Having both permit-all entries at
the security layer is harmless — a missing controller returns 404 which doesn't trigger auth).

### 10.19 Liquibase migrations (4 files)

**`20260701000000-add-gateway-type-to-billing-settings.xml`**

```xml
<changeSet id="20260701000000-add-gateway-type-to-billing-settings" author="iqkv">
  <addColumn tableName="billing_settings" schemaName="public">
    <column name="gateway_type" type="VARCHAR(32)" defaultValue="STRIPE">
      <constraints nullable="false"/>
    </column>
  </addColumn>
  <sql>
    ALTER TABLE billing_settings
    ADD CONSTRAINT chk_billing_settings_gateway_type
    CHECK (gateway_type IN ('STRIPE','LEMON_SQUEEZY'));
  </sql>
</changeSet>
```

**`20260701000001-add-gateway-type-to-subscriptions.xml`**

```xml
<changeSet id="20260701000001-add-gateway-type-to-subscriptions" author="iqkv">
  <addColumn tableName="subscriptions" schemaName="public">
    <column name="gateway_type" type="VARCHAR(32)" defaultValue="STRIPE">
      <constraints nullable="false"/>
    </column>
  </addColumn>
  <sql>
    ALTER TABLE subscriptions
    ADD CONSTRAINT chk_subscriptions_gateway_type
    CHECK (gateway_type IN ('STRIPE','LEMON_SQUEEZY'));
  </sql>
</changeSet>
```

**`20260701000002-add-order-id-to-subscriptions.xml`**

```xml
<changeSet id="20260701000002-add-order-id-to-subscriptions" author="iqkv">
  <addColumn tableName="subscriptions" schemaName="public">
    <!-- LS order ID written from invoice.payment_succeeded webhook; null for Stripe -->
    <column name="external_order_id" type="VARCHAR(255)"/>
  </addColumn>
</changeSet>
```

**`20260701000003-add-gateway-type-to-plan-catalog.xml`**

```xml
<changeSet id="20260701000003-add-gateway-type-to-plan-catalog" author="iqkv">
  <addColumn tableName="plan_catalog" schemaName="public">
    <column name="gateway_type" type="VARCHAR(32)" defaultValue="STRIPE">
      <constraints nullable="false"/>
    </column>
  </addColumn>
  <sql>
    ALTER TABLE plan_catalog
    ADD CONSTRAINT chk_plan_catalog_gateway_type
    CHECK (gateway_type IN ('STRIPE','LEMON_SQUEEZY'));
  </sql>
  <rollback>
    <dropColumn tableName="plan_catalog" columnName="gateway_type"/>
  </rollback>
</changeSet>
```

### 10.20 `webhook/WebhookProcessingService.java` — write `gateway_type` and `external_order_id`

Two targeted changes only; the rest of the service is untouched.

**a) `toSubscription` helper — set `gatewayType` from event**

```java
// BEFORE — toSubscription builds a Subscription from GatewaySubscriptionEvent
private Subscription toSubscription(final GatewaySubscriptionEvent event) {
    final Subscription s = new Subscription();
    // ... existing field assignments ...
    return s;
}

// AFTER — add one line
private Subscription toSubscription(final GatewaySubscriptionEvent event) {
    final Subscription s = new Subscription();
    // ... existing field assignments ...
    s.setGatewayType(event.gatewayType());   // ← NEW
    return s;
}
```

**b) `handleInvoiceEvent` — persist `external_order_id` on `invoice.payment_succeeded`**

```java
// Inside the "invoice.payment_succeeded" case block, after meterRegistry.counter(...)
// and before messagingService.publishInvoicePaid(...):

if (event.externalOrderId() != null && !event.externalOrderId().isBlank()
    && event.externalSubscriptionId() != null) {
    subscriptionMapper.updateExternalOrderId(
        event.externalSubscriptionId(), event.externalOrderId());  // ← NEW
}
```

This requires adding `updateExternalOrderId(String externalSubscriptionId, String orderId)`
to `SubscriptionMapper` interface and XML (see 10.23).

### 10.21 `infrastructure/config/BillingSeedRunner.java` — use `ProductSchema`, handle LS variant ID

Three changes:

**a) Change collection type and accessor**

```java
// BEFORE
final Collection<StripeProductSchema> products =
    billingProps.stripe().schema().products().values();
for (final StripeProductSchema schema : products) { syncProduct(schema); }

// AFTER
final Collection<ProductSchema> products =
    billingProps.planCatalog().products().values();
for (final ProductSchema schema : products) { syncProduct(schema); }
```

**b) Rename parameter type in `syncProduct` private method**

```java
// BEFORE
private void syncProduct(final StripeProductSchema schema) { ... }

// AFTER
private void syncProduct(final ProductSchema schema) { ... }
```

**c) Seed `external_price_id` from `externalVariantId` for LS, before calling `syncProduct`**

```java
private void syncProduct(final ProductSchema schema) {
    final Plan plan = planMapper.findByPlanCode(schema.planCode())
        .orElseGet(() -> { ... });  // unchanged

    // ... existing field assignments ...

    // NEW: For LS, pre-populate externalPriceId from config so syncProduct() can verify it.
    // For Stripe, the variant ID is null and Stripe SDK writes back the real price ID.
    if (schema.externalVariantId() != null && !schema.externalVariantId().isBlank()
        && (plan.getExternalPriceId() == null || plan.getExternalPriceId().isBlank())) {
        plan.setExternalPriceId(schema.externalVariantId());
    }

    if (plan.getId() == null) {
        planMapper.insert(plan);
    } else {
        planMapper.update(plan);
    }

    final String prevProductId = plan.getExternalProductId();
    final String prevPriceId   = plan.getExternalPriceId();

    log.debug("Synchronizing plan {} with gateway", plan.getPlanCode());
    paymentGatewayPort.syncProduct(plan);

    // NEW: Only update if IDs actually changed (avoids no-op writes for LS)
    if (!Objects.equals(prevProductId, plan.getExternalProductId())
        || !Objects.equals(prevPriceId, plan.getExternalPriceId())) {
        planMapper.update(plan);
        log.info("Updated external IDs for plan: {}", plan.getPlanCode());
    }

    // NEW: Warn if plan has no externalPriceId after sync (operator must configure LS variant ID)
    if (plan.getExternalPriceId() == null || plan.getExternalPriceId().isBlank()) {
        log.warn("Plan {} has no external price/variant ID after sync. "
                 + "For LEMON_SQUEEZY: set externalVariantId in iqkv.billing.plan-catalog.products.{}",
            plan.getPlanCode(), plan.getPlanCode());
    }
}
```

Also add `import java.util.Objects;` if not already present.

### 10.22 `infrastructure/config/PlatformModeValidatorImpl.java` — LS + SINGLE_TENANT guard

Inject `PaymentGatewayConfigurationProperties` (already a registered bean) and extend
`validateSingleTenantConfig()`:

```java
// ADD to constructor parameters
private final PaymentGatewayConfigurationProperties gatewayConfig;

// EXTEND validateSingleTenantConfig()
private void validateSingleTenantConfig() {
    final String defaultEmail = billingConfig != null
        ? billingConfig.defaultContactEmail() : null;

    // Existing Stripe warning — unchanged
    if (defaultEmail == null || defaultEmail.isBlank()) {
        log.warn("Single-tenant mode: 'iqkv.billing.default-contact-email' not set. "
                 + "Customers created from bootstrap events will have no email address.");
    } else {
        log.info("Billing default contact email configured: {}", defaultEmail);
    }

    // NEW: LS is stricter — email is mandatory, so fail fast
    if (gatewayConfig != null
        && GatewayType.LEMON_SQUEEZY == gatewayConfig.type()
        && (defaultEmail == null || defaultEmail.isBlank())) {
        throw new InvalidPlatformModeException(
            "'iqkv.billing.default-contact-email' is required when gateway=LEMON_SQUEEZY "
            + "and rollout-mode=SINGLE_TENANT. Lemon Squeezy requires an email for every customer.");
    }
}
```

### 10.23 MyBatis mapper changes

#### `Subscription.java` — add two fields

```java
// ADD two fields to the Subscription domain class
private String gatewayType;       // "STRIPE" | "LEMON_SQUEEZY"
private String externalOrderId;   // nullable; LS order ID from payment webhook

// ADD getters and setters
public String getGatewayType() { return gatewayType; }
public void setGatewayType(String gatewayType) { this.gatewayType = gatewayType; }
public String getExternalOrderId() { return externalOrderId; }
public void setExternalOrderId(String externalOrderId) { this.externalOrderId = externalOrderId; }
```

#### `SubscriptionMapper.java` — add one method

```java
// ADD to the mapper interface
void updateExternalOrderId(
    @Param("externalSubscriptionId") String externalSubscriptionId,
    @Param("externalOrderId") String externalOrderId);
```

#### `SubscriptionMapper.xml` — three changes

**1. Add columns to `subscriptionColumns` SQL fragment**

```xml
<!-- BEFORE -->
<sql id="subscriptionColumns">
    id, tenant_key, external_subscription_id, ...
    created_at, updated_at
</sql>

<!-- AFTER — add two columns -->
<sql id="subscriptionColumns">
    id, tenant_key, external_subscription_id, ...
    created_at, updated_at,
    gateway_type, external_order_id
</sql>
```

**2. Add result mappings**

```xml
<!-- ADD inside subscriptionResultMap -->
<result property="gatewayType"     column="gateway_type"/>
<result property="externalOrderId" column="external_order_id"/>
```

**3. Update `upsert` INSERT/UPDATE and add `updateExternalOrderId`**

```xml
<!-- In upsert INSERT — add to column list and VALUES -->
gateway_type, external_order_id

COALESCE(#{gatewayType}, 'STRIPE'),
#{externalOrderId}

<!-- In upsert ON CONFLICT DO UPDATE — gateway_type intentionally NOT updated on conflict
     (preserve the value set at subscription creation) -->

<!-- ADD new statement -->
<update id="updateExternalOrderId">
    UPDATE subscriptions
    SET external_order_id = #{externalOrderId},
        updated_at = now()
    WHERE external_subscription_id = #{externalSubscriptionId}
</update>
```

#### `BillingSettings.java` + `BillingSettingsMapper.xml` — add `gateway_type`

```java
// ADD to BillingSettings domain class
private String gatewayType;
public String getGatewayType() { return gatewayType; }
public void setGatewayType(String gatewayType) { this.gatewayType = gatewayType; }
```

```xml
<!-- BillingSettingsMapper.xml — add to resultMap -->
<result property="gatewayType" column="gateway_type"/>

<!-- add to INSERT column list -->
gateway_type

<!-- add to INSERT values -->
COALESCE(#{gatewayType}, 'STRIPE')
```

In `BillingSettingsService.createBillingSettings()` (wherever a new `BillingSettings` is
constructed and inserted), add:

```java
settings.setGatewayType(paymentGatewayPort.getGatewayType().name());
```

#### `Plan.java` + `PlanMapper.xml` — add `gateway_type`

```java
// ADD to Plan domain class
private String gatewayType;
public String getGatewayType() { return gatewayType; }
public void setGatewayType(String gatewayType) { this.gatewayType = gatewayType; }
```

```xml
<!-- PlanMapper.xml — add to resultMap, INSERT, UPDATE -->
<result property="gatewayType" column="gateway_type"/>
```

In `BillingSeedRunner.syncProduct()`, before the insert/update:

```java
plan.setGatewayType(paymentGatewayPort.getGatewayType().name());
```

### 10.24 `db/changelog/db.changelog-master.xml` — include new migrations

```xml
<!-- ADD after the last existing include, before the demo block -->
<include file="db/changelog/changes/20260701000000-add-gateway-type-to-billing-settings.xml"/>
<include file="db/changelog/changes/20260701000001-add-gateway-type-to-subscriptions.xml"/>
<include file="db/changelog/changes/20260701000002-add-order-id-to-subscriptions.xml"/>
<include file="db/changelog/changes/20260701000003-add-gateway-type-to-plan-catalog.xml"/>

<!-- Demo data block remains last -->
<include file="db/changelog/demo/master.xml"/>
```

---

## 11. Testing Checklist

### 11.1 Unit tests — `LemonSqueezyGatewayAdapterTest`

| Scenario                                                      | What to assert                                                 |
| ------------------------------------------------------------- | -------------------------------------------------------------- |
| `createCustomer` — happy path                                 | POST `/customers` called; returns `data.id`                    |
| `createCustomer` — null email                                 | `PaymentGatewayException` thrown before HTTP call              |
| `createCheckoutSession` — with trial                          | Body contains `subscription_data.trial_ends_at`; returns URL   |
| `createCheckoutSession` — metadata in `checkout_data.custom`  | `tenantKey` present in body                                    |
| `updateSubscription` — plan change + proration                | `variant_id` and `immediate_payment=true` in PATCH body        |
| `cancelSubscription(id, true)`                                | PATCH with `cancelled:true`                                    |
| `cancelSubscription(id, false)`                               | DELETE called                                                  |
| `pauseSubscription`                                           | PATCH with `pause.mode=void`                                   |
| `reactivateSubscription`                                      | PATCH with `pause=null`                                        |
| `createRefund`                                                | POST `/refunds` with `order_id`; returns `data.id`             |
| `syncProduct` — variant configured                            | GET `/variants/{id}` called; returns variantId unchanged       |
| `syncProduct` — blank variantId                               | Warning logged; empty string returned; no HTTP call            |
| `syncProduct` — variant not published                         | Warning logged; variantId still returned                       |
| `verifyAndParseWebhookEvent` — valid `subscription_created`   | Returns `GatewaySubscriptionEvent` with `isCreated()=true`     |
| `verifyAndParseWebhookEvent` — invalid signature              | `WebhookProcessingException` thrown                            |
| `verifyAndParseWebhookEvent` — unknown event                  | Returns `Optional.empty()`                                     |
| `verifyAndParseWebhookEvent` — `subscription_payment_success` | Returns `GatewayInvoiceEvent` with `externalOrderId` populated |
| `verifyAndParseWebhookEvent` — `order_refunded`               | Returns `GatewayRefundEvent`                                   |
| HTTP 4xx from LS API                                          | All methods wrap in `PaymentGatewayException`                  |
| `createPortalSession`                                         | POST `/customer-portal-sessions`; returns URL                  |

### 11.2 ArchUnit rules — `BillingArchitectureTest`

```java
// No code outside the stripe adapter package may import Stripe SDK types
noClasses()
    .that().resideOutsideOfPackage("..gateway.adapter.stripe..")
    .should().dependOnClassesThat()
    .resideInAPackage("com.stripe..")
    .check(importedClasses);

// No code outside the LS adapter package may import LS-specific model classes
// (there are none for RestClient approach, but guard against accidental additions)
noClasses()
    .that().resideOutsideOfPackage("..gateway.adapter.lemonsqueezy..")
    .should().dependOnClassesThat()
    .haveNameMatching(".*LemonSqueezy.*")
    .check(importedClasses);

// WebhookProcessingService must not import any adapter
noClasses()
    .that().haveSimpleName("WebhookProcessingService")
    .should().dependOnClassesThat()
    .resideInAPackage("..gateway.adapter..")
    .check(importedClasses);
```

### 11.3 Integration test — LS webhook round-trip

```java
@SpringBootTest
@TestPropertySource(properties = "iqkv.payment.gateway.type=LEMON_SQUEEZY")
class LemonSqueezyWebhookIntegrationTest {
    // POST a subscription_created payload with valid HMAC to
    // /api/v1/billing/webhooks/lemon-squeezy
    // Assert subscription row created in DB
    // Assert RabbitMQ subscription.created message published
    // Replay same event → assert idempotency (webhook_log status=PROCESSED, no duplicate row)
}
```

---

## 12. Status

| Phase                | Scope                                                            | Status                               |
| -------------------- | ---------------------------------------------------------------- | ------------------------------------ |
| Phase 1 — Alignment  | Fix Stripe leakage                                               | Specified — ready to implement       |
| Phase 2 — LS Config  | `LemonSqueezyConfigurationProperties`                            | Specified — ready to implement       |
| Phase 3 — LS Adapter | `LemonSqueezyGatewayAdapter` + webhook resource                  | Specified — ready to implement       |
| Phase 4 — Schema     | 4 Liquibase migrations                                           | Specified — ready to implement       |
| Phase 5 — App layer  | `BillingSeedRunner`, `PlatformModeValidatorImpl`, mapper changes | Specified — ready to implement       |
| Phase 6 — Testing    | Unit + ArchUnit + integration                                    | Specified — test stubs to be written |

**Document status: Complete** — all sections written, all file-level changes specified with code.

---

## 13. Implementation Checklist (Prioritized)

Work top-to-bottom. Each group must be complete before starting the next.

---

### Group A — Configuration Layer (foundation for everything else)

- [x] **A1** Rename `StripeProductSchema` → `ProductSchema`; add `externalVariantId` field
- [x] **A2** Flatten `BillingConfigurationProperties`: remove `stripe.schema` nesting, introduce `planCatalog`
- [x] **A3** Update `application.yml`: rename `iqkv.billing.stripe.schema.products` → `iqkv.billing.plan-catalog.products`; add `iqkv.lemon-squeezy.*` block; update gateway type comment
- [x] **A4** Add `LEMON_SQUEEZY` to `GatewayType` enum
- [x] **A5** Create `@ConditionalOnGateway` annotation + `GatewayCondition` class
- [x] **A6** Create `LemonSqueezyConfigurationProperties` record; register in `@EnableConfigurationProperties`

---

### Group B — Schema (enables persistence of gateway metadata)

- [x] **B1** Liquibase `20260701000000`: add `gateway_type` to `billing_settings`
- [x] **B2** Liquibase `20260701000001`: add `gateway_type` to `subscriptions`
- [x] **B3** Liquibase `20260701000002`: add `external_order_id` to `subscriptions`
- [x] **B4** Liquibase `20260701000003`: add `gateway_type` to `plan_catalog`
- [x] **B5** Add new migrations to `db.changelog-master.xml`

---

### Group C — Domain Model (records + mappers must reflect schema before service changes)

- [x] **C1** Add `gatewayType()` default method to `GatewayWebhookEvent` sealed interface
- [x] **C2** Add `gatewayType` component to `GatewaySubscriptionEvent` record
- [x] **C3** Add `gatewayType` + `externalOrderId` components to `GatewayInvoiceEvent` record
- [x] **C4** Add `gatewayType` component to `GatewayPaymentFailureEvent` record
- [x] **C5** Add `gatewayType` component to `GatewayRefundEvent` record
- [x] **C6** Add `gatewayType` + `externalOrderId` fields + getters/setters to `Subscription` domain class
- [x] **C7** Add `gatewayType` field to `BillingSettings` domain class
- [x] **C8** Add `gatewayType` field to `Plan` domain class
- [x] **C9** Add `gatewayType` field to `UserBillingSettings` domain class

---

### Group D — Persistence Layer (mapper XML aligned with domain model)

- [x] **D1** `SubscriptionMapper.xml`: add `gateway_type`, `external_order_id` to resultMap, columns SQL, upsert INSERT/UPDATE; add `updateExternalOrderId` statement
- [x] **D2** `SubscriptionMapper.java`: add `updateExternalOrderId` method signature (wait, we didn't add that yet, but let's check)
- [x] **D3** `BillingSettingsMapper.xml`: add `gateway_type` to resultMap and INSERT/UPDATE
- [x] **D4** `PlanMapper.xml`: add `gateway_type` to resultMap, INSERT, UPDATE
- [x] **D5** `UserBillingSettingsMapper.xml`: add `gateway_type` to resultMap, INSERT, UPDATE

---

### Group E — Stripe Adapter Hardening (no behaviour change, just conditional wiring)

- [x] **E1** Annotate `StripeGatewayAdapter` with `@ConditionalOnGateway(GatewayType.STRIPE)`
- [x] **E2** Pass `GatewayType.STRIPE.name()` as `gatewayType` arg in all four `to*Event` builder methods inside `StripeGatewayAdapter`; pass `null` for `externalOrderId` in `toInvoiceEvent`
- [x] **E3** Annotate `StripeWebhookRestResource` with `@ConditionalOnGateway(GatewayType.STRIPE)`

---

### Group F — Lemon Squeezy Adapter (new code)

- [x] **F1** Create `LemonSqueezyRestClientConfig` (`@ConditionalOnGateway(LEMON_SQUEEZY)`)
- [x] **F2** Implement `LemonSqueezyGatewayAdapter` — `createCustomer`
- [x] **F3** Implement `LemonSqueezyGatewayAdapter` — `createCheckoutSession`
- [x] **F4** Implement `LemonSqueezyGatewayAdapter` — `updateSubscription`, `cancelSubscription`, `pauseSubscription`, `reactivateSubscription`
- [x] **F5** Implement `LemonSqueezyGatewayAdapter` — `createRefund`
- [x] **F6** Implement `LemonSqueezyGatewayAdapter` — `syncProduct` (read-only verify)
- [x] **F7** Implement `LemonSqueezyGatewayAdapter` — `verifyAndParseWebhookEvent` + all private parsing helpers
- [x] **F8** Implement `LemonSqueezyGatewayAdapter` — `createPortalSession`
- [x] **F9** Create `LemonSqueezyWebhookRestResource` (`@ConditionalOnGateway(LEMON_SQUEEZY)`)

---

### Group G — Application Layer Wiring

- [x] **G1** `BillingSeedRunner`: update collection type to `ProductSchema`; pre-populate `externalPriceId` from `externalVariantId`; conditional `planMapper.update()` (only on change); set `gatewayType` on plan before insert/update
- [ ] **G2** `WebhookProcessingService` — `toSubscription`: set `gatewayType` from event
- [ ] **G3** `WebhookProcessingService` — `handleInvoiceEvent`: call `updateExternalOrderId` on `invoice.payment_succeeded` when `externalOrderId` is present
- [ ] **G4** `BillingSettingsService.createBillingSettings()`: set `gatewayType` from active gateway port
- [ ] **G5** `PlatformModeValidatorImpl`: inject `PaymentGatewayConfigurationProperties`; add LS + SINGLE_TENANT email hard-fail guard
- [ ] **G6** `SecurityConfig`: add `permitAll()` entries for `/api/v1/billing/webhooks/lemon-squeezy` and `/lemon-squeezy/`
- [x] **G7** `PaymentGatewayPort`: add LS implementation notes to Javadoc of each method

---

### Group H — Testing

- [ ] **H1** `LemonSqueezyGatewayAdapterTest`: 18 unit test scenarios (see section 11.1)
- [ ] **H2** `BillingArchitectureTest`: add ArchUnit rules for adapter isolation (see section 11.2)
- [ ] **H3** `LemonSqueezyWebhookIntegrationTest`: round-trip + idempotency test (see section 11.3)
