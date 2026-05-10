# Single-Tenant Mode Implementation Spec

## 1. Purpose

Define the required implementation changes so the platform can run in **single-tenant mode** where:

- users self-register via signup,
- each new user is auto-joined to one pre-provisioned default tenant,
- tenant key is always NanoID-based (including default tenant),
- there is no functional tenant-owner concept for customer users (an internal system user may exist only for audit/bootstrap compatibility).

This spec aligns `system-design-documentation`, `foundation-iam-service`, and `foundation-billing-service`.

## 2. Current State (As-Is)

### 2.1 Documentation

`docs/platform/tenancy.md` and `docs/platform/architecture.md` already describe hybrid tenancy and a hidden default tenant for single mode.

### 2.2 IAM implementation gaps

- Signup currently always creates a brand-new tenant and grants `TENANT_OWNER`.
- `RegisterUserRequest` requires `tenantName`.
- No effective `tenancy.mode` switch exists in runtime logic.
- No bootstrap workflow exists to resolve/create a single default tenant record.
- `TenantMapper` supports uniqueness by tenant name only; no concept of "default tenant marker."
- Tenant owner is assumed by APIs/authorization and retry provisioning logic.

### 2.3 Billing implementation gaps

- `TenantEventConsumer` requires `ownerEmail` for `TENANT_CREATED` and throws if missing.
- Billing bootstrap path is tied to owner-centric event payload semantics.
- No behavior branch for single-tenant ownerless onboarding.

## 3. Target Behavior (To-Be)

### 3.1 Mode switch

Introduce explicit tenancy mode configuration:

- `iqkv.tenancy.mode = MULTI | SINGLE` (default `MULTI`).
- Platform rollout mode must be globally consistent across core services (see section 13).

Rules:

- `MULTI`: current behavior unchanged.
- `SINGLE`: one default tenant is used by all users; signup never creates tenants.

### 3.2 Default tenant identity

- Default tenant must use a generated NanoID key (same format as other tenant keys).
- Key must be stable for the deployment lifecycle.
- No hardcoded human-readable key such as `default` or `demo0001`.

Recommended resolution strategy (in order):

1. If `iqkv.tenancy.default-tenant-key` is configured, use it (must pass NanoID validation).
2. Else, look up `public.tenants` by `is_default = true`.
3. Else, generate NanoID, create tenant row, mark `is_default = true`.

### 3.3 Signup behavior in single mode

On `POST /api/v1/iam/auth/signup` in `SINGLE` mode:

1. upsert user as today,
2. resolve default tenant key,
3. create active membership to default tenant if absent,
4. assign base authority (`MEMBER` by default),
5. emit user created and verification events as today,
6. return signup response with default tenant key and tenant status.

No tenant creation from signup.
No tenant owner assignment for end users.

### 3.4 Provisioning behavior in single mode

At IAM startup (or dedicated bootstrap job):

1. resolve/create default tenant,
2. ensure tenant status is `PROVISIONING` or `ACTIVE`,
3. if newly created, publish `tenant.created`,
4. same provisioning consumer flow sets `ACTIVE`,
5. retries remain supported.

This preserves event-driven provisioning and Billing bootstrap consistency.

### 3.5 Login and tenant discovery

- `signin` continues requiring tenant context, but UI/gateway should auto-supply default tenant key in single mode.
- `users/tenants` returns exactly one active membership for typical single-mode users.

## 4. Data Model Changes

## 4.1 IAM `public.tenants`

Add columns:

- `is_default BOOLEAN NOT NULL DEFAULT false`
- `tenant_mode_origin VARCHAR(16)` with values like `SINGLE_BOOTSTRAP`, `MULTI_SIGNUP`, `ADMIN_CREATED`

Constraints:

- Partial unique index ensuring at most one row with `is_default = true`.

## 4.2 Optional metadata table (alternative)

If tenant table change is undesirable, add `public.platform_settings`:

- key/value store with `default_tenant_key`.

Preferred option remains `is_default` on tenants for direct queryability.

## 5. IAM Service Changes

### 5.1 Configuration

Extend `TenancyConfigurationProperties` with:

