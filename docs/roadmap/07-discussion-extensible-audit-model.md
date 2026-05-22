# Proposal: Extensible Audit Model (Provider-Friendly Design)

## Overview

This document proposes the design for a flexible, extensible audit logging system for the IQKV platform.

The key principle is **"No Vendor Lock-in"** — consistent with the overall philosophy of the platform (inspired by Magento/Oro flexibility). The audit system should allow different implementations while providing a solid default.

## Goals

- Enable services (`foundation-iam-service`, `foundation-billing-service`, future services) to log activities consistently.
- Support multiple audit backends (default DB, Elasticsearch, DataDog, custom SIEM, etc.).
- Make it easy for users to plug in their own audit provider.
- Keep the core services decoupled from storage details.
- Support both simple and enterprise-grade compliance requirements.

## Proposed Architecture

### 1. New Repositories & Modules

- **`foundation-audit-spi`** — Service Provider Interface (contracts)
- **`foundation-audit-model`** — Shared neutral data models, events, and enums
- **`foundation-audit-starter`** — Spring Boot Starter for active auditing (optional)
- **`foundation-audit-service`** — Centralized microservice for event consumption and storage

### 2. High-Level Design

```mermaid
graph TD
    subgraph "Domain Services"
        IAM[foundation-iam-service]
        Billing[foundation-billing-service]
    end

    subgraph "Event Bus"
        EB[RabbitMQ / iqkv.events]
    end

    subgraph "Centralized Audit"
        AS[foundation-audit-service]
        Store[(Audit Store)]
    end

    IAM -->|publishes BusinessEvent| EB
    Billing -->|publishes BusinessEvent| EB

    EB -->|consumes| AS
    AS --> Store

    subgraph "Active Auditing (Optional)"
        Starter[foundation-audit-starter]
        IAM -.-> Starter
        Starter -.->|publishes AuditEvent| EB
    end
```

### 3. The "Service-Centric" Approach (Recommended)

Creating a dedicated `foundation-audit-service` to "grab" RabbitMQ events is the most lightweight and non-intrusive way to implement auditing across the platform.

#### How it works:

1. **Passive Observation**: The `foundation-audit-service` acts as a platform-wide observer. It binds its own queues to the existing `iqkv.events` exchange.
2. **Event Transformation**: When it receives a `UserEvent` or `InvoiceEvent`, it maps the domain-specific data into a generic `AuditRecord`.
3. **Storage Isolation**: The audit service maintains its own database (PostgreSQL by default, potentially Elasticsearch later), ensuring that audit logs never compete for resources with business transactions.

#### Comparison: Starter vs. Service

| Feature           | Audit Starter (AOP)                  | Audit Service (Event-Driven)             |
| :---------------- | :----------------------------------- | :--------------------------------------- |
| **Intrusiveness** | Low (requires annotation/dependency) | **Zero** (no changes to domain services) |
| **Visibility**    | Captures internal method calls       | Captures public business events          |
| **Context**       | Full technical context (IP, Agent)   | Limited to what's in the event payload   |
| **Complexity**    | Distributed across services          | Centralized in one service               |

### 4. Hybrid Strategy

To achieve a complete audit trail, we will use a hybrid approach:

1. **Primary (Audit Service)**: Consumes existing business events for 80% of audit needs (signups, payments, settings changes).
2. **Secondary (Audit Starter)**: Used only for high-sensitivity actions that don't trigger business events (e.g., "Admin viewed customer credit card last 4 digits" or "Exported user list").

### 5. Context Capture Strategy: Gateway-First Approach

The `foundation-gateway-service` serves as the single entry point for all client requests. This makes it the ideal place to capture and normalize technical context before it reaches downstream services.

#### Gateway Responsibilities:

