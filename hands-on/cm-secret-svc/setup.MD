# Hands-on: NGINX HTTPS with TLS Secret + ConfigMap + Service

Use a **self-signed TLS certificate**, store it in a **Secret**, put **nginx.conf** in a **ConfigMap**, run a **Deployment**, and expose it with a **Service** (NodePort / ClusterIP / LoadBalancer).

---

## Step 1: Generate a self-signed certificate

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=nginx.local/O=nginx"
```

You get:

| File | Purpose |
|------|---------|
| `tls.crt` | Certificate |
| `tls.key` | Private key |

---

## Step 2: Create a Kubernetes TLS Secret

```bash
kubectl create secret tls nginx-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

Verify:

```bash
kubectl get secret nginx-tls-secret
```

---

## Step 3: Create NGINX config with SSL

Create `nginx.conf`:

```nginx
events {}

http {
  server {
    listen 443 ssl;

    ssl_certificate /etc/nginx/ssl/tls.crt;
    ssl_certificate_key /etc/nginx/ssl/tls.key;

    location / {
      return 200 "Hello from HTTPS NGINX!\n";
    }
  }
}
```

---

## Step 4: Store config in a ConfigMap

```bash
kubectl create configmap nginx-config \
  --from-file=nginx.conf
```

---

## Step 5: Create Deployment

**Mounts:**

| Source | Mount |
|--------|--------|
| TLS Secret | `/etc/nginx/ssl` |
| ConfigMap (`nginx.conf`) | `/etc/nginx/nginx.conf` (via `subPath`) |

Save as `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ssl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ssl
  template:
    metadata:
      labels:
        app: nginx-ssl
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 443
          volumeMounts:
            - name: tls-secret
              mountPath: /etc/nginx/ssl
              readOnly: true
            - name: nginx-config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
      volumes:
        - name: tls-secret
          secret:
            secretName: nginx-tls-secret
        - name: nginx-config
          configMap:
            name: nginx-config
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

---

## Step 6: Create Service (port 443)

Save as `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ssl-service
spec:
  selector:
    app: nginx-ssl
  ports:
    - protocol: TCP
      port: 443
      targetPort: 443
  type: NodePort   # or ClusterIP / LoadBalancer
```

Apply:

```bash
kubectl apply -f service.yaml
```

---

## Step 7: Test

### Inspect Service

```bash
kubectl get svc nginx-ssl-service
```

Example output (your ClusterIP / NodePort will differ):

```text
CLUSTER-IP    PORT(S)        NODE PORT
10.96.x.x     443/TCP        31586
```

### Check NGINX inside the Pod

```bash
kubectl get pods
kubectl exec -it <pod-name> -- /bin/sh
curl -vk https://localhost:443
```

- **`-k` / `--insecure`**: ignore self-signed cert verification in `curl` (optional for local testing).
- **`https://`**: required for TLS.

```bash
kubectl logs <pod-name>
```

---

## How to access the Service

After `kubectl get svc nginx-ssl-service`, note **ClusterIP**, **port**, and **NodePort** (if `type: NodePort`).

Example values (replace with yours):

| Field | Example |
|-------|---------|
| ClusterIP | `10.96.234.209` |
| Port | `443` |
| NodePort | `31586` |

### Option A: `localhost` (e.g. Docker Desktop)

Docker Desktop often maps NodePorts to `localhost`:

```bash
curl -vk https://localhost:31586
```

Or open in a browser: `https://localhost:31586`

Expect a **certificate warning** (self-signed) — that is normal.

### If that does not work

1. **Confirm Pod is running**

   ```bash
   kubectl get pods -o wide
   ```

2. **Check Endpoints** (selector must match Pod labels)

   ```bash
   kubectl get endpoints nginx-ssl-service
   ```

   You should see a Pod IP and port **443**. If **empty**, fix **Service `selector`** vs **Pod labels**.

3. **Port-forward** (bypasses NodePort)

   ```bash
   kubectl port-forward svc/nginx-ssl-service 8443:443
   ```

   Then:

   ```bash
   curl -vk https://localhost:8443
   ```

4. **Test from inside the Pod**

   ```bash
   kubectl exec -it <pod-name> -- curl -vk https://localhost:443
   ```

---

## Quick mental model

```text
Browser / curl → localhost:<NodePort>
       ↓
NodePort (<NodePort> → 443 on Pod)
       ↓
Service (port 443 → targetPort 443)
       ↓
Pod (nginx listening on 443, TLS)
```

**Often useful for a quick check:**

```bash
curl -vk https://localhost:<NodePort>
```

(Use the NodePort value from `kubectl get svc`.)
