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

## 12) Complete Kubernetes Secret Command Reference

Use namespace flag whenever needed:

```bash
# one-time for command
kubectl -n <namespace> <command>

# or set default namespace in current context
kubectl config set-context --current --namespace=<namespace>
```

### A) Discover and inspect Secret resources

```bash
# list all Secret objects in current namespace
kubectl get secrets

# list in all namespaces
kubectl get secrets -A

# show wide table with labels
kubectl get secrets --show-labels

# describe one Secret (metadata + events style output)
kubectl describe secret <secret-name>

# full YAML
kubectl get secret <secret-name> -o yaml

# full JSON
kubectl get secret <secret-name> -o json

# watch Secrets in real time
kubectl get secrets -w

# API info for Secret kind
kubectl explain secret
kubectl explain secret.data
kubectl explain secret.type
kubectl explain secret.stringData
kubectl api-resources | rg -i secret
```

### B) Create Secrets with kubectl (all common subcommands)

```bash
# generic (Opaque) from literals
kubectl create secret generic app-credentials \
  --from-literal=DB_USER=app_user \
  --from-literal=DB_PASSWORD='super-secret-password'

# generic from file (single key from file name)
kubectl create secret generic app-config \
  --from-file=./config.env

# generic from directory (each file becomes one key)
kubectl create secret generic tls-bundle \
  --from-file=./secret-files/

# generic with custom key name from file
kubectl create secret generic app-keys \
  --from-file=private.key=./keys/app-private.key

# docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=my-registry.example.com \
  --docker-username=myuser \
  --docker-password='mypassword' \
  --docker-email='me@example.com'

# TLS secret from cert/key
kubectl create secret tls ingress-tls \
  --cert=./tls.crt \
  --key=./tls.key

# dry-run output as YAML (great for GitOps)
kubectl create secret generic app-credentials \
  --from-literal=DB_USER=app_user \
  --from-literal=DB_PASSWORD='super-secret-password' \
  --dry-run=client -o yaml > secret.yaml

# create from manifest file
kubectl apply -f secret.yaml
```

### C) Create built-in typed Secrets via YAML

```bash
# basic auth secret type
kubectl apply -f basicauth-secret.yaml

# ssh auth secret type
kubectl apply -f ssh-secret.yaml

# service-account-token (legacy; usually avoid)
kubectl apply -f service-account-token-secret.yaml

# bootstrap token (kubeadm flows)
kubectl apply -f bootstrap-token-secret.yaml
```

### D) Decode and read Secret values safely

```bash
# get all keys (base64 encoded) in a Secret
kubectl get secret <secret-name> -o jsonpath='{.data}'

# decode one key
kubectl get secret <secret-name> \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo

# decode all key-values with go-template
kubectl get secret <secret-name> -o go-template='{{range $k,$v := .data}}{{printf "%s=" $k}}{{printf "%s" $v | base64decode}}{{"\n"}}{{end}}'

# extract secret type only
kubectl get secret <secret-name> -o jsonpath='{.type}'; echo
```

> Security note: decoded values can end up in terminal history/logging. Prefer secure shells and avoid shared terminals/screens.

### E) Update or rotate Secret data

```bash
# edit Secret in-place (opens editor)
kubectl edit secret <secret-name>

# replace from updated manifest
kubectl apply -f secret.yaml

# patch Secret data field (value must be base64)
kubectl patch secret <secret-name> \
  -p '{"data":{"DB_PASSWORD":"bmV3LXBhc3M="}}'

# safer rotation workflow: recreate manifest and apply
kubectl create secret generic app-credentials \
  --from-literal=DB_USER=app_user \
  --from-literal=DB_PASSWORD='new-password' \
  --dry-run=client -o yaml | kubectl apply -f -
```

After rotating a secret, restart workload if your app only reads env vars at startup:

```bash
kubectl rollout restart deployment/<deployment-name>
kubectl rollout status deployment/<deployment-name>
```

### F) Work with immutable Secrets

```bash
# check immutable flag
kubectl get secret <secret-name> -o jsonpath='{.immutable}'; echo

# set immutable=true (via patch)
kubectl patch secret <secret-name> -p '{"immutable":true}'
```

If immutable Secret must change:

```bash
# delete and recreate with new values
kubectl delete secret <secret-name>
kubectl apply -f secret.yaml
```

### G) Use Secrets with ServiceAccounts and image pulls

```bash
# create docker pull secret
kubectl create secret docker-registry regcred \
  --docker-server=my-registry.example.com \
  --docker-username=myuser \
  --docker-password='mypassword' \
  --docker-email='me@example.com'

# attach imagePullSecret to default ServiceAccount
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets":[{"name":"regcred"}]}'

# verify ServiceAccount config
kubectl get sa default -o yaml
```

### H) RBAC and access checks for Secrets

```bash
# can current identity read secrets in namespace?
kubectl auth can-i get secrets

# can current identity list all secrets?
kubectl auth can-i list secrets

# can a specific SA read a secret?
kubectl auth can-i get secret/<secret-name> \
  --as=system:serviceaccount:<namespace>:<serviceaccount> \
  -n <namespace>
```

### I) Troubleshooting Secret-related Pod issues

```bash
# inspect pod events/errors (missing secret, missing key, pull auth issues)
kubectl describe pod <pod-name>

# list events sorted by time
kubectl get events --sort-by=.metadata.creationTimestamp

# check pod manifest as seen by API server
kubectl get pod <pod-name> -o yaml

# view env vars inside container (be careful: sensitive)
kubectl exec -it <pod-name> -- env

# check mounted secret files
kubectl exec -it <pod-name> -- ls -la /etc/secret
kubectl exec -it <pod-name> -- sh -c 'cat /etc/secret/<key-file>'
```

### J) Delete and cleanup

```bash
# delete one secret
kubectl delete secret <secret-name>

# delete from manifest
kubectl delete -f secret.yaml

# delete all secrets in namespace (dangerous)
kubectl delete secrets --all
```

### K) Useful one-liners for audits

```bash
# list all secret names + type in all namespaces
kubectl get secrets -A -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,TYPE:.type'

# count number of secrets per namespace
kubectl get secrets -A --no-headers | awk '{print $1}' | sort | uniq -c

# find pods referencing imagePullSecrets
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.spec.imagePullSecrets[*].name}{"\n"}{end}'
```

### L) Kustomize and Secret generation commands

```bash
# create kustomization with generated secret
cat <<'EOF' > kustomization.yaml
secretGenerator:
  - name: app-credentials
    literals:
      - DB_USER=app_user
      - DB_PASSWORD=super-secret-password
generatorOptions:
  disableNameSuffixHash: false
EOF

# build and apply
kubectl kustomize .
kubectl apply -k .
```

### M) Quick command map by Secret type

```bash
# Opaque
kubectl create secret generic <name> --from-literal=key=value

# dockerconfigjson
kubectl create secret docker-registry <name> --docker-server=... --docker-username=... --docker-password=...

# tls
kubectl create secret tls <name> --cert=tls.crt --key=tls.key
```

For `kubernetes.io/basic-auth`, `kubernetes.io/ssh-auth`, `kubernetes.io/service-account-token`, and `bootstrap.kubernetes.io/token`, create via YAML and apply with:

```bash
kubectl apply -f <secret-manifest>.yaml
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
