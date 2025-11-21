# Lab 10.D: GitHub Actions Terraform Automation

## Objectives
- Create Terraform infrastructure code
- Configure GitHub Actions for Terraform
- Implement automated plan and apply
- Use remote state storage
- Implement approval workflow
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- GitHub account for repository
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab10d-tfstate"
STORAGE_ACCOUNT="tfstate$RANDOM$RANDOM"
CONTAINER_NAME="tfstate"
REPO_NAME="github-terraform-automation"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$STORAGE_ACCOUNT"
```

---

## Step 2 – Create Resource Group for State

```bash
# Create resource group for Terraform state
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Storage for State

```bash
# Create storage account
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --encryption-services blob

# Create container for state
az storage container create \
  --name "$CONTAINER_NAME" \
  --account-name "$STORAGE_ACCOUNT" \
  --auth-mode login

echo "Terraform state storage created"
```

---

## Step 4 – Get Storage Account Key

```bash
# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query '[0].value' \
  --output tsv)

echo "Storage key retrieved"
```

---

## Step 5 – Create Service Principal

```bash
# Get subscription ID
SUBSCRIPTION_ID=$(az account show --query id --output tsv)

echo "$SUBSCRIPTION_ID"

# Create service principal
SP_OUTPUT=$(az ad sp create-for-rbac \
  --name "sp-github-terraform" \
  --role Contributor \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --output json)

# Extract credentials
CLIENT_ID=$(echo "$SP_OUTPUT" | python3 -c "import sys, json; print(json.load(sys.stdin)['appId'])")
CLIENT_SECRET=$(echo "$SP_OUTPUT" | python3 -c "import sys, json; print(json.load(sys.stdin)['password'])")
TENANT_ID=$(echo "$SP_OUTPUT" | python3 -c "import sys, json; print(json.load(sys.stdin)['tenant'])")

echo "Service principal created"
```

---

## Step 6 – Create Application Directory

```bash
# Create application directory
mkdir -p "$REPO_NAME/terraform"
cd "$REPO_NAME"

echo "Application directory created"
```

---

## Step 7 – Create Terraform Configuration

```bash
# Create provider configuration
cat > terraform/provider.tf << EOF
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  
  backend "azurerm" {
    resource_group_name  = "$RG_NAME"
    storage_account_name = "$STORAGE_ACCOUNT"
    container_name       = "$CONTAINER_NAME"
    key                  = "terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
}
EOF

# Create variables
cat > terraform/variables.tf << 'EOF'
variable "location" {
  description = "Azure region"
  type        = string
  default     = "australiaeast"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "resource_group_name" {
  description = "Resource group name"
  type        = string
  default     = "rg-terraform-automation"
}
EOF

# Create main infrastructure
cat > terraform/main.tf << 'EOF'
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
  
  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Automation  = "GitHub Actions"
  }
}

resource "azurerm_storage_account" "app" {
  name                     = "st${var.environment}${random_string.suffix.result}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  min_tls_version          = "TLS1_2"
  
  tags = {
    Environment = var.environment
  }
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]
  
  tags = {
    Environment = var.environment
  }
}

resource "azurerm_subnet" "app" {
  name                 = "subnet-app"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}
EOF

# Create outputs
cat > terraform/outputs.tf << 'EOF'
output "resource_group_name" {
  description = "Resource group name"
  value       = azurerm_resource_group.main.name
}

output "storage_account_name" {
  description = "Storage account name"
  value       = azurerm_storage_account.app.name
}

output "vnet_id" {
  description = "Virtual network ID"
  value       = azurerm_virtual_network.main.id
}
EOF

echo "Terraform configuration created"
```

---

## Step 8 – Create GitHub Actions Workflow

```bash
# Create workflow directory
mkdir -p .github/workflows

# Create Terraform workflow
cat > .github/workflows/terraform.yml << 'EOF'
name: Terraform Automation

on:
  push:
    branches: [main]
    paths:
      - 'terraform/**'
  pull_request:
    branches: [main]
    paths:
      - 'terraform/**'
  workflow_dispatch:

env:
  TF_VERSION: '1.6.6'
  WORKING_DIR: './terraform'

jobs:
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Terraform Init
        working-directory: ${{ env.WORKING_DIR }}
        run: |
          terraform init \
            -backend-config="access_key=${{ secrets.TERRAFORM_STATE_KEY }}"

      - name: Terraform Format Check
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform fmt -check

      - name: Terraform Validate
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform validate

      - name: Terraform Plan
        working-directory: ${{ env.WORKING_DIR }}
        run: |
          terraform plan -out=tfplan -input=false
        env:
          ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}

      - name: Upload Plan
        uses: actions/upload-artifact@v3
        with:
          name: tfplan
          path: ${{ env.WORKING_DIR }}/tfplan

  terraform-apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: tfplan
          path: ${{ env.WORKING_DIR }}

      - name: Terraform Init
        working-directory: ${{ env.WORKING_DIR }}
        run: |
          terraform init \
            -backend-config="access_key=${{ secrets.TERRAFORM_STATE_KEY }}"

      - name: Terraform Apply
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform apply -auto-approve tfplan
        env:
          ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}

      - name: Terraform Output
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform output -json
EOF

echo "GitHub Actions workflow created"
```

