# Configuration & Security

## Part 1: Core settings — Repositories & Clusters

Argo CD sits between two worlds: where your code lives (**repositories**) and where your code runs (**clusters**). If these connections aren't configured correctly, nothing works.

### Configuring repositories (the source of truth)

By default, Argo CD can read any public repository without configuration. But your company's code is in a private repository, so you must give the argocd-repo-server the keys to your Git provider.

Three ways to authenticate:

- **HTTPS (username & token):** provide a username and a Personal Access Token (PAT). Easiest to start, but tokens expire — which can cause sudden deployment failures if you forget to rotate them.
- **SSH (private keys):** generate an SSH key pair, put the public key in GitHub, and give the private key to Argo CD. The traditional, highly secure method.
- **GitHub Apps (the enterprise way):** instead of tying access to a single user's token, install a GitHub App for Argo CD. This provides fine-grained permissions and tokens rotate automatically.

Under the hood, Argo CD stores these credentials as standard Kubernetes Secrets with a specific label (`argocd.argoproj.io/secret-type: repository`). You can define them as code using Helm or Kustomize rather than clicking around the UI.

### Configuring clusters (the deployment targets)

**The default: `in-cluster`.** When you install Argo CD, it automatically configures a connection to the cluster it runs inside, using the internal Kubernetes address `https://kubernetes.default.svc`. If you deploy to only one cluster, you point your applications at that URL — zero setup.

**Multi-cluster management.** If Argo CD lives in a "management cluster" but deploys to external Dev/Staging/Prod clusters, you configure remote clusters. Each remote target needs two things:

- **A Service Account:** an identity on the remote cluster.
- **RBAC permissions:** rules telling that identity what it may create or delete (usually cluster-admin).

The easiest way to add a remote cluster is the CLI. Running `argocd cluster add <context-name>` automatically reaches out to the remote cluster, creates the Service Account, generates the token, and securely stores it. This turns Argo CD into a centralized control plane in a **hub-and-spoke** architecture.

---

## Part 2: Certificates & Accounts

In a strict corporate environment you can't rely on self-signed certificates or basic admin passwords. You integrate Argo CD into your company's existing security infrastructure.

### Managing certificates (TLS & trust)

Argo CD needs certificates for two reasons: **inbound trust** (so users and webhooks can securely connect to the dashboard) and **outbound trust** (so Argo CD can connect to private Git repositories or custom Helm registries using internal corporate certificates).

**Inbound certificates (securing the dashboard).** By default the argocd-server uses a self-signed TLS certificate, so your browser flashes a "Connection is Not Private" warning. In production you don't configure this inside Argo CD itself — you place a Kubernetes Ingress Controller (NGINX or Traefik) in front of Argo CD, configured with a valid signed certificate (often automated via cert-manager and Let's Encrypt). The Ingress handles TLS encryption and routes secure traffic to the argocd-server. **Crucial:** because Argo CD uses gRPC, you must configure your Ingress to allow gRPC traffic, or the CLI and UI will break.

**Outbound certificates (trusting corporate Git servers).** If your company hosts its own Git server (on-prem GitLab or Bitbucket) with internal SSL certificates, Argo CD refuses to connect, throwing `x509: certificate signed by unknown authority`. Fix: tell Argo CD to trust your company's Certificate Authority by adding your custom CA certificates into the Kubernetes ConfigMap `argocd-tls-certs-cm`. Argo CD mounts this ConfigMap and adds the certificates to its trust store, after which the repo-server clones from your internal Git servers without x509 errors.

### Managing accounts (humans vs machines)

Argo CD handles authentication differently for humans clicking buttons versus machines making requests.

**Human accounts (SSO integration).** Never create individual user accounts with passwords inside Argo CD — that's a security nightmare. Instead use SSO. Argo CD integrates with your Identity Provider (Okta, Google Workspace, Azure AD, GitHub). It bundles a tool called **Dex** that acts as the translator between Argo CD and your corporate directory. When a developer clicks Log In, Dex redirects them to their corporate login page; once authenticated, they're bounced back into Argo CD. You then use RBAC to say "Anyone in the 'Engineering' group in Okta may sync apps in Dev."

**Machine accounts (local users & API tokens).** Sometimes a machine needs to talk to Argo CD — a Jenkins pipeline triggering a webhook, or a monitoring tool querying the API. Machines can't use SSO. You create "local users" for automation: create a user named `jenkins-bot`, generate a long-lived API token instead of a password, and use RBAC to lock it down so it can only do one specific thing (e.g., sync the 'billing-app' and nothing else).

---

## Part 3: argocd-cm vs argocd-secret

The brain of Argo CD is controlled by two Kubernetes objects: `argocd-cm` (a ConfigMap) and `argocd-secret` (a Secret). In plain terms, `argocd-cm` is the **public rulebook** and `argocd-secret` is the **bank vault**.

### argocd-cm (the public rulebook)

A standard Kubernetes ConfigMap holding the systemic configuration of your installation. Everything inside is plain text, so you only put non-sensitive configuration here:

- **SSO/OIDC configuration:** the rules for connecting to Okta or Google (e.g., the identity provider URL).
- **UI customization:** changing the dashboard URL, adding a custom banner ("PRODUCTION CLUSTER — BE CAREFUL"), or linking to internal wikis.
- **Resource inclusions/exclusions:** telling Argo CD to ignore certain Kubernetes resources globally so it doesn't waste CPU tracking them.
- **Custom health checks:** Argo CD knows how to check standard resources, but for third-party Custom Resources (like a Kafka Topic) it won't know how to read health. You write a custom Lua script in `argocd-cm` to teach it.

