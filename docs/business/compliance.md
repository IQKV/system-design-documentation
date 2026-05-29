# Compliance & Data Isolation

## Tenancy Modes

The platform supports two deployment modes, configured at deploy time via Helm values — no code changes required.

| Mode          | Configuration                                      | Use case                                                |
| ------------- | -------------------------------------------------- | ------------------------------------------------------- |
| Multi-tenant  | `tenancy.mode: multi` (default)                    | Standard B2B SaaS — each customer gets their own schema |
| Single-tenant | `tenancy.mode: single` + `tenancy.defaultTenant.*` | Internal tools, dedicated deployments, white-label      |

In single-tenant mode, IAM provisions one default tenant at startup. The schema isolation model is identical — the platform simply operates with one tenant instead of many. Switching from single to multi-tenant later requires no schema or code changes.

---

## Schema-Per-Tenant

Each tenant has a dedicated PostgreSQL schema within the IAM database (`foundation_iam`). The schema boundary is enforced at the database engine level, not in application code.

Practical implications:

| Concern          | How it's addressed                                                                   |
| ---------------- | ------------------------------------------------------------------------------------ |
| Data leakage     | Cross-tenant queries impossible in normal flow — wrong `search_path` returns no rows |
| GDPR erasure     | `DROP SCHEMA tenant_x CASCADE` — no orphaned records in shared tables                |
| Backup / restore | Dump or restore a single schema without affecting others                             |
| Audit scope      | One schema = one tenant — no filtering required                                      |
| Tenant migration | Move schema to dedicated instance by updating connection string — no code change     |

---

## Audit Trail

The platform includes a dedicated `foundation-audit-service` that provides a centralized, tamper-resistant activity log across all tenants.

- **Passive consumption** — binds to the `iqkv.events` RabbitMQ exchange; domain services require zero code changes for basic auditing
- **Technical context enrichment** — the API Gateway captures client IP and User-Agent and propagates them as `X-Audit-IP` / `X-Audit-UA`; the Audit Service stores these alongside every record
- **Storage isolation** — the Audit Service maintains its own dedicated PostgreSQL database, ensuring high-volume logging does not impact business-critical transactions
- **Pluggable backends** — `foundation-audit-spi` defines an `AuditProvider` interface; PostgreSQL is the default implementation; Elasticsearch or custom SIEM backends can be substituted without touching core services
- **Admin search API** — secured, paginated, filterable endpoint restricted to `PLATFORM_ADMIN`; supports filtering by user, tenant, and action type
- **JSONB metadata** — full domain event payload preserved for deep inspection and forensic queries

---

## Regulatory Alignment

This is a reference, not a certification. Actual compliance requires audit and legal review.

**GDPR** — Schema isolation supports data subject rights (access, erasure, portability). `DROP SCHEMA tenant_x CASCADE` removes all tenant data with no orphaned records. The Audit Service provides a per-tenant activity trail exportable for data subject access requests.

**SOC 2 Type II** — Schema-level isolation is a stronger control than application-level `tenant_id` filtering. The centralized Audit Service provides the activity logging required for availability and security monitoring controls. Helm-based IaC supports change management controls.

**HIPAA** — PHI can be isolated per schema. Access enforced at IAM + Gateway layer. Full audit trail via the Audit Service with IP and User-Agent enrichment. Object storage (MinIO/S3) for avatars uses presigned URLs with short TTLs — no direct public access.

**ISO 27001** — IaC (Helm) supports asset management and change control. K8s RBAC + IAM service covers access control requirements. Audit Service provides the event log required for A.12.4 (logging and monitoring).

---

## Access Control

Every request passes through the API Gateway, which validates the JWT and resolves tenant context before forwarding. No request reaches IAM, Billing, or Audit without passing this layer.

The Gateway enforces:

- **Header sanitization** — strips all `X-User-*`, `X-Tenant-ID`, and `X-Audit-*` headers from incoming client requests before JWT processing; clients cannot inject identity or audit context
- **JWT RS256 validation** — tokens validated against the IAM JWKS endpoint; expired, revoked (JTI denylist), or globally signed-out tokens are rejected
- **Context propagation** — after validation, the Gateway sets `X-User-ID`, `X-Username`, `X-User-Email`, `X-User-Authorities`, `X-Tenant-ID`, `X-Correlation-ID`, `X-Audit-IP`, and `X-Audit-UA` for downstream services

IAM enforces:

- Organization boundary (tenant isolation via schema routing)
- Role-based access (`TENANT_OWNER`, `ADMIN`, `MEMBER`, `PLATFORM_ADMIN`)
- Token revocation: JTI denylist (single session) + global signout timestamp (`last_global_signout_at`)
- Brute-force lockout: 5 failed attempts triggers a 15-minute account lock

---

## Object Storage

Avatar uploads use a two-phase presigned URL flow (IAM → MinIO/S3):

1. Client requests a presigned `PUT` URL from IAM — URL has a short TTL and is scoped to a single object key
2. Client uploads directly to object storage — IAM is not in the data path
3. Client confirms upload — IAM persists the avatar URL and deletes the previous object

No avatar data passes through the application tier. Old avatars are automatically deleted on replacement or account deletion.

---

## Operational Security

- Network policies restrict inter-service communication within the cluster
- No hardcoded secrets in Helm charts — values injected at deploy time via CI pipeline
- Service-to-service auth via internal tokens
- TLS at ingress layer via cert-manager
- Dependencies pinned in lock files; Dependabot / Renovate compatible
- Audit Service database is isolated from business databases — a compromise of the audit store does not expose business data, and vice versa
