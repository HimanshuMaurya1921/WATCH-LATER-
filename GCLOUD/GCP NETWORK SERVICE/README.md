# GCP NETWORK SERVICE

# GCP — Virtual Private Cloud (VPC)

> Topic: VPC Fundamentals, Subnets, Peering, Shared VPC & VPC Peering Important Points
🌐 VPC is the foundation of all networking in GCP. Everything runs inside one.
> 

---

## 1. What is a VPC?

A **VPC (Virtual Private Cloud)** is a virtualized private network inside Google Cloud where your resources talk to each other securely.

> 🧠 Simple English: Think of VPC as your company's private office building inside Google's infrastructure. You decide who can enter, which floors talk to each other, and what goes in and out. Google just provides the land.
> 

It is the service that allows you to create networks inside the cloud with:

- **Public connectivity** — for internet-facing resources
- **Private connectivity** — for internal resources
- **Endpoints** — for accessing Google services privately

You control: IP ranges, subnets, routing, and firewall rules.

---

## 2. Key Characteristics of GCP VPC

> ⚠️ These are exam favourites. Read twice.
> 

| Property | Value |
| --- | --- |
| **VPC scope** | **Global** — one VPC spans all regions |
| **Subnet scope** | **Regional** — each subnet lives in one region |
| **VM scope** | Zonal |
| **Firewall rules scope** | VPC-level |
| **IP ranges on VPC** | VPCs do **NOT** have IP ranges — subnets do |
| **Internal communication** | Resources in same VPC talk via private IPv4 |
| **IPv6 support** | ✅ GCP VPC now supports IPv6 (verify current status with GCP docs) |

> 🧠 Simple English: The VPC is the house. Subnets are the rooms. VMs are the furniture in specific rooms. The house exists globally — but each room is in one city (region).
> 

> ⚠️ **Not every cloud works this way.** In AWS, VPCs are regional. In GCP, VPCs are global. This trips people up on exams constantly.
> 

---

## 3. Network Types

| Type | Description | Use when |
| --- | --- | --- |
| **Default VPC** | Auto-created when you enable Compute Engine. Has pre-made subnets + default firewall rules. | Quick testing only |
| **Auto Mode VPC** | GCP automatically creates one subnet per region. Predefined IP ranges. | Simple setups |
| **Custom Mode VPC** | You create subnets manually. Full control over IP ranges. | **Production — always** |

> 🧠 Simple English: Auto mode is like a furnished apartment — convenient but you can't change the layout. Custom mode is an empty flat you design yourself. In production, you always want to design your own layout.
> 

**⚠️ Important rule:**

- You **CAN** switch from Auto Mode → Custom Mode
- You **CANNOT** switch back from Custom Mode → Auto Mode
- It's a one-way door. Choose wisely.

---

## 4. Default VPC — What GCP Gives You

When a new project is created, GCP automatically creates a **Default VPC** with:

- IP range: `10.128.0.0/9`
- One `/20` subnet in each region
- Route to Default Internet Gateway (so all VMs have internet access by default)

| Region | Default Subnet |
| --- | --- |
| us-east1 | 10.142.0.0/20 |
| us-central1 | 10.128.0.0/20 |
| europe-west1 | 10.132.0.0/20 |
| asia-east1 | 10.140.0.0/20 |
| australia-southeast1 | 10.152.0.0/20 |
| southamerica-east1 | 10.158.0.0/20 |
| northamerica-northeast1 | 10.162.0.0/20 |

> 💀 The default VPC is great for "I just want something running in 5 minutes." It is catastrophically bad for production. Every VM gets internet access by default, firewall rules are wide open, and you have zero control over IP planning. Delete it and start with custom mode for anything serious.
> 

---

## 5. Subnets

- Subnets are **regional** resources — they live in one region
- Each subnet has its own **CIDR IP range**
- Resources in the same VPC (even across regions/subnets) can communicate by default

### Expanding a Subnet (Growing the IP Range)

You **can** expand a subnet by decreasing the subnet mask:

```
/24  →  /23   (doubles the available IPs)
/24  →  /22   (4x the available IPs)
```

> 🧠 Simple English: Smaller the number after `/`, bigger the network. `/23` gives you more IPs than `/24`.
> 

**Rules:**

- ✅ You CAN expand (grow) a subnet
- ❌ You CANNOT shrink a subnet — once IPs are allocated, you can't take them back
- ❌ You CANNOT overlap CIDR ranges between subnets in the same VPC

---

## 6. VPC Internal Communication

Resources in the **same VPC** talk to each other using **private internal IPv4 addresses** — automatically, across all regions, no extra config needed.

```
VM in us-central1  ←──(private IP)──→  VM in europe-west1
Both in the same VPC → works automatically
```

- **Different VPCs** = isolated by default. They cannot communicate without VPC Peering or Shared VPC.
- **Different projects** = also isolated by default.

> 🧠 Simple English: Same VPC = same building. Different VPC = different buildings. People in the same building can walk to each other. People in different buildings need a bridge.
> 

---

## 7. Firewall Rules

- Applied at **VPC level**, affect VM instances
- Default behaviour:
    - **Ingress (incoming):** Deny all by default
    - **Egress (outgoing):** Allow all by default

Default VPC comes with pre-made rules that allow SSH, RDP, and internal traffic — which is why it's convenient but dangerous in production.

---

## 8. External Connectivity Options

| Option | Use case |
| --- | --- |
| **Internet Gateway** | VMs with public IPs accessing/receiving from the internet |
| **Cloud NAT** | Private VMs (no public IP) that need outbound internet access only |
| **Cloud VPN** | Encrypted tunnel between on-premises and GCP over the internet |
| **Cloud Interconnect** | Dedicated physical connection between your data centre and GCP (high speed, low latency) |

---

## 9. VPC Network Peering

### What is it?

VPC Peering allows **two VPC networks to communicate using internal/private IP addresses** — without going through the public internet.

> 🧠 Simple English: Two separate office buildings getting a private tunnel between them. People can walk between buildings through the tunnel — no need to go outside onto the public street.
> 

### Key Facts

- Works between VPCs in the **same project, same org, OR different projects and different orgs** — no restriction
- Traffic travels through **Google's internal network** — never touches the public internet
- Even if the two orgs are on different continents — the traffic stays internal to Google
- Results in: **lower latency, more security, cheaper than using public IPs**

### Non-Overlapping CIDR is MANDATORY

Both VPCs must have **non-overlapping subnet IP ranges** to peer.

**The tricky part — why peered networks' CIDRs also matter:**

```
VPC-A: 10.0.1.0/24   ←──(active peering)──→   VPC-B: 10.0.3.0/24

Now NEW-NETWORK (10.0.1.0/24) tries to peer with VPC-B
→ BLOCKED ❌
```

