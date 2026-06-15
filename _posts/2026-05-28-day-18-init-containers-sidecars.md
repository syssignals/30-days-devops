---
title: "Day 18: Init Containers and Sidecars — Multi-Container Pod Patterns"
date: 2026-05-28
categories: [Kubernetes]
tags: [kubernetes, init-containers, sidecar, multi-container-pod, jobs, devops]
excerpt: "A Pod can hold more than one container, and the order they start in is a feature, not an accident. Init containers run once and must finish before the app starts — perfect for setup and dependency gating. Sidecars run for the whole Pod lifetime — log shippers, metric exporters, mesh proxies. Day 18 adds both to the webapp, then shows the native sidecar pattern (an init container with restartPolicy: Always, beta-on-by-default since Kubernetes 1.29) that finally fixes the long-standing 'sidecar keeps a Job from ever completing' bug."
header:
  overlay_color: "#0d1117"
  overlay_filter: 0.6
author_profile: false
read_time: true
toc: true
toc_sticky: true
toc_label: "On this page"
---

> **30 Days of DevOps** — Day 18 of 30. [← Day 17: StatefulSets and Persistent Volumes](/articles/2026/05/27/day-17-statefulsets-persistent-volumes/)

Every Pod in this series so far has held exactly one container: nginx, or Postgres. But a Pod is not "a container with extra steps" — it is a **shared execution context** for one or more containers that share a network namespace, can share volumes, and are scheduled and torn down together as a unit. The moment you put a second container in a Pod, you unlock the most flexible composition primitive Kubernetes has.

Two patterns dominate, and the difference between them is **lifecycle**:

- **Init containers** run **once**, in order, **before** the app containers start, and each must exit `0` before the next begins. They are for setup that must complete before the workload is allowed to run: rendering a config file, running a database migration, waiting for a dependency to become reachable, fetching a secret or an artifact. If an init container fails, the Pod restarts and tries again — the app never sees a half-prepared environment.

- **Sidecar containers** run **alongside** the app container for the Pod's entire lifetime. They are for continuous companion work: shipping logs, exporting metrics, refreshing credentials, proxying traffic (the service-mesh model). The classic way to write one — just add another entry to `spec.containers` — has two long-standing problems: there is **no ordering guarantee** (the sidecar might not be ready when the app starts), and in a **Job**, a sidecar that never exits keeps the Pod from ever reaching `Completed`, so the Job hangs forever.

Kubernetes 1.29 made the fix **beta and on by default**: a **native sidecar** is an init container with `restartPolicy: Always`. It starts in the init sequence (so it is up *before* the app), it keeps running for the Pod's life (unlike a normal init container), and the kubelet terminates it automatically *after* the app containers exit — which means Jobs with sidecars finally complete. Today you will use all three.

## What you will build

By the end of this article you will have:

- An **init container** added to the webapp Pod that renders an `index.html` into a shared `emptyDir` volume once, before nginx starts — and exits
- A **native sidecar** (`restartPolicy: Always` init container) that writes a refreshing `health.txt` into the same shared volume every 15 seconds, for the Pod's whole lifetime
- The nginx container serving both files from the shared volume — so `curl https://webapp.local/` shows the init-rendered page and `curl https://webapp.local/health.txt` shows a timestamp that advances every 15 seconds (proving the sidecar is alive)
- All of it committed to the `gitops-webapp` chart and synced by Argo CD, with every extra container satisfying the **PSS restricted** profile from Day 14
- A side-by-side demonstration of the **Job + sidecar problem**: a Job with a *traditional* sidecar that hangs at `0/1` forever, and the same Job with a *native* sidecar that completes cleanly — the single clearest reason native sidecars exist

---

## How a multi-container Pod starts and stops

The order is the whole point. This is the lifecycle of the webapp Pod after today's changes.

