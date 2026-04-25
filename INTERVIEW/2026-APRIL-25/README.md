# DevOps Interview Prep — Read This Every Morning

> **How to use this:** Don't memorize. Read it like a story. Each answer has a flow — a way of thinking. Internalize that flow, not the words.

---

## SECTION 1 — About You

---

### Q1. Tell me about your current role and the DevOps tools you use.

**The answer structure:** Start with your title → your main area → specific tools → one sentence on your goal.

> "I work as a Junior Solution Engineer, focusing on cloud and container-based environments — specifically on Google Cloud Platform.
>
> My core responsibility is Kubernetes. Our team owns all Kubernetes-related decisions: deployments, scaling, and troubleshooting across multiple projects.
>
> On the CI/CD side, I build and maintain Jenkins pipelines for automated deployments. We use GitHub for version control, Docker for containerization, and Argo CD for GitOps-based deployments in some projects.
>
> For event-driven scaling, I work with KEDA. For monitoring, I use Prometheus and Grafana to keep an eye on application health and performance."

**Why this works:** It moves logically — who you are → what you own → what tools you use → why it matters. No rambling.

---

### Q2. Brief us about your profile for a new interviewer.

> "I have around 1.5 years of experience in DevOps. My focus has been building and maintaining CI/CD pipelines using Jenkins, working with Kubernetes for deployments and scaling, Docker for containerization, and Terraform for writing infrastructure as code.
>
> My goal is simple — make deployments faster, reliable, and fully automated."

**Keep it under 60 seconds. This is not your life story.**

---

## SECTION 2 — Kubernetes Daily Work

---

### Q3. You said you handle Kubernetes deployments. What exactly do you do daily?

**Don't say "I deploy things." Be specific.**

> "On a daily basis, I write and manage YAML manifests — Deployments, Services, ConfigMaps. I make sure pods have proper liveness and readiness probes so Kubernetes can self-heal when something goes wrong.
>
> For scaling, I set up HPA for CPU/memory-based scaling and KEDA for event-driven workloads like Kafka queues. For high-traffic services, we also do pre-scaling before US peak hours to avoid latency spikes.
>
> When issues come up, I debug using pod logs, events, and resource metrics. I also manage ConfigMaps and Secrets for application configs and sensitive data."

---

### Q4. A pod is in `CrashLoopBackOff`. Walk me through your debugging steps.

**This is a story, not a checklist. Think out loud.**

> "CrashLoopBackOff means the container is starting, crashing, and Kubernetes keeps retrying it. So my first question is — *why is it crashing?*
>
> **Step 1 — Check pod status.**
> ```bash
> kubectl get pods
> ```
> I look at restart count. High restarts = it's been failing for a while.
>
> **Step 2 — Read the logs.**
> ```bash
> kubectl logs <pod-name>
> kubectl logs <pod-name> --previous   # if it already restarted
> ```
> Usually the error is right here — a missing config, a connection failure, an app crash.
>
> **Step 3 — Describe the pod.**
> ```bash
> kubectl describe pod <pod-name>
> ```
> I check the Events section at the bottom. This tells me about image pull errors, OOMKilled (ran out of memory), or probe failures.
>
> **Step 4 — Check resources.**
> If it says OOMKilled — I increase memory limits. If CPU is throttled — I adjust CPU requests.
>
> **Step 5 — Check configuration.**
> Wrong environment variable? Missing Secret? Dependency service (DB or API) not reachable? These kill containers silently sometimes.
>
> **Step 6 — Fix and redeploy.**
> Update the config or image, apply, and monitor."

---

### Q5. Logs are empty. Pod is still crashing. Now what?

**This is where most people freeze. Show them you go deeper.**

> "Empty logs mean the container is crashing *before* the application even starts. So I shift my focus from the app to the container itself.
>
> **First**, I go back to `kubectl describe pod` and look at the exit code and container state — is it `OOMKilled`, `Error`, or `Completed` (which means wrong command, it just ran and exited)?
>
> **Second**, I check the Dockerfile — specifically `ENTRYPOINT` and `CMD`. A wrong command means the container exits immediately, which produces zero logs.
>
> **Third**, I check the liveness probe. If `initialDelaySeconds` is too short, Kubernetes kills the container before the app even has time to start. Fix: increase the initial delay.
>
> **Fourth**, I run the same image locally with Docker to confirm it actually starts correctly. If it works locally but not in Kubernetes — that's a different problem entirely (see next question).
>
> **Fifth**, I check if required environment variables are missing, or if the image tag is wrong."

