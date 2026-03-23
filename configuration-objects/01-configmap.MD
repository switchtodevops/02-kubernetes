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

## Example 1: Create ConfigMap from multiple files

### 1) Create folder and files

```bash
mkdir -p config
```

Create `config/index.html`:

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

Create `config/nginx.conf`:

```nginx
server {
    listen 8080;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

### 2) Create ConfigMap from the folder

```bash
kubectl create configmap my-config --from-file=./config
```

Each file becomes a key:
- `index.html` -> value is full HTML content
- `nginx.conf` -> value is full nginx config content

### 3) Verify using describe

```bash
kubectl describe cm my-config
```

Sample output:

```text
Name:         my-config
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

nginx.conf:
----
server {
    listen 8080;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}

BinaryData
====

Events:  <none>
```

### 4) Pod manifest to use this ConfigMap (Example 1)

Use this single `pod.yaml` to run nginx with config + html from `my-config`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom-8080
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf   # Example 1 key name from ConfigMap
        - name: html-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html   # Example 1 key name from ConfigMap
  volumes:
    - name: nginx-conf
      configMap:
        name: my-config
        items:
          - key: nginx.conf
            path: nginx.conf
    - name: html-volume
      configMap:
        name: my-config
        items:
          - key: index.html
            path: index.html
```

---

## Example 2: Create ConfigMap with custom key names

Use this when you want ConfigMap keys to be different from original filenames.

### 1) Generic pattern

```bash
# Create a config map with explicit keys (instead of file basenames)
kubectl create configmap my-config \
  --from-file=key1=/path/to/bar/file1.txt \
  --from-file=key2=/path/to/bar/file2.txt
```

### 2) Real command with your files

```bash
kubectl create configmap my-config \
  --from-file=indexhtml=./config/index.html \
  --from-file=nginxconfig=./config/nginx.conf
```

Note: using `./config/nging.conf` will fail if that filename does not exist. Use `./config/nginx.conf`.

### 3) Verify output

```bash
kubectl describe cm my-config
```

Sample output:

```text
Name:         my-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
indexhtml:
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

nginxconfig:
----
server {
    listen 8080;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}

BinaryData
====

Events:  <none>
```

### 4) Pod manifest to use this ConfigMap (Example 2)

Use this exact `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom-8080
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
        - name: html-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
  volumes:
    - name: nginx-conf
      configMap:
        name: app-files
        items:
          - key: nginx.conf
            path: nginx.conf
    - name: html-volume
      configMap:
        name: app-files
        items:
          - key: index.html
            path: index.html
```
