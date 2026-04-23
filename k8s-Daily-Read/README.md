# ☸️ Kubernetes: Daily 20-Minute Review
### Mid → Senior Engineer Interview Mastery Guide
> **How to use this:** Read it every day. Don't memorize — *internalize*. By week 3, you'll answer questions before the interviewer finishes asking them. By week 6, you'll be correcting the interviewer. (Gently. They control the offer letter.)

---

## ⏱️ Reading Map (~20 min)

| Section | Time | What You'll Own |
|---|---|---|
| Core Architecture | 4 min | "Explain K8s to me" — nailed |
| Workloads & Controllers | 3 min | Pods, Deployments, StatefulSets — no confusion |
| Networking | 3 min | Services, Ingress, CNI — confidently |
| Storage | 2 min | PV/PVC/StorageClass — clearly |
| Security | 3 min | RBAC, PSA, Secrets — like a pro |
| **CSI Secret Management + GKE Workload Identity** | **3 min** | **The senior-level topic most candidates blank on** |
| Scheduling & Resources | 2 min | Requests, Limits, Affinity — precisely |
| Observability & Troubleshooting | 2 min | The part that gets you hired at senior level |
| STAR Answers | 1 min | Practice 3 real scenarios |

---

## 1. 🏗️ Core Architecture (4 min)

### The One-Line Pitch
> Kubernetes is a **container orchestration platform** that automates deployment, scaling, self-healing, and management of containerized applications across a cluster of machines.

### Control Plane (The Brain)

```
┌─────────────────────────────────────────────────────┐
│                   CONTROL PLANE                     │
│                                                     │
│  kube-apiserver ──► etcd (source of truth)         │
│       │                                             │
│  kube-scheduler ──► decides where Pods run          │
│       │                                             │
│  kube-controller-manager ──► runs reconciliation   │
│       │                                             │
│  cloud-controller-manager ──► talks to cloud API   │
└─────────────────────────────────────────────────────┘
```

| Component | What It Does | Simple Analogy |
|---|---|---|
| **kube-apiserver** | Front door for all K8s communication | The receptionist who validates everything |
| **etcd** | Distributed key-value store, holds all cluster state | The company database — if it dies, everyone panics |
| **kube-scheduler** | Assigns unscheduled Pods to Nodes | HR assigning desks based on requirements |
| **kube-controller-manager** | Runs control loops (Deployment, ReplicaSet, etc.) | The manager checking if the "desired state" matches reality |
| **cloud-controller-manager** | Manages cloud-specific resources (LBs, volumes) | The cloud vendor liaison |

### Worker Node (The Muscle)

| Component | What It Does |
|---|---|
| **kubelet** | Node agent — ensures Pods run as specified by the API server |
| **kube-proxy** | Manages network rules for Services on the node |
| **Container Runtime** | Actually runs containers (containerd, CRI-O) |

### 🔑 Key Concept: The Reconciliation Loop
> K8s is **declarative**. You say *what* you want (desired state). K8s figures out *how* to get there and constantly reconciles actual state → desired state. This is the soul of the entire system.

```
Desired State (YAML) ──► API Server ──► Controller watches
                                              │
                          "Is actual == desired?"
                                 NO? Fix it.
                                 YES? Keep watching.
```

---

## 2. ⚙️ Workloads & Controllers (3 min)

### The Hierarchy
```
Deployment
  └── ReplicaSet
        └── Pod
              └── Container(s)
```

### Workload Quick Reference

| Object | Use Case | Key Behavior |
|---|---|---|
| **Pod** | Smallest deployable unit. Rarely used alone. | Ephemeral. Dies, it's gone. |
| **ReplicaSet** | Ensures N replicas of a Pod are running | Usually managed by Deployment, not you |
| **Deployment** | Stateless apps (APIs, web servers) | Rolling updates, rollback support |
| **StatefulSet** | Stateful apps (DBs, queues) | Stable network identity, ordered deployment |
| **DaemonSet** | Run one Pod per node (logging agents, monitoring) | Auto-adds to new nodes |
| **Job** | Run-to-completion tasks | Runs once (or N times with parallelism) |
| **CronJob** | Scheduled tasks | Wraps a Job with a cron schedule |

### Deployment Strategy — Know This Cold

**RollingUpdate (default):**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Max extra Pods during update
    maxUnavailable: 0  # Zero downtime