---

### Q6. Container works fine locally in Docker but fails in Kubernetes. Why?

> "This is almost always an environment mismatch. The app is fine — the *environment* it's running in is different.
>
> I check in this order:
>
> **Configuration** — Locally, devs often hardcode values. In Kubernetes, those same values come from ConfigMaps or Secrets. If a Secret is missing or wrong, the app breaks.
>
> **Networking** — Locally, services talk to each other via `localhost`. In Kubernetes, they use service names and DNS. If the app still has `localhost:5432` hardcoded, it will never find the database.
>
> **Resource limits** — We define CPU and memory limits in Kubernetes. If limits are too tight, the container gets OOMKilled even though it ran fine on a developer's machine with unlimited RAM.
>
> **File permissions** — Container may not have write access to a volume path it needs.
>
> **Startup timing** — App needs 30 seconds to start, but the liveness probe fires after 10 seconds and kills it. Locally, there's no probe."

---

### Q7. What is the difference between requests and limits in Kubernetes? What happens if you don't define them?

> "Think of it this way: **requests** is what you *reserve*, **limits** is what you *cap*.
>
> **Requests** — The scheduler uses this to decide which node to place your pod on. If you request 500m CPU and a node doesn't have it free, the pod stays Pending.
>
> **Limits** — The hard ceiling. If memory goes above the limit, the pod is killed (OOMKilled). If CPU goes above the limit, it's throttled — slowed down, but not killed.
>
> **What happens with no requests/limits?** The pod gets scheduled fine. But then it can eat all the CPU and memory on a node, starving other pods. HPA also breaks because it can't calculate utilization without a request value. And during resource pressure, pods without defined requests are first in line to get evicted.
>
> Always define both."

---

### Q8. How does HPA work? What metric does it use, and how does it calculate scaling?

> "HPA watches your pods and scales them up or down based on resource usage.
>
> You define a target — for example, 60% CPU. HPA checks actual CPU usage every 15 seconds (by default) and compares it to that target.
>
> The formula is:
> ```
> desired replicas = (current usage / target usage) × current replicas
> ```
>
> Example: You have 2 pods. Target is 60% CPU. Current usage is 90%.
> ```
> desired = (90 / 60) × 2 = 3 pods
> ```
> HPA spins up a third pod.
>
> **Important detail** — utilization is calculated against **requests**, not limits. So if a pod has 0 CPU requests defined, HPA literally cannot do the math. It won't work."

---

### Q9. What if CPU requests are NOT defined? Will HPA still work?

**Short, confident, no fluff.**

> "No. HPA calculates utilization as `current usage ÷ request`. If request is zero or undefined, the division doesn't work. HPA will either error out or produce wrong results.
>
> Same applies to memory-based HPA — memory requests must be defined for it to work.
>
> One-liner to remember: **HPA only works when the corresponding resource requests are defined.**"

---

### Q10. What are the different service types in Kubernetes?

> "Kubernetes has five service types, each for a different exposure need:
>
> **ClusterIP** — Default. Only accessible inside the cluster. Use for internal microservice communication.
>
> **NodePort** — Exposes the app on a port on every node's IP. Useful for testing, not for production.
>
> **LoadBalancer** — Provisions a cloud load balancer (GCP, AWS, etc.) to route external traffic to your pods. This is how you expose apps to the internet in production.
>
> **ExternalName** — Maps a Kubernetes service to an external DNS name. Useful when you want to abstract a third-party service (like an external database) behind a Kubernetes service name.
>
> **Headless Service** — No ClusterIP assigned. Kubernetes returns individual pod IPs directly. Used with StatefulSets when each pod needs to be addressed individually (like Kafka, Cassandra)."

---

### Q11. How do you run a pod on a specific node?

