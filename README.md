#  Argo CD Demo – GitOps on Kubernetes (AWS EC2)

A hands-on demo of **GitOps-based continuous delivery** using **Argo CD**. This repository holds the Kubernetes manifests for a sample application deployed in the `vamsi` namespace on a Kubernetes cluster running on AWS EC2. Argo CD watches this repo and keeps the cluster in sync with whatever is committed here.

---

##  About

In a GitOps workflow, Git is the **single source of truth**. Instead of running `kubectl apply` manually, you commit changes to this repo and Argo CD automatically detects the drift and syncs the cluster to match.

This demo covers the most common Kubernetes building blocks:

| Resource | Name | Purpose |
|----------|------|---------|
| Namespace | `vamsi` | Isolates all demo resources |
| Deployment | `my-deployment` | Runs 3 replicas of the `foodapp` container (port 80) |
| DaemonSet | `my-daemonset` | Runs one `foodapp` pod on every worker node (2 nodes) |
| Service (LoadBalancer) | `mysvc` | Exposes the app publicly via an AWS ELB (port 80 → 80) |
| ConfigMap | `my-configmap` | Stores non-sensitive configuration (6 key/value pairs + `app.config.json`) |
| Secret | `my-secret` | Base64-encoded secret values (Opaque, 4 keys) |
| Secret | `myapp-secret-plain` | Secret created with `stringData` (Opaque, 3 keys) |

**Container image:** `vamsikrishnakakani/vamsi-repo:foodapp` (Docker Hub)

> The Deployment and DaemonSet both use the label `app: foodapp`, so the `mysvc` Service load-balances traffic across all of their pods.

---

##  Architecture

```
GitHub Repo (manifests)
        │
        ▼
     Argo CD  ──── watches & syncs ────►  Kubernetes Cluster (AWS EC2)
                                                │
                                          Namespace: vamsi
                                                │
              ┌──────────────┬──────────────────┼───────────────┬──────────────┐
              ▼              ▼                  ▼               ▼              ▼
        Deployment      DaemonSet           Service        ConfigMap       Secrets
       (3 replicas)   (1 pod / node)    (LoadBalancer:80)
              │                               │
              ▼                               ▼
          3 Pods                       AWS ELB (public URL)
```

---

##  Repository Structure

```
Argo-CD-demo/
├── namespace.yaml       # Namespace: vamsi
├── deployment.yaml      # my-deployment (3 replicas)
├── daemonset.yaml       # my-daemonset (one pod per node)
├── service.yaml         # mysvc (LoadBalancer, port 80)
├── configmap.yaml       # my-configmap
├── secret.yaml          # my-secret and myapp-secret-plain
└── README.md
```

> Adjust file names if yours are different.

---

##  Configuration Details

### ConfigMap: `my-configmap`

| Key | Value |
|-----|-------|
| `NODE_ENV` | `production` |
| `LOG_LEVEL` | `info` |
| `APP_PORT` | `3000` |
| `DB_HOST` | `mongo-service` |
| `DB_PORT` | `27017` |
| `DB_NAME` | `myappdb` |
| `app.config.json` | Feature flags (`newUI`, `betaFeatures`) and timeouts (`request`, `db`) |

### Secrets

| Secret | Method | Keys |
|--------|--------|------|
| `my-secret` | `data` (base64-encoded) | `DB_USER`, `DB_PASSWORD`, `JWT_SECRET`, `API_KEY` |
| `myapp-secret-plain` | `stringData` (auto-encoded by Kubernetes) | `DB_USER`, `DB_PASSWORD`, `JWT_SECRET` |

Encode a value manually with:

```bash
echo -n 'admin' | base64
```

> The values in this repo are **dummy demo values** only.

---

##  Prerequisites

- An AWS EC2 instance with a running Kubernetes cluster
- `kubectl` configured to talk to the cluster
- Argo CD installed on the cluster (see the next section)
- A GitHub account and this repository

---

##  Part 1: Install Argo CD

### 1. Create the namespace and install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify that all Argo CD pods are running:

```bash
kubectl get all -n argocd
```

### 2. Expose the Argo CD server

By default, `argocd-server` is a ClusterIP service. Change it to a LoadBalancer so you can reach the UI from your browser:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Install `jq` (used to parse JSON output):

```bash
yum install jq -y
```

