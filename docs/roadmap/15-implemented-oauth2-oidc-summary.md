# OAuth2 / OIDC - Implementation Summary

**Status:** Implemented with hardening and focused test coverage still open  
**Implementation Date:** June 30, 2026  
**Primary Targets:** `foundation-iam-service`, `foundation-gateway-service`, `foundation-ui-app`, `foundation-ui-platform-admin`

---

## 1. Overview

This document records what was actually built from
`15-proposal-oauth2-oidc.md`.

The platform now supports **OAuth2 / OpenID Connect identity federation** as a parallel
authentication path alongside the existing password-based and magic-link flows. IAM remains
the broker: external provider identities are exchanged for the platform's standard RS256 JWTs,
so Gateway, Billing, Audit, CMS, and future services continue trusting only IAM-issued tokens.

The delivered scope includes:

- social login for Google, GitHub, and Microsoft
- tenant-scoped enterprise SSO via custom OIDC provider configuration
- server-side PKCE with Redis-backed state storage
- account linking / unlinking
- admin remediation via linked-identity listing and forced unmerge
- tenant-app UX for sign-in, callback handling, connected accounts, and tenant SSO config
- platform-admin UX for linked-identity review and forced unmerge
- deployment/config/documentation updates across IAM, Helm, and Drone CI

Remaining work is concentrated in:

- focused automated tests
- stronger `id_token` validation hardening
- audit / notification follow-up for auto-linking
- admin audit-history endpoints

---

## 2. Architectural Outcome

### 2.1 What Shipped

The proposal's central design principle was preserved:

**IAM brokers identity and still issues the only tokens trusted by the platform.**

Delivered flow:

1. Browser or SPA starts OAuth2/OIDC with IAM
2. IAM builds provider authorization requests with signed state and PKCE
3. provider callback returns to IAM
4. IAM validates state, loads PKCE verifier from Redis, exchanges the authorization code
5. IAM normalizes the provider identity and provisions or links the local user
6. IAM issues the standard internal `TokenResponse`
7. Gateway and downstream services continue using the existing JWT/JWKS trust model

### 2.2 Providers and Modes

Implemented providers:

- Google
- GitHub
- Microsoft
- tenant-scoped custom OIDC providers stored in IAM and exposed as `oidc:{tenantKey}`

Supported modes:

- browser-based redirect flow
- SPA callback exchange flow
- existing-account linking / unlinking
- first-time user auto-provisioning
- tenant-owner managed enterprise SSO

---

## 3. Implemented Areas

### 3.1 Database and Persistence

Implemented:

- Liquibase changelog for `user_identities`
- Liquibase changelog for `tenant_oidc_providers`
- required constraints and indexes
- MyBatis mappers and XML mappings for both tables

Outcome:

- external identities are linked to internal users in a dedicated table
- per-tenant enterprise SSO configuration is stored centrally in IAM

### 3.2 IAM Dependencies and Configuration

Implemented:

- added `spring-boot-starter-security-oauth2-client`
- added `iqkv.auth.oauth2` configuration in `application.yml`
- added environment variable defaults for provider credentials, base URL, redirect URI, state TTL, encryption key, and auto-link settings
- moved static provider registration logic away from fragile eager binding so blank provider credentials do not break test startup

Outcome:

- deployments can enable providers selectively
- static providers remain inactive when credentials are blank
- tests can start without dummy Microsoft credentials

### 3.3 IAM Core Implementation

Implemented:

- new `oauth2` package structure
- `OidcIdentity`
- `OidcState`
- `OidcStateJwtService`
- `OidcStateStore` backed by Redis
- `AesGcmEncryptionService`
- `OidcProvisioningException`
- `OidcDtos`
- `UserIdentityMapper`
- `TenantOidcProviderMapper`
- `OidcUserProvisioningService` and implementation
- `DynamicClientRegistrationRepository`
- `GitHubEmailFetcher`
- `OidcAuthorizationRestResource`
- `TenantSsoRestResource`
- `TenantSsoService`
- `OidcAdminRestResource`
- `OidcAdminService` and implementation

Outcome:

- IAM now exposes a full OAuth2/OIDC surface without altering downstream token consumers
- Redis-backed state storage replaces the earlier in-memory approach and aligns with the finalized architecture

### 3.4 Security and Gateway Integration

Implemented:

- `SecurityConfig` updated to permit public OAuth2/OIDC endpoints and enable `oauth2Client`
- `JwtAuthenticationFilter.shouldNotFilter()` updated for OAuth2/OIDC flows
- gateway public paths updated with `/api/v1/iam/auth/oauth2/**`
- tenant extraction explicitly skips OAuth2/OIDC endpoints

Outcome:

- the redirect-based auth flow is reachable without tenant-header interference
- gateway and IAM cooperate cleanly for browser-based federated auth

### 3.5 Frontend Implementation

Implemented in `foundation-ui-app`:

- social login buttons on sign-in
- enterprise SSO entry point using tenant key
- callback page for token consumption
- connected accounts UI for link / unlink
- tenant-owner security panel for custom OIDC / enterprise SSO configuration

Implemented in `foundation-ui-platform-admin`:

- user-detail OIDC identities tab
- forced-unmerge action for admin remediation

Outcome:

- the feature is not backend-only; the shipped UX supports the main operator and end-user flows

### 3.6 Documentation and Deployment

Implemented:

- IAM API docs updated
- IAM `README.md` updated
- IAM `README.template.md` updated
- Helm chart updated for Redis, OIDC encryption key, and provider secret wiring
- Drone pipeline updated for Redis and OAuth2/OIDC secrets
- system-design documentation updated to include identity federation and enterprise SSO as a core v0.4 outcome

Outcome:

- implementation, deployment assets, and documentation are aligned

---

