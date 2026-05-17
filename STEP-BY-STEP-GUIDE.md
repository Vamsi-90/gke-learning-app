# GKE Learning Project — Step by Step Guide
# Complete Guide: From Zero to Deploying on Google Kubernetes Engine

---

## What We Built

A Python web app that:
- Runs on Google Kubernetes Engine (GKE)
- Auto-deploys when you push code to GitHub
- Stays running 24/7 — even if pods crash
- Can rollback in seconds if something breaks

**Live App URL:** http://136.116.116.254
**ArgoCD Dashboard:** https://34.57.95.151

---

## The Big Picture — Full Flow

```
You write Python code
        ↓
Push to GitHub
        ↓
GitHub Actions triggers automatically
  → Builds Docker image (packages your app)
  → Pushes image to Artifact Registry (GCP storage)
        ↓
ArgoCD (running inside GKE) detects change
  → Reads k8s/ folder from GitHub
  → Deploys pods to GKE
        ↓
Your app is live on the internet!
  → 2 pods always running
  → If one crashes → Kubernetes restarts it automatically
```

---

## Tools Used

| Tool | What it does |
|------|-------------|
| **Python + Flask** | The web app code |
| **Docker** | Packages your app into an image |
| **GitHub** | Stores your code |
| **GitHub Actions** | Auto-builds Docker image when you push |
| **Artifact Registry** | GCP storage for Docker images |
| **GKE** | Google's Kubernetes — runs your containers |
| **ArgoCD** | Watches GitHub, auto-deploys to GKE |
| **kubectl** | Command line tool to talk to Kubernetes |

---

## Kubernetes Concepts — Simple Explanation

```
GKE CLUSTER  (like a warehouse building)
├── NODE 1   (a virtual machine — like a shelf)
│   ├── POD  (your app running — like a box)
│   └── POD  (another copy of your app)
└── NODE 2   (another virtual machine)
    └── POD  (another copy of your app)
```

| Word | Simple meaning |
|------|---------------|
| **Cluster** | The whole Kubernetes system |
| **Node** | A virtual machine inside the cluster |
| **Pod** | One running copy of your app |
| **Container** | Your app packaged with Docker |
| **Namespace** | A folder inside the cluster |
| **Deployment** | Instructions for how many pods to run |
| **Service** | Gives your app a public IP address |
| **ReplicaSet** | A snapshot of each deployment version |

---

## GCP Resources We Created

| Resource | Name | Purpose |
|----------|------|---------|
| Project | gke-learning-2025 | Container for all resources |
| Cluster | gke-learning-cluster | Kubernetes cluster (zone: us-central1-a) |
| Nodes | 2 x e2-medium | Virtual machines (2 CPU, 4GB RAM each) |
| Artifact Registry | gke-repo | Stores Docker images |
| Service Account | github-actions-sa | Robot user for GitHub Actions |
| Budget Alert | GKE Learning - $250 Alert | Email alert at $200 and $250 spent |

---

# STEP BY STEP — EXACTLY WHAT WE DID

---

## PHASE 1 — Setup GCP

### Step 1 — Login to GCP
```bash
gcloud auth login
# Opens browser → sign in with your Google account
```

### Step 2 — Create a new GCP Project
```bash
gcloud projects create gke-learning-2025 --name="GKE Learning"
gcloud config set project gke-learning-2025
```
**Why:** Everything in GCP lives inside a project. Like a folder.

### Step 3 — Link Billing
```bash
gcloud billing projects link gke-learning-2025 --billing-account=YOUR_BILLING_ACCOUNT_ID
```
**Why:** Without billing, GCP won't let you create resources.

To find your billing account ID:
- Go to console.cloud.google.com/billing
- Copy the Account ID (format: XXXXXX-XXXXXX-XXXXXX)

### Step 4 — Enable Required APIs
```bash
gcloud services enable container.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com
```
**Why:** GCP services are OFF by default. You turn on only what you need.
- `container.googleapis.com` → GKE (Kubernetes)
- `artifactregistry.googleapis.com` → Docker image storage
- `cloudbuild.googleapis.com` → Building images

