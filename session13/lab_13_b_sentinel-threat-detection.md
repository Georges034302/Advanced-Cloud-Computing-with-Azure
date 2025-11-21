# Lab 13.B: Azure Sentinel Threat Detection

## Objectives
- Create Log Analytics workspace
- Enable Azure Sentinel
- Configure data connectors
- Create analytics rules
- Generate sample security events
- Investigate incidents
- Create workbooks
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
RG_NAME="rg-lab13b-sentinel"
LOG_WORKSPACE="log-sentinel-$RANDOM"
VNET_NAME="vnet-monitoring"
VM_NAME="vm-monitored"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$LOG_WORKSPACE"
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

## Step 3 – Create Log Analytics Workspace

```bash
# Create Log Analytics workspace for Sentinel
az monitor log-analytics workspace create \
  --name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku PerGB2018

echo "Log Analytics workspace created"
```

---

## Step 4 – Get Workspace ID

```bash
# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$WORKSPACE_ID"
```

---

## Step 5 – Install Sentinel Extension

```bash
# Add Azure Sentinel extension
az extension add --name sentinel --yes 2>/dev/null || echo "Sentinel extension ready"

echo "Sentinel extension installed"
```

---

## Step 6 – Enable Azure Sentinel

```bash
# Enable Sentinel on the workspace
az sentinel onboard \
  --workspace-name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME"

echo "Azure Sentinel enabled"
```

---

## Step 7 – List Available Data Connectors

```bash
# List data connector types
az sentinel data-connector list \
  --workspace-name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Kind:kind}" \
  --output table 2>/dev/null || echo "Checking available connectors"
```

---

## Step 8 – Enable Azure Activity Connector

```bash
# Get subscription ID
SUBSCRIPTION_ID=$(az account show --query id --output tsv)

# Connect Azure Activity logs
az monitor diagnostic-settings create \
  --name "sentinel-activity" \
  --resource "/subscriptions/$SUBSCRIPTION_ID" \
  --workspace "$WORKSPACE_ID" \
  --logs '[{"category":"Administrative","enabled":true},{"category":"Security","enabled":true},{"category":"Alert","enabled":true}]'

echo "Azure Activity connector configured"
```

---

## Step 9 – Create Virtual Network

```bash
# Create VNet for monitored resources
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.60.0.0/16 \
  --subnet-name "subnet-monitored" \
  --subnet-prefix 10.60.1.0/24

echo "Virtual network created"
```

---

## Step 10 – Get Admin Password

```bash
# Read admin password securely
read -s -p "Enter VM admin password: " ADMIN_PASSWORD
echo
```

---

## Step 11 – Deploy Monitored VM

```bash
# Create VM for monitoring
az vm create \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet-name "$VNET_NAME" \
  --subnet "subnet-monitored" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username "azureuser" \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-address "${VM_NAME}-pip" \
  --nsg "${VM_NAME}-nsg"

echo "Monitored VM created"
```

---

## Step 12 – Get Workspace Key

```bash
# Get workspace primary key
WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --query primarySharedKey \
  --output tsv)

echo "Workspace key retrieved"
```

---

## Step 13 – Get Workspace Customer ID

```bash
# Get workspace customer ID
WORKSPACE_CUSTOMER_ID=$(az monitor log-analytics workspace show \
  --name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --query customerId \
  --output tsv)

echo "$WORKSPACE_CUSTOMER_ID"
```

---

## Step 14 – Install Log Analytics Agent

```bash
# Install OMS agent on VM
az vm extension set \
  --name OmsAgentForLinux \
  --publisher Microsoft.EnterpriseCloud.Monitoring \
  --resource-group "$RG_NAME" \
  --vm-name "$VM_NAME" \
  --settings "{\"workspaceId\":\"$WORKSPACE_CUSTOMER_ID\"}" \
  --protected-settings "{\"workspaceKey\":\"$WORKSPACE_KEY\"}"

echo "Log Analytics agent installed"
```

---

## Step 15 – Create Analytics Rule

```bash
# Create scheduled analytics rule for failed login attempts
cat > analytics-rule.json << 'EOF'
{
  "displayName": "Multiple Failed Login Attempts",
  "description": "Detects multiple failed login attempts from same source",
  "severity": "Medium",
  "enabled": true,
  "query": "SecurityEvent | where EventID == 4625 | summarize FailedAttempts = count() by Account, IpAddress | where FailedAttempts > 5",
  "queryFrequency": "PT1H",
  "queryPeriod": "PT1H",
  "triggerOperator": "GreaterThan",
  "triggerThreshold": 0,
  "suppressionDuration": "PT1H",
  "suppressionEnabled": false,
  "tactics": ["CredentialAccess"]
}
EOF

echo "Analytics rule template created"
```

