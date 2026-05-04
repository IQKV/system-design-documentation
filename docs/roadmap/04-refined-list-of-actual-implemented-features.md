## Complete Feature Analysis — All Three Services

---

## IAM Service (`foundation-iam-service`)

### `authentication/`

**`AuthenticationRestResource`** — `@RestController /api/v1/iam/auth`

- `POST /signup` → `UserService.registerUser()`, returns 201 + `SignupResponse`
- `POST /signin` → `AuthenticationService.signIn()`, returns access + refresh tokens
- `POST /refresh` → rotates token pair
- `POST /signout` → adds JTI to denylist
- `POST /signout-all` → sets `last_global_signout_at` on user + denylists current token
- `POST /validate` → decodes token, checks denylist, returns userId/email/tenantId/authorities (gateway introspection)

**`AuthenticationServiceImpl`** — `@Service @Transactional`

- `signIn`: resolves tenant from `TenantContext`, rejects SUSPENDED/DELETED/PROVISIONING_FAILED tenants, BCrypt password check, lockout check, resolves membership + authorities, generates RS256 access + refresh tokens
- `refresh`: validates `type=refresh` claim, checks tenant context match, re-resolves membership, issues new pair
- `signOut`: `TokenDenylistService.denyToken(jti, userId, expiresAt)`
- `signOutAll`: updates `last_global_signout_at` + denylists current token
- `validateToken`: decodes JWT, checks denylist, returns user context
- `listUserTenants`: credential-gated (no JWT), verifies password, returns all ACTIVE memberships in ACTIVE tenants
- `verifyEmail`: consumes 64-char hex token, marks email verified, publishes `EMAIL_VERIFIED` notification
- `resendVerification`: rate-limited (3/hour), generates new token, publishes `VERIFY_EMAIL` notification

**`JwtTokenGenerator`** — `@Component`

- Loads RSA private key from PEM at startup
- `generateAccessToken`: RS256 JWT with claims: `sub`, `iss`, `iat`, `exp`, `jti`, `type=access`, `userId`, `username`, `email`, `firstName`, `lastName`, `tenant_id`, `email_verified`, `authorities`
- `generateRefreshToken`: RS256 JWT with `type=refresh`, minimal claims

---

### `denylist/`

**`TokenDenylist`** — domain: `id`, `jti`, `userId`, `expiresAt`, `createdAt`

**`TokenDenylistService`** — `@Component`

- `denyToken(jti, userId, expiresAt)`: inserts into DB denylist
- `isRevoked(jti)`: checks by JTI
- `cleanupExpired()`: `@Scheduled(cron="0 0 * * * *")` + `@SchedulerLock` — hourly cleanup

---

### `email/`

**`EmailVerificationRestResource`** — `@RestController /api/v1/iam/users/email`

- `POST /verify` → consumes one-time token, marks email verified
- `POST /resend-verification` → rate-limited resend, returns 202

**`EmailVerificationToken`** — domain: `id`, `userId`, `token`, `expiresAt`, `resendCount`, `lastResendAt`, `createdAt`

**`EmailVerificationTokenMapper`** — `@Mapper`: `insert`, `findByToken`, `deleteByUserId`, `deleteByExpiresAtBefore`, `incrementResendCount`, `countResendsWithinWindow`

**`ExpiredVerificationTokenReaperJob`** — `@Scheduled(cron="0 0 * * * *")` + `@SchedulerLock` — hourly cleanup

---

### `invitation/`

**`TenantInvitation`** — domain: `id`, `tenantKey`, `invitedEmail`, `invitedBy`, `authority`, `token` (32-byte hex), `status`, `expiresAt`, `acceptedAt`, audit fields. `isPending()` helper.

**`InvitationStatus`** — enum: `PENDING`, `ACCEPTED`, `REVOKED`, `EXPIRED`

**`InvitationDtos`** — records:

- `SendInvitationRequest`: `@Email email`, `@Pattern(ADMIN|MEMBER) authority`
- `AcceptInvitationRequest`: `firstName`, `lastName`, `@NotBlank password`
- `InvitationResponse`, `InvitationPreviewResponse`, `AcceptInvitationResponse`

**`InvitationServiceImpl`** — `@Service @Transactional`

- `sendInvitation`: MULTI_TENANT only; verifies inviter has `TENANT_OWNER` or `ADMIN`; guards duplicate PENDING + existing membership; generates 32-byte hex token; publishes `user.invited` event + `INVITATION` email
- `previewInvitation`: public; returns tenant name, email, authority, expiry, `requiresSignup` flag
- `acceptInvitation`: validates pending token; existing user → verifies password with lockout; new user → creates with `emailVerified=true`; creates membership + grants authority; consumes invitation; issues JWT pair
- `revokeInvitation`: inviter, `TENANT_OWNER`, or `ADMIN` only; sets `REVOKED`
- `listInvitations`: all `PENDING` for a tenant

**`InvitationRestResource`** — `@RestController`

- `POST /api/v1/iam/tenants/{tenantKey}/invitations` — `@PreAuthorize("hasAnyAuthority('TENANT_OWNER', 'ADMIN')")`
- `GET /api/v1/iam/tenants/{tenantKey}/invitations` — same
- `DELETE /api/v1/iam/tenants/{tenantKey}/invitations/{invitationId}` — same
- `GET /api/v1/iam/invitations/{token}` — public
- `POST /api/v1/iam/invitations/{token}/accept` — public

**`InvitationReaperJob`** — `@Scheduled(fixedDelay=PT1H)` + `@SchedulerLock` — bulk-expires stale PENDING invitations

---

### `lockout/`

**`AccountLockoutManager`** — `@Component`

- Reads `loginAttempts` threshold + `lockoutDuration` from `AuthConfigurationProperties`
- `recordFailedAttempt(email)`: inserts failed attempt record
- `isLocked(email)`: counts attempts within window; true if ≥ threshold
- `reset(email)`: deletes all failed attempts for email

---

### `membership/`

**`MembershipStatus`** — enum: `ACTIVE`, `SUSPENDED`, `REMOVED`

**`TenantMembership`** — domain: `id`, `userId`, `tenantKey`, `status`, audit fields

**`TenantMemberAuthority`** — domain: `id`, `membershipId`, `authority`

**`MembershipServiceImpl`** — `@Service @Transactional(readOnly=true)`

- `resolveMembership`: finds membership, throws `MembershipNotFoundException` if SUSPENDED or REMOVED
- `getAuthorities`: returns list of authority strings for a membership

---

### `passwordreset/`

**`PasswordResetServiceImpl`** — `@Service @Transactional`

- `initiate`: silently ignores unknown emails (prevents enumeration); rate-limits (configurable); generates 32-byte hex token; publishes `PASSWORD_RESET_INITIATE` notification
- `complete`: validates token + expiry; enforces password complexity (lowercase + uppercase + digit + special char, 8–128 chars); updates password hash; deletes token; sets `last_global_signout_at` (invalidates all sessions); publishes `PASSWORD_RESET_CONFIRMED` notification

**`PasswordResetRestResource`** — `@RestController /api/v1/iam/users/password`

- `POST /forgot` → always returns 200 (prevents enumeration)
- `POST /reset` → 200 on success, 400 on invalid/expired token

**`ExpiredPasswordResetTokenReaperJob`** — `@Scheduled(cron="0 0 * * * *")` + `@SchedulerLock` — hourly cleanup

---

### `security/`