**Why?** Even though NEW-NETWORK and VPC-B don't overlap, NEW-NETWORK overlaps with VPC-A — which is already peered with VPC-B. VPC-B's routing table includes VPC-A's routes. If NEW-NETWORK with the same range connects, VPC-B won't know which network to route traffic to. This is called **routing ambiguity** — GCP blocks it to prevent routing chaos.

> 🧠 Simple English: Imagine two delivery guys with the same house address trying to get packages from the same warehouse. The warehouse doesn't know who to send the package to. GCP blocks this before it becomes a disaster.
> 

### Transitive Peering — NOT Supported

```
VPC-A  ←──peered──→  VPC-B  ←──peered──→  VPC-C

Does VPC-A communicate with VPC-C? ❌ NO
```

Peering is **not transitive**. A→B and B→C does NOT mean A→C. Each connection must be set up directly.

> 🧠 Simple English: Being friends with someone's friend doesn't make you friends. You have to make the connection yourself.
> 

### VPC Peering Process (Step by Step)

**Scenario: Peer Project-A (VPC-A) with Project-B (VPC-B)**

**Step 1 — From Project-A:**

1. Go to: VPC → VPC Network Peering → Create Connection
2. Click Continue (accept disclaimer)
3. Enter:
    - Peering connection name
    - Your VPC network (VPC-A)
    - Peered VPC network:
        - Same project → select VPC name
        - Different project → enter Project-ID + VPC name
    - IP Stack: IPv4 (usually)
    - Tick: Import custom routes + Export custom routes
    - Tick subnet route exchange options as needed
4. Click Create

**At this point → Status: INACTIVE** (only one side configured)

**Step 2 — From Project-B:**

- Repeat the exact same steps but in reverse (Project-B → Project-A, VPC-B → VPC-A)

**Step 3 → Status changes to: ACTIVE** 🟢

- Subnet routes and custom routes are now exchanged
- VM-A1 and VM-B1 can now communicate privately

> 💀 Forgetting to configure both sides is the #1 reason peering stays inactive forever. It's like calling someone and waiting — someone has to pick up on the other end too.
> 

### VPC Peering Limits & Important Points

| Point | Detail |
| --- | --- |
| Max peering connections per VPC | **25** |
| CIDR overlapping | Not allowed (including CIDRs of already-peered networks) |
| Transitive peering | Not supported |
| To delete a VPC | Must first delete all peering configs |
| IAM role required | `roles/editor` or `roles/compute.networkAdmin` |
| Works with | Compute Engine, GKE, App Engine flexible environment |

---

## 10. Shared VPC

### What is it?

Shared VPC allows **multiple projects** to share a **single VPC network** managed by one central "host" project.

> 🧠 Simple English: One team manages the network (the landlord). Other teams use it (the tenants). Tenants don't manage the plumbing — they just use the building.
> 

### Structure

```
Host Project
   └── Shared VPC Network
         ├── Subnet A
         ├── Subnet B
         └── Subnet C

Service Project 1 → Uses Subnet A  (e.g. Frontend team)
Service Project 2 → Uses Subnet B  (e.g. Backend team)
Service Project 3 → Uses Subnet C  (e.g. Data team)
```

### Key Facts

- **Same organization only** — Shared VPC does NOT work across different organizations
- Centralized network control (the host project manages all networking)
- Each service project has separate billing, IAM, and resources
- Great for enterprises where networking needs to be centrally managed

### Shared VPC vs VPC Peering — Key Difference

|  | VPC Peering | Shared VPC |
| --- | --- | --- |
| Across organizations | ✅ Yes | ❌ No (same org only) |
| Network ownership | Each VPC owns its own network | Host project owns the network |
| Use case | Two separate teams/orgs that need to talk | One org with multiple teams sharing a network |

---

## 11. Quick Exam Q&A

**Q: Is VPC peering transitive?**
❌ No. A↔B and B↔C does NOT mean A↔C. You must peer A↔C directly.

**Q: Can VPC peering connect different organizations?**
✅ Yes. No restriction on org or project.

**Q: Can Shared VPC connect different organizations?**
❌ No. Same organization only.

**Q: What IAM role is needed for VPC peering?**`roles/compute.networkAdmin` (or `roles/editor`)

**Q: What are the two steps to create VPC peering between VPC-A and VPC-B?**

1. Create peering from VPC-A → VPC-B
2. Create peering from VPC-B → VPC-A
Both sides must be configured for status to become ACTIVE.

**Q: Can you shrink a subnet?**
❌ No. You can only expand (grow) it by reducing the subnet mask number.

**Q: Can you switch from Custom Mode VPC to Auto Mode VPC?**
❌ No. One-way only: Auto → Custom.

**Q: Does VPC peering traffic go through the public internet?**
❌ No. It travels through Google's internal network — even across continents and different organizations.

---

## 12. Quick Reference Cheat Sheet

| Concept | Key Point |
| --- | --- |
| VPC scope | **Global** |
| Subnet scope | **Regional** |
| VM scope | Zonal |
| VPC has IP range? | ❌ No — subnets have IP ranges |
| Default ingress | Deny all |
| Default egress | Allow all |
| Subnet expansion | ✅ Allowed (grow only) |
| Subnet shrink | ❌ Not allowed |
| Auto → Custom | ✅ One-way only |
| Custom → Auto | ❌ Not allowed |
| VPC Peering traffic | Google internal network — never public internet |
| Transitive peering | ❌ Not supported |
| Shared VPC | Same org only |
| VPC Peering | Any org, any project |
| Max peering per VPC | 25 |
| Overlapping CIDRs | ❌ Not allowed in peering |

---

> 💀 Common exam trap: "VPC is global" sounds like subnets are global too. They're not.
VPC = global. Subnet = regional. VM = zonal. Tattoo this on your brain.
> 
> 
> Another one: Shared VPC = same org. VPC Peering = any org. Examiners love asking this.
> 

---

*Source: GCP Associate Cloud Engineer (ACE) prep | Screenshots + PDF notes | exampro.co/gcp-ace⚠️ IPv6 support details — verify current status with GCP official docs as this is an evolving feature.*

# GCP VPC — Routing & Private Google Access

> Topic: VPC Routing (Types, Order, Special Paths) + Private Google Access
🗺️ Routes tell your packets where to go. Without them, your traffic is lost — literally.
> 

---

## 1. What is Routing?

A **Route** defines the network traffic path from one destination to another.

> 🧠 Simple English: Routes are like Google Maps for your network packets. When a packet leaves a VM, it checks the routing table and asks "which way should I go?" — the route with the best match wins.
> 

### How a route works in GCP VPC:

- Every route has a **single destination (CIDR)** and a **single next hop**
- All routes are stored in the **routing table** for the VPC
- When a packet leaves a VM → GCP checks all routes → picks the **most specific match** → sends to next hop
- If multiple routes match → **priority number** decides (lower number = higher priority)

