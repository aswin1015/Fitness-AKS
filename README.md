# Fitness Tracker Infrastructure Deployment Guide (Azure Portal)

## Architecture Overview

This deployment creates:

* Resource Group
* Virtual Network with dedicated subnets
* Azure Key Vault
* Azure Cosmos DB (Mongo API) with Private Endpoint
* Private AKS Cluster
* User Assigned Managed Identity
* Azure Monitor Container Insights
* Azure Managed Prometheus
* Azure Managed Grafana
* Workload Identity Integration
* Key Vault CSI Driver
* Federated Credential Authentication
* Application Deployment
* KGateway Installation

---

# Step 1: Create Resource Group

Create a Resource Group:

| Setting             | Value                 |
| ------------------- | --------------------- |
| Resource Group Name | Fitness-RG            |
| Region              | East US (Recommended) |

---

# Step 2: Create Virtual Network

Create a Virtual Network:

| Setting       | Value       |
| ------------- | ----------- |
| Name          | Fit-VNet    |
| Address Space | 10.0.0.0/16 |

## Create Subnets

### AKS Subnet

| Property      | Value       |
| ------------- | ----------- |
| Name          | aks-subnet  |
| Address Range | 10.0.1.0/24 |
| Available IPs | 256         |

### Cosmos DB Subnet

| Property      | Value         |
| ------------- | ------------- |
| Name          | Cosmos-Subnet |
| Address Range | 10.0.2.0/24   |
| Available IPs | 256           |

### Azure Bastion Subnet

> Azure requires the subnet name to be exactly **AzureBastionSubnet**

| Property      | Value              |
| ------------- | ------------------ |
| Name          | AzureBastionSubnet |
| Address Range | 10.0.3.0/26        |
| Available IPs | 64                 |

---

# Step 3: Create Azure Key Vault

Create a Key Vault:

| Setting      | Value                  |
| ------------ | ---------------------- |
| Name         | ara-key                |
| Region       | Same as Resource Group |
| Pricing Tier | Standard               |

---

# Step 4: Create Azure Cosmos DB

Create Azure Cosmos DB Account.

| Setting      | Value                       |
| ------------ | --------------------------- |
| Account Name | fitcosmos                   |
| API          | Azure Cosmos DB for MongoDB |

## Networking Configuration

### Connectivity Method

Select:

* Private Endpoint

### Configure Firewall

Enable:

* Allow Azure services and resources to access this account

### Create Private Endpoint

| Setting               | Value         |
| --------------------- | ------------- |
| Private Endpoint Name | fitmongope    |
| Virtual Network       | Fit-VNet      |
| Subnet                | Cosmos-Subnet |

After deployment verify:

```text
Cosmos DB
 └── Networking
      └── Private Endpoint Connection = Approved
```

---

# Step 5: Create User Assigned Managed Identity

Create:

| Setting | Value        |
| ------- | ------------ |
| Name    | Fit-AKS-uami |

After creation save:

* Client ID
* Principal ID
* Resource ID

These values will be used later.

---

# Step 6: Create AKS Cluster

Create AKS Cluster:

| Setting        | Value            |
| -------------- | ---------------- |
| Name           | Fit-AKS          |
| Resource Group | Fitness-RG       |
| Node Size      | Standard D4ds v4 |

## Networking Configuration

Select:

* Bring your own virtual network

Choose:

| Setting | Value      |
| ------- | ---------- |
| VNet    | Fit-VNet   |
| Subnet  | aks-subnet |

Enable:

* Private Cluster

## Kubernetes Networking

| Setting        | Value       |
| -------------- | ----------- |
| Service CIDR   | 11.0.0.0/16 |
| DNS Service IP | 11.0.0.10   |

## Identity

Select:

* User Assigned Managed Identity

Choose:

* Fit-AKS-uami

## Monitoring

Enable:

| Feature            | Status  |
| ------------------ | ------- |
| Container Insights | Enabled |
| Managed Prometheus | Enabled |

Create:

| Resource                | Name                 |
| ----------------------- | -------------------- |
| Azure Monitor Workspace | prometheus-workspace |
| Azure Managed Grafana   | fitness-grafana      |

