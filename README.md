<!-- SPDX-License-Identifier: Apache-2.0 --> 

# Still A Work In Progress

# Tazama Cloud Installation and Deployment Guide

This is an end to end cloud setup guide for installing Tazama software ((Real-time Antifraud and Money Laundering Monitoring System)) in a production environment using Kubernetes, Terraform, Helm, ArgoCD, Kustomize, gitops principles and officially published dockerhub images for the different Tazama compinents.

It covers setup, dependency installation (via Helm), application management, and environment customization through Kustomize overlays.

## **Intended Users:** 
- DevOps engineers
- Developers / Engineers 
- Open-source contributors deploying or extending the Tazama microservices.

`This guide also assumes the above users have foundational understanding of Kubernetes concepts e.g pods, namespaces, ingress and other general DevOps tools e.g Docker, Helm and ArgoCD.`

---

## Overview

**Tazama** is an open-source platform for **real-time fraud detection and transaction monitoring**. This repository and guide provide a fully automated GitOps workflow:

- Declarative infrastructure & apps managed via **Argo CD**
- Configuration through **Kustomize overlays** ( e.g `staging`, `prod`)
- Dependency management using **Helm charts**
- Secrets management via **Kubernetes Secrets** (This section will be added later)
- Continuous delivery triggered by **GitHub Actions** (This section will be added later)

---

## 1. Prerequisites

| Tool | Description | Installation |
|------|--------------|----------|
| **kubectl** | CLI for interacting with Kubernetes | [Install guide](https://kubernetes.io/docs/tasks/tools/) |
| **helm** | Package manager for Kubernetes | [Install Helm](https://helm.sh/docs/intro/install/) |
| **kustomize** | Native k8s configuration management | Included in kubectl ≥ v1.14 |
| **Argo CD CLI** (optional) | Manage apps from terminal | [Install Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) |
| **docker** | Verify images locally | [Install Docker](https://docs.docker.com/get-docker/) |
| **Kubernetes cluster** | Cloud (EKS/GKE/AKS) | Check next section `Cluster Setup` |


> Verify your cluster is ready:
```bash
kubectl get nodes
```

---

## 2. Cluster Setup

If you don’t yet have a running kubernetes cluster but have a cloud account on AWS, we have provided terraform scripts for now in this repository to help you set up an EKS cluster. Navigate to the `terraform scripts` -> `eks-terraform` folder and follow steps in the README.md to setup a cluster with the necessary specs that Tazama requires.

- [Link](https://github.com/tazama-lf/cloud-infrastructure-deploy/tree/dev/terraform-scripts) to the terraform scripts. Currently, only EKS scripts exist. AKS and GKE will be added soon.

Once ready, confirm connectivity:

```bash
kubectl cluster-info
kubectl get ns
```

---

## 3. Install Helm Dependencies

Before deploying Tazama apps, install the supporting infrastructure using Helm.

| Dependency | Chart | Purpose |
|-------------|--------|----------|
| **NATS** | `nats/nats` | Messaging backbone |
| **PostgreSQL** | `bitnami/postgresql` | Core transactional database |
| **Keycloak** | `codecentric/keycloakx` | Identity & access management |
| **NGINX Ingress** | `ingress-nginx/ingress-nginx` | Reverse proxy & routing |
| **Prometheus & Grafana** | `prometheus-community/kube-prometheus-stack` | Metrics and dashboards |
| **Elastic Stack (ELK)** | `elastic/helm-charts` | Centralized logging and observability |


### Add Helm Repositories

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add nats https://nats-io.github.io/k8s/helm/charts/
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add elastic https://helm.elastic.co
helm repo update
```

### Install the added Helm Repositories
```bash
# NATS - messaging backbone
helm install nats nats/nats -n nats --create-namespace
# PostgreSQL - main database
helm install postgres bitnami/postgresql -n db --create-namespace
# NGINX Ingress - reverse proxy
helm install ingress ingress-nginx/ingress-nginx -n ingress --create-namespace
# Prometheus & Grafana - monitoring stack
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
# Elastic Stack (ELK) - logging and observability
helm install elasticsearch elastic/elasticsearch -n logging --create-namespace
```

> Note: These default installations are suitable for development and testing environments. For production deployments, review and customize each chart’s values.yaml file and enable persistence, authentication, and proper resource limits.

---

## 4. Install Argo CD

Argo CD is a declarative, GitOps-based continuous delivery tool for Kubernetes. It continuously monitors your Git repository and automatically syncs your manifests to your cluster.

### Install Argo CD in the argocd namespace:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Once installed, verify the pods:

```bash
kubectl get pods -n argocd
```

Expected output should include components like:

```bash
argocd-server
argocd-repo-server
argocd-application-controller
argocd-dex-server
```

### Access the UI

To access the UI locally, port-forward the service:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# Retrieve the initial admin password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

```

Open your browser at:
https://localhost:8080


```bash
Credentials:
Username: admin
Password: (admin password value retrieved)
```

- **Tip**: Change the default password after first login: `Settings → Accounts → admin → Update Password`

### Optional: Install Argo CD CLI

If you prefer managing Argo CD from the command line, install the CLI tool:

```bash
# macOS (Homebrew)
brew install argocd
# Linux
sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```

Verify installation:

```bash
argocd version
```

Then log in using the same credentials:

```bash
argocd login localhost:8080
```

---

## 5. Deploy Tazama Services via Argo CD

Now that Argo CD is installed and running, you can deploy the entire **Tazama Platform** using the **App-of-Apps** pattern — a GitOps best practice for managing multiple Kubernetes applications from a single source of truth.


### Overview: App-of-Apps Pattern

The **App-of-Apps** pattern allows you to define one “root” Argo CD Application that automatically creates and manages all other microservice apps (rule-001, rule-002, event-flow, etc.). This will sync the services into the staging namespace and deploys images from `tazamaorg/*:2.2.0`

This setup ensures:
- Consistent configuration across environments (staging, production).
- Centralized version control for all manifests.
- Automated, self-healing synchronization from Git.


### Apply the Root Application

To bootstrap your cluster with all Tazama services:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/tazama-lf/cloud-infrastructure-deploy/main/apps/app-of-apps.yaml
```

- Monitor progress by running `kubectl -n argocd get applications` Or open the Argo CD UI → view synced apps.

> The manifest above defines a root Argo CD Application pointing to the [cloud-infrastructure-deploy](https://github.com/tazama-lf/cloud-infrastructure-deploy) repo, which in turn manages all the defined Tazama components. Once the manifest is applied, Argo CD detects the new “App-of-Apps” definition, It clones the repository, creates child applications for each service defined under `apps/` and each child app deploys its manifests from `k8s/base` and environment overlays (`k8s/overlays/staging or prod`)


Verify Applications in Argo CD by checking that your apps have been created and synced
```bash
kubectl get applications -n argocd
```

Expected output:
```bash
NAME             SYNC STATUS   HEALTH STATUS
rule-001         Synced        Healthy
rule-002         Synced        Healthy
etc.
```

> You can also view them in the Argo CD UI:

Open https://localhost:8080
 → Log in → Applications Dashboard.

### Verify Deployment 

Check workloads running in the Cluster by running;

```bash
kubectl get pods -n staging
kubectl get svc -n staging
```
