# claude-project1: Kubernetes nginx-ingress + ArgoCD + WebApp

> Deployed and documented with Claude AI on Docker Desktop Kubernetes

---

## Overview

This project sets up a complete Kubernetes environment on **Docker Desktop** with:

- **nginx-ingress-controller** — single NodePort entry point for all HTTP traffic
- **Argo CD** — GitOps continuous delivery platform, accessible via Ingress
- **Simple Web App** — nginx-based app with custom HTML, accessible via Ingress

Both Argo CD and the Web App are accessible from your browser using host-based routing through a single NodePort (`30080`).

---

## Architecture

```
+-------------------------------------------------------------------------+
|                     Docker Desktop Kubernetes Cluster                   |
|                                                                         |
|   Browser                                                               |
|   http://argocd.local:30080 --+                                         |
|   http://webapp.local:30080 --+                                         |
|                               |                                         |
|                               v                                         |
|          +----------------------------------------+                    |
|          |        Namespace: ingress-nginx         |                    |
|          |                                        |                    |
|          |   ingress-nginx-controller (Deployment)|                    |
|          |   Service: NodePort                    |                    |
|          |     port 30080 -> 80  (HTTP)           |                    |
|          |     port 30443 -> 443 (HTTPS)          |                    |
|          +------------+---------------+-----------+                    |
|                       |               |                                 |
|              Host: argocd.local  Host: webapp.local                   |
|                       |               |                                 |
|          +------------v----------+  +-v-----------------------+        |
|          | Namespace: argocd     |  | Namespace: webapp       |        |
|          |                       |  |                         |        |
|          |  argocd-server        |  |  webapp (2 replicas)   |        |
|          |  argocd-repo-server   |  |  image: nginx:alpine   |        |
|          |  argocd-app-ctrl      |  |                         |        |
|          |  argocd-redis         |  |  Service: ClusterIP    |        |
|          |                       |  |  Ingress: webapp.local |        |
|          |  Ingress: argocd.local|  |                         |        |
|          +-----------------------+  +-------------------------+        |
|                                                                         |
|  Cluster Resources:                                                     |
|  CRDs: applications, appprojects, applicationsets (argoproj.io)        |
|  IngressClass: nginx (default)                                          |
+-------------------------------------------------------------------------+
```

---

## Directory Structure

```
claude-project1/
├── README.md
├── ingress-nginx/
│   ├── namespace.yaml
│   ├── serviceaccount.yaml
│   ├── clusterrole.yaml
│   ├── clusterrolebinding.yaml
│   ├── role.yaml
│   ├── rolebinding.yaml
│   ├── configmap.yaml
│   ├── ingressclass.yaml
│   ├── deployment.yaml
│   └── service-nodeport.yaml
├── argocd/
│   ├── namespace.yaml
│   ├── serviceaccounts.yaml
│   ├── rbac.yaml
│   ├── configmaps.yaml
│   ├── secret.yaml
│   ├── crds.yaml
│   ├── appproject-default.yaml
│   ├── redis.yaml
│   ├── repo-server.yaml
│   ├── server.yaml
│   ├── application-controller.yaml
│   └── ingress.yaml
└── webapp/
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

## Components

### 1. nginx-ingress-controller (`ingress-nginx/`)

The single entry point for all external HTTP/HTTPS traffic. Exposed via NodePort so it is reachable from your local machine.

| Resource | Purpose |
|----------|---------|
| Namespace | Isolates ingress components |
| ServiceAccount | Identity for the controller pod |
| ClusterRole + ClusterRoleBinding | Cluster-wide watch permissions |
| Role + RoleBinding | Namespace-scoped ConfigMap/lease permissions |
| ConfigMap | Controller configuration |
| IngressClass | Registers `nginx` as the default class |
| Deployment | Runs ingress-nginx/controller:v1.10.0 |
| Service (NodePort) | Exposes port 30080 (HTTP) and 30443 (HTTPS) |

**How routing works:** nginx reads the `Host` header of each request and matches it against `Ingress` resources across all namespaces, then proxies to the correct backend Service.

---

### 2. Argo CD (`argocd/`)

GitOps CD platform that monitors Git repos and syncs desired state to the cluster.

| Resource | Purpose |
|----------|---------|
| Namespace | argocd namespace |
| ServiceAccounts | Per-component identities |
| ClusterRoles + Bindings | Full cluster access for server and controller |
| ConfigMap: argocd-cm | Main config, sets external URL |
| ConfigMap: argocd-rbac-cm | RBAC policy defaults |
| Secret: argocd-secret | Admin password + server secret key |
| CRDs | Application, AppProject, ApplicationSet |
| AppProject: default | Allows all sources and destinations |
| Deployment: argocd-redis | Session/state cache |
| Deployment: argocd-repo-server | Clones and renders Git manifests |
| Deployment: argocd-server | Web UI + API (runs --insecure for HTTP) |
| StatefulSet: argocd-app-controller | Reconciles Git state to cluster |
| Ingress | Routes argocd.local -> argocd-server:80 |

**Access:** http://argocd.local:30080  
**Login:** admin / admin

---

### 3. Simple Web App (`webapp/`)

A 2-replica nginx app serving a custom HTML landing page built at startup via initContainer.

| Resource | Purpose |
|----------|---------|
| Namespace | webapp namespace |
| Deployment | 2 x nginx:alpine with HTML via initContainer |
| Service (ClusterIP) | Internal port 80 |
| Ingress | Routes webapp.local -> webapp:80 |

**Access:** http://webapp.local:30080

---

## Quick Start

### Prerequisites

- Docker Desktop with Kubernetes enabled
- kubectl configured to docker-desktop context

```bash
kubectl config use-context docker-desktop
```

### Deploy

```bash
# Clone
git clone https://github.com/kiranch97/claude-project1.git
cd claude-project1

# 1. nginx-ingress
kubectl apply -f ingress-nginx/
kubectl rollout status deployment/ingress-nginx-controller -n ingress-nginx

# 2. Argo CD
kubectl apply -f argocd/
kubectl get pods -n argocd -w

# 3. Web App
kubectl apply -f webapp/
kubectl rollout status deployment/webapp -n webapp
```

### Configure /etc/hosts

Add to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
127.0.0.1  argocd.local
127.0.0.1  webapp.local
```

### Access

| App | URL | Credentials |
|-----|-----|-------------|
| Argo CD | http://argocd.local:30080 | admin / admin |
| Web App | http://webapp.local:30080 | — |

---

## Verification

```bash
kubectl get namespaces
kubectl get pods,svc -n ingress-nginx
kubectl get pods -n argocd
kubectl get pods,svc -n webapp
kubectl get ingress -A
```

---

## Cleanup

```bash
kubectl delete -f webapp/
kubectl delete -f argocd/
kubectl delete -f ingress-nginx/
```

---

## Security Notes

- Change the ArgoCD admin password after first login
- argocd-server uses --insecure (HTTP only) — add TLS in production
- The default AppProject allows all sources/destinations — restrict in production

---

## Author

**kiranch97** — Deployed collaboratively with Claude AI  
Date: March 2026
