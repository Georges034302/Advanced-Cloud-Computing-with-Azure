# Lab 9.D: Terraform Azure Infrastructure

## Objectives
- Install Terraform CLI
- Create Terraform configuration files
- Deploy Azure infrastructure with Terraform
- Manage state file
- Update and destroy infrastructure
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab9d-terraform"
TERRAFORM_DIR="terraform-azure"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$TERRAFORM_DIR"
```

---

## Step 2 – Install Terraform

```bash
# Download and install Terraform
wget -O terraform.zip https://releases.hashicorp.com/terraform/1.6.6/terraform_1.6.6_linux_amd64.zip

# Unzip Terraform binary
unzip -o terraform.zip

# Move to system path
sudo mv terraform /usr/local/bin/

# Verify installation
terraform version
```

---

## Step 3 – Authenticate Azure

```bash
# Login to Azure
az login

# Set default subscription
az account set --subscription "$(az account show --query id -o tsv)"

# Show current subscription
az account show --query "{Name:name, ID:id}" --output table
```

---

## Step 4 – Create Terraform Directory

```bash
# Create directory for Terraform files
mkdir -p "$TERRAFORM_DIR"
cd "$TERRAFORM_DIR"

echo "Terraform directory created"
```

---

## Step 5 – Create Provider Configuration

```bash
# Create provider.tf
cat > provider.tf << 'EOF'
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}
EOF

echo "Provider configuration created"
```

---

## Step 6 – Create Variables Configuration

```bash
# Create variables.tf
cat > variables.tf << 'EOF'
variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "australiaeast"
}

variable "resource_group_name" {
  description = "Resource group name"
  type        = string
  default     = "rg-lab9d-terraform"
}

variable "vnet_address_space" {
  description = "Virtual network address space"
  type        = list(string)
  default     = ["10.0.0.0/16"]
}

variable "subnet_prefixes" {
  description = "Subnet address prefixes"
  type        = map(string)
  default = {
    web  = "10.0.1.0/24"
    app  = "10.0.2.0/24"
    data = "10.0.3.0/24"
  }
}

variable "admin_username" {
  description = "Admin username for VM"
  type        = string
  default     = "azureuser"
}

variable "admin_password" {
  description = "Admin password for VM"
  type        = string
  sensitive   = true
}

variable "vm_size" {
  description = "Virtual machine size"
  type        = string
  default     = "Standard_B2s"
}
EOF

echo "Variables configuration created"
```

---

## Step 7 – Create Network Configuration

```bash
# Create network.tf
cat > network.tf << 'EOF'
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

resource "azurerm_network_security_group" "main" {
  name                = "nsg-terraform"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "AllowHTTP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "AllowSSH"
    priority                   = 120
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-terraform"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = var.vnet_address_space
}

resource "azurerm_subnet" "web" {
  name                 = "subnet-web"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.subnet_prefixes["web"]]
}

resource "azurerm_subnet" "app" {
  name                 = "subnet-app"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.subnet_prefixes["app"]]
}

resource "azurerm_subnet" "data" {
  name                 = "subnet-data"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.subnet_prefixes["data"]]
}

resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.web.id
  network_security_group_id = azurerm_network_security_group.main.id
}
EOF

echo "Network configuration created"
```

---

## Step 8 – Create Storage Configuration

```bash
# Create storage.tf
cat > storage.tf << 'EOF'
resource "azurerm_storage_account" "main" {
  name                     = "sttf${random_string.suffix.result}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  min_tls_version          = "TLS1_2"

  blob_properties {
    delete_retention_policy {
      days = 7
    }
  }
}

resource "azurerm_storage_container" "data" {
  name                  = "data"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "logs" {
  name                  = "logs"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}
EOF

echo "Storage configuration created"
```

---

## Step 9 – Create Compute Configuration

```bash
# Create compute.tf
cat > compute.tf << 'EOF'
resource "azurerm_public_ip" "main" {
  name                = "pip-vm-terraform"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_network_interface" "main" {
  name                = "nic-vm-terraform"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.web.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.main.id
  }
}

