# Lab 9.B: Bicep Modular Deployment

## Objectives
- Create modular Bicep templates
- Deploy network infrastructure module
- Deploy storage module
- Deploy compute module
- Link modules together
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
RG_NAME="rg-lab9b-bicep"
DEPLOYMENT_NAME="bicep-modular-deployment"
MODULES_DIR="bicep-modules"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$DEPLOYMENT_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Modules Directory

```bash
# Create directory structure
mkdir -p "$MODULES_DIR"
cd "$MODULES_DIR"

echo "Modules directory created"
```

---

## Step 4 – Create Network Module

```bash
# Create network module
cat > network.bicep << 'EOF'
@description('Location for all resources')
param location string = resourceGroup().location

@description('Virtual network name')
param vnetName string

@description('Virtual network address prefix')
param vnetAddressPrefix string = '10.0.0.0/16'

@description('Subnet configurations')
param subnets array = [
  {
    name: 'subnet-web'
    addressPrefix: '10.0.1.0/24'
  }
  {
    name: 'subnet-app'
    addressPrefix: '10.0.2.0/24'
  }
  {
    name: 'subnet-data'
    addressPrefix: '10.0.3.0/24'
  }
]

resource nsg 'Microsoft.Network/networkSecurityGroups@2023-04-01' = {
  name: '${vnetName}-nsg'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowHTTP'
        properties: {
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '80'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 100
          direction: 'Inbound'
        }
      }
      {
        name: 'AllowHTTPS'
        properties: {
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 110
          direction: 'Inbound'
        }
      }
    ]
  }
}

resource vnet 'Microsoft.Network/virtualNetworks@2023-04-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        vnetAddressPrefix
      ]
    }
    subnets: [for subnet in subnets: {
      name: subnet.name
      properties: {
        addressPrefix: subnet.addressPrefix
        networkSecurityGroup: {
          id: nsg.id
        }
      }
    }]
  }
}

output vnetId string = vnet.id
output vnetName string = vnet.name
output subnetIds array = [for i in range(0, length(subnets)): vnet.properties.subnets[i].id]
output nsgId string = nsg.id
EOF

echo "Network module created"
```

---

## Step 5 – Create Storage Module

```bash
# Create storage module
cat > storage.bicep << 'EOF'
@description('Location for all resources')
param location string = resourceGroup().location

@description('Storage account name prefix')
param storageAccountPrefix string

@description('Storage account SKU')
@allowed([
  'Standard_LRS'
  'Standard_GRS'
  'Standard_ZRS'
])
param storageSku string = 'Standard_LRS'

@description('Container names')
param containerNames array = [
  'data'
  'logs'
  'backups'
]

var storageAccountName = '${storageAccountPrefix}${uniqueString(resourceGroup().id)}'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
  }
}

resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    deleteRetentionPolicy: {
      enabled: true
      days: 7
    }
  }
}

resource containers 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = [for containerName in containerNames: {
  parent: blobService
  name: containerName
  properties: {
    publicAccess: 'None'
  }
}]

output storageAccountId string = storageAccount.id
output storageAccountName string = storageAccount.name
output primaryEndpoints object = storageAccount.properties.primaryEndpoints
EOF

echo "Storage module created"
```

---

## Step 6 – Create Compute Module

```bash
# Create compute module
cat > compute.bicep << 'EOF'
@description('Location for all resources')
param location string = resourceGroup().location

@description('Virtual machine name')
param vmName string

@description('Virtual machine size')
param vmSize string = 'Standard_B2s'

@description('Admin username')
param adminUsername string

@description('Admin password')
@secure()
param adminPassword string

@description('Subnet ID for VM')
param subnetId string

@description('Operating system')
@allowed([
  'Ubuntu2204'
  'Windows2022'
])
param osType string = 'Ubuntu2204'

var nicName = '${vmName}-nic'
var publicIpName = '${vmName}-pip'
var imageReference = {
  Ubuntu2204: {
    publisher: 'Canonical'
    offer: '0001-com-ubuntu-server-jammy'
    sku: '22_04-lts-gen2'
    version: 'latest'
  }
  Windows2022: {
    publisher: 'MicrosoftWindowsServer'
    offer: 'WindowsServer'
    sku: '2022-datacenter-azure-edition'
    version: 'latest'
  }
}

resource publicIp 'Microsoft.Network/publicIPAddresses@2023-04-01' = {
  name: publicIpName
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource nic 'Microsoft.Network/networkInterfaces@2023-04-01' = {
  name: nicName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: subnetId
          }
          privateIPAllocationMethod: 'Dynamic'
          publicIPAddress: {
            id: publicIp.id
          }
        }
      }
    ]
  }
}

resource vm 'Microsoft.Compute/virtualMachines@2023-03-01' = {
  name: vmName
  location: location
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    osProfile: {
      computerName: vmName
      adminUsername: adminUsername
      adminPassword: adminPassword
    }
    storageProfile: {
      imageReference: imageReference[osType]
      osDisk: {
        createOption: 'FromImage'
        managedDisk: {
          storageAccountType: 'Premium_LRS'
        }
      }
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: nic.id
        }
      ]
    }
  }
}

output vmId string = vm.id
output vmName string = vm.name
output publicIp string = publicIp.properties.ipAddress
EOF

echo "Compute module created"
```

