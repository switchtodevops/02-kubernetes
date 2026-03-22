# Kubernetes ConfigMap — Developer Guide

> A ConfigMap is an API object used to store **non-confidential** data in **key-value pairs**. Pods can consume ConfigMaps as **environment variables**, **command-line arguments**, or as **configuration files in a volume**.

A **ConfigMap** is **namespaced** configuration you inject without baking it into the image — **crisp, complete, and production-safe** (expanded below).

---

## What is a ConfigMap?

- **Object** in a namespace (`ConfigMap` is namespaced).
- Holds **plain text** data — **not** for passwords, API keys, or TLS private keys (use **Secret**).
- Decouples **config** from **container image** so the same image runs in dev/stage/prod with different settings.

**Mental model:** “**Inject config** into Pods without baking it into the image.”

Official reference: [ConfigMaps | Kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/).

---

## ConfigMap object (API shape)

Unlike many resources that use a **`spec`**, a ConfigMap stores data in:

| Field | Content |
|-------|---------|
| **`data`** | UTF-8 **strings** (key → string value). |
| **`binaryData`** | **Binary** values stored as **base64-encoded** strings in the manifest. |

- **`data` and `binaryData` keys must not overlap** (same key name in both is invalid).
- Each key name may use **alphanumeric characters**, **`-`**, **`_`**, or **`.`**.
- The object **`metadata.name`** must be a **valid DNS subdomain** name.

---

## ConfigMap vs Secret

| Topic | ConfigMap | Secret |
|--------|-----------|--------|
| **Sensitivity** | Non-sensitive config | Sensitive data (base64-encoded at rest; still not encryption by default) |
| **Use for** | URLs, feature flags, `config.properties`, nginx templates | Passwords, tokens, certs |

> **Never put secrets in a ConfigMap** — they are visible in etcd, API, logs, and `kubectl describe` flows.

---

## Ways to use ConfigMap in a Pod

