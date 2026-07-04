# Sync, Diff & Drift

## The Sync operation in detail

The Sync operation is the heartbeat of GitOps. It takes the **desired state** (Git) and forces the **live state** (Kubernetes) to match it. At a beginner level it's a button; underneath it's a multi-stage orchestration engine.

### 1. The triggers (how a sync starts)

- **Manual trigger:** an engineer clicks Sync in the UI or runs `argocd app sync`.
- **Git mutation (Auto-Sync):** with automated sync on, the repo-server detects a new commit (usually via webhook), calculates that the cluster is now OutOfSync, and immediately triggers a sync.
- **Cluster mutation (Self-Heal):** with self-heal on, a rogue engineer runs `kubectl edit deployment` and changes a replica count from 3 to 5. Argo CD detects the drift and instantly triggers a sync to overwrite the change back to 3.

### 2. The sync options (the rulebook)

When a sync is triggered, it looks at the `syncPolicy` rules in your Application to know how aggressively to act.

- `Prune=true` — if a file was deleted from Git, delete the corresponding resource from the cluster. (If false, orphaned resources are left running.)
- `CreateNamespace=true` — if the target namespace doesn't exist, create it before deploying pods.
- `ServerSideApply=true` — by default Argo CD acts like a massive `kubectl apply`, but if your YAML files are too large (giant CRDs) it fails. Server-Side Apply pushes the calculation work to the Kubernetes API server, allowing massive deployments.

### 3. The orchestration (sync waves and hooks)

If you have a Database, a Secret, and a Web API, you cannot deploy them at the exact same millisecond — the Web API will crash if the Database isn't ready. Argo CD solves this using phases and waves.

**The 4 sync phases (chronological order).** You attach scripts (Kubernetes Jobs) to run at these moments using annotations (e.g., `argocd.argoproj.io/hook: PreSync`):

- **PreSync:** runs before any infrastructure is updated (run a database migration; back up a volume).
- **Sync:** the actual Kubernetes manifests are applied.
- **PostSync:** runs after the sync is 100% healthy (send a Slack notification; run integration tests).
- **SyncFail:** runs only if the sync crashes or times out (ping PagerDuty).

**Sync waves (the ordering engine).** Inside the Sync phase, Argo CD groups your YAML files into waves using an annotation: `argocd.argoproj.io/sync-wave: "1"`. Waves run from lowest to highest number (e.g., -5, 0, 1, 2, 10). The golden rule: Argo CD applies a wave and **pauses until every resource in it is Healthy** before beginning the next wave.

A sample wave strategy:

```
Wave -1 → Namespaces and ClusterRoles     (the foundation)
Wave  0 → ConfigMaps and Secrets          (the configuration)
Wave  1 → Databases and Redis caches       (the stateful backend)
Wave  2 → Backend APIs                      (they need the DB healthy first)
Wave  3 → Frontend UI                        (needs the API healthy first)
```

### 4. The execution (the apply)

Once everything is ordered into waves, the application-controller talks to the Kubernetes API. It looks at the live state, calculates a diff (patch), and sends a series of API requests to create, patch, or delete resources.

### 5. The health assessment (when is it done?)

A sync is not "complete" just because the YAML was sent successfully. Argo CD shifts into watch mode, monitoring the Kubernetes event stream. For a Deployment, it waits to see the new pods spin up, waits for readiness probes to pass, and waits for the old pods to terminate. Only when the cluster reports everything is operational does Argo CD mark the Application Healthy and close the sync.

Mastering the sync process — using hooks to run database migrations and waves to order your microservices — is what transforms a messy, crashing deployment into a fully automated rollout.

---

## Manual vs Automated Sync

The Sync Policy tells Argo CD what to do when it finds a difference between Git and the cluster.

### 1. Manual sync (the controlled gate)

With a manual policy, Argo CD acts purely as an observer and alerter. When it detects drift, the application turns yellow (`OutOfSync`) and then stops and waits. It takes zero action until a human clicks Sync or runs `argocd app sync`.

Capabilities of manual sync:

