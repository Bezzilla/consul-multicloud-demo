# Multi-Cloud Kubernetes E-Commerce with Consul Service Mesh

This project demonstrates deploying a multi-cluster, multi-cloud e-commerce application using **HashiCorp Consul** as a service mesh with automatic failover between clusters.

> **Original Authors / Credits:**
> - Application source & container images: [Google Cloud Platform – microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) (Apache 2.0 License)
> - Consul setup, manifests & tutorial: [TechWorld with Nana – Consul Crash Course](https://www.youtube.com/watch?v=s3I1kKKfjtQ) / [GitLab repo](https://gitlab.com/twn-youtube/consul-crash-course)


## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/downloads) >= 1.0
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/) >= 3.x
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- AWS account with EKS permissions
- Linode account (for LKE second cluster)

---

## Hosting the Application on AWS

### Step 1 — Provision EKS Cluster with Terraform

Navigate to the terraform directory:
```sh
cd terraform/
```

Create a `terraform.tfvars` file:
```sh
# Windows
type nul > terraform.tfvars

# Mac / Linux
touch terraform.tfvars
```

Configure the required variables:
```hcl
aws_access_key_id     = "your-aws-access-key-id"
aws_secret_access_key = "your-aws-access-secret-key"
```

The following variables are optionally configurable:
```hcl
aws_region       = "ap-southeast-1"  # AWS region for resources
instance_types   = ["t3.small"]      # AWS free tier supports t3, t4 small for accounts after 2015
k8s_cluster_name = "myapp-eks-cluster"  # Kubernetes cluster name
k8s_version      = "1.30"           # Kubernetes version (< 1.33 deprecated after June 2026)
```

Create the AWS resources:
```sh
terraform init -upgrade
terraform apply -var-file terraform.tfvars -auto-approve
```

### Step 2 — Connect kubectl to EKS

```sh
# Authenticate AWS CLI
aws configure

# Update kubeconfig to point to EKS cluster
aws eks update-kubeconfig --region eu-central-1 --name myapp-eks-cluster

# Verify connection
kubectl get svc
```

### Step 3 — Deploy Microservices (Plain Kubernetes)

```sh
cd kubernetes/

# Deploy all microservices
kubectl apply -f config.yaml

# Check pod status
kubectl get pods

# Check services
kubectl get services
```

> **Note:** `paymentservice` will show `CrashLoopBackOff` — this is intentional in the demo.

### Step 4 — Create EBS Storage for Consul

```sh
kubectl apply -f config-gp3.yaml
```

### Step 5 — Install Consul on EKS

```sh
# Register the Consul Helm repository
helm repo add hashicorp https://helm.releases.hashicorp.com

# Install Consul on EKS cluster
helm install eks hashicorp/consul --version 1.0.0 \
  --values consul-values.yaml \
  --set global.datacenter=eks
```

Verify Consul pods are running:
```sh
kubectl get pods
```

Expected output:
```
NAME                                               READY   STATUS    RESTARTS   AGE
eks-consul-connect-injector-65cb48f9bb-bwcht       1/1     Running   0          45s
eks-consul-mesh-gateway-66bf4dd558-gwkfz           1/1     Running   0          23m
eks-consul-server-0                                1/1     Running   0          23m
eks-consul-webhook-cert-manager-845479654f-92pth   1/1     Running   0          23m
```

### Step 6 — Redeploy Microservices with Consul Sidecar Injection

```sh
# Remove existing deployments
kubectl delete -f config.yaml

# Redeploy with Consul sidecar injection enabled
kubectl apply -f config-consul.yaml
```

### Step 7 — Enable Cluster Peering (EKS ↔ LKE)