---

## Step 9 – Initialize Git Repository

```bash
# Initialize git
git init
git add .
git commit -m "Initial commit with Terraform automation"

echo "Git repository initialized"
```

---

## Step 10 – Create GitHub Repository

```bash
# Set GitHub username
read -p "Enter your GitHub username: " GITHUB_USERNAME

echo "$GITHUB_USERNAME"

# Create GitHub repository
if command -v gh &> /dev/null; then
  gh repo create "$REPO_NAME" --public --source=. --remote=origin --push
  echo "GitHub repository created"
else
  echo "GitHub CLI not installed. Create repository manually at https://github.com/new"
  echo "Repository name: $REPO_NAME"
  echo "Then run: git remote add origin https://github.com/$GITHUB_USERNAME/$REPO_NAME.git"
  echo "Then run: git push -u origin main"
fi
```

---

## Step 11 – Configure GitHub Secrets

```bash
# Create Azure credentials JSON
AZURE_CREDENTIALS=$(cat <<EOF_CREDS
{
  "clientId": "$CLIENT_ID",
  "clientSecret": "$CLIENT_SECRET",
  "subscriptionId": "$SUBSCRIPTION_ID",
  "tenantId": "$TENANT_ID"
}
EOF_CREDS
)

echo "Configure the following GitHub Secrets:"
echo ""
echo "1. Go to: https://github.com/$GITHUB_USERNAME/$REPO_NAME/settings/secrets/actions"
echo "2. Click: New repository secret"
echo ""
echo "Secret 1 - AZURE_CREDENTIALS:"
echo "$AZURE_CREDENTIALS"
echo ""
echo "Secret 2 - ARM_CLIENT_ID:"
echo "$CLIENT_ID"
echo ""
echo "Secret 3 - ARM_CLIENT_SECRET:"
echo "$CLIENT_SECRET"
echo ""
echo "Secret 4 - ARM_SUBSCRIPTION_ID:"
echo "$SUBSCRIPTION_ID"
echo ""
echo "Secret 5 - ARM_TENANT_ID:"
echo "$TENANT_ID"
echo ""
echo "Secret 6 - TERRAFORM_STATE_KEY:"
echo "$STORAGE_KEY"
echo ""
read -p "Press Enter after configuring secrets..."
```

---

## Step 12 – Create GitHub Environment

```bash
echo "Create GitHub Environment:"
echo ""
echo "1. Go to: https://github.com/$GITHUB_USERNAME/$REPO_NAME/settings/environments"
echo "2. Click: New environment"
echo "3. Name: production"
echo "4. Enable: Required reviewers (optional)"
echo "5. Click: Configure environment"
echo ""
read -p "Press Enter after creating environment..."
```

---

## Step 13 – Trigger Workflow

```bash
# Trigger workflow
echo "# Terraform Automation" > README.md
git add README.md
git commit -m "Trigger Terraform workflow"
git push origin main

echo "Workflow triggered"
echo "Monitor at: https://github.com/$GITHUB_USERNAME/$REPO_NAME/actions"
```

---

## Step 14 – Monitor Workflow

```bash
echo "Waiting for workflow to complete..."
sleep 120

echo "Check workflow status at:"
echo "https://github.com/$GITHUB_USERNAME/$REPO_NAME/actions"
```

---

## Step 15 – Verify Infrastructure

```bash
# List resources in created resource group
az resource list \
  --resource-group "rg-terraform-automation" \
  --query "[].{Name:name, Type:type}" \
  --output table
```

---

## Step 16 – Cleanup

```bash
# Delete service principal
az ad sp delete --id "$CLIENT_ID"

# Delete Terraform-managed resources
az group delete \
  --name "rg-terraform-automation" \
  --yes \
  --no-wait

# Delete state storage
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local repository
cd ..
rm -rf "$REPO_NAME"

echo "Cleanup complete"
echo "Manually delete GitHub repository if needed"
```

---

## Summary

You created Terraform infrastructure code with remote state storage, configured GitHub Actions workflow for automated Terraform plan and apply, implemented approval workflow with GitHub environments, and automated infrastructure deployment with version control integration.
