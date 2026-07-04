# Networking & the Gateway API

## Part 1: Argo CD behind an API Gateway

When you put Argo CD behind a standard API Gateway or Ingress Controller (NGINX, AWS ALB, Emissary/Ambassador), things often break. The Web UI might load, but the CLI throws errors, or the UI drops its streaming connection.

### The problem: two languages, one port

The argocd-server (which powers the UI and CLI) does something unusual. By default it listens on a single port (443) but speaks **two different protocols** on that same port:

- **Standard HTTP/HTTPS:** used by your web browser for the UI.
- **gRPC (HTTP/2):** used by the `argocd` CLI and by the UI to stream real-time logs and sync status.

Most API Gateways (especially cloud load balancers like AWS ALB or older NGINX setups) assume one port equals one protocol. When the Gateway sees traffic, it tries to parse it as standard HTTP. When the CLI sends a binary gRPC message, the Gateway doesn't understand it, panics, and drops the connection.

Three ways to solve this:

### Solution 1: TLS passthrough (the blind gateway)

The easiest fix, especially with NGINX. Instead of the Gateway decrypting, inspecting, and routing the traffic (TLS termination), you configure it to act as a dumb pipe.

- **How it works:** you tell the Gateway "If traffic comes in for `argocd.mycompany.com`, do NOT decrypt it. Just pass the raw encrypted TCP stream directly to the argocd-server pod."
- **Why it works:** because the Gateway never looks inside the traffic, it doesn't get confused by gRPC packets. The argocd-server decrypts the traffic itself and handles the HTTP vs gRPC split.
- **The catch:** the Gateway loses visibility. It cannot apply Web Application Firewall (WAF) rules or path-based rate-limiting, because it can't see the paths.

### Solution 2: the two-ingress split (the AWS ALB way)

If you use AWS ALB or a Gateway that must terminate TLS (because you use AWS Certificate Manager), you cannot use passthrough — AWS ALB doesn't support it.

- **How it works:** create two separate Ingress objects (and sometimes two subdomains).
  - Ingress A (`argocd.mycompany.com`): configured for HTTP — routes browser traffic to the UI.
  - Ingress B (`grpc.argocd.mycompany.com`): configured with AWS annotations (like `alb.ingress.kubernetes.io/backend-protocol-version: HTTP2`) to tell the ALB to treat all traffic on this domain as gRPC.
- **Why it works:** you explicitly separate the traffic before it hits the Gateway, so the Gateway knows which protocol rules to apply.
- **The catch:** you must use the special gRPC domain when logging in with the CLI (`argocd login grpc.argocd.mycompany.com`).

### Solution 3: gRPC-Web (the modern fallback)

If you're behind a strict corporate firewall that blocks HTTP/2 or gRPC entirely, Argo CD offers a fallback called **gRPC-Web** — a protocol that wraps binary gRPC messages inside standard HTTP/1.1 requests.

- **Why it works:** to the Gateway, the traffic looks like normal HTTP requests. It passes through NGINX, ALB, or corporate firewalls without protocol errors. The argocd-server unpacks the HTTP request, extracts the gRPC message, and processes it.
- **How to use it:** you don't change the server. You tell your CLI to use it with a flag: `argocd login argocd.mycompany.com --grpc-web`.

In AWS or GCP, the two-ingress split or gRPC-Web are usually your best bets.

---

## Part 2: Ingress vs Gateway API

Modern Kubernetes is shifting from Ingress to the Gateway API. To understand why, look at how Ingress failed at scale.

### Kubernetes Ingress (the annotation problem)

The Ingress object was designed to be simple: take traffic from this domain and send it to this Service. Because it was so simple, it couldn't do anything advanced — no HTTP header matching, no canary deployments (sending 10% of traffic to a new version), and no gRPC routing for Argo CD.

