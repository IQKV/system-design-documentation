# Tenancy Deep Dive

This document outlines the implementation details of the IQKV **Hybrid Tenancy Model**, focusing on how the platform maintains architectural consistency between Multi-Tenant (SaaS) and Single-Tenant (Managed) deployments.

---

## 1. Hybrid Tenancy Philosophy

The core goal of the platform is **Architectural Symmetry**.

- In **Multi-Tenant mode**, the organization is a visible, primary entity.
- In **Single-Tenant mode**, the organization is a hidden, background entity.

Regardless of the mode, the **data path** is identical:
`API Request` → `Gateway (Context Injection)` → `Service (Context Propagation)` → `Persistence (Tenant Routing)`.

### Implementation Architecture

The tenancy system is built on several key components:

- **TenantContext**: Thread-local storage for current tenant key
- **MyBatisSchemaInterceptor**: Automatic PostgreSQL schema switching
- **TenantContextFilter**: Gateway-level tenant resolution and injection
- **Bootstrap Strategies**: Mode-specific tenant initialization patterns

---

## 2. NanoID Resolution & Defaulting

The system enforces the use of **NanoIDs** for all tenant keys to avoid brittle, human-readable identifiers in infrastructure (e.g., schema names).

### NanoID Generation

```java
// 8-character NanoID using alphabet [a-z0-9]
String tenantKey = NanoIdUtils.randomNanoId(NanoIdUtils.DEFAULT_NUMBER_GENERATOR, 
    "abcdefghijklmnopqrstuvwxyz0123456789".toCharArray(), 8);
// Example: "abc12345"
```

### The "Hidden" Default Tenant

In Single-Tenant mode, the system operates with a single "Master" workspace. To ensure this key is stable across deployments:

1. **Configuration-Based:** The default tenant key is configured via `iqkv.tenancy.default-tenant-key`
2. **Database Resolution:** On startup, the IAM service checks the `public.tenants` table for a record marked with `is_default: true`
3. **Auto-Generation:** If no default tenant exists, the bootstrap process creates one with a deterministic key
4. **Implicit Enrollment:** During the `POST /signup` flow, if the mode is `SINGLE_TENANT`, the backend automatically creates a `TenantMembership` for the user against the resolved Default NanoID

### Default Tenant Resolution Strategy

```java
@Component
public class DefaultTenantResolverImpl implements DefaultTenantResolver {
    
    public String resolveDefaultTenantKey() {
        // 1. Check configuration
        if (hasConfiguredKey()) {
            return tenancyProps.getDefaultTenantKey();
        }
        
        // 2. Query database for existing default
        Optional<Tenant> defaultTenant = tenantMapper.findDefaultTenant();
        if (defaultTenant.isPresent()) {
            return defaultTenant.get().getTenantKey();
        }
        
        // 3. Create new default tenant
        return createDefaultTenant();
    }
}
```

---

## 3. Bootstrapping Workflow

To ensure the system is ready for the first user, the IAM service performs an automated **Bootstrap** on its first-ever startup using pluggable bootstrap strategies:

### Multi-Tenant Bootstrap Strategy

1. **No Pre-provisioning:** No default tenant is created
2. **Per-Signup Provisioning:** Each signup creates a new tenant with `TENANT_OWNER` authority
3. **Async Processing:** Tenant provisioning happens via RabbitMQ messaging

### Single-Tenant Bootstrap Strategy

1. **Detection:** Checks `ROLLOUT_MODE=SINGLE_TENANT`
2. **Default Tenant Creation:** Generates/resolves the Default NanoID and inserts it into the Platform Registry with `status: PROVISIONING`
3. **Schema Provisioning:** Creates PostgreSQL schema `t_{tenantKey}` and runs Liquibase migrations
4. **Status Transition:** Tenant status transitions from `PROVISIONING` → `ACTIVE`
5. **Initial Admin:** If environment variables for a root admin are provided (e.g., `INITIAL_ADMIN_EMAIL`), the bootstrap process creates that user and links them to the default tenant with `MEMBER` authority

### Bootstrap Implementation

