# Advanced GitOps: The Four Pillars

To build a production-grade GitOps platform, you must solve four distinct problems: **Security** (secrets), **Scale** (multi-cluster), **Reliability** (rollouts), and **Automation** (CI/CD).

## Pillar 1: Secrets Management in GitOps

**The problem:** Git is plaintext. You cannot store database passwords, API keys, or TLS certificates in your Git repository. But Argo CD only deploys what is in Git. How do you get secrets into the cluster without breaking the GitOps workflow?

### Solution A: Sealed Secrets (the cryptographic approach)

Created by Bitnami, this lets you safely store encrypted secrets directly in Git.

- **The setup:** you install a SealedSecret controller in your cluster. It generates a public/private key pair. You download the public key to your laptop.
- **The workflow:** a developer writes a standard Kubernetes Secret containing a plaintext password. They run a CLI tool called `kubeseal` using the public key, which encrypts the file into a new custom resource called a SealedSecret.
- **The deployment:** the developer commits the SealedSecret to Git. Argo CD deploys it. The SealedSecret controller (which holds the private key) decrypts it and spawns a normal, readable Kubernetes Secret for your pods.

### Solution B: External Secrets Operator (the enterprise standard)

If your company uses AWS Secrets Manager, Azure Key Vault, or HashiCorp Vault, this is the modern standard. You store no secrets in Git, not even encrypted ones.

- **The workflow:** you store the real password (`super-secret-123`) securely in AWS Secrets Manager.
- **The Git manifest:** in your repo, you create an ExternalSecret manifest. It contains no passwords — it is a pointer that says "Go to AWS Secrets Manager and look for a key named `production-db-password`."
- **The deployment:** Argo CD deploys the ExternalSecret. The External Secrets Operator sees it, reaches out to AWS, fetches the real password, and injects it into a Kubernetes Secret.

---

## Pillar 2: Multi-Cluster Deployments

Once you move past a single cluster, managing applications manually becomes impossible. How do you deploy the NGINX Ingress controller to 50 different clusters simultaneously?

### The classic approach: "App of Apps" pattern

Before ApplicationSets existed, this pattern was used:

1. You create a single "Root" Application in Argo CD (e.g., `cluster-bootstrap`).
2. Instead of pointing this Application to a folder of Deployments and Services, you point it to a Git folder containing *other Application YAML files*.
3. **The chain reaction:** when Argo CD syncs the Root app, it spawns child apps. Those child apps then independently sync their respective Git folders, deploying the actual infrastructure.

### The modern approach: ApplicationSet Cluster Generator

Instead of hardcoding 50 Application YAMLs, use an ApplicationSet with the Cluster generator.

When you add a new cluster to Argo CD (`argocd cluster add ...`), Argo CD saves it as a Secret with specific labels (e.g., `environment=staging`). You write an ApplicationSet that says: "Find every cluster labeled 'staging'. For every cluster found, dynamically generate an Application and deploy the code to it."

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: deploy-to-all-staging-clusters
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: staging   # finds all staging clusters
  template:
    metadata:
      name: 'frontend-{{name}}'   # dynamically uses the cluster name
    spec:
      destination:
        server: '{{server}}'      # dynamically uses the cluster URL
      source:
        repoURL: https://github.com/my-company/frontend.git
        path: manifests
```

---

## Pillar 3: Progressive Delivery (Argo Rollouts)

**The problem:** the standard Kubernetes Deployment uses a "rolling update" — it kills old pods and starts new ones. If the new code crashes on startup, it stops. But if the new code starts up fine yet has a logical bug (e.g., the checkout button is broken), Kubernetes doesn't know. It rolls the broken code out to 100% of your users.

**The solution: Argo Rollouts** — a custom Kubernetes controller. You replace your Deployment YAML with a Rollout YAML, which gives you two capabilities:

### Blue/Green deployments

It spins up the entirely new version of your app (Green) while the old version (Blue) still handles 100% of user traffic. You run automated tests against Green invisibly. When confident, Argo Rollouts instantly flips the Kubernetes Service routing to send 100% of traffic to Green.

### Canary deployments

It shifts traffic gradually:

- **Step 1:** route 10% of traffic to the new version.
- **Step 2:** pause for 10 minutes.
- **Step 3:** query Prometheus. Are error rates spiking?
- **Step 4:** if errors are spiking, instantly auto-rollback to the old version. If metrics are healthy, proceed to 50%, then 100%.

---

## Pillar 4: CI/CD Integration & Notifications

### Argo CD Image Updater (the CI/CD bridge)

In a pure GitOps flow, Jenkins or GitHub Actions builds a Docker image (e.g., `v2.1.0`). To trigger Argo CD, your CI pipeline normally has to write a script to clone your infra Git repo, change the image tag in `values.yaml`, and push a commit.

The Image Updater eliminates this. It sits in your cluster and constantly watches your Docker registry (DockerHub, ECR, GCR). When it sees a new image tag pushed, it automatically updates Argo CD for you, creating a seamless bridge between CI (Continuous Integration) and CD (Continuous Deployment).

### Argo CD Notifications

When a sync fails, you can't rely on humans staring at the UI. You push alerts to the tools your team uses. Argo CD Notifications is built in and uses two concepts:

- **Triggers:** "When should I fire?" (e.g., `on-sync-failed`, `on-sync-succeeded`, `on-health-degraded`).
- **Templates:** "What should the message look like?" (e.g., a formatted payload with the application name and a dashboard link).

You configure it via a ConfigMap to connect to Slack, Microsoft Teams, PagerDuty, or webhooks.

```yaml
# Example Slack notification trigger
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  trigger.on-sync-failed: |
    - description: Application syncing has failed
      send:
      - app-sync-failed          # refers to a Slack template
      when: app.status.operationState.phase in ['Error', 'Failed']
```
