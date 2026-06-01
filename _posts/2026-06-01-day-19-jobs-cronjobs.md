---
title: "Day 19: Jobs and CronJobs — Batch Workloads, Parallelism, and Scheduled Backups"
date: 2026-06-01
categories: [Kubernetes]
tags: [kubernetes, job, cronjob, batch, parallelism, backup, devops]
excerpt: "A Deployment is the wrong workload type for anything that has a finishing line. Day 19 introduces the two right ones: Job (run a Pod to completion, retry on failure, scale work out across parallel replicas) and CronJob (do that on a schedule). Build a parallel Indexed Job that fans out across multiple Pods, see backoffLimit and activeDeadlineSeconds turn a flaky workload into a fast-failing one, then schedule a nightly pg_dump CronJob against the Day 17 Postgres StatefulSet — with concurrencyPolicy: Forbid so a slow night never collides with the next."
header:
  overlay_color: "#0d1117"
  overlay_filter: 0.6
author_profile: false
read_time: true
toc: true
toc_sticky: true
toc_label: "On this page"
---

> **30 Days of DevOps** — Day 19 of 30. [← Day 18: Init Containers and Sidecars](/articles/2026/05/28/day-18-init-containers-sidecars/)

Every workload controller you have used in this series so far — Deployments (Day 5+), StatefulSets (Day 17), DaemonSet-style ingress controllers (Day 7) — exists to keep a Pod running **forever**. Their reconciliation logic is built on the assumption that a Pod that exits is a Pod that needs to be restarted. That assumption is exactly wrong for anything with a finishing line.

A `pg_dump`, a database migration, a one-off image import, a nightly report, a parallel fan-out of `N` work items — none of these should "restart on success." A Deployment running them would loop forever, and the very phrase "the deploy succeeded" would become a lie. The right primitives are:

- **`Job`** — runs one or more Pods to **completion**. Tracks success count vs the desired count, retries failed Pods up to `backoffLimit`, supports parallelism with `completions` × `parallelism`, and stops the moment the target is met. The Job itself transitions to `Complete` (or `Failed`), and its Pods are kept around so you can read their logs.
- **`CronJob`** — a tiny scheduler on top of `Job`. It owns a `schedule` (standard cron syntax), a `jobTemplate`, and a history. Every time the schedule fires, the CronJob controller creates a new Job from the template. `concurrencyPolicy` decides what happens when a previous Job is still running; `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` keep the cluster clean.

Today you will use both: a parallel Job that fans out across multiple Pods using **Indexed completion mode** (so each Pod knows which chunk of work it owns), and a CronJob that takes a nightly `pg_dump` of the Postgres StatefulSet from Day 17 into a PersistentVolume.

## What you will build

By the end of this article you will have:

