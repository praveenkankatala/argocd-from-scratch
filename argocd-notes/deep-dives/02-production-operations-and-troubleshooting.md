# Production Operations & Troubleshooting

The final layer: running Argo CD itself like a production service. Custom health checks, scaling/sharding, disaster recovery, monitoring, notifications, and a real troubleshooting runbook.

---

# Part 1: Custom Health Checks (teach Argo CD to read health)

Argo CD knows how to assess health for built-in types (Deployment, StatefulSet, Service, Ingress, PVC, Job). For each it has logic like "a Deployment is Healthy when `availableReplicas == desired`."

**The problem:** the moment you deploy a **Custom Resource** (a `KafkaTopic`, a `Certificate` from cert-manager, a `Rollout`, a database `Cluster` CR), Argo CD has no idea how to read its health. It defaults to treating it as **Healthy immediately** — which silently breaks sync-wave ordering (Doc 01) and makes your sync status lie.

**The fix:** write a **custom Lua health check** in the `argocd-cm` ConfigMap. Argo CD embeds a Lua interpreter for exactly this.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations.health.cert-manager.io_Certificate: |
    hs = {}
    if obj.status ~= nil and obj.status.conditions ~= nil then
      for i, condition in ipairs(obj.status.conditions) do
        if condition.type == "Ready" and condition.status == "True" then
          hs.status = "Healthy"
          hs.message = "Certificate is issued"
          return hs
        end
        if condition.type == "Ready" and condition.status == "False" then
          hs.status = "Degraded"
          hs.message = condition.message
          return hs
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for certificate to be issued"
    return hs
```

The key format is `resource.customizations.health.<group>_<Kind>`. The Lua script receives the live object as `obj` and must return `hs.status` (one of `Healthy`, `Progressing`, `Degraded`, `Suspended`) plus a message. Now a sync-wave containing a `Certificate` will actually **wait** until the cert is issued before advancing — restoring the ordering guarantee.

**Related customizations in `argocd-cm`:**
- `resource.customizations.ignoreDifferences.<group>_<Kind>` — global ignore rules (vs per-Application).
- `resource.customizations.actions.<group>_<Kind>` — define custom UI actions (e.g., a "restart" button on a Rollout).
- `resource.exclusions` — tell Argo CD to completely ignore certain resource types cluster-wide to save CPU (e.g., high-churn `Events`, `EndpointSlices`).

---

# Part 2: Scaling & Sharding

Your notes covered HA and leader election for the controller. Here's how to scale each component under real load.

## The repo-server (usually your bottleneck)

Rendering Helm/Kustomize is CPU-heavy. Symptoms of an overloaded repo-server: slow syncs, `OOMKilled` restarts, Git provider rate-limiting.

```yaml
# scale out AND up
replicas: 5
resources:
  requests: { cpu: "1", memory: "1Gi" }
  limits:   { cpu: "2", memory: "2Gi" }
```

Tuning levers:
- `--parallelismlimit` — cap concurrent manifest generations per repo-server.
- Increase the **Redis** cache size — repo-server leans on it to avoid re-rendering unchanged apps.
- Enable manifest-generate-paths annotation (from your monorepo notes) so a commit only re-renders affected apps.

## The application-controller (sharding)

A single controller handles a few hundred apps fine. Past that — thousands of apps across dozens of clusters — you **shard**: run multiple controller replicas, each owning a subset of clusters.

```yaml
# argocd-application-controller StatefulSet
env:
- name: ARGOCD_CONTROLLER_REPLICAS
  value: "3"                          # 3 shards
spec:
  replicas: 3