```mermaid
%%{init: {'theme': 'dark'}}%%
flowchart LR
    START["Pod scheduled"]:::phase

    INIT["init-content\n(ordinary init container)\nrender index.html → exit 0"]:::init

    SIDE["clock-sidecar\n(init container +\nrestartPolicy: Always)\nstarts, stays running"]:::side

    MAIN["nginx\n(main container)\nstarts, serves /shared"]:::main

    RUN["Pod Running\nnginx + clock-sidecar\nboth alive (READY 2/2)"]:::run

    TERM["Pod terminating\nnginx stops first,\nTHEN clock-sidecar stops"]:::term

    START -->|"1 · init phase begins"| INIT
    INIT -->|"2 · exits 0, next starts"| SIDE
    SIDE -->|"3 · reaches Started,\n   main phase begins"| MAIN
    MAIN -->|"4 · all containers up"| RUN
    RUN -->|"5 · SIGTERM /\n   delete"| TERM

    classDef phase fill:#1c2128,stroke:#30363d,color:#8b949e
    classDef init fill:#2a1f10,stroke:#d29922,color:#e6edf3
    classDef side fill:#2d1a2d,stroke:#bc8cff,color:#e6edf3
    classDef main fill:#1d2d1d,stroke:#3fb950,color:#e6edf3
    classDef run fill:#1a2744,stroke:#58a6ff,color:#e6edf3
    classDef term fill:#2d1a1a,stroke:#f85149,color:#e6edf3
```

**Reading this diagram:**

Read left to right — the horizontal axis is time, from Pod scheduling to Pod termination. The five numbered arrows are the lifecycle transitions; the colour of each box tells you what kind of container it is.

**Arrow 1 — the init phase begins.** Once the scheduler places the Pod on a node, the kubelet starts the **init containers in array order**, one at a time. `init-content` (amber — an *ordinary* init container) runs first. It renders `index.html` into the shared volume and exits `0`. If it exited non-zero, the kubelet would restart it; the main containers would never start until it succeeds.

**Arrow 2 — the next init container starts.** Because `init-content` exited cleanly, the kubelet moves to the next entry in `initContainers`: `clock-sidecar` (purple — a *native sidecar*, distinguished by `restartPolicy: Always`). Here the rules change. An ordinary init container must *exit* before the next one starts; a native sidecar only needs to *start* (reach its Started state). The kubelet does not wait for it to finish — it never finishes.

**Arrow 3 — the main phase begins.** Once `clock-sidecar` has *started*, the kubelet launches the **main containers** in `spec.containers`: `nginx` (green). This ordering is the guarantee a plain `spec.containers` sidecar cannot give you — with a native sidecar, the companion is provably up before the app. nginx mounts the same shared volume and immediately finds the `index.html` that `init-content` wrote and the `health.txt` that `clock-sidecar` is already maintaining.

**Arrow 4 — the Pod is Running.** Now nginx and clock-sidecar are both alive. The Pod's `READY` column shows `2/2`: two long-lived containers, both Ready. Notice `init-content` is *not* counted — it completed during the init phase and is gone. Native sidecars count toward the Pod's container total; ordinary init containers do not.

**Arrow 5 — termination, in reverse.** When the Pod is deleted or rolled, the kubelet stops the **main containers first** (`nginx`), and only *then* stops the native sidecars (`clock-sidecar`). This reverse order is deliberate: a log-shipper sidecar should keep running long enough to flush the app's final log lines; a mesh-proxy sidecar should outlive the app so the app's last outbound requests still have a route. A plain `spec.containers` sidecar gets no such ordering — everything is torn down together.

The key insight: **`restartPolicy: Always` on an init container is the single field that converts "run once and exit" into "start early, run forever, stop last."** That one line is what makes Jobs-with-sidecars work, which the last part of this article demonstrates directly.

---

## Prerequisites

This article continues from Day 17. Required state:

- The `devops-cluster` kind cluster running **Kubernetes 1.29 or newer** — native sidecars (the `SidecarContainers` feature) are beta and on by default from 1.29; on an older cluster the `restartPolicy` field inside `initContainers` is silently ignored
- Argo CD managing the `gitops-webapp` repo; the webapp Deployment hardened to PSS `restricted` (Day 14) and running under the HPA (Day 12)
- The `default` namespace still labelled `pod-security.kubernetes.io/enforce: restricted` — every container we add must comply

Pre-flight check:

```bash
# Native sidecars need server version >= 1.29. Check the SERVER line.
kubectl version | grep -i server

# Confirm the webapp is healthy before we change its Pod shape
kubectl get application -n argocd webapp
```

Expected output:

```text
Server Version: v1.29.4

NAME     SYNC STATUS   HEALTH STATUS
webapp   Synced        Healthy
```

If your server version is below 1.29, the native-sidecar parts of this article will still apply YAML cleanly but the sidecar will behave like a normal init container (block forever, never letting nginx start). Upgrade the cluster or recreate it with a newer kind node image.

| Tool | Minimum version | Check |
|---|---|---|
| kubectl | 1.29 | `kubectl version --client` |
| Helm | 3.14 | `helm version --short` |
| gh CLI | 2.x | `gh --version` |
| Kubernetes (server) | **1.29** | `kubectl version` |

---

## Part 1 — Add the init container and native sidecar to the chart

All changes are in `webapp/templates/deployment.yaml` in the `gitops-webapp` repo. We add an `initContainers` block, a new shared `emptyDir` volume, and a mount of that volume into nginx.

```bash
cd ~/30-days-devops/day-12/gitops-webapp
```

### 1.1 — Rewrite `deployment.yaml` with the init container, sidecar, and shared volume

Rewrite `webapp/templates/deployment.yaml` with the full file below. Compared to your Day 14
version it adds three things: an **`initContainers:`** block (the `init-content` init container
and the `clock-sidecar` native sidecar), a **`shared-content` `volumeMount`** on nginx at its
web root, and a **`shared-content` `emptyDir`** volume:

```bash
cat > webapp/templates/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {% raw %}{{ include "webapp.fullname" . }}{% endraw %}
  labels:
    {% raw %}{{- include "webapp.labels" . | nindent 4 }}{% endraw %}
spec:
  {% raw %}{{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}{% endraw %}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: {% raw %}{{ .Values.rollingUpdate.maxSurge }}{% endraw %}
      maxUnavailable: {% raw %}{{ .Values.rollingUpdate.maxUnavailable }}{% endraw %}
  selector:
    matchLabels:
      {% raw %}{{- include "webapp.selectorLabels" . | nindent 6 }}{% endraw %}
  template:
    metadata:
      labels:
        {% raw %}{{- include "webapp.selectorLabels" . | nindent 8 }}{% endraw %}
    spec:
      serviceAccountName: webapp-runtime
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        fsGroup: 101
        seccompProfile:
          type: RuntimeDefault
      initContainers:
        # Ordinary init container: runs once, must exit 0 before anything else
        # starts. Renders the page nginx serves, into the shared volume.
        - name: init-content
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              cat > /shared/index.html <<HTML
              <!doctype html><html><body>
              <h1>{% raw %}{{ .Chart.Name }}{% endraw %}</h1>
              <p>Rendered by the init container at $(date -u +%FT%TZ)</p>
              </body></html>
              HTML
              echo "init-content: wrote /shared/index.html"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: shared-content
              mountPath: /shared
        # Native sidecar: an init container with restartPolicy: Always. It
        # STARTS during the init phase (up before nginx) but never exits —
        # the kubelet treats it as a long-lived sidecar.
        - name: clock-sidecar
          image: busybox:1.36
          restartPolicy: Always          # <-- the one line that makes it a sidecar
          command:
            - sh
            - -c
            - |
              while true; do
                echo "ok $(date -u +%FT%TZ)" > /shared/health.txt
                sleep 15
              done
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: shared-content
              mountPath: /shared
      containers:
        - name: {% raw %}{{ .Chart.Name }}{% endraw %}
          image: "{% raw %}{{ .Values.image.repository }}:{{ .Values.image.tag }}{% endraw %}"
          imagePullPolicy: {% raw %}{{ .Values.image.pullPolicy }}{% endraw %}
          ports:
            - name: http
              containerPort: {% raw %}{{ .Values.service.targetPort }}{% endraw %}
              protocol: TCP
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: {% raw %}{{ .Values.probes.readiness.initialDelaySeconds }}{% endraw %}
            periodSeconds: {% raw %}{{ .Values.probes.readiness.periodSeconds }}{% endraw %}
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: {% raw %}{{ .Values.probes.liveness.initialDelaySeconds }}{% endraw %}
            periodSeconds: {% raw %}{{ .Values.probes.liveness.periodSeconds }}{% endraw %}
          resources:
            {% raw %}{{- toYaml .Values.resources | nindent 12 }}{% endraw %}
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: shared-content
              mountPath: /usr/share/nginx/html
          envFrom:
            - secretRef:
                name: webapp-secret
      volumes:
        - name: tmp
          emptyDir: {}
        - name: shared-content
          emptyDir: {}
EOF
```