```

**Recreate:** Kill all, then start new. Causes downtime. Use for major DB schema migrations.

### StatefulSet vs Deployment — The Classic Interview Trap

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random suffix (pod-abc123) | Ordered (pod-0, pod-1, pod-2) |
| Storage | Shared or ephemeral | Each Pod gets its own PVC |
| Network identity | Random | Stable DNS (`pod-0.svc.ns.svc.cluster.local`) |
| Scale down order | Random | Reverse order (2, 1, 0) |
| Use for | APIs, web | DBs, Kafka, ZooKeeper |

---

## 3. 🌐 Networking (3 min)

### The 4 Networking Problems K8s Solves

1. **Container-to-Container** → Same Pod, use `localhost`
2. **Pod-to-Pod** → Every Pod gets a unique IP (flat network via CNI)
3. **Pod-to-Service** → Services provide stable virtual IPs
4. **External-to-Service** → Ingress, LoadBalancer, NodePort

### Services — Know All 4 Types

| Type | Access Scope | How |
|---|---|---|
| **ClusterIP** (default) | Within cluster only | Virtual IP, only reachable inside cluster |
| **NodePort** | External via `NodeIP:Port` | Opens a port (30000-32767) on every node |
| **LoadBalancer** | External via cloud LB | Provisions a cloud load balancer |
| **ExternalName** | DNS alias to external service | CNAME record, no proxying |

### Ingress — One Entry Point to Rule Them All
```yaml
# Single Ingress routing to two services
rules:
  - host: api.myapp.com
    http:
      paths:
        - path: /v1
          backend: service: name: api-v1
        - path: /v2
          backend: service: name: api-v2
```
> **Remember:** Ingress is just rules. You need an **Ingress Controller** (nginx, traefik, AWS ALB) to actually do anything.

### CNI Plugins — What to Know for Interviews

| Plugin | Key Trait |
|---|---|
| **Flannel** | Simple, no NetworkPolicy support |
| **Calico** | NetworkPolicy support, BGP routing |
| **Cilium** | eBPF-based, best observability, L7 policies |
| **Weave** | Easy setup, encryption built in |

### NetworkPolicy — The Firewall You Write in YAML
```yaml
# "Only allow pods labeled app=frontend to reach this pod on port 8080"
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
```
> ⚠️ **Default behavior:** Without any NetworkPolicy, all pods can talk to all pods. With one NetworkPolicy applied, it becomes default-deny for that pod. This trips people up.

### DNS in Kubernetes
```
<service>.<namespace>.svc.cluster.local
# Example: my-api.production.svc.cluster.local
# Short form within same namespace: my-api
```

---

## 4. 💾 Storage (2 min)

### The 3 Storage Objects

```
StorageClass ──► defines HOW storage is provisioned (cloud provider, SSD/HDD)
PersistentVolume (PV) ──► the actual piece of storage (provisioned manually or dynamically)
PersistentVolumeClaim (PVC) ──► the request for storage (what the Pod uses)
```

### Access Modes — Often Asked

| Mode | Abbreviation | Meaning |
|---|---|---|
| ReadWriteOnce | RWO | One node can read/write |
| ReadOnlyMany | ROX | Many nodes, read only |
| ReadWriteMany | RWX | Many nodes, read/write (NFS, EFS) |
| ReadWriteOncePod | RWOP | Only one **pod** (K8s 1.22+) |

### Reclaim Policy

| Policy | What Happens When PVC Deleted |
|---|---|
| **Retain** | PV and data kept. Manual cleanup needed. |
| **Delete** | PV and underlying storage deleted. |
| **Recycle** | Deprecated. Don't use. |

### Volume Types Quick Reference
- **emptyDir** — Temporary, lives as long as the Pod. Great for sharing between containers in same Pod.
- **hostPath** — Mounts a node directory. Security risk. Avoid in production.
- **configMap / secret** — Mount config/secrets as files.
- **CSI** — Standard interface to plug in any storage (AWS EBS, GCS, Ceph, etc.)

---

## 5. 🔒 Security (3 min)

### RBAC — The 4 Objects

```
ServiceAccount ──► the identity (who)
Role / ClusterRole ──► the permissions (what)
RoleBinding / ClusterRoleBinding ──► connects identity to permissions
```

| Object | Scope |
|---|---|
| Role + RoleBinding | Single namespace |
| ClusterRole + ClusterRoleBinding | Cluster-wide |
| ClusterRole + RoleBinding | ClusterRole scoped to one namespace (useful!) |

```yaml
# Give the "reader" ServiceAccount list/get pods in "production" namespace
kind: RoleBinding
subjects:
  - kind: ServiceAccount
    name: reader
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
```

### Pod Security Admission (PSA) — Replaced PodSecurityPolicy

Three levels, three modes:

| Level | What's Allowed |
|---|---|
| **Privileged** | Everything. No restrictions. |
| **Baseline** | Prevents obvious privilege escalation |
| **Restricted** | Hardened. Follows pod security best practices. |

```yaml
# Label namespace to enforce restricted policy
labels:
  pod-security.kubernetes.io/enforce: restricted
