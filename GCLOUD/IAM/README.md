# GCP IAM

Giving someone permissions in IAM involves the following three components:

- **Principal**: The identity of the person or system that you want to give permissions to.
- **Role**: The collection of permissions that you want to give the principal.
- **Resource**: The Google Cloud resource that you want to let the principal access.

# PRINCIPAL

→ Principals represent one or more identities that have authenticated to Google Cloud.

→ In Google Cloud you control access for *principals*.

NOTE : In the past, principals were referred to as *members*. Some APIs still use that term.

There are various types of principals in IAM, but they can be divided into two broad categories:

- **Human users**: 

→ Google Accounts, 

→ Google groups, and

-  **Workloads :**

→ service accounts

# GCP Cloud IAM — Overview Notes

> Topic: Identity and Access Management (IAM)
> 

---

## 1. Principle of Least Privilege

> A user, program, or process should have **only the minimum access** needed to do its job — nothing more.
> 

**Example:**
A user who only needs to upload files to a Cloud Storage bucket should have:

- ✅ `storage.objects.create`
- ❌ `storage.objects.get`
- ❌ `storage.objects.delete`
- ❌ `storage.objects.list`

---

## 2. Policy Architecture

A **Policy** is a collection of statements that define who has what type of access.

- It is **attached to a resource** and enforced every time that resource is accessed.

| Component | Description |
| --- | --- |
| **Policy** | Collection of statements defining who has what access |
| **Binding** | Links one or more members to a single role + optional conditions |
| **Metadata** | Contains `etag` (concurrency control) and `version` (schema version) |
| **Audit Config** | Configures audit logging for the policy |

---

## 3. Members — "The Who"

Members are the identities that can be granted access.

| Member Type | Description |
| --- | --- |
| **Google Account** | Any email tied to a Google account (gmail or other domain) |
| **Service Account** | An account for an app/service, not a human |
| **Google Groups** | A named collection of Google Accounts + service accounts |
| **G Suite Domain** | All accounts in an org's G Suite account |
| **Cloud Identity Domain** | Like G Suite, but without G Suite apps/features |
| **AllAuthenticatedUsers** | All service accounts + all internet users authenticated with Google |
| **AllUsers** | Literally everyone on the internet (authenticated + unauthenticated) |

> ⚠️ `AllUsers` = **public access**. Use only if you intentionally want the whole internet to access the resource.
> 

---

## 4. Permissions

- Defines **what operations are allowed** on a resource.
- Each permission maps **1-to-1 with a REST API method**.
- ⚠️ **You cannot grant a permission directly to a user.**
- You must grant a **Role**, which contains one or more permissions.

### Permission Format

```
compute . instances . list
───────   ─────────   ────
service    resource    verb
```

---

## 5. Roles

A **Role** is a collection of permissions.
You assign a role to a user → the user gets all permissions inside that role.

### 3 Types of Roles

| Type | Description | When to Use |
| --- | --- | --- |
| **Primitive** | Old broad roles: `Owner`, `Editor`, `Viewer` | ❌ Avoid — too permissive, violates least privilege |
| **Predefined** | Fine-grained roles created by Google (e.g. `roles/compute.viewer`) | ✅ Best for most cases |
| **Custom** | You define exactly which permissions go in the role | ✅ Most flexible, but needs manual maintenance |

---

## 6. Role Launch Stages

| Stage | Meaning |
| --- | --- |
| **alpha** | Still in testing — may change or be removed |
| **beta** | Tested, awaiting final approval |
| **ga** | Generally available — production-ready |

---

## 7. Conditions

Conditions add **attribute-based rules** to a binding.

- Access is granted **only if the condition evaluates to `true`**.
- Without a condition → access is always granted (when role matches).

### Example Condition Use Cases

- **Time-based:** Allow access only during business hours
- **Resource-based:** Allow access only to resources with a specific tag
- **Request-based:** Allow access only from a specific IP range

---

## 8. Metadata

| Field | Purpose |
| --- | --- |
| **etag** | Prevents race conditions when multiple people update a policy at the same time. IAM uses this for concurrency control. |
| **version** | Specifies the policy schema version. New versions are introduced to avoid breaking existing integrations on feature releases. |

---

## 9. Audit Config

Part of a Policy. Determines:

- Which **permission types** are logged
- Which **identities** (if any) are **exempt** from logging

> Basically the surveillance camera settings for your GCP resources.
> 

---

## 10. Resource Hierarchy & Policy Inheritance

GCP organizes all resources in a tree. **Policies set at a higher level are inherited downward.**

```
Domain (cloud level)
    └── Organization (root node)
            └── Folders (grouping + isolation boundary)
                    └── Projects (core organizational component)
                            └── Resources (VMs, Storage, etc.)
```

### Key Points

- A policy set on **Organization** → inherited by all folders, projects, and resources below it.
- A policy set on a **Project** → inherited by all resources in that project.
- **More permissive policy wins** — if a parent gives more access than a child policy, the parent's access applies.

---

# 

# GCP IAM — Policies and Conditions

> Topic: Policy Statements, Versions, Limitations, Conditions & Audit Logs
> 

---

## 1. Policy Statement

### What is it?

A **Policy Statement** is the actual JSON or YAML document that GCP stores to know "who can do what on which resource."

> 🧠 Simple English: It's like a rules file. GCP reads this file every time someone tries to access a resource and decides — allowed or denied.
> 

### Structure of a Policy (JSON)

```json
{
  "bindings": [
    {
      "role": "roles/storage.admin",
      "members": [
        "user:tonybowtieace@gmail.com"
      ]
    },
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:larkfederlagen@gmail.com"
      ],
      "condition": {
        "title": "Expires_January_1_2021",
        "description": "Do not grant access after Jan 2021",
        "expression": "request.time < timestamp('2021-01-01T00:00:00.000Z')"
      }
    }
  ],
  "etag": "BeEEja0YfWJ=",
  "version": 3
}
```

### Same Policy in YAML

