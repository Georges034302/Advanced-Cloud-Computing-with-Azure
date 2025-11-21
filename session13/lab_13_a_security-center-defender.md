# Lab 13.A: Microsoft Defender for Cloud

## Objectives
- Enable Microsoft Defender for Cloud
- Configure security policies
- Deploy vulnerable resources
- Review security recommendations
- Implement security controls
- Generate compliance reports
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Security Admin role recommended
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab13a-defender"
VNET_NAME="vnet-security"
SUBNET_NAME="subnet-vms"
VM_NAME="vm-test"
STORAGE_ACCOUNT="storage$RANDOM"
LOG_WORKSPACE="log-defender-$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$VM_NAME"
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

## Step 4 – Enable Defender for Cloud

```bash
# Enable Microsoft Defender for Cloud (Free tier)
az security pricing create \
  --name VirtualMachines \
  --tier Free

az security pricing create \
  --name StorageAccounts \
  --tier Free

az security pricing create \
  --name SqlServers \
  --tier Free

echo "Defender for Cloud enabled"
```

---

## Step 5 – Create Log Analytics Workspace

```bash
# Create Log Analytics workspace for Defender
az monitor log-analytics workspace create \
  --name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION"

echo "Log Analytics workspace created"
```

---

## Step 6 – Get Workspace ID

```bash
# Get workspace ID and key
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --query customerId \
  --output tsv)

echo "$WORKSPACE_ID"
```

---

## Step 7 – Configure Security Contacts

```bash
# Set security contact email
az security contact create \
  --name "default1" \
  --email "security@example.com" \
  --phone "1234567890" \
  --alert-notifications On \
  --alerts-admins On

echo "Security contacts configured"
```

---

## Step 8 – Create Virtual Network

```bash
# Create VNet for test resources
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.50.0.0/16 \
  --subnet-name "$SUBNET_NAME" \
  --subnet-prefix 10.50.1.0/24

echo "Virtual network created"
```

---

## Step 9 – Get Admin Password

```bash
# Read admin password securely
read -s -p "Enter VM admin password: " ADMIN_PASSWORD
echo
```

---

## Step 10 – Deploy Test VM

```bash
# Create VM for security assessment
az vm create \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_NAME" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username "azureuser" \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-address "${VM_NAME}-pip" \
  --nsg "${VM_NAME}-nsg"

echo "Test VM created"
```

---

## Step 11 – Create Insecure NSG Rule

```bash
# Create overly permissive NSG rule (will trigger Defender alert)
az network nsg rule create \
  --name AllowAllInbound \
  --nsg-name "${VM_NAME}-nsg" \
  --resource-group "$RG_NAME" \
  --priority 100 \
  --source-address-prefixes '*' \
  --destination-port-ranges '*' \
  --protocol '*' \
  --access Allow \
  --direction Inbound

echo "Insecure NSG rule created"
```

---

## Step 12 – Create Storage Account

```bash
# Create storage account without encryption
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --allow-blob-public-access true

echo "Storage account created"
```

---

## Step 13 – List Security Pricing Tiers

```bash
# Show current Defender pricing tiers
az security pricing list \
  --query "[].{Name:name, Tier:pricingTier}" \
  --output table
```

---

## Step 14 – Get Security Assessments

```bash
# List security assessments for subscription
az security assessment list \
  --query "[?resourceDetails.source=='Azure'].{Name:displayName, Status:status.code, Severity:metadata.severity}" \
  --output table 2>/dev/null || echo "Assessments are being generated"
```

---

## Step 15 – Show Secure Score

```bash
# Get secure score for subscription
az security secure-scores list \
  --query "[].{Name:displayName, CurrentScore:score.current, MaxScore:score.max, Percentage:score.percentage}" \
  --output table 2>/dev/null || echo "Secure score calculating"
```

---

## Step 16 – List Regulatory Compliance

```bash
# Show regulatory compliance standards
az security regulatory-compliance-standards list \
  --query "[].{Name:name, State:state}" \
  --output table 2>/dev/null || echo "Compliance data being collected"
```

---

## Step 17 – Get Security Alerts

```bash
# List security alerts (if any)
az security alert list \
  --query "[].{Name:alertDisplayName, Severity:severity, Status:status}" \
  --output table 2>/dev/null || echo "No alerts yet"
```

---

## Step 18 – Configure Auto-Provisioning

```bash
# Enable auto-provisioning of monitoring agent
az security auto-provisioning-setting update \
  --name default \
  --auto-provision On

echo "Auto-provisioning enabled"
```

---

## Step 19 – List Security Tasks

```bash
# Show security tasks/recommendations
az security task list \
  --query "[].{Name:securityTaskParameters.name, State:state}" \
  --output table 2>/dev/null || echo "Tasks are being generated"
```

---

## Step 20 – Check VM Security Status

```bash
# Get VM resource ID
VM_ID=$(az vm show \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$VM_ID"
```

---

## Step 21 – Show Defender Plans

```bash
# Display available Defender plans
cat << 'EOF'

Microsoft Defender for Cloud Plans:
====================================

Defender for Servers:
  - Vulnerability scanning
  - Just-in-time VM access
  - File integrity monitoring
  - Adaptive application controls

Defender for Storage:
  - Malware scanning
  - Sensitive data threat detection
  - Activity monitoring

Defender for SQL:
  - Vulnerability assessment
  - Threat detection
  - Data discovery and classification

Defender for Containers:
  - Image scanning
  - Runtime protection
  - Kubernetes security

Defender for App Service:
  - Threat detection
  - Vulnerability assessment

EOF
```

---

## Step 22 – Export Security Recommendations

```bash
# Export recommendations to JSON
az security assessment list \
  --query "[].{Name:displayName, Severity:metadata.severity, Status:status.code, Description:metadata.description}" \
  --output json > security-recommendations.json 2>/dev/null || echo "Recommendations file created"

echo "Recommendations exported"
```

---

## Step 23 – Show Compliance Dashboard

```bash
# Display compliance information
cat << 'EOF'

Compliance Standards Available:
================================
- Azure Security Benchmark
- PCI DSS 3.2.1
- ISO 27001
- SOC 2
- NIST SP 800-53
- CIS Microsoft Azure Foundations Benchmark

View in Azure Portal:
Security Center > Regulatory Compliance

EOF
```

---

## Step 24 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f security-recommendations.json

echo "Cleanup complete"
```

---

## Summary

You enabled Microsoft Defender for Cloud on your subscription, configured security policies and contact notifications, deployed test resources including a VM with insecure NSG rules and a storage account with public access, reviewed security assessments and secure score, explored regulatory compliance standards, configured auto-provisioning of monitoring agents, and examined security recommendations for improving your cloud security posture.
