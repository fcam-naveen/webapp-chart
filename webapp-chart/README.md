# webapp-chart

This Helm chart deploys a web application + PostgreSQL database onto Kubernetes, managed by ArgoCD.

**What ArgoCD does:** It watches your Git repo. Whenever you push a change, it automatically applies that change to your cluster — no manual `kubectl apply` needed.

---

## How It Works

```
You push a change to Git
        |
      ArgoCD notices (within ~3 min)
        |
   Applies it to Kubernetes automatically
        |
   Web app is live at http://<NODE-IP>:30000
```

---

## Prerequisites

- A running Kubernetes cluster (e.g., Vagrant + kubeadm)
- `kubectl` working against your cluster
- `helm` v3 installed

---

## Step 1 — Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for ArgoCD to finish starting up:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=180s
```

Check that all pods show `Running`:

```bash
kubectl get pods -n argocd
```

---

## Step 2 — Expose the ArgoCD UI

By default ArgoCD is not accessible from outside the cluster. Apply this NodePort service to open it on port **30080**:

```bash
kubectl apply -f templates/svc_argocd.yaml
```

---

## Step 3 — Open the ArgoCD UI

First, get your node's IP address:

```bash
kubectl get nodes -o wide
# Look at the INTERNAL-IP column
```

Then open this in your browser:

```
http://<NODE-IP>:30080
```

**Login credentials:**

- Username: `admin`
- Password: run this to get it:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

---

## Step 4 — Deploy the App via ArgoCD

This tells ArgoCD which Git repo to watch and where to deploy it.

Run this from the root of the `k8s-deployments` folder:

```bash
kubectl apply -f argo_webapp.yaml
```

ArgoCD will now pull the Helm chart from GitHub and deploy the web app + PostgreSQL to your cluster.

> If your repo is **private**, go to **Settings > Repositories** in the ArgoCD UI and add your GitHub credentials before this step.

Check the deployment status:

```bash
kubectl get application -n argocd webapp
```

You should see `Synced` and `Healthy`. You can also see it in the ArgoCD UI — a box labelled `webapp` will appear.

---

## Step 5 — Access the Web App

```
http://<NODE-IP>:30000
```

Same node IP as above, but port **30000**.

Confirm the pods are running:

```bash
kubectl get pods
```

---

## Making a Change and Seeing It Apply

This is the core of GitOps — you edit a file, push it, and ArgoCD handles the rest.

### Example: Scale the web app from 2 to 3 replicas

1. Open `values.yaml` and change:

```yaml
webapp:
  replicas: 3   # was 2
```

2. Push the change:

```bash
git add values.yaml
git commit -m "Scale webapp to 3 replicas"
git push origin main
```

3. ArgoCD will detect the change within ~3 minutes and sync automatically. Watch it:

```bash
kubectl get application -n argocd webapp -w
```

4. Confirm it worked:

```bash
kubectl get deployment web-app
# Should show 3/3 READY
```

### Example: Update the app image to a new version

1. Edit `values.yaml`:

```yaml
webapp:
  tag: 3.4   # was 3.3
```

2. Push:

```bash
git add values.yaml
git commit -m "Bump webapp to v3.4"
git push origin main
```

3. Watch the rollout:

```bash
kubectl rollout status deployment/web-app
```

### Need to sync immediately (don't want to wait 3 min)?

Click **Sync** in the ArgoCD UI on the `webapp` app — it applies instantly.

---

## Port Reference

| What | Port | URL |
|---|---|---|
| ArgoCD UI | 30080 | `http://<NODE-IP>:30080` |
| Web App | 30000 | `http://<NODE-IP>:30000` |

---

## Key Files

| File | What it does |
|---|---|
| `values.yaml` | The main config — change image tags, replicas, passwords here |
| `templates/deploy_webapp.yaml` | Web app Deployment |
| `templates/deploy_postgress.yaml` | PostgreSQL StatefulSet (with persistent storage) |
| `templates/svc_webapp.yaml` | Exposes web app on NodePort 30000 |
| `templates/svc_argocd.yaml` | Exposes ArgoCD UI on NodePort 30080 |
| `templates/secret_postgres.yaml` | Creates the Postgres password Secret |
| `argo_webapp.yaml` | Tells ArgoCD which repo to watch (one-time setup) |

---

## Troubleshooting

**App stuck as `OutOfSync` in ArgoCD:**
```bash
kubectl describe application webapp -n argocd
```

**Web app pods not starting:**
```bash
kubectl logs -l app=web-app
kubectl describe pod -l app=web-app
```

**Forgot the ArgoCD password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```
