# Lab 8.C: NSG Flow Logs

## Objectives
- Create virtual network with NSG
- Configure NSG flow logs
- Create virtual machine
- Generate network traffic
- Analyze flow logs
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
RG_NAME="rg-lab8c-flowlogs"
VNET_NAME="vnet-flowlogs"
SUBNET_NAME="subnet-vms"
NSG_NAME="nsg-flowlogs"
STORAGE_ACCOUNT="storage$RANDOM$RANDOM"
VM_NAME="vm-test"
ADMIN_USER="azureuser"
WATCHER_NAME="NetworkWatcher_${LOCATION}"
WATCHER_RG="NetworkWatcherRG"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$NSG_NAME"
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

## Step 3 – Create Network Security Group

```bash
# Create NSG
az network nsg create \
  --name "$NSG_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION"

# Get NSG ID
NSG_ID=$(az network nsg show \
  --name "$NSG_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$NSG_ID"
```

---

## Step 4 – Add NSG Rules

```bash
# Allow SSH
az network nsg rule create \
  --name "AllowSSH" \
  --nsg-name "$NSG_NAME" \
  --resource-group "$RG_NAME" \
  --priority 100 \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow

# Allow HTTP
az network nsg rule create \
  --name "AllowHTTP" \
  --nsg-name "$NSG_NAME" \
  --resource-group "$RG_NAME" \
  --priority 110 \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow

# Deny all other inbound
az network nsg rule create \
  --name "DenyAllInbound" \
  --nsg-name "$NSG_NAME" \
  --resource-group "$RG_NAME" \
  --priority 4096 \
  --destination-port-ranges "*" \
  --protocol "*" \
  --access Deny

echo "NSG rules created"
```

---

## Step 5 – Create Storage Account

```bash
# Create storage account for flow logs
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

## Step 6 – Enable Network Watcher

```bash
# Check if Network Watcher exists
if ! az network watcher show --location "$LOCATION" --resource-group "$WATCHER_RG" 2>/dev/null; then
  # Create Network Watcher resource group if it doesn't exist
  az group create --name "$WATCHER_RG" --location "$LOCATION"
  
  # Enable Network Watcher
  az network watcher configure \
    --locations "$LOCATION" \
    --enabled true \
    --resource-group "$WATCHER_RG"
fi

echo "Network Watcher enabled"
```

---

## Step 7 – Configure NSG Flow Logs

```bash
# Create NSG flow log
az network watcher flow-log create \
  --name "flowlog-${NSG_NAME}" \
  --location "$LOCATION" \
  --resource-group "$WATCHER_RG" \
  --nsg "$NSG_ID" \
  --storage-account "$STORAGE_ID" \
  --enabled true \
  --retention 7 \
  --format JSON \
  --log-version 2

echo "NSG flow logs configured"
```

---

## Step 8 – Create Virtual Network

```bash
# Create VNet
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.0.0.0/16

# Create subnet with NSG
az network vnet subnet create \
  --name "$SUBNET_NAME" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --address-prefix 10.0.1.0/24 \
  --network-security-group "$NSG_NAME"

echo "Virtual network created"
```

---

## Step 9 – Create Virtual Machine

```bash
# Read VM password from user
read -s -p "Enter VM password: " ADMIN_PASSWORD
echo ""

# Create VM
az vm create \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_NAME" \
  --image "Ubuntu2204" \
  --size "Standard_B1s" \
  --admin-username "$ADMIN_USER" \
  --admin-password "$ADMIN_PASSWORD" \
  --authentication-type password \
  --public-ip-sku Standard

echo "VM created"
```

---

## Step 10 – Get VM Public IP

```bash
# Get VM public IP
VM_IP=$(az vm show \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "$VM_IP"
```

---

## Step 11 – Install Web Server

```bash
# Install nginx on VM
az vm run-command invoke \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --command-id RunShellScript \
  --scripts "sudo apt-get update && sudo apt-get install -y nginx && sudo systemctl start nginx"

echo "Web server installed"
```

---

## Step 12 – Generate Network Traffic

```bash
# Generate HTTP traffic
echo "Generating HTTP traffic:"
for i in {1..10}; do
  curl -s "http://${VM_IP}/" > /dev/null && echo "Request $i successful" || echo "Request $i failed"
  sleep 1
done

# Try SSH connection (will succeed)
echo "Testing SSH connection:"
ssh-keyscan -H "$VM_IP" > /dev/null 2>&1 && echo "SSH port open" || echo "SSH port closed"

# Try denied port (will be blocked)
echo "Testing denied port:"
nc -zv "$VM_IP" 8080 2>&1 | grep -q "succeeded" && echo "Port 8080 open" || echo "Port 8080 blocked"

echo "Traffic generation complete"
```

---

## Step 13 – Wait for Flow Logs

```bash
# Wait for flow logs to be written
echo "Waiting for flow logs to be written..."
sleep 300
```

---

## Step 14 – Validate Flow Logs Storage

```bash
# List containers in storage account
az storage container list \
  --account-name "$STORAGE_ACCOUNT" \
  --query "[].name" \
  --output table

# Get storage connection string
STORAGE_CONNECTION=$(az storage account show-connection-string \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query connectionString \
  --output tsv)

# List blobs in flow log container
az storage blob list \
  --container-name "insights-logs-networksecuritygroupflowevent" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --query "[].name" \
  --output table
```

---

## Step 15 – View Flow Log Configuration

```bash
# Show flow log configuration
az network watcher flow-log show \
  --name "flowlog-${NSG_NAME}" \
  --location "$LOCATION" \
  --resource-group "$WATCHER_RG" \
  --query "{Name:name, Enabled:enabled, Retention:retentionPolicy.days, Format:format.type}" \
  --output table
```

---

## Step 16 – Download Flow Logs

```bash
# Create directory for logs
mkdir -p flow-logs

# Get latest flow log blob
LATEST_BLOB=$(az storage blob list \
  --container-name "insights-logs-networksecuritygroupflowevent" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --query "[-1].name" \
  --output tsv)

if [ -n "$LATEST_BLOB" ]; then
  # Download latest blob
  az storage blob download \
    --container-name "insights-logs-networksecuritygroupflowevent" \
    --account-name "$STORAGE_ACCOUNT" \
    --connection-string "$STORAGE_CONNECTION" \
    --name "$LATEST_BLOB" \
    --file flow-logs/latest.json
  
  echo "Flow log downloaded"
  
  # Display flow log sample
  echo "Flow log sample:"
  cat flow-logs/latest.json | python3 -m json.tool | head -50
else
  echo "No flow logs available yet"
fi
```

---

## Step 17 – Cleanup

```bash
# Disable flow logging
az network watcher flow-log delete \
  --name "flowlog-${NSG_NAME}" \
  --location "$LOCATION" \
  --resource-group "$WATCHER_RG"

# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf flow-logs

echo "Cleanup complete"
```

---

## Summary

You created a network security group with flow logging enabled, created a virtual machine with NSG attached, generated network traffic including allowed and denied connections, and analyzed NSG flow logs stored in blob storage.
