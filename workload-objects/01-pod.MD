# Kubernetes Pod — Developer Guide

Everything you need to **understand**, **create**, **run safely in production**, and **troubleshoot** Pods — in a simple, scannable format.

---

## What is a Pod?

A **Pod** is the **smallest deployable unit** in Kubernetes. It wraps **one or more containers** that share:

- **Network** — one IP address per Pod; containers talk via `localhost` on different ports
- **Storage** — shared volumes
- **Lifecycle** — scheduled and restarted together on the same node

**Mental model:** A Pod is like a **logical host** for your workload. You usually run **one main app container** per Pod; extra containers are **sidecars** (logging, proxy, mesh).

> **You rarely create bare Pods in production.** Prefer **Deployments**, **StatefulSets**, or **Jobs** — they manage Pod replicas, updates, and rollbacks for you.

---

## Pod vs Container

| Concept | Meaning |
|--------|---------|
| **Container** | Your actual process (image + command) |
| **Pod** | Wrapper that schedules containers together on one node |
| **Pause container** | Hidden “infra” container that holds the Pod’s network namespace (you don’t manage it directly) |

---

## Why Pods exist (not just containers)

- Kubernetes **schedules** and **tracks** workloads at Pod granularity.
- **Labels** on Pods drive **Services** (traffic) and **controllers** (replicas).
- **Volumes**, **secrets**, and **config** attach at Pod level.

---

## Pod phases (status)

| Phase | Meaning |
|-------|---------|
| **Pending** | Not running yet (scheduling, image pull, volumes) |
| **Running** | At least one container is running or starting |
| **Succeeded** | All containers terminated successfully (batch) |
| **Failed** | At least one container failed |
| **Unknown** | Node unreachable, state unclear |

Use `kubectl get pod -o wide` and `kubectl describe pod` for details.

---

## Multi-container Pods

- **Sidecar** — e.g. log shipper next to app
- **Adapter** — normalize metrics/logs for external systems
- **Ambassador** — proxy outbound calls

**Rule:** Only co-locate containers that **must share network/storage** or **tight coupling** — otherwise use separate Deployments/Services.

---

## Init containers

Run **before** app containers start, **in order**. Use for:

- DB migrations, config download, permission fixes

If an init container fails, the Pod stays **not Ready**.

---

## Probes (health)

| Probe | Purpose |
|-------|---------|
| **liveness** | Restart container if unhealthy |
| **readiness** | Remove Pod from Service endpoints if not ready |
| **startup** | Slow-start apps — delays liveness until app is up |

**Production:** Always set **readiness** for anything behind a Service; tune **liveness** carefully (wrong config = restart loops).

---

## Resources (requests & limits)

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

- **Requests** — used for **scheduling** (which node fits)
- **Limits** — cap usage (**OOMKill** if memory exceeds limit)

**Best practice:** Set requests = expected steady load; limits to prevent noisy neighbors (policy-dependent).

---

## Labels & annotations

- **Labels** — selectors for Services, Deployments (`app`, `version`, `tier`)
- **Annotations** — metadata only (not for selectors), e.g. `kubectl.kubernetes.io/restartedAt`

---

## How Pods get traffic

Pods get traffic **only via a Service** (ClusterIP, etc.) matching **labels**. No matching Service + ready Pods = no traffic.

---

## Create a Pod

### Imperative (quick / exams / demos)

**Single Pod (no Deployment):**

```bash
kubectl run my-pod --image=nginx:1.25 --restart=Never -n demo
```

**With labels and port (still imperative):**

```bash
kubectl run my-pod --image=nginx:1.25 --restart=Never -n demo \
  --labels=app=demo,tier=web \
  --port=80
```

**From a one-off command:**

```bash
kubectl run debug --image=busybox:1.36 --restart=Never -n demo -- sleep 3600
```

**Delete:**

```bash
kubectl delete pod my-pod -n demo
```

> `kubectl run` with `--restart=Always` creates a **Deployment** (not a bare Pod) on modern kubectl — check with `kubectl get deploy,po`.

### Declarative (recommended for Git / production patterns)

