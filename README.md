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
| **Kubernetes cluster** | Cloud (EKS/GKE/AKS) | — |


> Verify your cluster is ready:
```bash
kubectl get nodes
```

---

## 2. Cluster Setup

If you don’t yet have a running kubernetes cluster but have a cloud account on AWS, we have provided terraform scripts in this repository to help you set up an EKS cluster. Navigate to the `terraform scripts` folder -> `eks-terraform` folder and follow steps in the README.md to setup a cluster with the necessary specs that Tazama requires.

- [Link](https://github.com/tazama-lf/cloud-infrastructure-deploy/tree/dev/terraform%20scripts) to the terraform scripts. Currently, only EKS scripts exist. AKS and GKE will be added soon.

Once ready, confirm connectivity:

```bash
kubectl cluster-info
kubectl get ns
```

---
