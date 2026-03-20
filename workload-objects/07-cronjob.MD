# Kubernetes CronJob — Developer Guide

A **CronJob** runs **Jobs on a schedule** (cron). It does **not** run Pods directly — it **creates Job objects**, which then run **Pods to completion** — same style as your other workload docs: **complete, crisp, scannable**.

---

## What is a CronJob?

A **CronJob** is a controller that:

- Evaluates a **cron schedule** on the control plane clock
- **Creates a Job** each time a schedule fires (subject to **concurrency** rules)
- Relies on the **Job** controller for retries, completions, and Pod lifecycle

```
CronJob  →  Job (per run)  →  Pod(s)  →  complete / fail
```

**Mental model:** “**Every schedule tick**, create **the same batch Job** again.”

> Deep dive on Job fields (`backoffLimit`, `parallelism`, etc.) → see **`06-job.MD`**.

---

## CronJob vs Job

| Object | Role |
|--------|------|
| **Job** | Run work **once** (or N completions) |
| **CronJob** | **Repeat** that pattern on a **schedule** |

---

## Schedule (`spec.schedule`)

Uses **standard cron** (5 fields in Kubernetes):

```
# ┌──────────── minute (0–59)
# │ ┌────────── hour (0–23)
# │ │ ┌──────── day of month (1–31)
# │ │ │ ┌────── month (1–12)
# │ │ │ │ ┌──── day of week (0–6, 0 = Sunday)
# │ │ │ │ │
# * * * * *
```

**Examples:**

| Expression | Meaning |
|------------|---------|
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour at minute 0 |
| `0 2 * * *` | Daily at 02:00 |
| `0 2 * * 1` | Every Monday 02:00 |

**Tip:** Use a cron explainer / validator when unsure (timezone matters — see below).

---

## Optional: `spec.timeZone`

- **`timeZone`** (e.g. `America/New_York`) — supported on **CronJob** in modern Kubernetes (check your cluster version).
- Without it, schedule uses **control plane timezone** (often **UTC** — **verify** in your environment).

---

## Core concepts (don’t skip)

### 1) `jobTemplate`

- Embeds a full **`Job` spec** (`spec.jobTemplate.spec`) — same fields as a standalone Job (`parallelism`, `backoffLimit`, `ttlSecondsAfterFinished`, Pod template, etc.).
- Pod template **`restartPolicy`** must still be **`Never`** or **`OnFailure`**.

### 2) `concurrencyPolicy`

| Policy | Behavior |
|--------|----------|
| **`Allow`** (default) | Multiple Jobs from the same CronJob **may run overlapping** |
| **`Forbid`** | If **last Job still running**, **skip** this schedule |
| **`Replace`** | If **last Job still running**, **terminate** it and start a **new** one |

Choose **`Forbid`** for “**at most one run at a time**” (common for ETL / migrations).

### 3) `startingDeadlineSeconds`

- If the controller **misses** a scheduled time (apiserver down, backlog, etc.), it may still start **late** — but only within this **window** (seconds).  
- **`nil`** / unset → no extra late-start cutoff from this field (behavior still depends on controller backlog).

### 4) History limits

| Field | Purpose |
|-------|---------|
| **`successfulJobsHistoryLimit`** | How many **succeeded** Jobs to keep (default **3**) |
| **`failedJobsHistoryLimit`** | How many **failed** Jobs to keep (default **1**) |

Tune for **debuggability** vs **etcd / API clutter**.

### 5) `suspend`

- **`spec.suspend: true`** — **no new Jobs** created from the schedule (maintenance / freeze).

---

## Create a CronJob

### Imperative (quick / demos)

```bash
kubectl create cronjob demo \
  --image=busybox:1.36 \
  --schedule="*/5 * * * *" \
  -n demo \
  -- /bin/sh -c "date; echo ok"
```

**Common ops:**

```bash
kubectl get cronjobs -n demo
kubectl get cj -n demo                    # short name
kubectl describe cronjob demo -n demo

# One-off Job from this CronJob template
kubectl create job manual-run --from=cronjob/demo -n demo

kubectl delete cronjob demo -n demo
```

### Declarative (recommended)

