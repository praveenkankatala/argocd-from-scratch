# 🚀 Argo CD — The Complete GitOps Field Guide

An end-to-end guide to continuous delivery on Kubernetes with **Argo CD** — from the Push-vs-Pull fundamentals to production sharding, custom health checks, and a fully runnable hands-on lab.

![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Native-326CE5?logo=kubernetes&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)
![GitOps](https://img.shields.io/badge/DevOps-GitOps-blue)

## 📖 About

This repository is a complete, hands-on guide to using Argo CD for GitOps across the whole delivery lifecycle — from a single `Application` on your laptop to a sharded, highly-available control plane managing dozens of clusters.

It is written for engineers who want to actually **run** Argo CD, not just read about it. Every concept is paired with the real YAML or command that implements it, and every chapter leads with the *why* before the *how*.

## ✨ What's Inside

- 🔄 **GitOps fundamentals** — Push vs Pull, state reconciliation, Sync vs Health
- 🧠 **Architecture & internals** — the three core components, the reconciliation loop, gRPC, HA & leader election
- 🖥️ **Dashboard & CLI** — the UI inspector plus the full CLI workflow, login to rollback
- 📦 **The core CRDs** — `Application` (the engine) and `AppProject` (the guardrails)
- ⚖️ **Sync, diff & drift** — manual vs auto-sync, self-heal, `ignoreDifferences`, orphaned resources
- 🔐 **Configuration & security** — repos, clusters, certs, `argocd-cm` vs `argocd-secret`, RBAC (Casbin)
- 🌐 **Networking & Gateway API** — the gRPC/HTTP split, Ingress vs Gateway API, an Envoy lab
- 🧩 **Packaging & repo structure** — Helm with Argo CD, the monorepo trap, ApplicationSets
- 🏛️ **Advanced pillars** — secrets management, multi-cluster, Argo Rollouts, Image Updater & notifications
- 🌊 **Sync waves & hooks** — the ordering engine that stops crash-loops on deploy
- 🛠️ **Production operations** — custom Lua health checks, sharding, DR, Prometheus, and a real runbook
- 🧪 **Hands-on lab** — a copy-paste Guestbook lab you can run on a local cluster

## 🚀 Quick Start

**Install Argo CD into a cluster:**

```bash
# Create the namespace and install the full stack
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for everything to come up
kubectl wait --for=condition=available --timeout=300s deployment --all -n argocd
```

**Install the CLI and log in:**

```bash
# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/argocd

# Port-forward, grab the initial password, and log in
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d ; echo
argocd login localhost:8080 --username admin --insecure
```

**Your first Application (the GitOps way):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

```bash
kubectl apply -f guestbook-app.yaml   # Argo CD takes it from here
```

## 📚 Table of Contents

| # | Topic | Chapter |
|---|-------|---------|
| 1 | Why Argo CD? Push vs Pull | [Foundations](argocd-notes/docs/01-foundations.md) |
| 2 | State reconciliation & the three engines | [Foundations](docs/01-foundations.md) |
| 3 | The three core components | [Architecture](docs/02-architecture-and-internals.md) |
| 4 | The reconciliation loop | [Architecture](docs/02-architecture-and-internals.md) |
| 5 | gRPC, HA & leader election | [Architecture](docs/02-architecture-and-internals.md) |
| 6 | The dashboard & the inspector | [Dashboard & CLI](docs/03-dashboard-and-cli.md) |
| 7 | The CLI workflow (login → rollback) | [Dashboard & CLI](docs/03-dashboard-and-cli.md) |
| 8 | The Application CRD (the engine) | [Application & AppProject](docs/04-application-and-appproject.md) |
| 9 | The AppProject (the guardrails) | [Application & AppProject](docs/04-application-and-appproject.md) |
| 10 | Finalizers & blast-radius control | [Application & AppProject](docs/04-application-and-appproject.md) |
| 11 | The sync operation in detail | [Sync, Diff & Drift](docs/05-sync-diff-and-drift.md) |
| 12 | Manual vs automated sync | [Sync, Diff & Drift](docs/05-sync-diff-and-drift.md) |
| 13 | Self-heal & `ignoreDifferences` | [Sync, Diff & Drift](docs/05-sync-diff-and-drift.md) |
| 14 | Diff strategies (SSA, resource tracking) | [Sync, Diff & Drift](docs/05-sync-diff-and-drift.md) |
| 15 | Orphaned resources | [Sync, Diff & Drift](docs/05-sync-diff-and-drift.md) |
| 16 | Repositories & clusters | [Configuration & Security](docs/06-configuration-and-security.md) |
| 17 | Certificates & accounts (SSO/Dex) | [Configuration & Security](docs/06-configuration-and-security.md) |
| 18 | `argocd-cm` vs `argocd-secret` | [Configuration & Security](docs/06-configuration-and-security.md) |
| 19 | RBAC with Casbin | [Configuration & Security](docs/06-configuration-and-security.md) |
| 20 | Argo CD behind an API gateway | [Networking & Gateway API](docs/07-networking-and-gateway.md) |
| 21 | Ingress vs Gateway API | [Networking & Gateway API](docs/07-networking-and-gateway.md) |
| 22 | Envoy Gateway lab | [Networking & Gateway API](docs/07-networking-and-gateway.md) |
| 23 | Deploying Helm charts | [Packaging & Repo Structure](docs/08-packaging-and-repo-structure.md) |
| 24 | Mastering the monorepo | [Packaging & Repo Structure](docs/08-packaging-and-repo-structure.md) |
| 25 | ApplicationSets | [Packaging & Repo Structure](docs/08-packaging-and-repo-structure.md) |
| 26 | Secrets management (Sealed / ESO) | [Advanced Pillars](docs/09-advanced-pillars.md) |
| 27 | Multi-cluster deployments | [Advanced Pillars](docs/09-advanced-pillars.md) |
| 28 | Progressive delivery (Argo Rollouts) | [Advanced Pillars](docs/09-advanced-pillars.md) |
| 29 | Image Updater & notifications | [Advanced Pillars](docs/09-advanced-pillars.md) |
| 30 | Sync waves & hooks (deep dive) | [Sync Waves & Hooks](deep-dives/01-sync-waves-and-hooks.md) |
| 31 | Production ops & troubleshooting (deep dive) | [Production Operations](deep-dives/02-production-operations-and-troubleshooting.md) |

📄 **Start reading → [docs/01-foundations.md](docs/01-foundations.md)**

## ⚡ Cheat Sheet

```bash
# Log in (SSO or grpc-web fallback behind strict firewalls)
argocd login argocd.example.com
argocd login argocd.example.com --sso
argocd login argocd.example.com --grpc-web

# Register a repo and a remote cluster
argocd repo add git@github.com:my-org/infra.git --ssh-private-key-path ~/.ssh/id_rsa
argocd cluster add my-staging-context

# Create an Application from the CLI
argocd app create my-frontend \
  --repo https://github.com/my-org/frontend.git \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev

# Day-to-day
argocd app get my-frontend          # health, sync status, resource tree
argocd app diff my-frontend         # exact Git-vs-live differences
argocd app sync my-frontend         # force an immediate sync
argocd app get my-frontend --refresh  # re-read Git now, bypass cache

# Panic buttons
argocd app history my-frontend      # every synced commit (rollback targets)
argocd app rollback my-frontend <HISTORY_ID>

# Disaster recovery (back up all apps, projects, config, secrets)
argocd admin export -n argocd > argocd-backup.yaml
argocd admin import -n argocd - < argocd-backup.yaml
```

## 🏗️ Repository Structure

```
.
├── README.md                                       # You are here
├── LICENSE                                         # MIT
├── docs/                                           # Core curriculum (read in order)
│   ├── 01-foundations.md
│   ├── 02-architecture-and-internals.md
│   ├── 03-dashboard-and-cli.md
│   ├── 04-application-and-appproject.md
│   ├── 05-sync-diff-and-drift.md
│   ├── 06-configuration-and-security.md
│   ├── 07-networking-and-gateway.md
│   ├── 08-packaging-and-repo-structure.md
│   └── 09-advanced-pillars.md
├── deep-dives/                                     # Production-grade deep dives
│   ├── 01-sync-waves-and-hooks.md
│   └── 02-production-operations-and-troubleshooting.md
└── labs/
    └── ArgoCD-Guestbook-Lab.docx                   # Fully runnable hands-on lab
```

## 🎯 Who This Is For

- **DevOps engineers** moving deployments from `kubectl apply` pipelines to GitOps
- **Platform engineers** rolling out Argo CD as a multi-tenant, multi-cluster control plane
- **SREs** who need custom health checks, sharding, DR, and a troubleshooting runbook
- **Developers** who want to ship by opening a pull request instead of touching a cluster
- **Anyone** who installed Argo CD, saw the login screen, and wondered "now what?"

## 🌟 Why Argo CD?

| Feature | Why It Matters |
|---|---|
| 🔄 Declarative GitOps | Git is the single source of truth; your repo history *is* your audit log |
| 🩹 Self-healing | Manual drift in the cluster is detected and reverted automatically |
| 🌐 Multi-cluster | One control plane fans out to dozens of clusters via ApplicationSets |
| 🔐 Stronger security | Your CI server never holds cluster credentials — smaller blast radius |
| 🚦 Progressive delivery | Blue/Green and Canary rollouts with automated metric-based rollback |
| 🏆 CNCF graduated | Battle-tested, huge community, maintained under the Argo project |

## 🛠️ Common Workflows

**Local / Dev — sync merged code instantly:**

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

**Production — keep a human gate, never auto-delete:**

```yaml
syncPolicy:
  automated:
    selfHeal: true
    prune: false      # flag deletions, don't execute them
ignoreDifferences:
- group: apps
  kind: Deployment
  jsonPointers:
  - /spec/replicas    # let the HPA own replica count
```

**Multi-cluster — deploy to every staging cluster automatically:**

```yaml
kind: ApplicationSet
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: staging
  template:
    metadata:
      name: 'frontend-{{name}}'
    spec:
      destination:
        server: '{{server}}'
```

**Disaster recovery — rebuild the whole fleet from Git:** spin up a blank cluster, install Argo CD, `kubectl apply` your App-of-Apps root, and Argo CD reconstructs everything. See [Production Operations](deep-dives/02-production-operations-and-troubleshooting.md).

## 🤝 Contributing

Contributions are welcome — a typo fix, a clearer explanation, an extra example, or a whole new chapter.

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/awesome-improvement`)
3. Commit your changes (`git commit -m 'Add awesome improvement'`)
4. Push to the branch (`git push origin feature/awesome-improvement`)
5. Open a Pull Request

Drop new notes into `docs/` or `deep-dives/` as `NN-topic.md` and add a row to the Table of Contents.

## 📜 License

This guide is released under the [MIT License](LICENSE).

Argo CD itself is a graduated CNCF project, maintained under the Apache 2.0 license.

## 🔗 Useful Links

- 📘 [Official Argo CD Documentation](https://argo-cd.readthedocs.io/)
- 🐙 [Argo CD on GitHub](https://github.com/argoproj/argo-cd)
- 🧪 [Argo CD Example Apps](https://github.com/argoproj/argocd-example-apps)
- 🧩 [ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- 🚦 [Argo Rollouts (Progressive Delivery)](https://argoproj.github.io/argo-rollouts/)
- 💬 [Argo Project on CNCF Slack](https://argoproj.github.io/community/join-slack)

## ⭐ Show Your Support

If this guide helped you, give it a ⭐ — it helps others find it too.

---

*Built for the GitOps community. Git is the source of truth — everything else is just reconciliation.*
