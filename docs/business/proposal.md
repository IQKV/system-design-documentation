# Business Proposal

## Problem

Building B2B SaaS requires solving infrastructure before building product: user authentication, multi-tenant data isolation, subscription billing, API security, and tenant provisioning. Teams typically spend 4-6 months building these foundational systems before they can focus on their actual product features.

Existing solutions either provide only UI scaffolding (leaving infrastructure unsolved) or create vendor lock-in through managed services.

---

## What We Built

A complete SaaS infrastructure platform consisting of three microservices:

### IAM Service

- User registration with email verification
- JWT-based authentication (RS256, 15-min access + 7-day refresh tokens)
- Password reset with rate limiting and brute-force protection
- Multi-organization membership (users can belong to multiple tenants)
- Role-based access control (TENANT_OWNER, ADMIN, MEMBER)
- Email invitations with 72-hour expiring tokens
- PostgreSQL schema-per-tenant for complete data isolation

### API Gateway

- Request routing to backend services
- JWT validation and user context propagation
- Tenant resolution from token claims or configuration
- Header sanitization to prevent spoofing
- CORS configuration and public path management
- Prometheus metrics and health checks

### Billing Service

- Stripe Connect wrapper for customer and subscription management
- Plan catalog with tenant/user scoped billing models
- Webhook processing with signature verification and idempotency
- Local subscription caching for fast reads without Stripe API calls
- Email notification publishing (9 types) via RabbitMQ
- Support for both multi-tenant and single-tenant billing
- Scheduled jobs for trial ending and payment overdue notifications

### UI Application

- React SPA with Mantine UI components
- Authentication flows (signup, login, password reset)
- Organization management (create, invite members, manage roles)
- Account settings and profile management
- Billing portal integration with Stripe

### Deployment & Operations

- Kubernetes Helm charts for each service
- Docker Compose for local development
- Environment-specific configuration (local, staging, production)
- Database migrations with Liquibase
- Async tenant provisioning via RabbitMQ
- Structured logging and monitoring setup

---

## Key Features

### Hybrid Tenancy Model

Single codebase supports two deployment modes:

- **Multi-tenant**: Each signup creates a new organization
- **Single-tenant**: All users join a pre-configured default organization

Mode is controlled by configuration - no code changes required.

### Data Isolation

- Each tenant gets its own PostgreSQL schema (t_tenantkey)
- Automatic schema switching based on request context
- System data (users, tenants) in public schema
- Cross-tenant queries prevented by design

### Security

- RS256 JWT tokens with JWKS endpoint for validation
- Token revocation via denylist and global signout timestamps
- Brute-force protection (5 attempts, 15-minute lockout)
- Rate-limited password reset (3 requests per 15 minutes)

### Async Processing

- RabbitMQ for tenant provisioning workflows
- ShedLock for distributed job coordination
- Automatic cleanup of expired tokens and invitations
- Retry mechanisms for failed operations

---

## Current Status

**Implemented and Working:**

- Complete user authentication and authorization
- Multi-tenant data isolation with PostgreSQL schemas
- Stripe subscription billing integration
- React administrative interface
- Kubernetes deployment automation
- Email notifications (verification, password reset, invitations)
- Async tenant provisioning with failure handling

**Operational Features:**

- Health checks and readiness probes
- Prometheus metrics collection
- Structured JSON logging with correlation IDs
- Database connection pooling and migration management
- Configuration management via Helm values

---

## Target Users

Engineering teams (3-15 developers) building B2B SaaS applications who:

- Need multi-tenant architecture from day one
- Have Kubernetes deployment experience
- Want to avoid vendor lock-in while getting production-ready infrastructure
- Prefer open-source solutions they can modify and extend

Works for both multi-customer SaaS platforms and single-tenant enterprise deployments.

---

## Business Model

**Open Core**: Core services are Apache 2.0 licensed and freely available.

**Extension Model**: The RabbitMQ event bus allows paid or community extensions to subscribe to platform events without modifying core services.

**Potential Revenue Streams:**

- Premium extensions (SAML/SSO, advanced analytics, compliance tools)
- Managed hosting for teams without Kubernetes expertise
- Enterprise support and custom development services

---

## Technical Specifications

### Technology Stack

- **Backend**: Java 25, Spring Boot 4.0, MyBatis, PostgreSQL 17
- **Frontend**: React 18, TypeScript, Mantine UI, Vite
- **Infrastructure**: Kubernetes, Helm, Docker, RabbitMQ
- **Security**: JJWT (RS256), Spring Security OAuth2
- **Monitoring**: Micrometer, Prometheus, structured logging

### Scalability

- Stateless services for horizontal scaling
- Database read replicas and connection pooling
- Schema-per-tenant allows individual tenant migration to dedicated databases
- Event-driven architecture for async processing

### Deployment Options

- Kubernetes cluster with Helm charts
- Local development with Docker Compose
- Environment-specific configurations (dev, staging, production)
- Infrastructure-as-code with version control

---

## What's Not Included

Current limitations and out-of-scope features:

- SSO/SAML integration (planned as extension)
- Usage-based billing and metering
- Multi-region deployment
- Managed hosting service
- Advanced analytics and reporting
- Automated compliance tools (SOC2, GDPR, HIPAA)