- `mode` enum (`MULTI`, `SINGLE`)
- optional `defaultTenantKey`
- optional `defaultTenantName` (display name)
- optional `bootstrapSystemUserId` (audit field fallback)

### 5.2 Default tenant resolver/bootstrap components

Add:

- `DefaultTenantResolver` (resolve from config/db),
- `SingleTenantBootstrap` (idempotent startup initializer),
- `DefaultTenantProvisioningService` (create+publish when missing).

### 5.3 Signup flow refactor

Refactor `UserServiceImpl.registerUser(...)`:

- branch by tenancy mode:
  - `MULTI`: existing logic.
  - `SINGLE`: skip tenant creation and owner authority grant; attach to default tenant.

DTO/API adjustment:

- make `tenantName` optional at DTO level,
- enforce `tenantName` required only in `MULTI` mode,
- ignore/reject tenantName in `SINGLE` mode by policy:
  - recommended: ignore input and log debug for backward compatibility.

### 5.4 Membership authority policy in single mode

- new users receive `MEMBER`.
- optional operational admin can be a pre-created system account (not customer-facing owner).
- endpoint authorization should avoid hard dependency on `TENANT_OWNER` in single mode for core self-service flows.

### 5.5 Mapper/repository updates

`TenantMapper` additions:

- `findDefaultTenant()`
- `markDefaultTenant(tenantKey)`
- `insertDefaultIfAbsent(...)` (or compose via existing insert + update).

Liquibase:

- new changeset for `is_default` column + partial unique index.
- data backfill for environments that already have one tenant in single mode.

### 5.6 Strategy-pattern implementation blueprint (recommended)

Use strategy pattern as the primary implementation approach to avoid scattering mode checks.

Core contracts:

- `SignupStrategy`
  - `SignupResult registerUser(RegisterUserRequest request)`
- `TenantBootstrapStrategy`
  - `void bootstrapOnStartup()`
- optional `TenancyAuthorizationStrategy`
  - for mode-specific authority policy decisions where needed.

Implementations:

- `MultiTenantSignupStrategy`
  - contains current `UserServiceImpl.registerUser(...)` behavior (create tenant + owner authority).
- `SingleTenantSignupStrategy`
  - resolve default tenant + create membership + grant baseline authority (`MEMBER`).
- `MultiTenantBootstrapStrategy`
  - no-op (or existing operational bootstrap logic).
- `SingleTenantBootstrapStrategy`
  - idempotently ensure default tenant exists and is provisioned.

Resolver/factory:

- `TenancyModeStrategyResolver` maps `iqkv.tenancy.mode` to concrete strategy beans.
- Fail fast on unknown mode during startup.

Service wiring:

- `UserServiceImpl` delegates registration to `SignupStrategy` only.
- startup hook (`ApplicationRunner` or equivalent) delegates to `TenantBootstrapStrategy`.
- keep shared helper services (`DefaultTenantResolver`, mapper services, event publisher) mode-agnostic.

Pseudocode:

```java
public SignupResponse registerUser(RegisterUserRequest request) {
  return signupStrategy.registerUser(request).toResponse();
}
```

```java
@EventListener(ApplicationReadyEvent.class)
public void onReady() {
  tenantBootstrapStrategy.bootstrapOnStartup();
}
```

Benefits:

- clear separation of multi vs single behavior,
- low regression risk for existing multi-tenant path,
- straightforward unit testing per strategy,
- easier future extension (for example managed-enterprise mode).

Test shape for strategies:

- `MultiTenantSignupStrategyTest` verifies tenant creation + owner authority.
- `SingleTenantSignupStrategyTest` verifies no tenant creation + default tenant membership.
- `SingleTenantBootstrapStrategyTest` verifies idempotent default tenant bootstrap.

## 6. Billing Service Changes

### 6.1 Tenant event handling

Update `TenantEventConsumer.handleTenantCreated(...)`:

- remove hard failure on missing `ownerEmail`,
- derive billing contact using fallback chain:
  1. `ownerEmail` from event (multi-mode),
  2. configured default billing email (single-mode),
  3. nullable email (if Stripe allows for customer).