```sh
# Apply mesh gateway on both clusters
kubectl apply -f consul-mesh-gateway.yaml --context <eks-context>
kubectl apply -f consul-mesh-gateway.yaml --context <lke-context>

# Generate peering token on EKS
kubectl exec -it consul-server-0 -n consul --context <eks-context> -- \
  consul peering generate-token -name lke > /tmp/peering-token.txt

# Establish peering on LKE using the token
kubectl exec -it consul-server-0 -n consul --context <lke-context> -- \
  consul peering establish -name eks -peering-token "$(cat /tmp/peering-token.txt)"
```

### Step 8 — Configure Failover

```sh
# Export shippingservice from LKE to EKS peer
kubectl apply -f exported-service.yaml --context <lke-context>

# Allow traffic via intentions on EKS
kubectl apply -f intentions.yaml --context <eks-context>

# Configure automatic failover
kubectl apply -f service-resolver.yaml --context <eks-context>
```

### Step 9 — Access the Application

```sh
kubectl get svc frontend-external
# Open the EXTERNAL-IP in your browser
```

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

## Hosting the Application on Linode (LKE)

### Step 1 — Create a Linode Kubernetes Cluster

1. Log in to your Linode account and navigate to **Kubernetes**
2. Click **Create Cluster** and choose a name and a region closest to your AWS cluster
3. Disable any optional add-on services to reduce cost
4. Add at least **1 worker node** using a **Shared CPU** plan
5. Once the cluster is active, download the **kubeconfig** file

### Step 2 — Switch kubectl Context to Linode

```sh
# Mac / Linux
export KUBECONFIG=/path/to/lke-config.yaml

# Windows PowerShell
$env:KUBECONFIG="/path/to/lke-config.yaml"

# Confirm the active context
kubectl config current-context
```

### Step 3 — Install Consul on LKE

```sh
# Verify worker nodes are ready
kubectl get nodes

# Install Consul on the LKE cluster
helm install lke hashicorp/consul --version 1.0.0 \
  --values consul-values.yaml \
  --set global.datacenter=lke
```

### Step 4 — Deploy Microservices on LKE

```sh
# Navigate to the kubernetes directory
cd kubernetes/

# Deploy with Consul sidecar injection
kubectl apply -f config-consul.yaml
```

---

## Setting Up the Service Mesh

### Enable Mesh Gateway on Both Clusters

Run the following on **both** the AWS EKS and Linode LKE clusters:

```sh
kubectl apply -f consul-mesh-gateway.yaml
```

Validate by running:
```sh
kubectl get mesh
```

You should see 1 mesh entry per cluster.

---

## Setting Up Consul Cluster Peering

1. Get the Consul UI address on each cluster:
```sh
kubectl get services
# Find the consul-ui LoadBalancer service and note the external IP
```

2. Open the Consul UI in your browser using **HTTPS** and the LoadBalancer IP

3. On the **AWS EKS** Consul UI:
   - Navigate to the **Peers** tab
   - Click **Add peer connection**
   - Name it `lke`
   - Copy the generated peering token

4. On the **Linode LKE** Consul UI:
   - Navigate to the **Peers** tab
   - Click **Establish peering**
   - Name it `eks`
   - Paste the token from the EKS Consul UI

Once successful, the peering status will change to **Active**.

> **Note:** The peer names `eks` and `lke` must match what is configured in `service-resolver.yaml` and `exported-service.yaml`. If you use different names, update those files accordingly.

---

## Configuring Service Failover

On the **AWS EKS** cluster:
```sh
kubectl apply -f service-resolver.yaml
```

On the **Linode LKE** cluster:
```sh
kubectl apply -f exported-service.yaml
```

If configured correctly, the Consul peer connection will show **1 exported service** and **1 imported service**.

### Testing Failover

```sh
# Delete the shippingservice on EKS to simulate failure
kubectl delete deployment shippingservice

# The shopping cart page should still be accessible
# Consul automatically routes shipping traffic to the LKE cluster
```

---

## Cleanup

```sh
kubectl delete -f config-consul.yaml
helm uninstall eks -n default
cd terraform/
terraform destroy -var-file terraform.tfvars -auto-approve
```

