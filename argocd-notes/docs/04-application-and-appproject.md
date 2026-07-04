# The Core CRDs: Application & AppProject

Argo CD fundamentally revolves around two Kubernetes Custom Resource Definitions (CRDs): the **Application** and the **AppProject**. They are a symbiotic pair — one does the work, and the other builds the fence.

## The Application (the engine)

The Application CRD is the core unit of work — the blueprint that tells the controller exactly what to deploy, where, and how. It answers three mandatory questions:

- **The source (where is the code?):** Git repository URL, and the path/branch to look in.
- **The destination (where is it going?):** the target cluster and target namespace.
- **The engine strategy (how to deploy?):** use Helm, Kustomize, or raw YAML; auto-sync or wait for a human; prune old resources or not.

An Application has significant power. Left unchecked, a single Application file could tell Argo CD to wipe out the production database namespace. This is why the AppProject exists.

## The AppProject (the sandbox)

An AppProject is a strict security fence. Every Application must belong to an AppProject. By default, everything goes into the `default` project, which has zero restrictions. In a real setup you create specific projects (`project-dev-team-a`, `project-financial-services`) as isolated sandboxes.

The fences an AppProject can build:

- **Source restrictions:** "Applications in this project may only pull code from `github.com/my-trusted-company/*`."
- **Destination restrictions:** "Applications may only deploy to the staging and dev clusters. Block any attempt to deploy to prod."
- **Resource restrictions (blacklist/whitelist):** "Applications can create Pods and Services but are forbidden from creating ClusterRoles or modifying Namespaces."
- **RBAC:** "Only members of the 'Finance-Dev' group may click Sync for Applications in this project."

### The symbiosis in practice: multi-tenancy

Say three development teams share one Argo CD instance. You don't want Team A deleting Team B's applications, or anyone touching production without approval.

1. You create an AppProject named `team-a-sandbox`, restricted so it can only deploy to `team-a-namespace` in the Dev cluster.
2. Team A writes an Application trying to deploy to the production cluster.
3. **The rejection:** the controller reads the Application, sees it belongs to `team-a-sandbox`, checks the project's rules, and rejects the deployment.

The Application provides the thrust; the AppProject provides the steering and brakes. When an autonomous system (a chatbot or AI agent triggering deployments) generates an invalid request, an AppProject blocking it isn't a bug — it is the safety net preventing an automated script from wiping out production because of a typo.

---

## Anatomy of an Application

A standard, bare-bones Application YAML:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-ui
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-company/infrastructure.git
    targetRevision: main
    path: apps/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: prod-frontend
```

**1. `metadata` (who am I?)**
- `name: frontend-ui` — the name shown on the dashboard tile.
- `namespace: argocd` — **crucial rule:** the Application resource itself must always live in the same namespace where Argo CD is installed (usually `argocd`). This is a security measure so users can't create rogue apps in random namespaces. This is *not* where the pods will be deployed.

**2. `spec.project` (which sandbox?)**
- `project: default` — links the Application to an AppProject. `default` has no restrictions; `finance-team` would enforce that project's rules before allowing this Application to run.

**3. `spec.source` (where is the code?)**
- `repoURL:` — the full Git HTTPS or SSH URL.
- `targetRevision:` — a branch name (`main`, `dev`), a specific commit hash (`a1b2c3d`), or a Git tag (`v1.2.0`). Branches are fine for Dev, but for Production you should lock this to a specific tag or commit hash to prevent accidental deployments if someone pushes a bad change to main.
- `path:` — the specific folder inside the repo where the manifests or Helm charts live.

**4. `spec.destination` (where does it run?)**
- `server:` — the API address of the target cluster. For the same cluster Argo CD lives in, always use the internal address `https://kubernetes.default.svc`. For a remote cluster, use its external API URL.
- `namespace:` — the actual namespace in the target cluster where pods, services, and configmaps are created.

### The missing piece: syncPolicy

If you deploy the YAML above, the application appears in the UI but does nothing — it waits for a human to click Sync. To make it true GitOps, add a `syncPolicy`:

```yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- `automated:` — Argo CD constantly applies changes automatically.
- `prune: true` — (the dangerous one) deletes objects in the cluster if they are removed from Git.
- `selfHeal: true` — instantly overwrites any manual `kubectl edit` changes an engineer makes in the cluster.

---

## Anatomy of an AppProject

The AppProject is the guardrail that keeps the engine from driving off a cliff. Its purpose is **zero-trust GitOps**: by default it denies everything, and you must explicitly whitelist what is allowed.

The four boundaries it builds:

- **Source repos (where can code come from?):** prevents deploying untrusted code. If an app tries to pull from a personal GitHub repo instead of the corporate repo, it is blocked.
- **Destinations (where can code go?):** prevents the Dev team from accidentally deploying to Prod. Defines allowed target clusters and namespaces.
- **Resources (what can be created?):** prevents applications from creating dangerous Kubernetes objects. A web app should create Pods and Services, never a ClusterRoleBinding.
- **Access (who can click buttons?):** maps to your SSO. Defines which user groups may view, sync, or delete applications inside this project.

A full AppProject YAML:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-finance-sandbox
  namespace: argocd
spec:
  description: "Sandbox for the Finance Dev Team"

  # 1. The Source Whitelist
  sourceRepos:
  - "https://github.com/my-company/finance-*.git"

  # 2. The Destination Whitelist
  destinations:
  - namespace: "finance-dev"
    server: "https://kubernetes.default.svc"
  - namespace: "finance-staging"
    server: "https://external-staging-cluster.com"

  # 3. The Resource Whitelist / Blacklist
  clusterResourceWhitelist:
  - group: '*'
    kind: 'Namespace'
  namespaceResourceBlacklist:
  - group: 'rbac.authorization.k8s.io'
    kind: 'RoleBinding'
```