- A simple single-Pod `Job` that runs a short batch script and goes `Complete`
- A **parallel Indexed Job** with `completions: 5, parallelism: 2` — five Pods total, never more than two running at once, each one knowing its index from `JOB_COMPLETION_INDEX`
- A demonstration of **`backoffLimit`** and **`activeDeadlineSeconds`** turning a deliberately-failing Job into a fast `Failed` (instead of an infinite retry loop), and the difference between **`restartPolicy: Never`** (failures = new Pods) and **`OnFailure`** (failures = restarts inside the same Pod)
- A `PersistentVolumeClaim` (`postgres-backups`, 5Gi) in the `database` namespace to hold backup files
- A **`CronJob`** running `pg_dump` against `postgres-0` every night at 02:30 IST, writing timestamped dumps into that PVC, with `concurrencyPolicy: Forbid`, `startingDeadlineSeconds: 300`, `successfulJobsHistoryLimit: 3`, `failedJobsHistoryLimit: 1`
- A live trigger of the CronJob (so you don't have to wait until 02:30) and a verification that the dump file appeared in the PVC

---

## How a parallel Job schedules its Pods

The three numbers that govern a Job's execution are `completions` (how many successful Pods you want), `parallelism` (how many Pods may run *at the same time*), and `backoffLimit` (how many failures across all Pods before the Job gives up). The controller balances them on every reconcile.

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart LR
    JOB["Job\ncompletions: 5\nparallelism: 2\nbackoffLimit: 3"]:::controller

    subgraph WAVE1 ["t = 0 s · first wave"]
        P0["Pod 0\n(index 0)"]:::pod
        P1["Pod 1\n(index 1)"]:::pod
    end

    subgraph WAVE2 ["t ≈ 30 s"]
        P2["Pod 2\n(index 2)"]:::pod
        P3["Pod 3\n(index 3)"]:::pod
    end

    subgraph WAVE3 ["t ≈ 60 s"]
        P4["Pod 4\n(index 4)"]:::pod
    end

    DONE["Job\nCOMPLETIONS 5/5\nCondition: Complete"]:::done

    JOB -->|"1 · spawn up to\n   parallelism"| WAVE1
    WAVE1 -->|"2 · each Pod exits 0,\n   slot frees up"| WAVE2
    WAVE2 -->|"3 · last slot fills\n   with index 4"| WAVE3
    WAVE3 -->|"4 · 5 successes recorded"| DONE

    classDef controller fill:#2a1f10,stroke:#d29922,color:#e6edf3
    classDef pod fill:#1d2d1d,stroke:#3fb950,color:#e6edf3
    classDef done fill:#1a2744,stroke:#58a6ff,color:#e6edf3
```

**Reading this diagram:**

Read left to right — the horizontal axis is time, the boxes group the Pods that are running together at each moment.

The **Job** controller (amber, left) is the decision-maker. On every reconcile it asks one question: *"how many Pods do I currently have running, and is that less than `parallelism` while we still need more `completions`?"* If yes, it spawns more.

**Arrow 1** is the **first wave**. At `t = 0`, the controller spawns `parallelism = 2` Pods — Pod 0 and Pod 1 (green, the actual workload). Each one gets a unique `JOB_COMPLETION_INDEX` (0 and 1 respectively), an environment variable Kubernetes injects automatically when `completionMode: Indexed` is set. The Pod's command can branch on this index to process a specific slice of work.

**Arrow 2** is the **slot-refill moment**. Both first-wave Pods finish their task and exit `0`. The controller marks two completions, sees the running count drop to zero, sees the target (`5`) is still not met, and immediately spawns Pods 2 and 3 — the next two indices in the sequence. Crucially, it does this in **the same reconcile loop** as the success detection; you do not have to wait a polling interval.

**Arrow 3** is the **last slot**. Wave 2 finishes, four completions are now recorded, only one more is needed. The controller spawns just Pod 4 — it does not start a second one even though `parallelism` allows two, because that would mean six total Pods and only five are required.

**Arrow 4** closes the loop: Pod 4 exits `0`, the controller records the fifth completion, and the Job transitions to `Complete` (blue, the terminal state). The Pods are *not* deleted — `kubectl logs job/<name> --all-containers --prefix` still reads every Pod's output, which is what makes Jobs usable for diagnostic batch work.

The model that governs failure is symmetric. Replace "exits 0" above with "exits non-zero" and a `backoffLimit: 3` Job tolerates three failures across all Pods before transitioning to `Failed`. The fourth failure ends the Job; the third one starts the *next* Pod with an exponentially backed-off delay. `activeDeadlineSeconds` overrides both rules: when the Job hits that wall-clock age, it stops creating new Pods and transitions to `Failed` regardless of how many completions are left.

---

## Prerequisites

This article continues from Day 18. Required state:

- The `devops-cluster` kind cluster, kubectl 1.29+
- The Postgres StatefulSet from Day 17 running in the `database` namespace (StatefulSet `postgres`, Headless Service `postgres`, Secret `postgres-secret` with key `password`, default StorageClass `standard`)
- The Day 17 row in the `notes` table (so the dump has something to dump)

Pre-flight check:

```bash
# Postgres is up and has the row from Day 17
kubectl exec -n database postgres-0 -- \
  psql -U postgres -d appdb -c "SELECT count(*) FROM notes;"

# Default StorageClass is present (needed for the backups PVC in Part 4)
kubectl get storageclass
```

Expected output:

```text
 count
-------
     1
(1 row)

NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  3w
```

| Tool | Minimum version | Check |
|---|---|---|
| kubectl | 1.29 | `kubectl version --client` |
| Helm | 3.14 | `helm version --short` |

---

## Part 1 — A trivial single-Pod Job

The simplest possible Job: one Pod, runs a short script, exits `0`, Job goes `Complete`.

```bash
mkdir -p ~/30-days-devops/day-19 && cd ~/30-days-devops/day-19

cat > job-hello.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
  namespace: database          # no PSS enforcement here — short manifests
spec:
  # backoffLimit: how many Pod failures before the Job is marked Failed.
  # Default is 6. Setting 0 makes the Job fail fast for the demo below.
  backoffLimit: 2
  # Auto-delete the Job (and its Pods) 5 minutes after completion. Without
  # this, finished Jobs hang around forever — the cluster's "completed Job
  # graveyard" is the most common batch-related kubectl get clutter.
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never     # one of: Never | OnFailure (not: Always)
      containers:
        - name: hello
          image: busybox:1.36
          command: ["sh", "-c", "echo 'hello from a Job'; sleep 3; echo done"]
EOF

kubectl apply -f job-hello.yaml
```

Expected output:

```text
job.batch/hello created
```

Watch it complete:

```bash
kubectl wait --for=condition=complete job/hello -n database --timeout=60s

kubectl get job -n database hello
kubectl logs -n database job/hello
```

Expected output:

```text
job.batch/hello condition met

NAME    STATUS     COMPLETIONS   DURATION   AGE
hello   Complete   1/1           5s         12s

hello from a Job
done
```

`COMPLETIONS 1/1` is the headline number — one success out of one required. The Pod itself stays around (with `STATUS Completed`) for `ttlSecondsAfterFinished` so its logs remain readable.

---

## Part 2 — A parallel Indexed Job

A more useful pattern: fan a workload out across multiple Pods, each one handling a slice of the input identified by its index.

```bash
cat > job-parallel.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-demo
  namespace: database
spec:
  completions: 5               # we want 5 successful Pods in total
  parallelism: 2               # never run more than 2 Pods at the same time
  completionMode: Indexed      # each Pod gets a unique JOB_COMPLETION_INDEX (0..4)
  backoffLimit: 3
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "pod index=$JOB_COMPLETION_INDEX starting on $(hostname)"
              # Sleep a few seconds, varied per index, to make the parallelism
              # visible in `kubectl get pod --watch`. Using $JOB_COMPLETION_INDEX
              # (which busybox sh DOES support, unlike the bash-ism $RANDOM)
              # gives each Pod a deterministic, identifiable runtime: index 0
              # takes 5s, index 1 takes 6s, …, index 4 takes 9s.
              sleep $((5 + JOB_COMPLETION_INDEX))
              echo "pod index=$JOB_COMPLETION_INDEX done"
EOF

kubectl apply -f job-parallel.yaml
```

Expected output:

```text
job.batch/parallel-demo created
```

Watch the Pods in another terminal. The `batch.kubernetes.io/job-name` label (the modern, non-deprecated form of the older `job-name` label) selects every Pod the Job controller spawned:

```bash
kubectl get pod -n database -l batch.kubernetes.io/job-name=parallel-demo --watch
```

Expected output (interleaved over ~25 seconds — indices 0 and 1 sleep 5 s and 6 s; indices 2 and 3 start as they free up, sleeping 7 s and 8 s; index 4 starts last and sleeps 9 s):

```text
NAME                  READY   STATUS              RESTARTS   AGE
parallel-demo-0-abcd  0/1     ContainerCreating   0          1s
parallel-demo-1-efgh  0/1     ContainerCreating   0          1s
parallel-demo-0-abcd  1/1     Running             0          3s
parallel-demo-1-efgh  1/1     Running             0          3s
parallel-demo-0-abcd  0/1     Completed           0          8s
parallel-demo-2-ijkl  0/1     ContainerCreating   0          8s
parallel-demo-1-efgh  0/1     Completed           0          9s
parallel-demo-3-mnop  0/1     ContainerCreating   0          9s
parallel-demo-2-ijkl  0/1     Completed           0          17s
parallel-demo-4-qrst  0/1     ContainerCreating   0          17s
parallel-demo-3-mnop  0/1     Completed           0          19s
parallel-demo-4-qrst  0/1     Completed           0          28s
```

Two Pods at a time, indices `0..4`. The Pod name includes the index (`parallel-demo-0-...`) — a feature of `completionMode: Indexed` that makes it trivial to identify which Pod handled which slice.

Confirm the index landed inside each Pod:

```bash
for i in 0 1 2 3 4; do
  pod=$(kubectl get pod -n database \
        -l batch.kubernetes.io/job-name=parallel-demo,batch.kubernetes.io/job-completion-index=$i \
        -o jsonpath='{.items[0].metadata.name}')
  echo "--- index $i (pod $pod) ---"
  kubectl logs -n database "$pod"
done
```

Expected output:

```text
--- index 0 (pod parallel-demo-0-abcd) ---
pod index=0 starting on parallel-demo-0-abcd
pod index=0 done
--- index 1 (pod parallel-demo-1-efgh) ---
pod index=1 starting on parallel-demo-1-efgh
pod index=1 done
... (and so on for 2, 3, 4) ...
```

Each Pod's `$JOB_COMPLETION_INDEX` matched the index baked into its name. In a real workload, the command would use that index to read its slice of an input file, fetch a specific S3 object, claim a row range from a queue table, etc.

---

## Part 3 — Failure handling: `backoffLimit`, `activeDeadlineSeconds`, and the `restartPolicy` choice

Apply a deliberately failing Job to see how the controller responds:

```bash
cat > job-failing.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: failing
  namespace: database
spec:
  backoffLimit: 2              # tolerate 2 failures, fail the Job on the 3rd
  activeDeadlineSeconds: 60    # but no matter what, give up after 60 s wall-clock
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never     # failures spawn NEW Pods (not restarts of the same Pod)
      containers:
        - name: oops
          image: busybox:1.36
          command: ["sh", "-c", "echo attempt; sleep 2; exit 1"]
EOF

kubectl apply -f job-failing.yaml
```

Expected output:

```text
job.batch/failing created
```

Wait ~30 seconds and inspect:

```bash
kubectl get job -n database failing
kubectl get pod -n database -l job-name=failing
```

Expected output:

```text
NAME      STATUS   COMPLETIONS   DURATION   AGE
failing   Failed   0/1           25s        30s

NAME                  READY   STATUS   RESTARTS   AGE
failing-abc12         0/1     Error    0          28s
failing-def34         0/1     Error    0          22s
failing-ghi56         0/1     Error    0          14s
```

Three Pods, three failures. The Job's `STATUS Failed` and `COMPLETIONS 0/1` reflect that `backoffLimit: 2` allows **2 failures** and gives up on the **3rd**. The Pods stay around so you can read their logs — `kubectl logs failing-ghi56 -n database` would show `attempt`.

Read the Job's terminal condition to see *which* limit fired:

```bash
kubectl get job -n database failing \
  -o jsonpath='{range .status.conditions[*]}{.type}={.reason}{": "}{.message}{"\n"}{end}'
```

Expected output:

```text
Failed=BackoffLimitExceeded: Job has reached the specified backoff limit
```

The condition tells you `BackoffLimitExceeded`, not `DeadlineExceeded` — meaning the three quick failures hit the retry cap before the 60-second wall clock had a chance to fire. If the command had been `sleep 600` instead (one slow failure), the Job would have terminated with `DeadlineExceeded` after 60 seconds.

**`restartPolicy: Never` vs `OnFailure`** is the other choice you have to make:

- **`restartPolicy: Never`** — what we just used. Each failure spawns a **new Pod** (a fresh container, fresh filesystem, fresh logs). Easy to see the history (`kubectl get pod` shows every attempt as a separate row). Higher kubelet overhead per retry.
- **`restartPolicy: OnFailure`** — the *same Pod* restarts in place, incrementing `RESTARTS`. One row in `kubectl get pod`, but `kubectl logs` only shows the latest attempt (`--previous` gets the one before). Lower overhead. The `backoffLimit` still applies — Kubernetes counts container restarts, not Pod creations.

For Jobs that mutate state and need a known-clean start, prefer `Never`. For Jobs whose work is idempotent and replay-safe, prefer `OnFailure`.

Clean up the demo Jobs:

```bash
kubectl delete -f job-hello.yaml -f job-parallel.yaml -f job-failing.yaml
```

Expected output:

```text
job.batch "hello" deleted
job.batch "parallel-demo" deleted
job.batch "failing" deleted
```

---

## Part 4 — A nightly `pg_dump` CronJob

The CronJob is where Jobs become operational. We want every night at **02:30 IST** to take a logical dump of the `appdb` database (the one Day 17 set up), tagged with the timestamp, into a PersistentVolume.

### 4.1 — Create the backups PVC

```bash
cat > backups-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backups
  namespace: database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF

kubectl apply -f backups-pvc.yaml
```

Expected output:

```text
persistentvolumeclaim/postgres-backups created
```

Because the StorageClass uses `WaitForFirstConsumer` (Day 17), this PVC stays `Pending` until the first Job Pod that mounts it is scheduled — that is expected, not a bug.

### 4.2 — Define the CronJob

```bash
cat > cronjob-backup.yaml << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: database
spec:
  # Standard cron syntax. The CronJob controller defaults to UTC; spec.timeZone
  # (stable since Kubernetes 1.27) lets us pin to IST explicitly so the
  # "every night at 02:30 IST" intent does not drift across DST or operator
  # confusion. 02:30 IST = 21:00 UTC the previous day; specifying timeZone
  # spares the next-on-call the mental arithmetic.
  schedule: "30 2 * * *"
  timeZone: "Asia/Kolkata"

  # concurrencyPolicy: Forbid — if the previous Job is still running when
  # the next schedule fires (e.g., a slow dump that ran past 02:30 the
  # next night), SKIP this firing. The alternatives are Allow (overlap)
  # and Replace (kill the previous Job and start a new one). Forbid is
  # the safe default for write-heavy or DB-touching workloads.
  concurrencyPolicy: Forbid

  # startingDeadlineSeconds — if the controller missed the scheduled time
  # by more than this many seconds (because the API server was down or the
  # controller was restarting), DROP that firing instead of running it
  # late. 300 = "if we're more than 5 minutes behind, skip this one."
  startingDeadlineSeconds: 300

  # History limits prevent the namespace from accumulating completed Jobs
  # forever. Keep the last 3 successes (for log inspection) and 1 failure.
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1

  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 600   # a dump should finish in well under 10 min
      template:
        spec:
          restartPolicy: OnFailure
          volumes:
            - name: backups
              persistentVolumeClaim:
                claimName: postgres-backups
          containers:
            - name: dump
              # postgres:16-alpine ships with pg_dump baked in — same image
              # as the StatefulSet, no extra build needed.
              image: postgres:16-alpine
              env:
                # pg_dump reads PGPASSWORD from the env. Pulling it from the
                # same Secret the StatefulSet uses guarantees the dump
                # credentials stay in lockstep with the database.
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: password
              command:
                - sh
                - -c
                - |
                  set -eu
                  STAMP=$(date -u +%FT%TZ)
                  FILE=/backups/appdb-${STAMP}.dump
                  echo "dumping appdb to ${FILE}"
                  # -F c = custom (compressed binary) format — pg_restore-compatible
                  # postgres-0.postgres.database.svc.cluster.local is the
                  # stable Pod DNS from Day 17's Headless Service.
                  pg_dump \
                    -h postgres-0.postgres.database.svc.cluster.local \
                    -U postgres \
                    -d appdb \
                    -F c \
                    -f "${FILE}"
                  ls -lh /backups
              volumeMounts:
                - name: backups
                  mountPath: /backups
EOF

kubectl apply -f cronjob-backup.yaml
```

Expected output:

```text
cronjob.batch/postgres-backup created
```

Inspect the CronJob:

```bash
kubectl get cronjob -n database postgres-backup
```

Expected output:

```text
NAME              SCHEDULE      TIMEZONE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
postgres-backup   30 2 * * *    Asia/Kolkata   False     0        <none>          20s
```

`SUSPEND: False` means the controller will fire on schedule; `ACTIVE: 0` because no Job from this CronJob is currently running.

### 4.3 — Trigger the CronJob manually (don't wait until 02:30)

You can create an ad-hoc Job from the CronJob's template with one command, useful for both demos and runbooks:

```bash
kubectl create job -n database --from=cronjob/postgres-backup backup-manual-1
```

Expected output:

```text
job.batch/backup-manual-1 created
```

Wait for it:

```bash
kubectl wait --for=condition=complete job/backup-manual-1 -n database --timeout=120s
kubectl logs -n database job/backup-manual-1
```

Expected output:

```text
job.batch/backup-manual-1 condition met

dumping appdb to /backups/appdb-2026-06-01T11:34:09Z.dump
total 4K
-rw-r--r--    1 postgres postgres        3.2K Jun  1 11:34 appdb-2026-06-01T11:34:09Z.dump
```

The dump file is in the PVC. To prove it survives outside the Job's Pod, exec into a one-shot Pod that mounts the same PVC and list it:

```bash
kubectl run -n database -it --rm pvc-inspect \
  --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"b","persistentVolumeClaim":{"claimName":"postgres-backups"}}],"containers":[{"name":"x","image":"busybox:1.36","stdin":true,"tty":true,"command":["sh","-c","ls -lh /backups; sleep 2"],"volumeMounts":[{"name":"b","mountPath":"/backups"}]}]}}' \
  -- sh
