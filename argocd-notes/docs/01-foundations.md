# Argo CD Foundations: GitOps, Push vs Pull, and Core Function

## Why Argo CD? The Push vs Pull dilemma

To understand why Argo CD is the standard for Kubernetes deployments, look at the problem it solves. Traditional CI/CD pipelines deploy using a **Push** model. Argo CD flips this and uses a **Pull** model, which is the core of "GitOps."

### The old way: traditional pipelines (Push)

In a standard setup (Jenkins, GitHub Actions), the pipeline builds a container image, creates the Kubernetes manifests, and then runs `kubectl apply` to push changes to the cluster.

Problems with this:

- **Security risk:** the pipeline needs full admin credentials to your Kubernetes cluster. If the CI/CD server is compromised, the entire infrastructure is compromised.
- **Configuration drift:** if an engineer manually changes a setting (scaling a deployment from 3 to 5 pods), the pipeline has no idea. Git says 3, the cluster says 5. The pipeline is blind to reality.

### The Argo CD way: GitOps (Pull)

Argo CD lives inside your Kubernetes cluster. Instead of a pipeline pushing to the cluster, Argo CD constantly watches a Git repository. When you deploy, the pipeline simply updates files in Git. Argo CD sees Git has changed, pulls those changes, and applies them locally. The pipeline never touches the cluster.

Why this is a major upgrade:

- **Ironclad security:** the CI pipeline only needs permission to write to Git. Cluster credentials never leave the cluster.
- **Self-healing:** Argo CD constantly compares the cluster to Git. If someone manually changes a pod, Argo CD flags it as "Out of Sync" (drift) and can be configured to instantly overwrite the manual change, forcing the cluster to match Git exactly.
- **Git as the ultimate audit log:** to know exactly what is running in production, you look at the Git repository. To roll back, you click "Revert" in Git and Argo CD mirrors that old state in the cluster.

### Push vs Pull at a glance

| Feature | Traditional CI/CD (Push) | Argo CD (Pull) |
|---|---|---|
| Cluster access | CI server holds admin keys | CI server has no cluster access |
| Source of truth | Split between Git and cluster | Git is the absolute authority |
| Manual edits | Ignored by the pipeline | Detected and automatically reverted |

---

## Core function: state reconciliation

At its core, Argo CD does exactly one thing: **state reconciliation**. It is an endless loop asking one question: *"Does the reality of the Kubernetes cluster perfectly match the blueprint in Git?"* If yes, it sleeps for a few seconds. If no, it takes action to make reality match the blueprint.

Think of a thermostat. You set it to 72 degrees (your Git repository). The room is currently 78 degrees (your Kubernetes cluster). The thermostat (Argo CD) detects the difference and turns on the AC until the room hits 72.

In infrastructure automation like Terraform or Ansible, you trigger a process to compare code against reality and then apply changes. Argo CD is that same concept, but running continuously on an infinite loop, specifically for Kubernetes resources.

### The three engines

**1. The Repository Server (the reader).** Its only job is to talk to your Git repository. It clones your code and caches the files so it doesn't hammer your Git provider. It doesn't just read plain YAML — if you use Helm or Kustomize, it renders them on the fly, translating templates into raw Kubernetes configurations.

**2. The Application Controller (the brain).** This is the heart of Argo CD. It constantly compares the **desired state** (from the Repository Server) with the **live state** (what is actually running in the cluster). When it detects drift, it flags the application. With Auto-Sync on, it immediately applies the changes to fix the drift.

**3. The API Server (the bouncer and dashboard).** This is what you interact with. It powers the Web UI and the CLI. It handles security: when you log in via SSO, the API Server checks your credentials and decides what you are allowed to see and do (RBAC).

### Sync vs Health: two separate metrics

Argo CD judges applications on two completely separate metrics.

| Metric | What it means | Common statuses |
|---|---|---|
| Sync status | Do the files in Git match the configuration in the cluster? | Synced, OutOfSync |
| Health status | Is the application actually running properly in Kubernetes? | Healthy, Progressing, Degraded |

**Example:** you push a typo in your container image tag (`image: my-app:v1.0.typo`). Argo CD pulls that YAML and applies it. Your **Sync status** becomes Synced (Argo CD successfully updated the cluster to match Git), but your **Health status** becomes Degraded (Kubernetes cannot download the image and your pods crash).

---

## Anatomy of a traditional Push model

Tracing a single change from a developer's laptop to a live cluster. In a Push model, the central brain is your CI/CD server, acting as an external orchestrator that commands both your code repository and your infrastructure.

1. **The trigger (code commit):** a developer merges code into main. Git sends a webhook to the CI/CD server: "New code is here, start the pipeline."
2. **Continuous integration (the build):** the CI/CD server allocates a temporary worker, pulls the code, installs dependencies, and runs tests.
3. **Packaging the artifact:** once tests pass, the pipeline packages the app into a container image and pushes it to an image registry (AWS ECR, Docker Hub).
4. **The push (continuous deployment):** the CI/CD server pulls the Kubernetes manifests, uses stored admin credentials to log directly into your cluster from outside, and runs `kubectl apply`.