**The hack:** vendors (NGINX, Traefik, AWS) started hijacking the `annotations` field. Anything advanced ended up as a messy, vendor-specific script:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"   # vendor-specific hack
```

If your company switched from NGINX to AWS ALB, every Ingress file would break because ALB doesn't understand NGINX annotations. Furthermore, Ingress is a single file — the app developer (who wants to route a path) and the platform team (who manages TLS certificates) are forced to edit the same file. A developer typo could accidentally delete the company's SSL certificate.

### The Gateway API (the modern standard)

The Gateway API splits network routing into three distinct objects, aligning with actual human roles:

**GatewayClass (the infrastructure provider).** Owned by the cloud provider or platform team. Defines what kind of load balancer is available (e.g., "We use an AWS ALB").

**Gateway (the cluster admin / platform team).** Owned by the platform team. Defines the physical entry point: "Open Port 443, listen for HTTPS traffic, use the corporate TLS certificate." App developers are not allowed to touch this.

**HTTPRoute & GRPCRoute (the app developer).** Owned by software engineers. Defines the routing logic: "Take traffic for `/payments` and send it to my Pod." They attach this route to the platform team's Gateway.

**Why this fixes the Argo CD problem:** with the Gateway API there are no vendor-specific annotation hacks. gRPC is supported natively. You create an HTTPRoute for the UI and a GRPCRoute for the CLI, attach both to the Gateway, and it works across any cloud provider.

**Takeaway:** if you're building a new Kubernetes cluster today, skip Ingress and install the Gateway API. Argo CD supports Gateway API routing natively out of the box.

---

## Part 3: Gateway API components in depth

The Gateway API splits the network into modular pieces owned by different teams.

### The infrastructure layer

**1. GatewayClass (the blueprint).** Defines which brand of load balancer you use (AWS ALB, Google Cloud LB, NGINX, Traefik). Owned by the platform team or cloud provider. It's the catalog you order from: "In this cluster, whenever someone asks for a load balancer, give them an AWS ALB."

**2. Gateway (the physical door).** The actual entry point into your cluster. Deploying this YAML makes the cloud provider spin up a load balancer. It defines the IP address, open ports (80/443), and holds the TLS/SSL certificates. Owned by the platform team — the front door of the building: they build the door, put a lock on it (TLS certs), and decide when it's open. They don't care what happens after you walk inside.

### The routing layer

Once traffic walks through the Gateway, it needs directions. App developers write route files and attach them to the Gateway. Because apps speak different languages, there are different route types.

**3. HTTPRoute (the web traffic director).** A Layer 7 router that understands web traffic — URL paths, headers, query parameters. Use case: standard microservices. "If the user goes to `mycompany.com/payments`, send them to the Payments Pod. If they go to `/cart`, send them to the Cart Pod."

**4. GRPCRoute (the microservice / Argo CD director).** Also Layer 7, but built for gRPC (the binary, high-speed protocol). Unlike HTTP, gRPC doesn't really use URL paths — it uses native Service and Method names. A developer writes a rule: "If the incoming packet calls the `yages.Echo.Ping` method, send it to the Echo pod." **This is exactly how you fix the Argo CD CLI:** attach a GRPCRoute to your Gateway and it natively routes the CLI's binary commands to the argocd-server.

**5. TCPRoute (the raw pipe).** A Layer 4 router, completely blind to web traffic. It doesn't know what an HTTP header or URL path is — it sees a raw stream of bytes on a specific port. Use case: databases and legacy systems. If you run PostgreSQL in your cluster, Postgres doesn't speak HTTP. You create a TCPRoute: "Take every raw byte that hits the Gateway on Port 5432 and pipe it directly to the Database pod."

**How they snap together:** multiple Routes can attach to one Gateway. The platform team pays for one expensive cloud load balancer (the Gateway), and 50 app teams can safely attach their HTTPRoutes and TCPRoutes to it without editing each other's files.

---

## Part 4: The complete traffic flow

When a user types `argocd.mycompany.com` into their browser, or runs `argocd app sync`, here is the exact chronological path the packet takes.

**Step 1: External traffic & DNS.** The user's machine looks up `argocd.mycompany.com` in DNS (Route53, Cloudflare). DNS translates this into the public IP of your cloud provider's load balancer.

**Step 2: The Gateway (front door & security check).** The packet hits the physical load balancer provisioned by your Gateway object.
- **TLS termination:** the Gateway holds your SSL certificate. It intercepts the encrypted HTTPS traffic, decrypts it, and looks at the raw request. The Gateway now holds unencrypted traffic and needs to know where to send it.

**Step 3: Route matching (the lobby directory).** The Gateway looks at its attached routes to figure out the destination. This is where Argo CD's architecture splits the traffic:
- **The Web UI (browser):** the Gateway sees standard web paths (like `/login` or `/applications`) and matches them to the **HTTPRoute**.
- **The CLI (terminal):** the Gateway sees a binary HTTP/2 stream calling specific RPC methods and matches it to the **GRPCRoute**.

**Step 4: The Kubernetes Service (the internal switchboard).** Routes do not send traffic directly to Pods — Pods are ephemeral (they die and get new IPs constantly). Instead, the HTTPRoute and GRPCRoute forward traffic to a stable Kubernetes **Service** (the `argocd-server` service). The Service acts as an internal load balancer, keeping a real-time list of all healthy Argo CD pods.

**Step 5: The Argo CD Server Pod (the destination).** The Service forwards the packet to the running argocd-server Pod.
- If it came from the HTTPRoute, it usually hits the pod on internal Port 8080.
- If it came from the GRPCRoute, it usually hits the pod on internal Port 8083.

**Step 6: Response and action.** The argocd-server processes the request (e.g., fetching application status), packages the data back up, and sends it back up this exact same chain to the user's screen.

---

## Part 5: Practical Lab — Exposing Argo CD via Envoy Gateway API

This runbook walks through the 4-step process to deploy the Kubernetes Gateway API, install the Envoy Gateway controller, and route traffic to Argo CD.

### Step 1: Installation (CRDs and Envoy)

Before you can create Gateway or HTTPRoute resources, your cluster needs to understand those words. Install the CRDs first, then the Envoy Gateway controller.

**1. Install the standard Gateway API CRDs:**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

**2. Install Envoy Gateway via Helm:**
```bash
helm repo add envoyproxy https://envoyproxy.github.io/gateway/helm/
helm repo update
helm install eg envoyproxy/gateway -n envoy-gateway-system --create-namespace
```

**3. Verify the installation:**
```bash
kubectl get pods -n envoy-gateway-system
```
You should see a pod named `envoy-gateway-xxxx` in the Running state.

### Step 2: The Gateway resource & TLS

Create the "physical door" (the Gateway) and give it the TLS certificates so it can decrypt incoming HTTPS traffic.

**1. Create a TLS Secret** (assuming you have `cert.pem` and `key.pem`):
```bash
kubectl create secret tls argocd-tls-secret \
  --cert=cert.pem \
  --key=key.pem \
  -n argocd