Dissecting the specs:

- `metadata.name` — the unique identifier. This is the exact name you put in the `spec.project` field of your Application to bind them together.
- `spec.description` — a human-readable note explaining who this project is for.
- `spec.sourceRepos` — an array of allowed Git URLs. You can use wildcards (`*`) to allow multiple repos under an organization.
- `spec.destinations` — the exact namespace and server combinations this project may touch. If an Application tries to deploy to a namespace not on this list, it is rejected.
- `spec.clusterResourceWhitelist` — cluster-scoped resources (Namespaces, Nodes) are dangerous, so by default an AppProject blocks all of them. If your app must create a namespace, whitelist it here.
- `spec.namespaceResourceBlacklist` — namespace-scoped resources (Pods, ConfigMaps) are allowed by default. Use this blacklist to explicitly block dangerous things like RoleBindings, preventing developers from granting themselves admin permissions.

---

## Significant risks and how to mitigate them

### 1. The blast radius risk (the accidental delete)

Because Git is the source of truth, whatever is in Git becomes reality. If an engineer accidentally deletes a folder in Git, Argo CD's reconciliation loop will dutifully wipe those applications off your production cluster. A single bad PR merge can delete 50 production databases and web servers in seconds.

Mitigations:

- **Disable Prune on Auto-Sync for critical apps:** enable Auto-Sync but leave Prune *disabled*. If a developer deletes the database manifest, Argo CD flags it as OutOfSync (alerting you) but does not delete the running database pod. A human must confirm.
- **Resource exclusions in the AppProject:** configure a resource blacklist that forbids Argo CD from ever deleting specific critical resources (PersistentVolumeClaims, Namespaces), regardless of Git.
- **The `Prune=false` annotation:** attach `argocd.argoproj.io/sync-options: Prune=false` to a specific critical resource. Even if the entire Application is deleted, Argo CD leaves that resource untouched.

### 2. The monorepo bottleneck

Most teams start by putting all manifests and Helm charts into a single Git repository (a monorepo). As the company grows, every commit wakes the repo-server. With 500 applications configured via complex Helm charts, the repo-server tries to re-render all of them:

- Your Git provider rate-limits you for too many API calls.
- The repo-server spikes CPU to 100% and crashes (OOM Kill).
- Deployments slow to a crawl (30 minutes instead of 30 seconds).

Mitigations:

- **The two-repo split:** never store application source code (Java, Python, Node.js) in the same repository as your Argo CD Kubernetes manifests.
- **Enable webhooks:** don't rely on 3-minute polling. Configure webhooks so Argo CD only wakes when a relevant change happens.
- **Scale the repo-server:** run multiple replicas with significant CPU and memory requests.
- **Use Kustomize for environment overrides:** instead of 50 massive Helm value files, use a base chart and Kustomize to inject only the small per-environment changes.

---

## Resource deletion and finalizers

By default, deleting an Application in Argo CD only deletes the tracking record. It leaves the actual Kubernetes Pods, Services, and Ingresses running — creating **orphaned resources**.

To trigger a proper cascading delete, use the finalizer:

```yaml
metadata:
  finalizers:
  - resources-finalizer.argocd.argoproj.io
```

- **Without the finalizer:** deleting an Application orphans its workloads. For an automated system spinning up on-demand infrastructure, this leaves expensive orphaned resources running indefinitely on every "cleanup."
- **With the finalizer:** deleting the Application guarantees a ruthless, cascading cleanup of every Kubernetes object it created.

The primary use case is **temporary or preview environments.** When a pull request is opened, automation creates an Application with the finalizer. When the PR is closed, the automation deletes the Application, and the finalizer guarantees a full teardown of every object tied to it.

### The two pillars summarized

- **Application (mechanism):** the "what" and "where." (Deploy this, here.)
- **AppProject (governance):** the "what is allowed." (These are the rules.)

If the Application code is the engine, the AppProject ensures it is driven correctly.

### Policy failure troubleshooting

Three common errors when an Application violates its AppProject boundaries:

- **Repo not permitted:** often happens when developers point Argo CD to a personal fork instead of the official corporate repo. Fix: add the exact Git URL to the project's `sourceRepos`.
- **Namespace/server not permitted:** the classic "Dev trying to push to Prod" error. Fix: add the target cluster and namespace pair to `destinations`.
- **Resource kind not permitted:** if a locked-down project encounters a new Kubernetes object (a Deployment from `apps/v1`, a DaemonSet), Argo CD blocks it. Fix: add that Group and Kind to the whitelist.