```java
@Component
@ConditionalOnProperty(name = "iqkv.platform.rollout-mode", havingValue = "SINGLE_TENANT")
public class SingleTenantBootstrapStrategy implements TenantBootstrapStrategy {
    
    @EventListener(ApplicationReadyEvent.class)
    public void bootstrap() {
        if (!defaultTenantExists()) {
            String tenantKey = createDefaultTenant();
            provisionTenantSchema(tenantKey);
            createInitialAdminIfConfigured(tenantKey);
        }
    }
}
```

---

## 4. Schema Isolation Implementation

### PostgreSQL Schema-Per-Tenant

Each tenant gets its own PostgreSQL schema with the naming pattern `t_{tenantKey}`:

- **System Schema:** `public` - Contains platform-wide data (users, tenants, memberships)
- **Tenant Schemas:** `t_abc12345` - Contains tenant-specific data (business entities)

### MyBatis Schema Interceptor

Automatic schema switching is handled by a MyBatis interceptor:

```java
@Intercepts({
    @Signature(type = StatementHandler.class, method = "prepare", 
               args = {Connection.class, Integer.class})
})
public class MyBatisSchemaInterceptor implements Interceptor {
    
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Connection connection = (Connection) invocation.getArgs()[0];
        
        try {
            String tenantKey = TenantContext.getCurrentTenant();
            String schema = "t_" + tenantKey;
            
            try (PreparedStatement stmt = connection.prepareStatement(
                    "SET search_path TO " + schema + ", public")) {
                stmt.execute();
            }
        } catch (IllegalStateException e) {
            // No tenant context - use public schema
        }
        
        return invocation.proceed();
    }
}
```

### Liquibase Multi-Tenant Migrations

Separate changelog files for system and tenant schemas:

- **System Changelog:** `db/changelog/system/db.changelog-master.xml`
- **Tenant Changelog:** `db/changelog/tenant/master.xml`

```java
public class TenantLiquibaseRunner {
    
    public void runMigrationsForTenant(String tenantKey) throws Exception {
        String schema = "t_" + tenantKey;
        
        try (Connection connection = dataSource.getConnection()) {
            // Create schema if not exists
            try (Statement stmt = connection.createStatement()) {
                stmt.execute("CREATE SCHEMA IF NOT EXISTS " + schema);
                stmt.execute("SET search_path TO " + schema);
            }
            
            // Run Liquibase migrations
            Database database = DatabaseFactory.getInstance()
                .findCorrectDatabaseImplementation(new JdbcConnection(connection));
            database.setDefaultSchemaName(schema);
            
            try (Liquibase liquibase = new Liquibase(TENANT_CHANGELOG, 
                    new ClassLoaderResourceAccessor(), database)) {
                liquibase.update(new Contexts(), new LabelExpression());
            }
        }
    }
}
```

---

## 5. Gateway Tenant Context Resolution

The API Gateway handles tenant context resolution differently based on rollout mode:

### Multi-Tenant Mode

1. **JWT Extraction:** Reads `tenant_id` claim from validated JWT
2. **Header Propagation:** Sets `X-Tenant-ID` header for downstream services
3. **No Auto-Injection:** Missing tenant context results in 401/403 errors

### Single-Tenant Mode

1. **JWT Extraction:** Reads `tenant_id` claim from validated JWT (if present)
2. **Default Injection:** If no tenant context, injects configured default tenant key
3. **Seamless Operation:** Unauthenticated requests (like signup) get default tenant context

### Implementation

```java
@Component
public class TenantContextFilter implements GlobalFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Optional<String> resolved = resolveTenantContext(exchange);
        
        if (resolved.isEmpty()) {
            return chain.filter(exchange); // Multi-tenant: no injection
        }
        
        String tenantKey = resolved.get();
        ServerHttpRequest mutated = exchange.getRequest().mutate()
            .header("X-Tenant-ID", tenantKey)
            .build();
            
        return chain.filter(exchange.mutate().request(mutated).build());
    }
    
    private Optional<String> resolveTenantContext(ServerWebExchange exchange) {
        String existingTenantId = exchange.getRequest().getHeaders().getFirst("X-Tenant-ID");
        
        // Honor existing JWT-derived tenant context
        if (existingTenantId != null && !existingTenantId.isBlank()) {
            return Optional.of(existingTenantId);
        }
        
        if (platformConfig.rolloutMode() == RolloutMode.MULTI_TENANT) {
            return Optional.empty(); // No auto-injection
        }
        
        // Single-tenant: inject default tenant key
        return Optional.ofNullable(tenancyProperties.getDefaultTenantKey());
    }
}
```

