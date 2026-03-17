# Kubernetes Secrets - Complete Practical Guide

## 1) What is a Secret in Kubernetes?

A **Secret** is a Kubernetes object used to store a small amount of sensitive data, such as:
- passwords
- API tokens
- SSH private keys
- TLS certificates and private keys
- container registry credentials

Why Secrets exist:
- Keep confidential values outside container images and application code.
- Separate secret lifecycle from Pod lifecycle.
- Allow cluster-level controls (RBAC, encryption, auditing) around sensitive data.

> Important: Base64 in Secret `data` is **encoding**, not encryption.

---

## 2) How Secrets Work Internally

1. You create a `Secret` object in the API server.
2. It is stored in etcd (by default as plain data unless encryption at rest is enabled).
3. A Pod references that Secret (as volume, env var, or `imagePullSecrets`).
4. Kubelet on a node fetches only Secrets needed by Pods on that node.
5. When mounted as volume, Secret data is stored in node memory (`tmpfs`) and projected into containers.
6. If Secret data changes, mounted volumes update eventually (except `subPath` mount case).

---

## 3) Why Secrets Matter in Production

In real clusters, Secrets are critical for:
- database credentials for microservices
- JWT signing keys
- third-party API tokens (payments, messaging, observability)
- internal mTLS cert/key pairs
- private image registry pull credentials

If Secrets are mishandled, blast radius is high:
- leaked DB credentials
- compromised service accounts
- unauthorized image pulls or pushes
- lateral movement across workloads

---

## 4) Built-in Secret Types and Use Cases

Kubernetes supports these common Secret types:

| Type | Purpose | Common Real Use Case |
|---|---|---|
| `Opaque` | generic key-value secrets | app config credentials, API keys |
| `kubernetes.io/service-account-token` | legacy long-lived SA token | old integrations that cannot use TokenRequest |
| `kubernetes.io/dockercfg` | legacy Docker auth file | older Docker credential format |
| `kubernetes.io/dockerconfigjson` | Docker config JSON auth | private image registry pulls |
| `kubernetes.io/basic-auth` | username/password | basic auth for internal service |
| `kubernetes.io/ssh-auth` | SSH private key | git deploy key, SSH automation |
| `kubernetes.io/tls` | cert + private key | ingress TLS termination |
| `bootstrap.kubernetes.io/token` | kubeadm bootstrap token | node join/bootstrap flows |

Notes:
- If type is omitted, Kubernetes treats it as `Opaque`.
- You can define custom type names (recommended style: `your-domain/name`).

---

## 5) Creating Secrets (Three Common Patterns)

## A) From YAML (`stringData` easiest to read)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
type: Opaque
stringData:
  DB_USER: app_user
  DB_PASSWORD: super-secret-password
```

Why `stringData`:
- write plain text in manifest
- API server converts to `data` (base64) internally

## B) From kubectl

```bash
kubectl create secret generic app-credentials \
  --from-literal=DB_USER=app_user \
  --from-literal=DB_PASSWORD=super-secret-password
```

## C) Docker registry pull secret

```bash
kubectl create secret docker-registry regcred \
  --docker-server=my-registry.example.com \
  --docker-username=myuser \
  --docker-password='mypassword' \
  --docker-email='me@example.com'
```

---

## 6) Using Secrets in Pods

## A) As environment variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-secret-demo
spec:
  containers:
    - name: app
      image: nginx:alpine
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-credentials
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-credentials
              key: DB_PASSWORD
```

Best for:
- apps that read config from env vars

Tradeoff:
- process dumps/logs may expose env values if app is careless

## B) As files via Secret volume (preferred for certs/keys)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-secret-demo
spec:
  containers:
    - name: app
      image: nginx:alpine
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: app-credentials
```

Best for:
- TLS certs, private keys, rotating credentials

## C) For private image pull

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-demo
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: my-registry.example.com/team/app:v1
```

---

## 7) Secret Update Behavior (Very Important)

- Mounted Secret volumes update automatically with eventual consistency.
- Delay depends on kubelet sync + cache/watch propagation.
- If mounted using `subPath`, updates are **not** propagated automatically.
- Env var based Secret values are read at container start; usually require restart/rollout to pick updates.