### Structural weaknesses

- **The "God mode" credential problem:** for the final step, the CI/CD server must hold the master keys to your cluster. If breached, an attacker gains full control over production infrastructure.
- **The fire-and-forget blind spot:** once the pipeline pushes in step 4, the job finishes and goes to sleep. It assumes reality matches the code. If a node crashes or a teammate deletes a service, the pipeline has no idea.

### Limitations of the Push model

**1. Security risk ("God-mode" credentials).** The CI/CD server needs high-level admin credentials (a kubeconfig or powerful Service Account token). If compromised, an attacker has the keys to the kingdom. You also have to open a firewall hole to let the external CI server talk to your internal Kubernetes API.

**2. Blind to reality (configuration drift).** A push pipeline only runs when code is committed, then sleeps. If someone manually deletes a deployment or an autoscaler changes pod counts, the pipeline can't see it and can't fix it.

**3. Fire-and-forget deployments.** When a push pipeline runs `kubectl apply`, it throws the configuration over the wall. As long as Kubernetes accepts the YAML, the pipeline marks the deployment "Successful." But accepting the files doesn't mean the app started. Your app might crash-loop while the pipeline reports success. You're forced to write custom scripts just to verify pod health.

**4. The multi-cluster nightmare.** Pushing to one cluster is easy. Pushing to 20–50 clusters is a disaster: the pipeline must manage 50 sets of credentials, 50 API endpoints, and massive loops. If one cluster's API is down, the pipeline fails and you must untangle which clusters were updated.

---

## The Argo CD Pull workflow

In this model, Argo CD acts as the internal orchestrator, tucked safely inside your Kubernetes cluster. CI (Continuous Integration) and CD (Continuous Deployment) are completely separated.

1. **The build (CI):** a developer merges code. The CI server runs tests, builds the container image, and pushes it to a registry. **Crucial difference:** the CI pipeline stops here. It does not talk to Kubernetes.
2. **Updating the blueprint (the Git push):** instead of pushing to the cluster, the CI pipeline commits a small text update to your Git repository — e.g., "Update deployment.yaml to use image v2.0 instead of v1.0."
3. **The observation (Argo CD wakes up):** Argo CD, inside your cluster, is constantly monitoring that Git repository (polling every 3 minutes or listening for a webhook). It sees the new commit.
4. **State reconciliation (the pull):** Argo CD's Application Controller compares the new blueprint in Git (v2.0) against the live cluster (v1.0). It detects drift. Because it lives inside the cluster, it uses local, highly restricted permissions to apply the new configuration.
5. **Health verification (the feedback loop):** Argo CD doesn't fire-and-forget. It watches the Kubernetes API as new pods spin up, waits for them to pass health checks, and only then marks the application Healthy and Synced.

### How the Pull model solves the Push nightmares

- **Zero "God-mode" credentials:** your CI server never gets Kubernetes credentials. If breached, the worst an attacker can do is mess with your Git repository — and Git's version history lets you revert.
- **Drift protection (self-healing):** the observation and reconciliation steps run on an infinite loop. If someone manually deletes a critical service at 2 AM, Argo CD notices the cluster no longer matches Git and instantly recreates the service.
- **Multi-cluster made easy:** with 50 clusters, you don't need a pipeline managing 50 connections. You install Argo CD on each cluster and point them all at the same Git repository. They independently pull the changes.

---

## Advantages of Argo CD

**1. Instant disaster recovery.** Your entire cluster state is saved in Git. If a cluster dies, you spin up a blank Kubernetes cluster, install Argo CD, and point it at your Git repository. Within minutes, Argo CD rebuilds your entire infrastructure exactly as it was.

**2. Effortless auditing and compliance.** Because Git is the source of truth, your Git commit history is your audit log. To know who changed the database password or scaled the web servers, you look at the Git pull request — it shows the author, timestamp, and peer approvals.

**3. Better developer experience.** Developers never touch `kubectl` or log into a cluster. Their entire workflow is opening a Git pull request. Once approved and merged, Argo CD handles the rest, lowering the barrier to deploying applications.

**4. Guaranteed environment consistency.** Argo CD enforces the Git state across all environments. Using templating tools (Helm or Kustomize) with Argo CD, you guarantee that staging is an exact replica of production, just with different variables applied — eliminating "snowflake" clusters and the "works in dev but breaks in prod" problem.

### Business impact

| Advantage | Technical reality | Business value |
|---|---|---|
| Security | CI servers hold no cluster credentials | Drastically reduced blast radius if hacked |
| Reliability | Auto-healing fixes drift instantly | Fewer outages caused by manual errors |
| Speed | PR-based deployments remove pipeline bottlenecks | Faster feature delivery |
