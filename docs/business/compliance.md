# Compliance & Data Isolation

## Schema-Per-Tenant

Each tenant has a dedicated PostgreSQL schema within the IAM database (`iqscaffold_iam`). The schema boundary is enforced at the database engine level, not in application code.

Practical implications:

| Concern          | How it's addressed                                                                   |
| ---------------- | ------------------------------------------------------------------------------------ |
| Data leakage     | Cross-tenant queries impossible in normal flow — wrong `search_path` returns no rows |
| GDPR erasure     | `DROP SCHEMA tenant_x CASCADE` — no orphaned records in shared tables                |
| Backup / restore | Dump or restore a single schema without affecting others                             |
| Audit scope      | One schema = one tenant — no filtering required                                      |
| Tenant migration | Move schema to dedicated instance by updating connection string — no code change     |

---

## Regulatory Alignment

This is a reference, not a certification. Actual compliance requires audit and legal review.

**GDPR** — Schema isolation supports data subject rights (access, erasure, portability). Audit events per tenant are emitted to RabbitMQ and can be exported.

**SOC 2 Type II** — Schema-level isolation is a stronger control than application-level `tenant_id` filtering. Helm-based IaC supports change management controls.

**HIPAA** — PHI can be isolated per schema. Access enforced at IAM + Gateway layer. Audit trail via structured event log.

**ISO 27001** — IaC (Helm) supports asset management and change control. K8s RBAC + IAM service covers access control requirements.

---

## Access Control

Every request passes through the API Gateway, which validates the JWT and resolves tenant context before forwarding. No request reaches IAM or Billing without passing this layer.

IAM enforces:

- Organization boundary (tenant isolation)
- Role-based access (owner / admin / member / viewer)

---

## Operational Security

- Network policies restrict inter-service communication within the cluster
- No hardcoded secrets in Helm charts — values injected at deploy time
- Service-to-service auth via internal tokens
- TLS at ingress layer via cert-manager
- Dependencies pinned in lock files; Dependabot / Renovate compatible