**`JwtClaimNames`** — constants: `SUB`, `ISS`, `IAT`, `EXP`, `JTI`, `TYPE`, `USER_ID`, `USERNAME`, `EMAIL`, `FIRST_NAME`, `LAST_NAME`, `TENANT_ID`, `AUTHORITIES`, `EMAIL_VERIFIED`, `TYPE_ACCESS`, `TYPE_REFRESH`, `ISSUER`

**`JwtAuthenticationFilter`** — `@Component extends OncePerRequestFilter` — runs before `BearerTokenAuthenticationFilter`

- Check 1: JTI in `token_denylist` → 401
- Check 2: token `iat` ≤ `last_global_signout_at` → 401
- Skips public paths (signup, signin, refresh, validate, email verify, password reset)

---

### `signup/`

**`SignupStrategy`** — interface: `execute(RegisterUserRequest)` → `SignupResult`

**`SignupResult`** — record: `user`, `tenant`, `membership`, `authorities`

**`MultiTenantSignupStrategy`** — `@Service @ConditionalOnProperty(rollout-mode=MULTI_TENANT)`

- Upserts user by email (atomic, eliminates TOCTOU)
- Creates new tenant with NanoID 8-char key, status `PROVISIONING`
- Creates membership `ACTIVE`, grants `TENANT_OWNER`
- Publishes `tenant.created` event with owner fields

**`SingleTenantSignupStrategy`** — `@Service @ConditionalOnProperty(rollout-mode=SINGLE_TENANT)`

- Ignores `tenantName` field
- Upserts user, resolves default tenant via `DefaultTenantResolver`
- Guards against duplicate membership
- Creates membership `ACTIVE`, grants `MEMBER` (not `TENANT_OWNER`)
- Does NOT publish `tenant.created`

---

### `tenancy/`

**`TenantContext`** — `ThreadLocal<String>`: `setCurrentTenant`, `getCurrentTenant` (throws if not set), `clear`

**`TenantExtractionFilter`** — `@Component @Order(HIGHEST_PRECEDENCE+1) extends OncePerRequestFilter`

- Priority 1: `X-Tenant-ID` header; Priority 2: JWT `tenant_id` claim
- Returns 400 if unresolvable; always clears `TenantContext` in `finally`
- Skips public paths

**`MyBatisSchemaInterceptor`** — `@Intercepts(StatementHandler.prepare)` — sets PostgreSQL `search_path` to `t_{tenantKey}, public` before each MyBatis statement; validates key against `^[a-zA-Z0-9_]+$`

**`TenantLiquibaseRunner`** — `@Component ApplicationRunner`

- `run()`: runs system schema migrations on `public` at startup
- `runMigrationsForTenant(tenantKey)`: creates `t_{tenantKey}` schema + runs tenant Liquibase changelog

---

### `tenant/`

**`TenantStatus`** — enum: `PROVISIONING`, `ACTIVE`, `SUSPENDED`, `DELETED`, `PROVISIONING_FAILED`

**`TenantServiceImpl`** — `@Service @Transactional`

- `createTenant`: checks name uniqueness, NanoID key, inserts `PROVISIONING`, publishes `tenant.created`
- `updateTenantStatus`: enforces allowed transitions (`ACTIVE↔SUSPENDED`, `ACTIVE/SUSPENDED/PROVISIONING_FAILED→DELETED`); publishes `tenant.suspended` or `tenant.deleted`
- `retryProvisioning`: only from `PROVISIONING_FAILED`; re-publishes `tenant.created`

**`TenantRestResource`** — `@RestController /api/v1/iam/tenants`

- `GET /{tenantKey}` — `@PreAuthorize("hasAuthority('TENANT_OWNER')")`
- `PATCH /{tenantKey}/status` — same
- `POST /{tenantKey}/retry-provisioning` — same

**`TenantProvisioningConsumer`** — `@RabbitListener(TENANT_PROVISIONING_QUEUE)`: runs `TenantLiquibaseRunner.runMigrationsForTenant()`, sets `ACTIVE`, publishes `tenant.provisioned`; on failure sets `PROVISIONING_FAILED`, publishes `tenant.provisioning_failed`

**`StuckTenantReaperJob`** — `@Scheduled(cron="0 */5 * * * *")` + `@SchedulerLock` — every 5 min marks tenants stuck in `PROVISIONING` beyond timeout as `PROVISIONING_FAILED`

---

### `user/`

**`AccountStatus`** — enum: `ACTIVE` (single value)

**`User`** — domain: `id`, `email`, `passwordHash`, `firstName`, `lastName`, `status`, `emailVerified`, `lastGlobalSignoutAt`, audit fields

**`UserServiceImpl`** — `@Service @Transactional`

- `registerUser`: delegates to `SignupStrategy`, publishes `user.created`, generates email verification token, publishes `VERIFY_EMAIL` notification
- `listUsers`: paginated `PagedUserResponse`
- `createUser`: admin path — random temp password
- `patchUser`: partial update (firstName, lastName, status)
- `deleteUser`: removes membership only, publishes `user.removed`
- `deleteUserById`: hard-deletes user record

**`UserRestResource`** — `@RestController /api/v1/iam/users`

- `GET /me`, `PATCH /me`, `DELETE /me` — `@PreAuthorize("isAuthenticated()")`
- `POST /tenants` — credential-gated tenant discovery (no JWT)

**`UserOperatorRestResource`** — `@RestController /api/v1/iam/operator/users`

- `GET /` (paginated), `GET /{id}`, `POST /`, `PUT /{id}`, `PATCH /{id}`, `DELETE /{id}`

---

### `infrastructure/config/`

**`SecurityConfig`** — stateless, CSRF disabled; RSA public key from PEM; `JwtAuthenticationFilter` before `BearerTokenAuthenticationFilter`; BCrypt strength 12

**`AuthConfigurationProperties`** — `@ConfigurationProperties("iqkv.auth")`: JWT (privateKeyPath, publicKeyPath, expiry, refreshExpiry, issuer), Security (passwordEncoderStrength, minLength, RateLimiting{loginAttempts, lockoutDuration}), PasswordReset (tokenTtl, rateLimitWindow, rateLimitMaxRequests)

**`PlatformConfigurationProperties`** — `@ConfigurationProperties("iqkv.platform")`: `rolloutMode`

**`PlatformModeValidatorImpl`** — `@Order(HIGHEST_PRECEDENCE) ApplicationRunner`: validates rollout mode at startup; throws `InvalidPlatformModeException` if missing

**`RabbitMQConfig`** — `@ConditionalOnProperty(rabbitmq.enabled=true) @Profile("!test")`

- Exchange: `iqkv.events` (topic), `iqkv.dlx` (dead-letter)
- Queues: `iqkv.iam.user.events`, `iqkv.iam.notifications`, `iqkv.iam.tenant.provisioning`, `iqkv.iam.subscription.events`
- All queues: 24h TTL + dead-letter exchange

**`ShedLockConfig`** — `JdbcTemplateLockProvider` using DB time