---

## 6. UI/UX Adaptation

The UI remains "Tenancy-Blind" in Single-Tenant mode by observing **Capability Flags** provided by the IAM service:

| Feature         | Multi-Tenant Behavior          | Single-Tenant Behavior           |
| :-------------- | :----------------------------- | :------------------------------- |
| **Signup Form** | Asks for "Organization Name".  | Only asks for User details.      |
| **Login Flow**  | May require selecting an Org.  | Directly enters the default Org. |
| **Dashboard**   | Shows "Organization" switcher. | Switcher is hidden.              |
| **Settings**    | "Organization Management".     | "Workspace Configuration".       |

### Platform Mode Detection

The UI can detect the platform mode via the IAM service's actuator endpoint:

```json
GET /actuator/info
{
  "platform": {
    "rollout-mode": "SINGLE_TENANT"
  }
}
```

---

## 7. Async Tenant Provisioning

### RabbitMQ Event Flow

1. **Tenant Creation:** IAM service creates tenant record with `status: PROVISIONING`
2. **Event Publishing:** Publishes `tenant.provisioning.requested` event to RabbitMQ
3. **Consumer Processing:** `TenantProvisioningConsumer` handles schema creation
4. **Status Update:** Updates tenant status to `ACTIVE` or `PROVISIONING_FAILED`
5. **Event Publishing:** Publishes `tenant.provisioned` or `tenant.provisioning.failed` events

### Stuck Tenant Reaper

ShedLock-protected job that handles stuck provisioning:

```java
@Component
public class StuckTenantReaperJob {
    
    @Scheduled(cron = "0 */5 * * * *")
    @SchedulerLock(name = "StuckTenantReaperJob.reapStuckTenants")
    public void reapStuckTenants() {
        Instant cutoff = Instant.now().minus(tenancyProps.provisioningTimeout());
        List<Tenant> stuckTenants = tenantMapper.findStuckProvisioning(cutoff);
        
        for (Tenant tenant : stuckTenants) {
            tenantMapper.updateStatus(tenant.getTenantKey(), 
                TenantStatus.PROVISIONING_FAILED.name(), LocalDateTime.now());
            messagingService.publishTenantProvisioningFailed(tenant.getTenantKey());
        }
    }
}
```

---

## 8. Migration: Single to Multi

Because the Single-Tenant mode uses the exact same `schema-per-tenant` logic as the SaaS mode, migrating a customer from an "Internal Tool" to a "Public Platform" is a **Zero-Migration** event:

1. **Configuration Change:** Change `ROLLOUT_MODE` from `SINGLE_TENANT` to `MULTI_TENANT`
2. **Existing Users:** Remain in the `default` tenant with existing authorities
3. **New Registrations:** Trigger the standard "Create Organization" flow, spawning new schemas alongside the original one
4. **Gateway Behavior:** Stops auto-injecting default tenant context for new requests
5. **UI Adaptation:** Shows organization management features for new users

### Migration Checklist

- [ ] Update `ROLLOUT_MODE` configuration across all services (IAM, Gateway, Billing)
- [ ] Restart services to pick up new configuration
- [ ] Verify platform mode consistency via actuator endpoints
- [ ] Test new user signup flow creates separate tenants
- [ ] Verify existing users can still access their data
- [ ] Update UI configuration to show multi-tenant features

---

## 9. Configuration

### Platform Rollout Mode

The platform uses a single configuration variable to control tenancy behavior:

**Environment Variable:** `ROLLOUT_MODE`  
**Configuration Path:** `iqkv.platform.rollout-mode`  
**Valid Values:**

- `MULTI_TENANT` - Multi-tenant SaaS mode (default)
- `SINGLE_TENANT` - Single-tenant managed mode

**Example Configuration:**