```

Expected output:

```text
total 4K
-rw-r--r--    1 1000     1000        3.2K Jun  1 11:34 appdb-2026-06-01T11:34:09Z.dump
pod "pvc-inspect" deleted
```

The dump is durable; even after the Job's Pod is garbage-collected, the file stays in the PVC.

---

## Part 5 — CronJob hygiene in action

Two more behaviours worth seeing live, even if the schedule itself is once-a-day.

**Concurrency `Forbid` in action.** Fire two manual Jobs back-to-back; the second's underlying CronJob-spawned creation would be skipped if both were within the same minute via the schedule. To see the Forbid skip from the CronJob directly, you have to wait for a scheduled firing — but you can simulate by setting the schedule to every minute briefly:

```bash
# Temporarily speed the schedule up so we can observe two firings
kubectl patch cronjob -n database postgres-backup \
  --type=merge \
  -p '{"spec":{"schedule":"* * * * *","timeZone":"Etc/UTC"}}'

# Make the Job slow so the next firing arrives while it's still running
kubectl patch cronjob -n database postgres-backup --type=merge -p '{
  "spec": {
    "jobTemplate": {
      "spec": {
        "template": {
          "spec": {
            "containers": [
              {"name":"dump","image":"postgres:16-alpine","command":["sh","-c","sleep 90"]}
            ]
          }
        }
      }
    }
  }
}'

