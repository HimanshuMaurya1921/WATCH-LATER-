# AWS Mid-Level Engineer — Daily 20-Minute Study Read
> All 8 Sessions · Every Interview Trap · Every Golden Rule  
> Repetition rewires identity. Read this every day.

---

## Navigation

| # | Topic | Interview Weight |
|---|-------|-----------------|
| 01 | [EKS & Containers](#session-01-eks--containers) | 90% |
| 02 | [VPC & Networking](#session-02-vpc--networking) | 87% |
| 03 | [IAM](#session-03-iam) | 85% |
| 04 | [S3 & Storage](#session-04-s3--storage) | 80% |
| 05 | [RDS & Databases](#session-05-rds--databases) | 75% |
| 06 | [SQS / SNS / EventBridge](#session-06-sqs--sns--eventbridge) | 70% |
| 07 | [Compute — EC2 & Lambda](#session-07-compute--ec2--lambda) | 65% |
| 08 | [CI/CD — CodePipeline & CDK](#session-08-cicd--codepipeline--cdk) | 55% |

---

## SESSION 01: EKS & Containers
### 90% Interview Weight

---

### Managed Node Groups vs Fargate vs Self-Managed

| | Managed Node Groups | Fargate | Self-Managed |
|--|--|--|--|
| **Who manages nodes** | AWS patches, replaces | AWS owns all infra | You own everything |
| **DaemonSets** | Full support | ❌ Not supported | Full support |
| **Cost model** | EC2 pricing per node | Per-pod CPU+memory | EC2 pricing per node |
| **GPU support** | ✅ Yes | ❌ No | ✅ Yes |
| **Use when** | Most prod workloads | Serverless, compliance | Legacy or exotic needs |

> ⚠️ **TRAP:** Always pick Fargate for new serverless workloads UNLESS you need DaemonSets, GPU, or Windows containers. Same trap as GKE Autopilot — interviewers love this.

---

### Pod Identity vs IRSA — Know This Cold

**The problem:** Hardcoded AWS credentials in pods. They leak, never rotate, violate least privilege.

**Old way (IRSA — IAM Roles for Service Accounts):**
1. Create OIDC provider for your EKS cluster
2. Create IAM role with trust policy referencing the OIDC issuer + namespace + SA name
3. Annotate Kubernetes ServiceAccount with `eks.amazonaws.com/role-arn`
4. Pod gets temporary credentials via projected token

**New way (EKS Pod Identity — 2023+):**
1. Enable Pod Identity Agent addon on the cluster
2. Create `PodIdentityAssociation` linking IAM role → namespace/SA name
3. No OIDC configuration. No trust policy JSON. Much simpler.

> ✅ **GOLDEN RULE:** Pod Identity is simpler and AWS-recommended for new clusters. IRSA still works and is widely deployed. Know both. The exam will ask about IRSA. The real world is migrating to Pod Identity.

---

### HPA vs VPA vs Karpenter

| | HPA | VPA | Karpenter |
|--|--|--|--|
| **What it scales** | Pod REPLICAS | Pod CPU/memory REQUESTS | NODES |
| **Trigger** | CPU, memory, custom metrics | Historical usage analysis | Unschedulable pods |
| **Restart pods?** | No | Yes (to apply new requests) | No |
| **AWS native?** | K8s built-in | K8s built-in | AWS-built, replaces Cluster Autoscaler |
| **Use for** | Stateless services | Right-sizing batch jobs | Node provisioning (prefer over CA) |

> ⚠️ **TRAP:** HPA + VPA Auto mode = infinite conflict loop. VPA changes requests → HPA recalculates utilization → scales → VPA changes again. Use VPA in **Recommendation mode** alongside HPA.

> ✅ **GOLDEN RULE:** Use **Karpenter instead of Cluster Autoscaler** on EKS. Karpenter provisions the exact right instance type for your pod in ~30 seconds. Cluster Autoscaler is slow and instance-type-fixed.

---

### EKS Services — Same Four Types, Different AWS Gotchas

- **ClusterIP** — internal only. How pods talk. No AWS cost.
- **NodePort** — port on every EC2 node. Rarely used directly.
- **LoadBalancer** — creates one **Classic or NLB** per service. 20 services = 20 LBs = expensive. Use AWS Load Balancer Controller instead.
- **Headless** — DNS returns pod IPs. For StatefulSets (Kafka, Cassandra).

> ⚠️ **TRAP:** Install the **AWS Load Balancer Controller** addon. Without it, `type: LoadBalancer` creates a deprecated Classic Load Balancer. With it, you get ALB (Ingress) or NLB (Service) — both modern and cost-efficient.

---

### ALB Ingress vs NLB

| | ALB (via Ingress) | NLB (via Service) |
|--|--|--|
| **Layer** | L7 HTTP/HTTPS | L4 TCP/UDP |
| **Routing** | Host, path, header-based | IP + port only |
| **TLS termination** | At ALB (ACM cert) | At pod or at NLB |
| **WebSocket/gRPC** | ✅ Supported | ✅ Supported |
| **Use when** | Web apps, APIs, canary routing | Databases, MQTT, raw TCP |

---

### Key Debugging Commands

```bash
kubectl get endpoints svc-name          # FIRST command for connectivity issues. Empty = no pods match selector
kubectl get networkpolicy -n namespace  # Silent drops — NetworkPolicy is invisible by design
kubectl describe hpa                    # Check ScalingActive=False and metrics errors
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller  # ALB issues
eksctl get iamserviceaccount --cluster=<name>  # Debug IRSA bindings
```

---

## SESSION 02: VPC & Networking
### 87% Interview Weight

---

### The 3 Facts Unique to AWS Networking

1. **VPC is REGIONAL, subnets are AZ-scoped.** A VPC spans one region. Each subnet lives in exactly one AZ. To survive AZ failure: 3 subnets across 3 AZs.
2. **Security Groups are STATEFUL. NACLs are STATELESS.** SG: allow inbound → reply traffic automatically allowed. NACL: must explicitly allow BOTH directions.
3. **Security Groups are attached to ENIs (network interfaces), not subnets.** NACLs are attached to subnets. This is how AWS layered defence works.

---

### Security Groups vs NACLs

| | Security Groups | NACLs |
|--|--|--|
| **Stateful?** | ✅ Yes — tracks connections | ❌ No — every packet evaluated |
| **Attached to** | ENI (EC2, RDS, Lambda ENI) | Subnet |
| **Default** | Deny all inbound, allow all outbound | Allow all in both directions |
| **Rules** | Allow only — no explicit deny | Allow AND deny |
| **Evaluation** | All rules evaluated, most permissive wins | Rules evaluated in order by number |
| **Use for** | Most access control | Blocking specific IPs/CIDRs at perimeter |

> ⚠️ **TRAP:** NACLs evaluate rules in **number order, lowest first, first match wins.** Rule 90 DENY before Rule 100 ALLOW = traffic denied. Unlike Security Groups where all rules are evaluated.

> ✅ **GOLDEN RULE:** Security Groups are your primary defence. NACLs are for blocking specific bad IPs at the subnet perimeter. Most teams only use SGs and leave NACLs as default-allow.

---

### NAT Gateway vs VPC Endpoints

| | NAT Gateway | VPC Endpoint (Interface) | VPC Endpoint (Gateway) |
|--|--|--|--|
| **Traffic to** | All internet destinations | Specific AWS services (via PrivateLink) | S3, DynamoDB only |
| **Cost** | $0.045/hr + data transfer | Per-hour + per-GB | FREE |
| **Stays on AWS network?** | ❌ Exits to internet | ✅ Never leaves AWS | ✅ Never leaves AWS |
| **Config** | Per AZ (create one per AZ!) | Per subnet + SG | Route table entry |
| **Use for** | apt-get, external APIs | Secrets Manager, ECR, SSM, KMS | S3, DynamoDB access from private subnets |

> ⚠️ **TRAP:** NAT Gateway is **per-AZ.** One NAT Gateway in us-east-1a covers only that AZ. If your private subnet in us-east-1b routes through it → single point of failure AND cross-AZ data transfer costs. Create one NAT Gateway per AZ in production.

> ✅ **GOLDEN RULE:** Private subnets accessing S3 or DynamoDB → **always use Gateway Endpoints.** Free, stays on AWS backbone, no NAT Gateway data charges. Forgetting this is the single biggest unnecessary AWS networking cost.

---

### VPC Peering vs Transit Gateway vs PrivateLink

| | VPC Peering | Transit Gateway | PrivateLink |
|--|--|--|--|
| **Connects** | 2 VPCs | Many VPCs + on-prem via single hub | Producer service to consumer VPC |
| **Transitive?** | ❌ Non-transitive | ✅ Transitive by design | N/A |
| **CIDR overlap?** | ❌ Cannot overlap | ❌ Cannot overlap | ✅ CIDR irrelevant |
| **Cost** | Per-GB data transfer | Per attachment + per-GB | Per endpoint + per-GB |
| **Use when** | 2-3 VPCs, simple | 5+ VPCs or hybrid cloud | Expose SaaS service to customers |

> ⚠️ **TRAP:** VPC Peering is **not transitive.** A peers to B, B peers to C does NOT mean A can reach C. At 10 VPCs, peering becomes a mesh nightmare. Transit Gateway solves this — it's the hub.

---

### Direct Connect vs Site-to-Site VPN

| | Site-to-Site VPN | Direct Connect |
|--|--|--|
| **Medium** | Encrypted tunnel over internet | Dedicated fibre from your DC |
| **Bandwidth** | Up to 1.25 Gbps | 1–100 Gbps |
| **Latency** | Variable (internet) | Consistent, low |
| **Setup time** | Minutes | Weeks to months |
| **Cost** | Cheap | Expensive |
| **Use when** | Quick hybrid setup, backup path | Production hybrid, consistent latency |

> ✅ **GOLDEN RULE:** Direct Connect + VPN in parallel = best practice. Direct Connect for normal traffic (consistent latency), VPN as failover if DX goes down.

---

### Route53 Routing Policies — Know All 6

- **Simple** — one record, one or more IPs, random selection. No health checks.
- **Weighted** — split 80/20 between two targets. Canary deployments.
- **Latency** — route to the region with lowest latency for that user. Multi-region active/active.
- **Failover** — primary/secondary. Route53 health checks primary; on failure, switches to secondary.
- **Geolocation** — route by user's country/continent. Compliance (data residency), localisation.
- **Geoproximity** — route by geographic distance to resource. Bias control to shift traffic.

> ⚠️ **TRAP: Geolocation ≠ Geoproximity.** Geolocation routes by the user's **country/continent** — static rules. Geoproximity routes by **physical distance** with a bias slider. If asked about GDPR data residency → Geolocation (country-level control).

---

## SESSION 03: IAM
### 85% Interview Weight

> ✅ **GOLDEN RULE: IAM = WHO (principal) can do WHAT (action) on WHICH (resource) under WHAT CONDITIONS (condition block).** Every IAM question is a variation of these four things.

---

### Policy Types — Evaluation Order Matters

1. **Service Control Policies (SCPs)** — AWS Organizations. Maximum boundary for the entire account. Cannot grant permissions — can only restrict. Even root cannot bypass SCPs.
2. **Permission Boundaries** — maximum permissions for a specific IAM entity (user/role). Actual permissions = intersection of boundary + identity policy.
3. **Identity Policies** — attached to users, groups, roles. What they're allowed to do.
4. **Resource Policies** — attached to S3 buckets, KMS keys, SQS queues. Who can access the resource.
5. **Session Policies** — passed when assuming a role via CLI. Temporary restriction for that session.

> ⚠️ **TRAP:** **Deny always wins.** If ANY policy in the chain has an explicit Deny, access is denied regardless of Allow statements elsewhere. Implicit deny (no statement) also denies. The only exception: explicit Allow in a resource policy can grant cross-account access even without an identity policy Allow.

---

### SCP vs Permission Boundary vs IAM Policy

| | SCP | Permission Boundary | IAM Policy |
|--|--|--|--|
| **Scope** | Entire AWS account/OU | One IAM user or role | One identity |
| **Can grant access?** | ❌ Restrict only | ❌ Restrict only | ✅ Grant AND restrict |
| **Overrides root?** | ✅ Yes | No | No |
| **Use for** | Org-wide guardrails (no region X, no EC2 without tags) | Delegate IAM admin without privilege escalation | Normal access control |

> ✅ **GOLDEN RULE:** SCPs = the org-wide floor. Permission Boundaries = delegate IAM safely. Neither grants access on its own — they only cap what identity policies can grant.

---

### IAM Roles vs IAM Users

**IAM Users:** Long-lived credentials. Access keys that never expire unless you rotate them. Human or service. The CISO nightmare.

**IAM Roles:** Temporary credentials via STS AssumeRole. Automatically rotated. No long-lived keys. Used by EC2 instance profiles, Lambda execution roles, EKS IRSA, cross-account access, SAML federation.

> ⚠️ **TRAP:** The **default IAM role for EC2 is the instance profile role.** ALL processes on that EC2 inherit it. If your app and a rogue dependency run on the same EC2, they share credentials. Use IMDSv2 (`--metadata-options HttpTokens=required`) to make credential theft harder.

> ✅ **GOLDEN RULE:** Never create IAM access keys for services. Always use **IAM roles with instance profiles** (EC2), **execution roles** (Lambda), **IRSA/Pod Identity** (EKS). Keys rotate automatically. You sleep better.

---

### Cross-Account Access — The Pattern

```
Account A (your app)          Account B (data)
IAM Role A ──────────────────> IAM Role B (trust policy allows A's role)
              AssumeRole        Resource policy on S3 bucket
```

Two-step: Role B trust policy must allow Role A to assume it. AND Role A must have permission to call `sts:AssumeRole` on Role B.

> ⚠️ **TRAP:** Forgetting **both sides.** Trust policy on the target role (who can assume me) AND the identity policy on the source (can I call AssumeRole). Miss either one = AccessDenied. No error message tells you which side is broken.

---

### AWS Organizations Key Concepts

- **Management account** — root of the org. Never run workloads here. Only org management.
- **OUs (Organizational Units)** — folders for accounts. Apply SCPs at OU level.
- **SCPs** — apply to all accounts in the OU and below. Don't apply to management account.
- **AWS SSO / IAM Identity Center** — single login for all org accounts. Replaces per-account IAM users.

---

### Key Guardrail Policies (Equivalent to GCP Org Policies)

```json
// Deny all actions outside approved regions
"Condition": { "StringNotEquals": { "aws:RequestedRegion": ["us-east-1", "eu-west-1"] } }

// Require MFA for all actions
"Condition": { "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" } }  // Effect: Deny

// Prevent leaving the organization
"Effect": "Deny", "Action": "organizations:LeaveOrganization"

// Enforce IMDSv2
"Condition": { "StringNotEquals": { "ec2:MetadataHttpTokens": "required" } }
```

---

### CloudTrail — Audit Logs

- **Management events** — always on (in default trail). API calls creating/modifying resources. Free for first trail.
- **Data events** — S3 object-level (GetObject, PutObject), Lambda invocations, DynamoDB item-level. Off by default. Costs extra.
- **Insight events** — unusual API activity detection. Like anomaly detection for your audit log.

> ⚠️ **TRAP:** One CloudTrail trail per region covers that region. Enable **multi-region trail** to capture all regions in one S3 bucket. Without it, an attacker creating resources in us-west-2 is invisible if your trail only covers us-east-1.

---

## SESSION 04: S3 & Storage
### 80% Interview Weight

> ✅ **GOLDEN RULE:** S3 is not a file system. It is an object store. No directories (just key prefixes). No locking. No append. Every "folder" is a lie told to make you feel comfortable. The key IS the path.

---

### Storage Classes — Pick the Right One or Pay

| Class | Access Pattern | Min Duration | Retrieval | Cost |
|--|--|--|--|--|
| **Standard** | Frequent (daily) | None | Instant | Highest storage |
| **Intelligent-Tiering** | Unknown/changing | None | Instant (usually) | Small monitoring fee |
| **Standard-IA** | Infrequent (monthly) | 30 days | Instant | Retrieval fee |
| **One Zone-IA** | Infrequent + AZ loss ok | 30 days | Instant | 20% cheaper than IA |
| **Glacier Instant** | Quarterly | 90 days | Instant | Cheap storage |
| **Glacier Flexible** | Yearly | 90 days | Minutes–hours | Very cheap |
| **Glacier Deep Archive** | Rarely/never | 180 days | 12–48 hours | Cheapest |

> ⚠️ **TRAP:** **Standard-IA has a minimum 30-day charge AND a retrieval fee.** Storing a 1KB object for 1 day in Standard-IA then deleting it costs MORE than Standard because of the minimum duration charge. Use Intelligent-Tiering for unpredictable access patterns.

---

### S3 Security — The Three Layers

**1. Block Public Access (BPA)** — 4 settings, account-level and bucket-level. Enable all 4 for all buckets unless you intentionally serve public content. BPA overrides bucket policies.

**2. Bucket Policies** — resource-based policy. Controls who can access the bucket. Cross-account access requires both bucket policy + identity policy on the other side.

**3. IAM Policies** — identity-based. Your role's permission to access S3. Both must allow for cross-account access.

> ⚠️ **TRAP:** `s3:GetObject` in your IAM policy DOES NOT override an S3 bucket policy that explicitly Denies. Resource policy Deny beats identity policy Allow in cross-account scenarios. Both must say yes.

> ✅ **GOLDEN RULE:** Enable **S3 Block Public Access at the account level** on day 1. It prevents any bucket in the account from becoming accidentally public, even if a bucket policy grants public access.

---

### S3 Versioning, Replication, and Lifecycle

**Versioning:** Every PUT creates a new version. DELETE adds a delete marker (doesn't actually delete). Permanently delete = delete specific version ID. Enable for compliance and accidental deletion protection.

**Cross-Region Replication (CRR):** Asynchronous. Requires versioning on both buckets. Replicates new objects after replication is enabled — not existing objects. For DR, global read performance.

**Same-Region Replication (SRR):** Replicates within same region. For log aggregation, test environment sync.

**Lifecycle Rules:** Automate transitions between storage classes. Expire objects. Delete incomplete multipart uploads. Set on day 1 — without them, storage costs grow silently forever.

---

### S3 Performance

- **Prefix spread = parallel performance.** S3 achieves 3,500 PUT/5,500 GET per second **per prefix.** Old advice was to randomize prefixes. AWS scaled this — now just use logical prefixes.
- **Multipart upload:** Use for objects >100MB. Required for objects >5GB. Faster parallel upload, resume on failure.
- **Transfer Acceleration:** Routes upload through nearest CloudFront edge → AWS backbone. For global users uploading large files.
- **Byte-Range Fetches:** Download specific parts of an object in parallel. Like HTTP Range requests.

---

### S3 Object Lock & Compliance

- **Governance mode** — object can't be deleted/overwritten. Users with `s3:BypassGovernanceRetention` permission CAN override.
- **Compliance mode** — object CANNOT be deleted by ANYONE including root until retention period expires. WORM compliance for regulated industries.
- **Legal hold** — indefinite protection, no retention period. Toggle per object.

> ⚠️ **TRAP:** Object Lock must be enabled **at bucket creation.** You cannot enable it on an existing bucket. Plan for compliance requirements before creating buckets.

---

### EBS vs EFS vs S3

| | EBS | EFS | S3 |
|--|--|--|--|
| **Type** | Block (disk) | File (NFS) | Object |
| **Attached to** | One EC2 (usually) | Many EC2s simultaneously | Any client via HTTPS |
| **Persistence** | Survives instance stop | Persistent, managed NFS | Persistent, 11 9s durability |
| **Cross-AZ?** | Single AZ (except io2 Multi-Attach) | ✅ Multi-AZ | ✅ Global |
| **Use for** | OS disk, databases, low-latency | Shared config files, CMS media | Static assets, backups, data lake |

---

## SESSION 05: RDS & Databases
### 75% Interview Weight

---

### The Database Decision Flowchart

```
Need SQL (ACID transactions)?
├── YES → Need global writes OR >64TB OR 99.999% SLA?
│         ├── YES → Aurora Global Database or DynamoDB Global Tables
│         └── NO  → RDS Multi-AZ (Aurora or PostgreSQL/MySQL)
└── NO  → Need flexible schema / document model?
          ├── YES → DynamoDB
          └── NO  → Need microsecond latency?
                    ├── YES → ElastiCache (Redis)
                    └── NO  → Need search? → OpenSearch
```

---

### RDS vs Aurora

| | RDS (MySQL/PostgreSQL) | Aurora |
|--|--|--|
| **Storage** | Instance-attached EBS | Shared cluster volume, 6-way replicated |
| **Read replicas** | Up to 5, async replication | Up to 15, near-zero lag |
| **Failover time** | 60–120 seconds | ~30 seconds (Aurora) |
| **Storage auto-scale** | Manual | Automatic up to 128TB |
| **Cost** | Baseline | ~20% more, but better performance |
| **Writer endpoints** | One endpoint per instance | Cluster endpoint (auto-routes to writer) |
| **Multi-AZ** | Synchronous standby | Storage is inherently multi-AZ |

> ⚠️ **TRAP:** RDS Multi-AZ standby is **NOT a read replica.** It's a warm standby that only accepts traffic on failover. If you want to offload read traffic, you need a **read replica** — which is a separate endpoint. Two different features solving two different problems.

---

### DynamoDB — The Core Facts

**Data model:** Tables → Items (rows) → Attributes (columns). Each item needs a **Partition Key** (+ optional Sort Key). No schema — each item can have different attributes.

**Capacity modes:**
- **On-demand:** Pay per request. No capacity planning. Instantly handles any traffic. More expensive per request.
- **Provisioned:** Specify RCUs and WCUs. Cheaper at predictable load. Add **Auto Scaling** to handle spikes. Use **DAX** (DynamoDB Accelerator) for microsecond reads.

> ⚠️ **TRAP:** Hot partition = one partition key receiving >3,000 RCU or 1,000 WCU/sec. Everything queued. Symptom: `ProvisionedThroughputExceededException` on ONE item type. Fix: composite partition key (user_id + date), jitter writes, or use a write-sharding pattern.

> ✅ **GOLDEN RULE:** **Design your access patterns first, then your DynamoDB schema.** The inverse of relational design. The Partition Key is everything. Bad key = bad performance forever. You can't add an index to fix fundamental key design mistakes cheaply.

---

### ElastiCache — Redis vs Memcached

| | Redis | Memcached |
|--|--|--|
| **Data structures** | Strings, lists, sets, sorted sets, hashes | Strings only |
| **Persistence** | ✅ AOF/RDB snapshots | ❌ No |
| **Replication** | ✅ Cluster mode, read replicas | ❌ No |
| **Multi-AZ** | ✅ Yes | ❌ No |
| **Use for** | Sessions, leaderboards, pub/sub, rate limiting | Simple caching, horizontal scale |

> ⚠️ **TRAP: Memorystore/ElastiCache is NEVER primary storage.** It is a cache in front of your database. If Redis crashes without a backing store, you lost data. Always have a durable source of truth that the cache sits in front of.

---

### Aurora Serverless v2

Pay per ACU (Aurora Capacity Unit). Scales from 0.5 ACU to 128 ACU in seconds. Cost: idle at minimum ACUs. No more over-provisioning for dev/test environments that run 8 hours a day.

> ⚠️ **TRAP:** Aurora Serverless v2 does NOT scale to zero (minimum 0.5 ACU = ~$43/month). v1 could scale to zero but was slow to resume. v2 is fast but always costs something. For true zero-cost idle: use RDS snapshot + restore automation or just accept the ~$43.

---

## SESSION 06: SQS / SNS / EventBridge
### 70% Interview Weight

---

### SQS — Standard vs FIFO

| | Standard Queue | FIFO Queue |
|--|--|--|
| **Ordering** | Best-effort (not guaranteed) | Strict first-in-first-out |
| **Delivery** | At-least-once (possible duplicates) | Exactly-once (deduplication built-in) |
| **Throughput** | Unlimited | 3,000 msg/sec with batching |
| **Use for** | High throughput, order doesn't matter | Order matters (payments, inventory) |
| **Deduplication** | Your consumer must handle | 5-min deduplication window |

> ⚠️ **TRAP:** Standard queues deliver messages **at-least-once** — duplicates WILL happen. Your consumer MUST be **idempotent** (processing same message twice = same result). Don't charge a credit card twice. Don't decrement stock twice. Use a deduplication ID in your database.

---

### Visibility Timeout, DLQ, and Long Polling

**Visibility Timeout:** When a consumer picks up a message, it's hidden from others for X seconds (default 30s, max 12hr). If consumer crashes without ACKing, message becomes visible again → redelivery. Extend timeout with `ChangeMessageVisibility` for long-processing jobs.

**Dead Letter Queue (DLQ):** After N failed delivery attempts, message moves to DLQ automatically. Set `maxReceiveCount`. Monitor DLQ depth — a spike means your consumer is broken.

**Long Polling:** `ReceiveMessage` waits up to 20 seconds for a message. Reduces empty responses (billing). Use `WaitTimeSeconds=20` always. Short polling (default) hammers the API and burns money.

> ⚠️ **TRAP:** DLQ must be the **same type** as source queue (FIFO DLQ for FIFO queue, Standard DLQ for Standard queue). Mixing types = CloudFormation errors and midnight debugging sessions.

---

### SNS vs SQS vs EventBridge

| | SQS | SNS | EventBridge |
|--|--|--|--|
| **Pattern** | Queue (pull) | Fan-out pub/sub (push) | Event bus (rule-based routing) |
| **Persistence** | ✅ Messages stored until consumed | ❌ Fire and forget | ❌ Fire and forget |
| **Filtering** | Consumer polls + processes | Subscription filter policies | Rule patterns (JSON matching) |
| **Subscribers** | One consumer group | Multiple: Lambda, SQS, HTTP, email | 20+ AWS services as targets |
| **Use for** | Work queues, decoupling | Notify many subscribers at once | AWS service events, scheduled tasks, SaaS events |

> ✅ **GOLDEN RULE: SNS → SQS fan-out pattern.** One SNS topic. Multiple SQS queues subscribed. Each SQS queue has its own consumer service. Publisher writes once. All consumers get a copy independently. Adding a 6th consumer = zero code change in publisher. This is the backbone of AWS event-driven architectures.

---

### EventBridge Key Concepts

- **Default event bus** — receives all AWS service events (EC2 state changes, S3 events, RDS events)
- **Custom event bus** — for your own application events
- **Rules** — JSON pattern matching on event content → route to target
- **Scheduler** — cron or rate-based triggers (replaces cron EC2 instances and CloudWatch Events)
- **Pipes** — point-to-point connection between source (SQS, DynamoDB Streams, Kinesis) and target with optional enrichment

> ⚠️ **TRAP:** EventBridge has **at-least-once delivery** and can deliver events **out of order.** If you need ordered processing, consume from the source (SQS FIFO, Kinesis) directly — don't route through EventBridge and expect ordering.

---

### Kinesis vs SQS

| | Kinesis Data Streams | SQS |
|--|--|--|
| **Ordering** | Per-shard ordering | FIFO only in FIFO queues |
| **Replay** | ✅ Up to 7 days | ❌ Consumed = gone |
| **Multiple consumers** | ✅ Same data, different consumers | ❌ One message consumed by one consumer |
| **Throughput** | 1MB/sec write per shard | Unlimited |
| **Use for** | Analytics, real-time processing, replay | Work queues, task distribution |

---

## SESSION 07: Compute — EC2 & Lambda
### 65% Interview Weight

---

### The Compute Decision

| | EC2 | ECS Fargate | Lambda | App Runner |
|--|--|--|--|--|
| **Manage servers?** | ✅ You manage | ❌ Serverless | ❌ Serverless | ❌ Serverless |
| **Max duration** | Unlimited | Unlimited | 15 minutes | Unlimited |
| **Cold starts?** | No (always on) | Yes | Yes (100ms–10s) | Yes |
| **Scale to zero?** | ❌ Always billing | ✅ Yes | ✅ Yes | ✅ Yes |
| **GPU/custom hardware** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Use when** | Legacy, GPU, full control | Containerized, no K8s complexity | Events, <15min, variable load | Simplest container deploy |

---

### EC2 Pricing Models — This Saves or Costs Fortunes

| Model | Discount | Commitment | Use for |
|--|--|--|--|
| **On-Demand** | 0% | None | Dev/test, unpredictable, short-term |
| **Reserved (Standard)** | Up to 72% | 1 or 3 years | Steady-state production baseline |
| **Reserved (Convertible)** | Up to 66% | 1 or 3 years | Steady-state but might change instance type |
| **Savings Plans (Compute)** | Up to 66% | 1 or 3 years | Flexible — covers EC2, Fargate, Lambda |
| **Spot** | Up to 90% | None | Batch, CI/CD, stateless, fault-tolerant |

> ⚠️ **TRAP:** Spot instances receive a **2-minute termination notice.** Not 30 seconds like GCP. Design stateful spot workloads (ML training) to checkpoint every 1 minute so at most 1 minute of work is lost. Never use Spot for RDS, EFS, or anything that can't handle sudden death.

> ✅ **GOLDEN RULE:** Compute Savings Plans are more flexible than Reserved Instances. They apply to EC2 (any instance type/region), Fargate, and Lambda automatically. The default recommendation: Savings Plan over Reserved Instances unless you need specific Reserved pricing benefits.

---

### EC2 Instance Families

- **T** (t3, t4g) — burstable. Cheap. Dev/test, low-traffic web. CPU credits. Hit sustained high CPU = throttled.
- **M** (m6i, m7g) — balanced. General purpose web/app servers. Sweet spot for production.
- **C** (c6i, c7g) — compute-optimized. High CPU. Gaming servers, HPC, CPU-heavy batch.
- **R** (r6i, r7g) — memory-optimized. High RAM. Databases, caches, SAP.
- **I** (i3, i4i) — storage-optimized. NVMe SSD. Databases that need local fast storage.
- **P/G** (p4d, g5) — GPU. ML training, graphics rendering, CUDA.
- **Graviton** (g suffix: m7g, c7g) — ARM, 20% better price-performance. Use unless you need x86-specific software.

> ✅ **GOLDEN RULE:** Use **Graviton instances** (ARM) for anything that doesn't require x86. Java, Python, Node.js, Go, containers all run on Graviton. 20% cheaper, 40% better performance per watt. Interviewers love this answer.

---

### Lambda — Key Facts

**Memory:** 128MB to 10,240MB. CPU allocated proportionally to memory. Need more CPU? Raise memory.

**Execution:** Up to 15 minutes max. Beyond that: use Step Functions, ECS Fargate, or Batch.

**Concurrency:** Default 1,000 per region (soft limit, can raise). Each invocation = one concurrent execution. 10,000 simultaneous requests = 10,000 Lambda instances.

**Cold starts:** First invocation after idle period starts a new container. Java: 5-15s cold start. Python/Node: <500ms. Fix: Provisioned Concurrency (keeps N instances warm), SnapStart (Java only).

> ⚠️ **TRAP:** Lambda's **15-minute timeout** is a hard limit. A video transcoding job that takes 20 minutes will fail at 15 minutes and you lose all work. Use ECS Fargate Tasks or Batch for long-running jobs. Lambda is for short, event-driven work.

---

### Auto Scaling Groups — The Policies

- **Target Tracking:** Maintain a metric at X (e.g., "keep CPU at 50%"). AWS handles add/remove. Simplest, use this first.
- **Step Scaling:** Define ranges: if CPU 60-80% add 1, if CPU >80% add 3. More aggressive response.
- **Scheduled Scaling:** Scale out at 8am, scale in at 8pm. For predictable load patterns.
- **Predictive Scaling:** ML-driven forecast based on historical patterns. Pre-scales before traffic hits.

> ⚠️ **TRAP:** ASG scale-in has a **cooldown period** (default 300s). After scaling out, ASG waits 5 minutes before considering scale-in. During a traffic spike that lasts 4 minutes, you pay for the extra instances 5 minutes after traffic drops. Tune cooldown for your workload's traffic patterns.

---

## SESSION 08: CI/CD — CodePipeline & CDK
### 55% Interview Weight

---

### The AWS CI/CD Stack

```
Code push
    ↓
CodeCommit / GitHub / Bitbucket (source)
    ↓
CodePipeline (orchestrator)
    ↓
CodeBuild (build + test + Docker build)
    ↓
ECR (container registry)
    ↓
CodeDeploy / ECS Deploy / CDK Deploy (deploy)
    ↓
CloudFormation / CDK (infrastructure)
```

---

### CodeBuild — Key Concepts

**buildspec.yml:** The pipeline definition. Lives in your repo root. Phases: `install` → `pre_build` → `build` → `post_build`.

Built-in env vars: `$CODEBUILD_BUILD_ID` · `$CODEBUILD_SOURCE_VERSION` (commit SHA) · `$AWS_DEFAULT_REGION`

**Cache:** S3 cache for dependencies (`node_modules`, Maven `.m2`, pip packages). Without it, every build re-downloads 500MB of dependencies.

> ⚠️ **TRAP:** Never hardcode AWS credentials in buildspec.yml. CodeBuild has an **IAM service role** — grant it the permissions it needs. Credentials are injected automatically via the instance metadata. Hardcoding = secret in logs = 3am incident.

---

### The Three-Pipeline Pattern

**PR opened → CI only:**
- Run lint, unit tests, build Docker image
- Vulnerability scan (ECR scanning or Snyk)
- Security scan (Checkov for IaC, SAST tools)
- NO deploy. Fast feedback in <10 minutes.

**Merge to main → Staging:**
- Build + tag image with commit SHA (`$CODEBUILD_SOURCE_VERSION`)
- Push to ECR
- Deploy to staging ECS/EKS
- Run integration tests and smoke tests

**Git tag v1.2.3 → Production:**
- Pull already-built image (same SHA, no rebuild)
- Deploy via CodeDeploy with canary or blue-green
- CloudWatch alarms monitor error rates
- Auto-rollback if alarm triggers

> ⚠️ **TRAP:** Never tag images as `latest`. Use `$CODEBUILD_SOURCE_VERSION` (commit SHA). `latest` is mutable — you cannot tell what's running in prod. SHA is immutable — full traceability, safe rollback, audit compliance. Same trap as GCP. Same mistake. Every time.

---

### Deployment Strategies — CodeDeploy & ECS

| Strategy | Downtime | Rollback Speed | Cost | Use when |
|--|--|--|--|--|
| **Rolling** | Zero | Slow (pod by pod) | 1× | Most stateless updates |
| **Blue-Green (ECS)** | Zero | Instant (traffic switch) | 2× during switch | Breaking changes, DB migrations |
| **Canary (CodeDeploy)** | Zero | Instant (0% canary weight) | ~1.1× | Default choice — real traffic validation |
| **Linear (CodeDeploy)** | Zero | Instant | ~1.1× | Gradual compliance-sensitive rollouts |

> ✅ **GOLDEN RULE:** Default to **Canary**. Blue-Green only when v1 and v2 absolutely cannot coexist. Same principle as GCP — if services can coexist, canary. If they can't (breaking DB schema, breaking API contract), blue-green.

---

### AWS CDK vs CloudFormation vs Terraform

| | CloudFormation | CDK | Terraform |
|--|--|--|--|
| **Language** | YAML/JSON | TypeScript, Python, Java, Go | HCL |
| **Learning curve** | Low | Medium | Medium |
| **Type safety** | ❌ No | ✅ Yes | Partial |
| **Multi-cloud** | ❌ AWS only | ❌ AWS only | ✅ Multi-cloud |
| **State management** | AWS-managed | AWS-managed (via CFN) | S3 + DynamoDB lock |
| **Ecosystem** | AWS native | AWS native, compiles to CFN | Largest provider ecosystem |
| **Use when** | Simple, pure AWS, teams resist code | AWS-only, engineers prefer real languages | Multi-cloud or existing Terraform footprint |

> ✅ **GOLDEN RULE: CDK over CloudFormation** for any new AWS-only infrastructure. Real programming languages = loops, conditionals, reusable constructs, unit tests for infrastructure. YAML cannot do any of this without 400 lines of duct tape.

---

### CDK Key Concepts

- **App** → one or more **Stacks** → **Constructs** (L1/L2/L3)
- **L1 Constructs** — raw CloudFormation resources (`CfnBucket`). 1:1 mapping.
- **L2 Constructs** — opinionated wrappers with defaults (`s3.Bucket`). Use these.
- **L3 Constructs (Patterns)** — multi-resource patterns (`ecs_patterns.ApplicationLoadBalancedFargateService`). One line deploys entire architecture.
- **CDK Diff** — like `terraform plan`. Shows changes before applying.
- **CDK Bootstrap** — creates S3 bucket + ECR repo + IAM roles needed for CDK deploys. Run once per account+region.

> ⚠️ **TRAP:** CDK **synthesizes CloudFormation** — what you deploy is CloudFormation. CloudFormation limits still apply: 500 resources per stack, 200 outputs. Hit limits? Split into nested stacks or multiple stacks.

---

### Secrets Management

**AWS Secrets Manager:** Stores secrets, auto-rotates on schedule via Lambda. Costs $0.40/secret/month. Use for database passwords, API keys.

**AWS Systems Manager Parameter Store:** Free tier for standard parameters. SecureString encrypted with KMS. Use for config values, feature flags, non-rotating secrets.

> ⚠️ **TRAP: Never put secrets in environment variables hardcoded in task definitions, Lambda configs, or buildspec.yml.** They appear in CloudTrail, ECS task descriptions, and Lambda config — readable by anyone with DescribeTaskDefinition. Always pull from Secrets Manager at runtime, or use ECS secrets integration (pulls at container start).

---

## MASTER CHEATSHEET — Read This Last Every Day

---

### Decision Rules — Know These Instinctively

| If you need... | Answer |
|--|--|
| GCP API access from EKS pod | IRSA or EKS Pod Identity (no credential files ever) |
| External system → AWS access | IAM Identity Federation / OIDC (GitHub Actions, etc.) |
| SQL + global scale | Aurora Global Database |
| SQL + single region | RDS Aurora (or PostgreSQL) — 10× cheaper than global |
| Real-time mobile/web sync | AppSync (GraphQL) + DynamoDB |
| Petabyte analytics | Redshift (data warehouse) + S3 data lake |
| Event fan-out to many consumers | SNS → SQS (per consumer) |
| Ordered message processing | SQS FIFO or Kinesis (per-shard ordering) |
| Replay events | Kinesis (7-day window). SQS cannot replay. |
| Serverless containers | ECS Fargate or App Runner |
| Simplest serverless HTTP | Lambda + API Gateway or Lambda Function URL |
| Batch job replacing a cron EC2 | AWS Batch or Step Functions + Lambda |
| DB schema change + zero downtime | Expand/Contract pattern → Canary deploy |
| Breaking schema change | Blue-Green. Full environment swap. |
| Default deployment strategy | Canary. Always. Real traffic validation. |
| Cheap compute, fault-tolerant batch | Spot Instances. Up to 90% off. |
| Best price-performance EC2 | Graviton (ARM). 20% cheaper. Use unless x86-required. |
| Committed steady-state workloads | Compute Savings Plans (more flexible than Reserved) |
| S3 from private subnet | Gateway VPC Endpoint — FREE |
| Other AWS service from private subnet | Interface VPC Endpoint (PrivateLink) |
| Connect 5+ VPCs | Transit Gateway. Peering doesn't scale. |
| Secrets in containers | Secrets Manager. Never env vars. |

---

### Interview Traps — The Ones That Actually Get People

| Tag | Trap |
|--|--|
| **S3** | `SELECT *` equivalent = don't download entire objects when you need one field. Use S3 Select or Athena. |
| **S3** | Standard-IA has 30-day minimum charge + retrieval fee. Don't use for short-lived or frequently-accessed objects. |
| **EKS** | HPA + VPA Auto = infinite conflict. VPA in Recommendation mode only. |
| **EKS** | No AWS Load Balancer Controller = Classic LB on `type: LoadBalancer`. Always install the controller. |
| **SQS** | Standard queue delivers at-least-once. Your consumer MUST be idempotent. |
| **SQS** | NACK equivalent (not deleting message) on permanent error = infinite retry. Move to DLQ. |
| **SQS** | Short polling by default = empty responses, wasted money. Always use `WaitTimeSeconds=20`. |
| **VPC** | NACLs are stateless. MUST allow return traffic explicitly. SGs handle this automatically. |
| **VPC** | NACL rules evaluated in number order, first match wins. NACLs are not like Security Groups. |
| **VPC** | NAT Gateway is per-AZ. One NAT Gateway = single point of failure AND cross-AZ charges. |
| **IAM** | Explicit Deny in any policy = denied. Period. No Allow anywhere overrides it. |
| **IAM** | SCPs don't apply to management account. |
| **IAM** | Cross-account AssumeRole requires BOTH trust policy (target) AND identity policy (source). |
| **RDS** | Multi-AZ standby is NOT readable. Read Replica IS readable. Two different things. |
| **Lambda** | 15-minute hard limit. Long jobs → ECS Fargate. |
| **Lambda** | Memory allocation controls CPU allocation. Slow Lambda → increase memory first. |
| **EC2** | Spot = 2-minute warning. Not 30 seconds. Design checkpointing accordingly. |
| **CDK** | CDK synthesizes CloudFormation. CloudFormation limits (500 resources/stack) still apply. |
| **CI/CD** | Never tag images as `latest`. Commit SHA only. Immutability is non-negotiable. |
| **Kinesis** | VPC** Peering is not transitive. A–B + B–C ≠ A–C. Transit Gateway solves this. |

---

### Architecture Patterns — Drop These In Any Design Round

**Multi-region active/active API:**
Route53 Latency → ALB → ECS Fargate (per region) → Aurora Global (one writer, regional readers) → ElastiCache per region

**Secure microservices on EKS:**
3 namespaces + default-deny NetworkPolicy + IRSA/Pod Identity per service + Secrets Manager sidecar + NLB → ALB Ingress

**Event-driven data pipeline:**
S3 Event → SNS → SQS (per consumer) → Lambda/Fargate processors → Redshift/DynamoDB

**Cost-optimised batch processing:**
SQS queue → ASG Spot Fleet → process → S3 → Athena for queries. Pay for execution only.

**CI/CD pipeline:**
GitHub PR → CodeBuild (lint/test/scan) → merge → CodeBuild (build/push ECR) → CodeDeploy canary → CloudWatch alarm → auto-rollback

**Zero-trust fintech:**
AWS Organizations + SCPs (region lock, no public S3) + no NAT (only VPC Endpoints) + Secrets Manager rotation + CloudTrail data events enabled + GuardDuty + Security Hub

---

> ✅ **FINAL RULE: Depth beats breadth.** One topic known cold beats five topics known superficially. EKS + VPC + IAM appear together in every architecture round. Study them as a system. You have done that. Read this again tomorrow.

---

*AWS Mid-Level Engineer · Daily Read · 8 Sessions · All Traps · All Golden Rules*
