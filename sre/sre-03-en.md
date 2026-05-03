# Langfuse Self-Hosted on Kubernetes: Deployment, Scaling, and High Availability

*By Samuel Desseaux - Erythix*

---

## Why Self-Host

Langfuse Cloud works well for prototypes. For production infrastructure in financial services, defense, healthcare, or any environment with data sovereignty requirements, self-hosting is not an option — it is a constraint.

This module covers deploying Langfuse on Kubernetes via the official Helm chart, the critical configuration points for production, and strategies for scaling, high availability, and data retention.

---

## Component Architecture

Langfuse self-hosted runs four components:

```
┌─────────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                             │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐   │
│  │  SDK Clients │   │   Ingress     │   │  cert-manager    │   │
│  │  (apps)      │──▶│  nginx       │   │  (TLS)           │   │
│  └──────────────┘   └──────┬───────┘   └──────────────────┘   │
│                             │                                   │
│                    ┌────────▼────────┐                         │
│                    │  Langfuse Web   │  2-4 replicas           │
│                    │  (Next.js)      │  HPA: cpu/mem           │
│                    └────┬──────┬─────┘                         │
│                         │      │                               │
│          ┌──────────────▼──┐  ┌▼────────────────┐            │
│          │ Langfuse Worker  │  │     Redis        │            │
│          │ (async traces)   │  │ (cache + queues) │            │
│          └──────────────────┘  └─────────────────┘            │
│                         │                                      │
│           ┌─────────────▼──────────────┐                      │
│           │  PostgreSQL (external)      │  Required            │
│           │  + PgBouncer (pool)         │                      │
│           └─────────────────────────────┘                      │
│                         │                                      │
│           ┌─────────────▼──────────────┐                      │
│           │  ClickHouse (optional)      │  > 1M traces/month   │
│           │  analytics backend          │                      │
│           └─────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

In production: PostgreSQL with replication, persistent Redis, and optionally ClickHouse for volumes above ~1M traces/month. Below that threshold, PostgreSQL alone is sufficient.

---

## Helm Deployment

```bash
helm repo add langfuse https://langfuse.github.io/langfuse-k8s
helm repo update
```

Create your values file `langfuse-values.yaml`:

```yaml
replicaCount:
  web: 2
  worker: 2

image:
  tag: "2.x.x"   # Pin the version explicitly

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  hosts:
    - host: langfuse.internal.yourcompany.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: langfuse-tls
      hosts:
        - langfuse.internal.yourcompany.com

langfuse:
  nextauth:
    url: "https://langfuse.internal.yourcompany.com"
    secretKey:
      secretRef:
        name: langfuse-secrets
        key: nextauth-secret
  salt:
    secretRef:
      name: langfuse-secrets
      key: salt
  encryptionKey:
    secretRef:
      name: langfuse-secrets
      key: encryption-key

postgresql:
  enabled: false   # Use an external database in production
  external:
    host: "postgres.internal.yourcompany.com"
    port: 5432
    database: "langfuse"
    username: "langfuse"
    passwordSecretRef:
      name: langfuse-secrets
      key: postgres-password

redis:
  enabled: true
  architecture: replication
  auth:
    enabled: true
    existingSecret: langfuse-secrets
    existingSecretKey: redis-password

resources:
  web:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 2Gi
  worker:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi
```

Create secrets before deploying:

```bash
kubectl create namespace langfuse

kubectl create secret generic langfuse-secrets \
  --namespace langfuse \
  --from-literal=nextauth-secret=$(openssl rand -hex 32) \
  --from-literal=salt=$(openssl rand -hex 16) \
  --from-literal=encryption-key=$(openssl rand -hex 32) \
  --from-literal=postgres-password="your-strong-password" \
  --from-literal=redis-password="your-redis-password"
```

Deploy:

```bash
helm install langfuse langfuse/langfuse \
  --namespace langfuse \
  --values langfuse-values.yaml \
  --version 0.x.x
```

---

## PostgreSQL Configuration for Production

Langfuse uses Prisma as its ORM. Migrations run automatically on startup.

```sql
CREATE DATABASE langfuse;
CREATE USER langfuse WITH ENCRYPTED PASSWORD 'strong-password';
GRANT ALL PRIVILEGES ON DATABASE langfuse TO langfuse;
ALTER DATABASE langfuse OWNER TO langfuse;