Both extra containers carry the same `securityContext` the main container got on Day 14 — `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: ["ALL"]` — because the `default` namespace enforces PSS `restricted` and **every container in the Pod is checked, init and sidecar included**. They inherit `runAsNonRoot`, `runAsUser: 101`, and the `seccompProfile` from the Pod-level `securityContext`. Writing to `/shared` works despite `readOnlyRootFilesystem: true` because `/shared` is a mounted `emptyDir`, not part of the read-only root filesystem.

### 1.2 — How the shared volume is wired

The full file above already does this — no separate edit needed. Two of its lines carry the
shared volume: nginx mounts `shared-content` at `/usr/share/nginx/html` (its web root, alongside
the `/tmp` mount from Day 14), and the Pod declares a second `emptyDir` named `shared-content`
next to `tmp`.

Mounting an `emptyDir` at `/usr/share/nginx/html` hides the image's built-in default page and serves whatever the init container and sidecar write there instead. The volume is shared by all three containers — init-content writes it, clock-sidecar keeps writing to it, nginx reads from it. `emptyDir` lives as long as the Pod and is the canonical way to pass files between containers in the same Pod.

### 1.3 — Render locally before committing

```bash
helm template webapp ./webapp -f webapp/values-dev.yaml \
  | grep -A 40 'kind: Deployment' | grep -E 'initContainers|name:|restartPolicy|mountPath'
```

Expected output (abbreviated — confirm the structure is present):

```text
      initContainers:
        - name: init-content
          volumeMounts:
            - name: shared-content
              mountPath: /shared
        - name: clock-sidecar
          restartPolicy: Always
          volumeMounts:
            - name: shared-content
              mountPath: /shared
        - name: webapp
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: shared-content
              mountPath: /usr/share/nginx/html
```

`restartPolicy: Always` appearing **inside an `initContainers` entry** is the native-sidecar marker. If you see it there, the render is correct.

---

## Part 2 — Commit, sync, and watch the startup order

```bash
git add webapp/templates/deployment.yaml
git commit -m "feat: add init container + native sidecar to webapp Pod"
git push origin main

argocd app sync webapp --server argocd.local --insecure
```

Watch the new Pod come up. The `kubectl get pod` output during rollout shows the init phase explicitly:

```bash
kubectl get pod -n default -l app.kubernetes.io/instance=webapp --watch
```

