# Proposal: Plan Feature Access Control

## Overview

This document proposes a typed, config-driven plan feature system for the IQKV platform,
with enforcement at the gateway and minimal impact on downstream services.

The key principle is **"Billing owns the feature contract"** — plan features are defined
once in `foundation-billing-service` YAML configuration, served via an internal API,
and consumed by the gateway and downstream services through a local in-memory cache.
No feature definitions are duplicated across codebases, and no network call is made on
the hot enforcement path.

## Goals

- Replace the current opaque `entitlement` JSON string on plans with a typed, validated
  `PlanEntitlement` struct bound directly from YAML.
- Enforce boolean feature access at the gateway via route-level filters — downstream
  services require no changes for feature gating.
- Enable quota enforcement (`maxUsers`, `maxProjects`) in downstream services using the
  same cached plan data — checked at write time only.
- Keep `foundation-billing-service` as the single authority on what each plan includes.
- Support Helm-driven feature changes with no code changes — a values update + rolling
  deploy is sufficient.

## Proposed Architecture

### 1. Affected Modules

| Module                       | Role                                                                                                             |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `foundation-billing-service` | Source of truth — typed `PlanEntitlement`, `PlanFeatureRegistry`, internal plans endpoint, entitlements endpoint |
| `foundation-iam-service`     | Caches active `planCode` on tenant; stamps `plan_code` claim into JWT                                            |
| `foundation-gateway-service` | Propagates `X-Plan-Code` header; `PlanResolver`; `RequiresPlanFeatureFilter`                                     |
| Downstream services          | `PlanResolver` for quota checks at write time only                                                               |

### 2. High-Level Design

```
YAML / Helm values
        │
        ▼
BillingSeedRunner (on startup)
        │  upserts plan_catalog with entitlement JSON snapshot
        │  populates PlanFeatureRegistry (in-memory)
        │
        ├──► GET /api/v1/internal/plans ◄── Gateway PlanResolver   (startup + every 10m)
        │                                ◄── Downstream PlanResolver (startup + every 10m)
        │
Stripe webhook ──► WebhookProcessingService
                         │  upserts subscription
                         │  publishes SubscriptionEvent (+ planCode)
                         │
                         ▼
                  IAM SubscriptionEventConsumer
                         │  stores planCode on tenant
                         │
                         ▼
                  JwtTokenGenerator
                         │  stamps plan_code claim into JWT
                         │
HTTP Request ──► Gateway JwtContextPropagationFilter
                         │  adds X-Plan-Code header
                         │
                  RequiresPlanFeatureFilter
                         │  planResolver.resolveEntitlement(planCode).has("priority_support")
                         │  → 402 if false, forward if true
                         │
                         ▼
                  Downstream Service
                         │
                    quota check (write path only):
                    planResolver.resolveEntitlement(planCode).maxUsers()
                    vs current DB count → 402 if exceeded
```

### 3. `PlanEntitlement` — typed struct (`foundation-billing-service`)

New record `com.iqkv.foundation.billingservice.plan.PlanEntitlement`, bound from YAML via
`StripeProductSchema.features`. Replaces the current `String entitlement` field.

Quota fields (`maxUsers`, `maxProjects`) remain typed `int` for compile-time safety.
Display/boolean features are held in an open `Map<String, PlanFeature>` keyed by
feature code (snake_case, matches YAML). Adding a new feature requires only a YAML
change — no Java recompilation of any service.

