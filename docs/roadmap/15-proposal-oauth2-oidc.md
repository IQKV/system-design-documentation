# Proposal: OAuth2 / OpenID Connect (OIDC) Authentication

## Implementation Checklist

### Database & Migrations

- [ ] Add Liquibase changelog for `user_identities` table
- [ ] Add Liquibase changelog for `tenant_oidc_providers` table
- [ ] Verify DB constraints and indexes

### IAM Service Dependencies

- [ ] Add `spring-boot-starter-oauth2-client` to `foundation-iam-service/pom.xml`
- [ ] Verify dependencies with `mvn dependency:tree`

### Configuration

- [ ] Add OIDC config to `application.yml` (static providers, base-url, post-login-redirect-uri, auto-provision-users, encryption-key, state-ttl, auto-link.enabled)
- [ ] Add environment variable defaults

### Core Implementation

- [ ] Create `oauth2` package structure
- [ ] Implement `OidcIdentity` record
- [ ] Implement `OidcState` record
- [ ] Implement `OidcStateJwtService` (sign/verify state JWT)
- [ ] Implement `OidcStateStore` (Redis-based storage)
- [ ] Implement `AesGcmEncryptionService`
- [ ] Implement `OidcProvisioningException`
- [ ] Implement `OidcDtos` (OidcExchangeRequest, EnabledProvidersResponse, LinkedIdentityResponse)
- [ ] Implement `UserIdentityMapper` (MyBatis)
- [ ] Implement `TenantOidcProviderMapper` (MyBatis)
- [ ] Implement `OidcUserProvisioningService` interface & implementation (delegates to existing `SignupStrategy` for tenant provisioning)
- [ ] Implement `DynamicClientRegistrationRepository`
- [ ] Implement `GitHubEmailFetcher` (for GitHub non-OIDC handling)
- [ ] Implement `OidcAuthorizationRestResource` (all public endpoints)
- [ ] Implement `TenantSsoRestResource`, `TenantSsoService`
- [ ] Update `SecurityConfig` (permit OIDC endpoints, enable oauth2Client)
- [ ] Update `JwtAuthenticationFilter.shouldNotFilter()`

### Gateway Changes

- [ ] Add `/api/v1/iam/auth/oauth2/**` to gateway `public-paths`
- [ ] Exclude OIDC endpoints from tenant extraction filter

### Tests

- [ ] Unit tests for `OidcUserProvisioningServiceImpl`
- [ ] Unit tests for `OidcStateJwtService`
- [ ] Unit tests for `AesGcmEncryptionService`
- [ ] Unit tests for `DynamicClientRegistrationRepository`
- [ ] Integration tests for `OidcAuthorizationRestResource`

### Documentation

- [ ] Update API docs (`docs/api/`)
- [ ] Update README.md
- [ ] Update README.template.md

### Security & Audit

- [ ] Add audit logging for auto-linking events
- [ ] Implement admin notifications for auto-linking
- [ ] Implement admin endpoints for unmerge & audit history

## Overview

This document proposes adding OAuth2/OIDC as a parallel authentication path alongside
the existing email-and-password login. The goal is to let users sign in via external
Identity Providers (IdPs) such as Google, GitHub, Microsoft, and generic OIDC providers,
while keeping the internal JWT contract — and all downstream services — completely
unchanged.

The key principle is **"IAM brokers the identity"** — the IAM service acts as the
OAuth2 client, exchanges an OIDC identity for a locally issued RS256 JWT using the same
`JwtTokenGenerator` that password-based signin uses today. The gateway, billing, audit,
CMS, and all future services continue validating IAM-issued tokens via the existing
JWKS endpoint. No downstream service requires any change.

## Goals

- Provide social login (Google, GitHub, Microsoft) and generic OIDC provider support
  without breaking or replacing password-based signin, magic-link, or invitation flows.
- Preserve the full internal JWT claim structure (`userId`, `tenant_id`, `authorities`,
  `plan_code`, etc.) that every downstream service and the gateway depend on.
- Support account linking — an existing password-based account can attach one or more
  OIDC identities; subsequent OIDC logins authenticate the same user record.
- Support auto-provisioning — first-time OIDC signin creates a user account and
  (in MULTI_TENANT mode) triggers the existing tenant provisioning pipeline.
- Support per-tenant SSO configuration — a TENANT_OWNER can register a custom OIDC
  provider scoped to their tenant (enterprise SSO use case).
- Keep the `JwtAuthenticationFilter` (JTI denylist + global signout) applicable to all
  tokens regardless of how they were obtained.
