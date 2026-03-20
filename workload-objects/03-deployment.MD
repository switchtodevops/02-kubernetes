# Kubernetes Deployment — Developer Guide

**Deployment** is the standard way to run **stateless** apps in Kubernetes: **declarative updates**, **rolling rollouts**, **rollbacks**, and **ReplicaSet** management under the hood — crisp, complete, and easy to scan.

---

## What is a Deployment?

A **Deployment** is a controller that:

- Declares **desired Pod template** + **replica count**
- Owns one **active ReplicaSet** per “version” (revisions)
- Performs **rolling** or **recreate** updates when the template changes
- Supports **rollback** to previous ReplicaSets

```
Deployment  →  ReplicaSet(s)  →  Pods
   (you edit)   (revisions)       (running workloads)
```

**Mental model:** “Run **N** copies of **this** app, and **safely change** the app over time.”

---

## Deployment vs ReplicaSet vs Pod

| Object | You usually… |
|--------|----------------|
| **Pod** | Don’t create alone in prod (no self-healing at workload level) |
| **ReplicaSet** | Don’t create alone (no rollout/rollback) |
| **Deployment** | **Yes** — primary API for stateless long-running workloads |

---

## Core concepts (don’t skip)

### 1) Selector + Pod template labels

- **`spec.selector`** — immutable after creation (for `matchLabels`); must match **`spec.template.metadata.labels`**
- Each Pod gets extra labels (e.g. `pod-template-hash`) so revisions don’t fight each other

### 2) Revisions = ReplicaSets

- Each template change can create a **new ReplicaSet** (new revision)
- Old RS may remain for **history** / rollback (see `revisionHistoryLimit`)

### 3) Update strategy

| Strategy | Behavior | When |
|----------|----------|------|
| **RollingUpdate** (default) | Replace Pods gradually | Most apps |
| **Recreate** | Kill all old, then start new | Dev / single-replica / not compatible with rolling |

**RollingUpdate knobs:**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # extra Pods above desired during update
      maxUnavailable: 25%  # Pods that can be down during update
```

- **`maxSurge`**: burst capacity (e.g. `1` or `25%`)
- **`maxUnavailable`**: allowed downtime during rollout (e.g. `0` for zero-downtime if cluster has room)

### 4) Other important `spec` fields

| Field | Purpose |
|-------|---------|
| `replicas` | Desired Pod count |
| `revisionHistoryLimit` | How many old ReplicaSets to keep (default 10); `0` = no history for rollback |
| `progressDeadlineSeconds` | Time after which “stalled” rollout is reported failed (default 600) |
| `minReadySeconds` | Wait before Pod counts as available (optional) |
| `paused` | Stop rollouts (`kubectl rollout pause`) — canary / manual control |

---

## Create a Deployment

### Imperative (fast / scripts / exams)

```bash
# Basic
kubectl create deployment myapp --image=nginx:1.25 -n demo

# Replicas + labels + port metadata
kubectl create deployment myapp --image=nginx:1.25 --replicas=3 -n demo
kubectl label deployment myapp tier=web -n demo --overwrite

# Scale
kubectl scale deployment myapp --replicas=5 -n demo

# Change image (triggers rollout)
kubectl set image deployment/myapp nginx=nginx:1.26 -n demo

# Resources (example — container name often matches deployment name or "nginx")
kubectl set resources deployment myapp -n demo --limits=cpu=500m,memory=512Mi --requests=cpu=100m,memory=128Mi

# Rollout control
kubectl rollout status deployment/myapp -n demo
kubectl rollout history deployment/myapp -n demo
kubectl rollout undo deployment/myapp -n demo
kubectl rollout undo deployment/myapp --to-revision=2 -n demo
kubectl rollout restart deployment/myapp -n demo
kubectl rollout pause deployment/myapp -n demo
kubectl rollout resume deployment/myapp -n demo
```

**Expose Service (imperative):**

```bash
kubectl expose deployment myapp --port=80 --target-port=8080 --name=myapp-svc -n demo
```

**Delete:**

```bash
kubectl delete deployment myapp -n demo
```

### Declarative (GitOps / production)

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: demo
  labels:
    app: myapp
spec:
  replicas: 3
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: nginx:1.25-alpine
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
```

```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods -n demo
```

---

## Rollout: what happens on `apply` / image change

