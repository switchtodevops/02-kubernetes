# Kubernetes DaemonSet — Developer Guide

**DaemonSet** runs **exactly one Pod per matching node** (cluster-wide or a subset). Use it for **node agents**, **observability**, **network/CNI plugins**, and anything that must **follow the node fleet** — same style as your other workload docs: **complete, crisp, scannable**.

---

## What is a DaemonSet?

A **DaemonSet** is a controller that:

- Ensures a **Pod template** runs on **every eligible node** (or every node matching **node selectors / affinity**)
- **Adds** a Pod when a new node becomes eligible; **removes** when a node is drained or no longer matches
- Does **not** use a `replicas` field — **Pod count follows nodes**, not a fixed number

```
DaemonSet  →  Pod on node-1, Pod on node-2, Pod on node-3, …
```

**Mental model:** “**One copy of this workload on each node** (that should run it).”

---

## DaemonSet vs Deployment vs StatefulSet

| Workload | Pod placement |
|----------|----------------|
| **Deployment** | **N** replicas anywhere in the cluster (scheduler decides) |
| **StatefulSet** | **N** ordered replicas with stable identity + often PVCs |
| **DaemonSet** | **One Pod per node** (among matching nodes) — not “N replicas” in the Deployment sense |

| Question | Answer |
|----------|--------|
| Need **same count on every node**? | **DaemonSet** |
| Need **N identical replicas** (scaling number)? | **Deployment** |
| Need **ordered / named / storage per replica**? | **StatefulSet** |

---

## Common use cases

- **Logs / metrics agents** (Fluent Bit, Fluentd, node-exporter, Datadog agent)
- **Storage daemons** (CSI node plugin, Ceph/OSD patterns — product-specific)
- **Network** — CNI node components, service mesh sidecar injectors at node level (patterns vary)
- **Security / compliance** — node scanners, audit agents
- **GPU / hardware drivers** — when every node with a label must run a driver Pod

---

## Core concepts (don’t skip)

### 1) “How many Pods?” = “How many matching nodes?”

- There is **no** `spec.replicas` on DaemonSet.
- Adding a **node** → DaemonSet tries to schedule **one more** Pod (if template + resources allow).
- Removing / cordoning a node → that node’s Pod goes away (normal lifecycle).

### 2) Selector + template labels

- **`spec.selector.matchLabels`** must match **`spec.template.metadata.labels`**.
- **Selector is immutable** after create (same as Deployment).

### 3) Which nodes get a Pod?

By default: **all nodes** except those **excluded by the scheduler** (e.g. some control-plane nodes may be skipped unless you add **tolerations**).

To limit scope:

- **`spec.template.spec.nodeSelector`** — simple label match
- **`affinity`** / **`nodeAffinity`** — richer rules (e.g. only `workload=gpu`)

### 4) Taints and tolerations (critical)

Many clusters **taint** control-plane or special nodes. If your DaemonSet must run there (e.g. monitoring **all** nodes including masters), add matching **`tolerations`** in the Pod template.

**Typical gap:** “DaemonSet only runs on workers” → missing toleration for `node-role.kubernetes.io/control-plane` (or legacy `master`).

### 5) Update strategy

| `type` | Behavior |
|--------|----------|
| **`RollingUpdate`** (default) | Update Pods in a rolling way; tune **`maxUnavailable`** (often `1` or `%`) |
| **`OnDelete`** | New template applies **when each Pod is deleted** manually |

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # or e.g. 10%
```

### 6) Other useful fields

| Field | Purpose |
|-------|---------|
| `revisionHistoryLimit` | Old DaemonSet revisions to keep (like Deployment) |
| `minReadySeconds` | Wait before Pod counts available during rollout (optional) |

---

## Create a DaemonSet

### Imperative (limited)

Core `kubectl` **does not** provide a full `kubectl create daemonset --image=...` in all versions. Day-to-day commands:

```bash
kubectl get daemonset -n <namespace>
kubectl get ds -n <namespace>                    # short name
kubectl describe ds <name> -n <namespace>

kubectl rollout status daemonset/<name> -n <namespace>
kubectl rollout history daemonset/<name> -n <namespace>

