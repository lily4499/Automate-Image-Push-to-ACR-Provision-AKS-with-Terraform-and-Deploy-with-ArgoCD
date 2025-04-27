# Deploy AKS Cluster with Terraform + Push Image to ACR + Deploy App using ArgoCD

Of course!  
Here’s your **full content** formatted properly in **Markdown** for a **GitHub README.md** file:  

```markdown
# 🚀 Automate Image Push to ACR, Provision AKS with Terraform, and Deploy with ArgoCD CLI

---

## 📂 Project Structure
```bash
automate-aks-argocd/
├── terraform/
│   ├── providers.tf
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── backend.tf (optional if remote backend)
├── scripts/
│   ├── create-remote-backend.sh
├── Jenkinsfile (for ACR push)
├── README.md
```

---

# 🧠 Step-by-Step Guide

---

## 1️⃣ Authenticate Terraform with Azure

**✅ Purpose:** Let Terraform talk to Azure.

### Method 1: Interactive Login (Good for local testing)
```bash
az login
```
➡️ Terraform will automatically use this authentication.

---

### Method 2: Service Principal (Good for CI/CD, Jenkins)

## setup-sp.sh

```bash
#!/bin/bash

# Set variables
SP_NAME="aks-acr-jenkins-sp"
ACR_NAME="<your-acr-name>"
CREDENTIAL_FILE="sp_credentials.txt"
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

echo "[*] Creating Service Principal..."

# Create SP with Contributor Role and capture output
SP_OUTPUT=$(az ad sp create-for-rbac \
  --name $SP_NAME \
  --role Contributor \
  --scopes /subscriptions/$SUBSCRIPTION_ID \
  --sdk-auth)

# Extract values
APP_ID=$(echo $SP_OUTPUT | jq -r '.clientId')
PASSWORD=$(echo $SP_OUTPUT | jq -r '.clientSecret')
TENANT_ID=$(echo $SP_OUTPUT | jq -r '.tenantId')

# Get ACR Resource ID
ACR_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

echo "[*] Assigning 'acrpush' role on ACR..."

# Assign acrpush role
az role assignment create --assignee $APP_ID --role acrpush --scope $ACR_ID

echo "[*] Saving credentials to $CREDENTIAL_FILE..."

# Save credentials into a file
cat <<EOF > $CREDENTIAL_FILE
# Service Principal Credentials
ARM_CLIENT_ID=$APP_ID
ARM_CLIENT_SECRET=$PASSWORD
ARM_SUBSCRIPTION_ID=$SUBSCRIPTION_ID
ARM_TENANT_ID=$TENANT_ID
EOF

echo "✅ Service Principal Created, Configured, and Credentials Saved to $CREDENTIAL_FILE"


```
---

## Or setup manually  

```bash
az login

# Get your subscription ID
az account show --query id -o tsv

# Create Service Principal
az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/<your-subscription-id>"
```
➡️ Copy `appId`, `password`, and `tenant`.  
➡️ Export them:
```bash
export ARM_CLIENT_ID="<appId>"
export ARM_CLIENT_SECRET="<password>"
export ARM_SUBSCRIPTION_ID="<subscription_id>"
export ARM_TENANT_ID="<tenant_id>"
```
---

## Save the Service Principal into Jenkins Credentials
Go to Jenkins ➔ Manage Credentials and add:
 - Kind: "Username with password"
 - Username: appId (client ID)
 - Password: password (client secret)
 - ID: acr-sp-credentials
 - Description: "Service Principal for ACR login"
✅ Set ID = acr-sp-credentials  
(you will use it inside Jenkinsfile)  

---

## 2️⃣ Setup Terraform Templates

Inside `terraform/` folder:

### `providers.tf`
```hcl
provider "azurerm" {
  features {}
}

terraform {
  backend "azurerm" { }
}
```
➡️ This connects Terraform to Azure.

---

### `variables.tf`
```hcl
variable "resource_group_name" {}
variable "aks_cluster_name" {}
variable "location" {}
variable "acr_name" {}
```
➡️ Defines parameters to reuse in files.

---

### `main.tf`
```hcl
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = var.aks_cluster_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "aksdemo"

  default_node_pool {
    name       = "default"
    node_count = 2
    vm_size    = "Standard_DS2_v2"
  }

  identity {
    type = "SystemAssigned"
  }
}

resource "azurerm_container_registry" "acr" {
  name                = var.acr_name
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  sku                 = "Basic"
  admin_enabled       = true
}
```
➡️ Creates:
- Resource Group
- AKS Cluster
- ACR (Container Registry)

---

### `outputs.tf`
```hcl
output "kube_config" {
  value = azurerm_kubernetes_cluster.aks.kube_config_raw
  sensitive = true
}

