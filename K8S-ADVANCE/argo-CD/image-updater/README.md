# ArgoCD Image Updater Demo — Node.js + Docker Hub + Git Write-Back

A hands-on GitOps demo that automatically detects new Docker image tags on Docker Hub and commits the updated tag back to your Git repo — triggering ArgoCD to redeploy. Zero manual manifest edits after the initial push.

**Stack:** Node.js · Docker Hub · KinD · ArgoCD · ArgoCD Image Updater v1.x · Kustomize

---

## How It Works

```
You push a new image tag to Docker Hub
        ↓
ArgoCD Image Updater polls Docker Hub (every 2 min)
        ↓
Detects 1.0.1 > 1.0.0  →  commits new tag to kustomization.yaml in Git
        ↓
ArgoCD detects the new commit  →  auto-syncs
        ↓
Kubernetes rolls out the new pod
        ↓
You did nothing after the docker push. That's the point.
```

---

## Repo Structure You Will Create

```
.
├── argo-node-app.yaml        # ArgoCD Application manifest
├── imageupdater-cr.yaml      # ArgoCD Image Updater CR (v1.x CRD-based)
└── node-app/
    ├── app.js                # Simple Node.js HTTP server
    ├── Dockerfile            # node:20-alpine, exposes port 3000
    ├── package.json
    └── k8s/
        ├── deployment.yaml   # Node app Deployment
        ├── svc.yaml          # ClusterIP Service (port 80 → 3000)
        └── kustomization.yaml
```

---

## Prerequisites

Install these before starting:

| Tool | Install |
|---|---|
| Docker | https://docs.docker.com/get-docker |
| KinD | https://kind.sigs.k8s.io/docs/user/quick-start/#installation |
| kubectl | https://kubernetes.io/docs/tasks/tools |
| ArgoCD CLI | https://argo-cd.readthedocs.io/en/stable/cli_installation |
| Git | https://git-scm.com |

You also need:
- A **Docker Hub** account and a **public** repository named `node-app`
- A **GitHub** repository (public or private) to store your manifests
- A **GitHub Personal Access Token** with `Contents: Read & Write` permission on that repo

---

## Placeholders Used In This Guide

Replace these everywhere before running any command:

| Placeholder | Replace with |
|---|---|
| `YOUR_DOCKERHUB_USERNAME` | Your Docker Hub username |
| `YOUR_GITHUB_USERNAME` | Your GitHub username |
| `YOUR_GITHUB_REPO` | Your GitHub repo name (e.g. `argocd-image-updater-demo`) |
| `YOUR_GITHUB_PAT` | Your GitHub Personal Access Token |
| `YOUR_BRANCH` | Your default branch (`main` or `master`) |

---

## Step 1 — Create the GitHub Repo

Create a new GitHub repository — e.g. `argocd-image-updater-demo`. Clone it locally:

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/YOUR_GITHUB_REPO
cd YOUR_GITHUB_REPO
```

---

## Step 2 — Create the Node.js App

```bash
mkdir -p node-app/k8s
```

### `node-app/app.js`

```js
const http = require('http');
const PORT = process.env.PORT || 3000;

http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from Node App v1.0.0\n');
}).listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### `node-app/package.json`

```json
{
  "name": "node-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
```

### `node-app/Dockerfile`

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json .
COPY app.js .
EXPOSE 3000
CMD ["node", "app.js"]
```

---

## Step 3 — Create the Kubernetes Manifests

### `node-app/k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
      - name: node-app
        image: docker.io/YOUR_DOCKERHUB_USERNAME/node-app:1.0.0
        ports:
        - containerPort: 3000
```

### `node-app/k8s/svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-app
  namespace: default
spec:
  selector:
    app: node-app
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

### `node-app/k8s/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - svc.yaml

images:
  - name: docker.io/YOUR_DOCKERHUB_USERNAME/node-app
    newTag: 1.0.0