---

## Step 7 – Create Main Bicep Template

```bash
# Create main template that uses modules
cat > main.bicep << 'EOF'
@description('Location for all resources')
param location string = resourceGroup().location

@description('Environment name')
@allowed([
  'dev'
  'test'
  'prod'
])
param environmentName string = 'dev'

@description('Admin username')
param adminUsername string = 'azureuser'

@description('Admin password')
@secure()
param adminPassword string

var vnetName = 'vnet-${environmentName}'
var storagePrefix = 'st${environmentName}'
var vmName = 'vm-${environmentName}'

module networkModule 'network.bicep' = {
  name: 'networkDeployment'
  params: {
    location: location
    vnetName: vnetName
    vnetAddressPrefix: '10.0.0.0/16'
  }
}

module storageModule 'storage.bicep' = {
  name: 'storageDeployment'
  params: {
    location: location
    storageAccountPrefix: storagePrefix
    storageSku: 'Standard_LRS'
    containerNames: [
      'data'
      'logs'
    ]
  }
}

module computeModule 'compute.bicep' = {
  name: 'computeDeployment'
  params: {
    location: location
    vmName: vmName
    vmSize: 'Standard_B2s'
    adminUsername: adminUsername
    adminPassword: adminPassword
    subnetId: networkModule.outputs.subnetIds[0]
    osType: 'Ubuntu2204'
  }
  dependsOn: [
    networkModule
  ]
}

output vnetId string = networkModule.outputs.vnetId
output storageAccountName string = storageModule.outputs.storageAccountName
output vmPublicIp string = computeModule.outputs.publicIp
EOF

echo "Main template created"
```

---

## Step 8 – Create Parameters File

```bash
# Create parameters file
cat > main.parameters.json << 'EOF'
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environmentName": {
      "value": "dev"
    },
    "adminUsername": {
      "value": "azureuser"
    }
  }
}
EOF

echo "Parameters file created"
```

---

## Step 9 – Validate Bicep Templates

```bash
# Build main template to check syntax
az bicep build --file main.bicep

echo "Bicep templates validated"
```

---

## Step 10 – Deploy Bicep Template

```bash
# Read admin password from user
read -s -p "Enter VM admin password: " ADMIN_PASSWORD
echo ""

# Deploy main template
az deployment group create \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --template-file main.bicep \
  --parameters main.parameters.json \
  --parameters adminPassword="$ADMIN_PASSWORD"

echo "Deployment started"
```

---

## Step 11 – Monitor Deployment

```bash
# Check deployment status
az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.provisioningState" \
  --output tsv

# List deployment operations
az deployment operation group list \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].{Resource:properties.targetResource.resourceType, State:properties.provisioningState}" \
  --output table
```

---

## Step 12 – Get Deployment Outputs

```bash
# Get virtual network ID
VNET_ID=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.vnetId.value" \
  --output tsv)

echo "$VNET_ID"

# Get storage account name
STORAGE_NAME=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.storageAccountName.value" \
  --output tsv)

echo "$STORAGE_NAME"

# Get VM public IP
VM_PUBLIC_IP=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.vmPublicIp.value" \
  --output tsv)

echo "$VM_PUBLIC_IP"
```

---

## Step 13 – Validate Network Resources

```bash
# List virtual networks
az network vnet list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, AddressSpace:addressSpace.addressPrefixes[0]}" \
  --output table

# List subnets
az network vnet subnet list \
  --vnet-name "vnet-dev" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, AddressPrefix:addressPrefix}" \
  --output table
```

---

## Step 14 – Validate Storage Resources

```bash
# Show storage account details
az storage account show \
  --name "$STORAGE_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Sku:sku.name, Kind:kind}" \
  --output table

# List containers
az storage container list \
  --account-name "$STORAGE_NAME" \
  --auth-mode login \
  --query "[].name" \
  --output table
```

---

## Step 15 – Validate Compute Resources

```bash
# Show VM details
az vm show \
  --name "vm-dev" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Size:hardwareProfile.vmSize, OS:storageProfile.imageReference.offer}" \
  --output table

# Get VM status
az vm get-instance-view \
  --name "vm-dev" \
  --resource-group "$RG_NAME" \
  --query "instanceView.statuses[1].displayStatus" \
  --output tsv
```

---

## Step 16 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
cd ..
rm -rf "$MODULES_DIR"

echo "Cleanup complete"
```

---

## Summary

You created modular Bicep templates for network, storage, and compute resources, deployed infrastructure using a main template that references all modules, and validated the modular deployment approach with proper resource dependencies.
