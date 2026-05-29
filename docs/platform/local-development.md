# Local Development

Two modes, one decision: are you working on a single service, or do you want the entire platform running locally?

---

## Mode 1 — Per-Service Development (recommended for contributors)

Each service ships three compose files that cover every local workflow without touching any other service's repository.

| File                     | Purpose                                                                                    |
| ------------------------ | ------------------------------------------------------------------------------------------ |
| `compose.yaml`           | Infrastructure only — PostgreSQL, RabbitMQ, MailHog, MinIO. Run the service from your IDE. |
| `compose.base.yaml`      | Shared service definitions (extended by the other two files). Not used directly.           |
| `compose.container.yaml` | Full stack — infrastructure + the service itself built from source. No IDE required.       |

### Typical workflow (IDE development)

```bash
# 1. Clone the service
git clone https://github.com/IQKV/foundation-iam-service.git
cd foundation-iam-service

# 2. Install git hooks
pnpm install

# 3. Copy and review environment variables
cp .env.example .env.local

# 4. Start infrastructure only
docker compose up -d
# → PostgreSQL on :5432
# → RabbitMQ on :5672 (management UI :15672)
# → MailHog on :8025
# → MinIO on :9000 (console :9001)   ← IAM only

# 5. Run the service from your IDE or CLI
./mvnw spring-boot:run -Pdev
# → API:      http://localhost:8080
# → Actuator: http://localhost:8081/actuator/health
# → Swagger:  http://localhost:8080/swagger-ui.html
```

### Fully containerised (no IDE)

```bash
# Build and run everything — service + infrastructure
docker compose -f compose.container.yaml up -d --build
```

### Per-service infrastructure summary

| Service                      | PostgreSQL port | RabbitMQ ports | Extra                      |
| ---------------------------- | --------------- | -------------- | -------------------------- |
| `foundation-iam-service`     | 5432            | 5672 / 15672   | MailHog :8025, MinIO :9000 |
| `foundation-billing-service` | 5432            | 5672 / 15672   | MailHog :8025              |
| `foundation-audit-service`   | 5432            | 5672 / 15672   | —                          |
| `foundation-gateway-service` | —               | —              | Depends on IAM + Billing   |

> Each service uses its own named volumes and a dedicated Docker network, so running multiple services simultaneously on the same machine requires no port remapping — each compose stack is fully isolated.

---

## Mode 2 — Full Demo Stack (umbrella compose)

The `microservices-platform` monorepo contains a single `compose.demo.yaml` that starts the entire platform — all services, all infrastructure, and the full observability stack — behind a single Nginx reverse proxy on port 80.

### Prerequisites

- Docker Desktop (or Docker Engine + Compose plugin)
- `hosts` file entry for local domain routing (one-time setup)

**Add to `/etc/hosts` (Linux/macOS) or `C:\Windows\System32\drivers\etc\hosts` (Windows):**

```
127.0.0.1  api.iqkv.local
127.0.0.1  admin.iqkv.local
127.0.0.1  app.iqkv.local
```

### Quick start

```bash
git clone https://github.com/IQKV/microservices-platform.git
cd microservices-platform

# Copy and review environment variables (defaults work out of the box)
cp .env.example .env

# Linux / macOS
./demo.sh

# Windows (PowerShell)
.\demo.ps1
```

Both scripts run `docker compose -f compose.demo.yaml up -d --remove-orphans` and print the access URLs.

### What starts