```

Sharding strategies (set via `--sharding-method` or `argocd-cmd-params-cm`):
- **`legacy`** — hash cluster ID modulo replica count. Simple but uneven.
- **`round-robin`** — more even distribution across shards.
- **dynamic / consistent-hashing** (newer versions) — rebalances more gracefully when shards are added/removed.

Each shard owns whole clusters (not individual apps), so a cluster's apps are always handled by one controller. Watch the `argocd_cluster_connection_status` and per-shard CPU to confirm even distribution.

## The API server

Stateless — just scale horizontally behind your Gateway/LB:
```yaml
replicas: 3
```

---

# Part 3: Disaster Recovery & Backup

Argo CD's own state (Applications, AppProjects, repo/cluster secrets, RBAC) lives in the cluster. If the management cluster dies, you need it back.

## The export/import tooling

```bash
# back up EVERYTHING Argo CD knows (apps, projects, config, secrets)
argocd admin export -n argocd > argocd-backup.yaml

# restore into a fresh Argo CD install
argocd admin import -n argocd - < argocd-backup.yaml
```

`argocd admin export` dumps all Argo CD CRDs and config into one YAML. Store it encrypted (it contains secret material) in object storage, ideally on a schedule (CronJob).

## The purist DR answer: it's all in Git anyway

The beautiful part of GitOps: your **Applications and AppProjects should themselves be defined in Git** (via App-of-Apps or ApplicationSets). So true DR is:

1. Spin up a blank cluster.
2. Install Argo CD (Terraform / Helm).
3. Bootstrap secrets (ESO/Terraform).
4. `kubectl apply` your App-of-Apps root.
5. Argo CD rebuilds the entire fleet from Git.

The `admin export` backup is your safety net for the *imperative* bits (locally-created apps, cluster credentials, admin config) that didn't make it into Git. **In a mature setup, you barely need it** — which is itself a sign your GitOps is healthy.

## What to actually back up

- **Git repos** (obviously — but mirror them; don't depend on a single GitHub org).
- **The external secret store** (AWS Secrets Manager, Vault) — separate backup lifecycle.
- **`argocd admin export`** on a schedule for cluster credentials & imperative state.
- **The Sealed Secrets controller private key**, if you use Sealed Secrets — losing it means losing every sealed secret.

---

# Part 4: Monitoring Argo CD with Prometheus

Argo CD exposes Prometheus metrics on every component. Scrape them and alert.

## Key metrics to watch

| Metric | Why it matters |
|---|---|
| `argocd_app_info` | Inventory of apps with sync_status & health_status labels |
| `argocd_app_sync_total` | Sync counts by phase — spot rising failures |
| `argocd_app_reconcile` (histogram) | Reconciliation duration — rising = controller under strain |
| `argocd_git_request_total` | Git API calls — spot rate-limit risk / missing webhooks |
| `argocd_cluster_connection_status` | Is each managed cluster reachable? |
| `argocd_redis_request_total` | Cache health |
| `workqueue_depth` | Controller backlog — sustained growth means you need to shard |

## Alerts worth having

- **App stuck OutOfSync > 15m** — `argocd_app_info{sync_status="OutOfSync"}`.
- **App Degraded** — `argocd_app_info{health_status="Degraded"}`.
- **Sync failure rate rising** — rate of `argocd_app_sync_total{phase="Failed"}`.
- **Controller reconcile latency p99 climbing** — capacity signal.
- **Cluster unreachable** — `argocd_cluster_connection_status == 0`.

There's an official Grafana dashboard (ID 14584) that visualizes most of these out of the box.

---

# Part 5: Notifications

Humans can't stare at the UI. `argocd-notifications` (built into modern Argo CD) pushes events to Slack/Teams/PagerDuty/webhooks. Two concepts: **triggers** (when to fire) and **templates** (what to say).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token          # references a key in argocd-notifications-secret
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]
  template.app-sync-failed: |
    message: |
      🔴 Sync failed for {{.app.metadata.name}}
      Cluster: {{.app.spec.destination.server}}
      https://argocd.mycompany.com/applications/{{.app.metadata.name}}
```

Subscribe an app to notifications via annotation:
```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: my-team-channel
```