resource "azurerm_linux_virtual_machine" "main" {
  name                = "vm-terraform"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  size                = var.vm_size
  admin_username      = var.admin_username
  admin_password      = var.admin_password
  
  disable_password_authentication = false

  network_interface_ids = [
    azurerm_network_interface.main.id,
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
}
EOF

echo "Compute configuration created"
```

---

## Step 10 – Create Outputs Configuration

```bash
# Create outputs.tf
cat > outputs.tf << 'EOF'
output "resource_group_name" {
  description = "Resource group name"
  value       = azurerm_resource_group.main.name
}

output "vnet_id" {
  description = "Virtual network ID"
  value       = azurerm_virtual_network.main.id
}

output "storage_account_name" {
  description = "Storage account name"
  value       = azurerm_storage_account.main.name
}

output "vm_public_ip" {
  description = "Virtual machine public IP"
  value       = azurerm_public_ip.main.ip_address
}

output "vm_id" {
  description = "Virtual machine ID"
  value       = azurerm_linux_virtual_machine.main.id
}
EOF

echo "Outputs configuration created"
```

---

## Step 11 – Initialize Terraform

```bash
# Initialize Terraform
terraform init

echo "Terraform initialized"
```

---

## Step 12 – Validate Configuration

```bash
# Validate Terraform configuration
terraform validate

# Format Terraform files
terraform fmt

echo "Configuration validated"
```

---

## Step 13 – Plan Deployment

```bash
# Read admin password from user
read -s -p "Enter VM admin password: " ADMIN_PASSWORD
echo ""

# Create Terraform plan
terraform plan -var="admin_password=$ADMIN_PASSWORD" -out=tfplan

echo "Terraform plan created"
```

---

## Step 14 – Apply Configuration

```bash
# Apply Terraform configuration
terraform apply tfplan

echo "Infrastructure deployed"
```

---

## Step 15 – Show Outputs

```bash
# Show all outputs
terraform output

# Get specific outputs
RESOURCE_GROUP=$(terraform output -raw resource_group_name)
echo "$RESOURCE_GROUP"

STORAGE_ACCOUNT=$(terraform output -raw storage_account_name)
echo "$STORAGE_ACCOUNT"

VM_PUBLIC_IP=$(terraform output -raw vm_public_ip)
echo "$VM_PUBLIC_IP"
```

---

## Step 16 – Show Terraform State

```bash
# List resources in state
terraform state list

# Show specific resource
terraform state show azurerm_virtual_network.main
```

---

## Step 17 – Validate Deployed Resources

```bash
# List resources in resource group
az resource list \
  --resource-group "$RESOURCE_GROUP" \
  --query "[].{Name:name, Type:type}" \
  --output table

# Show virtual network
az network vnet show \
  --name "vnet-terraform" \
  --resource-group "$RESOURCE_GROUP" \
  --query "{Name:name, AddressSpace:addressSpace.addressPrefixes}" \
  --output table

# Show storage account
az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RESOURCE_GROUP" \
  --query "{Name:name, Sku:sku.name}" \
  --output table
```

---

## Step 18 – Update Infrastructure

```bash
# Update compute.tf to add tags
cat >> compute.tf << 'EOF'

resource "azurerm_resource_group_policy_assignment" "main" {
  name                 = "require-tags"
  resource_group_id    = azurerm_resource_group.main.id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025"
}
EOF

# Plan update
terraform plan -var="admin_password=$ADMIN_PASSWORD" -out=tfplan-update

# Apply update
terraform apply tfplan-update

echo "Infrastructure updated"
```

---

## Step 19 – Generate Dependency Graph

```bash
# Generate graph
terraform graph > graph.dot

# View graph as text
cat graph.dot

echo "Dependency graph generated"
```

---

## Step 20 – Cleanup

```bash
# Destroy all resources
terraform destroy -var="admin_password=$ADMIN_PASSWORD" -auto-approve

# Remove Terraform files
cd ..
rm -rf "$TERRAFORM_DIR"

# Remove Terraform binary
sudo rm -f /usr/local/bin/terraform

echo "Cleanup complete"
```

---

## Summary

You installed Terraform, created configuration files for network, storage, and compute resources, deployed Azure infrastructure using Terraform commands, managed state, validated outputs, updated infrastructure, and destroyed all resources using Terraform lifecycle management.
