
# 🛠️ ArgoCD Debug Checklist (When Sync Fails)

## 🚨 Start Here (don’t overthink it yet)

When you see:

> ❌ Sync failed / blocked

Your brain says: “cluster issue?”
Reality: **it’s almost always AppProject validation**

---

# 🔍 1. Check the Error Message (yes, actually read it)

Common ones:

* `namespace not permitted`
* `application destination is not permitted`
* `source repo not permitted`

👉 ArgoCD is blunt. It tells you exactly what rule you broke.

---

# 🧪 2. Verify the Application Spec

Look at the child app:

```yaml
spec:
  project: ???
  source:
    repoURL: ???
  destination:
    namespace: ???
    server: ???
```

### Ask:

* Is `project` correct?
* Is `namespace` correct?
* Is `repoURL` correct?

👉 One typo = full denial. ArgoCD doesn’t do “close enough”.

---

# 🔐 3. Open the AppProject (the real judge)

```yaml
spec:
  destinations:
  sourceRepos:
```

---

## ✅ 3A. Destination Check

Compare:

```yaml
Application:
  destination.namespace: frontend
```

vs

```yaml
AppProject:
  destinations:
    - namespace: frontend
```

### Must match EXACTLY:

* `frontend` ≠ `Frontend`
* `frontend` ≠ `frontend ` (yes, spaces matter… enjoy that)

---

## ✅ 3B. Server Check

Usually:

```yaml
https://kubernetes.default.svc
```

Mismatch = ❌ blocked

---

## ✅ 3C. Repo Check

```yaml
AppProject:
  sourceRepos:
    - https://github.com/org/repo.git
```

If your app uses a repo not listed:

❌ blocked

👉 ArgoCD: “Unknown repo? Absolutely not.”

---

# 🔄 4. Confirm Which Project Is Actually Used

This is where people mess up:

```yaml
spec:
  project: project-1
```

### Then ONLY check:

```yaml
AppProject: project-1
```

NOT:

* root app project
* default project (unless explicitly used)

👉 If you check the wrong project, congrats, you just debugged fiction.

---

# 🧨 5. Root App Trap (classic mistake)

If root app fails:

Check:

```yaml
destination:
  namespace: argocd
```

Then ensure its project allows:

```yaml
destinations:
  - namespace: argocd
```

👉 Yes, even the root app needs permission to exist. Democracy.

---

# 🧪 6. Quick Sanity Commands

If using CLI:

```bash
kubectl get app -n argocd
kubectl describe app <app-name> -n argocd
```

Look for:

* Events
* Validation errors

---

# 🧠 7. Fast Decision Tree (use this when tired)

```id="9lkrwp"
Sync failed?
   ↓
Check error message
   ↓
namespace error? → check destinations.namespace
repo error?      → check sourceRepos
project error?   → check project field
   ↓
Still broken?
   ↓
You checked the wrong AppProject. Go back.
```

---

# ⚠️ 8. Top 5 Real-World Causes

Because patterns > theory:

1. **Namespace typo**

   ```
   frontend vs front-end
   ```

2. **Wrong project in Application**

   ```
   project: default (oops)
   ```

3. **Repo not in sourceRepos**

4. **Root app missing argocd namespace permission**

5. **Case sensitivity**

   ```
   Frontend ≠ frontend
   ```

---

# 🧨 Final Reminder (tattoo-level important)

> **ArgoCD does not care who created the Application.
> It only cares what project it declares.**

---

If ArgoCD blocks you, it’s not broken.

It’s just doing exactly what you told it to do…
which is somehow worse.
