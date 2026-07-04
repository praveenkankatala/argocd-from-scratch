# Argo CD Architecture & Internals

Argo CD is not one monolithic program. It is a collection of distinct microservices that work together. Understanding how they split the workload is the key to debugging failures and scaling Argo CD.

## The three core components

### 1. argocd-server (the front door and API)

A stateless service that provides the gRPC and REST APIs powering the Web UI and CLI.

- **Core function:** when you log into the UI, run an `argocd` command, or trigger a webhook, you are talking to the argocd-server.
- **Security:** it handles all SSO integrations (Okta, Google, Dex) and enforces RBAC policies. It decides whether a given user can sync an app in a given cluster.
- **What if it crashes?** Your engineers can't log in and the Web UI goes down, but your deployments will *not* stop. The other components run independently. Argo CD keeps syncing Git to your cluster in the background.

### 2. argocd-repo-server (the translator)

Git repositories only hold raw files; Kubernetes only understands pure YAML manifests. The repo-server bridges this gap.

- **Core function:** it reaches out to your Git repositories, clones the code, and reads the files.
- **The heavy lifter:** if you use Helm or Kustomize, the repo-server runs the commands (like `helm template`) to translate charts and variables into raw Kubernetes YAML on the fly.
- **Scaling:** in a large environment, the repo-server is the biggest resource consumer. Rendering hundreds of complex Helm charts requires significant CPU and memory. When performance slows, this is usually the component to scale up (more CPU) or scale out (more replicas). It relies heavily on an internal Redis cache to avoid re-rendering unchanged files.

### 3. argocd-application-controller (the brain)

The heart of GitOps — an endless loop that executes the actual reconciliation.

- **Core function:** it gets the desired state (from the repo-server) and the live state (from the Kubernetes API). If there is drift, it executes the `kubectl apply` logic to fix the cluster.
- **The multi-cluster hub:** if Argo CD manages 10 remote clusters, the application-controller holds the credentials for those clusters and reaches across the network to update them.
- **Sharding:** a single controller can manage a few hundred applications. With 5,000 applications across 50 clusters, a single controller chokes. You then enable **sharding** — multiple application-controller pods split the load (e.g., Controller A handles clusters 1–25, Controller B handles 26–50).

---

## The reconciliation loop

This 6-step sequence is the infinite loop that runs inside the application-controller.

**1. Watch application (the triggers).** Argo CD is event-driven. It sits in a "watch" state waiting for a reason to act, listening for two triggers:
- A **Git event:** a webhook from GitHub telling it new code was merged. Without webhooks, Argo CD defaults to polling Git every 3 minutes.
- A **Kubernetes event:** a notification from the Kubernetes API that something in the cluster changed (a pod crashed, a ConfigMap was edited).

**2. Fetch desired state (the blueprint).** The controller asks the repo-server to fetch the latest code from Git. If you use Helm, this is the moment Argo CD runs the equivalent of `helm template`, injecting variables and outputting the final raw Kubernetes YAML — the desired state.

**3. Fetch live state (the reality).** The controller queries the Kubernetes API for what is actually running. Querying for thousands of resources every 3 minutes would crash the cluster, so Argo CD maintains a highly optimized, real-time cache of the cluster's state in its own memory and usually checks that cache.

**4. Compare and diff (the brain).** Argo CD overlays the desired state onto the live state to see if they match. It is smart enough to ignore certain fields — for example, Kubernetes automatically injects default annotations into running pods; Argo CD knows these aren't in Git and ignores them during the diff to avoid false alarms.

**5. Update status (Synced/OutOfSync).** Based on the diff:
- Match perfectly → status becomes **Synced**.
- Difference → status becomes **OutOfSync**.

If Auto-Sync is enabled, Argo CD doesn't just update the status — it immediately executes the equivalent of `kubectl apply` to force the cluster to match Git, then checks that the new pods actually started (updating the Health status).

**6. Repeat.** Once Synced and Healthy, the controller returns to step 1 and sleeps until the next webhook or 3-minute polling interval.

---

## Tracing a manual sync operation

When you click "Sync" in the UI or run `argocd app sync`, you force the system into immediate action. Tracing this path tells you exactly which component to investigate if a sync fails.