### 6.2 Payment gateway client compatibility

`PaymentGatewayClient.createCustomer(name, email)` should allow null/blank email path safely.

### 6.3 Billing settings bootstrap defaults

For single mode bootstrap tenant:

- create one `billing_settings` row as usual,
- `profile_owner_id` remains nullable,
- no owner coupling required.

### 6.4 Subscription model with pre-provisioned plans (updated)

Plans are platform-managed and pre-provisioned before user registration starts.

Rules:

- no end-user plan CRUD in IAM/Gateway/Billing UIs,
- platform admins manage catalog lifecycle,
- subscription assignment allowed to valid subject scope only.

Subject scope:

- `MULTI` mode: subscription subject is tenant (tenant owner/admin operates it).
- `SINGLE` mode: subscription subject is user (each user has own billing settings and subscription).

Recommended schema direction in Billing:

- `plan_catalog` (platform-owned)
  - `plan_code`, `display_name`, `billing_period`, `price_minor`, `currency`, `feature_set`, `active`, `scope`.
- `subscriptions`
  - add `subject_type` (`TENANT` or `USER`)
  - add `subject_key` (tenantKey or userId)
  - enforce one active subscription per subject.
- `billing_settings` remains tenant-level for tenant-subject flows.
- add `user_billing_settings` for user-subject flows in single mode.

Service policy contracts:

- `SubscriptionSubjectResolver` (resolve tenant vs user subject from platform mode),
- `PlanEligibilityPolicy` (validate selected plan supports subject scope),
- `EntitlementEvaluator` (validate active subscription by subject).

## 7. Event Contract Adjustments

`tenant.created` payload should treat owner fields as optional:

- `ownerEmail?: string`
- `ownerFirstName?: string`

Contract versioning:

- backward compatible if consumers stop requiring owner fields.
- document owner fields as "present in multi-mode, optional in single-mode."

Additional requirement:

- subscription events must include `subject_type` and `subject_key` so downstream consumers can evaluate entitlements consistently.

## 8. API Behavior Matrix

- `POST /auth/signup`
  - MULTI: creates tenant + user + owner membership.
  - SINGLE: creates user + default-tenant membership.
- `POST /auth/signin`
  - both modes require tenant context; single mode uses default tenant.
- `GET /tenants/{tenantKey}`
  - remains available for operational/admin flows.
- Invitation endpoints
  - optional in single mode; can be disabled by feature flag if product chooses pure self-signup.

Billing-facing behavior:

- plan catalog endpoints are platfrom admin-only (or internal-only).
- end-user endpoints allow selecting from existing plans only.
- no endpoint for end-user plan creation/update/deletion.

## 9. Non-Functional Requirements

- Idempotent bootstrap on restarts and concurrent replicas.
- No duplicate default tenant creation under race.
- Audit fields must remain populated (`created_by`, `updated_by`) using system user identity when needed.
- Multi-mode behavior must remain unchanged.

## 10. Migration Plan

1. Deploy IAM DB migration (`is_default` + index).
2. Deploy IAM with resolver/bootstrap and mode-aware signup.
3. Mark existing target tenant as default in single-mode environments.
4. Deploy Billing with optional ownerEmail handling.
5. Update gateway/UI capability flags to hide org selection and auto-set tenant context.

Rollback:

- set mode back to `MULTI`; existing tenant data remains valid.
- no schema rollback required for `is_default`.

## 11. Test Plan

### 11.1 IAM

- Signup in `SINGLE` does not create tenant rows.
- Signup in `SINGLE` creates membership to default tenant.
- Signup in `MULTI` behavior unchanged.
- Startup bootstrap creates exactly one default tenant and publishes one `tenant.created`.
- Repeated startup does not duplicate tenant or events.

### 11.2 Billing

- `tenant.created` with missing owner email does not fail.
- Billing settings created for bootstrap default tenant.
- Stripe customer creation works with fallback/nullable email strategy.
- Plan catalog is read-only for non-operator users.
- In `MULTI`, subscription stored/evaluated with `subject_type=TENANT`.
- In `SINGLE`, subscription stored/evaluated with `subject_type=USER`.

