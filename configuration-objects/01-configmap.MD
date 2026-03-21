# Kubernetes ConfigMap — Developer Guide

A **ConfigMap** stores **non-sensitive** configuration (key/value pairs, or files) that Pods can consume as **environment variables** or **mounted files**. **Crisp, complete, and production-safe.**

---

## What is a ConfigMap?

- **Object** in a namespace (`ConfigMap` is namespaced).
- Holds **plain text** data — **not** for passwords, API keys, or TLS private keys (use **Secret**).
- Decouples **config** from **container image** so the same image runs in dev/stage/prod with different settings.

**Mental model:** “**Inject config** into Pods without baking it into the image.”

---

## ConfigMap vs Secret

| Topic | ConfigMap | Secret |
|--------|-----------|--------|
| **Sensitivity** | Non-sensitive config | Sensitive data (base64-encoded at rest; still not encryption by default) |
| **Use for** | URLs, feature flags, `config.properties`, nginx templates | Passwords, tokens, certs |

> **Never put secrets in a ConfigMap** — they are visible in etcd, API, logs, and `kubectl describe` flows.

---

## Ways to use ConfigMap in a Pod

| Method | Behavior |
|--------|----------|
| **Single env var** | `valueFrom.configMapKeyRef` — one key → one env var |
| **Bulk env** | `envFrom` with `configMapRef` — all keys → env vars (names must be valid env names) |
| **Volume mount** | Mount as **directory** of files (one file per key) or **subPath** for one file |
| **Command args** | Reference `$(ENV_NAME)` if you set env from ConfigMap first |

**Update behavior (important):**

- **Env vars** set at Pod start → **no live reload** from ConfigMap; **restart Pod** to pick up changes.
- **Mounted volume** (without `subPath`) → kubelet can **sync** updated files **periodically** (default sync interval; not instant). **`subPath`** mounts **do not** auto-update on ConfigMap change.
- **Immutable ConfigMap** (`immutable: true`) → no updates; replace with new version + rollout.

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

- **Cannot** mutate `data` in place — **create new ConfigMap** (e.g. `app-config-v2`), update Deployment to reference it, then roll out.
- **Reduces** accidental drift and **etcd** watch churn.

---

## Limits and gotchas

| Topic | Detail |
|--------|--------|
| **Size** | Total ConfigMap size **≤ 1 MiB** (etcd limit); split large files or use volumes / external config store for huge data |
| **Keys** | Valid key names; env-from keys must be valid **ENV** variable names |
| **Namespace** | ConfigMap must be **same namespace** as Pod (unless using cross-namespace patterns — not native; avoid hacks) |
| **Required vs optional** | `optional: true` on `configMapKeyRef` avoids Pod failure if missing (use carefully) |

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

**ConfigMap** = **namespace-scoped, non-sensitive** config for Pods via **env** or **files**. **Create** with `kubectl create` or **YAML**; **consume** with `configMapKeyRef`, `envFrom`, or **volumes**. Treat updates as **deployment events** (restart or volume sync rules). **Never** store secrets here — use **Secret** or a **secret manager**.