1.  **Context Extraction**: The gateway extracts the client's IP address (`X-Forwarded-For`), User-Agent, and any existing Correlation ID.
2.  **Header Normalization**: It injects a set of standardized headers into the downstream request:
    - `X-Correlation-ID`: Trace ID for cross-service tracking.
    - `X-Audit-IP`: The original client IP.
    - `X-Audit-UA`: The client's User Agent.
    - `X-Audit-Source`: Identifies the entry point (e.g., `web-gateway`, `mobile-gateway`).
3.  **JWT Propagation**: As already implemented in `JwtContextPropagationFilter`, it passes user and tenant identity via `X-User-ID` and `X-Tenant-ID`.

### 6. Event Enrichment Flow (Downstream Services)

By moving context capture to the Gateway, downstream services (like `foundation-iam-service`) remain lightweight and focused on business logic.

#### Enrichment Flow:

1.  **Request Interception**: A common `AuditContextInterceptor` in the microservice (provided via a shared library or starter) reads the `X-Audit-*` headers.
2.  **Context Storage**: These values are stored in a `ThreadLocal` `AuditContextHolder`.
3.  **Automatic Enrichment**: When a service publishes a domain event (e.g., via `MessagingService`), it automatically merges the data from `AuditContextHolder` into the event payload.

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant IAM
    participant RabbitMQ
    participant AuditService

    Client->>Gateway: POST /api/v1/users (Login)
    Note over Gateway: Captures IP, UA, CorrelationID
    Gateway->>IAM: POST /api/v1/users (with X-Audit headers)
    Note over IAM: Interceptor populates AuditContextHolder
    IAM->>IAM: Process Business Logic
    IAM->>RabbitMQ: Publish UserLoggedInEvent (Enriched with AuditContext)
    RabbitMQ->>AuditService: Consume Event
    AuditService->>Store: Persist AuditRecord
```

### 7. Enriched Audit Event Model

The `AuditEvent` in `foundation-audit-model` should follow a structured format that supports both domain data and enriched context:

```java
public record AuditEvent(
    UUID id,
    String action,          // e.g., USER_LOGIN, INVOICE_PAID
    String entityType,      // e.g., USER, TENANT, INVOICE
    String entityId,
    AuditActor actor,       // Enriched: userId, email, ipAddress, userAgent
    String tenantKey,
    ActivitySeverity severity,
    Map<String, Object> details,
    Instant occurredAt,
    String traceId
) {}

public record AuditActor(
    String id,             // Subject ID (user id)
    String type,           // USER, SYSTEM, ANONYMOUS
    String email,
    String ipAddress,
    String userAgent,
    String impersonatorId  // If an admin is acting on behalf of a user
) {}
```

### 8. Messaging Pipeline (RabbitMQ)

- **Exchange**: `iqkv.audit.events` (Topic)
- **Routing Key**: `audit.[service-name].[action]`
- **Reliability**: Use the Outbox Pattern in microservices to ensure `AuditEvent` is published even if RabbitMQ is temporarily down.

### 9. Package Structure (Tactical DDD)

The `foundation-audit-service` follows the platform's standard Tactical DDD layout, ensuring consistency with `foundation-iam-service` and other microservices.

#### `foundation-audit-service`

- `com.iqkv.foundation.auditservice`
  - `shared/` — Shared kernel (exceptions, common utilities)
  - `infrastructure/` — Technical concerns
    - `config/` — RabbitMQ, Security, MyBatis configurations
    - `persistence/` — MyBatis Mapper interfaces and XMLs
    - `messaging/` — RabbitMQ Consumers (binding to `iqkv.events`)
  - `audit/` — Core Bounded Context
    - `domain/` — `AuditRecord` entity, `AuditStore` port
    - `application/` — `AuditLogService` (orchestration), `AuditSearchQuery`
    - `adapter/`
      - `in/rest/` — `AuditSearchController` (for Admin UI)
      - `out/persistence/` — MyBatis implementation of `AuditStore`

#### `foundation-audit-model` (Shared Library)

- `com.iqkv.foundation.audit.model`
  - `event/` — `AuditEvent.java`, `AuditActor.java`
  - `enums/` — `ActivitySeverity.java`, `ActivityAction.java`

#### `foundation-audit-spi` (Contract Library)

- `com.iqkv.foundation.audit.spi`
  - `AuditProvider.java` — SPI for custom audit backends

### 10. Technical Stack & Implementation Details

The `foundation-audit-service` will leverage the core platform technologies to ensure high performance and maintainability.

#### Technology Stack:

- **Runtime**: Java 25
- **Framework**: Spring Boot 4.x
- **Persistence**: MyBatis with PostgreSQL (Liquibase for migrations)
- **Messaging**: RabbitMQ (Spring AMQP)
- **API Documentation**: SpringDoc OpenAPI 3
- **Observability**: Micrometer + Prometheus

#### Key Implementation Patterns:

1.  **Passive Event Consumption**:
    - The service uses `@RabbitListener` to bind to the `iqkv.events` exchange.
    - It uses a **Topic Exchange** with wildcard routing keys (e.g., `user.#`, `tenant.#`) to capture relevant business events.