```yaml
bindings:
- members:
  - user: tonybowtieace@gmail.com
  role: roles/storage.admin
- members:
  - user: larkfederlagen@gmail.com
  role: roles/storage.objectViewer
  condition:
    title: expirable access
    description: Do not grant access after Jan 2021
    expression: request.time < timestamp('2021-01-01T00:00:00.000Z')
etag: BeEEja0YfWJ=
version: 3
```

> 😅 JSON and YAML are just two ways to write the same thing. Like WhatsApp vs. SMS — same message, different packaging.
> 

---

## 2. Fetching the IAM Policy (gcloud commands)

> 🧠 Simple English: These commands let you **read** the current IAM policy for a project, folder, or organization.
> 

```bash
# Get policy for a project
gcloud projects get-iam-policy <project-id>

# Get policy for a folder
gcloud resource-manager folders get-iam-policy <folder-id>

# Get policy for an organization
gcloud organizations get-iam-policy <organization-id>
```

### Example output from a real project:

```yaml
bindings:
- members:
  - serviceAccount:service-365547584538@compute-system.iam.gserviceaccount.com
  role: roles/compute.serviceAgent
- members:
  - serviceAccount:365547584538@cloudservices.gserviceaccount.com
  - serviceAccount:365547584538-compute@developer.gserviceaccount.com
  role: roles/editor
- members:
  - user:tonybowtieace@gmail.com
  role: roles/owner
etag: BwWtFEMmKL0=
version: 1
```

> 🔍 Notice: GCP automatically creates service accounts for compute and cloud services. You didn't ask for them — they just showed up, like uninvited guests at a wedding.
> 

---

## 3. Policy Versions

> 🧠 Simple English: Policy version tells GCP which "format" the policy is written in. Use the wrong version and your conditions simply won't work.
> 

| Version | When to use |
| --- | --- |
| **1** | Simple policies — no conditions. Basic role bindings only. |
| **3** | When your policy has a **condition** block. Required for conditional access. |

### Version 1 — No condition, simple bindings

```yaml
version: 1
```

### Version 3 — Has condition block

```yaml
condition:
  title: expirable access
  description: Do not grant access after Jan 2021
  expression: request.time < timestamp('2021-01-01T00:00:00.000Z')
version: 3
```

> ⚠️ If you use a condition but leave `version: 1`, the condition is **silently ignored**. No error. No warning. Just quietly ignored — like your PRs on Friday afternoon.
> 

---

## 4. Policy Limitations

> 🧠 Simple English: GCP policies have hard limits. If you hit them, you'll need to rethink your architecture.
> 

| Limit | Value |
| --- | --- |
| Policies per resource | **1** (only one policy per org/folder/project/resource) |
| Members per policy | **1,500** members OR **250** Google Groups |
| Propagation time | Up to **7 minutes** for changes to apply everywhere |
| Conditional role bindings per policy | **100** max |

> 🕐 That 7-minute propagation delay is why you shouldn't test IAM changes in production during an incident. Not that anyone has ever done that. Definitely not.
> 

---

## 5. Conditions — In Detail

### What are Conditions?

Conditions let you add **extra rules** to a role binding. Access is only granted if the condition evaluates to **true**.

> 🧠 Simple English: It's like saying "Yes, you have the key — but you can only come in during business hours, from our office IP, and only to the dev server." Much better than handing someone a master key.
> 

### Two types of condition attributes:

| Type | Based on |
| --- | --- |
| **Resource-based** | The resource itself (type, name, tags) |
| **Request-based** | Details about the request (timestamp, IP address) |

---

## 6. Time-Based Conditions

> 🧠 Simple English: Give someone access only during certain hours or days. Great for contractors, interns, or that one colleague you don't fully trust yet.
> 

```yaml
condition:
  title: Business_hours_access
  description: Business hours access Monday-Friday
  expression: >
    request.time.getHours("America/Toronto") >= 9 &&
    request.time.getHours("America/Toronto") <= 17 &&
    request.time.getDayOfWeek("America/Toronto") >= 1 &&
    request.time.getDayOfWeek("America/Toronto") <= 5
```

### Breakdown:

- `getHours()` — checks the hour of the request (9 = 9am, 17 = 5pm)
- `getDayOfWeek()` — 0 = Sunday, 1 = Monday ... 5 = Friday, 6 = Saturday
- Always specify a **timezone** (e.g., `"America/Toronto"`)

> 🕰️ Without a timezone, the condition uses UTC. Your "business hours" restriction might accidentally block access at 9am local time. UTC doesn't care about your workday.
> 

---

## 7. Resource-Based Conditions

> 🧠 Simple English: Limit access based on *which* resource is being accessed — its type, name, or path. Perfect for "devs can only touch dev resources."
> 

```yaml
condition:
  title: Dev_only_access
  description: Only access to development* VMs
  expression: >
    (resource.type == 'compute.googleapis.com/Disk' &&
     resource.name.startsWith('projects/project-cat-bowties/regions/us-central1/disks/development'))
    ||
    (resource.type == 'compute.googleapis.com/Instance' &&
     resource.name.startsWith('projects/project-cat-bowties/zones/us-central1-a/instances/development'))
    ||
    (resource.type != 'compute.googleapis.com/Instance' &&
     resource.type != 'compute.googleapis.com/Disk')
```

### Key attributes:

- `resource.type` — the type of resource (e.g., `compute.googleapis.com/Instance`)
- `resource.name` — the full path/name of the resource
- `startsWith()` — matches resources whose name starts with a given prefix

> 💀 That last OR block is important: it allows access to resources that are *neither* Instances nor Disks. Without it, the condition would deny access to things like Cloud Storage buckets entirely. Missing that line = unintentionally locking yourself out. Classic.
> 

---

## 8. Condition Limitations

> 🧠 Simple English: Conditions are powerful but have rules about where and how you can use them.
> 

| Limitation | Details |
| --- | --- |
| **Limited to specific services** | Not all GCP services support conditions — check docs first |
| **Primitive roles unsupported** | You cannot use conditions with Owner, Editor, or Viewer roles |
| **No AllUsers / AllAuthenticatedUsers** | Cannot apply conditions to "everyone on the internet" |
| **Max 100 conditional role bindings** | Per policy |
| **Max 20 bindings** | For the same role + same member combination |