```java
public record PlanFeature(
    String code,        // map key — e.g. "priority_support"
    String title,       // human-readable label
    String value,       // "true"/"false" for boolean; number string for limits
    String description  // optional tooltip text
) {
  public boolean isEnabled() {
    return "true".equalsIgnoreCase(value);
  }
}

public record PlanEntitlement(
    int maxUsers,       // 0 = unlimited
    int maxProjects,    // 0 = unlimited
    Map<String, PlanFeature> features
) {
  public static final PlanEntitlement NONE = new PlanEntitlement(1, 1, Map.of());

  public PlanEntitlement {
    if (maxUsers < 0)    throw new IllegalArgumentException("maxUsers must be >= 0");
    if (maxProjects < 0) throw new IllegalArgumentException("maxProjects must be >= 0");
    features = features != null
        ? Collections.unmodifiableMap(features)
        : Collections.emptyMap();
  }

  /** O(1) lookup — returns true if the feature code exists and value is "true". */
  public boolean has(final String code) {
    if (code == null || code.isBlank()) return false;
    final PlanFeature f = features.get(code);
    return f != null && f.isEnabled();
  }
}
```

YAML structure (Helm values follow the same path):

```yaml
iqkv:
  billing:
    stripe:
      schema:
        products:
          basic-monthly:
            planCode: "basic-monthly"
            displayName: "Basic Monthly"
            billingPeriod: "MONTHLY"
            priceMinor: 1000
            currency: "USD"
            scope: "TENANT"
            active: true
            maxUsers: 5
            maxProjects: 3
            features: {} # no boolean features on basic
          pro-monthly:
            planCode: "pro-monthly"
            displayName: "Pro Monthly"
            billingPeriod: "MONTHLY"
            priceMinor: 3000
            currency: "USD"
            scope: "TENANT"
            active: true
            maxUsers: 50
            maxProjects: 0 # 0 = unlimited
            features:
              priority_support:
                title: "Priority Support"
                value: "true"
                description: "Access to priority support channel"
```

### 4. `PlanFeatureRegistry` — in-memory, billing-service only

Populated at startup from `BillingConfigurationProperties`. Read-only after init.
Serves the internal plans endpoint with zero DB reads.

```java
@Component
public class PlanFeatureRegistry {

  private final Map<String, PlanEntitlement> registry;

  public PlanFeatureRegistry(final BillingConfigurationProperties props) {
    this.registry = props.stripe().schema().products().values().stream()
        .filter(s -> s.planCode() != null)
        .collect(Collectors.toUnmodifiableMap(
            StripeProductSchema::planCode,
            s -> s.features() != null ? s.features() : PlanEntitlement.NONE
        ));
  }

  public PlanEntitlement resolveEntitlement(final String planCode) {
    if (planCode == null) return PlanEntitlement.NONE;
    return registry.getOrDefault(planCode, PlanEntitlement.NONE);
  }

  public Set<String> knownPlanCodes() {
    return registry.keySet();
  }
}
```

### 5. Internal Plans Endpoint (`foundation-billing-service`)

Service-to-service endpoint, not exposed through the public gateway route.

```
GET /api/v1/billing/internal/plans

200 OK
[
  {
    "planCode": "basic-monthly",
    "features": { "maxUsers": 5, "maxProjects": 3, "features": {} }
  },
  {
    "planCode": "pro-monthly",
    "features": {
      "maxUsers": 50,
      "maxProjects": 0,
      "features": {
        "priority_support": {
          "code": "priority_support",
          "title": "Priority Support",
          "value": "true",
          "description": "Access to priority support channel"
        }
      }
    }
  }
]
```

Implementation: one `@RestController` backed by `PlanFeatureRegistry`. No DB read.
The `/billing/internal/` prefix keeps it under the existing billing service route in
the gateway — no new gateway route definition needed.

### 6. `PlanResolver` — gateway and downstream services

Fetches the internal plans endpoint at startup and refreshes on a schedule.
Falls back to last known state on transient billing unavailability.
Each consumer service holds its own instance — no shared library required.