Production implication:
- For dynamic rotation, prefer file mount + app reload signal or sidecar reloader pattern.

---

## 8) Secret Security in Production

Minimum controls you should always enable:

1. **Encryption at rest** for Secrets in etcd.
2. **Least-privilege RBAC** (avoid broad `list`/`watch` on Secrets).
3. **Namespace isolation** to reduce cross-team access.
4. **Restrict who can create Pods** (Pod creators can often indirectly access namespace Secrets).
5. **Audit logging** for Secret access patterns.
6. **Avoid privileged containers** where possible (node-level privilege is high risk).
7. **Rotate credentials** regularly with short TTL.
8. **Use external secret managers** for high-sensitivity environments.

---

## 9) Immutable Secrets

You can mark a Secret immutable:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-fixed-secret
immutable: true
type: Opaque
stringData:
  API_TOKEN: fixed-token
```

Benefits:
- prevents accidental updates
- improves API server/kubelet performance in very large clusters

Limitation:
- cannot edit later; must delete and recreate (then restart/redeploy consumers as needed)

---

## 10) Real-Time Production Scenarios

## Scenario 1: Database credential rotation
- App reads DB password from Secret mounted as file.
- DBA rotates password in DB and updates Secret.
- App watches file change (or receives reload signal), reconnects without full outage.

## Scenario 2: Ingress TLS renewal
- Cert manager (or external pipeline) updates `kubernetes.io/tls` Secret.
- Ingress controller reloads cert and serves renewed TLS.

## Scenario 3: Private registry in enterprise
- Workloads pull from private registry using `imagePullSecrets`.
- ServiceAccount is configured with default pull secret so all team pods inherit it.

## Scenario 4: Multi-tenant namespace safety
- Team A and Team B in separate namespaces.
- RBAC denies cross-namespace Secret access.
- Even if one app is compromised, credentials from other namespace are not directly exposed.

## Scenario 5: External secret store integration
- Vault / cloud secret manager stores source of truth.
- Operator syncs short-lived credentials to Kubernetes Secrets.
- Apps consume local Secrets; operator handles rotation lifecycle.

---

## 11) Common Pitfalls and Anti-Patterns

- Storing unencrypted secrets in Git.
- Assuming base64 means secure encryption.
- Giving developers or apps broad `list/watch secrets`.
- Using long-lived service account token Secrets when short-lived tokens are available.
- Logging secret values from application startup scripts.
- Mounting secrets to all containers in a Pod when only one needs them.
- Using `subPath` and expecting automatic secret refresh.

---

## 12) Operational Commands You Will Use Often

Create:

```bash
kubectl apply -f secret.yaml
```

Inspect metadata safely:

```bash
kubectl get secret app-credentials -o yaml
```

Read one key (careful in shared terminals):

```bash
kubectl get secret app-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

List secret type and key count:

```bash
kubectl get secrets
```

---

## 13) Design Checklist for Production Clusters

Before shipping, verify:
- [ ] Secrets encrypted at rest in etcd
- [ ] strict RBAC for `get/list/watch` on Secrets
- [ ] no plaintext secret values in Git or CI logs
- [ ] rotation policy defined (what, who, frequency, rollback)
- [ ] workload restart/reload strategy for rotated values
- [ ] no unnecessary secret mounts to unrelated containers
- [ ] audit trails enabled for sensitive API access
- [ ] preference for external secret manager where required by compliance

---

## 14) Quick Comparison: Secret vs ConfigMap

- Use **Secret** for sensitive data.
- Use **ConfigMap** for non-sensitive config.
- Both can be mounted as files or env vars.
- Secret needs stronger access and lifecycle controls.

---

## 15) Final Takeaway

Kubernetes Secrets are not just key-value objects. In production, they are part of your security boundary, incident response posture, and platform reliability model. Use the correct Secret type, consume secrets safely, enforce least privilege, enable encryption at rest, and implement rotation plus auditing as standard platform practice.

Reference:
- Kubernetes official docs on Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