```
Packet destination: 10.0.1.5

Routing table:
  10.0.0.0/8     → next hop: VPN tunnel   (priority 1000)
  10.0.1.0/24    → next hop: subnet-1    (priority 1000)  ← wins (more specific)
  0.0.0.0/0      → next hop: internet GW  (priority 1000)
```

---

## 2. Route Types

```
Routes
├── System-generated (auto-created by GCP)
│   ├── Default Route
│   └── Subnet Route
│
└── Custom Routes (you create)
    ├── Static Route
    └── Dynamic Route
```

---

## 3. System-Generated Routes

### 3.1 Default Route

> 🧠 Simple English: The "if nothing else matches, go here" route. It's the fallback — like "if you don't know where to go, go to the internet gateway."
> 

| Property | Detail |
| --- | --- |
| Destination | `0.0.0.0/0` (catch-all — everything) |
| Next hop | Default Internet Gateway |
| Priority | **1000** (lowest among system routes) |
| Purpose | Path to the internet + path for Private Google Access |

**Important rules:**

- Has the **lowest priority** of all system routes
- Can be **deleted only by replacing it with a custom route**
- You can't just delete it — you must replace it
- Removing it means VMs lose internet access (unless you add a custom replacement)

> 💀 Want to block all internet egress from your VPC? Replace the default route with a custom route that points to a firewall appliance or a black hole. Just deleting it won't work — you must replace it.
> 

---

### 3.2 Subnet Route

> 🧠 Simple English: Auto-created routes that tell the VPC "hey, subnet X lives here, send traffic for that IP range this way." They're created the moment you create a subnet.
> 

| Property | Detail |
| --- | --- |
| Auto-created when | A subnet is created |
| Destination | Primary IP range of the subnet |
| Also covers | Secondary IP ranges (if any) |
| Priority | **0** (highest — always wins over default route) |
| Can you delete it? | ❌ No — you must modify or delete the subnet itself |

```
Subnet created: 10.30.0.0/24  (us-west1)
→ GCP auto-creates: subnet route for 10.30.0.0/24 → next hop: virtual network
```

**Key point:** Subnet routes have priority **0** — they always take precedence over the default route (priority 1000) and most custom routes.

---

## 4. Custom Routes

Routes you create yourself. Two types:

### 4.1 Static Route

> 🧠 Simple English: You manually tell GCP "traffic going to X should go via Y." It's fixed — you set it, it stays until you change it.
> 

**Key facts:**