Per the [official documentation](https://kubernetes.io/docs/concepts/configuration/configmap/), there are **four** patterns:

| # | Method | How |
|---|--------|-----|
| 1 | **Container `command` / `args`** | Inject values (often via **env** substitution like `$(VAR)` after loading from ConfigMap). |
| 2 | **Environment variables** | `configMapKeyRef` or `envFrom.configMapRef`. |
| 3 | **Read-only volume** | Mount keys as files under a `mountPath`. |
| 4 | **Kubernetes API from app code** | Watch/read ConfigMap via API (can react to updates; **cross-namespace** possible from code — not from Pod fields alone). |

| Pattern (detail) | Behavior |
|------------------|----------|
| **Single env var** | `valueFrom.configMapKeyRef` — one key → one env var (env var **name** can differ from ConfigMap **key**). |
| **Bulk env** | `envFrom` + `configMapRef` — all keys → env vars. |
| **Volume mount** | Each key → file under `mountPath`, or use **`items`** to pick keys and rename paths (see below). |
| **`subPath`** | Single file path; **no** auto-update when ConfigMap changes. |

**Caution — static Pods:** A **static Pod** `spec` **cannot** reference a ConfigMap (or other API objects). Use normal workload Pods / controllers.

**Update behavior (important):**

- **Environment variables** — **not** updated when ConfigMap changes; **restart Pod** to pick up new values.
- **Mounted volume** (without `subPath`) — projected files are **eventually** updated when the ConfigMap changes: kubelet refreshes on its **sync interval**, plus **cache propagation** delay (watch vs TTL vs direct API — depends on kubelet config). **Not instant.**
- **`subPath`** volume mounts — **do not** receive ConfigMap updates.
- **Immutable ConfigMap** (`immutable: true`) — no in-place data changes; replace object + rollout.

**`envFrom` / env keys:** Environment variable names must follow **Kubernetes rules**. If a ConfigMap **key** is **not** a valid env name, that key may be **skipped** (not injected) — the Pod can still **start**, so missing env vars can look like “silent” bugs. Prefer keys like `LOG_LEVEL` or map explicitly with `configMapKeyRef`.

---

## Creating a ConfigMap

### Imperative: `kubectl create configmap` options

| Flag | What it does |
|------|----------------|
| **`--from-literal=key=value`** | Add **one** key/value pair. Repeat the flag for **multiple** keys. Values can be quoted if they contain spaces. |
| **`--from-env-file=path`** | Read a **`.env`-style file**: each non-empty, non-comment line is `KEY=VALUE`. All keys become ConfigMap entries in **one** shot. |
| **`--from-file=...`** | File contents become the value; filename (or custom key) is the key — see examples below. |

#### `--from-literal`

Use when you have **few** keys or you’re scripting one-off values.

```bash
kubectl create configmap app-config -n demo \
  --from-literal=LOG_LEVEL=info \
  --from-literal=FEATURE_X=true

# Value with spaces (quote carefully in your shell)
kubectl create configmap app-config -n demo \
  --from-literal='APP_MSG=hello world'
```

- You can pass **`--from-literal` many times** — each pair is one key in the ConfigMap.
- Keys must be valid for ConfigMap **`data`** keys; values are plain strings.

#### `--from-env-file`

Use when you already maintain config as **`KEY=VALUE`** lines (same idea as **Docker env files** / **dotenv**).

**Example file `app.env`:**

```text
LOG_LEVEL=info
FEATURE_X=true
# lines starting with # are ignored (comments)
EMPTY_OK=
```

```bash
kubectl create configmap app-config -n demo --from-env-file=app.env
```

- **One line = one variable.** Format: `KEY=value` (no spaces around `=` in the key name part; value can contain spaces if quoted per shell rules when needed).
- **Comments:** lines beginning with `#` are ignored.
- You can combine **`--from-env-file`** with **`--from-literal`** / **`--from-file`** in the same `kubectl create configmap` command (same as mixing other sources).

#### Other imperative patterns

```bash
# From file (filename becomes key, content = file body)
kubectl create configmap nginx-html -n demo --from-file=index.html

# From directory (each file = one key)
kubectl create configmap app-files -n demo --from-file=./config/

# Explicit key name for a file path
kubectl create configmap app-config -n demo --from-file=app.properties=./config/app.properties

# Combine sources: env file + extra literal + file
kubectl create configmap app-config -n demo \
  --from-env-file=app.env \
  --from-literal=BONUS_KEY=extra \
  --from-file=nginx.conf=./config/nginx.conf
```

**Replace existing:**

```bash
kubectl create configmap app-config -n demo --from-literal=k=v --dry-run=client -o yaml | kubectl apply -f -
# or delete + create, or use apply with full manifest
```

### Declarative (GitOps / production)

**configmap.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
  labels:
    app: myapp
    config-version: v1
data:
  LOG_LEVEL: "info"
  FEATURE_X: "true"
  # Multi-line file content as a single key
  app.properties: |
    server.port=8080
    spring.profiles.active=prod
binaryData: {}   # optional: base64 for raw bytes (rare; prefer Secret for sensitive binary)
```

```bash
kubectl apply -f configmap.yaml
```

---

## Example: Pod using ConfigMap

### 1) Environment variables (one key + bulk)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
  namespace: demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["/bin/sh", "-c", "echo LOG_LEVEL=$LOG_LEVEL && sleep 3600"]
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
              optional: false
      envFrom:
        - configMapRef:
            name: app-config
            optional: false
```

### 2) Volume mount (whole ConfigMap as directory)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
  namespace: demo
spec:
  containers:
    - name: app
      image: nginx:1.25-alpine
      volumeMounts:
        - name: cfg
          mountPath: /etc/nginx/conf.d
          readOnly: true
  volumes:
    - name: cfg
      configMap:
        name: app-config
        defaultMode: 0644
        optional: false
```

**Note:** Each **key** in `data` becomes a **file under `mountPath`** (e.g. `LOG_LEVEL` → file named `LOG_LEVEL`).

**Selective keys with `items`** (only some keys, custom filenames):

```yaml
volumes:
  - name: cfg
    configMap:
      name: app-config
      items:
        - key: app.properties
          path: app.properties
        - key: "game.properties"
          path: game.properties
```

If omit **`items`**, **every** key in the ConfigMap becomes a file named like the key.

### 3) Single file with `subPath` (fixed filename)

```yaml
# ... inside containerSpec ...
volumeMounts:
  - name: cfg
    mountPath: /app/config/app.properties
    subPath: app.properties
    readOnly: true
volumes:
  - name: cfg
    configMap:
      name: app-config
```

**Trade-off:** `subPath` = **no automatic** file sync when ConfigMap updates.

---

## Deployment example (industry pattern)

Use the same `env` / `volumeMounts` under **`spec.template.spec`** in a **Deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:1.2.3
          envFrom:
            - configMapRef:
                name: app-config
          volumeMounts:
            - name: config
              mountPath: /config
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: app-config
```

**After ConfigMap change:** roll Pods (`kubectl rollout restart deployment/myapp -n demo`) if you rely on env or need guaranteed file refresh.

---

## Immutable ConfigMaps

Stable since **Kubernetes v1.21**. Set `immutable: true` on the ConfigMap.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
immutable: true
data:
  LOG_LEVEL: info
```

- **Cannot** flip back to mutable or **edit** `data` / `binaryData` in place — **delete and recreate** (or create a new ConfigMap name/version).
- **Why use it:** avoids **accidental** updates that break apps; **reduces load** on the API server on clusters with **very large** ConfigMap usage (many mounts / watches).
- After **delete/recreate**, **Pods** with old mounts may need **recreation** so they point at the new object.

---

## References

- [ConfigMaps (concepts)](https://kubernetes.io/docs/concepts/configuration/configmap/)
- Task guide: [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

---

## Limits and gotchas

| Topic | Detail |
|--------|--------|
| **Size** | Total ConfigMap size **≤ 1 MiB** — not for large blobs; use volumes, DB, or external config for bigger data ([docs](https://kubernetes.io/docs/concepts/configuration/configmap/)). |
| **Keys** | Only **`a-z` `A-Z` `0-9` `-` `_` `.`** in key names; **`data`** vs **`binaryData`** keys must not overlap. |
| **Env from keys** | `envFrom` skips keys that are **illegal** as environment variable names; Pod may still **start** — verify with `kubectl exec` / logs. |
| **Namespace** | ConfigMap and consuming Pod must be in the **same namespace** (Pod fields cannot reference another namespace’s ConfigMap). |
| **Required vs optional** | `optional: true` on `configMapKeyRef` avoids Pod failure if missing (use carefully). |
| **Static Pods** | **Cannot** reference ConfigMap in the static Pod manifest. |

---

## Industry best practices

1. **Non-secrets only** — passwords/tokens → **Secret**; TLS → **Secret** or CSI.  
2. **Versioned names** — `app-config-v3` or labels for traceability; GitOps-friendly.  
3. **Immutable + rollout** when you need **auditable** change control.  
4. **Separate config from code** — same image, different ConfigMaps per env.  
5. **Least privilege** — mount **read-only**; don’t mount entire `/config` writable if you only need one file.  
6. **Document keys** — treat `data` keys as a **contract**; breaking renames break consumers.  
7. **Restart strategy** — document whether team uses **rollout restart** after CM changes; automate in CI if needed.  
8. **Avoid giant blobs** — use file mounts or external systems (Parameter Store, Vault, etc.) for large/complex config.  
9. **Validation** — validate config in app startup; fail fast with clear errors.  
10. **Don’t rely on ConfigMap for secrets** — even “hidden” keys are not secret.

---

## Troubleshooting

```bash
kubectl get configmap -n <ns>
kubectl describe configmap <name> -n <ns>
kubectl get configmap <name> -n <ns> -o yaml
```

| Symptom | Check |
|---------|--------|
| **CreateContainerConfigError** | Wrong ConfigMap **name/key**; `optional: false` but missing |
| **Env empty** | Key typo; wrong namespace |
| **File not updated** | Using **`subPath`**; need **restart** or non-subPath mount |

---

## Quick commands

```bash
kubectl create configmap NAME -n NS --from-literal=k=v
kubectl create configmap NAME -n NS --from-env-file=path/to/app.env
kubectl apply -f configmap.yaml
kubectl edit configmap NAME -n NS
kubectl delete configmap NAME -n NS
```

---

## Summary

**ConfigMap** = **namespace-scoped, non-sensitive** config (`data` / `binaryData` — no `spec`). **Create** with `kubectl create` or **YAML**; **consume** via **command/args**, **env**, **volume files**, or **in-app API** (advanced). **Env** does not update live; **mounted volumes** update **eventually** (not with **`subPath`**). **Immutable** ConfigMaps protect changes and scale. **Never** store secrets here — use **Secret** or a **secret manager**. See [kubernetes.io ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) for the full picture.