- **Selective sync:** you don't have to sync the whole app. If a PR changes a Deployment and a ConfigMap but you only want to test the ConfigMap first, you can sync only that specific resource.
- **Dry runs:** `argocd app sync --dry-run` asks the Kubernetes API "If I applied this, would you accept it or throw an error?" without changing the cluster.
- **The "stop the bleeding" pause:** if production is failing, you don't want an automated system constantly pushing new code while you debug. Manual sync gives you absolute control over the timeline.

Best for: production environments (unless you have mature automated testing), sensitive databases, and fragile legacy applications.

### 2. Automated sync (the autonomous engine)

With an automated policy, Argo CD acts as the enforcer. The moment it detects a difference, it immediately syncs to force the cluster to match Git. No human intervention.

Two critical flags:

1. `prune: true/false` (the deletion flag)
   - **True:** if a developer deletes `redis.yaml` from Git, Argo CD instantly deletes the Redis pods from the cluster.
   - **False:** Argo CD deploys and updates but refuses to delete anything. Orphaned resources are left running.
2. `selfHeal: true/false` (the anti-drift flag)
   - **True:** if a rogue engineer manually changes replicas from 3 to 10, Argo CD detects and overwrites it back to 3. Git is the absolute law.
   - **False:** Argo CD only pushes changes when Git is updated. It ignores manual tampering in the cluster.

Best for: Development, QA, and Staging where you want developers to see merged code running instantly; also used in mature Production environments that have automated integration tests gatekeeping the main Git branch.

### The hybrid strategy (sync windows)

Most enterprises use **Sync Windows** to combine both. For example, configure an Application to be strictly Manual during the day (so nobody accidentally breaks production during business hours), but automatically shift to Automated at 2 AM to roll out merged changes while traffic is low.

---

## Self-Heal in depth

Self-heal is the ultimate enforcer of GitOps. Turning it on is a declarative statement: "Git is the absolute, unquestionable law."

### How it works: the anti-drift engine

Normally Argo CD's automation is triggered by Git. With `selfHeal: true`, Argo CD also heavily scrutinizes the cluster. The application-controller actively watches the Kubernetes API for any events related to your app.

**Scenario:** at 2 AM an engineer bypasses Git, logs directly into the cluster, and runs `kubectl edit deployment my-web-app` to double the memory limits. Within milliseconds, Argo CD's controller sees the change, compares the live cluster to Git, realizes they no longer match (configuration drift), and — because self-heal is true — instantly overwrites the manual change, reverting the memory limits to whatever is in Git. If you want to change production, you must commit it to Git.

### The danger: the infinite sync loop (fighting controllers)

Self-heal has a famous blind spot: Kubernetes controllers.

Say you configure a Horizontal Pod Autoscaler (HPA) to scale from 3 pods to 10 based on CPU:

1. Traffic spikes. The HPA changes replicas to 5.
2. Argo CD looks at Git, which says `replicas: 3`.
3. Argo CD says "someone tampered with the cluster!" and self-heals back to 3.
4. The HPA immediately says "CPU is still too high, scaling to 5!"
5. Argo CD overwrites it back to 3.

Your cluster is now stuck in an infinite sync loop. The two controllers fight thousands of times a minute until the Kubernetes API server crashes from exhaustion.

### The fix: ignoreDifferences

Tell Argo CD to enforce Git as the law, *except* for specific fields managed by other systems:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-web-app
spec:
  # ... source and destination ...
  syncPolicy:
    automated:
      selfHeal: true

  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas   # ignore the replica count; let the HPA handle it