## 4. Delivered Endpoints and Capabilities

### Public OAuth2 / OIDC Endpoints

Implemented in IAM:

- `GET /api/v1/iam/auth/oauth2/authorize`
- `GET /api/v1/iam/auth/oauth2/callback`
- `POST /api/v1/iam/auth/oauth2/exchange`
- `GET /api/v1/iam/auth/oauth2/providers`
- `GET /api/v1/iam/auth/oauth2/link/{provider}`
- `GET /api/v1/iam/auth/oauth2/link/callback`
- `DELETE /api/v1/iam/auth/oauth2/link/{provider}`
- `GET /api/v1/iam/auth/oauth2/identities`

### Tenant SSO Management

Implemented in IAM:

- tenant-owner endpoints for configuring the tenant-scoped OIDC provider
- encrypted client secret storage using AES-256-GCM

### Admin Remediation

Implemented in IAM:

- `GET /api/v1/iam/admin/oidc/users/{userId}/identities`
- `DELETE /api/v1/iam/admin/oidc/users/{userId}/identities/{identityId}`

Implemented in platform-admin UI:

- linked-identity review on user detail
- force-unmerge action

---

## 5. Key Files Added or Significantly Changed

### IAM Service

Representative new or heavily changed files:

- `oauth2/OidcAuthorizationRestResource.java`
- `oauth2/OidcStateStore.java`
- `oauth2/OidcStateJwtService.java`
- `oauth2/OidcUserProvisioningService.java`
- `oauth2/OidcUserProvisioningServiceImpl.java`
- `oauth2/DynamicClientRegistrationRepository.java`
- `oauth2/GitHubEmailFetcher.java`
- `oauth2/AesGcmEncryptionService.java`
- `oauth2/OidcAdminRestResource.java`
- `oauth2/OidcAdminService.java`
- `oauth2/OidcAdminServiceImpl.java`
- `oauth2/sso/TenantSsoRestResource.java`
- `oauth2/sso/TenantSsoService.java`
- `oauth2/sso/TenantSsoServiceImpl.java`
- `oauth2/mapper/UserIdentityMapper.java`
- `oauth2/mapper/TenantOidcProviderMapper.java`
- `infrastructure/config/OAuth2ConfigurationProperties.java`
- `infrastructure/config/SecurityConfig.java`
- `infrastructure/config/MyBatisConfig.java`
- `tenancy/TenantExtractionFilter.java`
- `resources/application.yml`
- `resources/db/changelog/system/20260630000000-oauth2-user-identities.xml`
- `resources/db/changelog/system/20260630100000-tenant-oidc-providers.xml`

### Gateway

- gateway public path configuration for OAuth2/OIDC
- tenant extraction exclusion for `/api/v1/iam/auth/oauth2/**`

### Tenant App

- sign-in page updates
- callback page
- security page connected-accounts and tenant-SSO UI
- IAM API client additions

### Platform Admin UI

- OIDC identity entity types
- IAM admin API bindings
- user-detail OIDC identities tab and force-unmerge flow

---

## 6. Proposal Checklist Status

### Completed

- Database & Migrations: all checklist items complete
- IAM Service Dependencies: all checklist items complete
- Configuration: all checklist items complete
- Core Implementation: all listed items complete
- Gateway Changes: both checklist items complete
- Documentation: all listed items complete
- Security & Audit:
  - admin unmerge endpoints complete

### Still Open

Tests:

- Unit tests for `OidcUserProvisioningServiceImpl`
- Unit tests for `OidcStateJwtService`
- Unit tests for `AesGcmEncryptionService`
- Unit tests for `DynamicClientRegistrationRepository`
- Integration tests for `OidcAuthorizationRestResource`

Security / Audit:

- audit logging for auto-linking events
- admin notifications for auto-linking
- admin audit-history endpoints

Hardening:

- full JWK-backed `id_token` validation beyond the current nonce / claim checks

---

## 7. Important Implementation Decisions Preserved

The final implementation kept the proposal's key architecture decisions intact:

- **verified-email auto-linking** is the trust model for merging external identities into existing accounts
- **GitHub verified email** is required before sign-in succeeds
- **Redis-backed PKCE storage** is used instead of in-memory state
- **tenant extraction is bypassed** for OAuth2/OIDC routes
- **tenant client secrets are encrypted** with AES-256-GCM using `OIDC_ENCRYPTION_KEY`
- **unlink safeguards** prevent removal of the last remaining sign-in method when no password exists

One notable implementation refinement happened during delivery:

- static provider registration was refactored to use the platform's own `iqkv.auth.oauth2`
  configuration model instead of relying on an eagerly validated `OAuth2ClientProperties`
  bean, which had caused test-context startup failures when `microsoft.client-id` was blank

---

## 8. Verification Notes

Implementation was verified incrementally during delivery:

- IAM compile and test-compile were stabilized after the OIDC package introduction
- the OAuth2 test-context startup issue caused by empty Microsoft credentials was fixed
- the Spring test context was verified to boot successfully after the configuration refactor
- the AES-GCM implementation was adjusted to satisfy SpotBugs without changing behavior
- tenant-app and platform-admin UI builds were brought to a clean TypeScript state after adding the new OIDC features

---

## 9. Final State

The OAuth2/OIDC proposal is **substantially implemented end-to-end**.

What shipped is not only backend plumbing; it includes:

- IAM provider integration
- gateway/public-path support
- tenant-scoped enterprise SSO configuration
- account linking and unlinking
- admin remediation endpoints
- tenant-app user flows
- platform-admin operator flows
- deployment, CI, and documentation alignment

The remaining work is concentrated in **test coverage, audit history, and additional security hardening** rather than missing core product capability. This document therefore supersedes the proposal as the authoritative summary of what was actually delivered.