```java
@Component
public class PlanResolver {

  private volatile Map<String, PlanEntitlement> cache = Map.of();
  private final WebClient billingClient;

  @PostConstruct
  public void loadOnStartup() { refresh(); }

  @Scheduled(fixedDelayString = "${iqkv.plan-catalog.refresh-interval:PT10M}")
  public void refresh() {
    try {
      final List<PlanCatalogEntry> plans = billingClient.get()
          .uri("/api/v1/billing/internal/plans")
          .retrieve()
          .bodyToFlux(PlanCatalogEntry.class)
          .collectList()
          .block(Duration.ofSeconds(5));
      if (plans != null && !plans.isEmpty()) {
        cache = plans.stream().collect(
            Collectors.toUnmodifiableMap(PlanCatalogEntry::planCode, PlanCatalogEntry::features));
      }
    } catch (final Exception e) {
      log.warn("Failed to refresh plan catalog, using last known state: {}", e.getMessage());
    }
  }

  public PlanEntitlement resolveEntitlement(final String planCode) {
    if (planCode == null) return PlanEntitlement.NONE;
    return cache.getOrDefault(planCode, PlanEntitlement.NONE);
  }
}
```

`PlanEntitlement` and `PlanCatalogEntry` are local records in each consumer service —
only the fields that service needs. No cross-service jar dependency.

Configuration:

```yaml
iqkv:
  plan-catalog:
    refresh-interval: PT10M
    billing-base-url: ${BILLING_SERVICE_URL:http://foundation-billing-service}
```

### 7. JWT and Gateway Enforcement

**IAM** — `SubscriptionEventConsumer` caches `planCode` on the tenant when a
subscription is created or updated. `JwtTokenGenerator` stamps one new claim:

```java
.claim("plan_code", tenant.getActivePlanCode())   // e.g. "pro-monthly"
```

Feature definitions stay in billing. IAM has no knowledge of what features a plan
includes — only which plan a tenant is on.

**Gateway** — `JwtContextPropagationFilter` propagates one header downstream:

```java
setIfPresent(headers, "X-Plan-Code", jwt.getClaimAsString("plan_code"));
```

`RequiresPlanFeatureFilterFactory` enforces boolean features at the route level.
Downstream services receive the request only if the plan check passes:

```java
public class RequiresPlanFeatureFilterFactory
    extends AbstractGatewayFilterFactory<RequiresPlanFeatureFilterFactory.Config> {

  private final PlanResolver planResolver;

  @Override
  public GatewayFilter apply(final Config config) {
    return (exchange, chain) -> {
      final String planCode = exchange.getRequest().getHeaders().getFirst("X-Plan-Code");
      if (!planResolver.resolveEntitlement(planCode).has(config.getFeature())) {
        exchange.getResponse().setStatusCode(HttpStatus.PAYMENT_REQUIRED);
        return exchange.getResponse().setComplete();
      }
      return chain.filter(exchange);
    };
  }

  public static class Config {
    private String feature;
    public String getFeature() { return feature; }
    public void setFeature(final String feature) { this.feature = feature; }
  }
}
```