| Container                      | Role                                     |
| ------------------------------ | ---------------------------------------- |
| `foundation-nginx`             | Reverse proxy — single entry point `:80` |
| `foundation-iam-service`       | Identity & Access Management             |
| `foundation-billing-service`   | Payments & Subscriptions                 |
| `foundation-audit-service`     | Centralized Audit Logging                |
| `foundation-gateway-service`   | API Gateway (JWT validation, routing)    |
| `foundation-ui-app`            | Tenant-facing React SPA                  |
| `foundation-ui-platform-admin` | Platform Admin React SPA                 |
| `foundation-postgres-iam`      | PostgreSQL for IAM                       |
| `foundation-postgres-billing`  | PostgreSQL for Billing                   |
| `foundation-postgres-audit`    | PostgreSQL for Audit                     |
| `foundation-rabbitmq`          | RabbitMQ (AMQP + management UI)          |
| `foundation-redis`             | Redis (cache / session)                  |
| `foundation-mailhog`           | SMTP trap for transactional emails       |
| `foundation-prometheus`        | Metrics collection                       |
| `foundation-grafana`           | Dashboards                               |
| `foundation-loki`              | Log aggregation                          |
| `foundation-promtail`          | Log shipping                             |

### Access URLs

| Endpoint                                     | Description                    |
| -------------------------------------------- | ------------------------------ |
| `http://app.iqkv.local/`                     | Tenant app (sign up / sign in) |
| `http://admin.iqkv.local/`                   | Platform admin UI              |
| `http://api.iqkv.local/`                     | API Gateway                    |
| `http://api.iqkv.local/swagger-ui.html`      | Aggregated Swagger UI          |
| `http://api.iqkv.local/services/grafana/`    | Grafana dashboards             |
| `http://api.iqkv.local/services/prometheus/` | Prometheus                     |
| `http://api.iqkv.local/services/rabbitmq/`   | RabbitMQ management UI         |
| `http://api.iqkv.local/services/mailhog/`    | MailHog (captured emails)      |

### Environment variables

All variables have working defaults in `.env.example`. The only values you need to change for real Stripe integration are:

```dotenv
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

Everything else — database credentials, RabbitMQ credentials, JWT keys, rollout mode — works out of the box with the defaults.

### Useful commands

```bash
# Check service health
docker compose -f compose.demo.yaml ps

# Follow logs for a specific service
docker compose -f compose.demo.yaml logs -f foundation-iam-service

# Stop everything (keep volumes)
docker compose -f compose.demo.yaml down

# Stop and remove all data
docker compose -f compose.demo.yaml down -v

# Restart a single service after a config change
docker compose -f compose.demo.yaml restart foundation-gateway-service
```

---

## Choosing the right mode

| Situation                                              | Use                                         |
| ------------------------------------------------------ | ------------------------------------------- |
| Contributing to a single service                       | Per-service `compose.yaml`                  |
| Running the service fully containerised (no IDE)       | `compose.container.yaml`                    |
| Evaluating the platform end-to-end                     | Demo stack (`compose.demo.yaml`)            |
| Demoing to stakeholders                                | Demo stack                                  |
| Running E2E tests against the full platform            | Demo stack                                  |
| Developing a new service that depends on IAM + Billing | Demo stack (or per-service with cross-refs) |

---

## Startup order and health checks

The demo stack uses Docker Compose `depends_on` with `condition: service_healthy` throughout. Services start in dependency order:

```
PostgreSQL (×3) + RabbitMQ + Redis + MailHog
        ↓
    IAM Service
        ↓
    Billing Service + Audit Service
        ↓
    Gateway Service
        ↓
    UI Apps + Nginx + Observability Stack
```

Each Java service exposes a readiness probe at `http://localhost:8081/actuator/health/readiness`. The compose health check polls this endpoint every 30 seconds with a 90-second start period to allow JVM warm-up and Liquibase migrations to complete.

**Expected startup time:** 2–4 minutes on a modern machine with images already pulled.

---

## Rollout mode

The demo stack defaults to `ROLLOUT_MODE=MULTI_TENANT`. To run in single-tenant mode, set in `.env`:

```dotenv
ROLLOUT_MODE=SINGLE_TENANT
DEFAULT_TENANT_KEY=default
DEFAULT_TENANT_NAME=My Organization
```

The same variable must be consistent across IAM, Billing, and Gateway — the Gateway's `PlatformModeGuardFilter` polls IAM's `/actuator/info` and returns `503` if the modes diverge.