> "Three ways to control pod-to-node placement:
>
> **Node Selector** — Simplest. Label the node, then add a `nodeSelector` to the pod spec.
> ```yaml
> spec:
>   nodeSelector:
>     disktype: ssd
> ```
>
> **Node Affinity** — Same concept but with more flexible rules. You can say 'prefer this node but it's not required', or add multiple conditions.
>
> **Taints and Tolerations** — Works the other way. A **taint** on a node says 'reject all pods unless they tolerate me'. A **toleration** on a pod says 'I'm okay with that taint, let me in.'
>
> Use case: you have GPU nodes. You taint them. Only ML workloads with the right toleration will land there. Everything else stays off those expensive machines."

---

### Q12. What is an Admission Controller in Kubernetes?

> "Think of it as a security checkpoint that every API request passes through before it's stored in etcd.
>
> The request flow is: **Authentication → Authorization → Admission Controller → etcd**
>
> Admission Controllers do two things:
> - **Validate** — check if the request is allowed (e.g., block pods without resource limits)
> - **Mutate** — automatically modify the request (e.g., inject sidecars, set default values)
>
> Real examples:
> - Block pods that don't have labels defined
> - Enforce that every container has resource limits set
> - Auto-inject the Istio sidecar proxy
>
> Tools like OPA/Gatekeeper and Kyverno use this mechanism to enforce cluster-wide policies."

---

## SECTION 3 — Scaling

---

### Q13. How do you scale pods based on external metrics like Kafka queue length?

> "CPU and memory are reactive metrics — they tell you the pod is *already* under stress. For event-driven architectures, you want proactive scaling — scale *before* the pods are overwhelmed.
>
> That's where KEDA comes in. KEDA (Kubernetes Event-Driven Autoscaler) monitors external systems — Kafka, RabbitMQ, Azure Service Bus, even Prometheus queries — and scales pods based on those metrics.
>
> Example: If a Kafka topic has more than 100 unprocessed messages, KEDA increases the replica count. When the queue drains, it scales back down — even to zero, which saves cost.
>
> Under the hood, KEDA creates and manages an HPA object. So Kubernetes still does the actual scaling — KEDA just feeds it the right metrics."

---

## SECTION 4 — CI/CD

---

### Q14. Design a CI/CD pipeline for a microservice using GitHub, Jenkins, Docker, and Kubernetes.

**Walk through it like you're drawing a whiteboard diagram.**

> "Here's the flow from code commit to production:
>
> **1. Developer pushes code to GitHub** → this triggers a webhook to Jenkins automatically.
>
> **2. Jenkins runs the CI stage** → pulls the latest code, runs unit tests and builds. If tests fail, pipeline stops right here. No broken code moves forward.
>
> **3. Docker image is built** → Jenkins builds the image and tags it with the Git commit ID. This makes every build traceable.
>
> **4. Image is pushed to a container registry** — could be Docker Hub, GCR, or a private registry.
>
> **5. CD stage — deploy to Kubernetes** → Jenkins updates the Kubernetes deployment manifest with the new image tag and runs `kubectl apply`. This triggers a rolling update.
>
> **6. Kubernetes health checks take over** → readiness and liveness probes verify the new pods are healthy. If they fail, we roll back with `kubectl rollout undo`.
>
> Simple summary: **GitHub → Jenkins → Docker build → Registry → Kubernetes**"

---

### Q15. How do you ensure production deployments are safe? What about manual approval?

> "We add a gate between staging and production.
>
> After the build and tests pass, Jenkins automatically deploys to a **staging environment**. QA team or automated tests validate it there.
>
> Before touching production, the pipeline **pauses and waits for a human to approve**. In Jenkins, this is an `input` step. The pipeline literally sits there until a DevOps engineer or team lead clicks 'Proceed'.
>
> Only after approval does it deploy to production using a **rolling deployment** strategy — pods are replaced one by one, so there's zero downtime. At any point, if something looks wrong, we run `kubectl rollout undo` and we're back to the previous version in seconds.
>
> For critical services, manual approval is preferred over fully automated — a human sanity check is worth 30 seconds."

---