Get the Argo CD load balancer URL:

```bash
export ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname')
echo $ARGOCD_SERVER
```

Or run the command directly:

```bash
kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'
```

> The output is the load balancer URL. Open `https://<ARGOCD_SERVER>` in your browser. (If you see a certificate warning, proceed anyway. Argo CD uses a self-signed certificate by default.)

### 3. Get the Argo CD admin password

```bash
export ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo $ARGO_PWD
```

Or run the command directly:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 4. Log in

| Field | Value |
|-------|-------|
| URL | `https://<ARGOCD_SERVER>` |
| Username | `admin` |
| Password | Output of the command above |

---

##  Part 2: Setup & Deployment of the Demo App

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/Argo-CD-demo.git
cd Argo-CD-demo
```

### 2. Create the namespace

```bash
kubectl create namespace vamsi
```

### 3. (Option A) Deploy manually with kubectl

```bash
kubectl apply -f . -n vamsi
```

### 4. (Option B) Deploy with Argo CD (GitOps way)

Create an Argo CD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/Argo-CD-demo.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: vamsi
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply it:

```bash
kubectl apply -f application.yaml
```

Or via the Argo CD CLI:

```bash
argocd app create argocd-demo \
  --repo https://github.com/<your-username>/Argo-CD-demo.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace vamsi \
  --sync-policy automated
```

---

##  Verify the Deployment

```bash
kubectl get all -n vamsi
kubectl get secret -n vamsi
kubectl get configmap -n vamsi
```

**Expected output (all resources):**

```
NAME                                READY   STATUS    RESTARTS   AGE
pod/my-daemonset-c9lth              1/1     Running   0          17m
pod/my-daemonset-lgzb4              1/1     Running   0          17m
pod/my-deployment-95c6bbc64-cv9rw   1/1     Running   0          21m
pod/my-deployment-95c6bbc64-jpd44   1/1     Running   0          21m
pod/my-deployment-95c6bbc64-l5jl2   1/1     Running   0          21m

NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                          PORT(S)        AGE
service/mysvc   LoadBalancer   100.64.85.244   <aws-elb>.us-east-2.elb.amazonaws.com   80:30779/TCP   20m

NAME                          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
daemonset.apps/my-daemonset   2         2         2       2            2

NAME                            READY   UP-TO-DATE   AVAILABLE
deployment.apps/my-deployment   3/3     3            3
```

**Secrets:**

```
NAME                 TYPE     DATA   AGE
my-secret            Opaque   4      2m47s
myapp-secret-plain   Opaque   3      2m47s
```

### Access the application

Open the ELB hostname from the `EXTERNAL-IP` column in your browser:

```
http://<EXTERNAL-IP>
```

---

##  GitOps Workflow in Action

1. Edit a manifest (e.g. change `replicas: 3` → `replicas: 5` in `deployment.yaml`)
2. Commit and push to GitHub
   ```bash
   git add .
   git commit -m "Scale deployment to 5 replicas"
   git push origin main
   ```
3. Argo CD detects the change and syncs automatically (watch it live in the Argo CD UI)
4. Confirm:
   ```bash
   kubectl get pods -n vamsi
   ```

Try deleting a pod or editing a resource manually with `kubectl`. With `selfHeal: true`, Argo CD reverts it back to the state defined in Git.

---

##  Security Note

> ⚠️ **Never commit real secrets to a public GitHub repository.**
> Kubernetes Secrets are only base64-encoded, not encrypted. For production, use tools such as **Sealed Secrets**, **External Secrets Operator**, **AWS Secrets Manager**, or **HashiCorp Vault**. The secrets in this repo are for **demo purposes only**.

---

##  Cleanup

```bash
kubectl delete namespace vamsi
# or, if managed by Argo CD
argocd app delete argocd-demo
```

---

##  What You'll Learn

- Core Kubernetes objects: Deployment, DaemonSet, Service, ConfigMap, Secret
- Exposing apps with an AWS LoadBalancer
- Argo CD installation and Application setup
- Automated sync, self-healing, and pruning
- GitOps principles in practice

---

##  Tech Stack

`Kubernetes` · `Argo CD` · `AWS EC2` · `AWS ELB` · `GitHub` · `kubectl`

---

## 👤 Author

**Vamsi**
Feel free to ⭐ the repo if you found it helpful!
