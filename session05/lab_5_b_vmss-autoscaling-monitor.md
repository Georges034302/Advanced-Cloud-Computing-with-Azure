# Lab 5.B: VMSS Autoscaling with Monitor

## Overview
This lab demonstrates Virtual Machine Scale Sets with autoscaling based on CPU and schedule. You'll create a VMSS, configure autoscaling rules, trigger scale-out, and monitor scaling events.

---

## Objectives
- Create Virtual Machine Scale Set
- Configure CPU-based autoscaling rules
- Configure schedule-based autoscaling
- Monitor scaling with Azure Monitor
- Trigger scale-out using CPU load
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Basic understanding of autoscaling
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab5b-vmss"
VMSS_NAME="vmss-web"
VNET_NAME="vnet-vmss"
SUBNET_NAME="subnet-vmss"
LB_NAME="lb-vmss"
ADMIN_USER="azureuser"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$VMSS_NAME"
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

## Step 3 – Create Virtual Machine Scale Set

```bash
# Read SSH password from user
read -s -p "Enter VMSS password: " ADMIN_PW
echo ""

# Create VMSS with load balancer
az vmss create \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --image "Ubuntu2204" \
  --vm-sku "Standard_B2s" \
  --instance-count 2 \
  --admin-username "$ADMIN_USER" \
  --admin-password "$ADMIN_PW" \
  --authentication-type password \
  --lb "$LB_NAME" \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_NAME" \
  --backend-pool-name "vmss-backend" \
  --upgrade-policy-mode Automatic

# Verify VMSS creation
az vmss show \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Capacity:sku.capacity, UpgradePolicy:upgradePolicy.mode}" \
  --output table
```

---

## Step 4 – Install Web Server on VMSS Instances

```bash
# Create custom script extension to install nginx
az vmss extension set \
  --vmss-name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --version 2.1 \
  --settings '{
    "commandToExecute": "apt-get update && apt-get install -y nginx stress && echo \"<html><body><h1>VMSS Instance: $(hostname)</h1></body></html>\" > /var/www/html/index.html && systemctl restart nginx"
  }'

# Update VMSS instances
az vmss update-instances \
  --instance-ids "*" \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME"

echo "Web servers installed"
```

---

## Step 5 – Configure Load Balancer NAT Rules

```bash
# Create inbound NAT rule for HTTP
az network lb rule create \
  --name "http-rule" \
  --resource-group "$RG_NAME" \
  --lb-name "$LB_NAME" \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name "LoadBalancerFrontEnd" \
  --backend-pool-name "vmss-backend"

# Create health probe
az network lb probe create \
  --name "http-probe" \
  --resource-group "$RG_NAME" \
  --lb-name "$LB_NAME" \
  --protocol Http \
  --port 80 \
  --path "/"

# Update load balancing rule with probe
az network lb rule update \
  --name "http-rule" \
  --resource-group "$RG_NAME" \
  --lb-name "$LB_NAME" \
  --probe-name "http-probe"
```

---

## Step 6 – Get Load Balancer Public IP

```bash
# Get public IP address
LB_PUBLIC_IP=$(az network public-ip show \
  --name "${LB_NAME}PublicIP" \
  --resource-group "$RG_NAME" \
  --query ipAddress \
  --output tsv)

echo "$LB_PUBLIC_IP"

# Test web server
echo "Testing VMSS web servers:"
for i in {1..4}; do
  curl -s "http://${LB_PUBLIC_IP}" | grep -o '<h1>.*</h1>'
  sleep 1
done
```

---

## Step 7 – Configure CPU-Based Autoscaling

```bash
# Create autoscale profile with CPU-based rules
az monitor autoscale create \
  --name "vmss-autoscale" \
  --resource-group "$RG_NAME" \
  --resource "$VMSS_NAME" \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --min-count 2 \
  --max-count 5 \
  --count 2

# Create scale-out rule (CPU > 70%)
az monitor autoscale rule create \
  --resource-group "$RG_NAME" \
  --autoscale-name "vmss-autoscale" \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 1

# Create scale-in rule (CPU < 30%)
az monitor autoscale rule create \
  --resource-group "$RG_NAME" \
  --autoscale-name "vmss-autoscale" \
  --condition "Percentage CPU < 30 avg 5m" \
  --scale in 1

echo "CPU-based autoscaling configured"
```