### Q16. How do you pass GitHub credentials in a Jenkins pipeline?

> "Never hardcode credentials in a Jenkinsfile. That's how you end up on Twitter for the wrong reasons.
>
> Jenkins has a **Global Credentials Manager** — go to Manage Jenkins → Credentials → Add credentials. Store the GitHub personal access token there and give it a readable ID like `github-token`.
>
> In the Jenkinsfile, reference it:
> ```groovy
> withCredentials([string(credentialsId: 'github-token', variable: 'GIT_TOKEN')]) {
>     sh 'git clone https://$GIT_TOKEN@github.com/org/repo.git'
> }
> ```
>
> Jenkins injects the secret at runtime. It's masked in logs. The token never appears in code."

---

### Q17. What deployment strategies do you use in your CI/CD pipeline?

> "Three main strategies, each with a different risk tolerance:
>
> **Rolling Deployment** — Kubernetes replaces old pods with new ones gradually. If the new pod fails health checks, it stops. Zero downtime, low risk. This is our default.
>
> **Blue-Green Deployment** — You maintain two identical environments. 'Blue' is live, 'Green' gets the new version. Once Green is validated, you switch the load balancer. If something breaks, switch back instantly. Higher cost, but zero-risk cutover.
>
> **Canary Deployment** — You release the new version to, say, 5% of users first. You monitor metrics and errors. If everything's fine, gradually roll out to 100%. Best for high-risk releases. Argo Rollouts makes this easy in Kubernetes.
>
> I mostly use rolling deployment for day-to-day. Canary for risky feature releases."

---

## SECTION 5 — Terraform

---

### Q18. What is `null_resource` in Terraform and why do you use it?

> "In Terraform, every resource creates actual infrastructure — an EC2 instance, a GCS bucket, a VPC. But sometimes you just want to **run a command**, not create anything.
>
> That's what `null_resource` is for. It's a placeholder that lets you attach a `provisioner` to run scripts or commands.
>
> Example — after creating an EC2 instance, you want to run an Ansible playbook to configure it. You'd use `null_resource` with a `local-exec` or `remote-exec` provisioner to do that.
>
> Think of it as the `user-data` script equivalent inside Terraform, but more flexible — it can run at any point, be triggered by `triggers`, and isn't tied to instance creation.
>
> One-liner: `null_resource` is Terraform's way of running commands without creating infrastructure."

---

## SECTION 6 — NEW QUESTIONS (Added)

---

### Q19. What is the difference between a Deployment and a StatefulSet in Kubernetes?

> "Both manage pods, but they're for fundamentally different types of applications.
>
> **Deployment** — for stateless apps. Pods are interchangeable. If you have 3 pods and one dies, Kubernetes replaces it with another. The new pod gets a random name and can land on any node. Perfect for web servers, APIs, microservices.
>
> **StatefulSet** — for stateful apps like databases, Kafka, Zookeeper. Pods have sticky identities — `pod-0`, `pod-1`, `pod-2`. They start in order and terminate in reverse order. Each pod gets its own persistent volume that stays attached even if the pod restarts. The pod name is predictable, so other services can address them directly.
>
> Key difference: in a Deployment, pods are sheep — all the same. In a StatefulSet, pods are people — each one has an identity."

---

### Q20. What is a DaemonSet? When would you use it?

> "A DaemonSet ensures that a copy of a pod runs on **every node** in the cluster (or a subset of nodes with selectors). When a new node joins the cluster, the DaemonSet pod is automatically scheduled on it.
>
> Classic use cases:
> - **Log collection** — run Fluentd or Filebeat on every node to collect logs
> - **Monitoring** — run a Node Exporter on every node for Prometheus metrics
> - **Security agents** — run a Falco or Datadog agent on every node
>
> You'd never use a DaemonSet for your application. It's for infrastructure-level concerns that need to be on every machine."

---

### Q21. What is a ConfigMap vs a Secret? When do you use each?

