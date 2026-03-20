# Kubernetes StatefulSet — Developer Guide

**StatefulSet** runs **stateful** apps: **stable identity**, **ordered** deploy/scale, and **per-replica storage** — the right tool when **order and data** matter. Same style as Pod / ReplicaSet / Deployment: **complete, crisp, scannable**.

---

## What is a StatefulSet?

A **StatefulSet** is a workload controller that:

- Runs **N** replicas with **predictable names**: `<statefulset-name>-0`, `-1`, `-2`, …
- Gives each Pod **stable network identity** (DNS) via a **Headless Service**
- Can provision **one PersistentVolumeClaim per Pod** via **`volumeClaimTemplates`**
- Scales and updates **in order** (configurable) — unlike Deployment’s parallel rollouts

```
StatefulSet  →  Pods (web-0, web-1, …)  +  PVCs (data-web-0, …)
     ↑
Headless Service (DNS per Pod)
```

**Mental model:** “**Ordered**, **named** replicas with **their own disk** and **their own DNS name**.”

---

## StatefulSet vs Deployment

| Need | Use |
|------|-----|
| Identical replicas, no ordering, replaceable Pods | **Deployment** |
| Stable hostname, ordered rollout, **per-Pod PVC**, clusters (DB, Kafka, etcd) | **StatefulSet** |

| Feature | Deployment | StatefulSet |
|--------|------------|-------------|
| Pod names | Random suffix | **Stable ordinal** (`app-0`, `app-1`) |
| Storage | Usually shared/none via Deployment | **PVC per replica** (typical) |
| Create/scale order | Parallel | **Sequential** (default) or parallel (optional) |
| Service | ClusterIP / any | Often **Headless** (`clusterIP: None`) |

---

## Core concepts (don’t skip)

### 1) Headless Service + `spec.serviceName`

- **`spec.serviceName`** must point to a **Service** in the same namespace (usually **Headless**: `clusterIP: None`).
- DNS pattern (cluster DNS):  
  **`<pod-name>.<service>.<namespace>.svc.cluster.local`**  
  Example: `web-0.web.demo.svc.cluster.local`
- Short names work inside the same namespace: `web-0.web`

Without a proper Service + `serviceName`, **stable DNS** for each Pod won’t work as expected.

### 2) Stable network identity

- Each Pod has a **hostname** equal to its Pod name (`web-0`, etc.).
- Peers discover each other via **DNS** or env built from ordinals.

### 3) `volumeClaimTemplates`

- Defines **one PVC template**; StatefulSet creates **`data-<sts-name>-0`**, `-1`, … (name prefix = `volumeClaimTemplates.metadata.name`)
- **Storage is per Pod** — not shared by default (unless you use ReadWriteMany and shared volumes by design)

### 4) Selector + template labels

- **`spec.selector.matchLabels`** must match **`spec.template.metadata.labels`** (same rules as Deployment).
- **Selector is immutable** after create (matchLabels).

### 5) Pod management policy

| Value | Behavior |
|-------|----------|
| **`OrderedReady`** (default) | Start/scale **in order** 0→N; stop **reverse** N→0 |
| **`Parallel`** | Start/terminate **in parallel** (faster; no ordering guarantee) |

### 6) Update strategy

| `type` | Behavior |
|--------|----------|
| **`RollingUpdate`** (default) | Update Pods in **reverse ordinal** (often); use **`partition`** for canary (only Pods ≥ partition updated) |
| **`OnDelete`** | New template applies **only when Pod is deleted** manually |

**RollingUpdate fields:**

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0   # Pods with ordinal < partition are NOT updated (canary pattern)
```

- **`partition: N`**: only Pods with **ordinal ≥ N** get the new revision; lower ordinals stay on old version.

---

## Key `spec` fields (quick reference)

| Field | Purpose |
|-------|---------|
| `replicas` | Number of Pods |
| `serviceName` | Headless (or valid) Service for stable DNS |
| `volumeClaimTemplates` | Per-replica PVCs |
| `podManagementPolicy` | `OrderedReady` vs `Parallel` |
| `updateStrategy` | `RollingUpdate` (+ `partition`) or `OnDelete` |
| `template` | Pod spec (containers, probes, `volumeMounts`) |

---

## Create a StatefulSet

### Imperative (limited)

`kubectl` has **no** rich one-shot like `kubectl create statefulset --image=...` with PVC templates in all versions. Typical **imperative** ops:

```bash
kubectl get sts -n <namespace>
kubectl describe sts <name> -n <namespace>
kubectl scale sts <name> --replicas=4 -n <namespace>

# Delete (Pods removed in reverse order by default)
kubectl delete sts <name> -n <namespace>
```

**Optional:** check rollout-style status (supported for StatefulSet):

```bash
kubectl rollout status statefulset/<name> -n <namespace>
```

**Create** = almost always **`kubectl apply -f`** with full YAML.

### Declarative (recommended)

**Minimal pattern:** Headless Service + StatefulSet with `volumeClaimTemplates` (optional but common).

**01-service-headless.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: demo
  labels:
    app: web
spec:
  clusterIP: None
  selector:
    app: web
  ports:
    - port: 80
      name: http
```