### Step 5 — Create GKE Cluster
```bash
gcloud container clusters create gke-learning-cluster \
  --zone=us-central1-a \
  --num-nodes=2 \
  --machine-type=e2-medium \
  --disk-size=20
```
**Why:** This creates 2 virtual machines on Google's servers where your app will run.
**Takes:** 3-5 minutes

### Step 6 — Connect kubectl to your cluster
```bash
gcloud components install gke-gcloud-auth-plugin
gcloud container clusters get-credentials gke-learning-cluster --zone=us-central1-a --project=gke-learning-2025
```
**Why:** kubectl needs the cluster's address and password to talk to it. This saves it to ~/.kube/config

### Step 7 — Verify cluster is working
```bash
kubectl get nodes
# Should show 2 nodes with STATUS: Ready
```

### Step 8 — Set up Billing Alert
```bash
gcloud services enable billingbudgets.googleapis.com --project=gke-learning-2025
gcloud billing budgets create \
  --billing-account=YOUR_BILLING_ACCOUNT_ID \
  --display-name="GKE Learning - $250 Alert" \
  --budget-amount=250USD \
  --threshold-rule=percent=0.8 \
  --threshold-rule=percent=1.0
```
**Why:** Safety net — sends email when you've spent $200 and $250.

---

## PHASE 2 — Create the Python App

### Step 9 — Create GitHub repo
1. Go to github.com → click "+" → "New repository"
2. Name: `gke-learning-app`
3. Public, add README
4. Click "Create repository"

### Step 10 — Clone repo to your laptop
```bash
mkdir ~/gke-project
cd ~/gke-project
git clone https://github.com/YOUR_GITHUB_USERNAME/gke-learning-app.git
cd gke-learning-app
```

### Step 11 — Create app.py (the Python web app)
```python
# app.py
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from GKE! My app is running 24/7"

@app.route("/health")
def health():
    return "OK", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```
**Why:**
- `/` route → shows your message when someone visits the URL
- `/health` route → Kubernetes pings this to check if app is alive
- `port=8080` → app listens on this port
- `host="0.0.0.0"` → accept connections from anywhere

### Step 12 — Create requirements.txt
```
flask==3.0.0
```
**Why:** Lists Python packages your app needs. Docker reads this and installs them.

### Step 13 — Create Dockerfile
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 8080
CMD ["python", "app.py"]
```
**Why — line by line:**
- `FROM python:3.11-slim` → start with Python already installed
- `WORKDIR /app` → do everything inside /app folder
- `COPY requirements.txt .` → copy requirements into container
- `RUN pip install` → install Flask inside the container
- `COPY app.py .` → copy your app into container
- `EXPOSE 8080` → tell Docker your app uses port 8080
- `CMD` → when container starts, run your app

---

## PHASE 3 — Kubernetes YAML Files

### Step 14 — Create k8s folder
```bash
mkdir k8s
```
**Why:** Keep all Kubernetes config files organized in one folder.

### Step 15 — Create k8s/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gke-learning-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gke-learning-app
  template:
    metadata:
      labels:
        app: gke-learning-app
    spec:
      containers:
      - name: gke-learning-app
        image: us-central1-docker.pkg.dev/PROJECT_ID/gke-repo/gke-learning-app:latest
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
```
**Why — key lines:**
- `replicas: 2` → always keep 2 pods running
- `image` → which Docker image to run
- `readinessProbe` → only send traffic when app is ready
- `livenessProbe` → restart pod if health check fails

### Step 16 — Create k8s/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: gke-learning-app
spec:
  type: LoadBalancer
  selector:
    app: gke-learning-app
  ports:
  - port: 80
    targetPort: 8080
```
**Why — key lines:**
- `type: LoadBalancer` → creates a public IP address
- `port: 80` → internet traffic comes in on port 80
- `targetPort: 8080` → forwards to port 8080 inside pod (where Flask runs)

---

## PHASE 4 — GitHub Actions (Auto Build Pipeline)

### Step 17 — Create Artifact Registry repository
```bash
gcloud artifacts repositories create gke-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="GKE Learning App Images"
```
**Why:** Creates a folder in GCP to store your Docker images.

### Step 18 — Create Service Account for GitHub Actions
```bash
# Create the service account (robot user)
gcloud iam service-accounts create github-actions-sa \
  --display-name="GitHub Actions Service Account"