# Watch — you should see one Job created per minute, but a Forbid skip
# logged on the CronJob's events when the next minute fires while the
# previous Job is still in its 90-second sleep.
kubectl get events -n database --field-selector involvedObject.name=postgres-backup --watch
```

Expected output (after ~3 minutes):

```text
LAST SEEN   TYPE      REASON              OBJECT                      MESSAGE
30s         Normal    SuccessfulCreate    cronjob/postgres-backup     Created job postgres-backup-29347620
30s         Normal    SawCompletedJob     cronjob/postgres-backup     Saw completed job: ...
1m          Warning   JobAlreadyActive    cronjob/postgres-backup     Not starting job because prior execution is running and concurrency policy is Forbid
```

`Warning: JobAlreadyActive ... concurrency policy is Forbid` — the CronJob refused to start a second concurrent Job. That is the entire point of `Forbid`.

Restore the original schedule and dump command:

```bash
kubectl apply -f cronjob-backup.yaml
```

Expected output:

```text
cronjob.batch/postgres-backup configured
```

**History limits in action.** After several runs, only the last 3 successful Jobs and 1 failed Job are kept:

```bash
kubectl get jobs -n database --sort-by=.metadata.creationTimestamp | tail -10
```

Older completed Jobs have been garbage-collected automatically — you do not have to write a cleanup CronJob to clean up other CronJobs.

---

## Common Errors

**1. `pg_dump: error: connection to server ... failed: FATAL: password authentication failed`**

The CronJob's Pod could not authenticate. Three causes, in order of likelihood:

```bash
# (a) The PGPASSWORD env var did not get populated. Verify the Secret name/key.
kubectl get secret -n database postgres-secret -o jsonpath='{.data.password}' | base64 -d | head -c 20
echo
# (b) The Postgres server is reachable but expects a different user.
#     pg_dump defaults to using the OS user; the article passes -U postgres explicitly.
# (c) Network: the host name is wrong. The headless service's per-Pod DNS
#     requires `serviceName` on the StatefulSet (Day 17, Common Errors #1).
kubectl exec -n database statefulset/postgres -- nc -z postgres-0.postgres.database.svc.cluster.local 5432 \
  2>&1 || true
