![GitOps](https://img.shields.io/badge/GitOps-ArgoCD-blue)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Ready-blue)

# GitOps Live Demo – Kubernetes with Argo CD

This repository contains the **live demos** used in the **TechTalk With Nathan** YouTube series.

It demonstrates how to deploy and manage Kubernetes applications using **GitOps principles**, with **Git as the single source of truth** and **Argo CD** as the GitOps controller.

All changes to the cluster are driven **only through Git**.

---

## 📌 What This Demo Shows

- GitOps in practice on Kubernetes
- Git as the **only interface** for deployments
- Continuous reconciliation with Argo CD
- Automatic updates when Git changes
- Safe rollbacks and full auditability
- A clean foundation for advanced GitOps demos

**Key takeaway:**

> **Change Git → Argo CD reconciles → Kubernetes updates automatically**

---

## GitOps in One Sentence

GitOps is a set of practices where **Git defines the desired state** of your system, and an automated controller ensures the **actual state always matches Git**.

---

## Architecture Overview

```
Git Repository (desired state)
        ↓
   Argo CD (GitOps controller)
        ↓
Kubernetes Cluster (actual state)
```

* Git contains declarative Kubernetes manifests (YAML)
* Argo CD runs inside the cluster
* Argo CD pulls from Git and reconciles continuously
* No manual `kubectl apply` is required

---

## Repository Structure

```
apps/
  hello-world/
    deployment.yaml
    service.yaml
```

* `apps/hello-world/deployment.yaml` – Defines the desired state of the application
* `apps/hello-world/service.yaml` – Exposes the application inside the cluster

This structure is intentionally simple to focus on **GitOps fundamentals**.

---

## Prerequisites

You will need:

* Kubernetes cluster (Minikube, kind, or any Kubernetes cluster)
* `kubectl`
* `git`
* GitHub account (or any Git provider)
* Internet access (to install Argo CD)

This demo was recorded using **Minikube**, but works the same on any cluster.

---

## Getting Started

### 1️⃣ Start Kubernetes (example with Minikube)

```bash
minikube start --driver=docker --cni=false
```

Verify access:

```bash
kubectl get nodes
```

---

### 2️⃣ Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until all pods are running:

```bash
kubectl get pods -n argocd -w
```

---

### 3️⃣ Access the Argo CD UI (local demo) (optional)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open:

```
https://localhost:8080
```

Retrieve the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

---

### 4️⃣ Bootstrap Argo CD (App of Apps Pattern)
This repository uses the App of Apps pattern so Argo CD manages all applications declaratively.
```bash
kubectl apply -f app-of-apps.yaml
```

## 🔐 TLS Setup (Local CA for HTTPS Demos)

### 1️⃣ Wait for cert-manager : (cert-manager is installed via GitOps)

```bash
kubectl -n cert-manager wait \
  --for=condition=available \
  --timeout=600s deployment/cert-manager || true
```
### 2️⃣ Generate Local CA Certificate

```bash
mkdir -p .certs
openssl genrsa -out .certs/ca.key 4096
openssl req -x509 -new -nodes \
  -key .certs/ca.key \
  -sha256 -days 365 \
  -out .certs/ca.crt \
  -subj "/CN=example-local-ca"

```

### 3️⃣ Create CA Secret for cert-manager

```bash
kubectl -n cert-manager create secret tls example-local-ca \
  --cert=.certs/ca.crt \
  --key=.certs/ca.key
```

## 🌐 Gateway API & Networking (Minikube)

### 1️⃣ Verify Cilium GatewayClass Status

```bash
kubectl get gatewayclass cilium-gatewayclass \
  -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}{"\n"}'
```
Expected output:

```graphql
True
```
### 2️⃣ Restart Cilium Operator (if not Accepted)

```bash
kubectl -n kube-system rollout restart deploy/cilium-operator
```
### 3️⃣ Expose Gateway IP on Minikube

Minikube does not provide LoadBalancer IPs by default. Run the tunnel:

```bash
minikube tunnel
```
Keep this running in a separate terminal.

### 4️⃣ Verify Gateway Address In another terminal:

```bash
kubectl get gateway -n default
```
You should see:

* `Programmed=True`
* An external IP address

```bash
<GATEWAY-IP> hello-world.example.com
```

### 5️⃣ Map Gateway IP Locally
In your `/etc/hosts` file add a line 

```bash
<GATEWAY-IP> hello-world.example.com
```

## ✅ Verification
Open in your browser type the link `https://hello-world.example.com`
You should see:

* HTTPS enabled
* Valid certificate (local CA)
* Application reachable through GitOps-managed resources

## Cleanup

```bash
minikube delete
```
## 📄 License

This demo is provided for **educational purposes**.

Use it, modify it, and adapt it for your own learning or demos.