1. **Request (the entry point):** your click or command sends an HTTP/gRPC request to the **argocd-server**, carrying the message "Force an immediate sync for Application X."

2. **Validate (the security check):** the argocd-server checks your SSO identity and RBAC rules — "Is this user allowed to trigger a deployment for this application in this target cluster?" If yes, the request is approved.

3. **Execution (the handoff):** the argocd-server doesn't sync itself. It updates the application's record, setting a flag that says "Sync Requested." The **application-controller** sees this flag, drops what it's doing, and processes this application immediately, bypassing its normal 3-minute wait.

4. **Manifest generation (blueprint translation):** the application-controller asks the **repo-server** for the exact configuration from Git. The repo-server pulls the latest commit. If you use Helm or Kustomize, this is the heavy-lifting phase — it processes charts, injects variables, and generates the final raw Kubernetes YAML.

5. **Sync plan (the strategy):** the application-controller now has the raw YAML (desired state). It queries the Kubernetes API for the live state, compares the two, and creates a strict sync plan — e.g., "Create 1 new Service, update 1 Deployment, delete 1 old ConfigMap." This is also where Argo CD evaluates **sync waves and hooks** to decide the exact order of changes.

6. **Apply to cluster (the push):** the application-controller executes the plan, sending the equivalent of `kubectl apply`, `create`, and `delete` directly to the target Kubernetes API server.

7. **Monitor health (verification):** Argo CD doesn't assume success. It actively watches the resources it applied — new pods spinning up, passing readiness probes — and only when everything is running does it change the Health status to Healthy and mark the sync complete.

---

## gRPC, High Availability, and Leader Election

Moving Argo CD from a single-node setup to an enterprise-grade highly available system requires understanding how its components communicate and survive failures.

### The protocol: why gRPC?

When you run an `argocd` CLI command or when components talk to each other, they use **gRPC** (gRPC Remote Procedure Calls), not standard HTTP REST.

Why it matters: standard REST APIs send data as plain text (JSON). In a massive cluster with thousands of objects, sending all that YAML as text is slow and bandwidth-heavy. gRPC uses **Protocol Buffers (Protobuf)**:

- **Binary, not text:** data is compressed into an efficient binary format before sending — like sending a ZIP file instead of a giant text document.
- **Streaming:** gRPC runs on HTTP/2, allowing continuous two-way streaming. When you watch a deployment happen live, logs and status updates stream to you in real time rather than your browser constantly polling "Is it done yet?"

gRPC makes Argo CD fast enough for massive workloads without lagging.

### High Availability: scaling the components

To make Argo CD highly available, you run multiple replicas of its components. But you cannot treat all components the same way.

| Component | State type | HA strategy |
|---|---|---|
| API Server (argocd-server) | Stateless | Horizontal scaling: run 3+ copies behind a load balancer. Any request can go to any pod. |
| Repo Server (argocd-repo-server) | Stateless (mostly) | Horizontal scaling: run 3+ copies. Uses a shared Redis cache to store cloned Git data. |
| App Controller (application-controller) | Stateful | Leader Election: you cannot run 3 copies all working at once. |

**The problem with the application-controller:** it is the brain making changes to your cluster. If you run three copies and they all detect an OutOfSync application at the same millisecond, they will all try to run `kubectl apply` simultaneously, tripping over each other, causing race conditions, and potentially corrupting the cluster. You need backups in case the brain crashes, but only one brain can be in charge at a time.

### The solution: Leader Election

When you configure the application-controller for HA, you spin up multiple replicas (say 3). They figure out who is in charge:

1. **The race for the lock:** all pods race to grab a specific "lock" in the cluster (a Kubernetes Lease Object).
2. **The Leader:** the first pod to grab the lock wins. It becomes the **Leader** — the only one actively watching Git and pushing changes.
3. **The Standbys:** the others see the lock is taken and become **Standbys**, sitting idle, watching the lock.
4. **The heartbeat:** the Leader continuously renews its lease every few seconds to prove it's alive.
5. **The failover:** if the Leader crashes, the heartbeat stops and the lease expires. The Standbys race to grab it; whoever wins becomes the new Leader and resumes sync operations seamlessly.
