# Lab 8.A: Monitor Dashboard and Alerts

## Objectives
- Create virtual machine for monitoring
- Configure Azure Monitor metrics
- Create custom dashboard
- Configure metric alerts
- Test alert notifications
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
RG_NAME="rg-lab8a-monitor"
VM_NAME="vm-monitor"
ADMIN_USER="azureuser"
DASHBOARD_NAME="dashboard-monitoring"
ALERT_RULE="alert-cpu-high"
ACTION_GROUP="ag-notifications"

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

## Step 3 – Create Virtual Machine

```bash
# Read VM password from user
read -s -p "Enter VM password: " ADMIN_PASSWORD
echo ""

# Create VM
az vm create \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --image "Ubuntu2204" \
  --size "Standard_B2s" \
  --admin-username "$ADMIN_USER" \
  --admin-password "$ADMIN_PASSWORD" \
  --authentication-type password \
  --public-ip-sku Standard

echo "VM created"
```

---

## Step 4 – Get VM Resource ID

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

## Step 5 – View Available Metrics

```bash
# List available metrics for VM
az monitor metrics list-definitions \
  --resource "$VM_ID" \
  --query "[].{Name:name.value, Unit:unit, Type:metricAvailabilities[0].timeGrain}" \
  --output table
```

---

## Step 6 – Query CPU Metrics

```bash
# Get CPU percentage metrics
az monitor metrics list \
  --resource "$VM_ID" \
  --metric "Percentage CPU" \
  --start-time "$(date -u -d '30 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT1M \
  --aggregation Average \
  --output table
```

---

## Step 7 – Create Action Group

```bash
# Create action group for alert notifications
az monitor action-group create \
  --name "$ACTION_GROUP" \
  --resource-group "$RG_NAME" \
  --short-name "alerts"

echo "Action group created"
```

---

## Step 8 – Add Email to Action Group

```bash
# Read email address from user
read -p "Enter email for alerts: " ALERT_EMAIL

# Add email receiver to action group
az monitor action-group update \
  --name "$ACTION_GROUP" \
  --resource-group "$RG_NAME" \
  --add-action email admin "$ALERT_EMAIL"

echo "Email receiver added"
```

---

## Step 9 – Create Metric Alert Rule

```bash
# Create alert rule for high CPU usage
az monitor metrics alert create \
  --name "$ALERT_RULE" \
  --resource-group "$RG_NAME" \
  --scopes "$VM_ID" \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action "$ACTION_GROUP" \
  --description "Alert when CPU exceeds 80%"

echo "Alert rule created"
```

---

## Step 10 – View Alert Rules

```bash
# List all alert rules
az monitor metrics alert list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Enabled:enabled, Condition:criteria.allOf[0].name}" \
  --output table

# Show alert rule details
az monitor metrics alert show \
  --name "$ALERT_RULE" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Severity:severity, Threshold:criteria.allOf[0].threshold, Window:windowSize}" \
  --output table
```

---

## Step 11 – Create Dashboard

```bash
# Create dashboard JSON configuration
cat > dashboard.json << EOF
{
  "lenses": {
    "0": {
      "order": 0,
      "parts": {
        "0": {
          "position": {
            "x": 0,
            "y": 0,
            "colSpan": 6,
            "rowSpan": 4
          },
          "metadata": {
            "inputs": [
              {
                "name": "resourceId",
                "value": "$VM_ID"
              },
              {
                "name": "timeContext",
                "value": {
                  "durationMs": 3600000
                }
              },
              {
                "name": "chartType",
                "value": 2
              }
            ],
            "type": "Extension/Microsoft_Azure_Monitoring/PartType/MetricsChartPart"
          }
        }
      }
    }
  },
  "metadata": {
    "model": {
      "timeRange": {
        "type": "MsPortalFx.Composition.Configuration.ValueTypes.TimeRange",
        "value": {
          "relative": {
            "duration": 24,
            "timeUnit": 1
          }
        }
      }
    }
  },
  "name": "$DASHBOARD_NAME",
  "type": "Microsoft.Portal/dashboards",
  "location": "$LOCATION",
  "tags": {
    "hidden-title": "$DASHBOARD_NAME"
  }
}
EOF

echo "Dashboard configuration created"
```

---

## Step 12 – Deploy Dashboard

```bash
# Deploy dashboard
az portal dashboard create \
  --resource-group "$RG_NAME" \
  --name "$DASHBOARD_NAME" \
  --input-path dashboard.json \
  --location "$LOCATION"

echo "Dashboard deployed"
```

---

## Step 13 – Generate CPU Load

```bash
# Get VM public IP
VM_IP=$(az vm show \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "$VM_IP"

# Install stress tool and generate load
az vm run-command invoke \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --command-id RunShellScript \
  --scripts "sudo apt-get update && sudo apt-get install -y stress && stress --cpu 2 --timeout 300 &"

echo "CPU load generation started"
```

---

## Step 14 – Validate Metrics Collection

```bash
# Wait for metrics to be collected
echo "Waiting for metrics collection..."
sleep 120

# Query recent CPU metrics
az monitor metrics list \
  --resource "$VM_ID" \
  --metric "Percentage CPU" \
  --start-time "$(date -u -d '10 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT1M \
  --aggregation Average Maximum \
  --output table
```

---

## Step 15 – Check Alert Status

```bash
# List fired alerts
az monitor metrics alert list \
  --resource-group "$RG_NAME" \
  --output table

# Check alert history
az monitor activity-log list \
  --resource-group "$RG_NAME" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --query "[?contains(operationName.value, 'Alert')].{Time:eventTimestamp, Operation:operationName.localizedValue, Status:status.localizedValue}" \
  --output table
```

---

## Step 16 – View Network Metrics

```bash
# Query network in metrics
az monitor metrics list \
  --resource "$VM_ID" \
  --metric "Network In Total" \
  --start-time "$(date -u -d '30 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT5M \
  --aggregation Total \
  --output table

# Query network out metrics
az monitor metrics list \
  --resource "$VM_ID" \
  --metric "Network Out Total" \
  --start-time "$(date -u -d '30 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT5M \
  --aggregation Total \
  --output table
```

---

## Step 17 – Cleanup

```bash
# Delete dashboard
az portal dashboard delete \
  --name "$DASHBOARD_NAME" \
  --resource-group "$RG_NAME" \
  --yes

# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f dashboard.json

echo "Cleanup complete"
```

---

## Summary

You created a virtual machine, configured Azure Monitor metrics collection, created custom dashboard, set up metric alert rules with email notifications, generated CPU load to trigger alerts, and monitored resource metrics.
