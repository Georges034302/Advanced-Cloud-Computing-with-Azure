# Lab 9.A: ARM Template Virtual Network

## Objectives
- Create ARM template for virtual network
- Define parameters and variables
- Deploy network infrastructure
- Validate deployment
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
RG_NAME="rg-lab9a-arm"
TEMPLATE_FILE="vnet-template.json"
PARAMETERS_FILE="vnet-parameters.json"
DEPLOYMENT_NAME="vnet-deployment"

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

## Step 3 – Create ARM Template

```bash
# Create ARM template for virtual network
cat > "$TEMPLATE_FILE" << 'EOF'
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vnetName": {
      "type": "string",
      "metadata": {
        "description": "Name of the virtual network"
      }
    },
    "vnetAddressPrefix": {
      "type": "string",
      "defaultValue": "10.0.0.0/16",
      "metadata": {
        "description": "Address prefix for the virtual network"
      }
    },
    "subnet1Name": {
      "type": "string",
      "defaultValue": "subnet-web",
      "metadata": {
        "description": "Name of the first subnet"
      }
    },
    "subnet1Prefix": {
      "type": "string",
      "defaultValue": "10.0.1.0/24",
      "metadata": {
        "description": "Address prefix for the first subnet"
      }
    },
    "subnet2Name": {
      "type": "string",
      "defaultValue": "subnet-app",
      "metadata": {
        "description": "Name of the second subnet"
      }
    },
    "subnet2Prefix": {
      "type": "string",
      "defaultValue": "10.0.2.0/24",
      "metadata": {
        "description": "Address prefix for the second subnet"
      }
    },
    "subnet3Name": {
      "type": "string",
      "defaultValue": "subnet-data",
      "metadata": {
        "description": "Name of the third subnet"
      }
    },
    "subnet3Prefix": {
      "type": "string",
      "defaultValue": "10.0.3.0/24",
      "metadata": {
        "description": "Address prefix for the third subnet"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for all resources"
      }
    }
  },
  "variables": {
    "nsgName": "[concat(parameters('vnetName'), '-nsg')]"
  },
  "resources": [
    {
      "type": "Microsoft.Network/networkSecurityGroups",
      "apiVersion": "2023-04-01",
      "name": "[variables('nsgName')]",
      "location": "[parameters('location')]",
      "properties": {
        "securityRules": [
          {
            "name": "AllowHTTP",
            "properties": {
              "protocol": "Tcp",
              "sourcePortRange": "*",
              "destinationPortRange": "80",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 100,
              "direction": "Inbound"
            }
          },
          {
            "name": "AllowHTTPS",
            "properties": {
              "protocol": "Tcp",
              "sourcePortRange": "*",
              "destinationPortRange": "443",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 110,
              "direction": "Inbound"
            }
          },
          {
            "name": "AllowSSH",
            "properties": {
              "protocol": "Tcp",
              "sourcePortRange": "*",
              "destinationPortRange": "22",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 120,
              "direction": "Inbound"
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2023-04-01",
      "name": "[parameters('vnetName')]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.Network/networkSecurityGroups', variables('nsgName'))]"
      ],
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "[parameters('vnetAddressPrefix')]"
          ]
        },
        "subnets": [
          {
            "name": "[parameters('subnet1Name')]",
            "properties": {
              "addressPrefix": "[parameters('subnet1Prefix')]",
              "networkSecurityGroup": {
                "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('nsgName'))]"
              }
            }
          },
          {
            "name": "[parameters('subnet2Name')]",
            "properties": {
              "addressPrefix": "[parameters('subnet2Prefix')]",
              "networkSecurityGroup": {
                "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('nsgName'))]"
              }
            }
          },
          {
            "name": "[parameters('subnet3Name')]",
            "properties": {
              "addressPrefix": "[parameters('subnet3Prefix')]"
            }
          }
        ]
      }
    }
  ],
  "outputs": {
    "vnetId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Network/virtualNetworks', parameters('vnetName'))]"
    },
    "vnetName": {
      "type": "string",
      "value": "[parameters('vnetName')]"
    },
    "subnet1Id": {
      "type": "string",
      "value": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnet1Name'))]"
    },
    "subnet2Id": {
      "type": "string",
      "value": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnet2Name'))]"
    },
    "subnet3Id": {
      "type": "string",
      "value": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnet3Name'))]"
    },
    "nsgId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('nsgName'))]"
    }
  }
}
EOF

echo "ARM template created"
```

