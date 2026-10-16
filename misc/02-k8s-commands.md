# Kubernetes Commands for Daily Operations and Troubleshooting

This is a practical `kubectl` cheat sheet for real-world use.  
Examples use `-n <namespace>` where needed.

## Table of Contents

<p style="margin:14px 0 6px;font-size:12px;font-weight:600;letter-spacing:.04em;opacity:.75;">CLUSTER AND ACCESS</p>
<div style="display:grid;grid-template-columns:repeat(5,minmax(0,1fr));gap:8px;">
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#1-must-know-global-flags-use-everywhere">Global Flags</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#2-cluster-and-context">Cluster and Context</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#3-namespaces">Namespaces</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#14-rbac-and-service-accounts">RBAC</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#16-resource-usage-and-metrics">Resource Usage</a>
</div>

<p style="margin:14px 0 6px;font-size:12px;font-weight:600;letter-spacing:.04em;opacity:.75;">WORKLOADS</p>
<div style="display:grid;grid-template-columns:repeat(5,minmax(0,1fr));gap:8px;">
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#4-pods-most-frequent">Pods</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#5-deployments">Deployments</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#6-replicasets-rs">ReplicaSets</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#10-statefulsets-and-daemonsets">StatefulSets and DaemonSets</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#11-jobs-and-cronjobs">Jobs and CronJobs</a>
</div>

<p style="margin:14px 0 6px;font-size:12px;font-weight:600;letter-spacing:.04em;opacity:.75;">NETWORKING, CONFIG, AND STORAGE</p>
<div style="display:grid;grid-template-columns:repeat(5,minmax(0,1fr));gap:8px;">
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#7-services-and-endpoints">Services and Endpoints</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#12-ingress">Ingress</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#8-configmaps">ConfigMaps</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#9-secrets">Secrets</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#13-storage-pv-pvc-storageclass">Storage</a>
</div>

<p style="margin:14px 0 6px;font-size:12px;font-weight:600;letter-spacing:.04em;opacity:.75;">DAILY OPERATIONS</p>
<div style="display:grid;grid-template-columns:repeat(5,minmax(0,1fr));gap:8px;">
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#15-events-very-important-for-troubleshooting">Events</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#17-fast-troubleshooting-workflow">Troubleshooting</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#18-yamlmanifest-operations">YAML and Manifests</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#19-output-formatting-and-filtering-daily-use">Output Formatting</a>
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#20-optional-productivity-aliases">Aliases</a>
</div>

<p style="margin:14px 0 6px;font-size:12px;font-weight:600;letter-spacing:.04em;opacity:.75;">QUICK REFERENCE</p>
<div style="display:grid;grid-template-columns:repeat(5,minmax(0,1fr));gap:8px;">
<a style="display:flex;align-items:center;justify-content:center;min-height:48px;padding:8px 10px;border:1px solid rgba(128,128,128,.5);border-radius:8px;background:rgba(128,128,128,.08);text-align:center;text-decoration:none;line-height:1.35;" href="#quick-minimal-command-set-if-you-learn-only-15">Quick Minimal Command Set</a>
</div>

---

## 1) Must-Know Global Flags (Use Everywhere)

- `-n <namespace>`: target namespace
- `-A` or `--all-namespaces`: query all namespaces
- `-o wide`: show extra columns (node, IP, etc.)
- `-o yaml`: full YAML output
- `-o json`: JSON output
- `-w` or `--watch`: stream live changes
- `--sort-by=.metadata.creationTimestamp`: sort by creation time
- `--field-selector ...`: filter by field (e.g., status)
- `-l <key>=<value>`: filter by label
- `--dry-run=client -o yaml`: preview manifest without creating

Common pattern:

```bash
kubectl get <resource> -n <namespace> -o wide
kubectl get <resource> -A
kubectl get <resource> -w
kubectl get <resource> -o yaml
```

---

## 2) Cluster and Context