```

Combining `selfHeal: true` with `ignoreDifferences` gives you infrastructure that rejects human tampering but plays nicely with automated Kubernetes scaling.

---

## ignoreDifferences in detail

`ignoreDifferences` is the peacemaker between strict GitOps (Git is law) and the dynamic nature of Kubernetes (controllers constantly change things).

### The problem: the infinite sync loop

Modern clusters are full of automated tools that intentionally mutate your resources:

- **HPAs:** you define `replicas: 3`, but the HPA scales to 10.
- **Service meshes (Istio/Linkerd):** you deploy a pod with 1 container, but a mutating webhook injects a 2nd sidecar proxy.
- **Cloud providers:** AWS/GCP inject default storage classes or network annotations.

If self-heal is on, Argo CD overwrites these, the controller re-applies, and the fight never ends.

### The solution

Add `ignoreDifferences` to your Application. You are telling the diffing engine: "When comparing Git to the live cluster, completely ignore these specific fields. Pretend they match." Two targeting methods:

**1. The exact path (`jsonPointers`).** Use when you know the exact, static path to the field. Most common for `replicas`:

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

Argo CD enforces everything about the Deployment (images, ports, labels) but ignores the replica count.

**2. The smart filter (`jqPathExpressions`).** `jsonPointers` fail with arrays (lists), because the order of items might change. Use `jqPathExpressions` (based on the `jq` tool) to write a smart query:

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jqPathExpressions:
    - .spec.template.spec.containers[].env[] | select(.name == "INJECTED_DB_PASSWORD")
```

This tells Argo CD: "Look through all containers and their environment variables. If you see one named `INJECTED_DB_PASSWORD`, ignore it. But if someone manually adds a *different* environment variable that isn't in Git, delete it."

Without `ignoreDifferences`, true GitOps is nearly impossible in a complex environment. It lets you enforce strict Git control over what you care about (image tags, configuration) while giving Kubernetes freedom to automate what it cares about (scaling, sidecars).

---

## Reconciling a change: the lifecycle

When a change occurs — a developer commits new code or drift happens in the cluster — Argo CD passes through three milestones.

### Phase 1: Drift detection

Argo CD compares the target Kubernetes manifest (desired state) against its cache of the live cluster (actual state). If a single line differs (an image tag changes v1→v2, or a CPU limit is manually changed), it marks the application `OutOfSync`.

By default Argo CD polls Git every 3 minutes. In production, waiting 3 minutes is too slow, so advanced setups configure a **Git webhook** — the moment you merge code, Git fires a webhook, dropping detection time from minutes to milliseconds.

### Phase 2: Sync initiated

How this plays out depends on the sync policy:

- **Manual:** Argo CD turns red on the dashboard and waits. The sync happens only when a human clicks Sync.
- **Automated:** the moment drift is marked, the controller schedules a sync immediately.

When automation is enabled, two flags dictate behavior:
- **Prune:** if you delete a config file from Git, should Argo CD delete that resource from the cluster? If disabled, old resources run forever.
- **SelfHeal:** if someone manually edits the cluster, should Argo CD overwrite it? If enabled, manual changes are wiped.

### Phase 3: State reconciled

The application-controller applies the changes. Kubernetes processes the manifests, updates the workloads, and the dashboard status returns to `Synced`. But reaching Synced only means the configuration matches. The state is not truly reconciled until the application is also **Healthy** — Argo CD keeps watching until the pods pass readiness probes and serve traffic cleanly.

---

## Diff strategies

To decide if an app is OutOfSync, Argo CD calculates a diff between the desired state (Git) and live state (Kubernetes). By default it uses a client-side three-way merge, but at scale this often needs tuning.

### Strategy 1: Client-side diff (the default)

Argo CD downloads the live YAML from the Kubernetes API, downloads the desired YAML from Git, and compares them inside the application-controller pod.

**The problem — mutating admission webhooks.** If you use a service mesh like Istio, you deploy a standard Pod from Git, but Istio's webhook intercepts it and injects a large `istio-proxy` sidecar. Argo CD sees Git (1 container) and the cluster (2 containers), declares "Drift detected!", and tries to delete the sidecar. Istio re-injects it. You are in an infinite fight.

**The solution.** Argo CD handles most standard Kubernetes defaults automatically, but for custom webhooks and autoscalers you must explicitly instruct it using diff customizations.

### Strategy 2: Diff customization (ignoreDifferences)

The most powerful tool for managing drift. Tell the diffing engine to ignore specific lines in the live cluster. Configured in the Application YAML.

**Ignoring autoscalers (HPA):** if an HPA manages your replicas, Git should no longer dictate the replica count:

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

