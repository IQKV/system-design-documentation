# Tenancy Deep Dive

This document outlines the implementation details of the IQKV **Hybrid Tenancy Model**, focusing on how the platform maintains architectural consistency between Multi-Tenant (SaaS) and Single-Tenant (Managed) deployments.

---

## 1. Hybrid Tenancy Philosophy

The core goal of the platform is **Architectural Symmetry**.

- In **Multi-Tenant mode**, the organization is a visible, primary entity.
- In **Single-Tenant mode**, the organization is a hidden, background entity.

Regardless of the mode, the **data path** is identical:
`API Request` → `Gateway (Context Injection)` → `Service (Context Propagation)` → `Persistence (Tenant Routing)`.

---

## 2. NanoID Resolution & Defaulting

The system enforces the use of **NanoIDs** for all tenant keys to avoid brittle, human-readable identifiers in infrastructure (e.g., schema names).

### The "Hidden" Default Tenant

In Single-Tenant mode, the system operates with a single "Master" workspace. To ensure this key is stable across deployments:

1. **Deterministic Seed:** The NanoID for the default tenant can be generated using a fixed seed (e.g., a system-wide `TENANCY_SEED` secret).
2. **Platform Registry lookup:** On startup, the IAM service checks the `public.tenants` table for a record marked with `is_default: true`.
3. **Implicit Enrolment:** During the `POST /signup` flow, if the mode is `SINGLE`, the backend ignores the tenant creation step and automatically creates a `TenantMembership` for the user against the resolved Default NanoID.

---

## 3. Bootstrapping Workflow

To ensure the system is ready for the first user, the IAM service performs an automated **Bootstrap** on its first-ever startup:

1. **Detection:** Checks `ROLLOUT_MODE=SINGLE_TENANT`.
2. **Creation:** Generates the Default NanoID and inserts it into the Platform Registry with `status: PROVISIONING`.
3. **Provisioning:** Publishes a `tenant.provisioned` event to RabbitMQ.
4. **Finalization:** The Provisioning Worker creates the schema/database and migrations. The tenant status transitions to `ACTIVE`.
5. **Initial Admin:** If environment variables for a root admin are provided (e.g., `INITIAL_ADMIN_EMAIL`), the bootstrap process creates that user and links them to the default tenant.

---

## 4. UI/UX Adaptation

The UI remains "Tenancy-Blind" in Single-Tenant mode by observing **Capability Flags** provided by the IAM service:

| Feature         | Multi-Tenant Behavior          | Single-Tenant Behavior           |
| :-------------- | :----------------------------- | :------------------------------- |
| **Signup Form** | Asks for "Organization Name".  | Only asks for User details.      |
| **Login Flow**  | May require selecting an Org.  | Directly enters the default Org. |
| **Dashboard**   | Shows "Organization" switcher. | Switcher is hidden.              |
| **Settings**    | "Organization Management".     | "Workspace Configuration".       |

---

## 5. Migration: Single to Multi

Because the Single-Tenant mode uses the exact same `schema-per-tenant` logic as the SaaS mode, migrating a customer from an "Internal Tool" to a "Public Platform" is a **Zero-Migration** event:

1. Change `ROLLOUT_MODE` from `SINGLE_TENANT` to `MULTI_TENANT`.
2. The existing users remain in the `default` tenant.
3. New registrations will now trigger the standard "Create Organization" flow, spawning new schemas alongside the original one.

---

## 6. Configuration

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
    provisioning-timeout: PT5M # Timeout for tenant provisioning
    default-tenant-key: ${DEFAULT_TENANT_KEY:}
    default-tenant-name: ${DEFAULT_TENANT_NAME:Default Organization}
```

**Key Points:**

- `ROLLOUT_MODE` is the single source of truth for platform behavior
- Tenancy behavior (schema isolation, signup flow, UI) is derived from `ROLLOUT_MODE`
- No separate "tenancy mode" configuration is needed
- Configuration must be identical across IAM, Billing, and Gateway services

### Configuration Validation

The platform validates `ROLLOUT_MODE` at startup:

1. **Presence Check:** Ensures `ROLLOUT_MODE` is configured
2. **Value Validation:** Verifies it's either `MULTI_TENANT` or `SINGLE_TENANT`
3. **Cross-Service Consistency:** Gateway validates its mode matches IAM service

**Startup Logs:**

```
Platform rollout mode validated successfully: MULTI_TENANT
```

If validation fails, the service will not start and will log a clear error message.