```yaml
# application.yml
iqkv:
  platform:
    rollout-mode: ${ROLLOUT_MODE:MULTI_TENANT}
```

**Helm Values:**

```yaml
# values.yaml
platform:
  rolloutMode: "MULTI_TENANT"
  defaultTenantKey: "" # Required for SINGLE_TENANT mode
  defaultTenantName: "Default Organization"
```

### Tenancy Configuration

Additional tenancy settings control schema isolation and tenant provisioning:

```yaml
# application.yml
iqkv:
  tenancy:
    schema-prefix: t_ # Prefix for tenant schemas
    default-schema: public # Default schema for shared data
    provisioning-timeout: PT10M # Timeout for tenant provisioning
    default-tenant-key: ${DEFAULT_TENANT_KEY:}
    default-tenant-name: ${DEFAULT_TENANT_NAME:Default Organization}
  liquibase:
    system-change-log: db/changelog/system/db.changelog-master.xml
    tenant-change-log: db/changelog/tenant/master.xml
    tenant-runner-enabled: true
```

### Gateway Configuration

Gateway-specific tenancy settings:

```yaml
# gateway application.yml
iqkv:
  platform:
    rollout-mode: ${ROLLOUT_MODE:MULTI_TENANT}
  tenancy:
    default-tenant-key: ${DEFAULT_TENANT_KEY:}
  iam:
    service-url: ${IAM_SERVICE_URL:http://foundation-iam-service:8080}
```

**Key Points:**

- `ROLLOUT_MODE` is the single source of truth for platform behavior
- Tenancy behavior (schema isolation, signup flow, UI) is derived from `ROLLOUT_MODE`
- No separate "tenancy mode" configuration is needed
- Configuration must be identical across IAM, Billing, and Gateway services

### Configuration Validation

The platform validates `ROLLOUT_MODE` at startup and enforces consistency:

#### IAM Service Validation

1. **Presence Check:** Ensures `ROLLOUT_MODE` is configured
2. **Value Validation:** Verifies it's either `MULTI_TENANT` or `SINGLE_TENANT`
3. **Bootstrap Execution:** Runs appropriate bootstrap strategy based on mode

#### Gateway Mode Guard

The gateway validates mode consistency with the IAM service:

```java
@Component
public class PlatformModeGuardFilter implements GlobalFilter {
    
    @PostConstruct
    public void validateOnStartup() {
        performModeCheck();
    }
    
    @Scheduled(fixedDelay = 60_000)
    public void revalidatePeriodically() {
        performModeCheck();
    }
    
    private void performModeCheck() {
        // Query IAM /actuator/info endpoint
        // Compare local vs canonical mode
        // Block traffic on mismatch with 503 Service Unavailable
    }
}
```

**Startup Logs:**

```
Platform rollout mode validated successfully: MULTI_TENANT
Platform mode guard: local=MULTI_TENANT, canonical=MULTI_TENANT (consistent)
```

If validation fails, the service will not start and will log a clear error message:

```
Platform mode mismatch detected: local=SINGLE_TENANT, canonical=MULTI_TENANT
Service readiness set to REFUSING_TRAFFIC
```

---

## 10. Operational Considerations

### Monitoring and Observability

- **Tenant Provisioning Metrics:** Track provisioning success/failure rates
- **Schema Usage Metrics:** Monitor per-tenant database usage
- **Mode Consistency Alerts:** Alert on platform mode mismatches
- **Stuck Tenant Alerts:** Monitor tenants stuck in `PROVISIONING` status

### Backup and Recovery

- **Schema-Level Backups:** Each tenant schema can be backed up independently
- **Cross-Tenant Restore:** Restore individual tenants without affecting others
- **Migration Testing:** Test single→multi tenant migrations in staging environments

### Performance Considerations

- **Connection Pooling:** PostgreSQL connection pools handle schema switching efficiently
- **Query Performance:** Tenant-specific indexes and statistics per schema
- **Schema Limits:** Monitor total number of schemas (PostgreSQL limit: ~1000s)

### Security Implications

- **Schema Isolation:** Complete data isolation between tenants at database level
- **Privilege Separation:** Database users cannot access other tenant schemas
- **Audit Trails:** Per-tenant audit logs in separate schemas