- Created manually by you
- Can use the **next hop** feature to specify exactly where traffic should go
- Static routes for VPN tunnels are **automatically created** when you create a Cloud VPN tunnel (you don't have to make them yourself)

**Next hop options for static routes:**

| Next hop | Use case |
| --- | --- |
| Default Internet Gateway | Send traffic to the internet |
| Specify an instance | Route traffic to a specific VM (e.g. a NAT appliance) |
| Specify an IP address | Route to a specific internal IP |
| Specify a VPN tunnel | Route to on-premises via VPN |
| Forwarding rule (internal TCP/UDP LB) | Route to an internal load balancer |

### Static Route Parameters (when creating in console)

| Parameter | Description |
| --- | --- |
| **Name** | Unique name in the project (lowercase, numbers, hyphens) |
| **Network** | Which VPC this route belongs to |
| **Destination IP range** | IPv4 CIDR block — which traffic this route handles |
| **Priority** | Lower number = higher priority. `0` = highest, `1000` = default |
| **Instance tags** | Route only applies to VMs with this network tag |
| **Next hop** | Where to send matching traffic |

> 💡 Instance tags are powerful — you can create routes that apply to specific VMs only. e.g. "only route traffic from tagged 'nat-vm' through this path."
> 

---

### 4.2 Dynamic Route

> 🧠 Simple English: Instead of manually telling the network what routes exist, you let routers automatically learn routes from each other and update the routing table on the fly. No human intervention needed.
> 

**Key facts:**

- Managed by one or more **Cloud Routers**
- **Dynamically exchange routes** between your VPC and on-premises networks
- Destination IP ranges are **outside the VPC** (on-premises or remote networks)
- Used with **dynamically routed VPNs** (HA VPN) and **Cloud Interconnect**
- Uses **BGP (Border Gateway Protocol)** under the hood to exchange routes

**When to use dynamic routes:**

- Your on-premises network changes frequently (routes added/removed)
- You want automatic failover without manually updating routes
- You're using Cloud Interconnect or HA VPN

> Static routes = good for simple, fixed setups.
Dynamic routes = good for complex, changing networks like hybrid cloud.
> 
> 
> 🧠 Simple English: Static = you draw the map once. Dynamic = the map updates itself as roads are built or closed.
> 

---

## 5. Routing Order (How GCP Picks a Route)

When a packet leaves a VM, GCP evaluates routes in this order:

**Step 1:** Find all routes whose destination CIDR matches the packet's destination IP.

**Step 2:** Among matching routes, pick the **most specific** match (longest prefix wins).

```
0.0.0.0/0     matches 10.5.1.1  ← less specific
10.5.0.0/16   matches 10.5.1.1  ← more specific  ← WINS
10.5.1.0/24   matches 10.5.1.1  ← most specific  ← WINS over above
```

**Step 3:** If multiple routes have the same specificity → pick the one with the **lowest priority number** (0 = highest priority, 1000 = lower priority).

**Step 4:** If still tied → GCP uses ECMP (Equal Cost Multi-Path) — load balances across multiple next hops.

### Real routing table example:

| Route | Destination | Priority | Next Hop |
| --- | --- | --- | --- |
| subnet route (auto) | 10.168.0.0/20 | **0** | Virtual network |
| subnet route (auto) | 10.166.0.0/20 | **0** | Virtual network |
| subnet route (auto) | 10.178.0.0/20 | **0** | Virtual network |
| default route (auto) | 0.0.0.0/0 | **1000** | Default Internet Gateway |

> Notice: all subnet routes have priority 0 and the default route has priority 1000. Subnet traffic always wins over the default internet route — exactly as intended.
> 

---

## 6. Special Return Paths

GCP has some **special routes** that exist outside your normal routing table — they handle things like:

- Health check traffic from Google's health checking systems
- Traffic for Google's management systems

These routes are not visible in the routing table but are always active. They handle return traffic for specific Google services going through the firewall.

> 🧠 Simple English: Think of these as VIP back-corridors in your network. Google's internal systems need to reach your VMs for health checks and management — these special paths make that happen without showing up in your routing table.
> 

---

## 7. Private Google Access

### What is it?

Private Google Access allows **VMs without external IP addresses** to reach **Google APIs and services** (like Cloud Storage, BigQuery, Pub/Sub) without going through the public internet.

> 🧠 Simple English: Your VM has no public IP (it's fully private). Normally, it can't talk to anything outside the VPC. But you still need it to use Cloud Storage. Private Google Access is the solution — it creates a private path to Google services without ever touching the public internet.
> 

### How it works:

```
VM1 (no public IP, Private Google Access ON)
    → wants to access storage.googleapis.com
    → traffic goes via internal subnet route
    → hits Google APIs via private path  ✅ (never leaves Google's network)

VM2 (has public IP)
    → wants to access storage.googleapis.com
    → traffic goes via internet gateway
    → reaches Google APIs via internet  ✅ (normal path)
```

### Key facts:

- Enabled **per subnet** (not per VPC)
- When enabled, VMs in that subnet can reach Google APIs using **internal routing**
- The VM must have **no external IP** for this to be the relevant path (VMs with external IPs can just use the internet)
- Traffic to Google APIs never leaves Google's network
- The **Default Route** also serves as the path for Private Google Access

### To enable:

Subnet settings → Toggle "Private Google Access" → On

---

## 8. Other Private Access Options

| Option | What it does |
| --- | --- |
| **Private Google Access** | VMs without public IPs access Google APIs privately |
| **Private Google Access for on-premises hosts** | On-premises machines access Google APIs via Cloud VPN / Interconnect without internet |
| **Private Services Access** | Access Google-managed services (like Cloud SQL) via private IP |
| **Serverless VPC Access** | Cloud Run / Cloud Functions / App Engine access resources in your VPC privately |

> 🧠 Simple English:
> 
> - Private Google Access = your private VM talks to Google services
> - Private Services Access = your private VM talks to Google-managed databases
> - Serverless VPC Access = your serverless functions talk to your private VMs

---

## 9. Quick Reference Cheat Sheet

| Concept | Key Point |
| --- | --- |
| Route = | Destination CIDR + Next hop |
| All routes stored in | Routing table (per VPC) |
| Best route selection | Most specific CIDR match first, then lowest priority number |
| Default Route destination | `0.0.0.0/0` |
| Default Route priority | 1000 (lowest) |
| Default Route next hop | Default Internet Gateway |
| Can you delete default route? | Only by replacing with custom route |
| Subnet Route priority | 0 (highest) |
| Can you delete subnet route? | Only by modifying/deleting the subnet |
| Static Route | You create manually, fixed next hop |
| Dynamic Route | Auto-managed by Cloud Router via BGP |
| Dynamic Route used with | HA VPN, Cloud Interconnect |
| Private Google Access | Private VMs → Google APIs without internet |
| Private Google Access enabled | Per subnet (not VPC-wide) |

---

> 💀 Classic mistake: Someone disables the default route thinking "I'll block internet access" — and instead breaks Private Google Access too. The Default Route handles **both** internet traffic AND Private Google Access paths. Replace it carefully, don't just nuke it.
> 

---

*Source: GCP Associate Cloud Engineer (ACE) prep — exampro.co/gcp-ace*

# GCP VPC — IP Addressing

> Topic: Internal & External IP Addressing, Ephemeral vs Static, Alias IPs, Reservation
📍 Every VM needs an address. This is how GCP decides what kind.
> 

---

## 1. The Big Picture — IP Address Tree

```
IP Address
├── Internal (Private)
│   ├── Alias IP          [OPTIONAL — extra IPs on same NIC]
│   ├── Auto              [GCP picks the IP from subnet range]
│   │   ├── Ephemeral     [changes on stop/restart]
│   │   └── Static        [reserved, stays forever]
│   └── Custom            [you pick the IP manually]
│       ├── Ephemeral
│       └── Static
│
└── External (Public)
    ├── Ephemeral         [auto-assigned, lost on stop/restart]
    └── Static            [reserved, stays until you release it]
        ← can be promoted from Ephemeral TO Static
```

> 🧠 Simple English: Every VM gets an internal IP (always). External IP is optional — only if you need the VM to talk to the internet directly.
> 

---

## 2. Internal (Private) IP Address

- **Not publicly advertised** — only visible inside your VPC
- Used for **private communication between resources**
- Every VM gets one automatically — you can't skip this
- IP comes from the **subnet's IP range** in the region where the VM lives

> 🧠 Simple English: This is like your house's internal room number in a big apartment building. Other tenants (VMs in the VPC) can find you using it, but the outside world (internet) has no idea it exists.
> 

### Two ways to assign an internal IP:

| Mode | How it works |
| --- | --- |
| **Auto** | GCP picks an available IP from the subnet range automatically. Used in Auto Mode VPCs — automatically selected if available. |
| **Custom** | You manually specify which IP you want (must be within the subnet range). Used in Custom VPCs — must be selected manually. |

Both Auto and Custom can be either **Ephemeral** or **Static**.

---

## 3. Ephemeral vs Static — The Core Difference

|  | Ephemeral | Static |
| --- | --- | --- |
| **What it is** | Temporary — assigned automatically | Reserved — stays until you release it |
| **When released** | On VM stop, restart, or delete | Only when you explicitly release it |
| **Use case** | Short-lived workloads, testing | Production VMs, anything that needs a fixed IP |
| **Cost** | Free (while in use) | Small charge when reserved but not attached |
| **Can be promoted?** | ✅ Yes — promote ephemeral → static | — |

> 🧠 Simple English:
Ephemeral = hotel room. You check out, someone else gets the room number.
Static = your permanent home address. Same address forever until you move out.
> 

> 💀 If your app hardcodes an IP address (like a config file saying "connect to 10.5.1.4"), and that VM uses ephemeral internal IP — the IP changes after a restart and your app breaks. Use static internal IPs for anything that other services depend on.
> 

---

## 4. Internal IP — Ephemeral Details

- IP comes from the **region's subnet** range
- Released **only when the instance or forwarding rule is deleted**
- (For internal — stopping doesn't always release it unlike external ephemeral)

> ⚠️ Internal ephemeral IPs are more stable than external ephemeral IPs. Internal IPs are released on deletion. External IPs are released on stop/restart/deletion.
> 

---

## 5. Alias IP (Optional — Internal)

> 🧠 Simple English: One VM, multiple internal IPs — without adding extra network cards. Useful when a single VM runs multiple containers or services that each need their own IP.
> 

**What it does:**

- Assign a **range of internal IPs** to a single VM's network interface
- Each container or app running on that VM gets its own IP from that range
- No need for a separate network interface (NIC) per container

**Where the IPs come from:**

- Subnet's **primary** IP range
- OR subnet's **secondary** IP ranges

**Example:**

```
VM: my-vm
  Primary NIC IP: 10.0.1.5
  Alias IP range: 10.0.1.10/28  (gives 14 extra IPs to containers on this VM)
```

> 📌 Alias IPs are commonly used with **GKE (Google Kubernetes Engine)** — each pod gets its own Alias IP from the node's range.
> 

---

## 6. External (Public) IP Address

- Needed to **communicate with the internet**, resources in another network, or public GCP services
- Sources **outside GCP** can reach your resource using this IP
- Only resources with an external IP can **send and receive traffic** directly to/from outside the VPC
- Optional — not every VM needs one

> 🧠 Simple English: This is your VM's public-facing street address. If you want the world to knock on your door, you need one. If your VM only talks to other VMs inside GCP, skip it — it just adds attack surface.
> 

### External Ephemeral IP:

- **Automatically assigned** when the VM is created (if you choose ephemeral)
- **Released** when the VM is stopped, restarted, or deleted
- Can be **promoted to Static** if you want to keep it

### External Static IP:

- **Reserved** — assigned to your project until you explicitly release it
- Survives VM restarts, deletions (it's yours until you let go)
- Available as:
    - **Regional** — used with one specific region's VMs/load balancers
    - **Global** — used with global load balancers (HTTPS LB, etc.)

> 💀 Unused static external IPs cost money. GCP charges you for reserved IPs that aren't attached to anything. Don't reserve IPs you're not using — release them. Your billing team will have opinions on this.
> 

---

## 7. Reserving IP Addresses

### Internal IP Reservation — Two Ways:

**Method 1 (Reserve first, attach later):**

```
Unreserved (not in use)
    ↓ [1A. Create reserved internal IP]
Reserved (not in use)
    ↓ [1B. Create VM with this reserved IP]
Reserved (in use) ✅
```

**Method 2 (Use ephemeral first, then reserve):**

```
Unreserved (not in use)
    ↓ [2A. Create VM → gets auto ephemeral IP]
Unreserved (in use) — e.g. 10.12.4.3
    ↓ [2B. Promote to static]
Reserved (in use) ✅ — still 10.12.4.3
```

> 🧠 Simple English: You can either book a specific room before you arrive (Method 1), or get whatever room is available when you arrive and then tell the hotel "I want to keep this room number forever" (Method 2). Both work.
> 

---

### External IP Reservation — Two Ways:

Same logic as internal — either reserve first or promote from ephemeral.

**gcloud commands:**

```bash
# Reserve a Regional static external IP
gcloud compute addresses create ADDRESS_NAME \
  --region REGION

# Reserve a Global static external IP (IPv4 or IPv6)
gcloud compute addresses create ADDRESS_NAME \
  --global \
  --ip-version [IPV4 | IPV6]
```

| Type | Scope | Used with |
| --- | --- | --- |
| **Regional** | One region | VMs, regional load balancers |
| **Global** | Worldwide | Global HTTPS load balancers |

---

## 8. Promote Ephemeral → Static

You can promote any ephemeral IP (internal or external) to static at any time — **the IP address stays the same**, it just becomes permanent.

```
Ephemeral IP: 34.102.5.11
    ↓ [promote to static]
Static IP: 34.102.5.11  ← same IP, now permanently yours
```

> This is useful when you spin up a VM, get an ephemeral IP, configure DNS or firewall rules around it, and then realise "I should keep this IP." Promote it — don't recreate it.
> 

---

## 9. Quick Reference Cheat Sheet

| Concept | Key Point |
| --- | --- |
| Internal IP | Private, always assigned, never exposed to internet |
| External IP | Public, optional, required for internet access |
| Auto internal | GCP picks IP from subnet (Auto mode VPC) |
| Custom internal | You pick the IP (Custom mode VPC) |
| Ephemeral internal | Released on deletion (more stable than external ephemeral) |
| Ephemeral external | Released on **stop, restart, or delete** |
| Static IP | Reserved, yours until you release — survives restarts |
| Static external cost | Charged even when not attached to a resource |
| Alias IP | Multiple IPs on one NIC — for containers/pods |
| Alias IP source | Subnet primary or secondary ranges |
| Regional static | Used with regional VMs / LBs |
| Global static | Used with global load balancers only |
| Promote ephemeral | Same IP, becomes permanent |

---

> 💀 The IP addressing trap on the exam:
"Ephemeral external IP is released when the VM **stops**."
"Ephemeral internal IP is released when the VM is **deleted**."
They behave differently. Don't mix them up — examiners love this distinction.
> 

---

*Source: GCP Associate Cloud Engineer (ACE) prep — exampro.co/gcp-ace*

# GCP VPC — Firewall Rules

> Topic: VPC Firewall Rules — Characteristics, Components, Implied Rules & Default Rules
🔥 Firewalls are your VPC's bouncer. They decide who gets in and who gets thrown out.
> 

---

## 1. What are VPC Firewall Rules?

Firewall rules **control traffic flowing in and out of your VPC** at the VM instance level.

> 🧠 Simple English: Every packet trying to enter or leave your VM goes through the firewall first. The firewall checks it against the rules, and either lets it through or drops it. No rule matches? Implied rules kick in (deny incoming, allow outgoing).
> 

Each firewall rule controls either:

- **Incoming (Ingress)** traffic — traffic coming INTO the VM
- **Outgoing (Egress)** traffic — traffic going OUT of the VM
- One rule = one direction. **NOT BOTH.**

Each rule is defined by: `protocol`, `ports`, `sources`, `destinations`, `target`

---

## 2. Firewall Rule Characteristics

| Characteristic | Detail |
| --- | --- |
| **Direction** | Ingress OR Egress — one rule cannot be both |
| **IPv4 only** | Firewall rules only support IPv4 addresses (as of standard setup) |
| **Action** | Allow OR Deny — one rule cannot be both |
| **VPC scope** | Rules apply at VPC level — affect all VMs in the VPC (unless targeted) |
| **Stateful** | ✅ Yes — if a connection is allowed in, return traffic is automatically allowed out (no need for separate egress rule) |
| **No rule = implied rules apply** | GCP has implied rules that always apply regardless of your config |

> 🧠 **Stateful** = you don't need to write two rules for a conversation. If you allow SSH ingress on port 22, the return traffic (the VM's response) is automatically allowed out. The firewall remembers the connection.
> 
> 
> Contrast with stateless firewalls (like some ACLs) where you'd need to explicitly allow both directions. GCP firewalls are stateful — much less painful.
> 

---

## 3. Firewall Rule Components (All Fields Explained)

When creating a firewall rule, here's every field you'll see:

| Field | What it means |
| --- | --- |
| **Network** | Which VPC this rule belongs to |
| **Priority** | Number from 0–65535. Lower = higher priority. `0` = highest, `65535` = lowest |
| **Direction** | Ingress (incoming) or Egress (outgoing) |
| **Action on match** | Allow or Deny |
| **Targets** | Which VMs this rule applies to — all instances, or specific network tags / service accounts |
| **Target tags** | Apply rule only to VMs that have this network tag |
| **Source filter** | For Ingress: where traffic is coming FROM (IP ranges, source tags, source service account) |
| **Source IP ranges** | CIDR range of allowed/denied sources |
| **Second source filter** | Add a second source filter (e.g. source tag AND IP range) |
| **Protocols and ports** | Which protocols (TCP, UDP, ICMP, etc.) and ports to match |
| **Enforcement** | Enable or Disable the rule without deleting it |

### Priority Rules:

- Range: `0` to `65535`
- Lower number = **higher priority** (wins over higher-numbered rules)
- `0` = highest priority possible
- `65534` = priority used by default pre-populated rules
- `65535` = lowest (implied rules live here)

> 🧠 Simple English: If you have two rules that conflict — "allow port 22 from anywhere" (priority 1000) and "deny port 22 from anywhere" (priority 500) — the deny wins because 500 < 1000.
> 

### Source Filter Options (Ingress only):

- **IP ranges** — allow traffic from specific CIDR blocks
- **Source tags** — allow traffic from VMs that have a specific network tag
- **Source service account** — allow traffic from VMs running a specific service account

> Source tags are powerful for internal traffic rules. Instead of tracking IPs, you tag your web servers "web-tier" and say "only allow port 3306 from VMs tagged web-tier." The tag travels with the VM — no IP management needed.
> 

---

## 4. Implied Rules — Always Present, Can't Delete

GCP has **two implied rules** that exist in every VPC and cannot be deleted or modified. They are the last resort when no other rule matches.

| Rule | Direction | Action | Priority |
| --- | --- | --- | --- |
| **Implied allow egress** | Egress (outgoing) | Allow all | 65535 (lowest) |
| **Implied deny ingress** | Ingress (incoming) | Deny all | 65535 (lowest) |

> 🧠 Simple English:
By default: **everything going out is allowed, everything coming in is blocked.**
You add rules to open specific holes in the incoming wall.
You add rules to close specific holes in the outgoing wall.
> 

### Special Always-Allowed Traffic (not in routing table, always passes):

GCP always allows traffic to/from the **Metadata Server** regardless of firewall rules:

- IP: `169.254.169.254`
- Protocols: TCP, UDP, ICMP, GRE
- Services: DHCP, DNS, Instance Metadata, NTP

> You cannot block this with firewall rules. GCP needs it to function. The metadata server is how your VM knows its own name, project, service account, startup scripts, etc.
> 

### Always-Blocked Traffic:

- **TCP Port 25** (outbound SMTP) is always blocked on external IPs
- GCP blocks this to prevent spam/email abuse from their infrastructure

---

## 5. Pre-populated Rules in Default VPC

The **Default VPC** comes with 4 pre-populated firewall rules (priority 65534):

| Rule Name | Direction | Source | Protocol/Port | Action |
| --- | --- | --- | --- | --- |
| `default-allow-icmp` | Ingress | 0.0.0.0/0 (anywhere) | ICMP | Allow |
| `default-allow-internal` | Ingress | 10.128.0.0/9 (internal) | All | Allow |
| `default-allow-rdp` | Ingress | 0.0.0.0/0 (anywhere) | TCP:3389 | Allow |
| `default-allow-ssh` | Ingress | 0.0.0.0/0 (anywhere) | TCP:22 | Allow |

> 🧠 Simple English: The default VPC is very "friendly" — it allows SSH, RDP, and ping from literally anywhere on the internet. Great for getting started in 5 minutes. Terrible for production security. The first thing you should do in production is delete or restrict these rules.
> 

> 💀 `default-allow-ssh` from `0.0.0.0/0` means anyone in the world can attempt to SSH into your VMs. Your VMs are protected only by SSH key authentication. One leaked key and it's over. In production: restrict SSH to specific IP ranges (like your company VPN) or use Identity-Aware Proxy (IAP) instead.
> 

---

## 6. Ingress vs Egress — Quick Summary

|  | Ingress | Egress |
| --- | --- | --- |
| **Traffic direction** | INTO the VM | OUT of the VM |
| **Implied default** | Deny all | Allow all |
| **Source filter** | Source IPs, source tags, source SA | — |
| **Destination filter** | — | Destination IPs, destination tags |
| **Common use** | Opening ports (SSH, HTTP, HTTPS) | Restricting outbound traffic |

---

## 7. Enabling / Disabling Rules Without Deleting

You can **disable** a firewall rule without deleting it — useful for testing or temporarily suspending a rule.

`Enforcement: Enabled / Disabled`

> 🧠 Simple English: Like turning off a light switch without unscrewing the bulb. The rule is still there, just not active. Re-enable it anytime.
> 

---

## 8. Quick Reference Cheat Sheet

| Concept | Key Point |
| --- | --- |
| One rule = | One direction only (ingress OR egress) |
| Stateful | ✅ Return traffic auto-allowed |
| Priority range | 0 (highest) to 65535 (lowest) |
| Implied egress | Allow all outbound — priority 65535 |
| Implied ingress | Deny all inbound — priority 65535 |
| Metadata server | `169.254.169.254` — always reachable, can't be blocked |
| TCP port 25 | Always blocked outbound (anti-spam) |
| Default VPC rules priority | 65534 |
| Default VPC SSH rule | Allow TCP:22 from 0.0.0.0/0 — ⚠️ restrict in production |
| Target tags | Apply rule to specific VMs by tag, not all VMs |
| Source tags | Allow traffic from specific tagged VMs |
| Disable rule | Enforcement toggle — keep rule but suspend it |

---

> 💀 The classic firewall mistake: Creating an "allow all" ingress rule on port 80 to fix a web app that can't be reached. Yes, it works. You've also just opened port 80 to the entire internet on every VM in the VPC with that tag. 
Read what "Apply to all" actually means before clicking Create.
> 
> 
> And if you're wondering why your VM can't ping anything — you probably have a deny egress rule you forgot about, because the implied rule allows egress. Someone added a deny and never told you. Check priority 0–65534 egress rules first.
> 

---

*Source: GCP Associate Cloud Engineer (ACE) prep — exampro.co/gcp-ace*

# GCP Shared VPC

> Topic: Shared VPC — Architecture, Roles, Use Cases & Real-World Patterns
🏢 One network, many tenants. The corporate landlord model of GCP networking.
> 

---

## 1. What is Shared VPC?

Shared VPC allows **multiple projects (service projects) to use the subnets of a single VPC network owned by one central project (host project)**.

Resources in different service projects — or between a service project and the host project — can **communicate using private internal IPs** through the shared network.

> 🧠 Simple English: Instead of each team building their own separate network, one team (or a dedicated network team) builds and manages one big network. All other teams just plug their VMs into that network's subnets. Centralized control, separate billing and IAM.
> 

### Core Rule:

**A project can be either a Host project OR a Service project — never both at the same time.**

---

## 2. Key Concepts

| Term | What it means |
| --- | --- |
| **Host Project** | The project that owns and manages the Shared VPC network and subnets |
| **Service Project** | A project that is attached to the host project and uses its subnets |
| **Shared VPC Network** | The VPC network in the host project that gets shared |
| **Shared VPC Admin** | The person/role that sets up and manages the Shared VPC (org-level admin) |
| **Service Project Admin** | The person/role that manages resources within a service project |
| **Standalone Project** | A project that has its own independent VPC — not part of any Shared VPC |

---

## 3. How it Works

```
Host Project
   └── Shared VPC Network
         ├── subnet-1 (10.0.2.0/24) — us-west1
         └── subnet-2 (10.10.4.0/24) — us-central1

Service Project A → VM1 gets internal IP 10.0.2.15 from subnet-1
Service Project B → VM2 gets internal IP 10.10.4.6 from subnet-2

VM1 ←──(private IP communication)──→ VM2  ✅
```

- VM1 and VM2 live in **different service projects** but share the **same VPC network**
- They communicate over **internal IPs** — no internet, no VPN, no peering needed
- The **host project manages the network** — subnets, firewall rules, routes
- The **service projects manage their own VMs** — but use the host's subnets

---

## 4. IAM Roles in Shared VPC

Two levels of admin:

### Shared VPC Admin

- Has **org-level** permissions
- Enables Shared VPC on the host project
- Attaches service projects to the host
- Grants subnet access to service project admins
- Permissions: **project-level** OR **subnet-level**

### Service Project Admin

- Manages resources (VMs, services) within the service project
- Can create VMs in the subnets they are granted access to
- Can be granted access to:
    - **All subnets** in the host project
    - **Specific subnets** only (subnet-level permission — more secure)

> 🧠 Simple English: The Shared VPC Admin is the building manager who decides which tenant gets access to which floor. The Service Project Admin is the tenant — they can manage their apartment but can't change the building's plumbing.
> 

> 💡 Subnet-level permissions are better for security. If Team A only needs subnet-1, don't give them access to all subnets. Least privilege applies here too.
> 

---

## 5. Use Case 1 — Basic Shared VPC

Simple scenario: two service projects sharing one host project's network.

```
Service Project A                Service Project B
     VM1 (10.0.2.15)                  VM2 (10.10.4.6)
         |                                  |
         └──────────────┬───────────────────┘
                        ↓
               Host Project
               Shared VPC Network
               subnet-1: 10.0.2.0/24 (us-west1)
               subnet-2: 10.10.4.0/24 (us-central1)
```

> Also — a **Standalone Project** (with its own independent network) can exist alongside the Shared VPC setup. It's just not connected to the shared network.
> 

---

## 6. Use Case 2 — Multiple Host Projects

Large enterprises often have **separate host projects per environment** (dev and prod):

```
Dev-A Service Project  |  Dev-B Service Project
          ↓                        ↓
   Development Host Project (Development Network)
   subnet-1: 10.0.2.0/24   subnet-2: 10.10.4.0/24

Prod-A Service Project  |  Prod-B Service Project
          ↓                        ↓
   Production Host Project (Production Network)
   subnet-1: 10.0.2.0/24   subnet-2: 10.10.4.0/24
```

> 🧠 Simple English: Dev and Prod are completely separate networks with separate host projects. This gives you hard isolation between environments — a mistake in dev can't accidentally reach prod. No accidental cross-contamination between environments.
> 

> 💀 Sharing one host project between dev and prod to "save effort" is how dev workloads end up on the same network as production databases. Seen it. Don't do it.
> 

---

## 7. Use Case 3 — Hybrid Environment

Shared VPC + Cloud VPN = connect your on-premises network to all service projects through one VPN tunnel in the host project:

```
Service Project A    Service Project B
   prod-a (VM)           prod-b (VM)
       ↓                     ↓
       └──────┬──────────────┘
              ↓
       Host Project
       Shared VPC Network
       subnet-1: 10.0.2.0/24
       subnet-2: 10.10.4.0/24
              |
         [Cloud VPN]
              |
         Internet
              |
    On-Premises Network
    (Customer VPN Gateway)
```

**Key benefit:** You only need **one Cloud VPN tunnel** in the host project. All service projects automatically get connectivity to on-premises through it — no separate VPN setup per service project.

> 🧠 Simple English: Instead of each project setting up its own VPN to your office — one tunnel in the host project serves everyone. Central connectivity management. Much cheaper and simpler.
> 

---

## 8. Use Case 4 — Two-Tier Web Service (Real-World Architecture)

Classic production pattern using Shared VPC with two separate service projects for two application tiers:

```
External Client
    ↓
External IP Address → HTTP(S) Load Balancer (Tier 1 Service Project)
    ↓
Tier 1 Instances (10.0.2.3, 10.0.2.4, 10.0.2.5) — subnet-1
    ↓
Internal Load Balancer (Tier 2 Service Project)
    ↓
Tier 2 Instances (10.0.3.3, 10.0.3.4, 10.0.3.5) — subnet-2
    ↓
(All IPs live in Host Project's Shared VPC Network)
```

**Structure:**

- Tier 1 Service Project = frontend/web tier (public-facing)
- Tier 2 Service Project = backend/app tier (internal only)
- Host Project = owns the Shared VPC with subnet-1 and subnet-2
- Both tiers communicate via **internal IPs** over the shared network
- Separate IAM and billing per tier/project

> 🧠 Simple English: Web servers in one project, app servers in another, network owned by a third dedicated project. Each team manages their own VMs with their own IAM — but the network is centralised and secure.
> 

---

## 9. Shared VPC vs VPC Peering — When to Use Which

|  | Shared VPC | VPC Peering |
| --- | --- | --- |
| **Same org only?** | ✅ Yes | ❌ No — works across orgs |
| **Network ownership** | Host project owns it all | Each VPC owns its own network |
| **Admin model** | Centralized (host project manages network) | Decentralized (each team manages their VPC) |
| **Billing** | Separate per project | Separate per project |
| **IAM separation** | ✅ Yes | ✅ Yes |
| **Best for** | One org, multiple teams sharing a network | Two separate teams/orgs needing private connectivity |
| **Transitive comms** | ✅ All service projects can talk to each other | ❌ Not transitive |

---

## 10. Quick Reference Cheat Sheet

| Concept | Key Point |
| --- | --- |
| Shared VPC scope | Same organization only |
| Host project | Owns and manages the VPC network |
| Service project | Uses host project's subnets |
| One project = host OR service | Never both simultaneously |
| Communication method | Internal private IPs across service projects |
| Shared VPC Admin | Org-level, sets up the shared VPC |
| Service Project Admin | Project-level, manages VMs in service project |
| Subnet-level permissions | Grant access to specific subnets only (more secure) |
| Multiple host projects | Use separate host projects for dev and prod |
| Hybrid connectivity | One Cloud VPN in host project serves all service projects |
| Standalone project | Has its own independent VPC, not part of Shared VPC |

---

> 💀 The most common Shared VPC mistake: Giving a Service Project Admin access to ALL subnets "because it's easier." Now the dev team can deploy VMs in the prod subnet. Enjoy explaining that to your security team during the audit.
> 
> 
> Subnet-level permissions exist for a reason. Use them.
> 

---

*Source: GCP Associate Cloud Engineer (ACE) prep — exampro.co/gcp-ace*

# GCP VPC — Flow Logs

> Topic: VPC Flow Logs — What they are, use cases, record format & how to view them
📊 Flow logs are your network's black box recorder. Everything that flew through your VPC — logged.
> 

---

## 1. What are VPC Flow Logs?

VPC Flow Logs **record network traffic flows** sent from and received by VM instances, including GKE nodes. Every connection that passes through a subnet is captured.

> 🧠 Simple English: Imagine a CCTV camera but for network packets. Every time a VM sends or receives traffic, Flow Logs writes down: who sent it, who received it, how many bytes, which port, what protocol, and when. It's not the actual packet content — just the metadata/summary of the connection.
> 

### Key Facts:

- Enabled **per subnet** (not per VPC)
- Logs go to **Cloud Logging (Stackdriver)** — stored for **30 days** by default
- For **long-term storage** → export/sink logs to **Cloud Storage**
- Can also export to **BigQuery** for analysis or **Pub/Sub** for real-time streaming to SIEM tools

```
VPC Subnet (flow logs enabled)
    ↓  [captures all traffic flows]
Cloud Logging (30-day retention)
    ↓  [export sink]
Cloud Storage (long-term) OR BigQuery (analysis) OR Pub/Sub (real-time SIEM)
```

---

## 2. Use Cases

### Network Monitoring

- Real-time visibility into **network throughput and performance**
- See which VMs are sending the most traffic
- Detect unusual bandwidth spikes

### Analyse Network Usage & Optimise Costs

- Understand traffic patterns — where is traffic going?
- Identify expensive cross-region or egress traffic
- **Reduce network costs** by routing traffic more efficiently based on actual usage data

> 🧠 Simple English: If you're paying $3000/month in egress fees and have no idea why — Flow Logs will show you exactly which VMs are sending data where, and how much. Money well spent on the logging.
> 

### Network Forensics

- When an **incident occurs** — "who talked to who, and when?"
- Reconstruct the timeline of a security breach
- Trace suspicious connections back to their source
- Your lawyer will want these logs. Have them ready.

### Real-Time Security Analysis

- Stream flow logs to **Pub/Sub** → integrate with SIEM tools like:
    - **Splunk**
    - **Rapid7**
    - **LogRhythm**
- Detect threats and anomalies in real time as traffic flows

> 🧠 Simple English: Instead of investigating after an attack, you can detect it while it's happening. Flow Logs → Pub/Sub → Splunk = your security team gets an alert while the attacker is still in your network.
> 

---

---

## 3. How to View Flow Logs

### In Cloud Logging (Logs Viewer):

Filter by:

- **Resource type:** `gce_subnetwork`
- **Log name:** `compute.googleapis.com/vpc_flows`

Query example:

```
resource.type="gce_subnetwork"
logName="projects/YOUR-PROJECT/logs/compute.googleapis.com%2Fvpc_flows"
timestamp>="2024-01-01T00:00:00.000Z" timestamp<="2024-01-07T00:00:00.000Z"
```

### What a log entry looks like:

```json
{
  "dest_vpc": { ... },
  "dest_location": { ... },
  "src_location": { ... },
  "bytes_sent": "0",
  "reporter": "SRC",
  "connection": {
    "src_ip": "10.0.0.5",
    "dest_ip": "10.4.0.6",
    "src_port": 39732,
    "dest_port": 443,
    "protocol": 6
  },
  "start_time": "2020-10-06T20:43:00Z",
  "end_time": "2020-10-06T20:42:55Z"
}
```

---

## 4. Enabling Flow Logs

Flow logs are enabled at the **subnet level**:

```
VPC Network → Subnets → Edit subnet → Flow Logs: ON
```

Or via gcloud:

```bash
gcloud compute networks subnets update SUBNET_NAME \
  --region REGION \
  --enable-flow-logs
```

> ⚠️ Flow logs add cost — you pay for the log ingestion into Cloud Logging. For high-traffic subnets, this can add up. You can configure the **aggregation interval** and **sampling rate** to control cost vs detail:
> 
> - **Aggregation interval:** 5 sec (default) to 15 min
> - **Sampling rate:** 0.5 (50% of flows sampled) by default — can go 0.0 to 1.0

---

## 5. Quick Reference Cheat Sheet

| Concept | Key Point |
| --- | --- |
| Enabled at | Subnet level |
| Default retention | 30 days in Cloud Logging |
| Long-term storage | Export to Cloud Storage |
| Analysis | Export to BigQuery |
| Real-time streaming | Export to Pub/Sub → SIEM |
| Core fields | connection, timestamps, bytes, packets, rtt, reporter |
| IP fields | src_ip, src_port, dest_ip, dest_port, protocol |
| Metadata fields | instance, VPC, location, GKE details |
| Log resource type | `gce_subnetwork` |
| Log name filter | `compute.googleapis.com/vpc_flows` |
| Use cases | Network monitoring, cost optimisation, forensics, security |
| SIEM integration | Pub/Sub → Splunk / Rapid7 / LogRhythm |
| Sampling rate default | 50% of flows |
| Cost consideration | Charged per log ingestion — tune sampling rate for cost control |

---

> 💀 "We don't need flow logs, we'll turn them on if there's an incident."
Congratulations — you've just made your incident response useless.
Flow logs only capture traffic going forward from when they're enabled.
By the time you turn them on after an incident starts, the interesting part is already gone — unlogged, unrecoverable, and probably on page 4 of the incident report under "unknown."
> 
> 
> Turn them on proactively. Export to Cloud Storage for retention. Thank yourself later.
> 

---

*Source: GCP Associate Cloud Engineer (ACE) prep — exampro.co/gcp-ace*

# CLOUD DNS

WE CAN CREATE PUBLIC ZONE AND PRIVATE ZONE 

PUBLIC DNS ZONE WILL BE AVAILABE TO INTERNET :

EG.  [abc-app.org](http://abc-app.org) 

we can create private DNS ZONE ALSO 

THAT DOMAIN NAME WILL BE ONLY AWAILABLE IN YOUR SELECTED VPC-NETWORK 

NOT TO THE INTERNET 

MIGHT BE USEFUL