output "acr_login_server" {
  value = azurerm_container_registry.acr.login_server
}
```
➡️ Outputs cluster and registry info.

---

## 3️⃣ Setup Remote Backend (Terraform State Management)

Create a script inside `scripts/create-remote-backend.sh`:

```bash
#!/bin/bash

RESOURCE_GROUP_NAME="tfstate"
STORAGE_ACCOUNT_NAME="tfstate$RANDOM"
CONTAINER_NAME="tfstate"

az group create --name $RESOURCE_GROUP_NAME --location eastus

az storage account create --resource-group $RESOURCE_GROUP_NAME \
    --name $STORAGE_ACCOUNT_NAME --sku Standard_LRS --encryption-services blob

az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME
```

➡️ Run it:
```bash
chmod +x scripts/create-remote-backend.sh
./scripts/create-remote-backend.sh
```

---

**Add backend in `providers.tf` or `backend.tf`:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate"
    storage_account_name = "<your_storage_account_name>"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"
  }
}
```

➡️ Then:
```bash
terraform init
terraform apply
```

---

## 4️⃣ Jenkinsfile to Push Docker Image to ACR

In the root folder, `Jenkinsfile`:

```groovy
pipeline {
    agent any
    environment {
        REGISTRY = "myregistry.azurecr.io"
        IMAGE_NAME = "nodejs-webapp"
        TENANT_ID = "<your-tenant-id>"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $REGISTRY/$IMAGE_NAME:latest .'
            }
        }
        stage('Login to ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'acr-sp-credentials', usernameVariable: 'APP_ID', passwordVariable: 'APP_SECRET')]) {
                    sh '''
                    az login --service-principal -u $APP_ID -p $APP_SECRET --tenant $TENANT_ID
                    az acr login --name ${REGISTRY%%.azurecr.io}
                    '''
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                sh 'docker push $REGISTRY/$IMAGE_NAME:latest'
            }
        }
    }
}

```
✅ This Jenkinsfile:  
Logs into Azure using the Service Principal  
Then logs into ACR  
Then pushes the image  


➡️ Make sure ACR credentials are available to Jenkins.

---

## 5️⃣ Connect to AKS Cluster

```bash
az aks get-credentials --resource-group <your-resource-group> --name <your-aks-cluster> --overwrite-existing
```
➡️ Now your `kubectl` will point to AKS!

---

## 6️⃣ Install ArgoCD via CLI

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## 7️⃣ Access ArgoCD Dashboard

Get initial password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Port-forward:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
➡️ Access: `https://127.0.0.1:8080`

---

## 8️⃣ ArgoCD CLI Authentication

```bash
argocd login 127.0.0.1:8080
```
- Username: `admin`
- Password: (from previous step)

---

## 9️⃣ Create ArgoCD App via CLI

```bash
argocd app create my-app \
--repo https://github.com/lily4499/argocd-amazon-manifest.git \
--path ./ \
--dest-server https://kubernetes.default.svc \
--dest-namespace default \
--sync-policy automatic
```

➡️ This will **auto-deploy** your app.

---

## 🔄 Useful ArgoCD CLI Commands

| Command | Purpose |
|:--------|:--------|
| `argocd app list` | List all apps |
| `argocd app get my-app` | Get app details |
| `argocd app sync my-app` | Force sync app |
| `argocd app rollback my-app` | Rollback to previous version |
| `argocd app delete my-app` | Delete app |

---

# 🧹 Cleanup (Optional)

To delete storage group (Terraform backend):
```bash
az group delete --resource-group tfstate
```

---

# 📈 Overall Flow Summary

1. **Terraform** → creates **AKS** and **ACR**.
2. **Jenkins** → builds image, pushes to **ACR**.
3. **ArgoCD CLI** → deploys app automatically on **AKS** from GitHub repo.

---

# 📋 Final Checklist

| Task | Status |
|:----|:-------|
| Terraform templates created | ✅ |
| ACR push working | ✅ |
| AKS credentials set up | ✅ |
| ArgoCD installed | ✅ |
| ArgoCD apps deployed | ✅ |

---

# 🌟 That's a full breakdown!

---
```

---

✅ **This Markdown is ready to paste directly into your GitHub `README.md` file**!  
✅ **It will render perfectly on GitHub with code blocks, tables, and structure.**

---

Would you like me to also create a **flowchart** diagram for you (simple `.png` or `.svg`) showing the steps visually (Terraform ➔ ACR ➔ Jenkins ➔ AKS ➔ ArgoCD ➔ GitHub)? 🎯   
It would make your README even more professional! 🚀  
Would you like me to create it?