**cronjob-demo.yaml**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: demo
  namespace: demo
spec:
  schedule: "*/10 * * * *"
  # timeZone: "UTC"              # optional: set if your cluster supports it
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 200
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2
  suspend: false
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 3600
      backoffLimit: 3
      activeDeadlineSeconds: 600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: worker
              image: busybox:1.36
              command: ["/bin/sh", "-c", "date && sleep 5 && exit 0"]
              resources:
                requests:
                  cpu: 50m
                  memory: 64Mi
                limits:
                  cpu: 200m
                  memory: 128Mi
```

```bash
kubectl apply -f cronjob-demo.yaml
```

---

## Inspecting CronJobs and their Jobs

```bash
kubectl get cronjobs -n <ns>
kubectl get jobs -n <ns>              # Jobs created by CronJob have generated names
kubectl get pods -n <ns>              # Pods owned by those Jobs
kubectl describe cronjob <name> -n <ns>
```

**Find Jobs from a CronJob:** look at Job labels (`ownerReferences`) or list Jobs after a schedule fires.

**Logs:**

```bash
kubectl logs job/<job-name> -n <ns>
kubectl logs -l job-name=<job-name> -n <ns>   # if label exists
```

---

## Production best practices (industry standard)

1. **`concurrencyPolicy: Forbid`** when overlapping runs are dangerous (DB migrations, single-writer tasks).  
2. **`activeDeadlineSeconds`** on the **Job** template — cap runaway work.  
3. **`ttlSecondsAfterFinished`** — auto-delete finished Jobs.  
4. **History limits** — balance retention for debugging vs API noise.  
5. **Idempotent jobs** — schedules can overlap (if `Allow`) or retries can duplicate work.  
6. **Resources** — every tick can spawn load; set requests/limits.  
7. **Timezone clarity** — document whether schedule is **UTC** or use **`timeZone`** when supported.  
8. **Monitoring** — alert on **CronJob** not creating Jobs, or **Jobs** failing repeatedly.  
9. **`suspend`** — safe freeze for incidents or maintenance.  
10. **Security** — least-privilege RBAC; minimal image; secrets via **Secret**, not env in plaintext when avoidable.  
11. **Test manually** — `kubectl create job --from=cronjob/...` before trusting a new schedule.

---

## Troubleshooting CronJobs

### Quick checks

```bash
kubectl describe cronjob <name> -n <ns>
kubectl get jobs -n <ns> --sort-by=.metadata.creationTimestamp
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

### Symptom → action

| Symptom | Check |
|---------|--------|
| **No Jobs appearing** | **`suspend: true`**; invalid **schedule**; **RBAC**; controller issues |
| **Missed runs** | **`startingDeadlineSeconds`** too small; control plane load |
| **Overlaps you don’t want** | **`concurrencyPolicy: Allow`** → switch to **`Forbid`** |
| **Job fails every time** | Same as **Job** debugging — logs, `backoffLimit`, image, command |
| **Wrong local time** | **Timezone** — clarify UTC vs `timeZone` |

---

## Relationship to Job

- **CronJob** only **schedules**; **Job** defines **how** work runs.
- Fix batch logic in **`jobTemplate`**; fix timing in **`schedule`** / **`concurrencyPolicy`**.

---

## Cheat sheet

| Task | Command / field |
|------|------------------|
| Create (CLI) | `kubectl create cronjob NAME --image=IMG --schedule="CRON" -n NS -- <cmd>` |
| List | `kubectl get cronjob -n NS` |
| Suspend | `spec.suspend: true` |
| One-off run | `kubectl create job RUN --from=cronjob/NAME -n NS` |
| Overlap control | `spec.concurrencyPolicy` |
| Late start window | `spec.startingDeadlineSeconds` |
| Cleanup | `ttlSecondsAfterFinished` (on Job template), history limits |

---

## Summary

**CronJob** = **scheduled Jobs**. Master **`schedule`**, **`concurrencyPolicy`**, and a solid **`jobTemplate`** (deadlines, TTL, backoff, resources). Treat **timezone** explicitly in docs or **`timeZone`**, and validate with a **manual Job** (`--from=cronjob`) before relying on production schedules.