**02-statefulset.yaml**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: demo
spec:
  serviceName: web
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
              name: http
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
        # storageClassName: standard   # set if your cluster requires it
```

```bash
kubectl apply -f 01-service-headless.yaml
kubectl apply -f 02-statefulset.yaml
kubectl get sts,pvc,pods -n demo
```

**Pod names:** `web-0`, `web-1`, `web-2`. **PVCs:** `data-web-0`, `data-web-1`, …

---

## Scaling

```bash
kubectl scale statefulset/web --replicas=5 -n demo
```

- **`OrderedReady`**: new Pods created **in order**; scale-down terminates **highest ordinal first**.
- PVCs for removed replicas **usually remain** — clean up manually if needed (data safety).

---

## Updates & canary (`partition`)

```bash
kubectl apply -f 02-statefulset.yaml   # image or template change
kubectl rollout status statefulset/web -n demo
```

- Set **`partition`** to roll only **high ordinals** first (e.g. canary on `web-2` while `web-0`, `web-1` stay old).

---

## Delete behavior (important)

```bash
kubectl delete sts web -n demo
```

- Pods removed (order depends on policy).
- **PVCs are not deleted automatically** by default — they often **persist** so data isn’t lost accidentally.  
  Delete PVCs explicitly when you intend to wipe data:

```bash
kubectl delete pvc -l app=web -n demo   # example; verify labels
```

Always confirm **backup / retention** policies before deleting PVCs.

---

## Production best practices (industry standard)

1. **Headless Service** + correct **`serviceName`** — required for stable peer DNS in most patterns.  
2. **Pin images** — semver/digest; test **ordered** rollouts for your datastore.  
3. **Probes** — readiness before serving; liveness tuned to avoid flapping under load.  
4. **Storage class & size** — explicit `storageClassName`; right `accessModes` (usually **RWO** per node).  
5. **PodDisruptionBudget** — protect quorum (e.g. min available) during drains.  
6. **Backup/restore** — StatefulSet doesn’t replace DB backup strategy.  
7. **`partition` / `OnDelete`** — use when you need **manual** or **phased** upgrades (e.g. schema migrations).  
8. **Resource requests/limits** — predictable scheduling; avoid noisy neighbors.  
9. **Security** — `securityContext`, non-root where supported by the workload.  
10. **Don’t treat StatefulSet like Deployment** — scaling and updates have **ordering** semantics; test failure cases.  
11. **GitOps** — manifests in Git; same as Deployment for reviewability.

---

## Troubleshooting StatefulSets

### Quick checks

```bash
kubectl get sts -n <ns>
kubectl describe sts <name> -n <ns>
kubectl get pods -n <ns> -l app=<label>
kubectl get pvc -n <ns>
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

### Symptom → action

| Symptom | Check |
|---------|--------|
| **Pod stuck Pending** | PVC **Pending** (storage class, size, quota), scheduling (taints, resources) |
| **Pod `0` never ready** | First replica must pass readiness — blocks **OrderedReady** chain |
| **DNS doesn’t resolve** | Headless Service? `serviceName` matches? labels on Service selector? |
| **Wrong disk / lost data** | PVC name binding; **don’t** reuse PVC names incorrectly across STS |
| **Update not rolling** | **`OnDelete`** strategy — delete Pods to pick up template |
| **Scale stuck** | Previous Pod not Ready; events on STS and Pod |

### Useful

```bash
kubectl logs web-0 -n demo
kubectl exec -it web-0 -n demo -- /bin/sh
kubectl get sts web -n demo -o yaml | less
```

---

## When *not* to use StatefulSet

| Situation | Prefer |
|-----------|--------|
| Stateless HTTP API, interchangeable replicas | **Deployment** |
| One Pod per node (agents, monitoring) | **DaemonSet** |
| Batch / cron | **Job / CronJob** |

---

## Cheat sheet

| Task | Command / note |
|------|------------------|
| List | `kubectl get sts -n <ns>` |
| Describe | `kubectl describe sts <name> -n <ns>` |
| Scale | `kubectl scale sts <name> --replicas=N -n <ns>` |
| Status | `kubectl rollout status sts/<name> -n <ns>` |
| Delete STS | `kubectl delete sts <name> -n <ns>` |
| PVCs | `kubectl get pvc -n <ns>` — **survive** STS delete unless you delete them |
| DNS | `<pod>.<service>.<ns>.svc.cluster.local` |

---

## Summary

**StatefulSet** = **ordered**, **named** replicas + **per-Pod storage** + **stable DNS** via **Headless Service** and **`serviceName`**. **Create** with **YAML** (`apply`); **scale** with `kubectl scale`; **upgrade** with **RollingUpdate** / **`partition`** or **`OnDelete`** when you need control. **Respect PVC lifecycle** — data outlives the StatefulSet by design unless you delete PVCs on purpose.
