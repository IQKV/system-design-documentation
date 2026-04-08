# Scalability

## Design Principles

The platform is designed to scale horizontally at every layer. No component is a single point of failure in a production configuration.

---

## Service Scaling

Each microservice is stateless and scales independently via Kubernetes HorizontalPodAutoscaler.

```yaml
# IAM service HPA (values-prd.yaml)
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

Services do not share in-process state. Session state is stored in the database (token denylist, global signout timestamp), not in memory — any replica can handle any request.

---

## Database Scaling

### Vertical scaling (initial)

For most deployments, a single well-sized PostgreSQL instance handles hundreds of tenants comfortably. Schema-per-tenant adds negligible overhead compared to a shared-table model.

### Read replicas

Read-heavy workloads (reporting, analytics queries) can be offloaded to read replicas. The application connection pool supports read/write splitting.

### Tenant migration to dedicated instances

When a tenant's workload justifies dedicated infrastructure:

1. Dump the tenant's schema from the shared instance
2. Restore it to a dedicated PostgreSQL instance
3. Update the tenant's connection string in the platform registry
4. The application routes that tenant's traffic to the dedicated instance automatically

No code changes. No downtime for other tenants.

### Connection pooling

PgBouncer is included in the Helm chart for connection pooling. At scale, this is critical — each service replica does not hold a direct connection to PostgreSQL.

---

## Message Queue Scaling

RabbitMQ scales via clustering. For high-throughput provisioning scenarios, the provisioning worker scales independently of the IAM service — more workers consume the queue faster without affecting the HTTP request path.

Queue depth and consumer lag are exposed as Prometheus metrics, enabling autoscaling of worker pods based on queue backlog.

---

## Tenant Tiers

The platform supports tiered infrastructure allocation per tenant:

| Tier              | Database                  | Resources  | Use case                      |
| ----------------- | ------------------------- | ---------- | ----------------------------- |
| Shared            | Schema on shared instance | Default    | Standard tenants              |
| Dedicated DB      | Own PostgreSQL instance   | On request | High-volume tenants           |
| Dedicated cluster | Own K8s namespace/cluster | Enterprise | Compliance-required isolation |

Tier assignment is a configuration change, not an architectural change.

---

## Benchmarks (Reference)

These are reference figures for planning purposes, not guarantees. Actual performance depends on hardware, query patterns, and tenant workload distribution.

| Metric                                 | Value                            |
| -------------------------------------- | -------------------------------- |
| Tenant provisioning time (async)       | < 5 seconds end-to-end           |
| API Gateway request overhead           | < 5ms added latency              |
| Tenants per shared PostgreSQL instance | 500–2000 (depending on activity) |
| IAM token validation                   | < 2ms (cached)                   |

---

## Production Sizing (Minimum)

Derived from `values-prd.yaml` defaults across services.

| Component   | Minimum                         | Recommended                    |
| ----------- | ------------------------------- | ------------------------------ |
| IAM         | 2 replicas, 512Mi/0.5 CPU each  | 3 replicas, 1Gi/1 CPU each     |
| API Gateway | 2 replicas, 256Mi/0.25 CPU each | 3 replicas, 512Mi/0.5 CPU each |
| Billing     | 2 replicas, 256Mi/0.25 CPU each | 2 replicas, 512Mi/0.5 CPU each |
| PostgreSQL  | 4 CPU, 8Gi RAM, 100Gi SSD       | 8 CPU, 16Gi RAM, 500Gi SSD     |
| RabbitMQ    | 3-node cluster, 2Gi RAM each    | 3-node cluster, 4Gi RAM each   |
