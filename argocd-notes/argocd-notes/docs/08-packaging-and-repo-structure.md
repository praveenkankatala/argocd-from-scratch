# Packaging & Repository Structure

## Part 1: Deploying Helm Charts with Argo CD

Argo CD has native, first-class support for Helm. It automatically detects if a directory contains a `Chart.yaml` file and processes it. Because Argo CD is a GitOps tool, there are two distinct ways to deploy a Helm chart.

### Method 1: The remote registry approach (3rd-party apps)

The most common way to deploy off-the-shelf software like Redis, NGINX, or Prometheus. Instead of downloading the chart into your own Git repo, you tell Argo CD to pull it directly from the official public Helm repository and pass your custom values in the Application YAML.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-ingress
  namespace: argocd
spec:
  project: default
  source:
    # 1. Point directly to the official Helm repository URL
    repoURL: 'https://charts.bitnami.com/bitnami'
    # 2. Specify the exact chart name
    chart: nginx
    # 3. Pin the exact version (never use 'latest' in production)
    targetRevision: 15.0.2

    # 4. The Helm-specific configuration block
    helm:
      # Option A: pass specific parameters (like --set in the CLI)
      parameters:
      - name: service.type
        value: LoadBalancer

      # Option B: pass a block of raw YAML values (like a values.yaml)
      values: |
        replicaCount: 3
        resources:
          limits:
            cpu: 100m
            memory: 128Mi

  destination:
    server: 'https://kubernetes.default.svc'
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Pros:** you don't have to copy third-party code into your own Git repos.
**Cons:** your values configuration is embedded directly inside the Application YAML, which gets messy with hundreds of lines.

### Method 2: The umbrella chart approach (true GitOps)

If you have a massive custom `values.yaml`, or you're deploying your own in-house microservices, the true GitOps way is to store your `values.yaml` inside your own Git repository.

Create a folder in your repo (e.g., `apps/my-database/`) with two files:

**`apps/my-database/Chart.yaml`** (declares the remote chart as a dependency):
```yaml
apiVersion: v2
name: my-database-deployment
version: 1.0.0
# We pull the actual logic from Bitnami as a dependency
dependencies:
  - name: postgresql
    version: 12.1.0
    repository: https://charts.bitnami.com/bitnami
```

**`apps/my-database/values.yaml`** (your custom configuration):
```yaml
# Target the dependency name to override its values
postgresql:
  auth:
    postgresPassword: "super-secret-password"  # in production, use External Secrets
  primary:
    persistence:
      size: 50Gi
```

**The Application YAML** points at your own Git repository, not the Helm repo. Argo CD sees the `Chart.yaml`, downloads the Bitnami dependency, merges it with your `values.yaml`, and deploys:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-database
  namespace: argocd
spec:
  project: default
  source:
    # 1. Point to YOUR Git repository, not the Helm repo
    repoURL: 'https://github.com/my-company/infra-monorepo.git'
    targetRevision: main
    # 2. Point to the folder containing your Chart.yaml and values.yaml
    path: apps/my-database

    # Argo CD auto-detects it's a Helm chart and uses the values.yaml by default
    helm:
      valueFiles:
      - values.yaml

  destination:
    server: 'https://kubernetes.default.svc'
    namespace: database
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Helm gotchas with Argo CD

Because Argo CD runs `helm template` instead of `helm install`, be aware of a few edge cases:

- **`helm ls` won't work:** if you log into your cluster and run `helm list`, it shows nothing. Helm doesn't know the app exists because Argo CD deployed the raw rendered YAML.
- **Helm hooks vs Argo CD hooks:** Helm has its own lifecycle hooks (`helm.sh/hook: pre-install`). Argo CD translates most of these into Argo CD hooks automatically, but complex Helm hooks can fail. If a chart behaves weirdly, you may need to manually map Helm hooks to Argo CD hooks (`argocd.argoproj.io/hook: PreSync`).
- **CRD management:** Helm is notoriously bad at managing Custom Resource Definitions. If a chart includes CRDs, you often need to set `Replace=true` or use `ServerSideApply` in the Argo CD sync options so Argo CD can manage them properly.