```

### Secrets — The Disappointing Truth
> Secrets are base64-encoded by default. Not encrypted. They're stored as plain text in etcd unless you configure **encryption at rest**. This is a common interview gotcha.

**Real secret security requires:**
- etcd encryption at rest (`EncryptionConfiguration`)
- External secret managers (Vault, AWS Secrets Manager, External Secrets Operator)
- RBAC to restrict `get` on secrets

### Security Checklist (Show This Off in Interviews)
- [ ] Run containers as non-root (`runAsNonRoot: true`)
- [ ] Read-only root filesystem (`readOnlyRootFilesystem: true`)
- [ ] Drop all capabilities (`capabilities: drop: ["ALL"]`)
- [ ] No privilege escalation (`allowPrivilegeEscalation: false`)
- [ ] Use NetworkPolicy to restrict traffic
- [ ] Scan images for vulnerabilities (Trivy, Snyk)
- [ ] Use IRSA/Workload Identity — not static AWS credentials in Pods

---

## 5b. 🔐 CSI Secret Management + GKE Workload Identity (3 min)

> This is the topic most mid-level engineers hand-wave through. Seniors explain it end-to-end. You will be a senior.

### Why Not Just Use K8s Secrets?

| Problem | Reality |
|---|---|
| Base64 ≠ encryption | Secrets are readable by anyone with etcd access |
| Secret sprawl | Secrets duplicated across namespaces, clusters, CI/CD pipelines |
| Rotation | K8s Secrets don't auto-rotate. You forget. Bad things happen. |
| Audit trail | Who read the DB password at 3am? K8s has no idea. |

**The solution:** Store secrets in a proper secret manager (Google Secret Manager, AWS Secrets Manager, HashiCorp Vault) and use the **Secrets Store CSI Driver** to mount them into Pods as volumes — without storing anything in etcd.

---

### The Full Architecture: GKE + Google Secret Manager

```
┌──────────────────────────────────────────────────────────────────┐
│                        GKE CLUSTER                               │
│                                                                  │
│  Pod (annotated with KSA)                                        │
│   │                                                              │
│   │ mounts volume                                                │
│   ▼                                                              │
│  Secrets Store CSI Driver (DaemonSet on every node)             │
│   │                                                              │
│   │ calls                                                        │
│   ▼                                                              │
│  GCP Provider Plugin                                             │
│   │                                                              │
│   │ authenticates via Workload Identity (no keys!)               │
│   ▼                                                              │
└──────────────────────────────────────────────────────────────────┘
         │
         │ fetches secret value
         ▼
  Google Secret Manager
  (projects/my-project/secrets/db-password/versions/latest)
         │
         │ returns plaintext value
         ▼
  CSI Driver writes to tmpfs (in-memory, not on disk)
         │
  Pod reads secret as a file at /mnt/secrets/db-password
```

> **tmpfs** = memory-only filesystem. Secret never touches the node's disk. This is the right way.

---

### Concept 1: Workload Identity — The Keyless Auth Chain

> Workload Identity lets a **Kubernetes Service Account (KSA)** impersonate a **Google Service Account (GSA)** — no JSON key files, no secrets in env vars, no crying when keys leak.

**The trust chain:**
```
Pod
 └── annotated with KSA
      └── KSA bound to GSA  (via IAM annotation)
           └── GSA has IAM permission on Secret Manager
                └── Pod can read secrets — no keys anywhere