Route configuration:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: priority-support-route
          uri: lb://foundation-support-service
          predicates:
            - Path=/api/v1/support/priority/**
          filters:
            - RequiresPlanFeature=priority_support
```

### 8. Quota Enforcement in Downstream Services

Quota checks (`maxUsers`, `maxProjects`) require current DB counts and cannot run at
the gateway. The owning service checks at **write time only** using its local
`PlanResolver` — no synchronous call to billing on the hot path:

```java
final PlanEntitlement planEntitlement = planResolver.resolveEntitlement(
    request.getHeader("X-Plan-Code"));

final int current = userRepository.countByTenantKey(tenantKey);
if (planEntitlement.maxUsers() > 0 && current >= planEntitlement.maxUsers()) {
    throw new QuotaExceededException("User limit reached for your current plan");
}
```

`QuotaExceededException` maps to HTTP `402`.

### 9. Entitlements Endpoint (`foundation-billing-service`)

User-facing endpoint for UI plan status display and one-off lookups.

```
GET /api/v1/billing/entitlements/me
Authorization: Bearer <jwt>
X-Tenant-ID: <tenantKey>

200 OK
{
  "planCode": "pro-monthly",
  "status": "active",
  "currentPeriodEnd": "2026-07-15T00:00:00Z",
  "features": {
    "maxUsers": 50,
    "maxProjects": 0,
    "features": {
      "priority_support": {
        "code": "priority_support",
        "title": "Priority Support",
        "value": "true",
        "description": "Access to priority support channel"
      }
    }
  }
}

404 — no active subscription
```

Delegates to `EntitlementEvaluator`. One `@RestController` class, no new service layer.

## Benefits

- **Single source of truth.** Feature definitions live exclusively in billing YAML.
  Adding or changing a feature requires editing one file and deploying one service.
- **Zero downstream coupling for boolean gating.** The gateway handles all feature
  enforcement. Downstream services need no `@PreAuthorize` annotations, no shared
  billing dependency, no knowledge of plans.
- **No hot-path network calls.** Both the gateway filter and downstream quota checks
  use an in-memory cache. The only I/O is a periodic background refresh.
- **Helm-friendly.** Feature changes are a `values.yaml` diff — auditable, reviewable,
  rollback-safe via standard Helm release history.
- **Graceful degradation.** If billing is temporarily unreachable, the last known
  catalog serves requests. The cache only resets if the service restarts with an
  empty cache and billing is still unreachable.

## Implementation Roadmap

### Phase 1 — Billing Service Internals

- [x] Add `PlanEntitlement` record to `foundation-billing-service`.
- [x] Replace `String entitlement` with `PlanEntitlement features` in `StripeProductSchema`.
- [x] Update YAML plan definitions to include typed `features` block.
- [x] Add `PlanFeatureRegistry` bean.
- [x] Update `BillingSeedRunner` to serialize `PlanEntitlement` into the `entitlement` DB column.
- [x] Update `EntitlementDetails` record — replace raw `entitlement` string with `planCode` + `PlanEntitlement`.
- [x] Update `DefaultEntitlementEvaluator` to use `PlanFeatureRegistry`.
- [x] Add `GET /api/v1/billing/internal/plans` endpoint.
- [x] Add `GET /api/v1/billing/entitlements/me` endpoint.

### Phase 2 — IAM and JWT

- [x] Add `active_plan_code VARCHAR(64)` column to `tenants` table (Liquibase migration).
- [x] Add `planCode` field to `SubscriptionEvent`.
- [x] Extend `SubscriptionEventConsumer` in IAM to handle `SUBSCRIPTION_CREATED` and
      `SUBSCRIPTION_UPDATED` — persist `planCode` on tenant.
- [x] Update `JwtTokenGenerator` to include `plan_code` claim.

### Phase 3 — Gateway Enforcement

- [x] Update `JwtContextPropagationFilter` to propagate `X-Plan-Code` header.
- [x] Add `PlanResolver` with `@Scheduled` refresh in `foundation-gateway-service`.
- [x] Implement `RequiresPlanFeatureFilterFactory`.
- [x] Add `RequiresPlanFeature` filter to applicable routes in gateway configuration.

### Phase 4 — Downstream Quota Checks (per service, incremental)

- [x] For each service that manages a quota-bounded resource: add local `PlanResolver`
      and enforce quota limits at resource creation.

## What Is Not Included

- No feature flag DB table — features are a product decision owned by config.
- No per-tenant feature overrides — if needed later, add an override table in
  billing-service and merge it in `PlanFeatureRegistry.resolveEntitlement()`. The internal
  endpoint reflects the merged result transparently; consumers need no changes.
- No real-time toggle without a cache refresh — a 10-minute TTL is acceptable since
  plan features change at deploy time, not at runtime.
- No `foundation-billing-spi` shared library — `PlanResolver` and the local
  `PlanEntitlement` record are small, per-service copies. Extract to a shared lib only
  if the pattern spreads to four or more services.