**`PlatformModeInfoContributor`** — exposes `platform.rollout-mode` to `/actuator/info` (consumed by Gateway's `PlatformModeGuardFilter`)

---

### `infrastructure/messaging/`

**`MessagingService`** — publishes: `tenant.created/provisioned/provisioning_failed/suspended/deleted/updated`, `user.created/deleted/removed/invited`, `notification.iam.email`

**`UserEventPublisher`** — `publishUserCreated`, `publishUserDeleted`, `publishUserRemoved`

**`NotificationConsumer`** — `@RabbitListener(NOTIFICATIONS_QUEUE)` → `EmailService`; swallows exceptions (DLQ)

**`SubscriptionEventConsumer`** — `@RabbitListener(SUBSCRIPTION_EVENTS_QUEUE)`: on `subscription.cancelled`, suspends tenant via `TenantService.updateTenantStatus(SUSPENDED)`

---

### `infrastructure/security/`

**`JwksRestResource`** — `@RestController /.well-known/jwks.json`: exposes RSA public key as JWKS

**`CorrelationIdFilter`** — `@Order(HIGHEST_PRECEDENCE)`: generates/propagates `X-Correlation-ID`; puts in MDC

---

### `infrastructure/metrics/`

**`UserServiceMetrics`** — Micrometer: counters `auth.success` (tag: tenantId), `auth.failure` (tags: tenantId, reason), `tenant.created`; timer `auth.duration` (tag: tenantId)

---

## Gateway Service (`foundation-gateway-service`)

### `infrastructure/config/`

**`SecurityConfig`** — `@EnableWebFluxSecurity` (reactive)

- CSRF, HTTP Basic, form login disabled
- Public paths from `GatewayProperties.publicPaths`
- OAuth2 Resource Server with JWKS endpoint (from IAM)
- Custom `ReactiveJwtAuthenticationConverter` extracts `authorities` claim
- 401 on auth failure, 403 on access denied

**`GatewayProperties`** — `@ConfigurationProperties("iqkv.gateway")`: `publicPaths: List<String>`

**`GatewayConfigurationProperties`** — nested:

- `Tenancy @ConfigurationProperties("iqkv.tenancy")`: `defaultTenantKey`
- `Iam @ConfigurationProperties("iqkv.iam")`: `serviceUrl` (default: `http://foundation-iam-service:8080`)

**`PlatformModeValidatorImpl`** — `@Order(HIGHEST_PRECEDENCE) ApplicationRunner`: validates rollout mode; on failure sets readiness to `REFUSING_TRAFFIC`

**`PlatformModeGuardFilter`** — `GlobalFilter, Ordered` (order: `HIGHEST_PRECEDENCE`)

- `@PostConstruct` + `@Scheduled(fixedDelay=60s)`: queries IAM `/actuator/info` for `platform.rollout-mode`
- Mismatch → `modeMismatchDetected=true`, `ReadinessState.REFUSING_TRAFFIC`, returns 503 on all requests
- IAM unreachable → fail-open (logs warning, allows traffic)
- Mismatch resolved → restores `ACCEPTING_TRAFFIC`

---

### `infrastructure/security/`

**`CorrelationIdFilter`** — `GlobalFilter` (order: `-200`): generates/propagates `X-Correlation-ID`; stores in exchange attributes

**`HeaderSanitizationFilter`** — `GlobalFilter` (order: `-190`): strips `X-User-ID`, `X-Username`, `X-User-Email`, `X-User-Authorities`, `X-User-Permissions`, `X-Tenant-ID`, `X-Organization-ID` from incoming requests (prevents identity spoofing)

**`JwtContextPropagationFilter`** — `GlobalFilter` (order: `-100`): reads validated JWT from `ReactiveSecurityContextHolder`; adds downstream headers: `X-User-ID`, `X-Username`, `X-User-Email`, `X-Tenant-ID`, `X-User-Authorities`

**`TenantContextResolutionPolicy`** — interface: `resolveTenantContext(exchange)` → `Optional<String>`

**`TenantContextFilter`** — `GlobalFilter, TenantContextResolutionPolicy` (order: `-50`)

- `MULTI_TENANT`: uses `X-Tenant-ID` from JWT propagation; no auto-injection
- `SINGLE_TENANT`: if `X-Tenant-ID` absent, injects `iqkv.tenancy.default-tenant-key`

**`ResponseTransformationFilter`** — `GlobalFilter` (order: `MIN_VALUE+1`): adds `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `X-XSS-Protection: 1; mode=block`, `Referrer-Policy: strict-origin-when-cross-origin`; echoes `X-Correlation-ID` to client

**Filter execution order:**

1. `-200` `CorrelationIdFilter`: generate/propagate correlation ID
2. `-190` `HeaderSanitizationFilter`: strip spoofable headers
3. Spring Security: validate JWT via JWKS
4. `-100` `JwtContextPropagationFilter`: JWT claims → downstream headers
5. `-50` `TenantContextFilter`: inject/validate tenant context
6. Route to downstream service
7. `MIN_VALUE+1` `ResponseTransformationFilter`: add security response headers

---

## Billing Service (`foundation-billing-service`)

### `plan/`

**`Plan`** — domain: `id`, `planCode`, `displayName`, `billingPeriod` (MONTHLY|ANNUAL), `priceMinor` (cents), `currency`, `featureSet` (JSON string), `scope` (TENANT|USER), `active`

**`PlanCatalogRestResource`** — `@RestController /api/v1/billing/plans`

- `GET /` — list active plans (any authenticated user)
- `GET /{planCode}` — get by planCode (any authenticated user)
- `POST /` — `@PreAuthorize("hasAuthority('PLATFORM_OPERATOR')")` — create
- `PUT /{planCode}` — `PLATFORM_OPERATOR` — replace
- `DELETE /{planCode}` — `PLATFORM_OPERATOR` — soft-delete (`active=false`)

**`PlanEligibilityPolicyImpl`** — `@Service`: validates plan `scope` matches `SubjectType` (TENANT plan → TENANT subject only); throws `PlanScopeMismatchException` on mismatch

---

### `settings/`

**`BillingSettings`** — domain: `id`, `tenantKey`, `externalCustomerId` (Stripe `cus_xxx`), `billingEmail`, `companyName`, `billingAddress` (JSONB as String), `taxId`, `taxIdType`, `currency`, `profileOwnerId` (soft ref to IAM user, no FK)

**`BillingSettingsRestResource`** — `@RestController /api/v1/billing/settings`

- `GET /{tenantKey}` — `@PreAuthorize("hasAuthority('TENANT_OWNER')")` — validates JWT tenant matches path
- `PATCH /{tenantKey}` — same — partial update, publishes `BILLING_UPDATED` notification

**`BillingSettingsService`** — `@Service`: `getByTenantKey`, `update` (validates tenant context match, applies non-null fields)

---

### `subscription/`

**`Subscription`** — domain: `id`, `tenantKey`, `externalSubscriptionId`, `externalCustomerId`, `status` (active|past_due|canceled|unpaid|trialing), `planId`, `currentPeriodStart`, `currentPeriodEnd`, `cancelAtPeriodEnd`, `canceledAt`, `subjectType` (TENANT|USER), `subjectKey`

**`SubscriptionRestResource`** — `@RestController /api/v1/billing/subscriptions`

- `GET /{tenantKey}/active` — `@PreAuthorize("hasAuthority('TENANT_OWNER')")` — active subscription for tenant
- `GET /{tenantKey}` — `TENANT_OWNER` — all subscriptions for tenant
- `GET /me/active` — `TENANT_OWNER` or `MEMBER` — active subscription for resolved subject (tenant or user depending on mode)
- `GET /me` — same — all subscriptions for resolved subject

**`SubscriptionService`** — `@Service`: reads local Stripe cache (no gateway round-trips); uses `SubscriptionSubjectResolver` to determine TENANT vs USER scope

**`EntitlementEvaluator`** — interface: `evaluateEntitlements(subject)` → `Optional<EntitlementDetails>`

**`DefaultEntitlementEvaluator`** — `@Component`: finds active subscription by subject; enriches with plan `featureSet` from plan catalog; returns `EntitlementDetails` (subject, planId, status, currentPeriodEnd, featureSet)

**`TrialNotificationService`** — `@Service @ConditionalOnProperty(rabbitmq.enabled=true)`

- `sendTrialEndingNotifications()`: `@Scheduled(cron="0 0 9 * * *")` + `@SchedulerLock` — daily 9AM UTC; finds trials ending in 2–3 days; publishes `TRIAL_ENDING` notification
- `sendPaymentOverdueNotifications()`: `@Scheduled(cron="0 0 10 * * *")` + `@SchedulerLock` — daily 10AM UTC; finds `past_due` subscriptions; publishes `PAYMENT_OVERDUE` notification

**`MultiTenantSubscriptionSubjectResolver`** — `@ConditionalOnProperty(rollout-mode=MULTI_TENANT)`: subject = `TENANT / tenantKey`

**`SingleTenantSubscriptionSubjectResolver`** — `@ConditionalOnProperty(rollout-mode=SINGLE_TENANT)`: subject = `USER / userId`

---

### `webhook/`

**`WebhookLog`** — domain: `id`, `externalEventId`, `eventType`, `status` (RECEIVED|PROCESSED|FAILED), `errorMessage`, `receivedAt`, `processedAt`

**`StripeWebhookRestResource`** — `@RestController /api/v1/billing/webhooks`

- `POST /stripe` — no auth; verifies `Stripe-Signature` header via `Webhook.constructEvent()`; delegates to `WebhookProcessingService`; always returns 200 after idempotency check

**`WebhookProcessingService`** — `@Service`

- Idempotency via `webhook_log` (checks `existsByExternalEventId` before processing)
- Status lifecycle: `RECEIVED` → `PROCESSED` or `FAILED`
- Handles: `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`
- `handleSubscriptionCreated`: resolves subject (TENANT or USER); validates plan eligibility; upserts subscription; publishes `subscription.created` event; sends `SUBSCRIPTION_ACTIVATED` notification
- `handleSubscriptionUpsert`: upserts subscription; sends `SUBSCRIPTION_UPDATED` notification
- `handleSubscriptionDeleted`: sets status `canceled`; publishes `subscription.cancelled` event; sends `SUBSCRIPTION_CANCELLED` notification
- `handleInvoicePaid`: resolves billing settings by Stripe customer ID; publishes `invoice.paid` event; sends `INVOICE_PAID` notification
- `handleInvoicePaymentFailed`: publishes `payment.failed` event; sends `PAYMENT_FAILED` notification

---

### `userbilling/`

**`UserBillingSettings`** — domain (SINGLE_TENANT mode only): `id`, `userId`, `externalCustomerId`, `billingEmail`, `companyName`, `billingAddress` (JSON), `taxId`, `taxIdType`, `currency`

**`UserBillingSettingsService`** — interface: `getOrCreateUserBillingSettings(userId)` — idempotent

**`UserBillingSettingsServiceImpl`** — `@Service @ConditionalOnProperty(rollout-mode=SINGLE_TENANT)`: checks for existing settings; if absent, creates Stripe customer (no email), inserts `user_billing_settings` row

---

### `infrastructure/messaging/`

**`TenantEventConsumer`** — `@Component @ConditionalOnProperty(rabbitmq.enabled=true)` — `@RabbitListener(TENANT_EVENTS_QUEUE)`

- `TENANT_CREATED`: creates Stripe customer via `PaymentGatewayClient`, inserts `BillingSettings`; resolves billing email via `BillingContactResolver` fallback chain
- `TENANT_PROVISIONED`: sends `SUBSCRIPTION_ACTIVATED` notification to billing contact
- `TENANT_PROVISIONING_FAILED`: logs warning, no billing action
- `TENANT_SUSPENDED`: sends `ACCOUNT_SUSPENDED` notification
- `TENANT_DELETED`: logs, future: cancel Stripe customer

**`UserEventConsumer`** — `@Component @ConditionalOnProperty(rabbitmq.enabled=true)` — `@RabbitListener(USER_EVENTS_QUEUE)`

- `USER_REMOVED`: clears `profileOwnerId` on that tenant's billing settings if it matches
- `USER_DELETED`: clears `profileOwnerId` across all billing settings for that user

**`MessagingService`** — `@Service`: publishes `subscription.created`, `subscription.cancelled`, `invoice.paid`, `payment.failed`, `notification.billing.email` to `iqkv.events` exchange

**`BillingEmailConsumer`** — `@RabbitListener(NOTIFICATIONS_QUEUE)` → `EmailService`; swallows exceptions (DLQ)

---

### `infrastructure/config/`

**`StripeConfigurationProperties`** — `@ConfigurationProperties("iqkv.stripe")`: `@NotBlank secretKey`, `@NotBlank webhookSecret`

**`BillingConfigurationProperties`** — `@ConfigurationProperties("iqkv.billing")`: `defaultContactEmail` (fallback for SINGLE_TENANT mode)

**`PaymentGatewayClient`** — `@Component`: initializes `Stripe.apiKey` at construction; `createCustomer(name, email)` — creates Stripe customer, returns `cus_xxx` ID

**`SecurityConfig`** — `@EnableWebSecurity @EnableMethodSecurity`

- Stateless, CSRF disabled
- Public: actuator, api-docs, swagger-ui, `POST /api/v1/billing/webhooks/stripe`
- OAuth2 Resource Server with RSA public key from PEM
- Registers `CorrelationIdFilter` before `BearerTokenAuthenticationFilter`, `TenantExtractionFilter` after

---

## Cross-Cutting Summary

**Rollout modes:** Both `MULTI_TENANT` and `SINGLE_TENANT` are fully implemented. Strategy pattern (`SignupStrategy`, `SubscriptionSubjectResolver`, `UserBillingSettingsService`) selects behavior via `@ConditionalOnProperty`. All three services validate mode consistency at startup; Gateway additionally polls IAM `/actuator/info` every 60s and blocks traffic on mismatch.

**Token revocation:** Two-layer — JTI denylist (per-signout) + `last_global_signout_at` timestamp (signout-all). Both checked in `JwtAuthenticationFilter` before Spring Security processes the request.

**Multi-tenancy:** Schema-per-tenant isolation in IAM (`t_{tenantKey}` PostgreSQL schemas). `MyBatisSchemaInterceptor` sets `search_path` per request. `TenantLiquibaseRunner` provisions schemas on `tenant.created` events.

**Messaging:** RabbitMQ topic exchange `iqkv.events`. IAM publishes tenant/user lifecycle events; Billing consumes them to bootstrap Stripe customers and billing settings. Billing publishes subscription/invoice/payment events back. All queues have 24h TTL + dead-letter exchange. ShedLock prevents duplicate scheduled job execution in clustered deployments.

**Security headers (Gateway):** `HeaderSanitizationFilter` strips all user/tenant context headers from client requests before JWT propagation adds them back from the validated token — prevents identity spoofing.

I'll read all the requested files in parallel across all three services.

Now I have the full picture of all files. Let me read them all in large parallel batches.

Here is the complete, thorough breakdown of every file read across all three services.

---

## IAM Service (`foundation-iam-service`)

### `authentication/`

**`AuthenticationRestResource`** — `@RestController` at `/api/v1/iam/auth`

- `POST /signup` → delegates to `UserService.registerUser()`, returns 201 with `SignupResponse`
- `POST /signin` → calls `AuthenticationService.signIn()`, returns access + refresh tokens
- `POST /refresh` → calls `AuthenticationService.refresh()`, rotates token pair
- `POST /signout` → extracts JTI from JWT, calls `AuthenticationService.signOut()` (adds to denylist)
- `POST /signout-all` → calls `AuthenticationService.signOutAll()` (sets `last_global_signout_at`)
- `POST /validate` → calls `AuthenticationService.validateToken()`, returns user context for gateway introspection

**`AuthenticationService`** — interface defining: `signIn`, `refresh`, `signOut`, `signOutAll`, `validateToken`, `listUserTenants`, `verifyEmail`, `resendVerification`

**`AuthenticationServiceImpl`** — `@Service @Transactional`

- `signIn`: resolves tenant from `TenantContext`, checks tenant status (rejects SUSPENDED/DELETED/PROVISIONING_FAILED), loads user, checks lockout, verifies BCrypt password, resolves membership + authorities, resets lockout counter, generates RS256 access + refresh tokens
- `refresh`: decodes refresh token, validates `type=refresh` claim, checks tenant context match, re-resolves membership, issues new token pair
- `signOut`: calls `TokenDenylistService.denyToken(jti, userId, expiresAt)`
- `signOutAll`: updates `last_global_signout_at` on user record + denylists current token
- `validateToken`: decodes JWT, checks denylist, returns `ValidateTokenResponse` with userId/email/tenantId/authorities
- `listUserTenants`: credential-gated (no JWT), verifies password, returns all ACTIVE memberships in ACTIVE tenants
- `verifyEmail`: consumes 64-char hex token, marks user email verified, deletes token, publishes `EMAIL_VERIFIED` notification
- `resendVerification`: rate-limited (3/hour), generates new 32-byte hex token, publishes `VERIFY_EMAIL` notification

**`JwtTokenGenerator`** — `@Component`

- Loads RSA private key from PEM file at startup
- `generateAccessToken(user, tenantKey, authorities)`: RS256 JWT with claims: `sub`, `iss`, `iat`, `exp`, `jti`, `type=access`, `userId`, `username`, `email`, `firstName`, `lastName`, `tenant_id`, `email_verified`, `authorities`
- `generateRefreshToken(user, tenantKey)`: RS256 JWT with `type=refresh`, minimal claims

---

### `denylist/`

**`TokenDenylist`** — domain class: `id`, `jti`, `userId`, `expiresAt`, `createdAt`

**`TokenDenylistMapper`** — `@Mapper` (MyBatis): `insert`, `existsByJti`, `findByUserId`, `deleteByExpiresAtBefore`

**`TokenDenylistService`** — `@Component`

- `denyToken(jti, userId, expiresAt)`: inserts into denylist table
- `isRevoked(jti)`: checks existence by JTI
- `cleanupExpired()`: `@Scheduled(cron="0 0 * * * *")` + `@SchedulerLock` — hourly cleanup of expired entries

---

### `email/`

**`EmailVerificationRestResource`** — `@RestController` at `/api/v1/iam/users/email`

- `POST /verify` → calls `AuthenticationService.verifyEmail(token)`
- `POST /resend-verification` → calls `AuthenticationService.resendVerification(email)`, returns 202

**`EmailVerificationToken`** — domain class: `id`, `userId`, `token`, `expiresAt`, `resendCount`, `lastResendAt`, `createdAt`

**`EmailVerificationTokenMapper`** — `@Mapper`: `insert`, `findByToken`, `deleteByUserId`, `deleteByExpiresAtBefore`, `incrementResendCount`, `countResendsWithinWindow`

**`ExpiredVerificationTokenReaperJob`** — `@Component`

- `cleanup()`: `@Scheduled(cron="0 0 * * * *")` + `@SchedulerLock` — hourly deletion of expired tokens

---

### `invitation/`

**`TenantInvitation`** — domain class: `id`, `tenantKey`, `invitedEmail`, `invitedBy`, `authority`, `token`, `status`, `expiresAt`, `acceptedAt`, audit fields. Has `isPending()` helper.

**`InvitationStatus`** — enum: `PENDING`, `ACCEPTED`, `REVOKED`, `EXPIRED`

**`InvitationDtos`** — records:

- `SendInvitationRequest`: `@Email email`, `@Pattern(ADMIN|MEMBER) authority`
- `AcceptInvitationRequest`: `firstName`, `lastName`, `@NotBlank password`
- `InvitationResponse`, `InvitationPreviewResponse`, `AcceptInvitationResponse`

**`InvitationService`** — interface: `sendInvitation`, `previewInvitation`, `acceptInvitation`, `revokeInvitation`, `listInvitations`

**`InvitationServiceImpl`** — `@Service @Transactional`

- `sendInvitation`: only available in `MULTI_TENANT` mode; verifies inviter has `TENANT_OWNER` or `ADMIN`; guards against duplicate PENDING invites and existing membership; generates 32-byte hex token; publishes `user.invited` event + `INVITATION` notification email
- `previewInvitation`: public (no auth); returns tenant name, email, authority, expiry, `requiresSignup` flag
- `acceptInvitation`: validates pending token; resolves or creates user (existing user: verifies password with lockout protection; new user: creates with `emailVerified=true`); creates membership + grants authority; consumes invitation; issues JWT token pair
- `revokeInvitation`: only inviter, `TENANT_OWNER`, or `ADMIN` can revoke; sets status to `REVOKED`
- `listInvitations`: returns all `PENDING` invitations for a tenant

**`InvitationRestResource`** — `@RestController`

- `POST /api/v1/iam/tenants/{tenantKey}/invitations` — `@PreAuthorize("hasAnyAuthority('TENANT_OWNER', 'ADMIN')")`
- `GET /api/v1/iam/tenants/{tenantKey}/invitations` — same authority
- `DELETE /api/v1/iam/tenants/{tenantKey}/invitations/{invitationId}` — same authority
- `GET /api/v1/iam/invitations/{token}` — public
- `POST /api/v1/iam/invitations/{token}/accept` — public

**`InvitationReaperJob`** — `@Component`: `@Scheduled(fixedDelayString="${iqkv.invitation.reaper-interval:PT1H}")` + `@SchedulerLock` — bulk-expires stale PENDING invitations

**`InvitationMapper`** — `@Mapper`: `insert`, `findByToken`, `findById`, `findPendingByTenantKey`, `findByTenantKey`, `existsPendingForTenantAndEmail`, `updateStatus`, `markAccepted`, `expireStale`

**`InvitationDtoMapper`** — static utility: `toResponse`, `toPreviewResponse`

---

### `lockout/`

**`FailedLoginAttempt`** — domain class: `id`, `email`, `attemptedAt`

**`FailedLoginAttemptMapper`** — `@Mapper`: `insert`, `countByEmailAndAttemptedAtAfter`, `deleteByEmail`

**`AccountLockoutManager`** — `@Component`

- Reads `loginAttempts` threshold and `lockoutDuration` from `AuthConfigurationProperties`
- `recordFailedAttempt(email)`: inserts a failed attempt record
- `isLocked(email)`: counts attempts within the lockout window; returns true if ≥ threshold
- `reset(email)`: deletes all failed attempts for the email

---

### `membership/`

**`MembershipStatus`** — enum: `ACTIVE`, `SUSPENDED`, `REMOVED`

**`TenantMembership`** — domain class: `id`, `userId`, `tenantKey`, `status`, audit fields

**`TenantMemberAuthority`** — domain class: `id`, `membershipId`, `authority`

**`TenantMembershipMapper`** — `@Mapper`: `insert`, `findByUserIdAndTenantKey`, `existsByUserIdAndTenantKey`, `findByTenantKey`, `findByUserId`, `deleteById`

**`TenantMemberAuthorityMapper`** — `@Mapper`: `insert`, `findByMembershipId`, `findAuthorityValuesByMembershipId`, `deleteByMembershipId`

**`MembershipService`** — interface: `resolveMembership(userId, tenantKey)`, `getAuthorities(membershipId)`

**`MembershipServiceImpl`** — `@Service @Transactional(readOnly=true)`

- `resolveMembership`: finds membership, throws `MembershipNotFoundException` if SUSPENDED or REMOVED
- `getAuthorities`: returns list of authority strings for a membership

---

### `passwordreset/`

**`PasswordResetToken`** — domain class: `id`, `userId`, `token`, `expiresAt`, `createdAt`

**`PasswordResetTokenMapper`** — `@Mapper`: `insert`, `findByToken`, `deleteByToken`, `deleteByUserId`, `deleteByExpiresAtBefore`, `countByUserIdAndCreatedAtAfter`

**`PasswordResetService`** — interface: `initiate(email)`, `complete(token, newPassword)`

**`PasswordResetServiceImpl`** — `@Service @Transactional`

- `initiate`: silently ignores unknown emails (prevents enumeration); rate-limits (configurable max requests per window); generates 32-byte hex token; publishes `PASSWORD_RESET_INITIATE` notification
- `complete`: validates token + expiry; enforces password complexity (regex: lowercase + uppercase + digit + special char, 8–128 chars); updates password hash; deletes token; sets `last_global_signout_at` (invalidates all sessions); publishes `PASSWORD_RESET_CONFIRMED` notification

**`PasswordResetRestResource`** — `@RestController` at `/api/v1/iam/users/password`

- `POST /forgot` → always returns 200
- `POST /reset` → returns 200 on success, 400 on invalid/expired token

**`ExpiredPasswordResetTokenReaperJob`** — `@Component`: `@Scheduled(cron="0 0 * * * *")` + `@SchedulerLock` — hourly cleanup

---

### `security/`

**`JwtClaimNames`** — constants class: `SUB`, `ISS`, `IAT`, `EXP`, `JTI`, `TYPE`, `USER_ID`, `USERNAME`, `EMAIL`, `FIRST_NAME`, `LAST_NAME`, `TENANT_ID`, `AUTHORITIES`, `EMAIL_VERIFIED`, `TYPE_ACCESS`, `TYPE_REFRESH`, `ISSUER`

**`JwtAuthenticationFilter`** — `@Component extends OncePerRequestFilter`

- Runs before Spring Security's `BearerTokenAuthenticationFilter`
- Checks two revocation conditions: (1) JTI in `token_denylist`; (2) token `iat` ≤ `last_global_signout_at`
- Returns 401 `application/problem+json` if either condition is met
- Skips public paths (signup, signin, refresh, validate, email verify, password reset)

---

### `signup/`

**`SignupStrategy`** — interface: `execute(RegisterUserRequest)` → `SignupResult`

**`SignupResult`** — record: `user`, `tenant`, `membership`, `authorities`

**`MultiTenantSignupStrategy`** — `@Service @ConditionalOnProperty(rollout-mode=MULTI_TENANT)`

- Upserts user by email (atomic, eliminates TOCTOU)
- Creates new tenant with NanoID 8-char key, status `PROVISIONING`
- Creates membership with `ACTIVE` status
- Grants `TENANT_OWNER` authority
- Publishes `tenant.created` event with owner fields

**`SingleTenantSignupStrategy`** — `@Service @ConditionalOnProperty(rollout-mode=SINGLE_TENANT)`

- Ignores `tenantName` field (logs debug message)
- Upserts user by email
- Resolves default tenant via `DefaultTenantResolver`
- Guards against duplicate membership
- Creates membership with `ACTIVE` status
- Grants `MEMBER` authority (not `TENANT_OWNER`)
- Does NOT publish `tenant.created` event

---

### `tenancy/`

**`TenantContext`** — `ThreadLocal<String>` holder: `setCurrentTenant`, `getCurrentTenant` (throws `IllegalStateException` if not set), `clear`

**`TenantExtractionFilter`** — `@Component @Order(HIGHEST_PRECEDENCE+1) extends OncePerRequestFilter`

- Priority 1: `X-Tenant-ID` header
- Priority 2: JWT `tenant_id` claim from Bearer token
- Returns 400 if tenant cannot be resolved
- Always clears `TenantContext` in `finally` block
- Skips public paths (signup, validate, email verify, password reset)

**`MyBatisSchemaInterceptor`** — `@Intercepts(StatementHandler.prepare)` — sets PostgreSQL `search_path` to `t_{tenantKey}, public` before each MyBatis statement when tenant context is active; validates tenant key against `^[a-zA-Z0-9_]+$`

**`TenantLiquibaseRunner`** — `@Component ApplicationRunner @ConditionalOnProperty(tenant-runner-enabled=true)`

- `run()`: runs system schema migrations on `public` schema at startup
- `runMigrationsForTenant(tenantKey)`: creates `t_{tenantKey}` schema and runs tenant-specific Liquibase changelog

---

### `tenant/`

**`Tenant`** — domain class: `id`, `tenantKey`, `name`, `status`, `isDefault`, `tenantModeOrigin`, audit fields

**`TenantStatus`** — enum: `PROVISIONING`, `ACTIVE`, `SUSPENDED`, `DELETED`, `PROVISIONING_FAILED`

**`TenantMapper`** — `@Mapper`: `insertIfAbsent`, `findByTenantKey`, `findByStatus`, `existsByName`, `updateStatus`, `findStuckProvisioning`, `findOwnerByTenantKey`, `findDefaultTenant`, `markDefaultTenant`, `insertIfAbsentDefault`

**`TenantService`** — interface: `createTenant`, `getTenantByKey`, `updateTenantStatus`, `retryProvisioning`

**`TenantServiceImpl`** — `@Service @Transactional`

- `createTenant`: checks name uniqueness, generates NanoID key, inserts with `PROVISIONING` status, publishes `tenant.created` event
- `getTenantByKey`: read-only lookup
- `updateTenantStatus`: enforces allowed transitions (`ACTIVE↔SUSPENDED`, `ACTIVE/SUSPENDED/PROVISIONING_FAILED→DELETED`); publishes `tenant.suspended` or `tenant.deleted` events
- `retryProvisioning`: only from `PROVISIONING_FAILED` state; re-publishes `tenant.created` event

**`TenantRestResource`** — `@RestController` at `/api/v1/iam/tenants`

- `GET /{tenantKey}` — `@PreAuthorize("hasAuthority('TENANT_OWNER')")`
- `PATCH /{tenantKey}/status` — `@PreAuthorize("hasAuthority('TENANT_OWNER')")`
- `POST /{tenantKey}/retry-provisioning` — `@PreAuthorize("hasAuthority('TENANT_OWNER')")`

**`TenantProvisioningConsumer`** — `@Component @ConditionalOnProperty(rabbitmq.enabled=true)`

- `@RabbitListener(TENANT_PROVISIONING_QUEUE)`: runs `TenantLiquibaseRunner.runMigrationsForTenant()`, sets status to `ACTIVE`, publishes `tenant.provisioned`; on failure sets `PROVISIONING_FAILED`, publishes `tenant.provisioning_failed`

**`StuckTenantReaperJob`** — `@Component`: `@Scheduled(cron="0 */5 * * * *")` + `@SchedulerLock` — every 5 minutes marks tenants stuck in `PROVISIONING` beyond configured timeout as `PROVISIONING_FAILED`

**`DefaultTenantResolver` / `DefaultTenantResolverImpl`** — resolves the default tenant key for `SINGLE_TENANT` mode

**`MultiTenantBootstrapStrategy` / `SingleTenantBootstrapStrategy`** — mode-specific bootstrap logic (conditional on rollout mode)

---

### `user/`

**`AccountStatus`** — enum: `ACTIVE` (only one value currently)

**`User`** — domain class: `id`, `email`, `passwordHash`, `firstName`, `lastName`, `status`, `emailVerified`, `lastGlobalSignoutAt`, audit fields

**`UserMapper`** — `@Mapper`: `upsertByEmail`, `findById`, `findByEmail`, `existsByEmail`, `findAll(limit, offset)`, `countAll`, `update`, `deleteById`, `updateLastGlobalSignoutAt`, `findLastGlobalSignoutAt`, `setEmailVerified`, `updatePassword`

**`UserService`** — interface: `registerUser`, `getUserById`, `listUsers`, `createUser`, `updateUser`, `patchUser`, `deleteUser`, `deleteUserById`

**`UserServiceImpl`** — `@Service @Transactional`

- `registerUser`: delegates to `SignupStrategy`, publishes `user.created` event, generates email verification token, publishes `VERIFY_EMAIL` notification
- `listUsers`: paginated, returns `PagedUserResponse`
- `createUser`: admin path — creates user with random temp password
- `patchUser`: partial update (firstName, lastName, status)
- `updateUser`: full name replacement
- `deleteUser`: removes membership (not the user account), publishes `user.removed` event
- `deleteUserById`: hard-deletes user record

**`UserRestResource`** — `@RestController` at `/api/v1/iam/users`

- `GET /me` — `@PreAuthorize("isAuthenticated()")`
- `PATCH /me` — update own profile
- `DELETE /me` — remove own membership from current tenant
- `POST /tenants` — credential-gated tenant discovery (no JWT required)

**`UserOperatorRestResource`** — `@RestController` at `/api/v1/iam/operator/users`

- `GET /` — paginated list (page, size params, max 100)
- `GET /{id}` — get by UUID
- `POST /` — create user
- `PUT /{id}` — full replace
- `PATCH /{id}` — partial update
- `DELETE /{id}` — hard delete

---

### `infrastructure/config/`

**`SecurityConfig`** — `@Configuration @EnableWebSecurity`

- Stateless, CSRF disabled
- Public paths: actuator, api-docs, swagger-ui, `.well-known`, signup, signin, refresh, validate, email verify, password reset, admin users, invitation accept flow
- `TENANT_OWNER` required for tenant management endpoints
- OAuth2 Resource Server with RSA public key loaded from PEM file
- Registers `JwtAuthenticationFilter` before `BearerTokenAuthenticationFilter`
- `PasswordEncoder`: BCrypt strength 12

**`AuthConfigurationProperties`** — `@ConfigurationProperties("iqkv.auth")`: JWT (privateKeyPath, publicKeyPath, expiry, refreshExpiry, issuer), Security (passwordEncoderStrength, minLength, RateLimiting{loginAttempts, lockoutDuration}), PasswordReset (tokenTtl, rateLimitWindow, rateLimitMaxRequests)

**`PlatformConfigurationProperties`** — `@ConfigurationProperties("iqkv.platform")`: `rolloutMode` (MULTI_TENANT | SINGLE_TENANT)

**`RolloutMode`** — enum: `MULTI_TENANT`, `SINGLE_TENANT`

**`PlatformModeValidator` / `PlatformModeValidatorImpl`** — `@Component @Order(HIGHEST_PRECEDENCE) ApplicationRunner`: validates `rolloutMode` is set at startup; throws `InvalidPlatformModeException` if missing

**`RabbitMQConfig`** — `@Configuration @ConditionalOnProperty(rabbitmq.enabled=true) @Profile("!test")`

- Exchange: `iqkv.events` (topic), `iqkv.dlx` (dead-letter)
- Queues: `iqkv.iam.user.events`, `iqkv.iam.notifications`, `iqkv.iam.tenant.provisioning`, `iqkv.iam.subscription.events`
- Routing keys: `tenant.created`, `tenant.provisioned`, `tenant.provisioning_failed`, `tenant.updated`, `tenant.deleted`, `tenant.suspended`, `user.created`, `user.updated`, `user.deleted`, `user.removed`, `user.invited`, `subscription.cancelled`, `notification.iam.email`
- All queues have 24h TTL and dead-letter exchange

**`ShedLockConfig`** — `@Configuration`: `JdbcTemplateLockProvider` using DB time for distributed scheduler locking

**`InvitationConfigurationProperties`** — invitation TTL config

**`LiquibaseConfigurationProperties`** — Liquibase contexts config

**`TenancyConfigurationProperties`** — provisioning timeout config

**`PlatformModeHealthIndicator`** — exposes platform mode to `/actuator/health`

**`PlatformModeInfoContributor`** — exposes `platform.rollout-mode` to `/actuator/info` (consumed by Gateway's `PlatformModeGuardFilter`)

---

### `infrastructure/messaging/`

**`MessagingService`** — `@Service`: publishes to `iqkv.events` exchange — `publishTenantCreated/Provisioned/ProvisioningFailed/Suspended/Deleted/Updated`, `publishUserEvent`, `publishUserInvited`, `publishNotification`

**`UserEventPublisher`** — `@Component`: `publishUserCreated`, `publishUserDeleted`, `publishUserRemoved`

**`UserEventListener`** — `@Component`: `@RabbitListener(USER_EVENTS_QUEUE)` — logs received user events

**`NotificationConsumer`** — `@Component`: `@RabbitListener(NOTIFICATIONS_QUEUE)` — dispatches to `EmailService`; swallows exceptions (routes to DLQ)

**`SubscriptionEventConsumer`** — `@Component @ConditionalOnProperty(rabbitmq.enabled=true)`: `@RabbitListener(SUBSCRIPTION_EVENTS_QUEUE)` — on `subscription.cancelled`, suspends the tenant via `TenantService.updateTenantStatus(SUSPENDED)`

---

### `infrastructure/security/`

**`JwksRestResource`** — `@RestController` at `/.well-known/jwks.json`: exposes RSA public key as JWKS for downstream JWT verification

**`CorrelationIdFilter`** — `@Component @Order(HIGHEST_PRECEDENCE)`: generates/propagates `X-Correlation-ID` header; puts in MDC

---

### `infrastructure/metrics/`

**`UserServiceMetrics`** — `@Component`: Micrometer counters `auth.success` (tag: tenantId), `auth.failure` (tags: tenantId, reason), `tenant.created`; timer `auth.duration` (tag: tenantId)

---

## Gateway Service (`foundation-gateway-service`)

### `infrastructure/config/`

**`SecurityConfig`** — `@Configuration @EnableWebFluxSecurity`

- Reactive (WebFlux) security
- CSRF, HTTP Basic, form login all disabled
- Public paths loaded from `GatewayProperties.publicPaths`
- All other exchanges require authentication
- OAuth2 Resource Server with JWKS endpoint (from IAM service)
- Custom `ReactiveJwtAuthenticationConverter` extracts `authorities` claim → `SimpleGrantedAuthority`
- Returns 401 on auth failure, 403 on access denied

**`GatewayProperties`** — `@ConfigurationProperties("iqkv.gateway")`: `publicPaths: List<String>`

**`GatewayConfigurationProperties`** — nested config class:

- `Tenancy @ConfigurationProperties("iqkv.tenancy")`: `defaultTenantKey`
- `Iam @ConfigurationProperties("iqkv.iam")`: `serviceUrl` (default: `http://foundation-iam-service:8080`)

**`PlatformConfigurationProperties`** — `@ConfigurationProperties("iqkv.platform")`: `rolloutMode`

**`RolloutMode`** — enum: `MULTI_TENANT`, `SINGLE_TENANT`

**`PlatformModeValidator` / `PlatformModeValidatorImpl`** — `@Component @Order(HIGHEST_PRECEDENCE) ApplicationRunner`: validates rollout mode at startup; on failure sets readiness to `REFUSING_TRAFFIC`

**`PlatformModeGuardFilter`** — `@Component implements GlobalFilter, Ordered` (order: `HIGHEST_PRECEDENCE`)

- On `@PostConstruct` and every 60 seconds: queries IAM `/actuator/info` for `platform.rollout-mode`
- If mismatch detected: sets `modeMismatchDetected=true`, publishes `ReadinessState.REFUSING_TRAFFIC`, returns 503 on all requests
- If IAM unreachable: fail-open (logs warning, allows traffic)
- If mismatch resolved: restores `ACCEPTING_TRAFFIC`

---

### `infrastructure/security/`

**`CorrelationIdFilter`** — `GlobalFilter, Ordered` (order: `-200`)

- Generates or propagates `X-Correlation-ID` header
- Stores in exchange attributes for later use by `ResponseTransformationFilter`

**`HeaderSanitizationFilter`** — `GlobalFilter, Ordered` (order: `-190`)

- Strips all user/tenant context headers from incoming client requests to prevent identity spoofing
- Removed headers: `X-User-ID`, `X-Username`, `X-User-Email`, `X-User-Authorities`, `X-User-Permissions`, `X-Tenant-ID`, `X-Organization-ID`

**`JwtContextPropagationFilter`** — `GlobalFilter, Ordered` (order: `-100`)

- Reads validated JWT from `ReactiveSecurityContextHolder`
- Adds downstream headers: `X-User-ID` (userId claim), `X-Username`, `X-User-Email`, `X-Tenant-ID` (tenant_id claim), `X-User-Authorities` (comma-separated)

**`TenantContextResolutionPolicy`** — interface: `resolveTenantContext(exchange)` → `Optional<String>`

**`TenantContextFilter`** — `GlobalFilter, Ordered, TenantContextResolutionPolicy` (order: `-50`)

- Runs after JWT propagation
- `MULTI_TENANT` mode: uses `X-Tenant-ID` already set by JWT propagation; no auto-injection
- `SINGLE_TENANT` mode: if `X-Tenant-ID` absent, injects `iqkv.tenancy.default-tenant-key`; logs warning if not configured

**`ResponseTransformationFilter`** — `GlobalFilter, Ordered` (order: `Integer.MIN_VALUE + 1`)

- Adds security response headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `X-XSS-Protection: 1; mode=block`, `Referrer-Policy: strict-origin-when-cross-origin`
- Echoes `X-Correlation-ID` back to client

**Filter execution order summary:**

1. `CorrelationIdFilter` (-200): generate/propagate correlation ID
2. `HeaderSanitizationFilter` (-190): strip spoofable headers
3. Spring Security OAuth2 validation (validates JWT via JWKS)
4. `JwtContextPropagationFilter` (-100): extract JWT claims → downstream headers
5. `TenantContextFilter` (-50): inject/validate tenant context
6. Route to downstream service
7. `ResponseTransformationFilter` (MIN_VALUE+1): add security response headers

---

## Billing Service (`foundation-billing-service`)

### `plan/`

**`Plan`** — domain class: `id`, `planCode`, `displayName`, `billingPeriod` (MONTHLY|ANNUAL), `priceMinor` (cents), `currency`, `featureSet` (JSON string), `scope` (TENANT|USER), `active`

**`PlanRequest`** — record (request body): `@NotBlank planCode`, `displayName`, `billingPeriod`, `@Positive priceMinor`, `currency`, `featureSet`, `@NotBlank scope`, `active`

**`PlanCatalogRestResource`** — `@RestController` at `/api/v1/billing/plans`

- `GET /` — list all active plans (any authenticated user)
- `GET /{planCode}` — get by planCode (any authenticated user)
- `POST /` — `@PreAuthorize("hasAuthority('PLATFORM_OPERATOR')")` — create plan
- `PUT /{planCode}` — `@PreAuthorize("hasAuthority('PLATFORM_OPERATOR')")` — replace plan
- `DELETE /{planCode}` — `@PreAuthorize("hasAuthority('PLATFORM_OPERATOR')")` — soft-delete (sets `active=false`)

**`PlanEligibilityPolicy`** — interface: `validatePlanEligibility(planCode, subjectType)` — throws `PlanNotFoundException` or `PlanScopeMismatchException`

**`PlanEligibilityPolicyImpl`** — `@Service`: validates plan `scope` matches `SubjectType` (TENANT plan can only be assigned to TENANT subject)

---

### `settings/`

**`BillingSettings`** — domain class: `id`, `tenantKey`, `externalCustomerId` (Stripe `cus_xxx`), `billingEmail`, `companyName`, `billingAddress` (JSONB as String), `taxId`, `taxIdType`, `currency`, `profileOwnerId` (soft ref to IAM user, no FK)

**`BillingSettingsDtos`** — `UpdateBillingSettingsRequest` record, `BillingSettingsResponse` record

**`BillingSettingsRestResource`** — `@RestController` at `/api/v1/billing/settings`

- `GET /{tenantKey}` — `@PreAuthorize("hasAuthority('TENANT_OWNER')")` — validates JWT tenant matches path
- `PATCH /{tenantKey}` — `@PreAuthorize("hasAuthority('TENANT_OWNER')")` — partial update, validates tenant match

**`BillingSettingsService`** — `@Service`

- `getByTenantKey`: throws `ResourceNotFoundException` if not found
- `update`: validates tenant context match; applies non-null fields; publishes `BILLING_UPDATED` notification

---

### `subscription/`

**`Subscription`** — domain class: `id`, `tenantKey`, `externalSubscriptionId`, `externalCustomerId`, `status` (active|past_due|canceled|unpaid|trialing), `planId`, `currentPeriodStart`, `currentPeriodEnd`, `cancelAtPeriodEnd`, `canceledAt`, `subjectType` (TENANT|USER), `subjectKey`

**`SubjectType`** — enum: `TENANT`, `USER`

**`SubscriptionSubject`** — record: `type`, `key`

**`SubscriptionSubjectResolver`** — interface:

Now I have a complete, precise picture of what's actually implemented.