---

## Step 8 – Configure Schedule-Based Autoscaling

```bash
# Create schedule-based profile for business hours
az monitor autoscale profile create \
  --name "BusinessHours" \
  --resource-group "$RG_NAME" \
  --autoscale-name "vmss-autoscale" \
  --min-count 3 \
  --max-count 5 \
  --count 3 \
  --timezone "AUS Eastern Standard Time" \
  --start "09:00" \
  --end "17:00" \
  --recurrence week Mon Tue Wed Thu Fri

# Create schedule-based profile for off-hours
az monitor autoscale profile create \
  --name "OffHours" \
  --resource-group "$RG_NAME" \
  --autoscale-name "vmss-autoscale" \
  --min-count 2 \
  --max-count 3 \
  --count 2 \
  --timezone "AUS Eastern Standard Time" \
  --start "17:00" \
  --end "09:00" \
  --recurrence week Mon Tue Wed Thu Fri

echo "Schedule-based autoscaling configured"
```

---

## Step 9 – View Autoscale Configuration

```bash
# Show autoscale settings
az monitor autoscale show \
  --name "vmss-autoscale" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, MinCount:profiles[0].capacity.minimum, MaxCount:profiles[0].capacity.maximum}" \
  --output table

# List autoscale rules
az monitor autoscale rule list \
  --resource-group "$RG_NAME" \
  --autoscale-name "vmss-autoscale" \
  --query "[].{MetricName:metricTrigger.metricName, Operator:metricTrigger.operator, Threshold:metricTrigger.threshold, ScaleAction:scaleAction.direction}" \
  --output table
```

---

## Step 10 – Trigger Scale-Out with CPU Load

```bash
# Get list of VMSS instance IDs
INSTANCE_IDS=$(az vmss list-instances \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].instanceId" \
  --output tsv)

echo "Current instances:"
echo "$INSTANCE_IDS"

# Generate CPU load on all instances
for INSTANCE_ID in $INSTANCE_IDS; do
  az vmss run-command invoke \
    --name "$VMSS_NAME" \
    --resource-group "$RG_NAME" \
    --instance-id "$INSTANCE_ID" \
    --command-id RunShellScript \
    --scripts "nohup stress --cpu 2 --timeout 600 > /dev/null 2>&1 &" \
    --no-wait
  
  echo "CPU load started on instance ${INSTANCE_ID}"
done

echo "CPU stress test running for 10 minutes"
```

---

## Step 11 – Monitor Autoscaling Activity

```bash
# Check current instance count
echo "Monitoring instance count (checking every 60 seconds):"
for i in {1..10}; do
  CURRENT_COUNT=$(az vmss show \
    --name "$VMSS_NAME" \
    --resource-group "$RG_NAME" \
    --query "sku.capacity" \
    --output tsv)
  
  echo "Time: $(date +%H:%M:%S) - Instances: ${CURRENT_COUNT}"
  sleep 60
done
```

---

## Step 12 – View Autoscale History

```bash
# Get autoscale activity log
az monitor activity-log list \
  --resource-group "$RG_NAME" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --query "[?contains(operationName.value, 'autoscale')].{Time:eventTimestamp, Operation:operationName.localizedValue, Status:status.localizedValue}" \
  --output table
```

---

## Step 13 – Monitor VMSS Metrics

```bash
# Get VMSS resource ID
VMSS_ID=$(az vmss show \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$VMSS_ID"

# Get CPU metrics
az monitor metrics list \
  --resource "$VMSS_ID" \
  --metric "Percentage CPU" \
  --start-time "$(date -u -d '30 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT1M \
  --aggregation Average \
  --output table
```

---

## Step 14 – Manually Scale VMSS

```bash
# Manually scale to specific instance count
az vmss scale \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --new-capacity 3

echo "Manual scaling to 3 instances"

# Wait for scaling to complete
az vmss wait \
  --updated \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --interval 10

# Verify instance count
az vmss show \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --query "sku.capacity" \
  --output tsv
```

---

## Step 15 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

echo "Cleanup complete"
```

---

## Summary

You created a Virtual Machine Scale Set with CPU-based and schedule-based autoscaling, triggered scale-out with CPU load, and monitored scaling activity with Azure Monitor.