- Zero changes to `foundation-billing-service`, `foundation-audit-service`,
  `foundation-cms-service`, or any future downstream service.

## Non-Goals

- SAML 2.0 support (deferred — the roadmap item is "SSO / SAML adapter, extension not core").
- Acting as a full OAuth2 Authorization Server (no `authorization_code` grant issued
  by the IAM service to third-party apps).
- Replacing the existing password credential flow — OIDC is additive, not a migration.

## Current Architecture Snapshot

```
Browser / Client
      │  POST /api/v1/iam/auth/signin  (email + password + X-Tenant-ID)
      │
      ▼
foundation-gateway-service
      │  HeaderSanitizationFilter (order -190) — strips X-User-*, X-Plan-Code
      │  SecurityWebFilterChain — JWKS via IAM /.well-known/jwks.json
      │  JwtContextPropagationFilter (order -100) — adds X-User-*, X-Tenant-ID downstream
      │
      ▼  (proxied to IAM, /api/v1/iam/** routes)
foundation-iam-service
      │  JwtAuthenticationFilter — JTI denylist + global signout check
      │  SecurityFilterChain — NimbusJwtDecoder (RSA public key)
      │  AuthenticationServiceImpl.signIn() — BCrypt verify, JwtTokenGenerator.generateAccessToken()
      │
      └─► RS256 JWT (iss: foundation-iam-service, claims: userId, tenant_id, authorities, plan_code, …)
              │
              ▼  subsequent API calls  (Authorization: Bearer <token>)
      foundation-gateway-service validates via JWKS
              │
              ▼  X-User-ID, X-Tenant-ID, X-User-Authorities, X-Plan-Code headers
      foundation-billing-service / foundation-audit-service / foundation-cms-service
```

The entire downstream trust model rests on the IAM-issued JWT. OIDC tokens from Google
or GitHub carry none of the platform-specific claims required downstream. The brokering
approach is therefore the only design that avoids touching every downstream service.

## Finalized Architectural Decisions

This section records all finalized architecture decisions before implementation begins:

| Decision ID | Decision                                                                                                                                                         | Rationale                                                                                           |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| DEC-001     | User email uniqueness enforced at DB and app layer                                                                                                               | Already exists (unique constraint on `users.email`), satisfies requirement for global user accounts |
| DEC-002     | Allow multiple OIDC providers linked to same user                                                                                                                | Standard user-friendly pattern (e.g., Google + GitHub)                                              |
| DEC-003     | Auto-link when `emailVerified=true` from IdP and user exists                                                                                                     | Trust verified emails as proof of account ownership                                                 |
| DEC-004     | State JWT TTL: 10 minutes default, configurable (max 15 min)                                                                                                     | Balances security and usability                                                                     |
| DEC-005     | Server-side PKCE with short-lived Redis storage                                                                                                                  | Secure and aligns with existing JTI denylist pattern                                                |
| DEC-006     | Tenant resolution (no tenantKey provided):<br>- MULTI_TENANT: User becomes MEMBER in "platform" tenant<br>- SINGLE_TENANT: Join default tenant with TENANT_OWNER | Aligns with existing `SignupStrategy` behavior                                                      |
| DEC-007     | Gateway: Exclude `/api/v1/iam/auth/oauth2/**` from tenant extraction                                                                                             | OIDC flows don't use X-Tenant-ID header                                                             |
| DEC-008     | GitHub: Reject signin if no verified email found                                                                                                                 | Email is required for global user account                                                           |
| DEC-009     | Rate limiting: Apply to `/authorize`, `/callback`, and `/exchange`                                                                                               | Prevent abuse across all OIDC endpoints                                                             |
| DEC-010     | Unlink: Require re-authentication before unlinking                                                                                                               | Prevent accidental lockout                                                                          |
| DEC-011     | Email conflict handling: Soft-merge + admin notification + manual unmerge                                                                                        | Mitigate account takeover risk from reused corporate emails                                         |

## Proposed Architecture

### High-Level Flow — Browser-Based OIDC Login

