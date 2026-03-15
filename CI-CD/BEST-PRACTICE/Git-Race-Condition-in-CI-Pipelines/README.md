# Git Race Condition in CI Pipelines (and the Fix)

## Problem

In CI/CD pipelines, a common workflow looks like this:

```bash
checkout main → build → update values.yaml → commit → push
```

Between the **checkout** and the **push**, another developer (or another pipeline) may push a commit to `main`.

### Example timeline

```
Pipeline:   checkout ───────────── commit ── push ❌
Teammate:           ───── push new commit
```

Now the pipeline is trying to push based on an **outdated commit**.

This causes:

```
! [rejected] main -> main (non-fast-forward)
```

Git refuses the push because the remote branch has moved ahead.
This situation is known as a **Git race condition**.

---

## Solution: Rebase Before Push

The pipeline must **synchronize with the remote branch before pushing**.

Correct pipeline flow:

```bash
git add values.yaml
git commit -m "update image tag"
git pull --rebase origin main
git push origin main
```

### What this does

1. Fetch latest commits from the remote repository.
2. Replay the pipeline commit **on top of the latest commit**.
3. Push a clean linear history.

Before rebase:

```
Remote: A --- B --- C
Local : A --- B --- P
```

After `git pull --rebase`:

```
A --- B --- C --- P
```

The pipeline commit (`P`) moves to the latest position. Push succeeds.

---

## Why Rebase Instead of Merge?

The default `git pull` performs a **merge**.

History with merge:

```
A --- B --- C
       \     \
        P ---- M
```

This creates a **merge commit (`M`)**.

In GitOps repositories (Helm charts, Kubernetes manifests), we prefer **clean linear history**
because the commit history acts as a **deployment audit log**.

Using rebase keeps history simple:

```
A --- B --- C --- P
```

No extra merge commits.

---

## Important Safety Rule

Never stage the entire workspace in CI.

❌ Dangerous:

```bash
git add .
```

CI workspaces often contain:

```
node_modules/
dist/
tmp/
coverage/
build/
```

Accidentally committing these files can pollute the GitOps repository.
Always stage **specific files only**.

✅ Safe:

```bash
git add values.yaml
```

---

## Why Conflicts Are Rare in GitOps Pipelines

Typically responsibilities are separated:

| Actor       | What they edit               |
|-------------|------------------------------|
| Developers  | replicas, env vars, resources |
| DevOps      | Helm templates               |
| CI Pipeline | **image tag or digest only** |

Example:

```yaml
image:
  repository: myapp
  tag: sha123
```

Since the pipeline edits **only the tag**, conflicts are extremely rare.

---

## Advanced Issue: Multiple Pipelines Running at the Same Time

Even with `git pull --rebase`, a race condition can still occur when
**two pipelines run simultaneously**.

Example timeline:

```
Pipeline A: checkout → build → commit
Pipeline B: checkout → build → commit
```

Both pipelines start from the same commit.

1. Pipeline A rebases and pushes first:

```
A --- B --- PA
```

2. Pipeline B now tries to push:

```
A --- B --- PB  ❌ rejected — remote already at PA
```

---

## How Large Companies Solve This: Push Retry Loop

Many production CI systems implement a **retry mechanism**.

If push fails, the pipeline:
1. Pulls the latest commits again
2. Rebases again
3. Attempts to push again

```bash
MAX_RETRIES=3
COUNT=0

until git push origin main; do
  COUNT=$((COUNT+1))
  if [ $COUNT -ge $MAX_RETRIES ]; then
    echo "Push failed after multiple retries"
    exit 1
  fi
  echo "Push rejected. Rebasing and retrying..."
  git pull --rebase origin main
done
```

### What this achieves

If another pipeline pushes first:

```
Pipeline A → push success
Pipeline B → push rejected → pull --rebase → retry push → success
```

Result:

```
A --- B --- PA --- PB
```

Both changes land. No manual intervention required.

---

## Difference: `git rebase` vs `git pull --rebase`

| Command              | What it does                                              |
|----------------------|-----------------------------------------------------------|
| `git rebase`         | Reapply local commits on top of another branch            |
| `git pull --rebase`  | Fetch remote commits **and then run rebase automatically** |

Manual equivalent:

```bash
git fetch origin
git rebase origin/main
```

Shortcut:

```bash
git pull --rebase origin main
```

---

## ⚠️ Important: Watch Your Shared Library

If your `git_commit_and_push` shared library wraps the push in a `try/catch`
and swallows the exit code, the retry loop will **never trigger** — it always
sees a success even on rejection.

Either:
- Implement the retry logic **inside the shared library**, or
- Call raw `sh` directly in the pipeline stage for the push step

---

## Final Best Practice for CI Pipelines

Recommended pipeline Git workflow:

```bash
git add values.yaml
git commit -m "update image tag"
git pull --rebase origin main
git push origin main
```

With retry logic for safety. This ensures:

- Clean commit history
- Minimal conflicts
- Safe handling of concurrent pipeline pushes
- Reliable GitOps deployments