```

> ⚠️ Do not manually edit `newTag` once Image Updater is running — it owns this field and will commit a new value here on every release.

---

## Step 4 — Create the ArgoCD Manifests

These two files live at the repo root.

### `argo-node-app.yaml`

The ArgoCD Application that watches your `k8s/` folder. No image-updater annotations — those belonged to v0.x. v1.x uses the separate `ImageUpdater` CR below.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: node-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB_USERNAME/YOUR_GITHUB_REPO
    targetRevision: YOUR_BRANCH
    path: node-app/k8s/
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### `imageupdater-cr.yaml`

The `ImageUpdater` custom resource. This is the v1.x replacement for the old annotation-based config. It tells Image Updater what to watch, how to pick a new tag, and how to write it back to Git.

```yaml
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: node-app-updater
  namespace: argocd
spec:

  applicationRefs:
    - namePattern: "node-app"
      images:
        - alias: node-app
          imageName: docker.io/YOUR_DOCKERHUB_USERNAME/node-app

          commonUpdateSettings:
            updateStrategy: semver
            allowTags: "regexp:^[0-9]+\\.[0-9]+\\.[0-9]+$"

          manifestTargets:
            kustomize:
              name: docker.io/YOUR_DOCKERHUB_USERNAME/node-app

  writeBackConfig:
    method: git
    gitConfig:
      repository: "https://github.com/YOUR_GITHUB_USERNAME/YOUR_GITHUB_REPO"
      writeBackTarget: kustomization
      branch: YOUR_BRANCH
```

**Key fields explained:**

| Field | Value | What it does |
|---|---|---|
| `namePattern` | `node-app` | ArgoCD Application to watch |
| `imageName` | `YOUR_DOCKERHUB_USERNAME/node-app` | Docker Hub image to poll |
| `updateStrategy` | `semver` | Pick the highest valid `x.y.z` tag |
| `allowTags` | `regexp:^[0-9]+\.[0-9]+\.[0-9]+$` | Ignore tags like `latest`, `dev`, `edge` |
| `writeBackTarget` | `kustomization` | Edit `kustomization.yaml` in Git |
| `method` | `git` | Commit the new tag back to your repo |

---

## Step 5 — Push Everything to GitHub

```bash
git add .
git commit -m "feat: initial node-app with argocd image updater setup"
git push origin YOUR_BRANCH
```

---

## Step 6 — Build and Push the Initial Docker Image

```bash
cd node-app/

docker build -t YOUR_DOCKERHUB_USERNAME/node-app:1.0.0 .
docker push YOUR_DOCKERHUB_USERNAME/node-app:1.0.0
```

> Make sure the `node-app` repo on Docker Hub is set to **Public**.
> Docker Hub → Repositories → node-app → Settings → Visibility → Public

---

## Step 7 — Create the KinD Cluster

```bash
kind create cluster --name argocd-demo

# Verify
kubectl cluster-info --context kind-argocd-demo
kubectl get nodes
```

---

## Step 8 — Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=120s
```

**Access ArgoCD:**

```bash
# Port-forward (run in background)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Login via CLI
argocd login localhost:8080 \
  --username admin \
  --password <PASTE_PASSWORD_HERE> \
  --insecure
```

Open `https://localhost:8080` in your browser (username: `admin`, accept the self-signed cert warning).

---

## Step 9 — Install ArgoCD Image Updater

> ⚠️ Use `config/install.yaml` — the old `manifests/install.yaml` URL is deprecated in v1.x and will install the wrong version.

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml

kubectl wait --for=condition=Ready pods \
  -l app.kubernetes.io/name=argocd-image-updater \
  -n argocd --timeout=90s

# Confirm it's running
kubectl get pods -n argocd | grep image-updater
```

---

## Step 10 — Register Your GitHub Repo with ArgoCD

Image Updater reuses ArgoCD's repo credentials for git write-back — register once, it works for both reading manifests and committing tag updates.

```bash
argocd repo add https://github.com/YOUR_GITHUB_USERNAME/YOUR_GITHUB_REPO \
  --username YOUR_GITHUB_USERNAME \
  --password YOUR_GITHUB_PAT

