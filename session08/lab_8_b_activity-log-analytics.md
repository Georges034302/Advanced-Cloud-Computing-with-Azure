# Lab 8.B: Activity Log Analytics

## Objectives
- Create Log Analytics workspace
- Configure activity log integration
- Create diagnostic settings
- Query activity logs
- Create custom queries
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
RG_NAME="rg-lab8b-logs"
WORKSPACE_NAME="law-monitoring"
STORAGE_ACCOUNT="storage$RANDOM$RANDOM"
DIAGNOSTIC_SETTING="diag-activity-logs"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$WORKSPACE_NAME"
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
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --workspace-name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku PerGB2018

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --workspace-name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$WORKSPACE_ID"
```

---

## Step 4 – Get Workspace Key

```bash
# Get workspace customer ID
WORKSPACE_CUSTOMER_ID=$(az monitor log-analytics workspace show \
  --workspace-name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --query customerId \
  --output tsv)

echo "$WORKSPACE_CUSTOMER_ID"

# Get workspace shared key
WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --workspace-name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --query primarySharedKey \
  --output tsv)

echo "Workspace keys retrieved"
```

---

## Step 5 – Create Storage Account

```bash
# Create storage account for diagnostic logs
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

# Get storage account ID
STORAGE_ID=$(az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$STORAGE_ID"
```

---

## Step 6 – Configure Activity Log Diagnostic Settings

```bash
# Get subscription ID
SUBSCRIPTION_ID=$(az account show --query id --output tsv)

echo "$SUBSCRIPTION_ID"

# Create diagnostic setting for activity log
az monitor diagnostic-settings subscription create \
  --name "$DIAGNOSTIC_SETTING" \
  --location "$LOCATION" \
  --workspace "$WORKSPACE_ID" \
  --storage-account "$STORAGE_ID" \
  --logs '[
    {"category": "Administrative", "enabled": true},
    {"category": "Security", "enabled": true},
    {"category": "ServiceHealth", "enabled": true},
    {"category": "Alert", "enabled": true},
    {"category": "Policy", "enabled": true}
  ]'

echo "Diagnostic settings configured"
```

---

## Step 7 – Generate Activity Log Entries

```bash
# Create test resources to generate logs
az storage container create \
  --name "test-container" \
  --account-name "$STORAGE_ACCOUNT"

echo "Created test container"

# Create and delete a network security group
az network nsg create \
  --name "nsg-test" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION"

echo "Created NSG"

az network nsg delete \
  --name "nsg-test" \
  --resource-group "$RG_NAME" \
  --yes

echo "Deleted NSG"

# Wait for logs to be ingested
echo "Waiting for log ingestion..."
sleep 120
```

---

## Step 8 – View Activity Logs

```bash
# Query recent activity log entries
az monitor activity-log list \
  --resource-group "$RG_NAME" \
  --start-time "$(date -u -d '30 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --query "[].{Time:eventTimestamp, Operation:operationName.localizedValue, Status:status.localizedValue, User:caller}" \
  --output table
```

---

## Step 9 – Query Logs by Category

```bash
# Query administrative operations
az monitor activity-log list \
  --resource-group "$RG_NAME" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --category Administrative \
  --query "[].{Time:eventTimestamp, Resource:resourceId, Operation:operationName.localizedValue}" \
  --output table
```

---

## Step 10 – Query Log Analytics

```bash
# Query AzureActivity table
az monitor log-analytics query \
  --workspace "$WORKSPACE_CUSTOMER_ID" \
  --analytics-query "AzureActivity | where TimeGenerated > ago(1h) | summarize count() by OperationNameValue" \
  --output table
```

---

## Step 11 – Query Failed Operations

```bash
# Query failed operations
az monitor log-analytics query \
  --workspace "$WORKSPACE_CUSTOMER_ID" \
  --analytics-query "AzureActivity | where TimeGenerated > ago(1h) | where ActivityStatusValue == 'Failed' | project TimeGenerated, OperationNameValue, Caller, ResourceGroup" \
  --output table
```

---

## Step 12 – Query by Resource Type

```bash
# Query operations by resource type
az monitor log-analytics query \
  --workspace "$WORKSPACE_CUSTOMER_ID" \
  --analytics-query "AzureActivity | where TimeGenerated > ago(2h) | summarize Count=count() by ResourceProviderValue | order by Count desc" \
  --output table
```

---

## Step 13 – Query User Activity

```bash
# Query activity by caller
az monitor log-analytics query \
  --workspace "$WORKSPACE_CUSTOMER_ID" \
  --analytics-query "AzureActivity | where TimeGenerated > ago(2h) | summarize Operations=count() by Caller | order by Operations desc" \
  --output table
```

---

## Step 14 – Create Custom Query

```bash
# Query resource creation and deletion events
az monitor log-analytics query \
  --workspace "$WORKSPACE_CUSTOMER_ID" \
  --analytics-query "
    AzureActivity 
    | where TimeGenerated > ago(2h) 
    | where OperationNameValue contains 'Microsoft.Network/networkSecurityGroups/write' or OperationNameValue contains 'Microsoft.Network/networkSecurityGroups/delete'
    | project TimeGenerated, OperationNameValue, ResourceId, Caller, ActivityStatusValue
    | order by TimeGenerated desc
  " \
  --output table
```

---

## Step 15 – Validate Query Results

```bash
# Query all activity in resource group
az monitor log-analytics query \
  --workspace "$WORKSPACE_CUSTOMER_ID" \
  --analytics-query "AzureActivity | where ResourceGroup == '$RG_NAME' | where TimeGenerated > ago(1h) | project TimeGenerated, OperationNameValue, ActivityStatusValue, Caller | order by TimeGenerated desc" \
  --output table
```

---

## Step 16 – Export Query Results

```bash
# Export activity log data to JSON
az monitor activity-log list \
  --resource-group "$RG_NAME" \
  --start-time "$(date -u -d '2 hours ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --output json > activity-log-export.json

echo "Activity log exported to activity-log-export.json"

# Show summary
cat activity-log-export.json | python3 -c "import sys, json; data = json.load(sys.stdin); print(f'Total events: {len(data)}')"
```

---

## Step 17 – Cleanup

```bash
# Delete diagnostic setting
az monitor diagnostic-settings subscription delete \
  --name "$DIAGNOSTIC_SETTING" \
  --yes

# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f activity-log-export.json

echo "Cleanup complete"
```

---

## Summary

You created a Log Analytics workspace, configured activity log diagnostic settings to send logs to the workspace, generated activity log entries by creating and deleting resources, and queried logs using Log Analytics queries.
