# Kubernetes Job — Developer Guide

A **Job** runs a **Pod (or Pods) to completion**: **batch work**, **migrations**, **one-off tasks**, and **parallel fan-out** — same style as your other workload docs: **complete, crisp, scannable**.

---

## What is a Job?

A **Job** is a controller that:

- Creates **one or more Pods** from a template
- Retries **failed** Pods according to policy (until limits hit)
- Marks the Job **Complete** when **enough successful completions** occur
- Marks the Job **Failed** when limits (retries, time) are exceeded

**Mental model:** “**Run this work until it succeeds N times** (often **N = 1**), then stop.”

```
Job  →  Pod(s)  →  exit 0 = success; retries / limits = failure handling
```

> For **scheduled** runs (cron), use **CronJob** — it creates **Job** objects on a schedule. This doc focuses on **Job** itself.

---

## Job vs Deployment

| Aspect | Deployment | Job |
|--------|------------|-----|
| Purpose | Long-running service | **Run to completion** |
| Pod restarts | Keep replicas running | **Retry until success** or **fail Job** |
| Typical `restartPolicy` in template | `Always` (via ReplicaSet) | **`Never`** or **`OnFailure`** only |

---

## Core concepts (don’t skip)

### 1) Pod template `restartPolicy`

In a Job’s Pod spec, **`restartPolicy`** must be **`Never`** or **`OnFailure`** (not `Always`).

| Value | Meaning |
|-------|---------|
| **`OnFailure`** | Pod **restarts on the same node** on failure (container restarts) |
| **`Never`** | Pod **not restarted** on failure; Job controller may create **a new Pod** (see `backoffLimit`) |

### 2) `completions` and `parallelism`

| Field | Meaning |
|-------|---------|
| **`completions`** | How many **successful** Pod completions the Job needs (default **1**) |
| **`parallelism`** | Max **concurrent** Pods running at once (default **1**; **0** = pause) |

**Examples:**

- **One-shot:** `completions: 1`, `parallelism: 1` (default) — single Pod, run once to success.
- **Fixed parallel batch:** `completions: 10`, `parallelism: 3` — 10 successful runs, up to 3 at a time.
- **Work-queue style:** `completions` unset with **parallelism only** — rare pattern; see Kubernetes docs for your use case.

### 3) `completionMode` (optional, advanced)

| Mode | Behavior |
|------|----------|
| **`NonIndexed`** (default) | Pods are interchangeable |
| **`Indexed`** | Each Pod gets **`JOB_COMPLETION_INDEX`** — useful for **sharded** batch (0..completions-1) |

### 4) Failure and limits

| Field | Purpose |
|-------|---------|
| **`backoffLimit`** | Failed retries before Job is **Failed** (default **6**); set **`0`** to fail on first Pod failure |
| **`activeDeadlineSeconds`** | Max **duration** for the Job; Job **terminates** Pods and marks **Failed** if exceeded |
| **`ttlSecondsAfterFinished`** | Auto-delete Job (and Pods) **after** finish — keeps API/etcd clean |

### 5) Suspend

- **`spec.suspend: true`** — Job does **not** create Pods until set back to `false` (GitOps / manual gating).

### 6) Selector (advanced)

- Usually **auto-generated**. Avoid manual **`spec.selector`** unless you know **Pod tracking** implications.

---

## Create a Job

### Imperative (quick / exams)

**Single-run Job from image + command:**

```bash
kubectl create job demo --image=busybox:1.36 -n demo -- echo "hello"
```

**From CronJob (one-off run of that template):**

```bash
kubectl create job manual-run --from=cronjob/my-cron -n demo
```

**Common ops:**

```bash
kubectl get jobs -n demo
kubectl describe job demo -n demo
kubectl logs job/demo -n demo
kubectl logs -f job/demo -n demo

kubectl delete job demo -n demo
```

**Note:** Complex parallelism / indexed Jobs → use **YAML** (`apply`).

### Declarative (recommended)

**Simple Job (one Pod, one success):**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: demo
  namespace: demo
spec:
  ttlSecondsAfterFinished: 100
  backoffLimit: 4
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: worker
          image: busybox:1.36
          command: ["/bin/sh", "-c", "echo done && exit 0"]
```

```bash
kubectl apply -f job-demo.yaml
```

**Parallel Job (multiple completions):**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-demo
  namespace: demo
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 6
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command: ["/bin/sh", "-c", "sleep 2 && exit 0"]
```

---

## Inspecting Jobs

```bash
kubectl get jobs -n <ns>
kubectl get jobs -n <ns> -o wide
kubectl describe job <name> -n <ns>
kubectl get pods -n <ns> -l job-name=<name>
```

**Status columns:** `COMPLETIONS` (e.g. `3/5`), `DURATION`, `AGE`.

---

## Production best practices (industry standard)

1. **Set `activeDeadlineSeconds`** — avoid stuck Jobs consuming quota forever.  
2. **Set `ttlSecondsAfterFinished`** — auto-cleanup finished Jobs in busy clusters.  
3. **Tune `backoffLimit`** — balance retries vs noisy failure loops.  
4. **Resources** — requests/limits; parallel Jobs multiply load.  
5. **Idempotent work** — Jobs **can** run more than once; design tasks to be safe on retry.  
6. **Logs + exit codes** — app should **exit non-zero** on failure for clear Job status.  
7. **Security** — non-root, minimal image, RBAC scoped to what the batch needs.  
8. **CronJob** — for schedules; keep Job YAML as the **template of record** where possible.  
9. **Suspend** — use for maintenance windows or approval gates.  
10. **Indexed Jobs** — when each shard must know its **index** (partitioned backfill).

---

## Troubleshooting Jobs

### Quick checks

```bash
kubectl describe job <name> -n <ns>
kubectl get pods -n <ns> -l job-name=<name>
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
```

### Symptom → action

| Symptom | Check |
|---------|--------|
| **Job Failed** | `describe job` → events; `backoffLimit` exhausted; exit code in logs |
| **Pods Pending** | Quota, image pull, scheduling, PVC |
| **Stuck Running** | `activeDeadlineSeconds`; app hung; probe not applicable same as Deployment |
| **Too many retries** | Misconfigured command; missing env/Secret |
| **0 Pods** | **`suspend: true`**; invalid template |

### Useful

```bash
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl delete job <name> -n <ns>   # deletes Pods (policy-dependent)
```

---

## Relationship to CronJob

- **CronJob** creates **Job** objects on a schedule.
- Debug Cron issues by inspecting the **Job** and **Pods** the CronJob spawned.

---

## Cheat sheet

| Task | Command / field |
|------|------------------|
| Create (CLI) | `kubectl create job NAME --image=IMG -- <cmd>` |
| From CronJob | `kubectl create job NAME --from=cronjob/CJ -n NS` |
| Logs | `kubectl logs job/NAME -n NS` |
| Delete | `kubectl delete job NAME -n NS` |
| Parallelism | `spec.parallelism`, `spec.completions` |
| Time limit | `spec.activeDeadlineSeconds` |
| Auto cleanup | `spec.ttlSecondsAfterFinished` |
| Retries | `spec.backoffLimit` |
| Pause | `spec.suspend: true` |

---

## Summary

**Job** = **run Pod(s) to completion** with **retries** and **optional parallelism**. Prefer **`kubectl apply`** for anything beyond a simple one-liner; set **deadlines**, **TTL**, and **backoff** for production. Treat **failures** as normal — make work **idempotent** and **exit codes** meaningful.