```

**Step-by-step setup:**

```bash
# Step 1: Enable Workload Identity on GKE cluster
gcloud container clusters update my-cluster \
  --workload-pool=MY_PROJECT_ID.svc.id.goog

# Step 2: Create a Google Service Account (GSA)
gcloud iam service-accounts create gke-secret-reader \
  --display-name="GKE Secret Reader"

# Step 3: Grant GSA access to Secret Manager
gcloud projects add-iam-policy-binding MY_PROJECT_ID \
  --member="serviceAccount:gke-secret-reader@MY_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Step 4: Bind KSA to GSA (the magic link)
gcloud iam service-accounts add-iam-policy-binding \
  gke-secret-reader@MY_PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:MY_PROJECT_ID.svc.id.goog[my-namespace/my-ksa]"
```

```yaml
# Step 5: Create the Kubernetes Service Account with annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-ksa
  namespace: my-namespace
  annotations:
    iam.gke.io/gcp-service-account: gke-secret-reader@MY_PROJECT_ID.iam.gserviceaccount.com
```

> ☝️ That annotation is the handshake. KSA says "I am this GSA". GCP verifies. Done.

---

### Concept 2: Secrets Store CSI Driver

The **Secrets Store CSI Driver** is a DaemonSet that runs on every node. It implements the CSI spec and fetches secrets from external providers (GCP, AWS, Vault, Azure) and mounts them into Pods.

**Install (Helm):**
```bash
# Install the CSI driver
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true   # Optional: sync to K8s Secrets too

# Install GCP provider
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/\
secrets-store-csi-driver-provider-gcp/main/deploy/provider-gcp-plugin.yaml
```

---

### Concept 3: SecretProviderClass — The Bridge Object

`SecretProviderClass` is a CRD that tells the CSI driver: *which secrets to fetch, from where, and how to present them.*

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: gcp-secrets
  namespace: my-namespace
spec:
  provider: gcp                          # ← use the GCP provider
  parameters:
    secrets: |
      - resourceName: "projects/MY_PROJECT_ID/secrets/db-password/versions/latest"
        path: "db-password"              # ← filename inside the Pod's mount
      - resourceName: "projects/MY_PROJECT_ID/secrets/api-key/versions/3"
        path: "api-key"                  # ← pin to version 3, not latest
  secretObjects:                         # ← OPTIONAL: also sync to K8s Secret
    - secretName: app-secrets-k8s        # name of the K8s Secret created
      type: Opaque
      data:
        - objectName: db-password        # must match a path above
          key: DB_PASSWORD               # key in the K8s Secret
```

> `secretObjects` is optional but useful when your app reads `env vars` instead of files. It syncs the external secret into a regular K8s Secret — but only as long as at least one Pod is using the `SecretProviderClass`. When the Pod dies, the K8s Secret is deleted too. This is intentional.

---

### Concept 4: Pod — Consuming the Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: my-namespace
spec:
  serviceAccountName: my-ksa             # ← must match the annotated KSA
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"      # ← secrets appear as files here
          readOnly: true
      env:
        # Option A: Read directly from mounted file
        # Option B: From synced K8s Secret (requires secretObjects above)
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets-k8s
              key: DB_PASSWORD
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: gcp-secrets   # ← points to our SecretProviderClass
```

---

### Secret Rotation — The Part Everyone Forgets

By default, CSI driver polls for secret updates every **2 minutes** (configurable). When Google Secret Manager has a new version:

```
New secret version in GSM
  └── CSI driver detects on next poll
        └── Updates the file at /mnt/secrets/db-password
              └── App must watch the file or restart to pick up change
                    └── Use a sidecar or inotify watcher for hot reload
```

```bash
# Configure rotation poll interval at install time
helm install csi-secrets-store ... \
  --set rotationPollInterval=60s   # check every 60 seconds
```

> ⚠️ **Interview trap:** Rotation updates the *file*. It does NOT restart your Pod or update env vars automatically. If your app reads the secret once at startup (most do), you need a process to reload. Options: watch the file, use a config reloader sidecar, or accept a rolling restart on rotation.

---

### The Complete Mental Model — One Diagram

```
Developer pushes secret to Google Secret Manager
              │
              ▼
  gcloud secrets create db-password --data-file=./secret.txt
  gcloud secrets add-iam-policy-binding db-password \
    --member=serviceAccount:gke-secret-reader@... \
    --role=roles/secretmanager.secretAccessor
              │
              ▼
  SecretProviderClass references the secret (in K8s)
              │
              ▼
  Pod uses CSI volume + annotated ServiceAccount
              │
              ▼
  Node's CSI driver → authenticates as GSA via Workload Identity
              │
              ▼
  Secret Manager returns value → written to tmpfs
              │
              ▼
  App reads /mnt/secrets/db-password  ✅  No keys. No etcd. No tears.