**SRE pattern:** wire `on-sync-failed` and `on-health-degraded` to PagerDuty for prod-project apps, and `on-sync-succeeded` to a low-noise Slack channel for visibility. This closes the loop with Argo Rollouts (Doc 04) — a canary auto-abort fires `on-health-degraded`.

---

# Part 6: The Troubleshooting Runbook

When something breaks, work the components in order. The whole point of understanding the architecture is knowing *which* component to blame.

## Symptom: App stuck `OutOfSync`, won't self-heal

1. `argocd app diff <app>` — see exactly what differs.
2. If the diff shows fields you *don't* control (replicas, injected sidecar) → a controller/webhook is fighting you → add `ignoreDifferences` (you know this).
3. If diff shows nothing but status is still OutOfSync → `argocd app get <app> --refresh` (forces repo-server to re-read Git, bypassing cache).
4. Check the **application-controller logs** for reconcile errors.

## Symptom: `ComparisonError` / manifests won't render

Almost always the **repo-server**.
- `x509: certificate signed by unknown authority` → missing CA in `argocd-tls-certs-cm` (your notes covered this).
- `authentication required` → bad/expired repo credential secret.
- Helm/Kustomize render error → check repo-server logs; reproduce locally with `helm template` / `kustomize build`.
- `rpc error: code = ResourceExhausted` → repo-server OOM/CPU starved → scale it (Part 2).

## Symptom: Sync succeeds but app is `Degraded`

The config applied fine but the app isn't running.
1. Open the resource tree → find the red resource.
2. **Events tab** → e.g. `Back-off pulling image` (bad tag), `CrashLoopBackOff` (app bug), `FailedScheduling` (no node capacity).
3. **Logs tab** → the container's stdout/stderr.
4. If it's a CRD showing Degraded unexpectedly → you may be **missing a custom health check** (Part 1) or one is misfiring.

## Symptom: Sync hangs forever on a Hook or Wave

- A **PreSync hook Job** is failing/stuck → `kubectl logs job/<hook>`; the sync won't advance until it succeeds.
- A **sync-wave** is blocked because a resource never reaches Healthy → check that resource's health; if it's a CRD, suspect a missing/incorrect Lua health check letting an *earlier* wave falsely report progressing/degraded.

## Symptom: CLI works but UI won't load (or vice versa)

Networking — the gRPC vs HTTP split you already mastered.
- UI loads, CLI fails → gRPC path broken (GRPCRoute / backend-protocol / try `--grpc-web`).
- Both flaky → Gateway/Ingress protocol config; check `server.insecure` matches your TLS-termination setup.

## Symptom: Everything red after a controller restart / high latency

- Check `workqueue_depth` and reconcile latency (Part 4). Sustained backlog → **shard the controller** (Part 2).
- Check `argocd_cluster_connection_status` — a dead managed cluster makes its apps all report Unknown.

## The four-command triage kit

```bash
argocd app get <app>            # health, sync, resource tree, conditions
argocd app diff <app>           # exact desired-vs-live differences
argocd app history <app>        # what deployed when, by whom (rollback target)
argocd app get <app> --refresh  # force repo-server to re-read Git now
```

---

## SRE takeaways

- **Custom Lua health checks** are not optional once you use CRDs — without them, waves and sync status lie.
- **repo-server is your usual bottleneck**; controller **sharding** is the answer at thousands of apps.
- **True DR is your Git repo**; `argocd admin export` is the safety net for imperative state.
- **Scrape the Prometheus metrics and alert on OutOfSync/Degraded/queue-depth.**
- **Notifications close the loop** — route prod failures to PagerDuty.
- **Troubleshoot by component:** OutOfSync→controller/diff, render errors→repo-server, Degraded→the workload, gRPC weirdness→networking.

**Lab to lock it in:** Write a Lua health check for a cert-manager `Certificate`, put it in a sync-wave, and confirm the next wave waits for issuance. Then enable controller sharding with 2 replicas and watch clusters distribute. Finally, `argocd admin export`, delete an app, and `import` it back.