2.  **MyBatis for Flexible Persistence**:
    - Uses XML mappers for complex audit log queries (e.g., filtering by dynamic metadata).
    - Implements custom `TypeHandler` for JSONB fields (storing the `details` map).
3.  **Outbox Pattern Support**:
    - While primarily a consumer, the service will support the Outbox Pattern if it needs to publish its own events (e.g., `AuditAlertTriggered`).
4.  **Security**:
    - Protected by Spring Security and OAuth2 Resource Server.
    - Only users with `PLATFORM_ADMIN` authority can access the search API.

### 11. Configuration

```yaml
iqkv:
  audit:
    service:
      enabled: true
      storage-type: postgres # postgres | elasticsearch
    starter:
      enabled: false # Only enable if AOP auditing is needed
```

### Benefits

- **Zero-Touch for Core Logic**: No code changes needed in IAM or Billing for basic auditing.
- **Decoupled**: Services don't care how or where audit logs are stored.
- **Scalable**: Audit processing doesn't steal CPU/Memory from checkout or login flows.
- **Compliance Ready**: Centralized place to implement GDPR "right to be forgotten" or retention policies.

### Implementation Roadmap

**Phase 1 (v0.3)**

- Define `foundation-audit-model` (Neutral event structures).
- Create `foundation-audit-service` (The central consumer).
- Implement IAM event listeners in Audit Service (Logins, Signup).
- Basic PostgreSQL storage.

**Phase 2**

- Implement Billing event listeners in Audit Service (Payments, Subscriptions).
- Create `foundation-audit-starter` for `@Auditable` support (Active Auditing).
- Add "Audit Log" tab to Platform Admin UI.

**Phase 3**

- Advanced search/filtering API.
- Elasticsearch provider for high-volume logs.
- Automated retention and archiving.

---

### Very basic, high-level example

```
com.iqkv.foundation.audit

├── spi/                          # Service Provider Interface (Core Contract)
│   ├── AuditLogService.java             # Main interface
│   ├── AuditEventPublisher.java
│   ├── ActivityLogRepository.java
│   └── AuditProvider.java               # Marker / Config interface
│
├── model/                        # Shared Data Models (Neutral)
│   ├── event/
│   │   ├── UserActivityEvent.java
│   │   ├── AuditEvent.java
│   │   └── ...
│   ├── record/
│   │   ├── UserActivityLog.java
│   │   ├── Actor.java
│   │   └── ActivityContext.java
│   └── enum/
│       ├── ActivityAction.java
│       ├── EntityType.java
│       └── ActivitySeverity.java
│
├── provider/                     # Default Implementation
│   ├── internal/
│   │   ├── DefaultAuditLogService.java
│   │   └── InMemoryAuditRepository.java   #
│   └── config/
│       └── DefaultAuditAutoConfiguration.java
│
└── dto/                          # API transfer objects
```

---

**Status**: Proposal  
**Author**: [Your Name]  
**Date**: May 22, 2026  
**Version**: 1.0
