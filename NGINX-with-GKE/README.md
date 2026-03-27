# 🍕 NGINX on Kubernetes & GKE — 10 Min Read

> **Who this is for:** You know k8s basics (pods, services, deployments). You have a real GKE project coming.  
> **What this covers:** How NGINX fits into k8s, what Ingress is, GKE-specific choices, and one critical 2026 warning you need to know before you start.

---

## Table of Contents

1. [The Mental Shift — NGINX is not the same thing here](#1-the-mental-shift)
2. [What is an Ingress? — The Restaurant District Problem](#2-what-is-an-ingress)
3. [Ingress Controller — Who Actually Does the Work](#3-ingress-controller)
4. [GKE's Dirty Secret — Two Different Ingress Options](#4-gkes-dirty-secret)
5. [🚨 The 2026 Warning — Read This Before You Choose](#5-the-2026-warning)
6. [Installing NGINX Ingress Controller on GKE](#6-installing-nginx-ingress-controller-on-gke)
7. [Writing Your First Ingress YAML](#7-writing-your-first-ingress-yaml)
8. [SSL on GKE — cert-manager replaces certbot](#8-ssl-on-gke)
9. [ConfigMap — How You Configure NGINX in k8s](#9-configmap)
10. [Common GKE-Specific Mistakes](#10-common-gke-specific-mistakes)
11. [Quick Reference](#11-quick-reference)

---

## 1. The Mental Shift

On a regular Linux server, NGINX is software you install, configure with `.conf` files, and manage with `systemctl`.

On Kubernetes, **you don't touch NGINX directly at all.**

You write YAML files. Kubernetes reads them. The NGINX Ingress Controller reads them and configures NGINX automatically behind the scenes.

### The Pizza Metaphor

On a regular server, you ARE the pizza chef. You write the recipes, control the oven, manage everything.

On Kubernetes, you're the **restaurant owner who writes the menu** (YAML). There's a **head chef (Ingress Controller)** who reads the menu and tells the kitchen (NGINX) what to do.

You never touch the kitchen directly. You just update the menu.

```
Old world:  You → edit nginx.conf → reload nginx
K8s world:  You → apply ingress.yaml → controller → configures nginx
```

---

## 2. What is an Ingress?

### The Problem First

Your GKE cluster has 3 apps running as Services:

```
frontend-service     → port 80
api-service          → port 8080
admin-service        → port 9000
```

Users are outside the cluster. They type `yourapp.com` in a browser.

How does traffic from the internet reach the right Service inside the cluster?

**Option A:** Give every Service its own public IP (LoadBalancer type).  
→ 3 apps = 3 Load Balancers = 3 monthly bills on GCP. Expensive. Messy.

**Option B:** One entry point that routes to the right Service based on path or domain.  
→ This is what **Ingress** is.

### The Pizza Metaphor

Your pizza brand now has 3 counters inside one mall:

- 🍕 Main counter (frontend)
- 🍕 Delivery desk (API)
- 🍕 Manager's office (admin)

Instead of 3 separate mall entrances (3 Load Balancers), there's **one main mall entrance** (Ingress) with a **directory board** (routing rules) that sends people to the right counter.

One entrance. One bill. Smart routing.

```
                  Internet
                     |
               [Ingress / LB]
               /      |      \
        /app/      /api/     /admin/
           |          |          |
    frontend-svc  api-svc   admin-svc
```

### The Ingress Resource (YAML)

An Ingress is just a Kubernetes object that defines the routing rules:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  rules:
  - host: yourapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

This YAML says: requests to `yourapp.com/` go to `frontend-service`, requests to `yourapp.com/api` go to `api-service`.

**But this YAML does nothing by itself.** It needs a controller to enforce it.

---

## 3. Ingress Controller

### The Problem First

The Ingress object is just a set of rules written on paper. Something has to actually read those rules and enforce them — configure a real proxy, handle real traffic.

That's the **Ingress Controller**.

In the real world, an Ingress Controller is a pod (or set of pods) running inside your cluster. It watches for Ingress objects and configures itself accordingly.

**NGINX Ingress Controller** = a pod running NGINX that auto-configures itself based on your Ingress YAMLs.

### How It Works

```
You apply ingress.yaml
        ↓
Ingress Controller pod sees the new Ingress object
        ↓
Controller updates NGINX config inside the pod
        ↓
Traffic from internet → NGINX pod → right Service → right Pod
```

No `nginx -t`. No `systemctl reload`. The controller handles all of that automatically every time you update an Ingress object.

---

## 4. GKE's Dirty Secret — Two Different Ingress Options

On GKE specifically, you have a choice that confuses everyone:

### Option A: GKE Ingress (Built-in, Google's own)

GKE comes with its own built-in Ingress controller that provisions a **Google Cloud HTTP(S) Load Balancer** when you create an Ingress object.

```
Ingress YAML → GKE Ingress Controller → Google Cloud Load Balancer (external to cluster)
```

**Pros:**
- Zero setup — it just works on GKE
- Native Google Cloud integration (Cloud Armor, IAP, CDN)
- Traffic goes directly to pods via NEGs (faster, fewer hops)

**Cons:**
- One Load Balancer per Ingress object (gets expensive with many apps)
- Less flexible than NGINX (fewer features)
- Services must be `NodePort` type (GKE requirement)

### Option B: NGINX Ingress Controller (Self-installed)

You install NGINX Ingress Controller yourself. It runs as pods inside the cluster. GKE creates ONE Load Balancer pointing to the NGINX pods, and NGINX handles routing internally.

```
Ingress YAML → NGINX Controller (pod) → routes to Services
                    ↑
          One GCP Load Balancer in front
```

**Pros:**
- One Load Balancer for all apps (cheaper)
- Full NGINX feature set (rate limiting, custom headers, auth, rewrites)
- Familiar if you already know NGINX config
- Works the same on any k8s cluster (not GKE-specific)

**Cons:**
- Extra setup required
- You manage the NGINX pods
- ⚠️ **See Section 5 before choosing this**

### Which to Use

| Situation | Use |
|-----------|-----|
| Simple apps, native GCP features (Cloud Armor, IAP) | GKE Ingress |
| Many microservices, need NGINX-specific features | NGINX Ingress Controller |
| Learning/personal project on GKE | GKE Ingress (simpler) |
| Production, many apps, cost matters | NGINX Ingress (one LB) |

---

## 5. 🚨 The 2026 Warning

This is the most important thing in this entire document. Read it.

### What Happened

The community-maintained NGINX Ingress Controller (`kubernetes/ingress-nginx`) — the one used in virtually every tutorial you'll find online — **was officially retired in March 2026.**

No more security patches. No more bug fixes. No more releases. The GitHub repository is archived (read-only).

Existing deployments will continue to function and installation artifacts will remain available — but there will be no further releases, no bugfixes, and no updates to resolve any security vulnerabilities that may be discovered.

In February 2026, just before the deadline, four new HIGH-severity CVEs were disclosed and patched. From March 2026 onward, they would not have been.

### What This Means For You

If you're starting a new GKE project **right now (March 2026)**, you should NOT install the community `ingress-nginx` controller for production. It's unmaintained and will accumulate unpatched security vulnerabilities.

### Your Actual Options in 2026

**Option 1: Use GKE's built-in Ingress (simplest)**  
Google's own controller. Actively maintained. Works great for most projects.

**Option 2: NGINX Ingress by F5/NGINX Inc.**  
There are TWO different NGINX controllers. The retired one is `kubernetes/ingress-nginx` (community). The still-maintained one is `nginxinc/kubernetes-ingress` — maintained by F5/NGINX Inc. (the actual NGINX company).

```
❌ kubernetes/ingress-nginx    ← RETIRED (the one in most tutorials)
✅ nginxinc/kubernetes-ingress ← Still maintained by F5/NGINX Inc.
```

**Option 3: Gateway API (the future standard)**  
Gateway API is a General Availability networking standard that has maintained a "standard channel" without a single breaking change or API version deprecation for over two years. GKE has native Gateway API support. This is where the whole ecosystem is heading.

### Pizza Metaphor for This Situation

The most popular pizza chain in the city (ingress-nginx) announced it's closing all locations. The building still exists, you can still walk in, but there's no staff, no health inspections, and if there's a cockroach problem nobody's coming to fix it.

You wouldn't open a new restaurant franchise there.

**Use one of the other chains that's still operating.**

---

## 6. Installing NGINX Ingress Controller on GKE

If you still need NGINX Ingress for your project (because of features or familiarity), use the **F5/NGINX Inc. version** or the **community version with awareness** of the above.

### Install via Helm (recommended)

```bash
# Add the repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install into its own namespace
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Verify it's running

```bash
# Check pods are up
kubectl get pods -n ingress-nginx

# Wait until controller pod is ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Get the external IP (takes a minute to provision)
kubectl get service ingress-nginx-controller -n ingress-nginx
```

The `EXTERNAL-IP` column shows the GCP Load Balancer IP. Point your DNS A record here.

### GKE Private Cluster Extra Step

If your cluster is private, you'll need to add a firewall rule that allows master nodes access to port 8443/tcp on worker nodes.

```bash
gcloud compute firewall-rules create allow-nginx-webhook \
  --allow tcp:8443 \
  --source-ranges <master-node-CIDR> \
  --target-tags <your-node-tags>
```

---

## 7. Writing Your First Ingress YAML

### Basic Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    # Tell k8s to use NGINX controller (not GKE's built-in)
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: yourapp.com
    http:
      paths:
      # Frontend — React app
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

      # Backend API
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### With HTTPS (TLS)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    # Auto-redirect HTTP to HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Rate limiting (requests per second per IP)
    nginx.ingress.kubernetes.io/limit-rps: "10"
    # Max upload size (equivalent to client_max_body_size)
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  tls:
  - hosts:
    - yourapp.com
    secretName: yourapp-tls   # This secret holds the SSL cert
  rules:
  - host: yourapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Important: `pathType` Values

| Value | Behaviour |
|-------|-----------|
| `Prefix` | Matches `/api` and `/api/users` and `/api/anything` |
| `Exact` | Only matches `/api` exactly — not `/api/users` |
| `ImplementationSpecific` | Controller decides (GKE Ingress default) |

For most cases use `Prefix`. Use `Exact` when you want to lock a path down strictly.

---

## 8. SSL on GKE

### The Problem

On a regular server, you ran `certbot` and it modified your nginx config automatically.

On k8s, nginx config is auto-generated and you don't touch it directly. Certbot doesn't work here.

### The Solution — cert-manager

`cert-manager` is the k8s equivalent of certbot. It:
- Watches for TLS secrets referenced in Ingress objects
- Automatically provisions certificates from Let's Encrypt
- Renews them before expiry

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

```yaml
# Create a ClusterIssuer (who provides the cert — Let's Encrypt)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

```yaml
# Now your Ingress gets auto-SSL by referencing the issuer
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"   # ← this line does the magic
spec:
  tls:
  - hosts:
    - yourapp.com
    secretName: yourapp-tls   # cert-manager creates this Secret automatically
  rules:
  - host: yourapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

cert-manager sees the Ingress → requests cert from Let's Encrypt → stores it as a k8s Secret → NGINX uses it. Automatic renewal included. Same concept as certbot, different execution.

---

## 9. ConfigMap — How You Configure NGINX in k8s

Remember all those nginx.conf settings from the regular NGINX notes? In k8s you can't edit nginx.conf directly.

Instead, the NGINX Ingress Controller reads a **ConfigMap** for global settings.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  # Global client_max_body_size equivalent
  proxy-body-size: "50m"

  # Enable gzip
  use-gzip: "true"
  gzip-level: "5"

  # Timeouts
  proxy-read-timeout: "60"
  proxy-send-timeout: "60"

  # Real IP from headers (important behind GCP LB)
  use-forwarded-headers: "true"
  compute-full-forwarded-for: "true"
```

Per-route settings go as **annotations** on the Ingress object:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"       # client_max_body_size
    nginx.ingress.kubernetes.io/ssl-redirect: "true"          # force HTTPS
    nginx.ingress.kubernetes.io/limit-rps: "10"               # rate limiting
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"     # for slow APIs
    nginx.ingress.kubernetes.io/rewrite-target: /             # path rewriting
```

### nginx.conf → k8s equivalent

| nginx.conf directive | k8s equivalent |
|---------------------|----------------|
| `client_max_body_size 50M` | annotation: `proxy-body-size: "50m"` |
| `proxy_pass` | handled automatically by Ingress rules |
| `ssl_certificate` | TLS Secret (via cert-manager) |
| `location /api/` | Ingress `path: /api` rule |
| `return 301 https://` | annotation: `ssl-redirect: "true"` |
| `limit_req_zone` | annotation: `limit-rps: "10"` |

---

## 10. Common GKE-Specific Mistakes

### ❌ Using community ingress-nginx for new production projects

As of March 2026 it's retired. Use GKE Ingress, F5 NGINX Ingress, or Gateway API instead.

### ❌ Expecting one Load Balancer IP per Ingress with GKE Ingress

With GKE's built-in Ingress, each Ingress object creates its own Load Balancer. 5 Ingress objects = 5 Load Balancers = surprise GCP bill.

Solution: Put all your routing rules in ONE Ingress object. Or use NGINX Ingress Controller (one LB for all).

### ❌ Service type is ClusterIP when using GKE Ingress

GKE's built-in Ingress requires Services to be `NodePort` type.

```yaml
# ❌ Wrong for GKE Ingress
spec:
  type: ClusterIP

# ✅ Right for GKE Ingress
spec:
  type: NodePort
```

NGINX Ingress Controller doesn't have this requirement — ClusterIP is fine.

### ❌ DNS pointing to nothing

After applying your Ingress, wait for the Load Balancer IP to be assigned:

```bash
kubectl get ingress myapp-ingress
# NAME            CLASS   HOSTS          ADDRESS         PORTS
# myapp-ingress   nginx   yourapp.com    34.100.X.X      80, 443
```

The `ADDRESS` field starts empty. It takes 1-5 minutes for GCP to provision the Load Balancer and assign an IP. Don't panic. Don't reinstall everything.

### ❌ cert-manager timeout because port 80 is blocked

cert-manager uses HTTP challenge to verify you own the domain. If port 80 is blocked by GCP firewall, verification fails. Ensure port 80 is open even if you intend to redirect to HTTPS.

```bash
gcloud compute firewall-rules create allow-http \
  --allow tcp:80 \
  --source-ranges 0.0.0.0/0
```

### ❌ Logs — where are they on k8s?

No `journalctl`, no `/var/log/nginx/`. Logs are in the pods:

```bash
# Get nginx ingress controller pod name
kubectl get pods -n ingress-nginx

# Stream logs
kubectl logs -n ingress-nginx ingress-nginx-controller-XXXXX -f

# Or for your app pod
kubectl logs -n default your-app-pod-XXXXX -f
```

---

## 11. Quick Reference

### Key Commands

```bash
# Apply any YAML
kubectl apply -f ingress.yaml

# Check ingress status and IP
kubectl get ingress

# Check ingress controller pods
kubectl get pods -n ingress-nginx

# View ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller -f

# Describe ingress (see events, errors)
kubectl describe ingress myapp-ingress

# Check cert-manager certificates
kubectl get certificates
kubectl describe certificate yourapp-tls
```

### The k8s NGINX Stack at a Glance

```
Your YAML files
    ├── Ingress         → routing rules (paths, hosts, TLS)
    ├── Service         → points to your app pods
    ├── ConfigMap       → global nginx settings
    └── Secret (TLS)    → SSL certificate (managed by cert-manager)

Running in the cluster
    ├── NGINX Ingress Controller pods  → reads Ingress YAMLs, runs nginx
    ├── cert-manager pods              → manages SSL certs
    └── Your app pods                 → the actual apps

GCP Infrastructure
    └── One Cloud Load Balancer       → public IP, points to NGINX pods
```

### 2026 Controller Decision Tree

```
Starting a new GKE project?
        |
        ├── Need simplicity, GCP-native features (Cloud Armor, IAP)?
        │       → Use GKE Ingress (built-in, no install needed)
        |
        ├── Need NGINX-specific features, many apps, cost control?
        │       → Use F5/NGINX Inc. controller (nginxinc/kubernetes-ingress)
        │         NOT the retired kubernetes/ingress-nginx
        |
        └── Building something new, greenfield, future-proof?
                → Learn Gateway API + GKE Gateway
                  That's where everything is heading anyway
```

---

*The pizza shop moved to the cloud. The oven is still NGINX. You just don't touch it directly anymore. 🍕*
