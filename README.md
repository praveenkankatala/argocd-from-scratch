# Argo CD Notes — A GitOps Field Guide

A structured, from-scratch set of notes on **Argo CD** and GitOps on Kubernetes — written to be read in order, but useful as a reference you can jump around in. It goes from *why* GitOps exists all the way to production sharding, custom Lua health checks, and a real troubleshooting runbook.

Everything here is plain Markdown (plus one hands-on `.docx` lab), so it renders directly on GitHub and stays easy to edit.

---

## What's inside

- **`docs/`** — the core curriculum, nine chapters read best in order (foundations → architecture → day-to-day usage → CRDs → sync/drift → security → networking → packaging → advanced pillars).
- **`deep-dives/`** — two focused, production-grade deep dives that build on the core: the sync-wave & hook orchestration engine, and running Argo CD itself as a production service.
- **`labs/`** — a fully executable, copy-paste hands-on lab against the public Guestbook app.

---

## Learning path

If you're new to Argo CD, read straight through `docs/` in order. Each chapter assumes the previous one.

| # | Chapter | What you'll walk away with |
|---|---------|----------------------------|
| 01 | [Foundations](docs/01-foundations.md) | Push vs Pull, GitOps, state reconciliation, Sync vs Health |
| 02 | [Architecture & Internals](docs/02-architecture-and-internals.md) | The three core components, the reconciliation loop, gRPC, HA & leader election |
| 03 | [Dashboard & CLI](docs/03-dashboard-and-cli.md) | The UI inspector, and the CLI workflow from login to rollback |
| 04 | [Application & AppProject](docs/04-application-and-appproject.md) | The two core CRDs, multi-tenancy, finalizers, blast-radius control |
| 05 | [Sync, Diff & Drift](docs/05-sync-diff-and-drift.md) | Manual vs auto-sync, self-heal, `ignoreDifferences`, orphaned resources |
| 06 | [Configuration & Security](docs/06-configuration-and-security.md) | Repos & clusters, certs, `argocd-cm` vs `argocd-secret`, RBAC (Casbin) |
| 07 | [Networking & the Gateway API](docs/07-networking-and-gateway.md) | The gRPC/HTTP split, Ingress vs Gateway API, an Envoy Gateway lab |
| 08 | [Packaging & Repo Structure](docs/08-packaging-and-repo-structure.md) | Helm with Argo CD, the monorepo trap, ApplicationSets |
| 09 | [Advanced Pillars](docs/09-advanced-pillars.md) | Secrets, multi-cluster, Argo Rollouts, Image Updater & notifications |

### Deep dives

Read these after the core chapters — they go deeper on two topics that make or break a production setup.

| # | Deep dive | Why it matters |
|---|-----------|----------------|
| 01 | [Sync Waves & Hooks](deep-dives/01-sync-waves-and-hooks.md) | Ordering + lifecycle scripting — the difference between a clean deploy and a 10-minute crash-loop |
| 02 | [Production Operations & Troubleshooting](deep-dives/02-production-operations-and-troubleshooting.md) | Custom health checks, scaling/sharding, DR, Prometheus, and a component-by-component runbook |

### Hands-on lab

- **[Argo CD Guestbook Lab](labs/ArgoCD-Guestbook-Lab.docx)** — a copy-paste lab you can run end to end on a local `kind`/Minikube cluster. Covers install, the CLI, all three deploy styles (declarative / Helm / Kustomize), drift & self-heal, prune, rollback, sync waves, hooks, `ignoreDifferences`, AppProject guardrails, App of Apps, and ApplicationSets.

---

## Topic index

Quick jump-to for common questions:

- **"Why not just `kubectl apply` from CI?"** → [Foundations](docs/01-foundations.md)
- **"Which component do I scale / blame?"** → [Architecture](docs/02-architecture-and-internals.md), [Production Ops](deep-dives/02-production-operations-and-troubleshooting.md)
- **"My app is Synced but Degraded"** → [Sync, Diff & Drift](docs/05-sync-diff-and-drift.md), [Production Ops runbook](deep-dives/02-production-operations-and-troubleshooting.md)
- **"Argo CD keeps overwriting my HPA / Istio sidecar"** → `ignoreDifferences` in [Sync, Diff & Drift](docs/05-sync-diff-and-drift.md)
- **"How do I stop Dev from deploying to Prod?"** → [Application & AppProject](docs/04-application-and-appproject.md)
- **"My CLI breaks behind the load balancer"** → [Networking & Gateway API](docs/07-networking-and-gateway.md)
- **"How do I get secrets into Git safely?"** → [Advanced Pillars](docs/09-advanced-pillars.md)
- **"How do I deploy to 50 clusters?"** → [Advanced Pillars](docs/09-advanced-pillars.md), [Packaging](docs/08-packaging-and-repo-structure.md)
- **"Ordering deploys so the DB comes up first"** → [Sync Waves & Hooks](deep-dives/01-sync-waves-and-hooks.md)

---

## How to use this repo

- **Reading:** browse it right here on GitHub — every file renders as formatted Markdown.
- **The lab:** open `labs/ArgoCD-Guestbook-Lab.docx` and follow along on a throwaway cluster. Nothing in it touches a real environment.
- **Contributing your own notes:** drop a new `NN-topic.md` into `docs/` or `deep-dives/` and add a row to the tables above.

---

## Prerequisites (to follow the lab)

- A local Kubernetes cluster (`kind` or Minikube)
- `kubectl`
- `helm`
- The `argocd` CLI (installed as part of the lab)

---

## License

Released under the [MIT License](LICENSE) — free to read, share, and adapt.