## Security

Enable:

| Feature                 | Status  |
| ----------------------- | ------- |
| OIDC Issuer             | Enabled |
| Workload Identity       | Enabled |
| Secret Store CSI Driver | Enabled |

Create the cluster.

---

# Step 7: Grant AKS Access to Key Vault

Navigate:

```text
ara-key
 └── Access Control (IAM)
      └── Add Role Assignment
```

Assign role:

| Role                   | Principal    |
| ---------------------- | ------------ |
| Key Vault Secrets User | Fit-AKS-uami |

---

# Step 8: Create Federated Credential

Navigate:

```text
Fit-AKS-uami
 └── Federated Credentials
      └── Add Credential
```

## Configuration

| Setting         | Value                                |
| --------------- | ------------------------------------ |
| Scenario        | Kubernetes accessing Azure resources |
| Name            | fitness-app-fed-cred                 |
| Namespace       | default                              |
| Service Account | fitness-app-sa                       |

### OIDC Issuer URL

Retrieve from:

```text
AKS Cluster
 └── Settings
      └── Security Configuration
           └── Issuer URL
```

Example:

```text
https://eastus.oic.prod-aks.azure.com/3076b773-f5d9-413e-9564-7deaa3d0d7b7/92f2e837-b880-4f97-96cb-48eec3515bdf/
```

Click **Add**.

---

# Step 9: Assign Yourself Key Vault Permissions

Navigate:

```text
ara-key
 └── Access Control (IAM)
```

Assign role:

| Role                      |
| ------------------------- |
| Key Vault Secrets Officer |

Assign to:

* Your Azure User Account

---

# Step 10: Create Secrets in Key Vault

Navigate:

```text
ara-key
 └── Secrets
```

Create:

## CosmosDbEndpointEndpoint

| Property | Value                           |
| -------- | ------------------------------- |
| Name     | CosmosDbEndpointEndpoint                |
| Value    | `<Cosmos DB Connection String>` |

## JwtSecret

| Property | Value                           |
| -------- | ------------------------------- |
| Name     | JwtSecret                       |
| Value    | `my-super-secret-string-12345!` |

---

# Step 11: Update Kubernetes Manifests

## File: 02-service-account.yaml

Replace:

```yaml
<YOUR_MANAGED_IDENTITY_CLIENT_ID>
```

with:

```yaml
<Client ID of Fit-AKS-uami>
```

## File: 03-secret-provider-class.yaml

Replace:

```yaml
<YOUR_MANAGED_IDENTITY_CLIENT_ID>
```

with:

```yaml
<Client ID of Fit-AKS-uami>
```

Replace:

```yaml
<YOUR_KEY_VAULT_NAME>
```

with:

```yaml
ara-key
```

Replace:

```yaml
<YOUR_AZURE_TENANT_ID>
```

with your Microsoft Entra Tenant ID.

Retrieve from:

```text
Microsoft Entra ID
 └── Overview
      └── Tenant ID
```

### Update Secret Objects

```yaml
objects: |
  array:
    - |
      objectName: CosmosDbEndpointEndpoint
      objectType: secret
      objectVersion: ""
    - |
      objectName: JwtSecret
      objectType: secret
      objectVersion: ""
```

---

# Step 12: Create Management VM

Create a Linux VM in Azure.

Recommended:

| Setting | Value            |
| ------- | ---------------- |
| OS      | Ubuntu 22.04 LTS |
| Size    | Standard B2s     |

---

# Step 13: Install Required Tools

SSH into VM.

## Install Azure CLI

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

## Install kubectl

```bash
sudo snap install kubectl --classic
```

Verify:

```bash
az version
kubectl version --client
```

---

# Step 14: Connect to AKS

Login:

```bash
az login
```

Get AKS Credentials:

```bash
az aks get-credentials \
  --resource-group Fitness-RG \
  --name Fit-AKS
```

Verify:

```bash
kubectl get nodes
```

---

# Step 15: Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

Verify:

```bash
kubectl get crds | grep gateway
```

---

# Step 16: Install KGateway CRDs

