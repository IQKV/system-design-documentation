# Roadmap

## v0.1 — Demo Release (current target)

Goal: all four components running on a public demo host with enough UI to evaluate the platform.

**Services**

- [ ] IAM — registration, login, JWT, org management, RBAC, invitations, account recovery
- [ ] API Gateway — JWT validation, tenant resolution, routing
- [ ] Billing — Stripe Connect integration, webhook handling, lifecycle events
- [ ] UI — React + Mantine, deployed as static build behind the Gateway

**UI screens required for demo**

- [ ] Sign up / login / password reset
- [ ] Dashboard (org overview, active members, subscription status)
- [ ] Users grid (list members, roles, invite status)
- [ ] User edit page (change role, revoke access)
- [ ] Billing portal link (Stripe-hosted)

**Infrastructure**

- [ ] Helm chart per service (iam, api-gateway, billing, ui)
- [ ] Docker Compose for local dev
- [ ] Demo environment deployed and publicly accessible

---

## Post-Demo

Items deferred until after the demo is stable:

- SSO / SAML adapter (extension, not core)
- Rate limiting (per-tenant and per-user)
- Tenant resolution by subdomain
- Usage-based billing metering
- Admin panel (platform-level tenant management)
- Multi-region support
- Managed hosting offering