```

---

### Interview Q&A — CSI Secrets & Workload Identity

**Q: Why use CSI driver instead of External Secrets Operator?**
> Both are valid. CSI mounts secrets as **files/volumes** — the secret lives in Pod memory, never in etcd. External Secrets Operator syncs to a **K8s Secret object** (which lives in etcd, encrypted if configured). CSI is lower etcd exposure; ESO is simpler for apps that expect env vars. Many teams use both — CSI for high-sensitivity secrets, ESO for general config.

**Q: What happens if Google Secret Manager is unreachable at Pod startup?**
> Pod fails to start. The CSI driver must successfully fetch the secret before the volume mount completes. This is a dependency. Mitigation: use Secret Manager's SLA (99.95%), retry logic in the driver, and consider caching for non-sensitive config.

**Q: How is Workload Identity different from using a service account JSON key?**
> JSON keys are static credentials that can be copied, leaked, and forgotten. Workload Identity uses short-lived tokens issued by the GKE metadata server — they expire, can't be extracted from the Pod, and are tied to the specific KSA. Zero credentials to rotate or accidentally commit to git.

**Q: Can you use Workload Identity without the CSI driver?**
> Yes. Workload Identity is an authentication mechanism, independent of how you consume secrets. A Pod with Workload Identity can call Secret Manager API directly via the Google Cloud SDK. The CSI driver just automates this and presents secrets as files — no SDK code needed in your app.

**Q: What's the difference between `versions/latest` and a pinned version?**
> `latest` always returns the newest enabled version — good for automatic rotation pickup. A pinned version (e.g., `versions/3`) gives you control and reproducibility. Best practice: use `latest` in production with rotation monitoring, pin versions for compliance/audit environments.

---

### STAR Answer: "How did you secure secrets on GKE?"

**S:** Our team was storing database credentials as K8s Secrets (base64). A security audit flagged that etcd contents were readable by cluster admins and the secrets were also being duplicated manually across 3 clusters.

**T:** I was asked to implement a proper secret management solution that removed secrets from etcd entirely and provided an audit trail for secret access.

**A:** I implemented Secrets Store CSI Driver with the GCP provider across all clusters. I created dedicated Google Service Accounts per microservice with minimal IAM roles (`secretAccessor` only on the specific secrets they needed). I then set up Workload Identity, binding each KSA to its GSA — eliminating JSON keys entirely. All secrets were migrated to Google Secret Manager with versioning. I enabled Secret Manager's audit logs in Cloud Logging so we now had per-secret access records. I configured rotation with a 60-second poll interval and added a file-watcher sidecar for hot-reload in the two services that needed it.

**R:** Zero secrets in etcd for production workloads. Audit revealed 4 stale service account keys that were immediately revoked. Secret rotation went from a manual, risky operation to a 30-second GSM version push. The security team used this as the company-wide standard for all GKE deployments.

---

## 6. 📊 Scheduling & Resources (2 min)

### Requests vs Limits — The Difference That Kills Production

| | Requests | Limits |
|---|---|---|
| **What it is** | What the container is **guaranteed** | What the container is **allowed** to use max |
| **Used for** | Scheduling decisions (scheduler picks node based on this) | Throttling (CPU) or killing (Memory) |
| **OOMKilled** | No | Yes — if memory exceeds limit |
| **CPU throttled** | No | Yes — CPU is throttled, not killed |

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "250m"     # 250 millicores = 0.25 cores
  limits:
    memory: "256Mi"
    cpu: "500m"
```

### QoS Classes — Kubernetes Decides Who Dies First

| Class | Condition | Eviction Priority |
|---|---|---|
| **Guaranteed** | requests == limits for all containers | Last to be evicted |
| **Burstable** | requests < limits | Evicted before Guaranteed |
| **BestEffort** | No requests or limits set at all | Evicted first |

### Scheduling Controls

