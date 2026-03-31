# Kubernetes Ingress — Developer Guide

Ingress gives HTTP/HTTPS routing into your cluster using host/path rules, TLS, and one public entrypoint. This guide is practical, concise, and production-focused.

---

## Summary highlights (quick reference)

- **Ingress is L7 (HTTP/HTTPS) routing**, not raw TCP/UDP load balancing.
- Ingress resource needs an **Ingress Controller** (NGINX, Traefik, cloud controller, etc.) to work.
- Typical flow: **Client -> LoadBalancer/NodePort -> Ingress Controller -> Service -> Pod**.
- Use **Ingress** for host/path routing, TLS termination, redirects, and shared entrypoint.
- Use **Service type LoadBalancer** when you need direct L4 exposure for one service.
- In production: enforce TLS, add health checks, request limits, and clear ownership per hostname.
- Prefer one stable Ingress class and controller per environment unless you intentionally run multiple.

---

## What is Ingress?

An **Ingress** is a Kubernetes API object that defines **HTTP/HTTPS routing rules** from outside the cluster to internal Services.

It lets you route by:
- **Host** (`api.example.com`, `app.example.com`)
- **Path** (`/api`, `/admin`)
- **TLS** (certificates for HTTPS)

Ingress itself is only configuration. Real traffic handling is done by an **Ingress Controller**.

---

## Ingress vs Service (important)

| Need | Use |
|------|-----|
| Internal stable endpoint | `Service` (usually `ClusterIP`) |
| Expose one service directly at L4 | `Service type LoadBalancer` |
| Expose many HTTP apps behind one entrypoint with host/path routing | `Ingress` + Ingress Controller |

Rule of thumb:
- **Service** = connectivity abstraction for Pods
- **Ingress** = external HTTP routing policy

---

## Prerequisite: Ingress Controller

Without an Ingress Controller, creating an Ingress does nothing.

Common controllers:
- NGINX Ingress Controller
- Traefik
- Cloud-specific controllers (varies by platform)

Check installed ingress classes:

```bash
kubectl get ingressclass
```

---

## Core Ingress concepts

### 1) `ingressClassName`

Tells Kubernetes which controller should process this Ingress.

```yaml
spec:
  ingressClassName: nginx
```

### 2) Rules

Each rule maps host/path to a backend Service.

```yaml
rules:
  - host: app.example.com
    http:
      paths:
        - path: /
          pathType: Prefix
```

### 3) Path types

| Type | Meaning |
|------|---------|
| `Prefix` | Matches by path prefix (most common) |
| `Exact` | Exact path only |
| `ImplementationSpecific` | Controller-dependent behavior |

### 4) TLS

Use a Secret containing cert + key (`kubernetes.io/tls`).

```yaml
tls:
  - hosts: [app.example.com]
    secretName: app-tls
```

---

## Minimal Ingress example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: demo
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-svc
                port:
                  number: 80
```

---

## Production-standard example (host + path + TLS + redirect)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: platform-ingress
  namespace: prod
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
        - api.example.com
      secretName: platform-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1-svc
                port:
                  number: 8080
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-svc
                port:
                  number: 8080
```

Notes:
- Annotation keys are controller-specific (example above is for NGINX ingress controller).
- Keep app routing simple: one team owns one host whenever possible.

---

## End-to-end setup flow (quick)

1. Deploy app Pods
2. Expose each app with `Service` (`ClusterIP`)
3. Install / verify Ingress Controller
4. Create TLS Secret
5. Apply Ingress rules
6. Point DNS (`A`/`CNAME`) to controller external address

---

## Common kubectl commands

```bash
# List ingress resources
kubectl get ingress -A
kubectl get ing -n <namespace>

# Detailed info + events
kubectl describe ingress <name> -n <namespace>

# Full YAML
kubectl get ingress <name> -n <namespace> -o yaml

# Ingress classes
kubectl get ingressclass

# Apply / delete ingress manifests
kubectl apply -f ingress.yaml
kubectl delete -f ingress.yaml

# Check backend services/endpoints
kubectl get svc -n <namespace>
kubectl get endpoints -n <namespace>

# Check controller pods (example namespace/name)
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=200
```

---

## Practical use cases

1. **One domain, multiple services**  
   `example.com/app`, `example.com/api`, `example.com/admin`

2. **Multiple domains, same cluster**  
   `shop.example.com` -> shop service, `api.example.com` -> api service

3. **TLS offloading at edge**  
   Terminate HTTPS in controller, send HTTP/HTTPS internally per policy

4. **Blue/green or canary (controller features)**  
   Controller-specific annotations or advanced routing config

---

## Production best practices

1. **Always use TLS** in production; redirect HTTP -> HTTPS.
2. **Use DNS + cert automation** (for example cert-manager) instead of manual cert rotation.
3. **Keep one clear ingress class** per traffic plane unless intentionally segmented.
4. **Use `ClusterIP` backends** behind Ingress; avoid exposing backend services directly.
5. **Set resource limits and HPA** on the ingress controller.
6. **Enable access logs and metrics**; monitor 4xx/5xx, latency, and saturation.
7. **Define request size and timeout limits** to protect upstream services.
8. **Use NetworkPolicies** and namespace boundaries to reduce blast radius.
9. **Version and review ingress changes** carefully; routing mistakes cause outages.
10. **Avoid giant monolithic ingress files**; split by domain/team with clear ownership.

---

## Troubleshooting checklist

### 1) Ingress has no external address

```bash
kubectl get ing -n <ns>
kubectl get svc -n ingress-nginx
```

Check if controller service has external IP/hostname.

### 2) Hostname resolves but wrong backend

```bash
kubectl describe ing <name> -n <ns>
kubectl get svc,endpoints -n <ns>
```

Check host/path rules, service name, and service port.

### 3) 404/503 from ingress

- 404: rule/path mismatch
- 503: backend service has no healthy endpoints

```bash
kubectl get endpoints <svc-name> -n <ns>
kubectl get pods -n <ns> -l app=<label>
```

### 4) TLS issues

```bash
kubectl get secret <tls-secret> -n <ns>
kubectl describe ing <name> -n <ns>
```

Check cert secret name, hostnames, expiration, and DNS.

### 5) Controller-level errors

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=300
```

Look for invalid annotation, upstream timeout, or config reload errors.

---

## Ingress limitations (know this)

- Ingress is primarily for **HTTP/HTTPS**.
- Advanced traffic policy behavior is often **controller-specific**.
- For modern advanced routing and portability, evaluate **Gateway API** over time.

---

## Quick cheat sheet

```bash
kubectl get ing -A
kubectl describe ing <name> -n <ns>
kubectl get ingressclass
kubectl apply -f ingress.yaml
kubectl get svc,endpoints -n <ns>
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=200
```

---

## Summary

Ingress is the standard Kubernetes way to expose multiple HTTP/HTTPS services behind one entrypoint with host/path routing and TLS. The Ingress resource defines rules; the Ingress Controller enforces them. In production, secure it with TLS, observability, strict limits, and clear ownership of domains and routes.