**minimal-pod.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: demo
  labels:
    app: demo
    tier: web
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 256Mi
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 3
        periodSeconds: 5
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10
```

```bash
kubectl apply -f minimal-pod.yaml
```

**Production note:** For real apps, put the same `spec.template` inside a **Deployment** — bare `Pod` YAML is for learning or special cases.

---

## Production best practices (industry standard)

1. **Prefer Deployments / StatefulSets / Jobs** over standalone Pods — self-healing and rolling updates.
2. **Pin image tags** — use digest or semver (`nginx:1.25-alpine`), avoid `:latest` in prod.
3. **Set requests and limits** — predictable scheduling; avoid OOM surprises.
4. **Readiness + liveness** — readiness for Service membership; conservative liveness.
5. **SecurityContext** — run as non-root, read-only root FS where possible, drop capabilities.
6. **One main workload per Pod** unless sidecar pattern is justified.
7. **Labels** consistent with Service selectors and org standards.
8. **Graceful shutdown** — handle `SIGTERM`; set `terminationGracePeriodSeconds` if needed.
9. **GitOps / declarative** — manifests in version control, reviewed like code.
10. **PodDisruptionBudgets** — for HA during node drains (with controllers, not lone Pods).

---

## Troubleshooting Pods (developer playbook)

### 1) Quick status

```bash
kubectl get pods -n <ns> -o wide
kubectl get pods -n <ns> --field-selector=status.phase!=Running
kubectl describe pod <name> -n <ns>
```

Check **Events** at the bottom of `describe` — scheduling, pull, mount errors.

### 2) Common states

| Symptom | What to check |
|---------|----------------|
| **Pending** | Resources, node selectors, taints, PVC binding, quota |
| **ImagePullBackOff** | Image name, tag, registry auth (`imagePullSecrets`) |
| **CrashLoopBackOff** | `kubectl logs`, wrong command, missing env/config |
| **OOMKilled** | Memory limits vs usage; `kubectl describe` events |
| **Error / CreateContainerConfigError** | Secret/ConfigMap missing, wrong mount path |
| **Not ready** | Readiness probe failing — logs + probe path/port |

### 3) Logs

```bash
kubectl logs <pod> -n <ns>
kubectl logs <pod> -c <container> -n <ns>
kubectl logs <pod> --previous -n <ns>   # crashed container
kubectl logs -f <pod> -n <ns>
```

### 4) Exec into Pod

```bash
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl exec <pod> -n <ns> -- nslookup kubernetes.default
```

### 5) Port forward (reach a Pod/Service from your laptop)

`kubectl port-forward` opens a **tunnel** from **localhost** (or another local address) to a **port inside the Pod** (or via **Deployment** / **Service**, which pick a backing Pod). The command **blocks** until you press **Ctrl+C**; it needs a working `kubectl` context.

**Syntax:** `LOCAL_PORT:REMOTE_PORT` — traffic to `localhost:LOCAL_PORT` goes to the container’s `REMOTE_PORT`.

```bash
# Pod — e.g. nginx listening on 80 inside the Pod
kubectl port-forward pod/my-pod 8080:80 -n demo
# Then: curl http://localhost:8080   or open http://localhost:8080 in a browser

# Same thing using resource/name form
kubectl port-forward pod/my-pod 8080:80 -n demo

# Deployment — forwards to a Pod owned by the Deployment (good when Pod name changes)
kubectl port-forward deployment/myapp 9000:8080 -n demo

# Service — forwards to a Pod selected by the Service (ClusterIP works; no Ingress/LB required for local testing)
kubectl port-forward svc/myapp-svc 8080:80 -n demo

# Multiple ports in one command
kubectl port-forward pod/my-pod 8080:80 8443:443 -n demo

# Listen on all interfaces (only if you need LAN access; avoid on untrusted networks)
kubectl port-forward --address 0.0.0.0 svc/myapp-svc 8080:80 -n demo
```

**Tip:** For a **multi-container** Pod, port-forward targets the Pod’s network namespace (container port must be listening). Use `kubectl exec` if you need a specific container’s process.

### 6) Debug copy / ephemeral debug (1.23+)

```bash
kubectl debug <pod> -n <ns> -it --image=busybox --target=<container>
```

### 7) Network / Service

```bash
kubectl get svc,endpoints -n <ns>
# From another Pod: curl http://<service>:<port>
```

### 8) YAML / diff

```bash
kubectl get pod <name> -n <ns> -o yaml
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

---

## Cheat sheet: one glance

| Topic | Takeaway |
|-------|----------|
| **Pod** | Smallest K8s unit; 1+ containers sharing network & volumes |
| **Production** | Use controllers + probes + resources + pinned images |
| **Create** | `kubectl run … --restart=Never` or `apply` YAML |
| **Debug** | `describe` → events → `logs` → `exec` → `port-forward` → endpoints |

---

## Summary

A **Pod** runs your containers on a node with shared networking and storage. **Create** it imperatively for quick tests or declaratively for real workflows — but in production, **let a Deployment own your Pod template** so Kubernetes can **keep your app running** while you focus on **images, probes, resources, and troubleshooting** using the flow above. Use **`kubectl port-forward`** when you need to hit an app port from your machine without a public Service.