| Tool | Purpose |
|---|---|
| **nodeSelector** | Simple label matching to schedule on specific nodes |
| **nodeAffinity** | Like nodeSelector but with operators (In, NotIn, Exists) |
| **podAffinity** | Schedule Pod near other Pods (same node/zone) |
| **podAntiAffinity** | Spread Pods apart (HA — don't put all replicas on one node) |
| **taints & tolerations** | Taints repel Pods from nodes; tolerations allow a Pod onto a tainted node |
| **topologySpreadConstraints** | Evenly spread Pods across zones/nodes |

### The Taint/Toleration Pattern (Dedicated Nodes)
```yaml
# Taint a node: "Only GPU workloads allowed"
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule

# Add toleration to Pod so it can run there
tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

---

## 7. 🔍 Observability & Troubleshooting (2 min)

> This section is what separates a mid engineer from a senior. Anyone can deploy. Seniors debug 2am incidents without crying. (Much.)

### The Debugging Flowchart

```
Pod not running?
│
├── kubectl get pods ──► What's the STATUS?
│         │
│         ├── Pending ──► Scheduling issue
│         │       └── kubectl describe pod → Events section
│         │           • Insufficient CPU/Memory → Node too small
│         │           • No nodes match nodeSelector/affinity
│         │           • PVC not bound
│         │
│         ├── CrashLoopBackOff ──► App is crashing on start
│         │       └── kubectl logs <pod> --previous
│         │           • Wrong command/entrypoint
│         │           • Missing env vars or secrets
│         │           • App crash on startup
│         │
│         ├── ImagePullBackOff ──► Can't pull container image
│         │       └── Wrong image name/tag, registry auth issue
│         │
│         ├── OOMKilled ──► Ran out of memory
│         │       └── Increase memory limits
│         │
│         └── Running but misbehaving ──► App-level issue
│                 └── kubectl exec -it <pod> -- /bin/sh
│                     kubectl port-forward pod/my-pod 8080:8080
```

### The 5 Commands You'll Use Every Day

```bash
# 1. Quick cluster state
kubectl get pods,svc,deployments -n <namespace>

# 2. The most useful debugging command in existence
kubectl describe pod <pod-name> -n <namespace>

# 3. Logs (add --previous for crashed containers)
kubectl logs <pod-name> -c <container-name> --tail=100 -f

# 4. Get into a running container
kubectl exec -it <pod-name> -- /bin/bash

# 5. Check events cluster-wide
kubectl get events --sort-by='.lastTimestamp' -n <namespace>
```

### Observability Stack (Know the Ecosystem)

| Layer | Tool Examples |
|---|---|
| **Metrics** | Prometheus + Grafana |
| **Logging** | ELK (Elasticsearch, Logstash, Kibana) / Loki + Grafana |
| **Tracing** | Jaeger, Zipkin, OpenTelemetry |
| **Alerting** | Alertmanager (with Prometheus) |
| **Dashboards** | Grafana, Datadog |

### Probes — The Self-Healing Mechanism

| Probe | Failure Action | Use For |
|---|---|---|
| **livenessProbe** | Restart the container | Detect deadlocks, hung processes |
| **readinessProbe** | Remove from Service endpoints | App not ready to receive traffic (warm-up, DB connecting) |
| **startupProbe** | Delay liveness checks | Slow-starting apps (prevents early restart) |

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3   # Restart after 3 failures
```

---

## 8. ⭐ STAR Method Interview Answers (1 min read, practice daily)

> STAR = **S**ituation, **T**ask, **A**ction, **R**esult

### Q: "Tell me about a time you troubleshot a production K8s issue."

**S:** Our payment service was intermittently returning 503 errors in production during peak hours.

**T:** I needed to identify and fix the root cause with minimal downtime during a high-traffic period.

**A:** I ran `kubectl get pods` and noticed frequent pod restarts. `kubectl describe` showed OOMKilled events. The memory limit was set to 256Mi but the service was spiking to 400Mi during traffic bursts. I increased limits to 512Mi, set requests to 256Mi to maintain Guaranteed QoS, and added a memory-based HPA. I also added a proper readinessProbe so pods wouldn't receive traffic until fully warmed up.

**R:** 503 errors dropped to zero. We added memory profiling to our CI pipeline to catch regressions early. The incident led to a company-wide standard for setting K8s resource limits.

---

### Q: "How do you ensure high availability in Kubernetes?"

**S:** At my previous company, we had a single-AZ deployment that caused a full outage when the availability zone had an issue.

**T:** I was asked to redesign the deployment for multi-AZ HA.

**A:** I implemented `topologySpreadConstraints` to spread pods across zones, used `podAntiAffinity` to ensure no two replicas landed on the same node, configured PodDisruptionBudgets to ensure at least 2 replicas stayed up during node drains, moved to a multi-AZ managed node group, and tested failover by cordoning and draining individual nodes.

**R:** We achieved 99.95% uptime in the following quarter. Simulated AZ failure tests showed traffic automatically rerouted within 30 seconds.

---

### Q: "How would you secure a Kubernetes cluster?"

**S:** A security audit flagged our K8s cluster as having overpermissive RBAC and no network isolation.

**T:** I led a cluster hardening project over 6 weeks.

**A:** I audited existing RBAC with `kubectl auth can-i --list` for all service accounts, applied least-privilege roles, enforced `restricted` Pod Security Admission on all namespaces, implemented NetworkPolicies to deny all ingress by default with explicit allows, enabled etcd encryption at rest, migrated from static credentials to IRSA (IAM Roles for Service Accounts), and integrated Trivy into CI to block images with critical CVEs.

**R:** The follow-up audit showed zero critical findings. We reduced the blast radius of any single compromised pod to nearly nothing.

---

## 9. 📋 Quick Concepts Glossary (Scan Daily)

| Term | One-Line Definition |
|---|---|
| **Namespace** | Virtual cluster inside a cluster. Scope for resources. |
| **Label** | Key-value pairs on objects. Used for selection. |
| **Selector** | Query to find objects by labels |
| **Annotation** | Key-value metadata NOT used for selection (for tooling, humans) |
| **ConfigMap** | Store non-sensitive config as key-value or files |
| **Secret** | Store sensitive data (base64, not encrypted by default — remember!) |
| **HPA** | Horizontal Pod Autoscaler — scale Pods based on CPU/memory/custom metrics |
| **VPA** | Vertical Pod Autoscaler — adjust requests/limits automatically |
| **KEDA** | Event-driven autoscaling (scale to zero, scale on queue depth) |
| **CRD** | Custom Resource Definition — extend K8s API with your own types |
| **Operator** | Controller that manages a CRD using domain-specific logic |
| **Helm** | Package manager for K8s. Charts = templates + values. |
| **Kustomize** | Native K8s config management via overlays. No templating. |
| **PodDisruptionBudget** | Guarantee minimum available pods during voluntary disruptions |
| **LimitRange** | Set default/min/max resources per namespace |
| **ResourceQuota** | Limit total resource consumption in a namespace |
| **ServiceMesh** | Istio/Linkerd — mTLS, traffic management, observability at the mesh layer |
| **Admission Controller** | Intercept API requests before persistence (webhooks, policy enforcement) |
| **OPA / Gatekeeper** | Policy engine for K8s — enforce rules via admission webhooks |

---

## 10. 🎯 Common Interview Questions — Rapid Fire Answers

**Q: What happens when you run `kubectl apply -f deployment.yaml`?**
> Your YAML hits the API server → authenticated + authorized → admission controllers run → stored in etcd → controllers detect new desired state → scheduler assigns Pods → kubelet on target node starts containers.

**Q: What's the difference between `kubectl apply` and `kubectl create`?**
> `create` fails if resource exists. `apply` creates or updates (declarative). Always use `apply` in CI/CD.

**Q: How does a Service find its Pods?**
> Via label selectors. The Endpoints object is dynamically populated with Pod IPs that match the Service selector and pass their readinessProbe.

**Q: What is a headless Service?**
> `clusterIP: None`. Returns individual Pod IPs via DNS instead of a single virtual IP. Used with StatefulSets for direct Pod addressing.

**Q: What happens when a node fails?**
> Node Controller marks node as `NotReady` after timeout (~40s). After `pod-eviction-timeout` (~5min), Pods on that node are marked for eviction and rescheduled on healthy nodes. With StatefulSets, manual intervention may be needed.

**Q: How does rolling update ensure zero downtime?**
> New Pods start and pass readinessProbe before old ones are terminated. `maxUnavailable: 0` ensures no capacity loss. Traffic only routes to ready Pods.

**Q: What is etcd and what happens if it goes down?**
> etcd is the cluster's source of truth. If it goes down: existing workloads keep running (kubelet operates independently), but you can't create/update/delete anything. The API server can't function. For HA: run etcd as a 3 or 5-node cluster with quorum.

**Q: Explain Pod lifecycle phases.**
> `Pending` → scheduled, not started | `Running` → at least one container running | `Succeeded` → all containers exited 0 | `Failed` → containers exited non-0 | `Unknown` → node communication lost

---

## 11. 🧠 Architecture Decisions — Think Like a Senior

> Interviewers at senior level don't just want answers. They want to see you **think in trade-offs**.

### "Should I use Deployment or StatefulSet?"
> Default to Deployment. Use StatefulSet only if you need: stable network identity, ordered deployment/scaling, or per-Pod persistent storage. If you're deploying a database, you probably shouldn't be running it in Kubernetes at all — use a managed service (RDS, Cloud SQL). If you must, StatefulSet.

### "How would you handle secrets?"
> Base64 in K8s Secrets is not enough for production. I'd use External Secrets Operator with AWS Secrets Manager/HashiCorp Vault, enable etcd encryption at rest, and restrict Secret access via RBAC. Pods access secrets via projected volumes, not env vars where possible (env vars can leak in logs).

### "How do you handle config across environments?"
> Kustomize overlays for per-environment values (dev, staging, prod) or Helm with separate values files. ConfigMaps for non-sensitive config, Secrets/external secret managers for sensitive. Never bake environment-specific config into the image.

### "How do you do zero-downtime deployments?"
> RollingUpdate strategy with `maxUnavailable: 0`, proper readinessProbes, PodDisruptionBudgets, preStop lifecycle hooks to drain gracefully, and setting `terminationGracePeriodSeconds` appropriately. For complex cases: blue-green via label switching or canary via Argo Rollouts.

---

## 12. 📐 Numbers & Defaults Worth Memorizing

| Parameter | Default Value |
|---|---|
| NodePort range | 30000–32767 |
| Pod CIDR (kubeadm default) | 10.244.0.0/16 |
| Service CIDR (kubeadm default) | 10.96.0.0/12 |
| Node heartbeat interval | 10 seconds |
| Node eviction timeout | ~5 minutes |
| CPU request unit: 1 core | 1000m (millicores) |
| Max Pods per node (default) | 110 |
| etcd recommended cluster size | 3 or 5 nodes (odd number for quorum) |
| Liveness probe initial delay | 0 (set it yourself — 30s is a good start) |
| `terminationGracePeriodSeconds` | 30 seconds |
| HPA min/max default | 1 / 10 (configurable) |
| K8s API version convention | `apps/v1`, `batch/v1`, `networking.k8s.io/v1` |

---

## 📅 Weekly Depth Plan

> Don't just read — rotate deep dives. One topic per week.

| Week | Deep Dive Topic |
|---|---|
| 1 | Networking internals (iptables, kube-proxy modes, DNS resolution) |
| 2 | RBAC design patterns & real-world least-privilege setup |
| 3 | Storage — CSI drivers, dynamic provisioning, backup strategies |
| 4 | Autoscaling — HPA, VPA, KEDA, Cluster Autoscaler interactions |
| 5 | GitOps — ArgoCD or Flux, progressive delivery, Argo Rollouts |
| 6 | Security hardening — PSA, OPA/Gatekeeper, image scanning, IRSA |
| 7 | CSI Secret Management deep dive — multi-cluster GSM, audit logs, rotation strategies |
| Repeat | You're now dangerous. Start contributing to CKA mock exams. |

---

## ✅ Daily 20-Min Completion Checklist

- [ ] Read entire document once (don't skim the architecture section)
- [ ] Say the STAR answers out loud — silence kills interviews
- [ ] Pick ONE concept and explain it to yourself like you're teaching a junior
- [ ] Run one `kubectl` command today, even if it's just `kubectl get nodes`
- [ ] Tomorrow: same document, different mental focus

---

*"The expert in anything was once a beginner who refused to stop showing up."*
*— You, after day 42 of reading this.*

---
> 📌 **Version:** 1.1 | **Target:** Mid → Senior K8s Engineer | **Prep time to results:** 4–6 weeks of daily reading