**Ignoring mutating webhooks (sidecars, injected env):** use JQ path expressions:

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jqPathExpressions:
    - .spec.template.spec.containers[].env[] | select(.name == "INJECTED_DB_PASSWORD")
```

### Strategy 3: Server-Side Apply (SSA)

Instead of the application-controller downloading the live state and doing the complex merge math itself (client-side), it pushes the YAML directly to the Kubernetes API server and says "You calculate the diff and tell me what needs to change."

Why enable it:

- **Massive CRDs:** deploying a huge Custom Resource (like a Prometheus rulebook) often crashes the default client-side diff because the annotation size exceeds Kubernetes' 262KB limit. SSA bypasses this entirely.
- **Field ownership:** SSA tracks exactly who owns which field ("managers"). It knows Argo CD owns the container image tag while the HPA controller owns the replicas. Because Kubernetes itself tracks this, diffing becomes far more accurate.

Enable it in syncOptions:

```yaml
spec:
  syncPolicy:
    syncOptions:
    - ServerSideApply=true
```

### Strategy 4: Resource tracking strategies

How does Argo CD know which resources in a massive cluster belong to your Application so it can diff them? It tracks them. You can change the method globally in the `argocd-cm` ConfigMap.

- **Label (the legacy way):** Argo CD attaches a label (`app.kubernetes.io/instance=my-app`) to every resource. The flaw: some Helm charts forbid changing their labels, so Argo CD fails to track them.
- **Annotation (the modern standard):** Argo CD injects an annotation (`argocd.argoproj.io/tracking-id`) instead. Almost no tools complain about extra annotations, making diffing far more reliable across complex third-party Helm charts.

---

## Orphaned resources: the ghosts in the cluster

An **orphaned resource** is a Kubernetes object (Pod, Service, Secret) that exists in the live cluster but does not exist in your Git repository. Since Git is the source of truth, an orphaned resource is undocumented, untracked infrastructure. In the UI these show a warning icon and can mark your application `OutOfSync`.

### How resources become orphaned

**Scenario 1: the "Prune = false" deletion.** Your repo contains `backend-api.yaml` and `redis-cache.yaml`. A developer deletes `redis-cache.yaml` from Git. Argo CD compares Git (1 file) to the cluster (2 resources). Because the default policy is `prune: false`, Argo CD refuses to delete the Redis pods — it leaves them running and labels them orphaned.

**Scenario 2: manual creation.** An engineer debugging in the `production-web` namespace runs `kubectl create configmap debug-config ...`, bypassing Git. Argo CD scans the namespace, sees the new ConfigMap, checks Git, doesn't find it, and flags it as orphaned.

### The dangers

- **Cost bleed:** if a developer deletes the YAML for an expensive Load Balancer or GPU node but `prune: false` leaves it running, you pay for infrastructure no one uses.
- **Routing nightmares:** if you rename a Deployment in Git (e.g., `api-v1` to `api-v2`) without pruning, both versions run. If your Service selector is too broad, it load-balances user traffic to both the new and the orphaned deprecated code randomly.
- **Security vulnerabilities:** orphaned pods are forgotten pods. They don't get updated when you patch base Docker images, becoming silent security holes inside your network.

### Orphaned resource monitoring at the AppProject level

By default, an Application only cares about resources it deployed. To lock down an entire namespace and know if anyone deploys something not in Git, enable orphaned resource monitoring on the AppProject:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: strict-production-project
spec:
  # ... other project rules ...

  orphanedResources:
    warn: true   # triggers a warning in the UI for the project
    ignore:
    # don't warn about default Kubernetes service accounts/tokens
    - group: ""
      kind: ServiceAccount
      name: default
```

When enabled, the AppProject acts like a security guard. If it sees a resource in the namespace without an Argo CD tracking label, it raises a global warning that someone is bypassing the GitOps pipeline.

### How to clean up orphaned resources

- **Manual (safe):** use the Web UI to find the orphaned resource, click the three dots, and Delete. Or use `kubectl delete` directly.
- **Automated (aggressive):** change the sync policy to `prune: true` and trigger a sync. Argo CD acts as a garbage collector and deletes every orphaned resource it was managing.