```bash
kubectl version --short
kubectl cluster-info
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context-name>
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl top nodes
```

---

## 3) Namespaces

```bash
kubectl get ns
kubectl create ns <name>
kubectl delete ns <name>
kubectl get all -n <namespace>
```

---

## 4) Pods (Most Frequent)

### List and inspect

```bash
kubectl get pods -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl get pods -n <namespace> -w
kubectl get pods -n <namespace> -o yaml
kubectl describe pod <pod-name> -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o yaml
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.phase}'
```

### Logs

```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs -f <pod-name> -n <namespace>
kubectl logs --tail=100 <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container-name> -n <namespace>
kubectl logs -f <pod-name> -c <container-name> -n <namespace>
kubectl logs --previous <pod-name> -n <namespace>
```

### Execute and debug

```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
kubectl cp <namespace>/<pod-name>:/path/in/container ./local-path
kubectl cp ./local-file <namespace>/<pod-name>:/tmp/
kubectl port-forward pod/<pod-name> 8080:80 -n <namespace>
```

### Delete/restart behavior

```bash
kubectl delete pod <pod-name> -n <namespace>
kubectl delete pod -l app=<label> -n <namespace>
```

---

## 5) Deployments

```bash
kubectl get deploy -n <namespace>
kubectl describe deploy <deploy-name> -n <namespace>
kubectl create deploy <name> --image=<image> -n <namespace>
kubectl scale deploy <deploy-name> --replicas=3 -n <namespace>
kubectl set image deploy/<deploy-name> <container-name>=<new-image> -n <namespace>
kubectl rollout status deploy/<deploy-name> -n <namespace>
kubectl rollout history deploy/<deploy-name> -n <namespace>
kubectl rollout undo deploy/<deploy-name> -n <namespace>
kubectl rollout restart deploy/<deploy-name> -n <namespace>
kubectl edit deploy <deploy-name> -n <namespace>
```

---

## 6) ReplicaSets (RS)

```bash
kubectl get rs -n <namespace>
kubectl describe rs <rs-name> -n <namespace>
kubectl scale rs <rs-name> --replicas=2 -n <namespace>
kubectl delete rs <rs-name> -n <namespace>
```

---

## 7) Services and Endpoints

```bash
kubectl get svc -n <namespace>
kubectl describe svc <svc-name> -n <namespace>
kubectl expose deploy <deploy-name> --port=80 --target-port=8080 -n <namespace>
kubectl get endpoints -n <namespace>
kubectl get endpointslice -n <namespace>
kubectl port-forward svc/<svc-name> 8080:80 -n <namespace>
```

---

## 8) ConfigMaps

```bash
kubectl get cm -n <namespace>
kubectl describe cm <cm-name> -n <namespace>
kubectl create cm <cm-name> --from-literal=key=value -n <namespace>
kubectl create cm <cm-name> --from-file=app.properties -n <namespace>
kubectl get cm <cm-name> -n <namespace> -o yaml
kubectl edit cm <cm-name> -n <namespace>
kubectl delete cm <cm-name> -n <namespace>
```

---

## 9) Secrets

```bash
kubectl get secrets -n <namespace>
kubectl describe secret <secret-name> -n <namespace>
kubectl create secret generic <secret-name> --from-literal=username=admin --from-literal=password=pass -n <namespace>
kubectl create secret generic <secret-name> --from-file=creds.txt -n <namespace>
kubectl get secret <secret-name> -n <namespace> -o yaml
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.password}' | base64 -d
kubectl edit secret <secret-name> -n <namespace>
kubectl delete secret <secret-name> -n <namespace>
```

---

## 10) StatefulSets and DaemonSets

```bash
kubectl get sts -n <namespace>
kubectl describe sts <sts-name> -n <namespace>
kubectl rollout status sts/<sts-name> -n <namespace>
kubectl scale sts <sts-name> --replicas=3 -n <namespace>

kubectl get ds -n <namespace>
kubectl describe ds <ds-name> -n <namespace>
kubectl rollout status ds/<ds-name> -n <namespace>
```

