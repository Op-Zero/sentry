# Sentry Helm Deployment

Production deployment of Sentry 26.5.0 on a single-node RKE2 Kubernetes cluster using Helm chart 32.2.0.

## Access

| | |
|---|---|
| **URL** | https://sent.iranserver.dev |
| **Username** | `admin@sentry.local` |
| **Password** | `Si3vJS1MDo2mUxc` |
| **Superuser** | Yes |

## Prerequisites

- Kubernetes cluster (tested on RKE2 v1.30.14)
- Helm 3.x
- `local-path` StorageClass (Rancher local-path-provisioner)

## Install

Complete fresh installation from scratch:

```bash
# 1. Update chart dependencies
cd /root/sentry
helm dependency update

# 2. Edit production-values.yaml with your settings
#    - system.url: your Sentry URL
#    - user.email / user.password: admin credentials
#    - ingress.hostname: your domain

# 3. Deploy
helm upgrade --install sentry . \
  --namespace sentry \
  --create-namespace \
  --values production-values.yaml \
  --set system.url=https://sent.iranserver.dev \
  --wait \
  --timeout=2700s
```

The `--wait` flag ensures all pods are ready before the command returns. `--timeout=2700s` (45 minutes) allows time for image pulls and hook execution on first install.

## Upgrade

```bash
helm dependency update

helm upgrade --install sentry . \
  --namespace sentry \
  --values production-values.yaml \
  --set system.url=https://sent.iranserver.dev \
  --wait \
  --timeout=2700s
```

Read the upgrade guide before upgrading to major versions: [Upgrade Guide](docs/UPGRADE.md)

## Uninstall

```bash
# Standard uninstall (preserves PVCs)
helm uninstall sentry -n sentry

# Full cleanup including PVCs and namespace
helm uninstall sentry -n sentry --no-hooks
kubectl delete namespace sentry --wait=false
kubectl wait --for=delete namespace/sentry --timeout=120s || \
  kubectl delete namespace sentry --force --grace-period=0
```

## Verify Installation

```bash
# Check all pods are Running
kubectl get pods -n sentry

# Check for CrashLoopBackOff (should be 0)
kubectl get pods -n sentry | grep -c CrashLoopBackOff

# Check Kafka topics exist (should be 111)
kubectl exec sentry-kafka-controller-0 -n sentry -c kafka -- \
  kafka-topics.sh --bootstrap-server localhost:9092 --list | wc -l

# Check Sentry UI is accessible
curl -sk -L -o /dev/null -w "%{http_code}" https://sent.iranserver.dev/
# Should return 200
```

## Architecture

Single-node cluster: 12 vCPU, 32 GB RAM, 150 GB NVMe.

| Component | Status | Replicas |
|---|---|---|
| Sentry Web | 1/1 Running | 1 |
| Relay | 1/1 Running | 1 |
| Kafka (KRaft) | 1/1 Running | 1 |
| PostgreSQL | 1/1 Running | 1 |
| Redis | 1/1 Running | 1 |
| ClickHouse 24.12 | 1/1 Running | 1 |
| Memcached | 1/1 Running | 1 |
| Snuba API | 1/1 Running | 1 |
| Snuba Replacer | 1/1 Running | 1 |
| Snuba Consumers | 1/1 Running | 20+ |
| Sentry Consumers | 1/1 Running | 15+ |
| Task Workers | 1/1 Running | 4 |
| Task Brokers | 1/1 Running | 4 |

**Total pods: ~65** (all Running, 0 CrashLoopBackOff)

## Key Production Decisions

| Decision | Setting | Rationale |
|---|---|---|
| **replicaCount: 1** | sentry-web, relay, taskWorker | Single-node; reduced to fit 12-core CPU budget. |
| **Kafka replication: 1** | `offsets.topic.replication.factor=1` | Single broker; required for `__consumer_offsets` topic to form. |
| **Kafka retention: 168h** | `log.retention.hours=168` | 7-day retention for Kafka topics. |
| **Event retention: 30 days** | `sentry.cleanup.days: 30` | Sentry cleanup cronjob runs daily. |
| **KRaft mode** | `kraft.enabled: true` | No ZooKeeper dependency. |
| **ClickHouse JSON** | `enable_json_type=1` | Required by Snuba migration `0050_add_attributes_array_column`. |
| **asHook: true** | Consumer Deployments as Helm hooks | Ensures Kafka topics are created before any consumer starts. Prevents `UNKNOWN_TOPIC_OR_PART` errors. |