# Give it permission to push Docker images
gcloud projects add-iam-policy-binding gke-learning-2025 \
  --member="serviceAccount:github-actions-sa@gke-learning-2025.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# Create a key file (password)
gcloud iam service-accounts keys create ~/gke-sa-key.json \
  --iam-account=github-actions-sa@gke-learning-2025.iam.gserviceaccount.com
```
**Why:** GitHub Actions needs permission to push to GCP. Service account = a robot user with limited permissions.

### Step 19 — Add GCP key as GitHub Secret
1. Copy contents of `~/gke-sa-key.json`
2. Go to github.com/YOUR_USERNAME/gke-learning-app
3. Settings → Secrets and variables → Actions
4. New repository secret
5. Name: `AKHILA_GCP_SA_KEY`
6. Paste the JSON content
7. Click Add secret

**Why:** Secrets are encrypted. Your workflow reads them without exposing the actual value.

### Step 20 — Create .github/workflows/build.yaml
```bash
mkdir -p .github/workflows
```

```yaml
# .github/workflows/build.yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v2
      with:
        credentials_json: ${{ secrets.AKHILA_GCP_SA_KEY }}

    - name: Configure Docker for Artifact Registry
      run: gcloud auth configure-docker us-central1-docker.pkg.dev

    - name: Build Docker image
      run: |
        docker build -t us-central1-docker.pkg.dev/gke-learning-2025/gke-repo/gke-learning-app:${{ github.sha }} .
        docker tag ...gke-learning-app:${{ github.sha }} ...gke-learning-app:latest

    - name: Push Docker image
      run: |
        docker push ...gke-learning-app:${{ github.sha }}
        docker push ...gke-learning-app:latest
```
**Why — key parts:**
- `on: push: branches: main` → run when code is pushed to main
- `runs-on: ubuntu-latest` → use a Linux machine
- `github.sha` → unique ID for each commit, used as image tag
- Steps run top to bottom automatically

### Step 21 — Push everything to GitHub
```bash
git add .
git commit -m "Add Flask app, Dockerfile, k8s manifests and GitHub Actions"
git push origin main
```

Go to GitHub → Actions tab → watch the workflow run automatically!

---

## PHASE 5 — Install and Setup ArgoCD

### Step 22 — What is ArgoCD?
ArgoCD runs inside your cluster and:
- Watches your GitHub repo (specifically the k8s/ folder)
- When k8s/ changes → automatically deploys to GKE
- Keeps cluster always matching GitHub (GitOps)

```
GitHub k8s/ folder = source of truth
ArgoCD = makes sure GKE always matches it
```

### Step 23 — Install ArgoCD
```bash
# Create a namespace (folder) for ArgoCD
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Check ArgoCD pods are running
kubectl get pods -n argocd
```

### Step 24 — Expose ArgoCD UI
```bash
# Give ArgoCD a public IP
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get the IP (wait 1-2 minutes for EXTERNAL-IP to appear)
kubectl get svc argocd-server -n argocd
```

### Step 25 — Login to ArgoCD
1. Open https://YOUR_ARGOCD_IP in browser
2. Click Advanced → Proceed anyway (SSL warning is normal)
3. Username: `admin`
4. Get password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Step 26 — Create ArgoCD Application
Create file `argocd-app.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gke-learning-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gke-learning-app
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f argocd-app.yaml
```

**Why — key parts:**
- `repoURL` → your GitHub repo to watch
- `path: k8s` → watch only the k8s/ folder
- `automated` → auto sync, no manual clicks
- `selfHeal: true` → if someone changes something manually, ArgoCD fixes it back

### Step 27 — Verify deployment
```bash
# Check ArgoCD app status
kubectl get application -n argocd

# Check your pods are running
kubectl get pods

# Get your app's public IP
kubectl get svc gke-learning-app
```

Open `http://YOUR_EXTERNAL_IP` in browser — your app is live!

---

## PHASE 6 — Testing 24/7 Uptime

