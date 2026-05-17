# GKE Learning Project — Complete Documentation

## What We Built

A Python web app deployed on Google Kubernetes Engine (GKE) with a full GitOps pipeline.

**Live URL:** http://136.116.116.254

---

## The Full Flow

```
You write code
      ↓
Push to GitHub (github.com/Vamsi-90/gke-learning-app)
      ↓
GitHub Actions triggers automatically
  → Builds Docker image
  → Pushes image to Artifact Registry (GCP)
      ↓
ArgoCD (running inside GKE) detects changes
  → Reads k8s/ folder from GitHub
  → Deploys pods to GKE cluster
      ↓
Your app runs 24/7 at http://136.116.116.254
  → If a pod crashes → Kubernetes auto-restarts it
  → Always 2 pods running
```

---

## Tools Used and What Each Does

| Tool | What it does |
|------|-------------|
| **Python + Flask** | The actual web app |
| **Docker** | Packages the app into an image |
| **GitHub** | Stores your code |
| **GitHub Actions** | Auto-builds Docker image when you push code |
| **Artifact Registry** | GCP storage for Docker images |
| **GKE** | Google's Kubernetes — runs your containers |
| **ArgoCD** | Watches GitHub and keeps GKE in sync |
| **kubectl** | Command line tool to talk to Kubernetes |

---

## Kubernetes Concepts

### Cluster Structure
```
GKE CLUSTER (gke-learning-cluster)
├── NODE 1 (e2-medium VM in Google's datacenter)
│   ├── Pod 1 — your app running
│   └── Pod 2 — your app running (backup)
└── NODE 2 (e2-medium VM in Google's datacenter)
```

| Term | Simple meaning |
|------|---------------|
| **Cluster** | The whole Kubernetes system (like a warehouse) |
| **Node** | A virtual machine inside the cluster (like a shelf) |
| **Pod** | One running copy of your app (like a box on the shelf) |
| **Container** | Your app packaged with Docker (the product inside the box) |
| **Namespace** | A folder inside the cluster to organize things |
| **Deployment** | Tells Kubernetes how many pods to run and what image to use |
| **Service** | Gives your app a public IP address |
| **LoadBalancer** | Splits traffic between your pods |

---

## GCP Resources Created

| Resource | Name | Details |
|----------|------|---------|
| Project | gke-learning-2025 | All resources live here |
| Cluster | gke-learning-cluster | Zone: us-central1-a |
| Nodes | 2 x e2-medium | 2 CPU, 4GB RAM each |
| Artifact Registry | gke-repo | Stores Docker images |
| Service Account | github-actions-sa | Used by GitHub Actions to push images |
| Budget Alert | GKE Learning - $250 Alert | Email alert at $200 and $250 |

---

## Files in This Project

```
gke-learning-app/
├── app.py                          # Python Flask web app
├── requirements.txt                # Python packages needed (flask)
├── Dockerfile                      # Recipe to build Docker image
├── argocd-app.yaml                 # ArgoCD application config
├── k8s/
│   ├── deployment.yaml             # Tells GKE: run 2 pods of this app
│   └── service.yaml                # Gives app a public IP (LoadBalancer)
└── .github/
    └── workflows/
        └── build.yaml              # GitHub Actions pipeline
```

---

## File Explanations

### app.py
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")           # When someone visits the URL, show this
def home():
    return "Hello from GKE! My app is running 24/7"

@app.route("/health")     # Kubernetes checks this to know app is alive
def health():
    return "OK", 200

app.run(host="0.0.0.0", port=8080)  # Listen on all interfaces, port 8080
```

### Dockerfile
```dockerfile
FROM python:3.11-slim        # Start with Python already installed
WORKDIR /app                 # Work in /app folder
COPY requirements.txt .      # Copy requirements file
RUN pip install -r requirements.txt  # Install Flask
COPY app.py .                # Copy your app
EXPOSE 8080                  # App uses port 8080
CMD ["python", "app.py"]     # Start the app
```

### k8s/deployment.yaml
```yaml
replicas: 2                  # Keep 2 pods running always
image: ...gke-learning-app:latest  # Use this Docker image
readinessProbe: /health      # Only send traffic when app is ready
livenessProbe: /health       # Restart pod if health check fails
```

### k8s/service.yaml
```yaml
type: LoadBalancer           # Create a public IP
port: 80                     # Internet traffic comes in on port 80
targetPort: 8080             # Forward to port 8080 inside the pod
```

### .github/workflows/build.yaml
Runs automatically on every push to main branch:
1. Checkout your code
2. Login to GCP using the secret key
3. Build Docker image
4. Push image to Artifact Registry

---

## Important Commands

### Check your cluster
```bash
# Connect kubectl to your cluster
gcloud container clusters get-credentials gke-learning-cluster --zone=us-central1-a --project=gke-learning-2025

# See your nodes (VMs)
kubectl get nodes

# See your pods (running app copies)
kubectl get pods

# See your services (public IPs)
kubectl get svc
```

### Check ArgoCD
```bash
# See if app is synced
kubectl get application -n argocd

# See ArgoCD details
kubectl describe application gke-learning-app -n argocd

# Force ArgoCD to refresh
kubectl annotate application gke-learning-app -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### Test 24/7 uptime
```bash
# Delete a pod — Kubernetes will auto-create a new one
kubectl delete pod <pod-name>

# Watch pods in real time
kubectl get pods -w
```

### View app logs
```bash
kubectl logs <pod-name>
```

---

## How to Deploy New Code

1. Make changes to `app.py`
2. Push to GitHub:
```bash
git add .
git commit -m "your message"
git push origin main
```
3. GitHub Actions automatically builds new Docker image
4. ArgoCD automatically deploys to GKE
5. Your live URL updates automatically

---

## ArgoCD UI
- **URL:** https://34.57.95.151
- **Username:** admin
- **Password:** stored in cluster (run below to get it)
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## Cost
- Billing account: 0104F5-48534D-553BCB
- Budget alert set at $250 (emails sent at $200 and $250)
- Estimated cost: ~$70-90/month for this setup
- GCP account: akhilaakhikatepalli@gmail.com

---

## What Keeps Your App Running 24/7

1. **replicas: 2** — always 2 pods running
2. **livenessProbe** — Kubernetes pings /health every few seconds. If no response, it restarts the pod
3. **Deployment** — if a pod crashes, Kubernetes creates a new one automatically
4. **2 Nodes** — if one VM goes down, pods move to the other node
