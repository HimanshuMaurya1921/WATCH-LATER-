#K8s EVENTS

## 🧠 What Events actually are

Events are:

> “A timeline of decisions Kubernetes made about your resources.”

They answer:

* Why did my pod restart?
* Why didn’t it schedule?
* Who killed it?
* What is Kubernetes trying to tell me (passive-aggressively)?

---

# 🔍 First command (your new reflex)

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

👉 Always sort. Otherwise it’s chaos.

---

# 🎯 Narrow it down (important)

## For a specific pod:

```bash
kubectl describe pod <pod-name>
```

Scroll to:

```text
Events:
```

👉 This is where truth lives.

---

# 🧩 Anatomy of an Event

Example:

```text
Warning  FailedScheduling  2m   default-scheduler  0/3 nodes available: insufficient memory
```

Break it down:

| Field   | Meaning          |
| ------- | ---------------- |
| Type    | Normal / Warning |
| Reason  | What happened    |
| Age     | When             |
| Source  | Who triggered it |
| Message | Details          |

---

# 🔥 Common Event Patterns (learn these = instant debugging)

---

## 💣 1. FailedScheduling

```text
0/3 nodes available: insufficient cpu
```

👉 Pod can’t be placed

### Causes:

* Not enough resources
* Node selectors / taints

---

## 💀 2. CrashLoopBackOff

```text
Back-off restarting failed container
```

👉 App crashing repeatedly

---

## 🔐 3. FailedMount

```text
Unable to mount volumes
```

👉 ConfigMap / Secret / PVC issue

---

## 🌐 4. FailedCreatePodSandBox

```text
network plugin failed
```

👉 CNI problem (network layer)

---

## 🔥 5. Unhealthy (probe failures)

```text
Liveness probe failed
```

👉 Kubernetes is killing your pod

---

## ⚠️ 6. ImagePullBackOff

```text
Failed to pull image
```

👉 Registry / auth / typo

---

# 🕵️ Reading Events Like a Detective

---

## 💣 Scenario 1: Pod not starting

You run:

```bash
kubectl describe pod my-app
```

See:

```text
Warning  FailedScheduling  
0/5 nodes available: insufficient memory
```

---

### 🧠 Conclusion:

👉 Not a code issue
👉 Not a container issue
👉 It’s **cluster capacity**

---

---

## 💣 Scenario 2: Pod keeps restarting

Events:

```text
Warning  BackOff  
Back-off restarting failed container
```

---

### 🧠 Next step:

👉 Check logs

Events told you:

> “It’s crashing — go look why”

---

---

## 💣 Scenario 3: Service not reachable

Events:

```text
Warning  FailedCreatePodSandBox  
network plugin failed
```

---

### 🧠 Conclusion:

👉 Networking problem (CNI), not app

---

---

## 💣 Scenario 4: Pod stuck in ContainerCreating

Events:

```text
Warning  FailedMount  
Unable to attach or mount volumes
```

---

### 🧠 Conclusion:

👉 Storage issue (PVC, volume, permissions)

---

# 📊 What events look like in real clusters

![Image](https://images.openai.com/static-rsc-4/rwbi3iSUOmp-AtDJ8gMYnj8dRqxoNhVsKlWlJDTaMgyda6mCgN_y1zjHt4n4y0BahcC8dvEkSX0QwYvN4NuEzInMMv9PdNaa-aFDvGrPSciiS6kSB9FuBIJgTGsPqKI7KalasKfhlrTVI6PIPAry9Qw8--wfI7qR1nymoOUWqaPOpYhJIvJFpwIrX7OBDQoO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SzR1huxa-n-Th0EaKnJIy0FcS8VGhg4shaEq89ULrp-eeEYYfY2yFVdf2G44whpYHcaHSsDAPiGye_l8iMEZeezGwBZd-ARBxD7HRixSqPyo4KyJ2-pHS6EliYIMcNPF3aj_xj0ww0nPSKAoZ0kJ8CIZ1vqIOSLswEnB6LJsOzBWITXxLIu4Sh4ZJWOPYtUN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/x3AOiBiE-_N8B8nx4hkdAJBe2wMb3vv1G_e44eHOAc0oecYnILtrfl3ixHWYHc1w4-8dMyetkw-517iPRqq5eKkryBzkKWScu4tDuokWJ65BIGlWgROER6c9UryWArajfJuDysaB6DsajinBmZ1gI7Wn8qfI6YhviTz3tryHzQnaBRQxEjRSj_E94iYGUibQ?purpose=fullsize)


---

# 🧠 Advanced Tricks

---

## 🔥 Filter only warnings

```bash
kubectl get events --field-selector type=Warning
```

👉 Cuts noise instantly

---

## 🔍 Watch events live

```bash
kubectl get events --watch
```

👉 Real-time debugging (very powerful)

---

## 🎯 Events for specific object

```bash
kubectl get events --field-selector involvedObject.name=<pod-name>
```

---

# ⚔️ Events vs Logs (critical distinction)

| Events        | Logs         |
| ------------- | ------------ |
| Cluster-level | App-level    |
| Why K8s acted | What app did |
| Short-lived   | Persistent   |
| High signal   | Often noisy  |

---

# 💀 Common Mistakes

---

### ❌ Ignoring events

👉 You miss the root cause

---

### ❌ Jumping straight to logs

👉 You debug the wrong layer

---

### ❌ Not sorting events

👉 Timeline becomes useless

---

### ❌ Missing scheduling errors

👉 Waste time on app instead of infra

---

# 🧠 Pro Debug Flow (add this to your brain)

```text
Events → Logs → Metrics → Network
```

Events FIRST.

Always.

---

# ⚡ Real DevOps Insight

Events are like:

> Kubernetes whispering: “Here’s exactly what went wrong… if you bother to read.”

---

# 💀 Brutal Truth

Most engineers:

* Ignore events
* Restart things
* Hope it works

Good engineers:

* Read events
* Understand cause
* Fix once