---

## Part 2: Mastering the Monorepo

When building a GitOps pipeline, you must decide where to store your Kubernetes YAML. A monorepo approach means putting all infrastructure configurations for every application and environment into a single, large Git repository.

### The two types of monorepos

- **The application monorepo:** where developers write source code. For example, one repo containing the React frontend, the Python backend, and the Go payment worker.
- **The infrastructure monorepo:** where the platform team lives. It contains zero source code — only Kubernetes YAML files, Helm charts, and Kustomize overlays for the entire company.

### The cardinal rule: never mix them

**Never mix your application monorepo with your infrastructure monorepo.** If you put your Kubernetes `Deployment.yaml` in the same repository as your Node.js application code, you create a bottleneck for Argo CD. Here's what happens:

1. A developer fixes a typo in the `README.md` of the Node.js app and pushes the commit.
2. A Git webhook fires and wakes up Argo CD.
3. Argo CD says "A commit happened in this repo! I need to re-render all the Kubernetes YAML to see if the infrastructure changed!"
4. Argo CD spins up CPU and memory to process the YAML, only to find nothing changed.

If you have 50 developers committing 100 times a day, Argo CD constantly thrashes its CPU processing infrastructure changes that don't exist.

### The standard: the two-repo split

- **Repo A (app code):** developers commit here. Jenkins/GitHub Actions builds the Docker image and pushes it to an image registry.
- **Repo B (infra monorepo):** the CI/CD pipeline then automatically commits the new Docker image tag (e.g., `v1.2.0`) to this repository. Argo CD only watches this repository.

### Why an infrastructure monorepo is great

- **Single pane of glass:** you can look at one repository and instantly see the entire state of the company's infrastructure.
- **Atomic PRs:** if you need to upgrade a database password across 50 microservices, you do it in one pull request rather than 50 PRs across 50 repos.
- **App of Apps pattern:** it makes it easy to use one master Argo CD Application to automatically deploy all the other folders in the repository.

### The danger: the repo-server bottleneck

Even with source code separated, putting 500 applications into a single infrastructure monorepo introduces a new risk. When a commit hits the monorepo, the `argocd-repo-server` has to figure out what changed. If you use Helm or Kustomize, it might re-render the manifests for all 500 applications just to see which one was updated, causing CPU spikes and OOM crashes.

Mitigations:

1. **Scale the repo-server:** run multiple replicas with high CPU/memory limits.
2. **Use Git webhooks:** don't let Argo CD poll every 3 minutes — use webhooks so it only works when told to.
3. **Enable path filtering (crucial):** in your Application YAML, configure the annotation `argocd.argoproj.io/manifest-generate-paths`. This tells Argo CD: "When a commit happens in the monorepo, only recalculate this specific app if the commit actually touched files inside this specific folder path."

---

## Part 3: The Git Directory ApplicationSet

This single file automatically deploys every folder inside your monorepo's `apps/` directory.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: monorepo-cluster-bootstrap
  namespace: argocd
spec:
  # ---------------------------------------------------------
  # PART 1: THE GENERATOR (finds the folders)
  # ---------------------------------------------------------
  generators:
  - git:
      repoURL: https://github.com/my-company/infra-monorepo.git
      revision: main
      directories:
      - path: apps/*          # find every subfolder inside 'apps/'

  # ---------------------------------------------------------
  # PART 2: THE TEMPLATE (the cookie cutter)
  # ---------------------------------------------------------
  template:
    metadata:
      # {{path.basename}} is a dynamic variable.
      # If the generator finds 'apps/frontend', this becomes 'frontend'
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/my-company/infra-monorepo.git
        targetRevision: main
        # It injects the exact path it found (e.g., 'apps/frontend')
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        # Create a namespace matching the folder name
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true   # auto-create the namespace if it doesn't exist
```

Drop a new folder into `apps/` and an Application for it appears automatically — no manual YAML for each app.