\c langfuse
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
```

For high volume, configure connection pooling via PgBouncer:

```ini
# pgbouncer.ini
[databases]
langfuse = host=postgres port=5432 dbname=langfuse

[pgbouncer]
pool_mode = transaction
max_client_conn = 500
default_pool_size = 20
min_pool_size = 5
server_reset_query = DISCARD ALL
```

The Langfuse web application can open many concurrent connections under load. PgBouncer in `transaction` mode is the standard choice for Next.js applications.

---

## ClickHouse for High Volume

Above approximately 1 million traces per month, analytical queries in Langfuse (score averages, long-range filtering) become slow on PostgreSQL. ClickHouse is Langfuse's built-in solution for this scenario.

```yaml
# Add to langfuse-values.yaml
clickhouse:
  enabled: true
  auth:
    username: "langfuse"
    existingSecret: langfuse-secrets
    existingSecretKey: clickhouse-password
  persistence:
    enabled: true
    size: 100Gi
    storageClass: "fast-ssd"
  resources:
    requests:
      cpu: 1000m
      memory: 4Gi
    limits:
      cpu: 4000m
      memory: 8Gi
  replicaCount: 2   # ClickHouse cluster mode — requires ClickHouse Keeper (built-in since CH 22.4)
                    # Single replica is simpler; only use cluster mode if you need HA on the analytics layer
```

With ClickHouse enabled, Langfuse uses a dual backend: PostgreSQL for transactional data (projects, users, configurations), ClickHouse for analytics data (traces, generations, scores). This split gives you fast OLAP queries on the analytics data without impacting transactional performance.

---

## Autoscaling and High Availability

HorizontalPodAutoscaler for the web component:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: langfuse-web-hpa
  namespace: langfuse
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: langfuse-web
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # Avoid flapping
```

Pod anti-affinity across availability zones:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values: [langfuse]
          topologyKey: topology.kubernetes.io/zone
```

PodDisruptionBudget for zero-downtime updates:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: langfuse-web-pdb
  namespace: langfuse
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: langfuse
      app.kubernetes.io/component: web
```

---

## Data Retention Strategy

Langfuse does not implement automatic retention natively. Two approaches:

**Retention via API (CronJob):**

```python
# delete-old-traces.py
from langfuse import Langfuse
from datetime import datetime, timedelta, timezone

langfuse = Langfuse()
cutoff = datetime.now(timezone.utc) - timedelta(days=90)

page = 1
deleted = 0
while True:
    traces = langfuse.get_traces(limit=100, page=page, to_timestamp=cutoff)
    if not traces.data:
        break
    for trace in traces.data:
        langfuse.delete_trace(trace.id)
        deleted += 1
    page += 1

print(f"Deleted {deleted} traces older than 90 days")
```

Kubernetes CronJob:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: langfuse-retention
  namespace: langfuse
spec:
  schedule: "0 2 * * 0"   # Weekly, Sunday 2am
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: retention
              image: python:3.12-slim
              command: ["python", "/scripts/delete-old-traces.py"]
              env:
                - name: LANGFUSE_PUBLIC_KEY
                  valueFrom:
                    secretKeyRef:
                      name: langfuse-secrets
                      key: public-key
                - name: LANGFUSE_SECRET_KEY
                  valueFrom:
                    secretKeyRef:
                      name: langfuse-secrets
                      key: secret-key
          restartPolicy: OnFailure
```

Document your retention policy in your data processing register if you are subject to GDPR.

---

## Network Policies

For secured environments, restrict inter-component communication:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: langfuse-restrict
  namespace: langfuse
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: langfuse
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 3000
    - from:
        - podSelector: {}   # Within namespace only
  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: postgresql
      ports:
        - port: 5432
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: redis
      ports:
        - port: 6379
    - ports:
        - port: 443   # Outbound HTTPS (for cloud LLM API calls if proxied)
```

---

## Production Go-Live Checklist

Before switching traffic to your self-hosted instance:

- Credential rotation (`nextauth-secret`, `encryption-key`) completed and stored in Vault
- PostgreSQL backup tested and restore validated
- Langfuse instance monitoring configured (Kubernetes pod metrics, CrashLoop alerts)
- PodDisruptionBudget applied for zero-downtime updates
- Retention policy documented, tested, and scheduled
- Network Policies applied and validated (deny-by-default)
- TLS configured with automatic renewal (cert-manager)
- Disaster recovery runbook written and tested

---