### 11.3 End-to-end

- Fresh single-mode environment: first user signup -> membership active -> sign in succeeds with default tenant key.
- Existing single-mode environment with precreated default tenant: signup auto-joins same tenant key.
- Entitlement checks in gateway/billing align with subject scope for the active mode.

## 12. Open Decisions

1. Should invitations be disabled entirely in single mode, or remain optional?
2. Should billing require a configured default billing email in single mode?
3. Should single-mode users ever receive `ADMIN`, or only `MEMBER` plus platform-level super-admin outside tenant authortities?

## 13. Global Platform Rollout Mode Contract (AIO requirement)

Core rule:

- IAM, Gateway, and Billing must run in the same platform rollout mode at all times.
- Mixed core runtime modes are invalid and must fail readiness.

Configuration:

- Introduce shared key: `iqkv.platform.rollout-mode = MULTI_TENANT | SINGLE_TENANT`.
- `iqkv.tenancy.mode` must be derived from (or validated against) this value.

Operational model:

1. One shared deployment source (for example Helm umbrella values) sets rollout mode once.
2. Each service validates rollout mode and required mode-specific config at startup.
3. Each service exposes mode via health/info metadata.
4. Readiness returns `DOWN` when:
   - rollout mode missing/invalid,
   - mode mismatch detected against platform contract source,
   - required mode-specific dependencies are absent.

Recommended handshake:

- IAM publishes canonical platform capabilities (including rollout mode).
- Gateway and Billing verify that local configured mode equals canonical mode.
- If mismatch, service stays non-ready and logs explicit mismatch diagnostics.

Safety requirements:

- rollout mode changes are controlled migrations, not ad-hoc runtime toggles,
- CI/CD pipeline validates all core charts/services share the same rollout mode before deployment.

## 14. Strict Restrictions, Gates, Components, and Authorities (based on current implementations)

This section defines enforceable guardrails aligned with the currently implemented code in IAM and Billing, plus required Gateway behavior.

### 14.1 Platform-level strict restrictions

- **R1: Single global runtime mode**
  - all core services must run the same `iqkv.platform.rollout-mode`,
  - mixed modes are a hard deployment error.
- **R2: No implicit fallback mode**
  - if rollout mode is missing, malformed, or unsupported, service must fail startup/readiness.
- **R3: Contract-before-traffic**
  - no core service should become ready until rollout mode contract validation passes.
- **R4: Controlled mode transitions only**
  - mode switch allowed only via migration procedure (never by ad-hoc env update on a running environment).

### 14.2 Platform-level gates

- **G1: CI/CD gate**
  - block deployment if IAM/Gateway/Billing manifests contain different rollout modes.
- **G2: Startup validation gate**
  - each service validates required mode-dependent config.
- **G3: Readiness gate**
  - readiness `DOWN` on mode mismatch, invalid mode, or missing mode-specific dependencies.
- **G4: Event contract gate**
  - reject/park malformed lifecycle events missing mandatory fields for their event type.

### 14.3 Core governance components

- **C1: `PlatformModeConfiguration`**
  - typed configuration object that parses and validates rollout mode.
- **C2: `PlatformModeValidator`**
  - shared logic for startup validation and fail-fast behavior.
- **C3: `PlatformCapabilitiesProvider` (IAM canonical)**
  - publishes canonical mode and capabilities for other services.
- **C4: `PlatformCapabilitiesConsumer` (Gateway/Billing)**
  - verifies local mode equals canonical mode.
- **C5: `ModeAwareHealthIndicator`**
  - exposes explicit diagnostics for mode mismatch.

### 14.4 IAM strict restrictions and gates

Current code facts used:

- tenant context is enforced through `TenantExtractionFilter`,
- signup currently bypasses tenant header checks,
- authority checks already enforce `TENANT_OWNER` for tenant lifecycle endpoints.

Required strict restrictions:

- **IAM-R1:** `signup` path must be strategy-driven by mode (no ad-hoc conditional spread).
- **IAM-R2:** in `SINGLE`, signup must not create tenant rows.
- **IAM-R3:** in `MULTI`, signup must require tenant creation semantics.
- **IAM-R4:** only one default tenant may exist when mode is `SINGLE`.
- **IAM-R5:** tenant owner authority must not be auto-assigned to every single-mode user.