> 🤦 So conditions don't work with primitive roles — another reason to avoid Owner/Editor/Viewer. As if "too permissive" wasn't enough of a reason.
> 

---

## 9. AuditConfig Logs

### What is it?

`auditConfigs` is a section inside the Policy that controls **what actions get logged** and **who is exempt** from logging.

> 🧠 Simple English: It's the GCP version of CCTV. You decide which cameras are on, what they record, and which VIPs get to walk past without being filmed.
> 

### Three log types:

| Log Type | What it records |
| --- | --- |
| `DATA_READ` | Someone read data (e.g., downloaded a file) |
| `DATA_WRITE` | Someone wrote/modified data |
| `ADMIN_READ` | Someone read admin/config info (e.g., viewed IAM policies) |

### Example YAML:

```yaml
auditConfigs:
- auditLogConfigs:
  - logType: DATA_READ
  - logType: ADMIN_READ
  - logType: DATA_WRITE
  service: allServices          # applies to ALL GCP services
                             # or "compute.googleapis.com" insted of allServices
- auditLogConfigs:
  - exemptedMembers:
    - tonybowtieace@gmail.com   # this person won't be logged
    logType: ADMIN_READ
  service: storage.googleapis.com   # only for Cloud Storage
```

## ✅ Recommended Audit Config (Balanced + Safe)

```bash
auditConfigs:
# 1. Minimal logging for all services (safe baseline)
- service: allServices
  auditLogConfigs:
  - logType: ADMIN_READ

# 2. Detailed logging for VMs (Compute Engine)
- service: compute.googleapis.com
  auditLogConfigs:
  - logType: DATA_WRITE

# 3. (Optional but recommended) Detailed logging for IAM changes
- service: iam.googleapis.com
  auditLogConfigs:
  - logType: DATA_WRITE
```

### Key points:

- `service: allServices` — applies the log config to every GCP service
- `service: storage.googleapis.com` — applies only to Cloud Storage
- `exemptedMembers` — these identities are **excluded** from that log type
- You can mix and match: log everything for all services, but exempt specific people for specific services

> 🕵️ The fact that you can *exempt* specific members from audit logs is both a legitimate admin tool and a spectacular way to hide things. Use responsibly. GCP knows what you're doing even when audit logs don't.
> 

---

## Quick Reference Cheat Sheet

| Topic | Key Point |
| --- | --- |
| Policy format | JSON or YAML — same thing, different look |
| Fetch policy | `gcloud projects get-iam-policy <project-id>` |
| Version 1 | No conditions |
| Version 3 | Has conditions — must explicitly set `version: 3` |
| Policy limit | 1 per resource, 1500 members, 7 min propagation |
| Condition types | Time-based or resource-based |
| Condition limit | 100 conditional bindings per policy |
| Primitive roles | Cannot use with conditions |
| AllUsers | Cannot use with conditions |
| Log types | DATA_READ, DATA_WRITE, ADMIN_READ |

---

*Source: GCP Associate Cloud Engineer (ACE) prep — exampro.co/gcp-ace*

# GCP — Smart Audit Logging Strategy

> Quick Note | Topic: Using Audit Logs to Catch "The Before" — Not Just "The After"
> 

---

## The Problem with Basic Logging

Most teams only log **what changed**.
But the real story starts *before* the change.

> 🧠 Simple English: Imagine your production database got wiped.
Basic logs tell you: *"Someone deleted the table at 3:14am."*
Smart logs tell you: *"Someone was reading the DB config at 2:58am, checked IAM permissions at 3:02am, then deleted the table at 3:14am."*
> 
> 
> One is a crime scene. The other is the whole story — motive included.
> 

---

## The Two-Layer Strategy

### Layer 1 — ADMIN_READ for allServices (the "who was snooping" layer)

```yaml
- service: allServices
  auditLogConfigs:
  - logType: ADMIN_READ
```

**What it captures:**

- `Who *viewed* configs, permissions, or infrastructure before the outage ??`
- Who was *exploring* the system before making a move  ??
- Silent activity — reads that leave no obvious trace

> 🧠 Simple English: This is your CCTV in the lobby.
It doesn't catch the theft — it catches who was casing the place beforehand.
The first sign of trouble is often someone *reading* things they shouldn't be reading.
> 

**Why allServices?**
Because attackers (and accidental disasters) don't stick to one service.
They look around — Compute, Storage, IAM, Networking — before they strike.
Casting a wide net here costs little but catches a lot.

---

### Layer 2 — DATA_WRITE for IAM (the "who changed the locks" layer)

```yaml
- service: iam.googleapis.com
  auditLogConfigs:
  - logType: DATA_WRITE
```

**What it captures:**

- Who added or removed roles
- Who modified policies or bindings
- Any change to *who has access to what*

> 🧠 Simple English: IAM is the master key system.
If someone changes IAM — they're changing who can open which doors.
`DATA_WRITE` on IAM tells you every time someone touched the locks.
This is the highest-value log in cloud security. Don't skip it.
> 

---

## Together: A Timeline of Intent

```
2:58am  →  ADMIN_READ  (allServices)  →  User read Compute instance configs
3:02am  →  ADMIN_READ  (allServices)  →  User read IAM policy on project
3:09am  →  DATA_WRITE  (IAM)          →  User granted themselves roles/owner
3:14am  →  [incident]                 →  Production data deleted
```

> Without this setup, you'd only see the 3:14am entry.
With it, you see the **full timeline of intent** — not just what happened, but *how they got there*.
> 

---

## Full Config (Copy-Paste Ready)

```yaml
auditConfigs:

  # Layer 1: Minimal baseline — catch who's looking around
  - service: allServices
    auditLogConfigs:
    - logType: ADMIN_READ

  # Layer 2: Detailed IAM logging — catch every permission change
  - service: iam.googleapis.com
    auditLogConfigs:
    - logType: DATA_WRITE
```

## in LOG-EXPLORED : CHECK WHO VIEWED RESOURCE’s

```bash
# QUERY THIS FOR VMs
protoPayload.resourceName="projects/piyush-gcp/global/instances"
logName:"cloudaudit.googleapis.com"

# QUERY THIS FOR FIREWALL
protoPayload.resourceName="projects/piyush-gcp/global/firewalls"
logName:"cloudaudit.googleapis.com"
```