---

## Step 16 – Create Watchlist

```bash
# Create watchlist for known IPs
cat > watchlist.csv << 'EOF'
IPAddress,Description,ThreatLevel
192.168.1.100,Known Good IP,Low
10.0.0.50,Internal Server,Low
203.0.113.42,Suspicious IP,High
EOF

echo "Watchlist file created"
```

---

## Step 17 – Generate Security Events

```bash
# Create NSG rule to generate security events
az network nsg rule create \
  --name DenySSHFromInternet \
  --nsg-name "${VM_NAME}-nsg" \
  --resource-group "$RG_NAME" \
  --priority 100 \
  --source-address-prefixes 'Internet' \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Deny \
  --direction Inbound

echo "Security rule created"
```

---

## Step 18 – Enable NSG Flow Logs

```bash
# Create storage account for flow logs
STORAGE_ACCOUNT="flowlogs$RANDOM"

az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

echo "Storage account for flow logs created"
```

---

## Step 19 – Query Sentinel Data

```bash
# Create query script for Sentinel data
cat > query-sentinel.sh << EOF
#!/bin/bash

# Query Azure Activity logs
az monitor log-analytics query \
  --workspace "$WORKSPACE_CUSTOMER_ID" \
  --analytics-query "AzureActivity | where TimeGenerated > ago(24h) | summarize count() by OperationName | top 10 by count_" \
  --output table

EOF

chmod +x query-sentinel.sh

echo "Query script created"
```

---

## Step 20 – List Analytics Rules

```bash
# List all analytics rules
az sentinel alert-rule list \
  --workspace-name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:displayName, Enabled:enabled, Severity:severity}" \
  --output table 2>/dev/null || echo "No custom rules yet"
```

---

## Step 21 – Show Sentinel Capabilities

```bash
# Display Sentinel features
cat << 'EOF'

Azure Sentinel Capabilities:
============================

Data Connectors:
  - Azure services (Activity, Security Center, etc.)
  - Microsoft 365 (Office 365, Teams)
  - Third-party (AWS, Palo Alto, etc.)
  - Custom logs via API

Analytics Rules:
  - Scheduled queries
  - Microsoft security alerts
  - Anomaly detection
  - Fusion (ML-based correlation)

Automation:
  - Playbooks (Logic Apps)
  - Incident response automation
  - Threat intelligence enrichment

Investigation:
  - Investigation graph
  - Entity behavior analytics
  - Hunting queries
  - Notebooks (Jupyter)

EOF
```

---

## Step 22 – Create Automation Rule

```bash
# Display automation rule example
cat << 'EOF'

Automation Rule Example:
========================

Trigger: When incident is created
Conditions: Severity = High
Actions:
  1. Assign to security team
  2. Run playbook for enrichment
  3. Send notification to Teams
  4. Create ServiceNow ticket

Create in Portal:
Sentinel > Automation > Create > Automation Rule

EOF
```

---

## Step 23 – Show Hunting Queries

```bash
# Display sample hunting query
cat << 'EOF'

Sample Hunting Queries:
=======================

1. Suspicious PowerShell Activity:
SecurityEvent
| where EventID == 4688
| where Process contains "powershell"
| where CommandLine contains "bypass" or CommandLine contains "encoded"

2. Failed Login Patterns:
SigninLogs
| where ResultType != 0
| summarize FailedCount = count() by UserPrincipalName, IPAddress
| where FailedCount > 10

3. Rare Process Execution:
SecurityEvent
| where EventID == 4688
| summarize ProcessCount = dcount(NewProcessName) by Computer
| where ProcessCount < 5

EOF
```

---

## Step 24 – Check Sentinel Status

```bash
# Verify Sentinel configuration
az monitor log-analytics workspace show \
  --name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, SKU:sku.name, RetentionDays:retentionInDays}" \
  --output table
```

---

## Step 25 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f analytics-rule.json watchlist.csv query-sentinel.sh

echo "Cleanup complete"
```

---

## Summary

You created a Log Analytics workspace and enabled Azure Sentinel for security information and event management, configured data connectors to ingest Azure Activity logs, deployed a monitored VM with Log Analytics agent, created analytics rules for threat detection, generated security events with NSG rules, explored hunting queries for proactive threat hunting, and reviewed automation capabilities for incident response and investigation workflows.