```bash
helm upgrade -i kgateway-crds \
oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds \
--create-namespace \
--namespace kgateway-system \
--version v2.3.1
```

---

# Step 17: Install KGateway Controller

```bash
helm upgrade -i kgateway \
oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
--namespace kgateway-system \
--version v2.3.1
```

Verify:

```bash
kubectl get pods -n kgateway-system
```

---

# Step 18: Deploy Fitness Tracker Application

Clone Repository:

```bash
git clone https://github.com/aravindroy1/Fitness_Tracker.git

cd Fitness_Tracker
```

Deploy:

```bash
kubectl apply -f azure_deploy/
```

---

# Step 19: Verify Gateway

Check Gateway:

```bash
kubectl get gateway main-gateway
```

Verify:

```bash
kubectl get pods
kubectl get svc
kubectl get gateway
kubectl get httproute
```

---

# Validation Checklist

## Azure Resources

* [ ] Fitness-RG created
* [ ] Fit-VNet created
* [ ] aks-subnet created
* [ ] Cosmos-Subnet created
* [ ] AzureBastionSubnet created
* [ ] ara-key created
* [ ] fitcosmos created
* [ ] fitmongope approved
* [ ] Fit-AKS-uami created
* [ ] Fit-AKS created
* [ ] prometheus-workspace created
* [ ] fitness-grafana created

## Security

* [ ] OIDC Enabled
* [ ] Workload Identity Enabled
* [ ] Secret Store CSI Driver Enabled
* [ ] Federated Credential Created
* [ ] Key Vault Secrets User assigned
* [ ] Key Vault Secrets Officer assigned

## Application

* [ ] CosmosDbEndpointEndpoint secret created
* [ ] JwtSecret created
* [ ] Kubernetes manifests updated
* [ ] AKS connected
* [ ] KGateway installed
* [ ] Fitness Tracker deployed

---

# Explanation of Each Component

## Resource Group

Logical container that holds all Azure resources for the Fitness Tracker application.

## Virtual Network

Provides network isolation and communication between AKS, Cosmos DB, and other Azure services.

## AKS Subnet

Dedicated subnet where AKS nodes and pods are deployed.

## Cosmos Subnet

Dedicated subnet used for Cosmos DB Private Endpoint to keep database traffic inside Azure private networking.

## Azure Bastion Subnet

Reserved subnet required if Azure Bastion is added later for secure VM access without public IPs.

## Key Vault

Stores sensitive secrets such as:

* Cosmos DB connection string
* JWT signing secret
* Future certificates and credentials

## Cosmos DB

MongoDB-compatible NoSQL database used by the Fitness Tracker application.

Private Endpoint ensures traffic never traverses the public internet.

## User Assigned Managed Identity

Provides a secure Azure identity to workloads without storing credentials.

## AKS Private Cluster

Kubernetes API server is accessible only from private networks, improving security.

## OIDC + Workload Identity

Allows Kubernetes Service Accounts to securely authenticate to Azure resources using federated identity instead of secrets.

## Secret Store CSI Driver

Mounts Key Vault secrets directly into Kubernetes Pods.

### Benefits

* No secrets stored in Git
* No Kubernetes Secret synchronization required
* Centralized secret management

## Federated Credential

Creates trust between:

```text
AKS Service Account
        ↓
Workload Identity
        ↓
Managed Identity
        ↓
Azure Key Vault
```

This enables pods to access Key Vault without passwords or connection secrets.

## Azure Managed Prometheus

Collects Kubernetes and application metrics.

Examples:

* CPU
* Memory
* Pod metrics
* Custom metrics

## Azure Managed Grafana

Visualization layer for Prometheus metrics.

Used for dashboards and monitoring.

## KGateway

Gateway API implementation providing:

* Ingress routing
* HTTP routing
* Traffic management
* API gateway capabilities

## Fitness Tracker Application Flow

```text
User
  ↓
KGateway
  ↓
Fitness Tracker Pods
  ↓
Key Vault (via Workload Identity)
  ↓
Cosmos DB (Private Endpoint)
```

This architecture follows Azure security best practices by using private networking, managed identities, workload identity federation, and centralized secret management.