---

## Log Types — Quick Reminder

| Log Type | What it records |
| --- | --- |
| `ADMIN_READ` | Someone *read* config or metadata (viewed policies, listed resources) |
| `DATA_WRITE` | Someone *wrote or changed* data or config (modified IAM, deleted a resource) |
| `DATA_READ (Noisy)` | Someone *read* actual data ( queried a DB, all Kubectl-command ) |

> 💡 `DATA_READ` for allServices is `intentionally left out` here — it's extremely noisy and expensive.
Every file download, every API call, every list operation gets logged.
Only enable it when you're investigating something specific, not as a baseline.
Your billing team will thank you.
> 

---

## When to Use This Pattern

| Scenario | Use this? |
| --- | --- |
| General security baseline for any GCP project | ✅ Yes — always |
| Investigating an incident after the fact | ✅ Yes — already have the trail |
| Compliance audit requiring access history | ✅ Yes — full visibility |
| Debugging who changed a specific IAM binding | ✅ Yes — DATA_WRITE on IAM catches it |
| Logging every single file download | ❌ No — use DATA_READ only when needed |

---

## The Senior's Insight (Worth Framing)

> *"In cloud security, what happens **before** the change matters just as much as the change itself."*
> 

ADMIN_READ = *who was looking*
DATA_WRITE on IAM = *who changed access*

Together = not just a log, but a **story**.

---

> 💀 Teams without this setup open a ticket after an incident asking "who did this?"
Teams with this setup open a ticket and already know — who did it, when they started planning it, and exactly what they touched.
One of these teams sleeps better at night.
> 

---

*Source: Senior engineer insight + GCP ACE prep | auditConfigs best practices*

# GCP — Labels

> Quick Note | Topic: What are Labels and Why Your Senior Won't Stop Talking About Them
> 

---

## What is a Label?

A **Label** is a **key-value pair** that you attach to a GCP resource to organize and identify it.

```
key:   environment
value: production
```

> 🧠 Simple English: Labels are sticky notes you put on your cloud resources.
"This VM belongs to the billing team, runs in production, and is owned by Raj."
Without labels, your resources are just a pile of unnamed VMs and buckets — good luck figuring out which one is costing you $800/month.
> 

---

## Syntax Rules

- Format: `key:value`
- Keys and values: **lowercase letters, numbers, hyphens, underscores**
- Max **64 labels** per resource
- Max key length: **63 characters**
- Max value length: **63 characters**

```
✅  environment:production
✅  team:backend
✅  cost-center:eng-101
❌  Environment:Production   (no uppercase)
❌  my label:yes             (no spaces)
```

---

## Common Label Examples

| Key | Value | Purpose |
| --- | --- | --- |
| `environment` | `production` / `staging` / `dev` | Which env is this? |
| `team` | `backend` / `data` / `infra` | Who owns it? |
| `cost-center` | `eng-101` / `marketing` | Who pays for it? |
| `app` | `payments-service` | Which app does this belong to? |
| `owner` | `raj` / `priya` | Who created it (and who to blame)? |

---

## Why Labels Matter — Real Use Cases

### 1. Cost Tracking 💰

Filter your billing report by label to see exactly how much each team/project/environment is spending.

> Without labels: "We spent ₹2 lakhs on GCP this month."
With labels: "The ML team's GPU VMs spent ₹1.8 lakhs. Have a word with them."
> 

### 2. Access Control with Conditions 🔐

Use labels in IAM Conditions to restrict access to only resources with specific labels.

```
resource.labels["environment"] == "dev"
```

> Devs can only touch resources labeled `environment:dev`. They cannot accidentally nuke production. *Accidentally.*
> 

### 3. Automation & Scripting 🤖

Write scripts that target resources by label — start/stop, backup, or delete in bulk.

```bash
# Stop all VMs labeled environment:dev at night
gcloud compute instances list --filter="labels.environment=dev"
```

### 4. Monitoring & Alerting 📊

Set up dashboards or alerts scoped to a label.
e.g. "Alert me if CPU > 90% on any resource labeled `app:payments-service`"

### 5. Auditing & Compliance 📋

During an audit, quickly identify which resources belong to which system, team, or compliance boundary.

> Auditor: "Show me all resources handling customer data."
You (with labels): *filters by `data-class:pii` in 5 seconds*
You (without labels): *spends 3 days clicking through the console*
> 

---

## Senior's Golden Rule 🏆

> **"Label everything before it's too late."**
> 

Once you have 200+ resources with no labels, untangling ownership and cost is a nightmare.
Labels cost nothing. Unlabeled chaos costs everything.

**Minimum recommended labels for every resource:**

```
environment: production | staging | dev
team:        <your-team-name>
app:         <application-name>
owner:       <person-or-team-responsible>
```

---

## Quick Cheat Sheet

| What | Details |
| --- | --- |
| Format | `key:value` (lowercase only) |
| Max per resource | 64 labels |
| Used for | Cost tracking, access control, automation, monitoring, auditing |
| Applied to | VMs, buckets, disks, databases — most GCP resources |
| Cost | Free — no charge for adding labels |

---

> 💀 Not labeling your resources is like throwing all your files into one folder named "stuff".
Works fine on day 1. Becomes a horror show by day 90.
> 

---

# GCP IAM — Service Accounts