Expected output (one Pod's progression):

```text
NAME                             READY   STATUS     RESTARTS   AGE
webapp-webapp-6b9f8c7d4-x7k2p    0/2     Init:0/2   0          2s
webapp-webapp-6b9f8c7d4-x7k2p    0/2     Init:1/2   0          4s
webapp-webapp-6b9f8c7d4-x7k2p    0/2     PodInitializing   0   6s
webapp-webapp-6b9f8c7d4-x7k2p    2/2     Running    0          9s
```

Read the STATUS column top to bottom:

- **`Init:0/2`** — the kubelet is running the first init container (`init-content`). The denominator is `2` because the total counts **every** entry in `initContainers`, and the native sidecar `clock-sidecar` lives there too.
- **`Init:1/2`** — `init-content` finished (the `1`); now the kubelet has started `clock-sidecar`. A native sidecar counts as "done" for init *progression* once it has **started** — it never exits, so the kubelet does not wait for completion before advancing.
- **`PodInitializing`** — init sequence satisfied, main containers starting.
- **`2/2 Running`** — nginx and clock-sidecar both Ready. The `2` is the two long-lived containers; `init-content` completed during the init phase and is not counted in the running total.

Confirm the container roster:

```bash
POD=$(kubectl get pod -n default -l app.kubernetes.io/instance=webapp \
  -o jsonpath='{.items[0].metadata.name}')

# init containers (includes the native sidecar — it lives in initContainers)
kubectl get pod -n default "$POD" -o jsonpath='{.spec.initContainers[*].name}{"\n"}'
# main containers
kubectl get pod -n default "$POD" -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

Expected output:

```text
init-content clock-sidecar
webapp
```

`clock-sidecar` lives in `initContainers` (that is where native sidecars are declared) but behaves like a long-running container — the best of both.

---

## Part 3 — Verify both containers' output through nginx

The whole point of the shared volume is that nginx serves what the other two containers produced. Check both files over the ingress:

```bash
# The init container rendered this once, at Pod start
curl -ks https://webapp.local/ | sed 's/<[^>]*>//g' | grep -v '^$'
```

Expected output:

```text
webapp
Rendered by the init container at 2026-05-28T09:00:14Z
```

```bash
# The sidecar refreshes this every 15s. Hit it twice, 16s apart.
curl -ks https://webapp.local/health.txt
sleep 16
curl -ks https://webapp.local/health.txt
```

Expected output:

```text
ok 2026-05-28T09:05:01Z
ok 2026-05-28T09:05:16Z
```

The timestamp **advanced** between the two requests — proof the sidecar is alive and looping, not a one-shot init container. The init-rendered `index.html`, by contrast, keeps its original Pod-start timestamp no matter how many times you fetch it, because `init-content` ran exactly once and exited.

Confirm the sidecar's run-once vs run-forever distinction from the Pod's status:

```bash
kubectl get pod -n default "$POD" \
  -o jsonpath='{range .status.initContainerStatuses[*]}{.name}{": started="}{.started}{" ready="}{.ready}{"\n"}{end}'
```

Expected output:

```text
init-content: started=false ready=false
clock-sidecar: started=true ready=true
```

`init-content` shows `started=false` because it has terminated (it is no longer a running process). `clock-sidecar` shows `started=true ready=true` — the kubelet tracks it as a live, ready container even though it lives in `initContainers`.

---

## Part 4 — The Job + sidecar problem (why native sidecars exist)

This is the demonstration that justifies the whole feature. A **Job** runs a Pod to completion: the Pod's containers do their work, exit `0`, and the Job records a success. But a *traditional* sidecar never exits — so the Pod never completes, and the Job hangs at `0/1` forever.

Create the two Jobs in the `database` namespace (no PSS enforcement there, so the manifests stay short — the comparison is about lifecycle, not security context):

```bash
mkdir -p ~/30-days-devops/day-18 && cd ~/30-days-devops/day-18

cat > jobs-compare.yaml << 'EOF'
# Job A — TRADITIONAL sidecar (an entry in spec.containers).
# The worker exits after 5s, but the sidecar's `sleep 3600` keeps the
# Pod alive, so the Job never reaches Completed.
apiVersion: batch/v1
kind: Job
metadata:
  name: job-traditional-sidecar
  namespace: database
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo worker doing work; sleep 5; echo worker done"]
        - name: sidecar
          image: busybox:1.36
          command: ["sh", "-c", "echo sidecar up; sleep 3600"]
---
# Job B — NATIVE sidecar (an initContainers entry + restartPolicy: Always).
# The worker exits after 5s; the kubelet then terminates the native
# sidecar automatically, and the Job completes.
apiVersion: batch/v1
kind: Job
metadata:
  name: job-native-sidecar
  namespace: database
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      initContainers:
        - name: sidecar
          image: busybox:1.36
          restartPolicy: Always       # native sidecar
          command: ["sh", "-c", "echo sidecar up; sleep 3600"]
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo worker doing work; sleep 5; echo worker done"]
EOF

kubectl apply -f jobs-compare.yaml
```

Expected output:

```text
job.batch/job-traditional-sidecar created
job.batch/job-native-sidecar created
```

Wait ~30 seconds, then compare:

```bash
kubectl get jobs -n database
```

Expected output:

```text
NAME                       STATUS     COMPLETIONS   DURATION   AGE
job-native-sidecar         Complete   1/1           8s         30s
job-traditional-sidecar    Running    0/1           30s        30s
```

The contrast is the entire lesson:

- **`job-native-sidecar`** completed in ~8 seconds. The worker exited `0`, the kubelet saw the only *main* container finish, terminated the native sidecar, and the Pod reached `Completed` — so the Job is `1/1 Complete`.
- **`job-traditional-sidecar`** is still `Running` at `0/1` and will stay that way until its `sleep 3600` elapses (or you delete it). The worker finished long ago, but the sidecar in `spec.containers` is still sleeping, the Pod cannot complete, and the Job is stuck.

Confirm the stuck Pod is alive only because of the sidecar:

```bash
kubectl get pod -n database -l job-name=job-traditional-sidecar
```

Expected output:

```text
NAME                            READY   STATUS    RESTARTS   AGE
job-traditional-sidecar-abcde   1/2     Running   0          45s
```

`1/2 Running` — the Pod phase is still `Running` (the sidecar keeps it alive), and `1/2` means one of its two containers is Ready: the sidecar. The worker has terminated, so it is no longer counted as Ready. Before native sidecars, the only workarounds were ugly: a shared `emptyDir` flag file the sidecar polls to know when to exit, a `preStop` hook race, or process-sharing hacks. Native sidecars make all of that unnecessary.

Clean up:

```bash
kubectl delete -f jobs-compare.yaml
```

Expected output:

```text
job.batch "job-traditional-sidecar" deleted
job.batch "job-native-sidecar" deleted
```

---

## Common Errors

**1. The Pod hangs in `Init:0/1` forever**

```bash
kubectl get pod -n default <pod>
# webapp-webapp-...   0/2   Init:0/1   0   3m
```

The first init container is stuck — it never exited `0`. Either its command is failing, or (the classic mistake) you wrote a long-running process as an *ordinary* init container instead of a native sidecar, so the kubelet waits forever for it to finish.

Fix: look at the init container's logs explicitly with `-c`:

```bash
kubectl logs -n default <pod> -c init-content
# If this is supposed to be a sidecar, add `restartPolicy: Always` to its
# initContainers entry so the kubelet treats it as start-and-continue.
```

**2. Native sidecar behaves like a blocking init container (Pod never reaches the main phase)**

The `restartPolicy: Always` was ignored because the cluster is older than 1.29 (the `SidecarContainers` feature gate is off), or the field was placed at the **Pod** level instead of inside the **initContainers entry**.

Fix:

```bash
kubectl version | grep -i server   # must be >= 1.29
# And confirm restartPolicy is INSIDE the initContainers list item, not in spec:
kubectl get pod -n default <pod> \
  -o jsonpath='{.spec.initContainers[?(@.name=="clock-sidecar")].restartPolicy}{"\n"}'
# Must print: Always
```

**3. `forbidden: violates PodSecurity "restricted"` on apply, naming the init or sidecar container**

```text
pods "webapp-..." is forbidden: violates PodSecurity "restricted:latest":
allowPrivilegeEscalation != false (container "init-content" must set ...)
```

PSS restricted (Day 14) checks **every** container in the Pod — including init containers and sidecars. A new container without the full security context is rejected, and the rejection names the offending container.

Fix: give each init/sidecar container the same container-level `securityContext` the main container has (`allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: ["ALL"]`). The Pod-level fields (`runAsNonRoot`, `runAsUser`, `seccompProfile`) are inherited and do not need repeating.

**4. Sidecar crash-loops with `can't create /shared/health.txt: Read-only file system`**

The sidecar has `readOnlyRootFilesystem: true` (correct, for PSS) but is trying to write somewhere that is *not* a mounted volume — for example a typo in the mount path, so `/shared` is part of the read-only root rather than the `emptyDir`.

Fix: confirm the `volumeMounts.mountPath` on the sidecar exactly matches the path its command writes to:

```bash
kubectl get pod -n default <pod> \
  -o jsonpath='{.spec.initContainers[?(@.name=="clock-sidecar")].volumeMounts[*].mountPath}{"\n"}'
# Must include /shared
```

**5. nginx serves the default page, not the init-rendered one**

`curl https://webapp.local/` shows the stock "Welcome to nginx!" page instead of the rendered content. The `shared-content` volume was not mounted at nginx's web root, so nginx is serving the image's built-in `/usr/share/nginx/html`.

Fix: confirm the nginx container mounts `shared-content` at `/usr/share/nginx/html` (not the init container's `/shared` — they are the same volume mounted at different paths in different containers):

```bash
kubectl get pod -n default <pod> \
  -o jsonpath='{.spec.containers[?(@.name=="webapp")].volumeMounts[*].mountPath}{"\n"}'
# Must include /usr/share/nginx/html
```

**6. Job with a traditional sidecar shows `1/2` and never completes — and you expected native-sidecar behaviour**

You added `restartPolicy: Always` but put the sidecar in `spec.containers` rather than `spec.initContainers`. `restartPolicy: Always` is only meaningful (and only allowed) on an *init container* entry; on a `spec.containers` entry it is not a valid field and the sidecar stays an ordinary main container that blocks Job completion.

Fix: move the sidecar definition from `spec.template.spec.containers` to `spec.template.spec.initContainers`, keeping `restartPolicy: Always` on it. Only the *worker* belongs in `containers`.

---

## Recap

In this article you:

- Learned the three multi-container roles by **lifecycle**: ordinary **init containers** (run once, in order, must exit `0` before the app starts), traditional **sidecars** (`spec.containers` entries that run alongside the app but with no ordering and no clean Job termination), and **native sidecars** (init containers with `restartPolicy: Always` — start early, run for the Pod's life, terminate after the main containers, **beta and on by default since Kubernetes 1.29, GA in 1.33**)
- Walked the full Pod startup/shutdown timeline with a Mermaid diagram: init phase → native sidecar starts → main phase → Running (`2/2`) → reverse-order termination
- Added an **`init-content` init container** that renders `index.html` into a shared `emptyDir` once, and a **`clock-sidecar` native sidecar** that refreshes `health.txt` every 15 seconds — both PSS-`restricted` compliant — and had nginx serve both from the shared volume
- Confirmed the distinction live: `index.html`'s timestamp is frozen (init ran once), `health.txt`'s timestamp advances every 15s (sidecar runs forever), and `initContainerStatuses` shows `init-content started=false` vs `clock-sidecar started=true ready=true`
- Demonstrated the **Job + sidecar problem** side by side: the traditional-sidecar Job stuck at `0/1 Running` with its Pod at `1/2`, the native-sidecar Job `1/1 Complete` in ~8 seconds — the concrete reason the native-sidecar feature was built
- Worked through six common errors: the `Init:0/1` hang, the silently-ignored `restartPolicy` on old clusters, PSS rejection naming an init/sidecar container, the read-only-filesystem write failure, the unmounted-web-root default page, and the `restartPolicy: Always`-in-the-wrong-place mistake

The webapp Pod is now a genuine multi-container unit, and you have the pattern that underpins service meshes, log/metric agents, and config reloaders across the whole ecosystem.

---

## What's next

[Day 19: Jobs and CronJobs — Batch Workloads, Parallelism, and Scheduled Backups →](/articles/2026/06/01/day-19-jobs-cronjobs/)

On Day 19 you will go deeper into the `Job` you met in Part 4. You will run a **parallel Job** with `completions` and `parallelism` to process a work queue, tune `backoffLimit` and `activeDeadlineSeconds` so a wedged Job fails fast instead of retrying forever, and learn how `restartPolicy: Never` vs `OnFailure` changes retry accounting. Then you will schedule a **CronJob** — a nightly `pg_dump` of the Postgres StatefulSet from Day 17 into a PersistentVolume — and use `concurrencyPolicy: Forbid`, `startingDeadlineSeconds`, and `successfulJobsHistoryLimit` to keep the schedule clean. Batch is where the init-container and native-sidecar patterns from today pay off most.
