# Kubernetes Service — Simple Guide

## What is a Service?

A **Service** is a stable **network endpoint** for a set of Pods. Pods are ephemeral (IPs change when they restart). A Service gives you a **fixed DNS name and virtual IP (ClusterIP)** that load-balances traffic to matching Pods using **labels**.

Think of it as: *“A front door that always works, even when the workers behind it change.”*

---

## Why do we need a Service?

| Problem | How Service helps |
|--------|-------------------|
| Pod IPs are not stable | Service provides a stable ClusterIP + DNS name |
| You have many replicas | Service distributes traffic across Pods |
| Other apps need to call your app | They use `ServiceName.Namespace.svc.cluster.local` instead of Pod IPs |
| Decoupling | Clients depend on the Service, not individual Pods |

**Without a Service**, you would track Pod IPs manually — fragile and not production-safe.

---

## Service types (at a glance)

| Type | Typical use | Exposure |
|------|-------------|----------|
| **ClusterIP** (default) | Internal traffic only | Inside cluster only |
| **NodePort** | Dev, quick tests, bare-metal hacks | Each node’s IP + high port |
| **LoadBalancer** | Cloud / managed LB | Internet (via cloud load balancer) |
| **ExternalName** | Point to external DNS | No proxy — DNS alias only |

---

## How Services find Pods: `selector` + `labels`

Pods must have **labels** that match the Service **`spec.selector`**. If labels don’t match, **Endpoints stay empty** and traffic won’t reach Pods.

---

## 1. ClusterIP (default)

**When:** Internal APIs, databases, backends — **most common in production** for east-west traffic.

### Imperative

```bash
kubectl expose deployment myapp --port=80 --target-port=8080 --name=myapp-svc -n demo
# Or create Service only (needs Pods with matching labels):
kubectl create service clusterip myapp-svc --tcp=80:8080 -n demo
```

### Declarative

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  namespace: demo
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - name: http
      port: 80        # port Service listens on
      targetPort: 8080 # port on Pod
```

```bash
kubectl apply -f myapp-svc.yaml
```

**Access from another Pod:** `http://myapp-svc.demo.svc.cluster.local` or `http://myapp-svc.demo` (short form inside same namespace: `http://myapp-svc`).

---

## 2. NodePort

**When:** Learning, local/minikube, or environments **without** Ingress/LoadBalancer — **not ideal as primary internet exposure** in production.

Exposes the Service on **each node’s IP** at a **static high port** (default range 30000–32767).

### Imperative

```bash
kubectl expose deployment myapp --type=NodePort --port=80 --target-port=8080 --name=myapp-np -n demo
# Optional: fix node port
kubectl expose deployment myapp --type=NodePort --port=80 --target-port=8080 --name=myapp-np -n demo --node-port=30080
```

### Declarative

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-np
  namespace: demo
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080   # optional; omit for auto-assign
```

---

## 3. LoadBalancer

**When:** You want a **cloud-managed load balancer** (AWS ELB, GCP LB, Azure LB) with a public (or internal) VIP — common for **north-south** traffic in cloud.

**Note:** On bare-metal / local clusters, behavior depends on your setup (e.g., MetalLB).

### Imperative

```bash
kubectl expose deployment myapp --type=LoadBalancer --port=80 --target-port=8080 --name=myapp-lb -n demo
```

### Declarative

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
  namespace: demo
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  # Optional on some clouds:
  # loadBalancerIP: x.x.x.x
```

**Check external IP:**

```bash
kubectl get svc myapp-lb -n demo -w
```

---

## 4. ExternalName

**When:** You want cluster DNS to **alias** an external hostname (no kube-proxy forwarding to Pods).

### Imperative

```bash
kubectl create service externalname my-db --external-name=db.example.com -n demo
```

### Declarative

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
  namespace: demo
spec:
  type: ExternalName
  externalName: db.example.com
```

Queries to `my-db.demo.svc.cluster.local` resolve like a **CNAME** to `db.example.com`.

---

## Headless Service (`clusterIP: None`)

**When:** StatefulSets, peer discovery, or **you want DNS per Pod** (e.g. `pod-0.myapp.default.svc.cluster.local`).

### Declarative

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
  namespace: demo
spec:
  clusterIP: None
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

Imperative creation is uncommon; **prefer YAML** for headless Services.

---

## Quick reference: create patterns

| Goal | Imperative hint | Declarative |
|------|-----------------|-------------|
| Internal only | `kubectl expose deploy ...` (default ClusterIP) | `type: ClusterIP` |
| Node access by port | `--type=NodePort` | `type: NodePort` |
| Cloud LB | `--type=LoadBalancer` | `type: LoadBalancer` |
| External DNS alias | `kubectl create service externalname ...` | `type: ExternalName` |

---

## Production best practices (industry standard)

1. **Default to ClusterIP** for app-to-app traffic inside the cluster.
2. **Expose HTTP(S) to users with Ingress** (or Gateway API), not raw NodePort — better TLS, routing, and policies.
3. **Use LoadBalancer** when you need a cloud LB (or internal LB annotations on some clouds) — document cost and security groups.
4. **Name ports** (`name: http`) — helps with probes, Ingress backends, and readability.
5. **Align `selector` with Deployment/Pod labels** — verify with `kubectl get endpoints -n <ns>` (or EndpointSlices).
6. **Avoid unnecessary public Services** — reduce attack surface; use NetworkPolicies with Services.
7. **Session affinity** (`sessionAffinity: ClientIP`) only when required — understand stickiness vs. scaling.
8. **Headless + StatefulSets** for stable network identity per replica.
9. **Prefer declarative YAML** in Git (GitOps) — imperative is fine for quick tests only.
10. **Document `port` vs `targetPort`** — many incidents come from mixing them up.

---

## Troubleshooting (crispy checklist)

```bash
kubectl get svc -n <namespace>
kubectl get endpoints <svc-name> -n <namespace>   # should list Pod IPs
kubectl describe svc <svc-name> -n <namespace>
```

- **No endpoints** → label mismatch or no ready Pods.
- **Connection refused** → wrong `targetPort` or app not listening.
- **Works inside cluster, not outside** → check type (ClusterIP vs LB/Ingress), firewall, cloud LB health.

---

## One-line summary

> A **Service** is the stable network abstraction for dynamic Pods; choose **ClusterIP** for internal traffic, **Ingress/Gateway** for HTTP from outside, **LoadBalancer** when you need a cloud LB, and **ExternalName** to point DNS at external systems.