> "Both inject configuration into pods. The difference is sensitivity.
>
> **ConfigMap** — for non-sensitive configuration. Database hostnames, feature flags, log levels, environment names. Stored as plain text in etcd.
>
> **Secret** — for sensitive data. Passwords, API keys, TLS certificates. Stored base64-encoded in etcd (not encrypted by default, but can be encrypted at rest with KMS).
>
> Both can be injected into pods as environment variables or mounted as files.
>
> Best practice: never put sensitive data in a ConfigMap. And for production, consider an external secrets manager like HashiCorp Vault or GCP Secret Manager, with tools like External Secrets Operator to sync them into Kubernetes Secrets."

---

### Q22. Your application is responding slowly. How do you debug performance issues in Kubernetes?

> "Slow response is a symptom. I need to find the root cause — is it the app, the infra, or the network?
>
> **Step 1 — Check pod resource usage.**
> ```bash
> kubectl top pods
> kubectl top nodes
> ```
> Is a pod using 100% CPU? That's CPU throttling — the app is waiting. Is memory near the limit? Garbage collection might be hammering performance.
>
> **Step 2 — Check HPA.** Is the app scaled enough for the current load? Maybe it needs more replicas.
>
> **Step 3 — Check Grafana dashboards.** Look at request latency, error rate, and saturation. Prometheus metrics will tell me if the slowness is consistent or spiky.
>
> **Step 4 — Check if it's a specific service.** In a microservice setup, one slow service can make everything feel slow. Distributed tracing with Jaeger or Zipkin helps isolate where time is being spent.
>
> **Step 5 — Check network.** Is a downstream dependency (database, API) responding slowly? Check connection pool exhaustion, DNS resolution times.
>
> Follow the data, don't guess."

---

### Q23. What is the difference between `kubectl apply` and `kubectl create`?

> "Both create resources, but they behave differently when the resource already exists.
>
> **`kubectl create`** — creates a resource. If it already exists, it throws an error and stops. It's imperative — 'create this thing right now'.
>
> **`kubectl apply`** — creates the resource if it doesn't exist, and *updates* it if it does. It's declarative — 'make the cluster look like this YAML'. It computes the diff and applies only what changed.
>
> In a CI/CD pipeline, always use `kubectl apply`. You don't know if the resource exists already, and you don't want your pipeline to fail because it does. `kubectl create` is fine for one-time manual operations."

---

### Q24. What is Argo CD and how does it fit into your workflow?

> "Argo CD is a GitOps-based continuous delivery tool for Kubernetes. The idea behind GitOps is simple: **Git is the single source of truth** for what should be running in the cluster.
>
> How it works: You store your Kubernetes manifests (or Helm charts) in a Git repo. Argo CD watches that repo. The moment you push a change — updated image tag, new config — Argo CD detects the diff and automatically syncs the cluster to match.
>
> In our workflow: Jenkins handles CI — build, test, push Docker image, update the image tag in the Git manifest repo. Argo CD handles CD — sees the new tag, deploys it to Kubernetes.
>
> Benefits over pure Jenkins CD: you get a UI to see what's deployed vs what's in Git, automatic drift detection (someone manually changed something? Argo will flag it), and easy rollback by reverting a Git commit."

---

### Q25. What is the difference between liveness probe and readiness probe?

> "Both probes check if a pod is healthy, but they answer different questions.
>
> **Readiness probe** — 'Is this pod ready to receive traffic?' If a readiness probe fails, the pod is removed from the Service's endpoint list. Traffic stops going to it. But the pod keeps running. Use this for apps that need time to warm up — loading caches, establishing DB connections.
>
> **Liveness probe** — 'Is this pod still alive?' If a liveness probe fails, Kubernetes **kills and restarts** the pod. Use this to recover from deadlocks — the app process is running but stuck and not serving requests.
>
> Common mistake: setting the liveness probe `initialDelaySeconds` too short. The app hasn't fully started yet, liveness fires, decides it's dead, kills it, restarts it — and you're stuck in CrashLoopBackOff forever. Always give the app more time than you think it needs."

---

### Q26. What is a PersistentVolume and PersistentVolumeClaim? Why do we need them?

