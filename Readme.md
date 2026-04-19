# Multi-Cloud Kubernetes E-Commerce with Consul Service Mesh

This project demonstrates deploying a multi-cluster, multi-cloud e-commerce application using **HashiCorp Consul** as a service mesh with automatic failover between clusters.

> **Original Authors / Credits:**
> - Application source & container images: [Google Cloud Platform – microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) (Apache 2.0 License)
> - Consul setup, manifests & tutorial: [TechWorld with Nana – Consul Crash Course](https://www.youtube.com/watch?v=s3I1kKKfjtQ) / [GitLab repo](https://gitlab.com/twn-youtube/consul-crash-course)

---

## Overview

The **Online Boutique** is a cloud-native microservices e-commerce demo by Google Cloud Platform. This project extends it by deploying across two Kubernetes clusters (AWS EKS + Linode LKE) connected via Consul service mesh with cluster peering and automatic failover for the `shippingservice`.

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Multi-Cloud Setup                        │
│                                                              │
│  ┌───────────────────────┐    ┌─────────────────────────┐   │
│  │   AWS EKS (Primary)   │◄──►│   Linode LKE (Secondary)│   │
│  │                       │    │                         │   │
│  │  Consul Server        │    │  Consul Server          │   │
│  │  + Mesh Gateway       │    │  + Mesh Gateway         │   │
│  │                       │    │                         │   │
│  │  All microservices    │    │  shippingservice        │   │
│  │  including            │    │  (failover target)      │   │
│  │  shippingservice      │    │                         │   │
│  │  (primary)            │    │                         │   │
│  └───────────────────────┘    └─────────────────────────┘   │
│                                                              │
│  Failover: EKS shippingservice down → routes to LKE         │
└──────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
.
├── README.md
├── kubernetes/
│   ├── config.yaml                           # Microservices without Consul injection
│   ├── config-consul.yaml                    # Microservices with Consul sidecar injection
│   ├── consul-values.yaml                    # Helm values for Consul installation
│   ├── consul-mesh-gateway.yaml              # Mesh CRD for cross-cluster peering
│   ├── exported-service.yaml                 # Export shippingservice to peer cluster
│   ├── intentions.yaml                       # Allow/deny traffic between services
│   ├── service-resolver.yaml                 # Failover config for shippingservice
│   ├── debug-pod.yaml                        # Debug pod for troubleshooting
│   └── values-examples-with-explanations.yaml  # Annotated Helm values reference
└── terraform/
    ├── main.tf                               # EKS cluster + VPC
    ├── variables.tf                          # Input variables
    ├── providers.tf                          # AWS provider
    ├── data.tf                               # Data sources
    └── terraform.tfvars                      # Your credentials (do NOT commit)
```

---

## Microservices (Online Boutique by GCP)

All images are pulled from Google Container Registry — no source build required.

| Service | Image | Port |
|---|---|---|
| frontend | `gcr.io/google-samples/microservices-demo/frontend:v0.8.0` | 8080 |
| cartservice | `gcr.io/google-samples/microservices-demo/cartservice:v0.8.0` | 7070 |
| productcatalogservice | `gcr.io/google-samples/microservices-demo/productcatalogservice:v0.8.0` | 3550 |
| currencyservice | `gcr.io/google-samples/microservices-demo/currencyservice` | 7000 |
| paymentservice | `gcr.io/google-samples/microservices-demo/paymentservice:v0.8.0` | 50051 |
| shippingservice | `gcr.io/google-samples/microservices-demo/shippingservice:v0.8.0` | 50052 |
| emailservice | `gcr.io/google-samples/microservices-demo/emailservice:v0.8.0` | 5000 |
| checkoutservice | `gcr.io/google-samples/microservices-demo/checkoutservice:v0.8.0` | 5050 |
| recommendationservice | `gcr.io/google-samples/microservices-demo/recommendationservice:v0.8.0` | 8080 |
| adservice | `gcr.io/google-samples/microservices-demo/adservice:v0.8.0` | 9555 |
| redis-cart | `redis:alpine` | 6379 |

---

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/downloads) >= 1.0
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/) >= 3.x
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- AWS account with EKS permissions
- Linode account (for LKE second cluster)

---

## Setup Instructions

### Step 1 — Provision EKS Cluster with Terraform

```sh
cd terraform/

# Create terraform.tfvars (do NOT commit this file)
cat > terraform.tfvars <<EOF
aws_access_key_id     = "YOUR_ACCESS_KEY"
aws_secret_access_key = "YOUR_SECRET_KEY"
EOF

terraform init
terraform apply -var-file terraform.tfvars -auto-approve
```

Configure kubectl:
```sh
aws configure
aws eks update-kubeconfig --region eu-central-1 --name myapp-eks-cluster
kubectl get svc
```

### Step 2 — Install Consul via Helm on Both Clusters

```sh
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Install on EKS
helm install consul hashicorp/consul \
  --values kubernetes/consul-values.yaml \
  --namespace consul --create-namespace \
  --kube-context <eks-context>

# Install on LKE
helm install consul hashicorp/consul \
  --values kubernetes/consul-values.yaml \
  --namespace consul --create-namespace \
  --kube-context <lke-context>
```

### Step 3 — Enable Cluster Peering

```sh
# Apply mesh gateway on both clusters
kubectl apply -f kubernetes/consul-mesh-gateway.yaml --context <eks-context>
kubectl apply -f kubernetes/consul-mesh-gateway.yaml --context <lke-context>

# Generate peering token on EKS
kubectl exec -it consul-server-0 -n consul --context <eks-context> -- \
  consul peering generate-token -name lke > /tmp/peering-token.txt

# Establish peering on LKE
kubectl exec -it consul-server-0 -n consul --context <lke-context> -- \
  consul peering establish -name eks -peering-token "$(cat /tmp/peering-token.txt)"
```

### Step 4 — Deploy Microservices

```sh
# Deploy all services on EKS with Consul injection
kubectl apply -f kubernetes/config-consul.yaml --context <eks-context>

# Deploy shippingservice on LKE as failover target
kubectl apply -f kubernetes/config-consul.yaml --context <lke-context>
```

### Step 5 — Configure Service Mesh

```sh
# Export shippingservice from LKE to EKS peer
kubectl apply -f kubernetes/exported-service.yaml --context <lke-context>

# Allow traffic via intentions on EKS
kubectl apply -f kubernetes/intentions.yaml --context <eks-context>

# Configure failover on EKS
kubectl apply -f kubernetes/service-resolver.yaml --context <eks-context>
```

### Step 6 — Access the Application

```sh
kubectl get svc frontend-external --context <eks-context>
# Open the EXTERNAL-IP in your browser
```

---

## Consul Helm Values

Key settings in `kubernetes/consul-values.yaml`:

```yaml
global:
  image: "hashicorp/consul:1.14.0"
  peering:
    enabled: true        # enables cluster peering
  tls:
    enabled: true        # encrypts all service mesh traffic

connectInject:
  enabled: true
  default: true          # auto-inject Consul sidecar to all pods

meshGateway:
  enabled: true          # required for cross-cluster traffic

ui:
  service:
    type: LoadBalancer   # expose Consul UI externally
```

---

## Failover Demo

`kubernetes/service-resolver.yaml` configures automatic failover:

```yaml
spec:
  failover:
    '*':
      targets:
        - peer: 'lke'    # if EKS shippingservice is down, route to LKE
```

Test it:
```sh
# Scale down shippingservice on EKS
kubectl scale deployment shippingservice --replicas=0 --context <eks-context>

# Checkout still works — Consul routes shipping calls to LKE automatically
```

---

## Consul UI

```sh
kubectl port-forward svc/consul-ui 8500:80 -n consul --context <eks-context>
# Open http://localhost:8500
```

---

## Terraform Commands

```sh
# Initialise project & download providers
terraform init

# Preview changes
terraform plan -var-file terraform.tfvars

# Apply with preview
terraform apply -var-file terraform.tfvars

# Apply without preview
terraform apply -var-file terraform.tfvars -auto-approve

# Destroy everything
terraform destroy -var-file terraform.tfvars -auto-approve

# Show current state
terraform state list
```

---

## Cleanup

```sh
kubectl delete -f kubernetes/config-consul.yaml --context <eks-context>
helm uninstall consul -n consul --kube-context <eks-context>
helm uninstall consul -n consul --kube-context <lke-context>
cd terraform/
terraform destroy -var-file terraform.tfvars -auto-approve
```

---

## Required Consul Ports

| Port | Protocol | Purpose |
|---|---|---|
| 8300 | TCP | RPC server communication |
| 8301 | TCP/UDP | LAN Serf gossip |
| 8302 | TCP/UDP | WAN Serf gossip |
| 8500 | TCP | HTTP API / UI |
| 8501 | TCP | HTTPS API (TLS) |
| 8600 | TCP/UDP | DNS |
| 8443 | TCP | Mesh Gateway |

---

## References

| Resource | Link |
|---|---|
| Consul Crash Course (TechWorld with Nana) | https://www.youtube.com/watch?v=s3I1kKKfjtQ |
| Nana's GitLab Repo | https://gitlab.com/twn-youtube/consul-crash-course |
| GCP Microservices Demo | https://github.com/GoogleCloudPlatform/microservices-demo |
| Consul Helm Chart Reference | https://developer.hashicorp.com/consul/docs/reference/k8s/helm |
| Consul Required Ports | https://developer.hashicorp.com/consul/docs/reference/architecture/ports |
| Consul Cluster Peering Docs | https://developer.hashicorp.com/consul/docs/connect/cluster-peering |