```

**2. Create the Gateway resource:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: argocd-gateway
  namespace: argocd
spec:
  gatewayClassName: eg   # use the Envoy Gateway controller
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: "argocd.local"
    tls:
      mode: Terminate
      certificateRefs:
      - name: argocd-tls-secret
```

### Step 3: Configure Argo CD & create the route

By default, the argocd-server uses its own self-signed TLS. Since Envoy now handles TLS termination, tell Argo CD to expect raw, unencrypted HTTP traffic from the Gateway.

**1. Configure Argo CD for HTTP (insecure mode):**
```bash
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge \
  -p '{"data":{"server.insecure":"true"}}'
```
Restart the argocd-server pods to pick up the config:
```bash
kubectl rollout restart deployment argocd-server -n argocd
```

**2. Create the HTTPRoute:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd-ui-route
  namespace: argocd
spec:
  parentRefs:
  - name: argocd-gateway
  hostnames:
  - "argocd.local"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: argocd-server
      port: 80
```

### Step 4: Verify access (DNS and connectivity)

Envoy has spun up a load balancer based on your Gateway resource. Find its IP and test.

**1. Get the Gateway IP:**
```bash
kubectl get gateway argocd-gateway -n argocd
```
Look under the ADDRESS column (assume `192.168.1.100`).

**2. Update `/etc/hosts` (for local testing).** Because you used `argocd.local` as the hostname, point that URL to the Envoy Gateway IP:
```
192.168.1.100    argocd.local
```

**3. Test the URL:** open your browser to `https://argocd.local`.

If you're running locally (Minikube or Kind) and can't get a LoadBalancer IP, use port-forwarding on the Envoy proxy service as a last resort:
```bash
kubectl port-forward svc/envoy-argocd-gateway -n argocd 8443:443
# Then visit https://localhost:8443
```
