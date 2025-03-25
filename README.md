# Kubernetes Failure Scenarios Reference

This document outlines common Kubernetes failure scenarios for testing, troubleshooting practice, and validation of monitoring systems.

## Basic Pod Failures

### CrashLoopBackOff

Container repeatedly exits, causing Kubernetes to continually restart it.

```yaml
containers:
- name: crash-loop
  image: busybox:latest
  command: ["sh", "-c", "exit 1"]
```

### ImagePullBackOff

Kubernetes cannot pull the container image.

```yaml
containers:
- name: pull-fail
  image: nonexistent/image:v999
```

### OOMKilled

Container exceeds its memory limits.

```yaml
containers:
- name: memory-hog
  image: polinux/stress
  resources:
    limits:
      memory: "64Mi"
  command: ["stress"]
  args: ["--vm", "1", "--vm-bytes", "128M"]
```

## Resource and Configuration Issues

### Failed Health Probes

Pod restarts due to failed readiness/liveness checks.

```yaml
containers:
- name: web
  image: nginx
  livenessProbe:
    httpGet:
      path: /nonexistent
      port: 80
```

### Resource Quota Exceeded

Prevents new pod creation when namespace limits are reached.

```yaml
apiVersion: v1
kind: ResourceQuota
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
```

### Insufficient Resources

Pod remains Pending due to impossible resource requests.

```yaml
containers:
- name: heavy
  resources:
    requests:
      cpu: "100"
      memory: "100Gi"
```

### Missing Dependencies

Pod fails due to missing required resources.

```yaml
# Missing PVC example
volumes:
- name: data
  persistentVolumeClaim:
    claimName: nonexistent-pvc

# Missing ConfigMap example
envFrom:
- configMapRef:
    name: nonexistent-config
```

## Network and Security Issues

### Network Policy Blocks

Prevents pod communication by blocking all traffic.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
```

### RBAC Permission Issues

Pod fails operations due to insufficient permissions.

```yaml
spec:
  serviceAccountName: restricted-sa  # SA with no roles
  containers:
  - name: kubectl
    image: bitnami/kubectl
    command: ["kubectl", "get", "pods"]
```

## Initialization and Scheduling Issues

### Init Container Failure

Main container never starts due to failed initialization.

```yaml
spec:
  initContainers:
  - name: init
    image: busybox
    command: ['sh', '-c', 'exit 1']
```

### Anti-Affinity Scheduling Failure

Pods cannot be scheduled due to placement constraints.

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - antiaffinity
        topologyKey: kubernetes.io/hostname
```

### Invalid Container Arguments

Container fails to start due to wrong configuration.

```yaml
containers:
- name: redis
  image: redis
  args:
  - "--totally-invalid-flag"
```

## Expected Symptoms

| Scenario | Typical Symptoms |
|----------|-----------------|
| CrashLoopBackOff | Pod repeatedly restarts |
| ImagePullBackOff | Pod stuck in pending, pull errors |
| OOMKilled | Container terminations, memory pressure |
| Failed Probes | Regular restarts, readiness failures |
| Resource Quota | New pods stuck in pending |
| Missing Dependencies | Pod stuck in pending or init |
| Network Policy | Connection timeouts, DNS failures |
| RBAC Issues | Permission denied errors |
| Init Failures | Pod stuck in Init phase |
| Anti-affinity | FailedScheduling events |
| Wrong Args | Application specific errors in logs |

Use these examples responsibly and preferably in a test environment to avoid impacting production workloads.