kubectl delete ds <name> -n <namespace>
```

**Note:** **`kubectl scale`** does **not** apply to DaemonSet — replica count is **not** user-controlled.

**Create** = **`kubectl apply -f`** with YAML.

### Declarative (recommended)

**daemonset-demo.yaml**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
  namespace: demo
  labels:
    app: log-agent
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: log-agent
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      # Example: only nodes labeled pool=workers (remove nodeSelector for all nodes)
      # nodeSelector:
      #   pool: workers
      tolerations:
        # Example: allow scheduling on control-plane if your cluster uses this taint
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: agent
          image: busybox:1.36
          command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          securityContext:
            runAsNonRoot: true
            runAsUser: 65534
```

```bash
kubectl apply -f daemonset-demo.yaml
kubectl get ds,pods -n demo -o wide
```

---

## Inspecting DaemonSets

```bash
kubectl get ds -A
kubectl get pods -n <ns> -l app=log-agent -o wide   # one per node (expected)
kubectl describe ds log-agent -n demo
```

Compare **desired** vs **ready** in `kubectl get ds` output — mismatches often mean **taints**, **image pull**, or **resource** issues on specific nodes.

---

## Production best practices (industry standard)

1. **Tolerations** — align with **which nodes** must run the agent (workers only vs entire fleet).  
2. **Resources** — set **requests/limits**; DaemonSets multiply by **node count** → cluster-wide impact.  
3. **PriorityClass** — avoid eviction of critical node agents if your platform uses priorities.  
4. **Update strategy** — tune **`maxUnavailable`** for blast radius vs rollout speed.  
5. **Security** — minimal RBAC, non-root where possible, read-only roots where applicable.  
6. **Pin images** — digest or semver; rolling updates affect **every node** over time.  
7. **Node selectors** — restrict to **GPU**, **region**, or **worker pool** when the agent is not needed everywhere.  
8. **Monitoring** — alert on **not-ready** DaemonSet Pods or rollout stuck.  
9. **GitOps** — manifests in Git; review changes that touch **all nodes**.  
10. **Avoid duplicating DaemonSets** — two DS with overlapping placement can fight for resources or confuse operators.

---

## Troubleshooting DaemonSets

### Quick checks

```bash
kubectl get ds -n <ns>
kubectl describe ds <name> -n <ns>
kubectl get pods -n <ns> -l <daemonset-labels> -o wide
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

### Symptom → action

| Symptom | Check |
|---------|--------|
| **Fewer Pods than nodes** | **Taints** without **tolerations**; **nodeSelector** too narrow; **Insufficient resources** |
| **Pod on wrong nodes** | **nodeSelector** / **affinity** rules |
| **Not updating** | **`OnDelete`** — delete Pods to roll forward |
| **Rollout stuck** | Image pull, probe failures, PDB (rare for DS), node NotReady |
| **OOM / CPU throttle** | Limits too low × **many nodes** |

### Per-node debug

```bash
kubectl get pods -n <ns> -o wide    # compare NODE column to cluster nodes
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
```

---

## When *not* to use DaemonSet

| Situation | Prefer |
|-----------|--------|
| Fixed **N** replicas independent of node count | **Deployment** |
| Ordered replicas + stable IDs + PVCs | **StatefulSet** |
| One-shot batch work | **Job** |

---

## Cheat sheet

| Task | Command / note |
|------|----------------|
| List | `kubectl get ds -n <ns>` or `-A` |
| Describe | `kubectl describe ds <name> -n <ns>` |
| Rollout | `kubectl rollout status ds/<name> -n <ns>` |
| History | `kubectl rollout history ds/<name> -n <ns>` |
| Delete | `kubectl delete ds <name> -n <ns>` |
| Scale | **Not supported** — placement drives count |
| Nodes | `kubectl get nodes` vs Pod count on each node |

---

## Summary

**DaemonSet** = **one Pod per matching node** — ideal for **per-node agents** and **infrastructure** workloads. **Create** with **`kubectl apply`**, control **where** it runs with **selectors/affinity** and **tolerations**, and **roll** updates with **`RollingUpdate`** / **`maxUnavailable`** or **`OnDelete`**. Watch **resources** and **taints** — they cause most real-world DaemonSet gaps.