## Disabled Features

Profiling, Feedback, Span processing, Uptime monitoring are disabled. Combined savings: ~1.15 CPU / ~3.2 Gi RAM.

## Infrastructure Notes

### Kafka Topics

111 topics are created automatically by the `kafka-topics-create` post-install hook (weight -1), which runs BEFORE any consumer Deployment hooks (weight 10+). Topics are created in parallel batches of 10 for fast execution (~33s total).

### ClickHouse

Deployed as a standalone StatefulSet (not a sub-chart). The `users.d/default-user.xml` is removed to allow passwordless connections from `::/0`. JSON type is enabled via ConfigMap mount.

### TLS / Ingress

TLS is managed by the chart's built-in self-signed certificate hook (`tls-cert-create`). Ingress is enabled via `ingress.enabled: true` in production-values.yaml.

### Storage Class

Uses `local-path` StorageClass (Rancher local-path-provisioner v0.0.30).

## Resources

- **CPU requests**: ~11.0 cores (under 12-core limit)
- **Memory requests**: ~25.0 Gi (under 32 Gi limit)
- **Storage**: PostgreSQL 20Gi, Kafka 20Gi, ClickHouse 30Gi

## Expected Storage (30 days)

| Store | Est. Size | PVC |
|---|---|---|
| PostgreSQL | ~1-3 Gi | 20 Gi |
| ClickHouse | ~12-24 Gi | 30 Gi |
| Kafka | ~6-15 Gi | 20 Gi |

## ClickHouse TTL for Metrics/Sessions

After deployment, connect to ClickHouse and run:

```sql
ALTER TABLE generic_metric_sets_raw_local MODIFY TTL toDate(timestamp) + INTERVAL 15 DAY;
ALTER TABLE generic_metric_distributions_raw_local MODIFY TTL toDate(timestamp) + INTERVAL 15 DAY;
ALTER TABLE generic_metric_counters_raw_local MODIFY TTL toDate(timestamp) + INTERVAL 15 DAY;
ALTER TABLE generic_metric_gauges_raw_local MODIFY TTL toDate(timestamp) + INTERVAL 15 DAY;
ALTER TABLE sessions_raw_local MODIFY TTL toDate(started) + INTERVAL 15 DAY;
```

## Troubleshooting

### Pods in CrashLoopBackOff

Check logs for the failing pod:

```bash
kubectl logs -n sentry <pod-name> --tail=50
```

Common causes:
- **`UNKNOWN_TOPIC_OR_PART`**: Kafka topics not created yet. The `kafka-topics-create` hook should handle this automatically.
- **`KeyError: system.rate-limit`**: Deprecated config option. Remove it from `config.sentryConfPy`.
- **Connection refused**: Dependency pod (Kafka, PostgreSQL, ClickHouse) not ready yet. Check with `kubectl get pods -n sentry`.

### Kafka topics missing

```bash
# List all topics
kubectl exec sentry-kafka-controller-0 -n sentry -c kafka -- \
  kafka-topics.sh --bootstrap-server localhost:9092 --list

# Count topics (should be 111)
kubectl exec sentry-kafka-controller-0 -n sentry -c kafka -- \
  kafka-topics.sh --bootstrap-server localhost:9092 --list | wc -l
```

### Sentry UI not accessible

```bash
# Check ingress
kubectl get ingress -n sentry

# Check TLS secret
kubectl get secret -n sentry | grep tls

# Test connectivity
curl -sk -L -o /dev/null -w "%{http_code}" https://sent.iranserver.dev/
```

### Full reset

```bash
helm uninstall sentry -n sentry --no-hooks
kubectl delete namespace sentry --wait=false
kubectl wait --for=delete namespace/sentry --timeout=120s || \
  kubectl delete namespace sentry --force --grace-period=0

# Then re-install
helm upgrade --install sentry . \
  --namespace sentry --create-namespace \
  --values production-values.yaml \
  --set system.url=https://sent.iranserver.dev \
  --wait --timeout=2700s
```