Required IAM gates:

- **IAM-G1: DTO gate**
  - `tenantName` required in `MULTI`, ignored/rejected in `SINGLE` by explicit validation policy.
- **IAM-G2: Bootstrap idempotency gate**
  - repeated starts cannot duplicate default tenant or duplicate provisioning events.
- **IAM-G3: Authority gate**
  - endpoints requiring `TENANT_OWNER` remain inaccessible to plain `MEMBER` in single mode unless explicitly remapped by policy.
- **IAM-G4: Tenancy header gate**
  - for protected endpoints, tenant must be resolvable from `X-Tenant-ID` or JWT claim.

Required IAM components:

- `TenancyModeStrategyResolver`
- `SignupStrategy` + `MultiTenantSignupStrategy` + `SingleTenantSignupStrategy`
- `TenantBootstrapStrategy` + mode implementations
- `DefaultTenantResolver`
- `SingleTenantBootstrap`

### 14.5 Gateway strict restrictions and gates

Current architecture facts used:

- gateway validates JWT and resolves tenant context for routed requests.

Required strict restrictions:

- **GW-R1:** gateway mode must equal canonical platform mode before accepting business traffic.
- **GW-R2:** gateway cannot route requests using a mode-dependent behavior inconsistent with IAM.
- **GW-R3:** entitlement scope checks must follow active mode (`TENANT` vs `USER` subject).

Required Gateway gates:

- **GW-G1: Mode-consistency gate**
  - readiness `DOWN` when canonical mode and local mode differ.
- **GW-G2: Tenant-context gate**
  - in single mode, allow default tenant auto-resolution path; in multi mode, preserve tenant selection flow.
- **GW-G3: Authorization passthrough gate**
  - do not widen authorities in gateway; enforce least privilege from JWT claims.

Required Gateway components:

- `PlatformModeGuardFilter`
- `TenantContextResolutionPolicy`
- `EntitlementSubjectResolver`

### 14.6 Billing strict restrictions and gates

Current code facts used:

- `TenantEventConsumer` consumes tenant lifecycle events,
- current implementation hard-fails `TENANT_CREATED` if `ownerEmail` is null,
- tenant context is enforced by Billing `TenantExtractionFilter`.

Required strict restrictions:

- **BIL-R1:** tenant lifecycle consumption must be owner-agnostic (owner fields optional).
- **BIL-R2:** plan catalog is platform admin-managed only (no user plan CRUD).
- **BIL-R3:** subscription records must include explicit subject scope.
- **BIL-R4:** billing mode behavior must be determined from rollout mode, not inferred ad hoc from payload shape.

Required Billing gates:

- **BIL-G1: Event validation gate**
  - accept `TENANT_CREATED` without owner fields; reject only truly invalid events.
- **BIL-G2: Scope gate**
  - enforce `subject_type=TENANT` in multi mode and `subject_type=USER` in single mode.
- **BIL-G3: Stripe sync gate**
  - allow null/derived billing contact path without consumer crash.
- **BIL-G4: Tenant-context gate**
  - protected API calls must include resolvable tenant context (header or JWT claim).

Required Billing components:

- `BillingContactResolver`
- `SubscriptionSubjectResolver`
- `PlanEligibilityPolicy`
- `EntitlementEvaluator`

### 14.7 Authorities and authority boundaries

Platform-level operational authorities:

- **`PLATFORM_ADMIN`** (recommended new authority, not tenant-scoped)
  - manages rollout mode migrations and plan catalog administration.

Tenant/user authorities (existing and mode-aware use):

- `TENANT_OWNER`, `PLATFORM_ADMIN`, `MEMBER` remain tenant-scoped IAM authorities.
- In `MULTI`, subscription operations may require `TENANT_OWNER`/`PLATFORM_ADMIN` for tenant-subject subscriptions.
- In `SINGLE`, end-user subscription actions are user-subject; tenant owner authority is not mandatory for every user.

Strict authority rules:

- **A1:** no tenant-scoped authority should implicitly grant platform-operator capabilities.
- **A2:** platform-operator authorities must not be minted from tenant membership authortities.
- **A3:** gateway and billing trust JWT authorities but must enforce subject-scope policy independently of authority naming.

### 14.8 Non-negotiable invariants

- Exactly one active runtime mode across IAM/Gateway/Billing.
- Exactly zero or one default tenant marker (never more than one).
- No cross-mode behavior drift at runtime.
- No entitlement evaluation without explicit subject scope.
- No user-managed plan catalog mutations.

## 15. Implementation Checklist (service owners, gates, acceptance criteria)

This checklist converts sections 3-14 into executable delivery items.

### 15.1 Platform rollout contract (shared/core)

- **Owner:** Platform/Core team
- **Tasks:**
  - define `iqkv.platform.rollout-mode` in shared deployment values,
  - add startup mode parser/validator library or duplicated strict validator in each service,
  - expose rollout mode in health/info metadata,
  - add deployment pipeline rule that all core services share identical mode.
- **Acceptance criteria:**
  - deployment blocked on mode mismatch across service manifests,
  - each service returns readiness `DOWN` on invalid/missing mode,
  - observability shows current mode per service instance.

### 15.2 IAM checklist

- **Owner:** IAM team
- **Tasks:**
  - implement `TenancyModeStrategyResolver`,
  - implement `SignupStrategy` (`MultiTenantSignupStrategy`, `SingleTenantSignupStrategy`),
  - implement `TenantBootstrapStrategy` + `SingleTenantBootstrap`,
  - implement `DefaultTenantResolver`,
  - extend tenancy config with mode/default-tenant settings,
  - make `tenantName` mode-aware at validation boundary,
  - add DB migration for `is_default` marker + uniqueness constraint.
- **Acceptance criteria:**
  - in `SINGLE`, signup does not create tenant and assigns membership to default tenant only,
  - in `MULTI`, signup retains current behavior and tests remain green,
  - repeated startup in `SINGLE` is idempotent (no duplicate default tenant/events),
  - default-tenant uniqueness enforced at DB level.

### 15.3 Gateway checklist

- **Owner:** Gateway team
- **Tasks:**
  - add `PlatformModeGuardFilter` for canonical mode consistency checks,
  - add mode-aware tenant context policy (`TenantContextResolutionPolicy`),
  - add subject resolver for entitlement evaluation (`EntitlementSubjectResolver`),
  - ensure gateway does not mutate/expand authorities from upstream JWT.
- **Acceptance criteria:**
  - readiness `DOWN` when local mode != canonical mode,
  - in `SINGLE`, default-tenant path works without organization chooser flow,
  - in `MULTI`, tenant selection flow remains unchanged,
  - authorization scope remains least-privilege and traceable.

### 15.4 Billing checklist

- **Owner:** Billing team
- **Tasks:**
  - update `TenantEventConsumer` to treat owner fields as optional,
  - add `BillingContactResolver` fallback chain,
  - add subscription subject model (`subject_type`, `subject_key`),
  - introduce platform admin-managed `plan_catalog`,
  - add `SubscriptionSubjectResolver`, `PlanEligibilityPolicy`, `EntitlementEvaluator`,
  - add `user_billing_settings` for single-mode user-subject billing.
- **Acceptance criteria:**
  - `TENANT_CREATED` without owner email is processed successfully,
  - in `MULTI`, subscriptions always evaluated by tenant subject,
  - in `SINGLE`, subscriptions always evaluated by user subject,
  - non-platform-admins cannot mutate plan catalog.

### 15.5 Security/authority checklist

- **Owner:** IAM + Security architecture
- **Tasks:**
  - define platform-level authorities (`PLATFORM_ADMIN`),
  - enforce strict separation between platform authorities and tenant authortities,
  - map endpoint-level authority policy for each service and mode,
  - add audit logging for authority-sensitive operations.
- **Acceptance criteria:**
  - tenant authortities cannot perform platform-operator actions,
  - platform-operator actions are auditable with actor identity and timestamp,
  - authority model documented and tested across both rollout modes.

