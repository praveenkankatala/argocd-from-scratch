# Using Argo CD: The Dashboard & CLI

## Part 1: The Dashboard

### The main dashboard (the fleet view)

When you log in, this is your high-level view. If you manage 50 microservices, this screen tells you what is on fire and what is stable.

- **Projects:** a "Project" is a security boundary. It restricts what an Application is allowed to do. A "Dev Project" might only allow deployments to a Dev cluster, while a "Prod Project" restricts deployments to Prod.
- **Application / Application name:** each tile represents one GitOps loop (e.g., "Frontend-UI" or "Payments-API").
- **Health status (high-level):** a heart icon — Green (Healthy), Yellow (Progressing/Updating), or Red (Degraded/Crashing).
- **Sync status (high-level):** a checkmark or alert icon telling you if the cluster currently matches Git.

### Application view (the command center)

When you click a specific Application tile, you enter its detailed view.

**Application status:**
- **Health status:** the physical state of the Kubernetes pods.
- **Sync status:** the state of the configuration (Git vs cluster).
- Note: an app can be **Synced** (Argo CD successfully applied the YAML) but **Degraded** (the pod is crashing because of a bad database password).

**Actions (the top bar):**
- **SYNC:** the "make it happen" button. Forces the Application Controller to apply the Git state to the cluster.
- **REFRESH:** often misunderstood. Clicking Refresh tells the repo-server to instantly look at Git and bypass its 3-minute wait. It does **not** change the cluster — it just updates Argo CD's internal view of whether Git has changed.
- **DELETE:** removes the application from Argo CD. **Warning:** by default this triggers a cascading delete, which also wipes out all the actual resources (pods, services) in your cluster.
- **APP DETAILS / HISTORY & ROLLBACK:** the History tab shows a timeline of every Git commit ever synced. The Rollback button reverts the cluster to a previous Git commit if a new deployment breaks production.

### The Inspector (the X-ray view)

Below the top bar is a visual web connecting your Git repo to your pods. Clicking any resource (a Pod or Service) opens a side panel — the Inspector.

- **SUMMARY:** quick metadata (labels, annotations, which image tag is currently running).
- **MANIFEST:** the exact raw Kubernetes YAML currently running in the cluster for that object.
- **DIFF:** the most important tab for debugging. If your app is OutOfSync, the Diff tab shows a side-by-side comparison (like a GitHub PR). The left side shows the Live State in the cluster; the right side shows the Desired State in Git, highlighting exactly what differs.
- **EVENTS:** streams native Kubernetes events for that object. If a pod fails to start, this tells you why (e.g., `Back-off pulling image "my-app:v2"`).
- **LOGS:** streams real-time application logs directly from the running container, so you don't need `kubectl logs`.
- **PARAMETERS:** if you use Helm, this shows all the Helm values injected into the template to build this deployment.

---

## Part 2: The CLI

The Web UI is great for visualizing deployments, but the CLI is where automation and fast debugging happen. When you need to automate a task, debug a broken deployment, or register 50 clusters, clicking through a UI is too slow.

### Phase 1: Connection (authentication)

**Standard login:**
```bash
argocd login argocd.local
```
The CLI reaches out over Port 443 using gRPC. If it hits your GRPCRoute successfully, it asks for your username and password, then saves a JWT (session token) locally.

**SSO login:**
```bash
argocd login argocd.local --sso
```
Opens your web browser, routes you to Okta/Google to authenticate, and passes the token back to your terminal.

**Firewall bypass (grpc-web):**
```bash
argocd login argocd.local --grpc-web
```
If your corporate firewall blocks HTTP/2 (gRPC) traffic, this flag wraps the CLI's binary commands inside standard HTTP/1.1 requests so they pass through.

### Phase 2: Plumbing (clusters and repos)

Repositories and clusters are stored as Kubernetes Secrets under the hood. Writing those Secrets manually is painful because you must base64-encode TLS certificates and SSH keys. The CLI does this for you.

**Connecting a Git repository:**
```bash
argocd repo add git@github.com:my-company/infrastructure.git --ssh-private-key-path ~/.ssh/id_rsa
```
Reads your local SSH key, encrypts it, sends it to Argo CD, and generates the correctly formatted Kubernetes Secret for the repo-server.

**Registering a remote cluster:**
```bash
argocd cluster add my-staging-cluster-context
```
The CLI looks at your local kubeconfig, reaches into the remote cluster, creates an Argo CD ServiceAccount, generates an RBAC token, brings that token back, and stores it securely so the application-controller can deploy there.

### Phase 3: Deployment (app management)

**Creating an Application:**
```bash
argocd app create my-frontend \
  --repo https://github.com/my-company/frontend.git \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev
```
Generates and applies the Application YAML.

**Forcing a sync:**
```bash
argocd app sync my-frontend
```
Bypasses the 3-minute polling wait and forces the Application Controller to immediately apply the Git state.

**Diff (X-ray vision):**
```bash
argocd app diff my-frontend
```
If your app is OutOfSync, prints a color-coded, side-by-side terminal output showing exactly what changed between Git and the live cluster (e.g., "Git says 3 replicas, cluster says 5").

### Phase 4: Emergency response (troubleshooting)

**Checking the heartbeat:**
```bash
argocd app get my-frontend
```
Prints a dashboard in your terminal showing the exact sync status, health status, and every Kubernetes resource (pods, services) attached to that app.

**The audit trail:**
```bash
argocd app history my-frontend
```
Lists every Git commit ever successfully deployed to this cluster, including who authored the commit and when it was synced.

**Rollback (the panic button):**
```bash
argocd app rollback my-frontend <HISTORY_ID>
```
If a new deployment breaks the database, run `history`, find the ID of the last working deployment, and run `rollback`. Argo CD instantly reverts the cluster to that previous state, ignoring what is currently in the main Git branch, until you can fix the code.
