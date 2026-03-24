# Kubernetes ConfigMap — Developer Guide

> A ConfigMap is an API object used to store **non-confidential** data in **key-value pairs**. Pods can consume ConfigMaps as **environment variables**, **command-line arguments**, or as **configuration files in a volume**.

A **ConfigMap** is **namespaced** configuration you inject without baking it into the image 

## Summary highlights (quick reference)

- **Use ConfigMap for non-sensitive data only**; use **Secret** for credentials/tokens/certs.
- **ConfigMap is namespaced** and consumed by Pods in the **same namespace**.
- Store values in **`data`** (UTF-8 strings) or **`binaryData`** (base64 binary); keys must not overlap.
- Main consumption modes: **env vars**, **command/args**, **volume files**, or **Kubernetes API** in app code.
- **Env values do not auto-refresh** after ConfigMap changes; restart Pods (or rollout restart) to pick changes.
- **Volume-mounted ConfigMap updates are eventual**; **`subPath` does not auto-update**.
- Prefer **declarative YAML + GitOps**; for imperative creation use `--from-literal`, `--from-file`, `--from-env-file`.
- ConfigMap total size limit is about **1 MiB**; avoid huge blobs.
- Use **`immutable: true`** for safer, stable config at scale; replace with new version on change.
- For production, use **versioned names** (e.g., `app-config-v2`) and a clear rollout strategy.

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

**Short explanation:**

- **`data`**: use for normal text config (URLs, flags, properties, JSON/YAML text).
- **`binaryData`**: use when value is raw bytes (for example, a small binary blob). Value must be base64 in YAML.
- If your value is regular text, prefer **`data`** for readability.

**Example:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  app.properties: |
    server.port=8080
binaryData:
  icon.bin: "AAEC"   # example base64 bytes
```

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

## Example 1: ConfigMap from a folder, single file, and nginx Pod

### 1) Create from a directory

Every file under the folder becomes one ConfigMap **key** (key name = file **basename**):

```bash
kubectl create configmap my-config --from-file=path/to/bar
```

### 2) Create from a single file (`cm1`)

Save the HTML below as `index.html` in your working directory, then:

```bash
kubectl create configmap cm1 --from-file=index.html
```

The key in the ConfigMap will be **`index.html`** (same as the filename).

### 3) `index.html` content

```bash
cat index.html
```

```html
<!DOCTYPE html>
<html>
<head><title>Custom Page</title></head>
<body style="background-color: lightblue;">
<h1>Hello from ConfigMap!</h1>
<p>Application version v1.0.0</p>
<p>ConfigMap example with multiple files.</p>
</body>
</html>
```

### 4) Describe the ConfigMap (`cm1`)

```bash
kubectl describe cm cm1
```

Sample output:

```text
Name:         cm1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
index.html:
----
<!DOCTYPE html>
<html>
<head><title>Custom Page</title></head>
<body style="background-color: lightblue;">
<h1>Hello from ConfigMap!</h1>
<p>Application version v1.0.0</p>
<p>ConfigMap example with multiple files.</p>
</body>
</html>

BinaryData
====

Events:  <none>
```

### 5) Pod manifest (`test-pod.yaml`)

```bash
cat test-pod.yaml
```

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: testpod
spec:
  containers:
    - name: webapp
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html

  volumes:
    - name: html-volume
      configMap:
        name: cm1
        items:
          - key: index.html
            path: index.html
```

Apply and test (optional):

```bash
kubectl apply -f test-pod.yaml
kubectl port-forward pod/testpod 8080:80
```

Open `http://localhost:8080` — you should see the custom page from the ConfigMap.

### 6) What `volumeMounts` and `volumes` are doing

**`volumeMounts`** (on the container):

| Field | Meaning |
|-------|---------|
| **`name: html-volume`** | Links to the volume defined below with the **same** `name` under `spec.volumes`. |
| **`mountPath: /usr/share/nginx/html/index.html`** | The path **inside the container** where this mount appears. Here it is a **single file path**, not a directory, because we use **`subPath`**. |
| **`subPath: index.html`** | Mount **only one file** from the volume instead of the whole volume root. The name **`index.html`** must match the **`path`** in `volumes[].configMap.items[]` (the filename Kubernetes projects into that volume). |

**`volumes`** (on the Pod):

| Field | Meaning |
|-------|---------|
| **`name: html-volume`** | Volume ID referenced by `volumeMounts.name`. |
| **`configMap.name: cm1`** | Use the ConfigMap named **`cm1`** (must exist in the **same namespace** as the Pod). |
| **`items`** | Optional. Pick **which keys** from the ConfigMap to expose and **what filename** they get inside the volume. |
| **`key: index.html`** | The **key** in `cm1.data` (must match `kubectl describe cm cm1`). |
| **`path: index.html`** | The **filename** inside the volume mount point’s “directory”. With **`subPath: index.html`**, the container sees that single file at **`mountPath`**. |

**Rule of thumb:** `items[].key` = ConfigMap key; `items[].path` = internal volume filename; **`subPath`** must equal **`path`** when you mount a single file to a single file path like this.

---