### How to deploy new code
```bash
# 1. Edit app.py — change the message
# 2. Push to GitHub
git add .
git commit -m "Update message"
git push origin main

# 3. GitHub Actions builds new image automatically
# 4. Run this to pick up the new image (until we set up auto-tagging)
kubectl rollout restart deployment/gke-learning-app
```

### Test that app survives pod crashes
```bash
# Delete a pod — watch Kubernetes recreate it
kubectl delete pod POD_NAME

# Watch pods in real time
kubectl get pods -w
```

### Test failure protection (intentional break)
```bash
# 1. Break your code on purpose (add syntax error)
# 2. Push to GitHub and restart
kubectl rollout restart deployment/gke-learning-app

# 3. Watch — new pod crashes, OLD pods keep running!
kubectl get pods -w
# You'll see: CrashLoopBackOff for new pod
# But old pods stay Running — website never goes down!
```

### Rollback to previous version
```bash
kubectl rollout undo deployment/gke-learning-app
# Instantly goes back to previous working version
```

### View rollout history
```bash
kubectl rollout history deployment/gke-learning-app
```

---

## Important Commands Cheat Sheet

### Cluster
```bash
# Connect to cluster
gcloud container clusters get-credentials gke-learning-cluster --zone=us-central1-a --project=gke-learning-2025

# See nodes (VMs)
kubectl get nodes

# See all namespaces
kubectl get namespaces
```

### Pods
```bash
# See running pods
kubectl get pods

# Watch pods in real time
kubectl get pods -w

# See pod logs
kubectl logs POD_NAME

# Delete a pod (Kubernetes recreates it)
kubectl delete pod POD_NAME
```

### Deployments
```bash
# See deployments
kubectl get deployments

# Restart all pods (picks up new Docker image)
kubectl rollout restart deployment/gke-learning-app

# Rollback to previous version
kubectl rollout undo deployment/gke-learning-app

# View deployment history
kubectl rollout history deployment/gke-learning-app
```

### Services
```bash
# See services and public IPs
kubectl get svc
```

### ArgoCD
```bash
# Check sync status
kubectl get application -n argocd

# Force ArgoCD to re-check GitHub
kubectl annotate application gke-learning-app -n argocd argocd.argoproj.io/refresh=hard --overwrite

# See ArgoCD pods
kubectl get pods -n argocd
```

---

## ArgoCD Dashboard — What Each Thing Means

| What you see | What it means |
|-------------|--------------|
| `APP HEALTH: Healthy` | Your app is running fine |
| `SYNC STATUS: Synced` | GKE matches what's in GitHub |
| `Synced to HEAD: abc123` | Which GitHub commit is deployed |
| `ReplicaSet Rev:1` | First deployment version |
| `ReplicaSet Rev:2` | Second deployment version |
| `CrashLoopBackOff` | Pod keeps crashing and restarting |
| `Running 1/1` | Pod is healthy and serving traffic |
| `Progressing` | New deployment in progress |

---

## What Keeps Your App Running 24/7

1. **replicas: 2** → always 2 pods running
2. **livenessProbe** → Kubernetes checks /health every few seconds, restarts pod if it fails
3. **readinessProbe** → new pod only gets traffic after /health returns OK
4. **2 Nodes** → if one VM goes down, pods move to the other
5. **Rolling updates** → new pods start before old ones stop

---

## Cost Breakdown

| Resource | Cost/month |
|----------|-----------|
| 2 x e2-medium nodes | ~$50-70 |
| Load Balancer | ~$18 |
| Artifact Registry | ~$0.10/GB |
| **Total** | **~$70-90/month** |

Budget alert set at:
- $200 → warning email
- $250 → final warning email

Your $300 GCP credit covers ~3-4 months of this setup.

---

## Project File Structure

```
gke-learning-app/
├── app.py                         # Python Flask web app
├── requirements.txt               # Python packages (flask)
├── Dockerfile                     # Recipe to build Docker image
├── argocd-app.yaml                # ArgoCD application config
├── LEARN.md                       # Concepts explained
├── STEP-BY-STEP-GUIDE.md          # This file
├── k8s/
│   ├── deployment.yaml            # Run 2 pods of this app
│   └── service.yaml               # Give app a public IP
└── .github/
    └── workflows/
        └── build.yaml             # GitHub Actions pipeline
```