1. New **ReplicaSet** created for new Pod template  
2. **RollingUpdate** adds new Pods (within `maxSurge`) and terminates old (within `maxUnavailable`)  
3. Deployment **status** tracks `updatedReplicas`, `availableReplicas`, conditions  
4. If stuck beyond **`progressDeadlineSeconds`** → failed condition; investigate image, probes, quota, scheduling

---

## Scaling

```bash
kubectl scale deployment myapp --replicas=10 -n demo
kubectl autoscale deployment myapp --min=2 --max=20 --cpu-percent=70 -n demo
```

**HPA** usually targets **Deployment** (`scaleTargetRef.kind: Deployment`).

---

## History & rollback

```bash
kubectl rollout history deployment/myapp -n demo
kubectl rollout history deployment/myapp --revision=3 -n demo
kubectl rollout undo deployment/myapp -n demo
kubectl rollout undo deployment/myapp --to-revision=2 -n demo
```

**Note:** Meaningful history often needs **change-cause** annotations (optional):

```bash
kubectl annotate deployment/myapp kubernetes.io/change-cause="v1.2.3" -n demo --overwrite
```

---

## Production best practices (industry standard)

1. **One Deployment per app component** — clear ownership and scaling boundaries  
2. **Pinned images** — semver or digest; avoid `:latest`  
3. **Probes** — readiness for traffic; careful liveness to avoid restart storms  
4. **Resources** — requests/limits set; align with HPA if used  
5. **RollingUpdate tuning** — `maxUnavailable: 0` only works if surge capacity exists  
6. **`revisionHistoryLimit`** — balance rollback vs etcd clutter (often 5–10)  
7. **PDB** — `PodDisruptionBudget` for HA during node drains  
8. **Anti-affinity / topology** — spread replicas across nodes/zones when needed  
9. **GitOps** — manifests in Git; `kubectl apply` / Argo CD / Flux  
10. **Don’t edit child ReplicaSets** — always edit **Deployment**  
11. **Security** — `securityContext`, non-root, read-only root FS where possible  
12. **Labels** — stable selectors; version labels optional for observability  

---

## Troubleshooting Deployments

### Quick checks

```bash
kubectl get deploy -n <ns>
kubectl describe deploy <name> -n <ns>
kubectl get rs -n <ns>
kubectl get pods -n <ns>
kubectl rollout status deploy/<name> -n <ns>
```

### Symptom → action

| Symptom | Check |
|---------|--------|
| **Rollout stuck / ProgressDeadlineExceeded** | `describe deploy`, Pod events, image pull, probes, quota, nodes |
| **New Pods CrashLoop** | `kubectl logs`, wrong command/env, readiness too aggressive |
| **Old Pods won’t terminate** | PDB too strict, `terminationGracePeriod`, preStop hooks |
| **0 ready replicas** | Readiness probe, app not listening on `targetPort` |
| **HPA not scaling** | metrics-server, requests set, HPA `minReplicas`/refs |

### Useful

```bash
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl logs deploy/myapp -n <ns> --tail=100
kubectl logs deploy/myapp -n <ns> --previous
```

---

## When *not* to use Deployment

| Use instead | Reason |
|-------------|--------|
| **StatefulSet** | Stable identity, ordered rollout, sticky storage |
| **DaemonSet** | Exactly one Pod per node |
| **Job / CronJob** | Run-to-completion work |

---

## Cheat sheet

| Task | Command |
|------|---------|
| Create | `kubectl create deploy NAME --image=IMG -n NS` |
| Apply YAML | `kubectl apply -f deployment.yaml` |
| Scale | `kubectl scale deploy/NAME --replicas=N -n NS` |
| New image | `kubectl set image deploy/NAME container=IMG -n NS` |
| Status | `kubectl rollout status deploy/NAME -n NS` |
| History | `kubectl rollout history deploy/NAME -n NS` |
| Rollback | `kubectl rollout undo deploy/NAME -n NS` |
| Restart | `kubectl rollout restart deploy/NAME -n NS` |
| Pause / resume | `kubectl rollout pause|resume deploy/NAME -n NS` |

---

## Summary

**Deployment** = desired state for **stateless** apps: **replicas**, **Pod template**, **update strategy**, and **revision history**. Prefer **declarative YAML** in production; use **rollout** commands for **status, history, undo, restart**. Own the full loop: **image + probes + resources + strategy + PDB** — that’s the day-to-day Kubernetes deployment story.