---

## 11) Jobs and CronJobs

```bash
kubectl get jobs -n <namespace>
kubectl describe job <job-name> -n <namespace>
kubectl logs job/<job-name> -n <namespace>
kubectl delete job <job-name> -n <namespace>

kubectl get cronjobs -n <namespace>
kubectl describe cronjob <cronjob-name> -n <namespace>
kubectl create job --from=cronjob/<cronjob-name> manual-run -n <namespace>
```

---

## 12) Ingress

```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
kubectl get ingress -A
kubectl get ingress <ingress-name> -n <namespace> -o yaml
```

---

## 13) Storage (PV, PVC, StorageClass)

```bash
kubectl get pv
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
kubectl get sc
kubectl describe pv <pv-name>
```

---

## 14) RBAC and Service Accounts

```bash
kubectl get sa -n <namespace>
kubectl describe sa <sa-name> -n <namespace>
kubectl get role,rolebinding -n <namespace>
kubectl get clusterrole,clusterrolebinding
kubectl auth can-i get pods -n <namespace>
kubectl auth can-i create deployments -n <namespace> --as=<user>
```

---

## 15) Events (Very Important for Troubleshooting)

```bash
kubectl get events -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl get events -A --sort-by=.lastTimestamp
kubectl events -n <namespace>
```

---

## 16) Resource Usage and Metrics

```bash
kubectl top pods -n <namespace>
kubectl top pod <pod-name> -n <namespace>
kubectl top nodes
```

---

## 17) Fast Troubleshooting Workflow

Use this exact sequence when something is down:

```bash
kubectl get pods -n <namespace> -o wide
kubectl get pods -n <namespace> --field-selector=status.phase!=Running
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --tail=100
kubectl logs <pod-name> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl get svc,endpoints -n <namespace>
kubectl get deploy,rs -n <namespace>
kubectl rollout status deploy/<deploy-name> -n <namespace>
```

Common failure checks:
- `ImagePullBackOff` -> image name/tag/registry credentials
- `CrashLoopBackOff` -> app crash, wrong command, missing env/config
- `Pending` -> scheduler constraints, no resources, PVC issues
- `Readiness probe failed` -> app not ready, wrong probe path/port

---

## 18) YAML/Manifest Operations

```bash
kubectl apply -f <file-or-dir>
kubectl apply -k <kustomize-dir>
kubectl delete -f <file-or-dir>
kubectl diff -f <file-or-dir>
kubectl replace -f <file>
kubectl patch deploy <deploy-name> -n <namespace> -p '{"spec":{"replicas":4}}'
kubectl explain deployment
kubectl explain deployment.spec.template.spec.containers
```

---

## 19) Output Formatting and Filtering (Daily Use)

```bash
kubectl get pods -n <namespace> -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[*].image,NODE:.spec.nodeName'
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get pods -A -l app=myapp
kubectl get pods -A --field-selector=status.phase=Running
```

---

## 20) Optional Productivity Aliases

```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
alias kdp='kubectl describe pod'
```

Then:

```bash
k get po -n <namespace>
k get po -w
k get po -o yaml
```

---

## Quick Minimal Command Set (If You Learn Only 15)

```bash
kubectl get pods -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl get pods -n <namespace> -w
kubectl describe pod <pod-name> -n <namespace>
kubectl logs -f <pod-name> -n <namespace>
kubectl logs --previous <pod-name> -n <namespace>
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
kubectl get svc -n <namespace>
kubectl get deploy,rs -n <namespace>
kubectl rollout status deploy/<deploy-name> -n <namespace>
kubectl rollout restart deploy/<deploy-name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl top pods -n <namespace>
kubectl apply -f <file>
kubectl delete -f <file>
```
