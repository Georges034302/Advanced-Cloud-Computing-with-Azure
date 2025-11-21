# Lab 13.D: Azure Policy and Governance

## Objectives
- Create custom Azure policies
- Assign policies to subscriptions
- Test policy compliance
- Create management groups
- Implement resource tags
- Review compliance reports
- Remediate non-compliant resources
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Owner or Policy Contributor role
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab13d-policy"
POLICY_NAME="require-tag-policy"
INITIATIVE_NAME="governance-initiative"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$POLICY_NAME"
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

## Step 3 – Get Subscription ID

```bash
# Get current subscription ID
SUBSCRIPTION_ID=$(az account show --query id --output tsv)

echo "$SUBSCRIPTION_ID"
```

---

## Step 4 – List Built-in Policies

```bash
# List common built-in policies
az policy definition list \
  --query "[?policyType=='BuiltIn'].{Name:displayName, Category:metadata.category}" \
  --output table | head -20
```

---

## Step 5 – Show Specific Policy Definition

```bash
# View allowed locations policy
az policy definition show \
  --name "e56962a6-4747-49cd-b67b-bf8b01975c4c" \
  --query "{Name:displayName, Description:description, PolicyRule:policyRule}" \
  --output json | head -30
```

---

## Step 6 – Create Custom Tag Policy

```bash
# Create policy to require specific tags
cat > tag-policy.json << 'EOF'
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Resources/subscriptions/resourceGroups"
        },
        {
          "anyOf": [
            {
              "field": "tags['Environment']",
              "exists": "false"
            },
            {
              "field": "tags['CostCenter']",
              "exists": "false"
            }
          ]
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
EOF

echo "Tag policy definition created"
```

---

## Step 7 – Create Policy Definition

```bash
# Create custom policy definition
az policy definition create \
  --name "$POLICY_NAME" \
  --display-name "Require Environment and CostCenter Tags" \
  --description "Ensures all resource groups have Environment and CostCenter tags" \
  --rules tag-policy.json \
  --mode Indexed

echo "Policy definition created"
```

---

## Step 8 – Create Allowed Locations Policy

```bash
# Create policy for allowed Azure regions
cat > locations-policy.json << 'EOF'
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "not": {
        "field": "location",
        "in": "[parameters('allowedLocations')]"
      }
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {
    "allowedLocations": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed locations",
        "description": "List of allowed Azure regions"
      }
    }
  }
}
EOF

# Create policy
az policy definition create \
  --name "allowed-locations-policy" \
  --display-name "Allowed Azure Regions" \
  --description "Restricts resources to specific Azure regions" \
  --rules locations-policy.json \
  --params '{"allowedLocations":{"type":"Array"}}' \
  --mode Indexed

echo "Locations policy created"
```

---

## Step 9 – Assign Policy to Resource Group

```bash
# Assign allowed locations policy
az policy assignment create \
  --name "restrict-locations" \
  --display-name "Restrict to Australia Regions" \
  --policy "allowed-locations-policy" \
  --params '{"allowedLocations":{"value":["australiaeast","australiasoutheast"]}}' \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME"

echo "Policy assigned"
```

---

## Step 10 – Create VM Size Restriction Policy

```bash
# Create policy to restrict VM SKUs
cat > vm-sku-policy.json << 'EOF'
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Compute/virtualMachines"
        },
        {
          "not": {
            "field": "Microsoft.Compute/virtualMachines/sku.name",
            "in": "[parameters('allowedVMSKUs')]"
          }
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {
    "allowedVMSKUs": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed VM SKUs",
        "description": "List of allowed virtual machine SKUs"
      }
    }
  }
}
EOF

# Create policy
az policy definition create \
  --name "allowed-vm-skus" \
  --display-name "Allowed Virtual Machine SKUs" \
  --description "Restricts VM sizes to approved SKUs" \
  --rules vm-sku-policy.json \
  --mode Indexed

echo "VM SKU policy created"
```

---

## Step 11 – Assign VM SKU Policy

```bash
# Assign VM SKU policy with parameters
az policy assignment create \
  --name "restrict-vm-sizes" \
  --display-name "Allow Only B-series VMs" \
  --policy "allowed-vm-skus" \
  --params '{"allowedVMSKUs":{"value":["Standard_B1s","Standard_B2s","Standard_B1ms"]}}' \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME"

echo "VM SKU policy assigned"
```

---

## Step 12 – Create Policy Initiative

```bash
# Create policy set (initiative) definition
cat > initiative.json << 'EOF'
[
  {
    "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c",
    "parameters": {
      "listOfAllowedLocations": {
        "value": ["australiaeast", "australiasoutheast"]
      }
    }
  }
]
EOF

az policy set-definition create \
  --name "$INITIATIVE_NAME" \
  --display-name "Governance Initiative" \
  --description "Collection of governance policies" \
  --definitions initiative.json

echo "Policy initiative created"
```

---

## Step 13 – List Policy Assignments

```bash
# List all policy assignments in resource group
az policy assignment list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:displayName, Policy:policyDefinitionId, Scope:scope}" \
  --output table
```

---

## Step 14 – Test Policy Compliance

```bash
# Try to create storage account in non-compliant location
az storage account create \
  --name "teststorage$RANDOM" \
  --resource-group "$RG_NAME" \
  --location "eastus" \
  --sku Standard_LRS 2>&1 || echo "Policy blocked non-compliant resource"
```

