# Sync Waves & Hooks — The Orchestration Engine

Your notes referenced this topic at least eight times but never delivered it. This is the single most important "advanced" concept in Argo CD, because it's the difference between a deploy that comes up cleanly and one that crash-loops for ten minutes.

## The core problem: Argo CD applies everything at once

By default, when Argo CD runs a sync, it takes every manifest it rendered and applies them to the Kubernetes API **in one big batch**. Kubernetes then schedules everything in parallel.

For a simple stateless app, that's fine. But imagine a realistic stack:

- A **Namespace**
- A **Secret** holding the DB password
- A **PostgreSQL** StatefulSet
- A **Backend API** that connects to Postgres on startup
- A **Frontend** that calls the Backend

If Argo CD applies all of these simultaneously, the Backend API pod starts *before* Postgres is accepting connections. It fails its startup connection, crashes, and enters `CrashLoopBackOff`. Kubernetes will *eventually* recover it once Postgres is up, but you'll stare at red pods for minutes and your Argo CD app will flap between `Progressing` and `Degraded`.

Argo CD gives you two tools to impose order: **Sync Waves** (ordering) and **Hooks** (lifecycle scripting).

---

## Part 1: Sync Waves — the ordering mechanism

A sync wave is just an integer annotation you attach to any resource:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

### The rules of waves

1. Waves run from **lowest number to highest** (negative numbers are allowed and run first: `-10`, `-5`, `0`, `1`, `5`).
2. The **default wave is 0** for any resource without the annotation.
3. **Argo CD applies all resources in a wave, then WAITS.** It will not start the next wave until every resource in the current wave is **Healthy**.
4. Within a single wave, Argo CD still applies a sensible default order (namespaces before the things inside them, CRDs before custom resources, etc.).

That "waits until Healthy" behavior is the whole point. It's a barrier.

### A real SRE wave strategy

```
Wave -2 → Namespaces, ResourceQuotas, LimitRanges       (the container)
Wave -1 → ServiceAccounts, Roles, RoleBindings, CRDs     (permissions & schema)
Wave  0 → ConfigMaps, Secrets                            (configuration)
Wave  1 → Databases, Redis, Kafka (stateful backends)    (data layer)
Wave  2 → Backend APIs (need the DB healthy first)       (app layer)
Wave  3 → Frontend / Ingress / Gateway routes            (edge layer)
```

Concretely, on your Postgres StatefulSet:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

And on your Backend:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

Now Argo CD applies Postgres (wave 1), waits for it to report Healthy, *then* applies the Backend (wave 2). The crash-loop is gone.

### The gotcha: "Healthy" depends on health checks

Wave progression is gated on the **health status** of the resources in the wave. For built-in types (Deployment, StatefulSet, Service) Argo CD knows how to assess health. But if your wave contains a **custom resource** (say, a `KafkaTopic` from the Strimzi operator), Argo CD doesn't know how to tell if it's healthy — so it treats it as healthy *immediately* and moves on, defeating the barrier. This is exactly why **custom Lua health checks** (Doc 06) matter: without them, your waves lie to you.

---

## Part 2: Hooks — scripting the lifecycle

Waves order your *infrastructure*. Hooks let you run **one-off tasks** (usually Kubernetes `Job`s) at precise moments around the sync. Think database migrations, backups, smoke tests, cache warming, Slack pings.

A hook is a resource (almost always a `Job`, but can be a `Pod` or anything) tagged with a hook annotation:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
```

### The 5 hook phases (chronological)

| Phase | When it runs | Classic use case |
|---|---|---|
| **PreSync** | Before any manifests are applied | Database schema migration; backup a volume |
| **Sync** | Interleaved *with* the normal sync (respects waves) | Complex ordered custom logic |
| **Skip** | Not a timing phase — tells Argo CD to **not apply** this resource | Keep a manifest in Git for reference without deploying it |
| **PostSync** | After all resources are applied **and Healthy** | Integration/smoke tests; cache warm; Slack "deploy succeeded" |
| **SyncFail** | Only if the sync fails or times out | PagerDuty alert; automatic cleanup/rollback of partial state |

### A PreSync database migration hook (the canonical example)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: my-company/db-migrator:v2.1.0
        command: ["flyway", "migrate"]
```

The sequence: Argo CD sees the sync is triggered → runs this Job → **waits for it to succeed** → only then applies your actual Deployments. If the migration Job fails, the entire sync aborts and your app is never deployed against an un-migrated schema. That's a huge safety win.

### Hook deletion policies (avoid Job clutter)

Every hook Job leaves a completed pod behind unless you tell Argo CD to clean it up. That's what `hook-delete-policy` controls:

| Policy | Behavior |
|---|---|
| `HookSucceeded` | Delete the hook right after it succeeds (most common for migrations) |
| `HookFailed` | Delete the hook if it fails |
| `BeforeHookCreation` | **Default.** Delete the *previous* instance of this hook right before creating a new one. Keeps the last run around for debugging. |

For a migration you usually want `HookSucceeded` (clean on success) combined with the default, so a *failed* migration Job sticks around for you to inspect the logs.

```yaml
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

### Hooks + Waves together

Hooks respect sync-wave annotations too. So you can order your hooks relative to each other:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "-1"   # run this migration before other PreSync hooks
```

---

## Part 3: Argo CD Hooks vs Helm Hooks (the collision)

This is a subtle production trap you'll hit if you deploy Helm charts (which you already covered).

Helm has its *own* hook system (`helm.sh/hook: pre-install`). When Argo CD renders a Helm chart, it **automatically translates** most Helm hooks into Argo CD hooks:

| Helm hook | Becomes Argo CD hook |
|---|---|
| `pre-install`, `pre-upgrade` | `PreSync` |
| `post-install`, `post-upgrade` | `PostSync` |
| `post-delete` | (no equivalent — can be lost) |

The gotcha: Argo CD runs `helm template`, not `helm install`, so it never sees Helm's release lifecycle. Complex Helm hooks that depend on `install` vs `upgrade` distinctions can behave unexpectedly. When a third-party chart misbehaves on sync, the first thing to check is whether its Helm hooks translated cleanly — sometimes you must fork the chart and re-annotate with native `argocd.argoproj.io/hook`.

---

## Part 4: Selective sync & the `Sync` phase resources

Two more annotations worth knowing:

**Prevent a resource from being pruned even during App deletion:**
```yaml
argocd.argoproj.io/sync-options: Prune=false
```

**Skip a resource entirely on apply** (Argo CD will diff it but never `kubectl apply` it):
```yaml
argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
```

**Force replace instead of apply** (needed for immutable fields or huge CRDs):
```yaml
argocd.argoproj.io/sync-options: Replace=true
```

---

## SRE takeaways

- **Waves = ordering.** Group your stack into layers; each wave is a "wait until healthy" barrier.
- **Hooks = lifecycle scripting.** PreSync for migrations, PostSync for tests, SyncFail for alerts.
- **Health checks gate everything.** A wave over a CRD with no custom health check silently breaks the ordering guarantee.
- **Always set a hook-delete-policy** or your cluster fills with completed Job pods.
- **Watch the Helm hook translation** when a chart deploys strangely.

**Lab to lock it in:** Deploy Postgres + API with no waves and watch the API crash. Add waves 1 and 2 and watch it come up clean. Then add a PreSync migration hook that intentionally `exit 1`s, and confirm the whole sync aborts before your app ever deploys.