### 15.6 Data migration checklist

- **Owner:** IAM + Billing DB owners
- **Tasks:**
  - IAM migration: `is_default` and unique default marker guard,
  - Billing migration: subject-scope fields and constraints,
  - Billing migration: pre-provisioned plan catalog tables + seed strategy,
  - Billing migration: `user_billing_settings` table.
- **Acceptance criteria:**
  - migrations are forward-only and idempotent for repeated deploy attempts,
  - rollback path documented at application/config level (no destructive DB rollback required),
  - data backfill scripts verified on staging snapshot.

### 15.7 Integration and release gates

- **Owner:** QA + Platform Release team
- **Tasks:**
  - add contract tests for platform mode consistency,
  - add end-to-end tests for signup/signin/entitlement in both modes,
  - add event contract tests for optional owner fields and subscription subject fields,
  - add chaos/restart test for single-mode bootstrap idempotency.
- **Acceptance criteria:**
  - zero mixed-mode startup allowed in test environments,
  - both modes pass full auth + billing + entitlement E2E paths,
  - event consumers tolerate optional legacy/new payload fields.

### 15.8 Suggested delivery sequence

1. shared rollout-mode contract + CI/CD gate,
2. IAM strategy + default tenant bootstrap + migration,
3. Billing owner-agnostic event handling,
4. Billing subject-scope and catalog model,
5. Gateway mode guard + subject resolver,
6. full E2E validation in both modes,
7. production rollout with controlled migration runbook.

### 15.9 Definition of done (overall)

- All three core services enforce one shared rollout mode at startup/readiness.
- Single-mode signup auto-joins default tenant and remains ownerless for end users.
- Billing operates correctly for both subject scopes with pre-provisioned plans only.
- Gateway/IAM/Billing entitlement behavior is mode-consistent and test-covered.
- Authority model is explicitly separated between platform operations and tenant operations.

## 16. Package placement (aligned with current design approach)

This section defines where new runtime-mode and tenancy components should live to stay consistent with existing code organization.

### 16.1 Primary rule

- Mode and tenant runtime behavior belongs in the `tenancy` package by default.
- Business-domain orchestration remains in existing domain packages (`user`, `tenant`, `subscription`, `settings`).
- `infrastructure.config` remains wiring/config only (avoid business branching there).

### 16.2 IAM package placement

Place in `tenancy`:

- `TenancyMode` enum,
- extended `TenancyConfigurationProperties`,
- `TenancyModeStrategyResolver`,
- `SignupStrategy` + `MultiTenantSignupStrategy` + `SingleTenantSignupStrategy`,
- `TenantBootstrapStrategy` + mode-specific implementations,
- `DefaultTenantResolver`,
- mode validation helpers (`PlatformModeValidator`) when focused on tenancy/runtime consistency.

Keep in existing packages:

- `user`: API/resource + orchestration entrypoints; delegates to tenancy strategies,
- `tenant`: tenant entity lifecycle and persistence details,
- `infrastructure.config`: bean registration, property binding, filter chain wiring.

### 16.3 Gateway package placement

Place mode and tenant-resolution policy classes in `tenancy` (or equivalent gateway tenancy module):

- `PlatformModeGuardFilter`,
- `TenantContextResolutionPolicy`,
- `EntitlementSubjectResolver`.

Keep token parsing/security plumbing in security package; keep rollout-mode branching out of raw config classes.

### 16.4 Billing package placement

Place rollout/subject-scope resolution policy in `tenancy` or `subscription.policy`:

- `SubscriptionSubjectResolver`,
- mode-aware `PlanEligibilityPolicy`,
- mode guard utilities.

Keep in billing domain packages:

- `settings`: `BillingContactResolver` and billing profile orchestration,
- `subscription`: persistence and lifecycle logic,
- `infrastructure.messaging`: event transport adapters only.

### 16.5 Optional split for future growth

If cross-service rollout contract becomes broader than tenancy, introduce `platform` package for:

- rollout-mode contract DTOs,
- capability handshake client/provider,
- shared platform-level validators.

Until then, defaulting to `tenancy` keeps implementation consistent and minimally invasive.