---

## Step 15 – Create Compliant Storage Account

```bash
# Create storage account in allowed location
STORAGE_ACCOUNT="compliant$RANDOM"

az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

echo "Compliant storage account created"
```

---

## Step 16 – Check Policy Compliance State

```bash
# Get compliance state for resource group
az policy state list \
  --resource-group "$RG_NAME" \
  --query "[].{Resource:resourceId, ComplianceState:complianceState, Policy:policyDefinitionName}" \
  --output table 2>/dev/null || echo "Compliance data being collected"
```

---

## Step 17 – Create Tag Remediation Policy

```bash
# Create policy to add missing tags
cat > add-tag-policy.json << 'EOF'
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "field": "tags['Environment']",
      "exists": "false"
    },
    "then": {
      "effect": "modify",
      "details": {
        "roleDefinitionIds": [
          "/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c"
        ],
        "operations": [
          {
            "operation": "add",
            "field": "tags['Environment']",
            "value": "Untagged"
          }
        ]
      }
    }
  }
}
EOF

echo "Tag remediation policy created"
```

---

## Step 18 – Apply Resource Tags

```bash
# Tag resource group
az group update \
  --name "$RG_NAME" \
  --tags Environment=Production CostCenter=IT Department=Engineering

echo "Resource group tagged"
```

---

## Step 19 – Tag Storage Account

```bash
# Apply tags to storage account
az storage account update \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --tags Environment=Production CostCenter=IT Application=Storage

echo "Storage account tagged"
```

---

## Step 20 – Query Resources by Tags

```bash
# Find all resources with specific tag
az resource list \
  --tag Environment=Production \
  --query "[].{Name:name, Type:type, Tags:tags}" \
  --output table
```

---

## Step 21 – Show Policy Exemptions

```bash
# Display policy exemption information
cat << 'EOF'

Policy Exemptions:
==================

Exemption Categories:
  - Waiver: Resource doesn't need to comply
  - Mitigated: Risk addressed by other means

Create Exemption:
az policy exemption create \
  --name "dev-exemption" \
  --policy-assignment "restrict-locations" \
  --exemption-category Waiver \
  --expires "2025-12-31T23:59:59Z"

EOF
```

---

## Step 22 – Create Audit Policy

```bash
# Create audit-only policy for storage encryption
cat > audit-encryption-policy.json << 'EOF'
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Storage/storageAccounts"
        },
        {
          "field": "Microsoft.Storage/storageAccounts/enableHttpsTrafficOnly",
          "notEquals": "true"
        }
      ]
    },
    "then": {
      "effect": "audit"
    }
  }
}
EOF

az policy definition create \
  --name "audit-storage-https" \
  --display-name "Audit Storage HTTPS Requirement" \
  --description "Audits storage accounts not requiring HTTPS" \
  --rules audit-encryption-policy.json \
  --mode Indexed

echo "Audit policy created"
```

---

## Step 23 – Show Compliance Dashboard

```bash
# Display compliance summary
cat << 'EOF'

Compliance Monitoring:
======================

View in Azure Portal:
1. Navigate to Azure Policy
2. Select "Compliance"
3. Review:
   - Overall compliance percentage
   - Non-compliant resources
   - Policy assignments
   - Compliance by policy

Generate Reports:
- Export compliance data
- Schedule automated reports
- Integrate with Azure Monitor

EOF
```

---

## Step 24 – List All Custom Policies

```bash
# Show all custom policy definitions
az policy definition list \
  --query "[?policyType=='Custom'].{Name:displayName, Description:description}" \
  --output table
```

---

## Step 25 – Show Policy Effects

```bash
# Display available policy effects
cat << 'EOF'

Policy Effects:
===============

Deny:
  - Prevents non-compliant resource creation
  - Returns error to user

Audit:
  - Logs non-compliant resources
  - Doesn't block deployment

Append:
  - Adds fields to resources during creation
  - Example: Add required tags

Modify:
  - Changes resource properties
  - Requires managed identity

DeployIfNotExists:
  - Deploys additional resources
  - Example: Deploy diagnostic settings

AuditIfNotExists:
  - Checks for related resources
  - Example: Check VM has antimalware

Disabled:
  - Policy exists but not enforced

EOF
```

---

## Step 26 – Cleanup

```bash
# Remove policy assignments
az policy assignment delete --name "restrict-locations" --resource-group "$RG_NAME"
az policy assignment delete --name "restrict-vm-sizes" --resource-group "$RG_NAME"

# Delete custom policies
az policy definition delete --name "$POLICY_NAME"
az policy definition delete --name "allowed-locations-policy"
az policy definition delete --name "allowed-vm-skus"
az policy definition delete --name "audit-storage-https"

# Delete policy initiative
az policy set-definition delete --name "$INITIATIVE_NAME"

# Delete resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f tag-policy.json locations-policy.json vm-sku-policy.json initiative.json add-tag-policy.json audit-encryption-policy.json

echo "Cleanup complete"
```

---

## Summary

You created custom Azure policy definitions with different effects including deny, audit, and modify, assigned policies to resource groups with parameters for allowed locations and VM SKUs, tested policy compliance by attempting to create non-compliant resources, created policy initiatives to group related policies, implemented resource tagging for governance and cost management, reviewed compliance states and reports, and explored policy exemptions and remediation options for managing exceptions and fixing non-compliant resources.