Because there are no passwords here, it is safe to commit your `argocd-cm` YAML directly into Git.

### argocd-secret (the bank vault)

A Kubernetes Secret holding the sensitive cryptographic keys and passwords Argo CD needs to operate and secure itself:

- **The admin password:** the bcrypt-hashed password for the local admin user.
- **Server signature keys:** Argo CD issues JSON Web Tokens (JWTs) when you log in and uses a cryptographic key stored here to sign them so attackers can't forge login sessions.
- **OIDC client secrets:** if you configure Okta in `argocd-cm`, the actual password (client secret) proving Argo CD may talk to Okta lives here.

**The GitOps problem:** you want your Argo CD configuration in Git, but you cannot put `argocd-secret` in Git, because anyone with repository access could steal your integration passwords or session keys.

**The solution:** never store the raw `argocd-secret` in Git. Use a tool like **External Secrets Operator** (which pulls the password from AWS Secrets Manager) or **Sealed Secrets** (which encrypts the secret before it goes into Git).

### The point of confusion: repository credentials

A common mistake is thinking all secrets go into `argocd-secret`. They do not. To give Argo CD the SSH key to clone your private repo or the TLS certificate for a remote cluster, you do **not** put them in `argocd-secret`. Instead you create brand-new, separate Kubernetes Secrets and tag them with specific labels (like `argocd.argoproj.io/secret-type: repository`). Argo CD constantly scans the cluster for any secret with that label and adds it to its list of known repositories or clusters.

### Cheat sheet

| Feature | argocd-cm | argocd-secret |
|---|---|---|
| Type | ConfigMap | Secret |
| Purpose | System behavior, UI tweaks, SSO routing | Cryptographic keys, admin passwords, SSO secrets |
| Data format | Plain text | Base64-encoded (sensitive) |
| Safe to put in Git? | Yes | No — requires external secret management |

---

## Part 4: RBAC — governing access

Argo CD handles access control using an internal authorization engine called **Casbin**. Instead of writing complex YAML objects per user, you define policies using a simple comma-separated (CSV) string format. All global RBAC rules live in a single ConfigMap called `argocd-rbac-cm`.

### The built-in roles

- **`role:readonly`** — can view everything (apps, clusters, repos) but cannot change, sync, or delete anything.
- **`role:admin`** — full control over the entire Argo CD instance.

For a small startup, `readonly` for developers and `admin` for platform staff might be enough. For an enterprise, you build custom roles.

### The policy syntax (Casbin)

The format:
```
p, <role/user/group>, <resource>, <action>, <object>, <effect>
```

- `p` — stands for "policy" (always starts with this).
- `role/user/group` — the name of the role (e.g., `role:developer`).
- `resource` — what they're touching (`applications`, `clusters`, `repositories`, `projects`, `accounts`).
- `action` — what they're doing (`get`, `create`, `update`, `delete`, `sync`, `override`).
- `object` — the specific target. For applications, formatted as `project-name/application-name`.
- `effect` — `allow` or `deny`.

**Example: the "safe developer" role.** Let developers manually sync any app in the `frontend` project, but not delete apps:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # Allow devs to view all frontend apps
    p, role:frontend-dev, applications, get, frontend/*, allow
    # Allow devs to trigger a sync for frontend apps
    p, role:frontend-dev, applications, sync, frontend/*, allow
    # Explicitly DENY deleting frontend apps
    p, role:frontend-dev, applications, delete, frontend/*, deny
```

### Mapping SSO groups (the `g` syntax)

Defining roles is useless without attaching them to people. When a user logs in, the SSO provider sends Argo CD a list of groups they belong to (e.g., `Engineering-Team`). Use the `g` (group) syntax to map an SSO group to an Argo CD role:

```yaml
data:
  policy.csv: |
    # Policy definitions
    p, role:frontend-dev, applications, sync, frontend/*, allow

    # Group mappings (g, <SSO-Group-Name>, <Argo-Role>)
    g, "GitHub-Frontend-Team", role:frontend-dev
    g, "Okta-SRE-Admins", role:admin
```

### Decentralized RBAC (AppProject-level)

**The monolith problem:** with 500 microservices and 50 teams, putting every RBAC rule into `argocd-rbac-cm` creates an unreadable bottleneck. Every new team must submit a PR to the central platform team to update one file.

**The solution:** delegate RBAC to the AppProject level. Define roles directly inside the project YAML, enabling multi-tenancy where each team manages their own access:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments-team-project
spec:
  sourceRepos:
  - "https://github.com/my-company/payments.git"

  # Decentralized RBAC defined directly in the project
  roles:
  - name: project-admin
    description: "Full control over payment apps"
    policies:
    # No 'p' prefix or role name here, just the action
    - p, applications, *, payments-team-project/*, allow
    groups:
    - "Okta-Payments-Team"   # maps an Okta group directly to this project role

  - name: project-viewer
    description: "Read-only access for QA team"
    policies:
    - p, applications, get, payments-team-project/*, allow
    groups:
    - "Okta-QA-Team"
```

This decentralizes administration. The central platform team creates the empty AppProject and hands it over; from then on the team lead maps their own groups to their own apps without bottlenecking the platform team.