# Verify
argocd repo list
```

> **GitHub PAT:** GitHub → Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens → grant `Contents: Read & Write` on your repo.

---

## Step 11 — Apply the ArgoCD Application

```bash
# From repo root
kubectl apply -f argo-node-app.yaml

# Trigger first sync manually
argocd app sync node-app

# Check status
argocd app get node-app
```

ArgoCD detects `node-app/k8s/` as a Kustomize app (because `kustomization.yaml` is present) and deploys `YOUR_DOCKERHUB_USERNAME/node-app:1.0.0`.

---

## Step 12 — Apply the ImageUpdater CR

```bash
kubectl apply -f imageupdater-cr.yaml

# Verify the CR was created
kubectl get imageupdater -n argocd

# Check Image Updater logs
kubectl logs -n argocd \
  -l app.kubernetes.io/name=argocd-image-updater \
  --tail=30
```

You should see `Processing ImageUpdater CR: node-app-updater` in the logs. If you see `No ImageUpdater CRs to process`, the CR was not applied — re-run the apply command above.

---

## Step 13 — Trigger a Deployment Update

Bump the version string in `node-app/app.js`:

```js
res.end('Hello from Node App v1.0.1\n');
```

Build and push the new tag:

```bash
docker build -t YOUR_DOCKERHUB_USERNAME/node-app:1.0.1 .
docker push YOUR_DOCKERHUB_USERNAME/node-app:1.0.1
```

That's the only manual step. Everything after this is automated.

---

## Step 14 — Watch the Full Automation Loop

**Terminal 1 — Image Updater logs:**

```bash
kubectl logs -n argocd \
  -l app.kubernetes.io/name=argocd-image-updater \
  -f --tail=30
```

Expected output:
```
level=info msg="Processing ImageUpdater CR: node-app-updater"
level=info msg="Found new image: YOUR_DOCKERHUB_USERNAME/node-app:1.0.1"
level=info msg="Committing changes to kustomization.yaml"
level=info msg="Successfully pushed to branch YOUR_BRANCH"
```

**Terminal 2 — Watch ArgoCD sync:**

```bash
watch argocd app get node-app
```

**Verify the running pod uses the new image:**

```bash
kubectl describe deployment node-app | grep Image
# Image:  YOUR_DOCKERHUB_USERNAME/node-app:1.0.1
```

**Check the automated Git commit on GitHub:**

```
https://github.com/YOUR_GITHUB_USERNAME/YOUR_GITHUB_REPO/commits/YOUR_BRANCH
```

You will see a commit authored by the Image Updater bot:
```
build: automatic update of node-app to YOUR_DOCKERHUB_USERNAME/node-app:1.0.1
```

---

## Troubleshooting

**`No ImageUpdater CRs to process` in logs**

You are on v1.x but the `ImageUpdater` CR has not been applied. v1.x does not read annotations from the ArgoCD Application — it needs its own CR:
```bash
kubectl apply -f imageupdater-cr.yaml
```

**Installed Image Updater with the wrong URL**

The `manifests/install.yaml` URL is deprecated. Reinstall using the correct one:
```bash
kubectl delete -n argocd \
  -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml
```

**Image Updater not detecting new tags**

- Confirm Docker Hub repo visibility is **Public**
- Confirm the new tag matches `x.y.z` format — the `allowTags` regexp filters everything else out
- Image Updater polls every 2 minutes by default — wait and re-check logs

**Git write-back failing — no commit appearing on GitHub**

- Confirm the repo was registered via `argocd repo add` with a PAT that has `Contents: Write`
- Image Updater reuses ArgoCD's repo credentials — no separate secret needed in the CR
- Check logs for `permission denied` or `authentication failed`

**ArgoCD app stuck in `OutOfSync`**

```bash
argocd app sync node-app --force
```

---

## Cleanup

```bash
kind delete cluster --name argocd-demo
```

---

## Version Reference

| Component | Version |
|---|---|
| ArgoCD Image Updater | v1.1.x |
| Node.js base image | `node:20-alpine` |
| Update strategy | `semver` |
| Write-back method | `git` → `kustomization` |
| Cluster | KinD |