---

## Step 4 – Create Parameters File

```bash
# Create parameters file
cat > "$PARAMETERS_FILE" << 'EOF'
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vnetName": {
      "value": "vnet-arm-deployment"
    },
    "vnetAddressPrefix": {
      "value": "10.0.0.0/16"
    },
    "subnet1Name": {
      "value": "subnet-web"
    },
    "subnet1Prefix": {
      "value": "10.0.1.0/24"
    },
    "subnet2Name": {
      "value": "subnet-app"
    },
    "subnet2Prefix": {
      "value": "10.0.2.0/24"
    },
    "subnet3Name": {
      "value": "subnet-data"
    },
    "subnet3Prefix": {
      "value": "10.0.3.0/24"
    }
  }
}
EOF

echo "Parameters file created"
```

---

## Step 5 – Validate ARM Template

```bash
# Validate template syntax
az deployment group validate \
  --resource-group "$RG_NAME" \
  --template-file "$TEMPLATE_FILE" \
  --parameters "$PARAMETERS_FILE" \
  --query "properties.provisioningState" \
  --output tsv
```

---

## Step 6 – Deploy ARM Template

```bash
# Deploy ARM template
az deployment group create \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --template-file "$TEMPLATE_FILE" \
  --parameters "$PARAMETERS_FILE"

echo "Deployment started"
```

---

## Step 7 – Monitor Deployment

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
  --query "[].{Operation:properties.targetResource.resourceType, State:properties.provisioningState}" \
  --output table
```

---

## Step 8 – Get Deployment Outputs

```bash
# Get virtual network ID
VNET_ID=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.vnetId.value" \
  --output tsv)

echo "$VNET_ID"

# Get virtual network name
VNET_NAME=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.vnetName.value" \
  --output tsv)

echo "$VNET_NAME"

# Get NSG ID
NSG_ID=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.nsgId.value" \
  --output tsv)

echo "$NSG_ID"
```

---

## Step 9 – Validate Virtual Network

```bash
# List virtual networks
az network vnet list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, AddressSpace:addressSpace.addressPrefixes[0], Location:location}" \
  --output table

# Show virtual network details
az network vnet show \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, AddressSpace:addressSpace.addressPrefixes, Subnets:subnets[].name}" \
  --output json
```

---

## Step 10 – Validate Subnets

```bash
# List subnets
az network vnet subnet list \
  --vnet-name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, AddressPrefix:addressPrefix, NSG:networkSecurityGroup.id}" \
  --output table
```

---

## Step 11 – Validate Network Security Group

```bash
# Show NSG details
az network nsg show \
  --name "${VNET_NAME}-nsg" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Rules:securityRules[].{Name:name, Priority:priority, Port:destinationPortRange}}" \
  --output json

# List NSG rules
az network nsg rule list \
  --nsg-name "${VNET_NAME}-nsg" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Priority:priority, Direction:direction, Access:access, Port:destinationPortRange}" \
  --output table
```

---

## Step 12 – Export Deployment Template

```bash
# Export deployed template
az deployment group export \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --output json > exported-template.json

echo "Template exported"

# Show exported template summary
cat exported-template.json | python3 -c "import sys, json; data = json.load(sys.stdin); print(f\"Resources: {len(data.get('resources', []))}\")"
```

---

## Step 13 – View Deployment History

```bash
# List all deployments
az deployment group list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, State:properties.provisioningState, Timestamp:properties.timestamp}" \
  --output table
```

---

## Step 14 – Test What-If Deployment

```bash
# Run what-if analysis
az deployment group what-if \
  --name "whatif-test" \
  --resource-group "$RG_NAME" \
  --template-file "$TEMPLATE_FILE" \
  --parameters "$PARAMETERS_FILE"
```

---

## Step 15 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f "$TEMPLATE_FILE" "$PARAMETERS_FILE" exported-template.json

echo "Cleanup complete"
```

---

## Summary

You created an ARM template defining a virtual network with three subnets and network security group, validated the template syntax, deployed the infrastructure using Azure CLI, verified all resources were created correctly, and explored deployment outputs and history.
