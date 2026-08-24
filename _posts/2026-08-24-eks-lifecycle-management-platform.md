---
layout: post
title: "Production-Ready EKS Lifecycle Platform: Multi-Tier Compute with Karpenter and GitOps"
date: 2026-08-24
categories: [Kubernetes, IaC]
tags: [EKS, Karpenter, ArgoCD, Terraform, AWS]
image:
  path: /assets/headers/2026-08-24.jpg
  alt: "Production-Ready EKS Lifecycle Platform"
  width: 1200
  height: 630
  aspect ratio: 1.91:1
published: true
---
> **Complete Tech Blog & Setup Guide**  
> *Author:* Phaneesh | *Date:* August 24, 2026 | *Repository:* [EKS Cluster Life Cycle Platform](https://github.com/Phaneesh-Git/eks-cluster-lifecycle-platform)

* * *

## Introduction

Managing the lifecycle of enterprise-grade Amazon EKS (Elastic Kubernetes Service) clusters requires a cohesive strategy that integrates Infrastructure as Code (IaC), dynamic compute scaling, and declarative continuous delivery. A common pitfall in platform engineering is treating these layers as isolated components, which inevitably leads to configuration drift, inefficient resource utilization, and complex deployment processes.

This guide explores a production-ready **EKS Cluster Lifecycle Management Platform** that solves these challenges through a unified **Hub-and-Spoke** architecture. By combining **Terraform** for base infrastructure, **Karpenter** for high-performance node auto-scaling, **Argo CD** for GitOps-driven deployment, and **Jenkins** for pipeline orchestration, this platform provides a robust, scalable, and self-healing environment for containerized workloads.

---

## The Architectural Design: Hub-and-Spoke Infrastructure Control

The platform is designed around a clean separation of concerns between **Infrastructure Provisioning** and **Workload Delivery**. It utilizes a Hub-and-Spoke model where a central orchestration engine manages downstream spoke clusters.

```text
+----------------------------------------------------------------------------------------------------+
| AWS SINGLE ACCOUNT (Production / Staging / Dev)                                                    |
|                                                                                                    |
|  +---------------------------------+                  +------------------------------------------+  |
|  |     Jenkins CI/CD Pipeline      |                  |                 Argo CD                  |  |
|  | - Executes `terraform`          |                  | - Central UI & Control Plane             |  |
|  | - Manages Cloud Infrastructure  |                  | - Watches Git & Syncs K8s State          |  |
|  +────────────────┬────────────────+                  +────────────────────┬────────────────-----+  |
|                   │                                                        │                        |
|                   │ (Provisions Cloud Infra)                               │ (Syncs K8s Manifests)  |
|                   ▼                                                        ▼                        |
|  +----------------------------------------------------------------------------------------------+  |
|  | VPC (CIDR: 10.86.0.0/16)                                                                     |  |
|  |                                                                                              |  |
|  |  +---------------------------+                      +-------------------------------------+  |  |
|  |  |   EKS Control Plane       |<────────────────────>|    KMS Customer Managed Key          |  |  |
|  |  +─────────────┬─────────────+                      +-------------------------------------+  |  |
|  |                │                                                                             |  |
|  |                ▼                                                                             |  |
|  |  +────────────────────────────────────────────────────────────────────────────────────────+  |  |
|  |  | Tier 1: System Managed Node Group (Terraform Provisioned)                               |  |  |
|  |  | - Hosts: CoreDNS, VPC-CNI, ALB Controller, Argo CD, & Karpenter Controller              |  |  |
|  |  +─────────────────────────────┬──────────────────────────────────────────────────────────+  |  |
|  |                                │                                                             |  |
|  |                                ▼                                                             |  |
|  |  +────────────────────────────────────────────────────────────────────────────────────────+  |  |
|  |  | Tier 2: Karpenter Dynamic NodePool (Auto-scaled by Karpenter)                           |  |  |
|  |  | - Scaled on-demand according to NodePool manifests synced via Argo CD                     |  |  |
|  |  +─────────────────────────────┬──────────────────────────────────────────────────────────+  |  |
|  |                                │                                                             |  |
|  |                                ▼                                                             |  |
|  |  +────────────────────────────────────────────────────────────────────────────────────────+  |  |
|  |  | Workloads: Application Pods, Services, & PodDisruptionBudgets (PDBs)                  |  |  |
|  |  +────────────────────────────────────────────────────────────────────────────────────────+  |  |
|  +----------------------------------------------------------------------------------------------+  |
+----------------------------------------------------------------------------------------------------+
```

In this architecture:
1. **Jenkins** acts as the IaC lifecycle orchestrator, running automated Terraform pipelines to build, modify, or destroy AWS environments securely.
2. **Argo CD** serves as the Continuous Delivery plane, continuously auditing and synchronizing Kubernetes manifests from Git to the EKS cluster.
3. **Multi-Tier Compute** divides the Kubernetes cluster into two specialized tiers: a stable system tier and an elastic, cost-optimized workloads tier.

---

## Bottom-Up IaC: Terraform Modules and Dependency Graphs

The foundation of the platform is constructed using modular Terraform code. Terraform automatically compiles a **Directed Acyclic Graph (DAG)** of dependencies to determine the optimal order of creation. This bottom-up approach ensures network and security policies are firmly established before compute nodes are attached:

1. **Virtual Private Cloud (VPC) Module** (`terraform/modules/vpc/`): Establishes isolated public and private subnets across multiple Availability Zones, custom routing tables, an Internet Gateway, and a highly available NAT Gateway. Crucially, it applies specific tagging structures required for Kubernetes discovery (e.g., `kubernetes.io/role/elb = 1`).
2. **EKS Control Plane Module** (`terraform/modules/eks_control_plane/`): Deploys a highly secure Kubernetes Control Plane (v1.31) with AWS KMS envelope encryption for Kubernetes secrets, and sets up the OpenID Connect (OIDC) provider needed for IAM Roles for Service Accounts (IRSA).
3. **Compute Modules** (`terraform/modules/system_node_group/` & `terraform/modules/karpenter/`): Concurrently provision system managed node groups and Karpenter's service roles/SQS interruption queues once the VPC and EKS Control Plane are healthy.

---

## The Dual-Compute Tier Strategy

A cornerstone of the platform’s efficiency is the **Dual-Compute Tier Strategy**. Instead of running all applications on a single, monolithic node group, the cluster segregates system components from tenant workloads.

### Tier 1: Fixed Systems Node Group
Managed directly by Terraform, this tier consists of a stable, highly available AWS Managed Node Group (typically 2-3 instances across AZs). It runs on Amazon Linux 2023 (`AL2023_x86_64_STANDARD`) with a conservative, predictable footprint. This tier is dedicated strictly to critical cluster-wide utilities:
* **CoreDNS** and **VPC-CNI**
* **AWS Load Balancer Controller**
* **Argo CD** Control Plane
* **Karpenter Controller**

By isolating system services on a separate node group, we ensure they are never starved of resources or disrupted by user application churn.

### Tier 2: Karpenter Dynamic NodePool
Workloads and user-facing applications are run exclusively on dynamically provisioned nodes managed by **Karpenter**. Karpenter bypasses traditional AWS Auto Scaling Groups (ASGs), interacting directly with the EC2 Fleet API to spin up custom-sized instances in millisecond speeds.

Below is the production Karpenter `NodePool` configuration used in the spoke production cluster (`gitops/spoke/01-cluster-bootstrap/karpenter/nodepool-general.yaml`):

```yaml
# ==============================================================================
# Component: Karpenter General NodePool
# Project: EKS_CLUSTER_LIFE_CYCLE
# ==============================================================================
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: eks-cluster-life-cycle-nodepool-general
  labels:
    project: EKS_CLUSTER_LIFE_CYCLE
spec:
  template:
    metadata:
      labels:
        project: EKS_CLUSTER_LIFE_CYCLE
        tier: workloads
    spec:
      nodeClassRef:
        name: EKS_CLUSTER_LIFE_CYCLE-ec2nodeclass
      requirements:
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["spot", "on-demand"]
        - key: "node.kubernetes.io/instance-type"
          operator: In
          values: ["m5.large", "m5.xlarge", "c5.large"]
  limits:
    cpu: "500"
    memory: "2000Gi"
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h # Forces node recycling every 30 days
```

### Why this design works:
* **Cost Optimization**: Karpenter mixes `spot` and `on-demand` instances. It dynamically schedules spot instances for stateless workloads and on-demand instances when consistency is required.
* **Underutilization Consolidation**: By setting `consolidationPolicy: WhenUnderutilized`, Karpenter automatically identifies idle nodes, deprovisions them, and reschedules pods onto more compact instances.
* **Security & Recyclability**: The `expireAfter: 720h` setting automatically recycles instances every 30 days, guaranteeing that nodes are constantly refreshed with the latest OS patches and security updates.

---

## Continuous Delivery via Argo CD App-of-Apps Pattern

With infrastructure in place, application deployment is handed over to Argo CD using the **App-of-Apps** pattern. 

Under this model, a central **Root Application** (`gitops/hub/root-app.yaml`) watches a single Git repository directory containing declarative configuration definitions for other "child" applications.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: EKS_CLUSTER_LIFE_CYCLE-root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/Phaneesh-Git/eks-cluster-lifecycle-platform.git'
    targetRevision: HEAD
    path: gitops/hub
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The Root Application synchronizes the downstream spoke cluster ecosystem applications, which are divided into two operational phases:
1. **01-Cluster-Bootstrap**: Provisions cluster-wide infrastructure apps such as Ingress Controllers, Monitoring, and the Karpenter `NodePool`/`EC2NodeClass` resources.
2. **02-Workloads**: Manages the deployment of actual user workloads using **Kustomize**. The workloads folder contains a `base/` configuration (deployments, services, PDBs) and specialized `overlays/` for different environments. For instance, the `overlays/prod/replicas-patch.yaml` scales production instances up to 5 replicas automatically.

---

## Critical Operations: The Kubernetes Pre-Destroy Hook

A massive issue in cloud orchestration is **state synchronization during environment tear-down**. When running `terraform destroy` on an EKS cluster, the process will frequently freeze, timeout, or fail. 

### Why does this happen?
1. **Orphan ENIs (Elastic Network Interfaces)**: Services with `Type: LoadBalancer` and Ingress controllers create AWS Load Balancers and attach ENIs to cluster private subnets. If the EKS cluster is deleted before these Kubernetes resources are cleared, AWS cannot delete the subnets, causing the VPC deletion to fail.
2. **Finalizer Deadlocks**: Custom Resources (such as Karpenter's `NodePool` or `EC2NodeClass`) contain **finalizers**—Kubernetes metadata hooks that block resource deletion until specific cleanups complete. If the Karpenter controller is deleted first, these finalizers will hang indefinitely.

To solve this, the platform utilizes a robust pre-destroy cleanup hook (`scripts/pre-destroy-cleanup.sh`):

```bash
#!/usr/bin/env bash
# ==============================================================================
# Script: Pre-Destroy Cleanup
# Project: EKS_CLUSTER_LIFE_CYCLE
# Description: Cleans up Kubernetes custom resources and finalizers before terraform destroy.
# ==============================================================================
set -Eeuo pipefail

CLUSTER_NAME="${1:-EKS_CLUSTER_LIFE_CYCLE-spoke-prod}"
REGION="${AWS_REGION:-us-east-1}"

echo "[INFO] Configuring kubeconfig for cluster: ${CLUSTER_NAME}..."
if aws eks update-kubeconfig --name "${CLUSTER_NAME}" --region "${REGION}" --quiet 2>/dev/null; then
  echo "[INFO] Removing Karpenter NodePools and EC2NodeClasses..."
  kubectl delete nodepools --all --ignore-not-found=true --timeout=30s || true
  kubectl delete ec2nodeclasses --all --ignore-not-found=true --timeout=30s || true

  echo "[INFO] Deleting application workloads..."
  kubectl delete deployments --all -n default --ignore-not-found=true --timeout=30s || true
  kubectl delete services --all -n default --ignore-not-found=true --timeout=30s || true
  
  echo "[SUCCESS] Kubernetes pre-destroy cleanup completed."
else
  echo "[WARNING] Could not connect to cluster control plane. Skipping K8s cleanup."
fi
```

This script is executed as a mandatory pre-destroy hook in the Jenkins pipeline before running `terraform destroy`. By systematically deleting Karpenter resources first, we release the dynamically scaled instances. Then, by deleting default namespace services and deployments, we release any AWS-managed resources and ENIs, clearing the path for Terraform to tear down the VPC cleanly.

---

## Key Takeaways & Operational Benefits

By implementing the `EKS_CLUSTER_LIFE_CYCLE` platform, engineering teams unlock several core benefits:

* **Surgical Resource Scheduling**: Compute separation ensures infrastructure stability while optimizing application runtime costs.
* **True GitOps Alignment**: The App-of-Apps pattern makes the entire cluster configuration auditable, repeatable, and self-healing.
* **Deterministic Lifecycles**: Operational scripts like `pre-destroy-cleanup.sh` and `aws-resource-audit.sh` transform volatile teardowns and audits into reliable, automated tasks.
* **Unified Control Plane**: Combining Terraform and Argo CD leverages the best of both worlds—imperative workflow boundaries for base infrastructure, and continuous state reconciliation for Kubernetes workloads.

This blueprint demonstrates how combining powerful cloud-native tools with careful design patterns results in a robust, production-ready, and highly automated Kubernetes lifecycle platform.
