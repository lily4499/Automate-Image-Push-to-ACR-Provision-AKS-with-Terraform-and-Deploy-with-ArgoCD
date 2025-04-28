
```markdown
# 🚀 Automate Image Push to ACR, Provision AKS with Terraform, and Deploy with ArgoCD CLI

---

### 📖 Project Background (Case Study)

> **A growing SaaS startup specializing in task management solutions needed a fully automated, scalable, and secure infrastructure to deploy their Node.js web application. To achieve this, we provisioned an Azure Kubernetes Service (AKS) cluster and Azure Container Registry (ACR) using Terraform for infrastructure as code. Jenkins was integrated to automate the build and push of Docker images to ACR, while ArgoCD was implemented to enable GitOps-based continuous deployment to AKS, directly syncing from the application’s GitHub repository. This architecture allowed the company to ship new features faster, achieve zero-downtime deployments, and scale seamlessly as their user base expanded from a few hundred to several thousand users globally — without changing the underlying deployment process.

---

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

## file-setup.py

```python
import os

# Define the base directory
base_dir = "/home/lilia/VIDEOS/automate-aks-argocd"

# Define file structures and contents
files = {
    "terraform/providers.tf": '''
provider "azurerm" {
  features {}
}

terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate"
    storage_account_name = "<your-storage-account-name>"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"
  }
}
''',

    "terraform/main.tf": '''
resource "azurerm_resource_group" "aks_rg" {
  name     = var.resource_group_name
  location = var.location
}

resource "azurerm_kubernetes_cluster" "aks_cluster" {
  name                = var.cluster_name
  location            = azurerm_resource_group.aks_rg.location
  resource_group_name = azurerm_resource_group.aks_rg.name
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
''',

    "terraform/variables.tf": '''
variable "resource_group_name" {
  default = "my-rg"
}

variable "location" {
  default = "eastus"
}

variable "cluster_name" {
  default = "lili-aks"
}
''',

    "terraform/outputs.tf": '''
output "kube_config" {
  value     = azurerm_kubernetes_cluster.aks_cluster.kube_config_raw
  sensitive = true
}
''',

    "terraform/backend.tf": '''
terraform {
  backend "azurerm" {}
}
''',

    "scripts/create-remote-backend.sh": '''
#!/bin/bash

RESOURCE_GROUP_NAME=tfstate
STORAGE_ACCOUNT_NAME=tfstate$RANDOM
CONTAINER_NAME=tfstate

# Create Resource Group
az group create --name $RESOURCE_GROUP_NAME --location eastus

# Create Storage Account
az storage account create --resource-group $RESOURCE_GROUP_NAME --name $STORAGE_ACCOUNT_NAME --sku Standard_LRS --encryption-services blob

# Create Blob Container
az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME
''',

    "Jenkinsfile": '''
pipeline {
    agent any
    environment {
        REGISTRY = "<your-acr-name>.azurecr.io"
        IMAGE_NAME = "nodejs-webapp"
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
                withCredentials([usernamePassword(credentialsId: 'acr-sp-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    sh '''
                    az login --service-principal -u $USERNAME -p $PASSWORD --tenant <your-tenant-id>
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
'''
}

# Function to create files
def create_files(base_dir, files):
    for relative_path, content in files.items():
        full_path = os.path.join(base_dir, relative_path)
        os.makedirs(os.path.dirname(full_path), exist_ok=True)
        with open(full_path, 'w') as f:
            f.write(content)
        print(f"Created: {full_path}")

# Run the function
create_files(base_dir, files)


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
✅ Now Terraform will automatically authenticate via this Service Principal.
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

You need a Service Principal that has acrpull and acrpush permissions on your Azure Container Registry.  
## Give Service Principal acrpush on ACR

```bash
ACR_ID=$(az acr show --name <acr-name> --query id --output tsv)
az role assignment create \
  --assignee <appId> \
  --role acrpush \
  --scope $ACR_ID
```
✅ This makes sure your SP can docker login into ACR properly from Jenkins.
✅ Jenkins will also login to Azure (via CLI) with this SP inside pipeline.

---

➡️ Make sure ACR credentials are available to Jenkins.

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