```

**2. Job stays `Pending` indefinitely with `FailedScheduling`**

```text
0/3 nodes are available: persistentvolumeclaim "postgres-backups" not found.
```

The CronJob references a PVC that does not exist (or is in a different namespace). The PVC must be in the same namespace as the Job's Pod (`database` in this article).

Fix:

```bash
kubectl get pvc -n database postgres-backups
# If missing, re-apply Part 4.1.
```

**3. `backoffLimit: 0` makes the Job fail on the FIRST transient error**

A Job set to `backoffLimit: 0` allows **zero** failures before the Job is marked `Failed`. If your workload has any non-deterministic startup (DNS not yet warmed, a one-off network glitch), the Job dies on the first try.

Fix: never set `backoffLimit: 0` for production batch work. The default of `6` is reasonable; bump it to `10` for genuinely flaky downstream dependencies. Use `activeDeadlineSeconds` to bound *time*, not `backoffLimit: 0` to bound *attempts*.

**4. CronJob fires on a schedule that does not match what you expected**

```bash
kubectl get cronjob -n database postgres-backup
# TIMEZONE column shows Etc/UTC (the default), not Asia/Kolkata.
```

Without `spec.timeZone`, the CronJob runs in **UTC**. `30 2 * * *` then means 02:30 UTC, which is 08:00 IST — not what you wrote down. The article sets `timeZone: "Asia/Kolkata"` for this reason; `spec.timeZone` has been stable since Kubernetes 1.27, so it is safe to use on any recent cluster.

**5. Two CronJob Pods overlap even with `concurrencyPolicy: Allow`, but the second one fails because the PVC is already mounted RWO**

```text
Error: Unable to attach or mount volumes: ... volume "..." is already used by pod ...
```

`ReadWriteOnce` PVCs can only be mounted by Pods on one node at a time. If two Jobs of the same CronJob run concurrently, the second cannot attach the PVC and stays `ContainerCreating` forever.

Fix: use `concurrencyPolicy: Forbid` (the article's choice) or switch to a `ReadWriteMany`-capable StorageClass for the backups PVC. On kind's `local-path`, RWX is not available — Forbid is the right answer.

**6. CronJob fires three times in quick succession after an API server outage**

If the controller was offline for, say, 4 hours and the CronJob fires every 30 minutes, by default the controller will try to make up the **eight missed runs** in quick succession when it comes back. That can take a tiny chip-shot dump job and turn it into a thundering herd against the database.

Fix: set `startingDeadlineSeconds` (the article uses `300`). Any scheduled time more than 5 minutes in the past is dropped — the controller only fires the next future schedule, not the backlog.

---

## Recap

In this article you:

- Mapped the difference between **continuous controllers** (Deployment, StatefulSet) and **terminating workload controllers** (Job, CronJob) — and why the latter exist for anything with a finishing line
- Walked through the Job controller's parallelism math with a Mermaid diagram: `completions` × `parallelism` × `backoffLimit`, with the controller balancing all three on every reconcile
- Ran three Jobs hands-on: a trivial single-Pod Job, a **parallel Indexed Job** (`completions: 5, parallelism: 2, completionMode: Indexed`) where each Pod got a unique `JOB_COMPLETION_INDEX` baked into its name and env, and a **failing Job** that hit `BackoffLimitExceeded` exactly when it should have
- Learned the **`restartPolicy: Never` vs `OnFailure`** trade-off: `Never` for state-mutating jobs that need a clean Pod per retry; `OnFailure` for idempotent work where the restart count is what matters
- Built a **production-grade CronJob** that runs `pg_dump` against the Day 17 Postgres StatefulSet every night at 02:30 IST (via `spec.timeZone: Asia/Kolkata`), writes timestamped dumps into a 5 Gi `postgres-backups` PVC, refuses to overlap with itself (`concurrencyPolicy: Forbid`), drops late firings (`startingDeadlineSeconds: 300`), and self-prunes its history (`successfulJobsHistoryLimit: 3 / failedJobsHistoryLimit: 1`)
- Triggered the CronJob manually with `kubectl create job --from=cronjob/...`, confirmed the dump landed in the PVC, and inspected the durable file from a second Pod
- Observed Forbid's `JobAlreadyActive` warning live by temporarily speeding the schedule up and slowing the dump down
- Worked through six common failures: password auth, missing PVC, the `backoffLimit: 0` footgun, UTC-vs-IST schedule confusion, the RWO + Allow overlap conflict, and the thundering-herd after an outage that `startingDeadlineSeconds` exists to prevent

The cluster now has both **workloads** and **operations** declared as Kubernetes resources — the database from Day 17 and the backup job that protects it both live in Git, both are reconciled by controllers, both audit cleanly. Operational tasks join the GitOps loop.

---

## What's next

[Day 20: DaemonSets — One Pod per Node, for Real Per-Node Concerns →](/articles/2026/06/02/day-20-daemonsets/)

On Day 20 you will meet the fourth workload type to round out the taxonomy: the **DaemonSet**. A Deployment runs N replicas wherever the scheduler likes; a DaemonSet runs **exactly one Pod on every node that matches a selector**. You will deploy a simple `node-info` agent as a DaemonSet, watch a new Pod appear automatically when you add a node (`kind create cluster --add-node`), and walk through the real-world DaemonSets already running on your cluster — kindnet, kube-proxy, and the Day 9 Promtail log shipper. Plus a careful look at **taints, tolerations, and `nodeSelector`** so a DaemonSet that should not run on the control-plane stays off it.