> "Containers are ephemeral — when a pod dies, all data inside it is gone. For applications that need to persist data (databases, file storage), we need volumes that outlive the pod.
>
> **PersistentVolume (PV)** — the actual storage. Could be a GCP persistent disk, an NFS share, or an AWS EBS volume. It's a cluster-level resource created by an admin (or dynamically provisioned).
>
> **PersistentVolumeClaim (PVC)** — a request for storage by a pod. The pod says 'I need 10Gi of storage with ReadWriteOnce access'. Kubernetes finds a matching PV and binds them together.
>
> The separation is intentional: developers write PVCs (they don't need to know the underlying storage). Admins manage PVs. **StorageClass** makes this even easier — with dynamic provisioning, you don't need to pre-create PVs at all. The PVC request automatically provisions one from the cloud provider."

---

### Q27. What happens when a node goes down in Kubernetes?

> "This is where Kubernetes self-healing shines.
>
> When a node goes down, the kubelet on that node stops sending heartbeats to the control plane. After the node heartbeat timeout (default ~5 minutes), the node is marked `NotReady`.
>
> After another configurable period, the pods on that node are marked for eviction. The scheduler then reschedules those pods onto healthy nodes — assuming there are available resources.
>
> For this to work properly:
> - Pods should be part of a Deployment (standalone pods are NOT rescheduled)
> - The cluster should have enough capacity on remaining nodes
> - PodDisruptionBudgets should be set so that too many replicas don't get evicted at once
>
> This is why you never run a single replica of anything important in production."

---

### Q28. What is Helm and why do you use it?

> "Helm is the package manager for Kubernetes. Instead of maintaining 15 separate YAML files for a single application, Helm bundles them into a **chart** — a reusable, versioned package.
>
> Charts have **templates** with variables. You fill in the values for different environments in a `values.yaml` file. Same chart, different values for dev/staging/production.
>
> Key benefits:
> - **Reusability** — write once, deploy everywhere with different configs
> - **Versioning** — `helm upgrade` and `helm rollback` for easy version management
> - **Community charts** — install Prometheus, Nginx, Grafana in minutes instead of writing YAMLs from scratch
>
> We use Helm to template our application charts and deploy them via Jenkins pipelines, overriding values per environment."

---

### Q29. How do you handle secrets securely in production? You wouldn't just put them in plain Kubernetes Secrets, right?

> "Correct — base64 is encoding, not encryption. Anyone with kubectl access can decode a Kubernetes Secret in seconds.
>
> In production, we integrate with a proper secrets manager. The approach I prefer:
>
> **External Secrets Operator** — it connects to an external secrets store (GCP Secret Manager, AWS Secrets Manager, HashiCorp Vault) and automatically syncs secrets into Kubernetes as native Secret objects. The actual secret values live in the external store — Kubernetes never stores the plaintext.
>
> Other considerations:
> - **RBAC** — restrict who can read Secrets in the cluster
> - **Encryption at rest** — enable etcd encryption so even the Kubernetes store is encrypted
> - **Audit logs** — track who accessed what secret and when
>
> And obviously — secrets never go in Git. Not even base64 encoded. That's how companies end up in breach reports."

---

## SECTION 7 — Quick-Fire Recall

> Read this section last. These are one-liner reminders for things you already know.

| Question | One-liner answer |
|---|---|
| CrashLoopBackOff | Container crashes on start — check logs, describe, probes, config |
| HPA without requests | Doesn't work — needs requests to calculate utilization |
| Requests vs Limits | Requests = what you reserve; Limits = what you cap |
| Rolling vs Canary | Rolling = gradual pod replacement; Canary = small % of users first |
| ConfigMap vs Secret | ConfigMap = plain config; Secret = sensitive data |
| Liveness vs Readiness | Liveness = is it alive; Readiness = is it ready for traffic |
| Deployment vs StatefulSet | Deployment = stateless; StatefulSet = stateful with persistent identity |
| null_resource (Terraform) | Run commands without creating infrastructure |
| Admission Controller | Validates/modifies K8s requests before they hit etcd |
| Headless Service | No ClusterIP — direct pod access by DNS |
| DaemonSet | One pod per node — for logging, monitoring agents |
| Helm | Package manager for K8s — templates + versioning |

---

*Last updated: April 25, 2026 — Keep adding to this as you face new questions.*