> Topic: Service Accounts — Types, Keys, Permissions, Scopes & Best Practices
⚠️ Some details (especially around default roles and key rotation periods) may change. Always verify with [official GCP docs](https://cloud.google.com/iam/docs/service-accounts).
> 

---

## What is a Service Account?

A **Service Account** is an account for a **program, app, or service** — not a human.

> 🧠 Simple English: When your web app needs to talk to a database, it can't log in with someone's personal Gmail. It uses a Service Account — a robot identity with its own permissions.
> 
> 
> Think of it as a badge you give to your app that says: *"This app is allowed to do X, Y, Z — and absolutely nothing else."*
> 

### Why use it?

- Your app needs to call GCP APIs (Cloud Storage, BigQuery, Pub/Sub, etc.)
- You need a VM in Project B to talk to resources in Project C
- You want to run automated jobs without any human's credentials involved

> 💀 Running production apps with a human account is a disaster waiting to happen. That human quits, gets locked out, or "accidentally" changes their password — and your production system dies with them.
> 

---

## Service Account Types

| Type | Who creates it | When |
| --- | --- | --- |
| **User-managed** | You | When you explicitly create one |
| **Default** | GCP (auto-created) | When you enable certain GCP services |
| **Google-managed** | Google | For internal Google service use |

### 1. User-Managed Service Account ✅ (Recommended for Production)

- You create it, you name it, you control it
- You assign exactly the roles it needs — nothing more
- **Google strongly recommends this for production workloads**

```
Email format:
service-account-name@project-id.iam.gserviceaccount.com
```

> 🧠 Simple English: You build the robot, give it a specific job, and only hand it the tools for that job.
> 

---

### 2. Default Service Account ⚠️ (Risky — Use with Caution)

- **Automatically created** when you enable App Engine or Compute Engine
- Automatically gets the **Editor role** on the project
- Editor role = read + write access to almost everything in the project

> ⚠️ **Editor role is risky for a service account.**
If this service account gets compromised, the attacker has Editor access to your entire project.
Use it for quick experiments only. Never in production.
> 

```
App Engine default:
project-id@appspot.gserviceaccount.com

Compute Engine default:
project-number-compute@developer.gserviceaccount.com
```

---

### 3. Google-Managed Service Account (Hands Off)

- Created and managed entirely by Google
- Used by Google's own internal services
- Some are visible in your IAM console, some are completely hidden
- Name ends with **"Service Agent"** or **"Service Account"**

> 🧠 Simple English: Google's own robots, doing Google things. You didn't hire them, you can't fire them, and you shouldn't mess with them.
> 

---

## Service Account Keys

A Service Account authenticates using **cryptographic keys**. Two types:

### Google-Managed Keys ✅ (Recommended)

What Google handles

---

Key storage

---

Key rotation (done regularly by Google)

---

Security

---

- You don't download, store, or manage anything
- Google rotates these keys automatically
- This is the **safest option** — zero key management burden on you

> 🧠 Simple English: Google holds the keys, rotates them regularly, and you never have to worry about them expiring or leaking. Like having a bank hold your valuables instead of keeping cash under your mattress.
> 

---

### User-Managed Keys ⚠️ (Handle with Care)

When you create and download a key (a `.json` file), **you are now responsible for:**

| Responsibility | Details |
| --- | --- |
| **Key storage** | Storing it securely — not in Git, not in Slack |
| **Key distribution** | Getting it to the right services securely |
| **Key revocation** | Disabling it when no longer needed |
| **Key rotation** | Regularly replacing old keys with new ones |
| **Key protection** | Preventing unauthorized access |
| **Key recovery** | Having a plan if the key is lost or stolen |

> 💀 User-managed keys are the #1 cause of GCP security incidents.
Developers download a key "just for testing," commit it to a public GitHub repo, and 45 minutes later a crypto miner has taken over their entire project.
This is not a hypothetical. It happens constantly.
> 

**If you must use user-managed keys:**

- Store in Secret Manager or a vault — never hardcode
- Rotate regularly using the IAM service account API
- Delete keys you're no longer using immediately

---

## Service Account Permissions

You can grant access to a service account at **two levels:**

### 1. Project Level

Grant a role to the service account on the entire project.
The service account can access all resources in that project that the role allows.

### 2. Service Account Level (Impersonation)

A **human user** can be granted permission to **act as** (impersonate) a service account.

> 🧠 Simple English: You let a human temporarily "wear the costume" of the service account and make calls on its behalf.
Useful for testing — instead of downloading keys, a developer just impersonates the service account.
> 

```
Required role to impersonate:
roles/iam.serviceAccountUser  →  act as a specific service account
roles/iam.serviceAccountTokenCreator  →  generate tokens for it
```

### Cross-Project Access

A service account in Project B can be granted access to resources in Project A or C.

```
Project A  ←──  service account (Project B)  ──→  Project C
```

> Service accounts don't care about project boundaries. Their access goes wherever you grant it.
> 

---

## Two Ways to Use a Service Account

### 1. Attach to a Resource (Binding)

Attach the service account directly to a GCP resource (like a VM or Cloud Run service).
The resource automatically uses that service account's identity — no keys needed.

> 🧠 Simple English: The VM *is* the service account. It just uses its identity automatically, like how your work laptop is logged into your company account.
> 

**This is the preferred approach for workloads running inside GCP.**

---

### 2. Impersonation

A human (or another service account) temporarily acts as the service account to make API calls.

> Useful for: developers testing locally, CI/CD pipelines, cross-service workflows.
> 

---

## Access Scopes (Legacy — Know for Exam)

Access Scopes are an **old method** of controlling what a VM's default service account can do.

> 🧠 Simple English: Before IAM roles were fully developed, Google used "scopes" to limit what a service account could access. It's like a second layer of permissions that came before roles existed.
> 

Options in the GCP console when creating a VM:

- **Allow default access** — basic read-only + write to logs/storage
- **Allow full access to all Cloud APIs** — everything (dangerous)
- **Set access for each API** — granular (but tedious)

> ⚠️ **The modern way:** Switch to a **custom (user-managed) service account** and use IAM roles instead of scopes. Scopes are legacy and limited.
> 
> 
> Changing to a custom service account = you control permissions fully via IAM roles, which is much cleaner.
> 

---

## Best Practices

| Practice | Why |
| --- | --- |
| **Audit service accounts and keys regularly** | Use `serviceAccount.keys.list()` or Logs Viewer to see what keys exist |
| **Delete external keys you don't need** | Unused keys are attack surface. Dead weight with a blast radius. |
| **Grant minimum permissions** | Principle of least privilege — only what the service needs |
| **Create one service account per service** | Don't reuse. Each service gets its own identity with its own minimal roles. |
| **Implement key rotation** | Use the IAM service account API to rotate user-managed keys regularly |
| **Prefer Google-managed keys** | Let Google handle key security — don't download keys unless absolutely necessary |
| **Use user-managed service accounts in production** | Never rely on default service accounts with Editor role in production |

---

## Quick Reference

| Concept | Key Point |
| --- | --- |
| What is it | A robot/app identity — not a human |
| User-managed | You create, name, and control it — **recommended for production** |
| Default | Auto-created by GCP, gets **Editor role** — ⚠️ risky |
| Google-managed | Google's internal robots — don't touch |
| Google-managed keys | Google rotates them — safest option, zero effort |
| User-managed keys | You manage everything — most dangerous, handle carefully |
| Impersonation | A human acts as the service account temporarily |
| Access scopes | Legacy method — use IAM roles instead |
| Best practice | One service account per service, minimum permissions, audit regularly |

---

> 💀 The lifecycle of a bad service account:
Day 1: "I'll just use the default service account for now."
Day 30: "I'll just download a key real quick."
Day 31: Key is in GitHub.
Day 32: You're explaining the incident to your manager.
> 

---

*Source: GCP Associate Cloud Engineer (ACE) prep — exampro.co/gcp-ace⚠️ Verify current defaults and key rotation periods with [GCP official docs](https://cloud.google.com/iam/docs/service-accounts) — some details may have changed.*

# 

# GCP — Cloud Identity

> Topic: Cloud Identity — IDaaS, Features, SSO, Device Management & GCDS
🔑 GCDS is a big and important part of Cloud Identity — covered in detail below.
> 

---

## What is Cloud Identity?

Cloud Identity is Google's **Identity as a Service (IDaaS)** product.

> 🧠 Simple English: It's Google's way of managing **who your employees are** — their accounts, their devices, their access — all from one central place in the cloud.
> 
> 
> Think of it as your company's HR system, but for digital identities. It knows who works here, what they're allowed to use, and kicks them out when they leave.
> 

It provides:

- **User and Group Management** — create, manage, and delete user accounts and groups
- **Identity Federation** — connect your existing company directory (like Microsoft Active Directory) to Google Cloud
- **Device Management** — control and secure devices accessing company resources
- **Single Sign-On (SSO)** — one login to rule them all
- **Security** — 2-step verification and more
- **Reporting** — audit logs and usage reports via BigQuery

---

## Cloud Identity Features

### 1. Device Management

> 🧠 Simple English: Your employees work from home, office, and airports. Cloud Identity lets you manage and secure their laptops and phones — from one place — no matter where they are.
> 
- Enforce **security policies** on employee devices (lock screen, encryption, etc.)
- Remotely **wipe a device** if it's lost or stolen
- Control which devices can access company resources
- Works for laptops, phones, and tablets

> 💀 Without device management, your company data is one lost laptop away from a very bad news article. Cloud Identity at least makes sure that laptop is encrypted and remotely wipeable.
> 

---

### 2. Security — 2-Step Verification (2SV)

> 🧠 Simple English: Password alone is not enough. 2-step verification adds a second check — proving you are who you say you are even if your password is compromised.
> 

Cloud Identity supports multiple 2SV methods:

| Method | What it is |
| --- | --- |
| **Security Key (hardware)** | Physical USB/NFC key — most secure |
| **Google Prompt** | Tap "Yes" on your phone |
| **Authenticator App** | Time-based OTP (e.g. Google Authenticator) |
| **Backup Codes** | Printed one-time codes for emergencies |

> Admins can **enforce** 2SV for all users in the organization — users can't opt out.
Because "I forgot my phone" is still better than "someone in Romania logged into my account."
> 

---

### 3. Single Sign-On (SSO)

> 🧠 Simple English: Log in once → access everything. No more 15 different passwords for 15 different apps.
> 

How it works:

1. User logs in **once** to Cloud Identity
2. Cloud Identity issues a **token** (proof of login)
3. User is automatically signed into App A, App B, App C — no more login prompts

```
User → [logs in once] → Cloud Identity → token → App A
                                               → App B
                                               → App C
```

Benefits:

- Users love it (fewer passwords to forget)
- IT loves it (one place to revoke access when someone leaves)
- Security loves it (centralized authentication = centralized control)

> When an employee quits, you disable their Cloud Identity account once — and they lose access to *everything* simultaneously. Beautiful.
> 

---

### 4. Reporting

> 🧠 Simple English: Cloud Identity keeps audit logs of user activity — who logged in, from where, using what device, and when.
> 

Flow:

```
Cloud Identity (audit logs) → BigQuery → Admin dashboard / reports
```

Use it for:

- Compliance reporting
- Detecting suspicious login activity
- Answering "who accessed what and when" during an incident

---

### 5. Directory Management

- Central directory of all users and groups in your organization
- Sync with external directories (see GCDS below)
- Manage user profiles, group memberships, and org structure

---

## Google Cloud Directory Sync (GCDS) — The Big One

> ⭐ GCDS is a major part of Cloud Identity. This is the bridge between your **existing company directory** and Google Cloud.
> 

### The Problem GCDS Solves

Most companies already have a user directory — usually **Microsoft Active Directory (AD)** — running on-premises. It has all employee accounts, groups, and permissions built up over years.

You don't want to recreate all of that manually in Google Cloud.

> 🧠 Simple English: Your company has 500 employees already set up in Microsoft Active Directory. GCDS copies those users into Cloud Identity automatically — so you don't have to manually create 500 Google accounts. It also keeps them in sync as people join, leave, or change roles.
> 

---

### How GCDS Works

```
On-premises                    Google Cloud
─────────────                  ─────────────────────────────
AD Domain ──[one-way sync]──▶  GCDS ──▶ Cloud Identity ──▶ GCP Organization
AD FS ◀──────[SSO]─────────────────────────────────────────  Other Google-Services
                                                              Corporate SaaS Apps
```

**Key points:**

- **One-way sync only** — data flows FROM Active Directory TO Cloud Identity. Changes in Google do NOT go back to AD.
- **GCDS** is the tool that runs the sync (it's a separate tool you configure and run)
- **AD FS** (Active Directory Federation Services) handles SSO — employees use their existing AD credentials to log into Google services
- **Passwords are NOT synced** — GCDS only syncs users and groups, not passwords. Authentication still goes through AD FS via SSO.

> 💀 "One-way sync" is an exam favourite. Remember: AD → Cloud Identity. Not the other way around.
If you delete someone from Cloud Identity, GCDS will happily add them back next sync because they still exist in AD. The source of truth is always your on-premises AD.
> 

---

### What Gets Synced

| Synced | Not Synced |
| --- | --- |
| Users (accounts) | Passwords |
| Groups | On-premises resources |
| Org units | AD-specific attributes not mapped |

---

### GCDS vs Manual Setup — Why It Matters

| Without GCDS | With GCDS |
| --- | --- |
| Manually create every Google account | Accounts auto-created from AD |
| Manually update accounts when employees change roles | Auto-updated on next sync |
| Manually delete accounts when people leave | Auto-deleted (or disabled) on next sync |
| Two separate identity systems to maintain | One source of truth: AD |

---

### SSO with AD FS

When GCDS is set up, employees don't need a separate Google password:

1. Employee tries to access Gmail / GCP Console / any Google service
2. Google redirects to **AD FS** (on-premises)
3. AD FS authenticates with the employee's existing AD password
4. AD FS sends a token back to Google
5. Employee is logged in

> 🧠 Simple English: Your company Windows login becomes your Google login. One password, one brain cell required.
> 

---

## Cloud Identity vs G Suite (Workspace)

|  | Cloud Identity | Google Workspace (G Suite) |
| --- | --- | --- |
| User management | ✅ Yes | ✅ Yes |
| SSO | ✅ Yes | ✅ Yes |
| Gmail, Docs, Drive | ❌ No | ✅ Yes |
| Cost | Free tier available | Paid |
| Best for | Companies that just need GCP access management | Companies that also want Google productivity apps |

> 🧠 Simple English: Cloud Identity = the "just the identity, no Gmail" version. Workspace = identity + all the Google apps.
> 

---

## Quick Reference Cheat Sheet

| Concept | Key Point |
| --- | --- |
| Cloud Identity | IDaaS — manages users, groups, devices, SSO, security |
| Device Management | Enforce policies, remotely wipe lost devices |
| 2-Step Verification | Hardware key, Google Prompt, Authenticator app, backup codes |
| SSO | Login once → access all apps via token |
| Reporting | Audit logs → BigQuery → admin dashboards |
| GCDS | Syncs on-premises Active Directory users/groups → Cloud Identity |
| Sync direction | One-way: AD → Cloud Identity only |
| Passwords synced? | No — authentication via AD FS + SSO |
| Cloud Identity vs Workspace | Cloud Identity = identity only, no Google apps |

---

> 💀 The classic enterprise mistake: Setting up GCP without GCDS, then manually maintaining two separate user directories for years. Then someone leaves the company, gets deprovisioned from AD, but nobody remembers to delete their GCP account. Six months later, that account is the entry point for a breach.
> 
> 
> GCDS exists so you only have to manage one directory. Use it.
> 

---

*Source: GCP Associate Cloud Engineer (ACE) prep — exampro.co/gcp-ace*

# GCP IAM — Best Practices

> Topic: IAM Best Practices — Least Privilege, Resource Hierarchy, Service Accounts, Auditing & Policy Management
💡 These are the rules senior engineers follow so they don't get paged at 3am.
> 

---

## 1. Least Privilege

> 🧠 Simple English: Give people **only** what they need to do their job — nothing more, nothing less.
It's not about being stingy. It's about making sure one compromised account can't burn down your entire infrastructure.
> 

### Rules to follow:

**Use predefined roles over primitive roles**

- Predefined = fine-grained, specific (`roles/storage.objectViewer`)
- Primitive = Owner / Editor / Viewer — way too broad
- Primitive roles are like giving a house key to someone who only needs to water the plants once

**Grant roles at the smallest scope**

- If someone needs access to one bucket → grant it on that bucket
- Not on the project. Not on the folder. On. That. Bucket.

**Child resources cannot restrict what the parent gave**

- If you grant Editor at the Organization level, that person has Editor on *everything* below
- A project-level policy cannot take away what the org-level policy already gave
- ⚠️ This is why granting permissions high up in the hierarchy is dangerous — it flows down and there's no blocking it

**Restrict who can create and manage service accounts**

- Not everyone needs to be able to create service accounts
- An untrusted person creating a service account with Owner role = backdoor into your project

**Be very cautious with Owner role**

- Owner can do anything — including changing IAM policies, deleting projects, billing changes
- Should be assigned to the fewest people possible
- Ideally: only break-glass emergency accounts have Owner

> 💀 The #1 IAM mistake: "I'll just give everyone Editor for now and clean it up later."
"Later" never comes. The permissions accumulate. Someone leaves. Their account still has Editor.
Six months later you're doing an incident post-mortem wondering how it happened.
> 

---

## 2. Resource Hierarchy Best Practices

> 🧠 Simple English: Your GCP folder/project structure should mirror your real company structure. If your company has a Finance dept and an Engineering dept — your GCP hierarchy should too. Then permissions flow naturally down the tree.
> 

**Mirror GCP hierarchy to your org structure**

```
Company
├── Engineering (folder)
│   ├── Backend (folder)
│   │   ├── prod (project)
│   │   └── dev (project)
│   └── Frontend (folder)
└── Finance (folder)
    └── billing-system (project)
```

**Use projects to group resources that share the same trust boundary**

- Resources in the same project trust each other more
- Resources in different projects are isolated by default
- Don't mix production and dev in the same project — they have very different trust requirements

**Set policies at the organization or project level — not the resource level**

- Managing permissions on individual VMs or buckets = chaos at scale
- Set policies high up, let inheritance do the work
- Exception: when a specific resource genuinely needs different access

**Use least privilege when granting IAM roles**

- Same rule, repeated because it's that important

**If a user/group spans multiple projects — grant roles at the folder level**

```
# Bad: Granting the same role on 10 projects individually
# Good: Grant the role once at the folder level → inherits to all 10 projects
```

> 🧠 Simple English: If your backend team needs access to 8 projects — don't click through each project 8 times. Put those projects in a folder and grant the role once on the folder.
> 

---

## 3. Service Account Best Practices

> 🧠 Simple English: Service accounts are powerful. One poorly managed service account can be the skeleton key to your entire project. Treat them like loaded weapons — carefully.
> 

**Treat each app as a separate trust boundary**

- One service account per app/service
- Don't reuse service accounts across different apps
- If one service account is compromised, only that app's blast radius is affected — not everything

**Do NOT delete service accounts that are in use by running services**

- Deleting an active service account = instantly breaking the app that relies on it
- Production outage incoming — no warning, no grace period, just broken
- Audit first, delete only after confirming it's unused

**Rotate user-managed service account keys**

- If you're using self-managed keys (downloaded `.json` files), rotate them regularly
- Old keys sitting around = attack surface
- Use the IAM API to automate rotation

> ⭐ **Production note:** Most production environments use **Google-managed keys** — not self-managed keys. Google rotates them automatically. If you are using self-managed keys for any reason, rotate them manually and regularly. No excuses.
> 

**Name service account keys to reflect their purpose and permissions**

- Bad: `key1.json`, `my-key.json`, `final_key_v3.json`
- Good: `payments-service-storage-writer.json`
- You should know what a key does just by reading its name

**Restrict service account access**

- Only grant the service account the permissions it actually needs
- A payments service doesn't need access to your ML pipelines

**Never check service account keys into source code**

- Not in Git. Not in GitHub. Not in a "private" repo.
- Private repos get leaked. People accidentally push. CI logs capture env vars.
- Use **Secret Manager** or environment variables injected at runtime
- If a key is ever committed to a repo — assume it's compromised. Revoke it immediately.

> 💀 True story: Developer commits `.json` key to GitHub "just temporarily."
GitHub scanners (and bots) detect it within minutes.
Crypto miners spin up 400 VMs in the project.
Monthly bill: $47,000.
"Just temporarily" is never just temporary.
> 

---

## 4. Auditing Best Practices

> 🧠 Simple English: Logs are your time machine. When something goes wrong — and it will — logs tell you who did what, when, and from where. But only if you set them up properly *before* the incident.
> 

**Use Cloud Audit Logs to regularly audit IAM policy changes**

- Who added or removed roles?
- When were policies changed?
- Set up automated alerts for suspicious IAM changes

**Audit who can edit IAM policies on projects**

- The ability to change IAM policies is itself a powerful permission
- Regularly review who has `resourcemanager.projects.setIamPolicy`

**Export audit logs to Cloud Storage for long-term retention**

- Cloud Logging has limited default retention (30 days for most logs)
- Export to Cloud Storage for months/years of retention
- Required for compliance in most regulated industries

```
Cloud Audit Logs → Log Sink → Cloud Storage bucket (long-term)
                            → BigQuery (analysis & dashboards)
```

**Regularly audit service account key access**

- List all existing keys: `gcloud iam service-accounts keys list`
- Delete keys that are old or unused
- Know which keys exist and who has them

**Restrict log access with logging roles**

- Not everyone should be able to read audit logs
- Use `roles/logging.viewer` for read-only log access
- Logs contain sensitive operational data — protect them like any other sensitive resource

> 🧠 Simple English: Giving everyone access to audit logs defeats the point of audit logs. If the person you're auditing can also read and delete the logs — congratulations, you've built a very expensive nothing.
> 

---

## 5. Policy Management Best Practices

> 🧠 Simple English: Managing permissions one person at a time is a nightmare. Use groups. Set policies at the right level. Let the hierarchy do the heavy lifting.
> 

**Grant access to all projects via organization-level policy**

- Need someone to have read access to every project? Set it once at the org level.
- Don't click through 50 projects and repeat yourself

**Grant roles to Google Groups — not individual users**

- User joins the backend team → add them to the `backend-team` Google Group → they get all the right permissions automatically
- User leaves the team → remove from group → permissions gone automatically
- No manual IAM policy updates required

```
# Bad: Grant role to each person individually
user:alice@company.com  → roles/compute.viewer
user:bob@company.com    → roles/compute.viewer
user:charlie@company.com → roles/compute.viewer

# Good: Grant role to the group once
group:backend-team@company.com → roles/compute.viewer
# Now add/remove people from the group, not from IAM
```

**If a task needs multiple roles — create a Google Group for that task**

- Instead of granting 5 different roles to 10 different people individually
- Create a group like `deployment-team`, grant it the needed roles
- Add people to the group as needed

> 🧠 Simple English: IAM policy lines are like phone contacts. Managing 200 individuals is chaos. Managing 10 groups of 20 people each is manageable. When someone's role changes, you update their group membership — not 200 individual IAM bindings.
> 

---

## Quick Reference — The IAM Best Practices Checklist

| Category | Rule |
| --- | --- |
| **Least Privilege** | Use predefined roles, not primitive |
| **Least Privilege** | Grant at smallest possible scope |
| **Least Privilege** | Be extremely cautious with Owner role |
| **Hierarchy** | Mirror GCP structure to org structure |
| **Hierarchy** | Set policies at org/project level, not resource level |
| **Hierarchy** | Use folder-level roles for multi-project access |
| **Service Accounts** | One SA per service — never share |
| **Service Accounts** | Never delete active service accounts |
| **Service Accounts** | Prefer Google-managed keys in production |
| **Service Accounts** | If using self-managed keys — rotate them |
| **Service Accounts** | Never commit keys to source code — ever |
| **Auditing** | Enable and regularly review Cloud Audit Logs |
| **Auditing** | Export logs to Cloud Storage for retention |
| **Auditing** | Restrict who can read audit logs |
| **Policy Management** | Grant roles to groups, not individuals |
| **Policy Management** | Use org-level policy for org-wide access |
| **Policy Management** | Group related roles into a Google Group |

---

> 💀 IAM misconfiguration is the leading cause of cloud security incidents.
Not sophisticated zero-day exploits. Not nation-state hackers.
Just someone clicking "Owner" because it was easier than figuring out the right role.
These best practices exist because real companies learned them the expensive way — so you don't have to.
> 

---

*Source: GCP Associate Cloud Engineer (ACE) prep — exampro.co/gcp-ace | Production engineering best practices*
