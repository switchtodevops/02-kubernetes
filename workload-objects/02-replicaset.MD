# Kubernetes ReplicaSet — Developer Guide

What a **ReplicaSet** is, how it relates to **Pods** and **Deployments**, how to **create and scale** it, **production guidance**, and **troubleshooting** — in a simple, scannable format.

---

## What is a ReplicaSet?

A **ReplicaSet (RS)** is a controller that **keeps a stable number of Pod replicas** running. It:

- Matches Pods using a **label selector**
- **Creates** Pods when count is low, **deletes** Pods when count is high
- **Replaces** Pods that fail or are deleted (self-healing)

**Mental model:** “Always run **N** Pods that look like this template.”

> **In production you usually do not create ReplicaSets by hand.** A **Deployment** creates and manages ReplicaSets for you (rolling updates, rollbacks, history). Use this doc to **understand** RS and for **CKA/CKAD**-style tasks.

---

## ReplicaSet vs Pod vs Deployment

| Object | Role |
|--------|------|
| **Pod** | One (or few) running instance(s) |
| **ReplicaSet** | Keeps **N** copies of that Pod template |
| **Deployment** | Manages **ReplicaSets** (versions), updates, rollback |

```
Deployment  →  ReplicaSet  →  Pods
   (you)          (managed)     (running workloads)
```

---

## Why do we need a ReplicaSet?

| Need | How ReplicaSet helps |
|------|------------------------|
| High availability | Multiple Pods across nodes |
| Self-healing | Recreates deleted/crashed Pods |
| Horizontal scale | Change `replicas` to scale in/out |

**Services** still target Pods by **labels** — ReplicaSet only ensures those Pods exist.

---

## How it works: selector + template labels (critical)

ReplicaSet has:

- **`spec.selector`** — which Pods this RS owns (must be **unique** in the namespace; no overlap with other controllers’ selectors)
- **`spec.template.metadata.labels`** — labels on Pods created from the template

**Rule:** Every label in `selector.matchLabels` **must** exist on `template.metadata.labels` with the **same values**. If they don’t match, the RS creates Pods that **don’t count** as its own → confusion and scaling bugs.

---

## Key fields (quick reference)

| Field | Purpose |
|-------|---------|
| `spec.replicas` | Desired number of Pods |
| `spec.selector` | Label query for owned Pods |
| `spec.template` | Pod spec (containers, volumes, probes, etc.) |

---

## Create a ReplicaSet

### Imperative (limited — scaling & inspection)

Core `kubectl` **does not** provide a one-liner like `kubectl create replicaset --image=...` (unlike some resources). Typical **imperative** operations:

```bash
# Scale an existing ReplicaSet
kubectl scale rs <rs-name> --replicas=3 -n <namespace>

# Autoscale (less common on bare RS)
kubectl autoscale rs <rs-name> --min=2 --max=10 --cpu-percent=80 -n <namespace>
```

**Creating** the object is almost always **declarative** (`kubectl apply -f file.yaml`). That’s the standard “as code” approach.

### Declarative (recommended for learning + exams)

**replicaset-demo.yaml**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  namespace: demo
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      tier: web
  template:
    metadata:
      labels:
        app: nginx
        tier: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

```bash
kubectl apply -f replicaset-demo.yaml
kubectl get rs -n demo
kubectl get pods -n demo -l app=nginx
```

**Delete:**

```bash
kubectl delete rs nginx-rs -n demo
```

> **Note:** If you use a **Deployment**, it creates a ReplicaSet with a **pod-template-hash** label — don’t fight that by hand; edit the Deployment instead.

---

## Inspecting ReplicaSets

```bash
kubectl get rs -n <namespace>
kubectl get rs -n <namespace> -o wide
kubectl describe rs <name> -n <namespace>
kubectl get rs <name> -n <namespace> -o yaml
```

See **owned Pods:**

```bash
kubectl get pods -n <namespace> -l app=nginx
```

---

## Production best practices (industry standard)

1. **Prefer Deployment over bare ReplicaSet** — rolling updates, rollbacks, and history need a Deployment.
2. **Don’t duplicate selectors** — two ReplicaSets with overlapping selectors → conflict.
3. **Pin images** — use specific tags or digests; avoid `:latest` in prod.
4. **Set Pod `resources`** (requests/limits) in the RS template — scheduling and stability.
5. **Use readiness/liveness probes** in the template — aligns with Services and restarts.
6. **GitOps / declarative** — manifests reviewed in PRs; avoid one-off manual RS in shared clusters.
7. **Scale via Deployment** in production (`kubectl scale deploy` or HPA on Deployment), not by editing child RS unless you know the implications.
8. **PodDisruptionBudgets** — protect availability during drains (usually defined with app lifecycle, often at Deployment workload level).

**Summary:** Treat **ReplicaSet** as an **implementation detail** of **Deployment** in real systems.

---

## Troubleshooting ReplicaSets

### 1) Desired replicas ≠ running

```bash
kubectl get rs -n <ns>
kubectl describe rs <name> -n <ns>
```

Check **Events** and **Replicas** (desired / current / ready).

### 2) Pods not created

- **Selector mismatch** — `selector` vs `template` labels
- **Quota / limits** — namespace ResourceQuota
- **Scheduling** — taints, node selectors, insufficient resources
- **Image pull** — wrong image or `imagePullSecrets`

### 3) Too many or wrong Pods

- Another controller with **overlapping selector**
- Manual Pods with same labels accidentally adopted (dangerous — avoid random labels)

### 4) Can’t scale or change image on “deployment-owned” RS

Edit the **Deployment**, not the child ReplicaSet — Deployment will reconcile.

### 5) Useful commands

```bash
kubectl get pods -n <ns> --show-labels
kubectl get rs -n <ns> -o yaml | less
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

---

## ReplicaSet vs Deployment (when to use what)

| Situation | Use |
|-----------|-----|
| Production apps, updates | **Deployment** |
| Learning / CKA task “create RS” | **ReplicaSet** YAML + `apply` |
| Raw scaling only, no rollout | Rare; still usually **Deployment** with one revision |

---

## Cheat sheet

| Topic | Takeaway |
|-------|----------|
| **ReplicaSet** | Keeps N matching Pods; self-heals |
| **Create** | `kubectl apply -f` with `kind: ReplicaSet` |
| **Scale** | `kubectl scale rs <name> --replicas=N` |
| **Production** | **Deployment** manages RS; don’t manage RS by hand |
| **Debug** | `describe rs`, label match, events, Pod status |

---

## Summary

A **ReplicaSet** ensures a **fixed number** of **labeled Pods** from a **template**. You **create** it with **declarative YAML** (`kubectl apply`); **scale** it with **`kubectl scale`**. For **production**, use a **Deployment** so updates and rollbacks are first-class — ReplicaSet stays the engine under the hood.
