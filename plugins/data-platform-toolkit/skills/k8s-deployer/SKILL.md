---
name: k8s-deployer
description: Enforces Kubernetes deployment best practices including resource limits and health probes.
---

# Kubernetes Deployment Rules

All Kubernetes deployments MUST comply with these rules before being applied to any cluster.

## Rule 1: All Containers Must Have CPU and Memory Limits

Every container in a Deployment, StatefulSet, or Job MUST specify both `resources.requests` and `resources.limits` for CPU and memory. Pods without limits can starve other workloads and cause node-level OOM kills. Any manifest missing these fields must be rejected.

```yaml
containers:
  - name: data-ingestion-worker
    image: registry.internal/ingestion-worker:latest
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"
      limits:
        cpu: "1000m"
        memory: "2Gi"
```

## Rule 2: Readiness Probes Must Target `/healthz` or Actuator Health

Every Deployment MUST include a `readinessProbe` that hits either `/healthz` (for Go/Python services) or `/actuator/health` (for Spring Boot / JVM services). Without a readiness probe, Kubernetes will route traffic to pods that are not ready, causing user-facing errors.

```yaml
# For Go / Python services
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

# For Spring Boot / JVM services
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
```

## Rule 3: Always Include a Liveness Probe

In addition to readiness probes, every long-running Deployment MUST include a `livenessProbe` to restart containers that enter a deadlocked or unrecoverable state. The liveness probe should have a higher `failureThreshold` than the readiness probe to avoid unnecessary restarts during transient load spikes.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 5
```
