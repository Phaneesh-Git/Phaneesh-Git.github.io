---
layout: post
title: "Self-Hosting personal website with Jenkins, Terraform, and Kustomize"
date: 2026-07-27
categories: [Kubernetes, Terraform]
tags: [Kubernetes, Terraform, Kind, Ingress, Jenkins, CI-CD, Cloudflare]
image:
  path: /assets/headers/2026-07-27.jpg
  alt: "Self-Hosting personal website with Jenkins, Terraform, and Kustomize"
  width: 1200
  height: 630
  aspect ratio: 1.91:1
published: true
---
> **Complete Tech Blog & Setup Guide**  
> *Author:* Phaneesh | *Date:* July 27, 2026 | *Repository:* [Phaneesh_website_on_kind_kustomize](https://github.com/Phaneesh-Git/Phaneesh_website_on_kind_kustomize)

* * *

## Introduction
Local development setups should mirror production as closely as possible. But historically, spinning up a local Kubernetes cluster that behaves like a real cloud environment—complete with load balancing, ingress routing, SSL certificate management, and a public URL—has been a painful chore. Developers often resort to manual hacks, inconsistent shell scripts, or bloated configurations that crumble under WSL hibernation or system reboots.

We wanted something better: a fully automated, declarative local platform. By pairing Terraform with KinD (Kubernetes in Docker), Kustomize, Jenkins, and Cloudflare Tunnels, we can provision a production-like cluster with a single command. In this guide, we'll dive deep into how this stack is built, how it handles complex local routing challenges, and how it automates the full CI/CD loop from a simple code commit to a secure, public HTTPS URL.

## Purpose of the Repository
The [Phaneesh_website_on_kind_kustomize](https://github.com/Phaneesh-Git/Phaneesh_website_on_kind_kustomize) repository acts as a blueprint for local platform engineering. Instead of treating local cluster creation as an afterthought, this repository treats the local development environment strictly as Infrastructure as Code (IaC). 

It automates:
- Provisioning a multi-node Kubernetes cluster locally using Terraform's Kind provider.
- Bootstrapping standard platform controllers: Ingress-NGINX, MetalLB, and Cert-Manager.
- Injecting security secrets (like Cloudflare Tunnel and API tokens) safely via Jenkins.
- Deploying and exposing a private containerized static website to the internet under a custom domain (`k8.phaneesh.dev`) without port forwarding or exposing home routers.

By treating the entire developer stack as code, any team member can stand up an identical, secure environment in minutes.

## Architecture and Workflow
The local cluster's architecture mimics a professional enterprise environment. The entire flow runs smoothly across a few key boundaries:

```
[ Local Git SCM ]
       │
       ▼ (Polls every 5m)
[ Jenkins Pipeline ] ──► Injects Secrets & Runs Makefile
       │
       ├─► [ Terraform ] ──► Provisions KinD Cluster (1 Control, 2 Workers)
       │
       └─► [ Kustomize ] ──► Installs Platform Bootstrap (Ingress-NGINX, MetalLB, Cert-Manager)
                                 │
                                 ├─► [ MetalLB ] (Allocates local VIPs)
                                 │
                                 ├─► [ Ingress-NGINX ] (Routes layer 7 HTTP traffic)
                                 │
                                 ├─► [ Cert-Manager ] (Requests TLS via Cloudflare DNS-01)
                                 │
                                 └─► [ Cloudflared ] (Establishes secure tunnel to Edge)
```

1. **Infrastructure Provisioning:** Terraform orchestrates the KinD cluster creation. It spins up a multi-node topology inside Docker, allowing us to validate scheduling behaviors and node-level configurations locally.
2. **Platform Bootstrapping:** A single `kustomization.yaml` deploys the core infrastructure controllers. MetalLB sets up Layer-2 address pools so Services of type `LoadBalancer` get real, routable IPs within Docker's bridge network. Ingress-NGINX sits on top of this, exposing standard ports.
3. **Automated Secret Configuration:** Jenkins acts as the orchestrator. It retrieves Zero-Trust API credentials from its credential store and safely applies them as Kubernetes secrets in the target namespaces. This keeps raw tokens out of git repositories.
4. **Edge Tunneling & DNS-01 Validation:** Instead of opening up firewall ports, an in-cluster Cloudflare Tunnel daemon (`cloudflared`) connects outbound to the Cloudflare Edge. This exposes our local ingress controller safely. Cert-manager uses Cloudflare DNS API tokens to solve ACME challenges, issuing real, trusted Let's Encrypt SSL certificates to the local cluster.

* * *

## Deep Dive: How the Traffic Components Integrate

To truly understand how this system operates, we must examine how the underlying network, routing, security, and edge tunneling components link up to route traffic from a public web browser directly down to our workstation.

### 1. The Network Foundation: MetalLB
In public cloud environments (such as AWS or GCP), requesting a Kubernetes Service of `type: LoadBalancer` automatically instructs the cloud provider to provision a physical, external load balancer (such as an ALB). However, locally inside a virtual environment like KinD, Kubernetes does not have a default load balancer controller. Without one, Services of type `LoadBalancer` remain stuck in a `<pending>` state indefinitely.

**MetalLB** resolves this issue. Operatng inside our local cluster in Layer-2 mode, it reads a configured IP address pool (defined in `k8s/config/metallb-config.yaml` as `172.18.255.200 - 172.18.255.250`) within the subnet of the KinD Docker bridge network. 

When the Ingress Controller requests a LoadBalancer Service, MetalLB assigns it a dedicated IP from this pool (e.g., `172.18.255.200`) and advertises this mapping via ARP (Address Resolution Protocol). This makes the Ingress controller instantly reachable inside our local environment.

### 2. The L7 Traffic Cop: Ingress-NGINX
Once MetalLB establishes a routable IP for our Ingress Controller, **Ingress-NGINX** acts as the reverse proxy for the cluster. It monitors the Kubernetes API server for `Ingress` resource declarations.

Our ingress manifest (`k8s/app/ingress.yaml`) dictates the layer-7 routing logic:
- If a request is received on port `80`/`443` where the HTTP `Host` header equals `k8.phaneesh.dev`, route it to the backend `website` Service on port `80`.
- NGINX reads this declaratively and dynamically recompiles its configuration file without requiring restarts or dropping active connections, providing zero-downtime updates.

### 3. DNS-01 Let's Encrypt Verification: Cert-Manager
Usually, securing a website with SSL from Let's Encrypt requires answering an `HTTP-01` challenge, which demands that Let's Encrypt servers reach your web server on port `80`. Because our local cluster resides behind a home router/NAT, Let's Encrypt cannot connect directly.

We solve this using **DNS-01 ACME challenges** with Cloudflare DNS:
1. **Request:** `cert-manager` requests a certificate for `k8.phaneesh.dev`.
2. **Challenge:** Let's Encrypt responds: *"Prove you own this domain by adding a temporary TXT record named `_acme-challenge.k8.phaneesh.dev` with this random verification token."*
3. **Execution:** Cert-Manager utilizes the Cloudflare API token (which Jenkins safely injected as a secret) to connect directly to the Cloudflare API, automatically adding the required TXT record to your DNS zone.
4. **Verification:** Let's Encrypt verifies the TXT record's presence publicly, approves ownership, and issues a valid, trusted SSL certificate.
5. **Cleanup:** Cert-Manager cleans up by deleting the temporary TXT record via Cloudflare API, and saves the TLS certificate as a Kubernetes Secret (`website-tls`). NGINX Ingress automatically consumes this secret to secure traffic.

### 4. Firewall-free Edge Routing: Cloudflare Tunnel
Traditionally, to expose a local web app to the public, you would have to open ports on your home router and set up dynamic DNS, introducing substantial security risks. 

**Cloudflare Tunnel** (`cloudflared`) establishes a secure, outbound-only connection. The `cloudflared` daemon deployed in our cluster initiates encrypted QUIC/TCP connections to Cloudflare's nearest edge servers using the secure token injected by Jenkins. Because this connection is outbound-only, your firewall allows it automatically, and you do not need to open any inbound ports.

When a public user visits `k8.phaneesh.dev`, Cloudflare's network forwards the traffic down this active tunnel directly to your local cluster, which points internally to the Ingress-NGINX service: `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local:80`.

### The Complete End-to-End Traffic Path

Here is how public internet requests securely resolve down to your workstation:

```
[ Public User ]
       │
       ▼ (HTTPS requests to k8.phaneesh.dev)
[ Cloudflare Global Edge ]
       │
       ▼ (Traffic routed securely DOWN the active outbound tunnel)
[ cloudflared Pod ] (Running inside KinD cluster)
       │
       ▼ (Forwards HTTP traffic locally)
[ NGINX Ingress Controller ] (Secured with Let's Encrypt SSL from website-tls Secret)
       │
       ▼ (Decrypts and applies Ingress L7 host rules)
[ website Service ] ──► [ website Pods ] (Renders static website assets)
```

* * *

## Code Snippets

Let's look at the core files that make this magic happen.

### Declaring KinD with Terraform
Using a specialized Terraform provider allows us to declare the cluster layout and immediately extract the `kubeconfig` outputs to a custom path. Here is how the multi-node setup is defined in `/terraform/main.tf`:

```terraform
resource "kind_cluster" "default" {
  name           = var.cluster_name
  wait_for_ready = true

  kind_config {
    kind        = "Cluster"
    api_version = "kind.x-k8s.io/v1alpha4"

    networking {
      api_server_address = "0.0.0.0"
      api_server_port    = 6443
    }

    node {
      role = "control-plane"
    }
    node { role = "worker" }
    node { role = "worker" }
  }
}
```

### Overcoming WSL/Container Networking Barriers
When running inside a containerized Jenkins agent or a Docker-in-Docker setup, connecting `kubectl` to the host's KinD cluster is notoriously tricky. The cluster's API endpoint is configured for the loopback interface, which is inaccessible from inside other containers.

The shell helper `scripts/deploy_kind_stack.sh` includes an ingenious network optimization routine. It dynamically discovers the host gateway IP and updates the `kubeconfig` server address on the fly:

```bash
fix_kubeconfig_routing() {
  if [ ! -f "$KIND_CONFIG" ]; then
    echo "Warning: KUBECONFIG not found at $KIND_CONFIG"
    return
  fi

  if [ -f /.dockerenv ] || grep -q 'docker' /proc/1/cgroup 2>/dev/null; then
    echo "==> Production Container Detected: Optimizing network routing..."
    need kubectl
    HOST_GATEWAY_IP=$(ip route show | awk '/default/ {print $3}')
    if [ -n "$HOST_GATEWAY_IP" ]; then
      CURRENT_CONTEXT=$(k config current-context)
      k config set-cluster "$CURRENT_CONTEXT" --server="https://${HOST_GATEWAY_IP}:6443" --insecure-skip-tls-verify=true
    fi
  fi
}
```

### Declarative Ingress and Let's Encrypt Integration
To glue our secure tunnel and public routing together, our `ingress.yaml` links directly to a Cert-Manager `ClusterIssuer` configured for Cloudflare DNS-01 validation. This ensures we receive a valid Let's Encrypt TLS certificate without exposing an HTTP endpoint publicly:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: website
  namespace: web
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare
    nginx.ingress.kubernetes.io/use-forwarded-headers: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - k8.phaneesh.dev
      secretName: website-tls
  rules:
    - host: k8.phaneesh.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: website
                port:
                  number: 80
```

## Key Takeaways

Building this automated environment taught us several crucial platform engineering lessons:

- **True Environment Parity:** Declaring local environments with the same GitOps tools (Kustomize/Terraform) used in production reduces the risk of "works on my machine" bugs. Your local cluster has real SSL, real routing, and real load balancers.
- **Firewallless Ingress:** Cloudflare Tunnels are a game-changer for local development. By establishing outbound tunnels, we completely bypass port forwarding and dynamic DNS configs.
- **Robust Local Recovery:** When operating system hibernation or WSL reboots empty out your local cluster's state, having a simple `make all` command restores the exact cluster state, routing tables, and deployments in under 3 minutes.
- **GitOps for Secrets:** Using Jenkins credentials injection combined with dry-run secret generation ensures local secrets are never stored in plain text, maintaining high security standards.

This setup proves that local Kubernetes development doesn't have to be a compromise. With the right tools and automation, your local desktop can run a fully secure, auto-recovering platform that mirrors a multi-million dollar cloud infrastructure!