```
Browser
  │  1. GET /api/v1/iam/auth/oauth2/authorize?provider=google&tenantKey=abc12345
  │
  ▼
foundation-iam-service (Spring OAuth2 Client)
  │  2. Builds authorization URL with PKCE + signed state JWT, stores (state_jti → code_verifier) in Redis, redirects to IdP
  │
  ▼
Google / GitHub / custom IdP
  │  3. User consents, IdP redirects to:
  │     GET /api/v1/iam/auth/oauth2/callback?code=…&state=…
  │
  ▼
foundation-iam-service
  │  4. Verifies state JWT (signature, expiry, nonce), retrieves code_verifier from Redis
  │  5. Exchanges code + code_verifier for OIDC ID token + access token
  │  6. Validates ID token (nonce, iss, aud, exp)
  │  7. Extracts email, sub, name → normalizes into OidcIdentity
  │  8. OidcUserProvisioningService:
  │       a. Find or create UserIdentity (provider + sub → user_id)
  │       b. Find or create User (by email); emailVerified = true
  │       c. Resolve tenant context
  │       d. Resolve authorities from TenantMembership
  │       e. JwtTokenGenerator.generateAccessToken()  ← same as password flow
  │  9. Redirect to post-login-redirect-uri#access_token=…&refresh_token=…
```

**SPA / headless variant** (client-side PKCE — no server redirect):

```
Browser / SPA
  │  POST /api/v1/iam/auth/oauth2/exchange
  │  { provider, code, codeVerifier, redirectUri, tenantKey }
  │
  ▼
foundation-iam-service  (steps 4–8 above, no redirect at end)
  │
  └─► TokenResponse { accessToken, refreshToken, expiresIn, tokenType }
      — identical shape to POST /auth/signin response
```

Both variants produce a standard `TokenResponse`. Token refresh, signout, global
signout-all, and tenant exchange all work without modification.

### Affected Modules

| Module                       | Change                                                                                                         |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `foundation-iam-service`     | Primary — new `oauth2` package, DB tables, `spring-boot-starter-oauth2-client` dep, `SecurityConfig` additions |
| `foundation-gateway-service` | Minor — add `/api/v1/iam/auth/oauth2/**` to `public-paths` and exclude from tenant extraction                  |
| Downstream services          | **No change** — continue validating IAM-issued JWTs via JWKS as today                                          |
| `foundation-ui-app`          | Social login buttons, account-linking UI in Profile & Security, SSO config panel for TENANT_OWNER              |

## Detailed Design

### 1. New Dependency — `spring-boot-starter-oauth2-client`

Add to `foundation-iam-service/pom.xml` only:

```xml
<!-- OAuth2 Client (OIDC / social login) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

`spring-boot-starter-oauth2-resource-server` (already present) pulls in
`spring-security-oauth2-jose` (Nimbus). The client starter adds
`spring-security-oauth2-client` on top. No explicit version — managed by `boot-parent-pom`.

The gateway and all downstream services require **no new dependencies**.

### 2. Database Schema

Two new tables are added via Liquibase changesets in the IAM service system changelog.

**`user_identities`** — links external IdP identities to internal user accounts:

```sql
-- db/changelog/system/20260630000000-oauth2-user-identities.xml
CREATE TABLE user_identities (
    id            UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider      VARCHAR(64)  NOT NULL,   -- "google" | "github" | "microsoft" | "oidc:<tenantKey>"
    provider_sub  VARCHAR(255) NOT NULL,   -- stable subject identifier from the IdP
    email         VARCHAR(255),            -- last known email (display only, not used for lookup)
    display_name  VARCHAR(255),
    avatar_url    VARCHAR(512),
    linked_at     TIMESTAMPTZ  NOT NULL DEFAULT now(),
    last_used_at  TIMESTAMPTZ,
    UNIQUE (provider, provider_sub)
);
CREATE INDEX idx_user_identities_user_id ON user_identities(user_id);
```

**`tenant_oidc_providers`** — per-tenant enterprise SSO configuration:

```sql
-- db/changelog/system/20260630100000-tenant-oidc-providers.xml
CREATE TABLE tenant_oidc_providers (
    id             UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id      UUID         NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider_key   VARCHAR(64)  NOT NULL UNIQUE, -- "oidc:<tenantKey>", matches user_identities.provider
    display_name   VARCHAR(128) NOT NULL,
    issuer_uri     VARCHAR(512) NOT NULL,
    client_id      VARCHAR(255) NOT NULL,
    client_secret  TEXT         NOT NULL,        -- AES-256-GCM encrypted at rest
    scopes         VARCHAR(255) NOT NULL DEFAULT 'openid profile email',
    enabled        BOOLEAN      NOT NULL DEFAULT true,
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_tenant_oidc_providers_tenant ON tenant_oidc_providers(tenant_id);
```

**Note:** User email uniqueness is already enforced at the database level with `UNIQUE (email)` on the `users` table.

### 3. OAuth2 Client Configuration — `application.yml`

Static built-in providers are opt-in via environment variables. A provider is silently
inactive when its `client-id` is blank, so deployments without credentials start cleanly.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${OAUTH2_GOOGLE_CLIENT_ID:}
            client-secret: ${OAUTH2_GOOGLE_CLIENT_SECRET:}
            scope: openid, profile, email
            redirect-uri: "{baseUrl}/api/v1/iam/auth/oauth2/callback/{registrationId}"
          github:
            client-id: ${OAUTH2_GITHUB_CLIENT_ID:}
            client-secret: ${OAUTH2_GITHUB_CLIENT_SECRET:}
            scope: read:user, user:email
            redirect-uri: "{baseUrl}/api/v1/iam/auth/oauth2/callback/{registrationId}"
          microsoft:
            client-id: ${OAUTH2_MICROSOFT_CLIENT_ID:}
            client-secret: ${OAUTH2_MICROSOFT_CLIENT_SECRET:}
            scope: openid, profile, email
            redirect-uri: "{baseUrl}/api/v1/iam/auth/oauth2/callback/{registrationId}"
            provider: microsoft
        provider:
          microsoft:
            issuer-uri: https://login.microsoftonline.com/${OAUTH2_MICROSOFT_TENANT_ID:common}/v2.0

iqkv:
  auth:
    oauth2:
      # Comma-separated list of enabled providers — controls which buttons the UI renders
      enabled-providers: ${OAUTH2_ENABLED_PROVIDERS:google,github}
      # IAM public base URL for constructing redirect URIs
      base-url: ${OAUTH2_BASE_URL:http://localhost:8080}
      # Where the IAM service redirects after completing the browser-based OIDC exchange
      post-login-redirect-uri: ${OAUTH2_POST_LOGIN_REDIRECT_URI:http://localhost:5173/auth/callback}
      # When false, first-time OIDC users are rejected unless they already have an account
      auto-provision-users: ${OAUTH2_AUTO_PROVISION:true}
      # AES-256-GCM key for encrypting client secrets in tenant_oidc_providers
      encryption-key: ${OIDC_ENCRYPTION_KEY:}
      # TTL for state JWT and Redis-stored code_verifier (ISO-8601 duration)
      state-ttl: ${OAUTH2_STATE_TTL:PT10M}
      # Maximum allowed state TTL (ISO-8601 duration)
      state-ttl-max: ${OAUTH2_STATE_TTL_MAX:PT15M}
      # Enable/disable auto-linking of OIDC identities to existing users by verified email
      auto-link.enabled: ${OAUTH2_AUTO_LINK_ENABLED:true}
```

### 4. New Package Layout — `com.iqkv.foundation.iamservice.oauth2`

All OIDC code is isolated in a single new top-level package. The existing
`authentication` package is not modified.

```
oauth2/
  OidcAuthorizationRestResource.java       — public OIDC endpoints
  OidcUserProvisioningService.java         — interface
  OidcUserProvisioningServiceImpl.java
  OidcIdentity.java                        — normalized IdP identity record
  OidcStateJwtService.java                 — sign/verify state JWT using IAM RSA key
  OidcStateStore.java                      — Redis-based state/code_verifier storage
  DynamicClientRegistrationRepository.java — static + DB-backed provider lookup
  AesGcmEncryptionService.java             — client secret encryption at rest
  OidcProvisioningException.java           — maps to 403 in global exception handler
  dto/
    OidcDtos.java                          — OidcExchangeRequest, EnabledProvidersResponse
  mapper/
    UserIdentityMapper.java                — MyBatis interface
    TenantOidcProviderMapper.java          — MyBatis interface
  sso/
    TenantSsoRestResource.java             — TENANT_OWNER SSO config endpoints
    TenantSsoService.java
    TenantSsoServiceImpl.java
```

### 5. `OidcAuthorizationRestResource` — New Endpoints

All public OIDC endpoints are `permitAll()` in `SecurityConfig` (same treatment as
`/auth/signin`). Authenticated account-linking endpoints require a valid bearer token.

```java
@RestController
@RequestMapping("/api/v1/iam/auth/oauth2")
public class OidcAuthorizationRestResource {

  // ── Public flows ─────────────────────────────────────────────────────────

  // Initiates the browser redirect flow. Builds the IdP authorization URL with
  // PKCE (S256) and a signed state JWT embedding tenantKey + nonce + redirectUri.
  @GetMapping("/authorize")
  public void authorize(@RequestParam String provider,
                        @RequestParam(required = false) String tenantKey,
                        HttpServletRequest request,
                        HttpServletResponse response) throws IOException { … }

  // IdP redirects here after user consent. Verifies state JWT, exchanges code,
  // calls OidcUserProvisioningService, then redirects to post-login-redirect-uri.
  @GetMapping("/callback")
  public void callback(@RequestParam String code,
                       @RequestParam String state,
                       HttpServletResponse response) throws IOException { … }

  // SPA/headless variant: client completes PKCE itself and sends the code here.
  // Returns TokenResponse — same shape as POST /auth/signin.
  @PostMapping("/exchange")
  public ResponseEntity<AuthenticationDtos.TokenResponse> exchange(
      @Valid @RequestBody OidcDtos.OidcExchangeRequest request) { … }

  // Returns enabled provider list for UI button rendering.
  @GetMapping("/providers")
  public ResponseEntity<OidcDtos.EnabledProvidersResponse> providers() { … }

  // ── Authenticated account-linking flows ──────────────────────────────────

  // Initiates linking an additional IdP identity to the current account.
  @GetMapping("/link/{provider}")
  @PreAuthorize("isAuthenticated()")
  public void initiateLink(@PathVariable String provider,
                           @AuthenticationPrincipal Jwt jwt,
                           HttpServletResponse response) throws IOException { … }

  // Callback for the linking flow (separate path to distinguish from login callback).
  @GetMapping("/link/callback")
  @PreAuthorize("isAuthenticated()")
  public void linkCallback(@RequestParam String code,
                           @RequestParam String state,
                           @AuthenticationPrincipal Jwt jwt,
                           HttpServletResponse response) throws IOException { … }

  // Removes a linked IdP identity. Enforces at-least-one-credential guard.
  @DeleteMapping("/link/{provider}")
  @PreAuthorize("isAuthenticated()")
  public ResponseEntity<Void> unlink(@PathVariable String provider,
                                     @AuthenticationPrincipal Jwt jwt) { … }

  // Lists all linked identities for the current user.
  @GetMapping("/identities")
  @PreAuthorize("isAuthenticated()")
  public ResponseEntity<List<OidcDtos.LinkedIdentityResponse>> identities(
      @AuthenticationPrincipal Jwt jwt) { … }
}
```

**Full endpoint inventory:**

| Method   | Path                                      | Auth   | Description                                  |
| -------- | ----------------------------------------- | ------ | -------------------------------------------- |
| `GET`    | `/api/v1/iam/auth/oauth2/authorize`       | public | Initiate browser-based OIDC flow             |
| `GET`    | `/api/v1/iam/auth/oauth2/callback`        | public | IdP redirect receiver; issues IAM JWT        |
| `POST`   | `/api/v1/iam/auth/oauth2/exchange`        | public | SPA/PKCE headless token exchange             |
| `GET`    | `/api/v1/iam/auth/oauth2/providers`       | public | List enabled providers (for UI)              |
| `GET`    | `/api/v1/iam/auth/oauth2/link/{provider}` | bearer | Initiate account linking                     |
| `GET`    | `/api/v1/iam/auth/oauth2/link/callback`   | bearer | Linking callback                             |
| `DELETE` | `/api/v1/iam/auth/oauth2/link/{provider}` | bearer | Unlink an OIDC identity                      |
| `GET`    | `/api/v1/iam/auth/oauth2/identities`      | bearer | List linked identities for current user      |
| `GET`    | `/api/v1/iam/tenants/sso`                 | bearer | Get tenant SSO config (TENANT_OWNER / ADMIN) |
| `PUT`    | `/api/v1/iam/tenants/sso`                 | bearer | Save / update tenant SSO config              |
| `DELETE` | `/api/v1/iam/tenants/sso`                 | bearer | Remove tenant SSO config                     |

### 6. `OidcUserProvisioningService` — Core Logic

This is the single point of truth for all identity federation decisions. It is the only
place that reads `user_identities` and the only place that decides whether to provision,
link, or reject an incoming OIDC identity.

**Architecture Note**: Tenant provisioning logic is delegated to existing `SignupStrategy`
implementations (`MultiTenantSignupStrategy` and `SingleTenantSignupStrategy`) to ensure
consistency across password-based and OIDC-based signups.

```java
public interface OidcUserProvisioningService {
  /**
   * Find or create a user for the given OIDC identity, resolve their tenant context,
   * and issue a full IAM JWT pair identical to the password-based signin result.
   *
   * @throws OidcProvisioningException on any condition that should surface as 4xx.
   */
  AuthenticationDtos.TokenResponse provisionAndIssueTokens(
      OidcIdentity identity,
      String tenantKey        // from state JWT or request body; null triggers auto-resolve
  );
}
```

**Provisioning decision tree:**

```
(provider, provider_sub) found in user_identities?
  ├── YES → load User, update last_used_at                              → token issuance
  └── NO  →
        email in users table AND emailVerified=true in IdP identity AND iqkv.auth.oauth2.auto-link.enabled=true?
          ├── YES → insert user_identities row (account linking); log audit event; notify admins → token issuance
          └── NO  →
                iqkv.auth.oauth2.auto-provision-users = true?
                  ├── YES →
                  │     create User (status=ACTIVE, emailVerified=true, passwordHash=null)
                  │     insert user_identities row
                  │     delegate to existing SignupStrategy for tenant provisioning (handles both MULTI_TENANT and SINGLE_TENANT modes)
                  │                                                      → token issuance
                  └── NO → throw OidcProvisioningException(PROVISIONING_DISABLED) → 403
```

**Token issuance (shared terminal path, calls existing code unchanged):**

```java
final TenantMembership membership = membershipMapper.findByUserAndTenant(userId, tenantKey);
final List<String> authorities    = authorityResolver.resolve(membership);
final String planCode             = tenantMapper.findActivePlanCode(tenantKey);

return new AuthenticationDtos.TokenResponse(
    jwtTokenGenerator.generateAccessToken(user, tenantKey, authorities, planCode),
    jwtTokenGenerator.generateRefreshToken(user, tenantKey),
    authProps.jwt().expiry().toSeconds()
);
```

The OIDC path never bypasses authority resolution, `plan_code` stamping, or any other
claim logic. The resulting token is structurally identical to one issued after password
authentication.

### 7. `OidcIdentity` — Normalized IdP Identity

Different providers expose user information in different shapes. This record normalizes
them before the provisioning service is invoked:

```java
public record OidcIdentity(
    String provider,       // "google" | "github" | "microsoft" | "oidc:<tenantKey>"
    String sub,            // IdP's stable subject identifier
    String email,          // null if scope excluded; validated before use
    boolean emailVerified,
    String firstName,
    String lastName,
    String avatarUrl,
    String rawIdToken      // kept for audit event only; never persisted
) {
  /** OIDC-compliant providers (Google, Microsoft, generic). */
  public static OidcIdentity fromOidcUser(OidcUser user, String provider) { … }

  /**
   * GitHub is not OIDC-compliant — no ID token. The caller must have already
   * fetched /user/emails to populate emailVerified.
   */
  public static OidcIdentity fromGitHubUser(OAuth2User user, String verifiedEmail) { … }
}
```

### 8. State and PKCE Correlation — Server-Side Storage

The OAuth2 `state` parameter must survive the browser round-trip to the IdP and back.
It carries the tenant key, a nonce, the post-login redirect URI, and the flow type
(login vs. link). `OidcState` is serialized to a **signed JWT** using the IAM RSA
private key, and the `code_verifier` is stored in Redis with the state JWT's JTI as key:

```java
public record OidcState(
    String jti,            // unique JWT ID for Redis lookup
    String nonce,          // random 32-byte hex; verified against OIDC ID token nonce claim
    String tenantKey,      // resolved at authorize time; recovered at callback
    String redirectUri,    // destination URI after successful login (browser flow only)
    String flowType,       // "login" | "link"
    String userId,         // populated only for "link" flows
    Instant expiresAt      // from iqkv.auth.oauth2.state-ttl (default 10 minutes)
) {}
```

The callback verifies the JWT signature and `expiresAt` before trusting any claim in
the state. PKCE (`code_challenge_method=S256`) is always required — `plain` is rejected.

### 9. `DynamicClientRegistrationRepository` — Per-Tenant Enterprise SSO

Enterprise tenants can register a custom OIDC provider (Okta, Azure AD, corporate
Keycloak, etc.) via the `PUT /api/v1/iam/tenants/sso` endpoint. These registrations are
stored in `tenant_oidc_providers` and loaded dynamically at request time:

```java
@Component
public class DynamicClientRegistrationRepository implements ClientRegistrationRepository {

  private final InMemoryClientRegistrationRepository staticRepo; // built-in providers
  private final TenantOidcProviderMapper providerMapper;
  private final AesGcmEncryptionService encryptionService;

  @Override
  public ClientRegistration findByRegistrationId(final String registrationId) {
    // Static providers (google, github, microsoft) checked first
    ClientRegistration reg = staticRepo.findByRegistrationId(registrationId);
    if (reg != null) return reg;

    // Dynamic tenant providers — registrationId format: "oidc:<tenantKey>"
    if (!registrationId.startsWith("oidc:")) return null;
    return providerMapper.findByProviderKey(registrationId)
        .filter(TenantOidcProvider::enabled)
        .map(p -> ClientRegistration.withRegistrationId(p.providerKey())
            .clientId(p.clientId())
            .clientSecret(encryptionService.decrypt(p.clientSecret()))
            .scope(p.scopes().split(","))
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .redirectUri("{baseUrl}/api/v1/iam/auth/oauth2/callback/{registrationId}")
            .issuerUri(p.issuerUri())
            .build())
        .orElse(null);
  }
}
```

`AesGcmEncryptionService` uses AES-256-GCM with a key from `iqkv.auth.oauth2.encryption-key`.
The decrypted secret is never logged or returned in API responses. The `TenantSsoRestResource`
returns a masked client secret (`"••••••••"`) on `GET /api/v1/iam/tenants/sso`.

### 10. `SecurityConfig` Changes — IAM Service

Two additions only. The rest of `SecurityConfig` is unchanged.

```java
// 1. Permit the four new public OIDC paths (same as existing signin/refresh permits)
.authorizeHttpRequests(auth -> auth
    // … all existing requestMatchers unchanged …
    .requestMatchers(HttpMethod.GET,  "/api/v1/iam/auth/oauth2/authorize").permitAll()
    .requestMatchers(HttpMethod.GET,  "/api/v1/iam/auth/oauth2/callback").permitAll()
    .requestMatchers(HttpMethod.POST, "/api/v1/iam/auth/oauth2/exchange").permitAll()
    .requestMatchers(HttpMethod.GET,  "/api/v1/iam/auth/oauth2/providers").permitAll()
    .anyRequest().authenticated()
)
// 2. Enable Spring OAuth2 client infrastructure (token fetching, PKCE support)
.oauth2Client(Customizer.withDefaults())
```

`JwtAuthenticationFilter.shouldNotFilter()` is extended with the same four paths so
the revocation check is skipped for requests that carry no bearer token.

No other class in the IAM service `SecurityConfig` is modified.

### 11. Gateway Changes

Two additive changes:

1. Add `/api/v1/iam/auth/oauth2/**` to `public-paths` in `application.yml`
2. Exclude `/api/v1/iam/auth/oauth2/**` from tenant extraction filter

```yaml
iqkv:
  gateway:
    public-paths:
      # … all existing entries …
      - /api/v1/iam/auth/oauth2/** # OIDC authorize, callback, exchange, providers
```

**`HeaderSanitizationFilter` — no change required.**
All OIDC flows produce an IAM-issued JWT at the IAM service boundary. The gateway never
sees a raw IdP token. `JwtContextPropagationFilter` already extracts all needed claims
from the IAM JWT.

The `X-Tenant-ID` allowed-paths list does not need updating because the tenant key is
encoded in the `state` JWT, not sent as a request header on the callback.

Only changes to `application.yml` and `TenantContextFilter` are required in the gateway.

### 12. Tenant Context Resolution in OIDC Flows

The `X-Tenant-ID` header approach used by password signin cannot be applied to browser
redirect flows (the browser sets no custom headers on a redirect). Tenant context is
resolved differently per flow:

| Flow                                  | Tenant resolution                                                                            |
| ------------------------------------- | -------------------------------------------------------------------------------------------- |
| Browser login                         | `tenantKey` query param on `/authorize` → embedded in `state` JWT → recovered on `/callback` |
| SPA exchange                          | `tenantKey` field in `POST /exchange` request body                                           |
| No tenantKey provided (MULTI_TENANT)  | User joins "platform" tenant as MEMBER                                                       |
| No tenantKey provided (SINGLE_TENANT) | User joins default tenant with TENANT_OWNER authority                                        |
| Tenant switch after login             | Existing `POST /auth/exchange` endpoint unchanged                                            |

### 13. Token Refresh and Signout

OIDC-originated sessions are fully lifecycle-compatible with password sessions:

- **Refresh:** `POST /api/v1/iam/auth/refresh` with the IAM refresh token and
  `X-Tenant-ID`. No interaction with the IdP. Unchanged.
- **Signout:** `POST /api/v1/iam/auth/signout` adds the JTI to the denylist. Unchanged.
- **Global signout:** `POST /api/v1/iam/auth/signout-all` sets `last_global_signout_at`.
  Unchanged.
- **Denylist check:** `JwtAuthenticationFilter` applies to all tokens regardless of
  origin because both password and OIDC paths produce tokens with a `jti` claim.

The IAM service does **not** revoke the IdP session on signout. Back-channel logout
(OIDC logout endpoint notification) can be added later as a `LogoutSuccessHandler`
event without altering the token flow.

### 14. Account Linking & Email Conflict Handling

Users authenticated via bearer token can attach additional IdP identities:

- **Link:** `GET /api/v1/iam/auth/oauth2/link/{provider}` initiates an authorization
  flow. The current `userId` is embedded in the `state` JWT (`flowType=link`). On
  callback a new `user_identities` row is inserted. If the IdP identity is already
  linked to a different account, `409 Conflict` is returned.
- **Unlink:** `DELETE /api/v1/iam/auth/oauth2/link/{provider}` removes the row. The
  service enforces that the user retains at least one active credential — either a
  non-null `passwordHash` or another `user_identities` row — to prevent lockout.
  **Requires re-authentication** before unlinking.
- **List:** `GET /api/v1/iam/auth/oauth2/identities` returns all linked identities for
  the current user (`provider`, `displayName`, `email`, `linkedAt`).

#### Email Conflict Mitigation (Soft-Merge)

To mitigate risk of account takeover from reused corporate emails:

1. **Auto-Linking with Audit Logging**: When auto-linking occurs (existing user found by `emailVerified=true` email), log a detailed audit event including:
   - Timestamp
   - User ID
   - Provider name
   - Provider subject ID
   - IP address
   - User agent

2. **Admin Notification**: Send an email notification to platform administrators about the new identity linkage.

3. **Manual Unmerge Capability**: Provide admin endpoints to:
   - List all identity linkages
   - Unmerge a specific identity from a user account (moving it to a new user record or re-linking to another account)
   - View audit history of linkages/unmerges

4. **Configuration**: Add a config flag `iqkv.auth.oauth2.auto-link.enabled` (default `true`) to allow disabling auto-linking entirely in high-security environments.

### 15. GitHub Special Handling

GitHub does not implement OIDC (no `/.well-known/openid-configuration`, no ID token).
It is an OAuth2-only provider. Two differences require explicit handling:

1. **No ID token.** Spring Security treats GitHub as a plain `OAuth2User`, not an
   `OidcUser`. `OidcIdentity.fromGitHubUser()` normalizes this into the same record.
2. **Email may be private.** GitHub users can hide their primary email. The service
   must call `GET https://api.github.com/user/emails` using the GitHub access token
   to retrieve the verified primary email. **If no verified email exists, signin is
   rejected** with `400 Bad Request` and a user-facing message asking them to make their
   GitHub email public or use a different provider.

A `GitHubEmailFetcher` component wraps this API call with the GitHub access token.
It is invoked only during the GitHub OIDC callback — not on every request.

### 16. Security Considerations

| Risk                              | Mitigation                                                                                                                                                        |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| State CSRF forgery                | `state` is an RSA-signed JWT; signature verified before code exchange                                                                                             |
| Open redirect on callback         | `post-login-redirect-uri` is a fixed allowlist in config; never read from user input                                                                              |
| IdP token replay                  | Nonce in `state` JWT verified against `nonce` claim in OIDC ID token                                                                                              |
| PKCE downgrade                    | `code_challenge_method=S256` required; `plain` rejected at authorize time                                                                                         |
| Account takeover via reused email | Email-based linking only when `emailVerified=true` from the IdP; detailed audit logging; admin notifications; manual unmerge capability; auto-link disable config |
| Brute-force on OIDC endpoints     | Rate-limiting infrastructure (same lockout as `/signin`) applied per IP to `/authorize`, `/callback`, `/exchange`                                                 |
| Privilege escalation via OIDC     | Authorities resolved exclusively from local `TenantMembership`; IdP roles/groups are never used                                                                   |
| Tenant spoofing via state         | `tenantKey` from state must correspond to an existing `TenantMembership` for the provisioned user                                                                 |
| Client secret exposure            | Stored AES-256-GCM encrypted; decrypted only in-process; masked in API responses; never logged                                                                    |
| Credential lockout on unlink      | Unlink enforces at-least-one-credential guard + requires re-authentication                                                                                        |
| OIDC tokens reaching downstream   | Gateway never sees IdP tokens — IAM exchanges them for IAM-issued JWTs before any token leaves the IAM service                                                    |
