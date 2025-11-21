# Lab 12.A: VNet Peering

## Objectives
- Create two virtual networks in different regions
- Deploy virtual machines in each VNet
- Configure VNet peering between networks
- Test cross-VNet connectivity
- Verify network isolation
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- SSH client installed
- Location: Australia East and Australia Southeast

---

## Step 1 – Set Variables

```bash
# Set Azure regions and resource naming
LOCATION1="australiaeast"
LOCATION2="australiasoutheast"
RG_NAME="rg-lab12a-peering"
VNET1_NAME="vnet-east"
VNET2_NAME="vnet-southeast"
SUBNET1_NAME="subnet-east"
SUBNET2_NAME="subnet-southeast"
VM1_NAME="vm-east"
VM2_NAME="vm-southeast"
ADMIN_USER="azureuser"

# Display configuration
echo "$LOCATION1"
echo "$LOCATION2"
echo "$RG_NAME"
echo "$VNET1_NAME"
echo "$VNET2_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION1"
```

---

## Step 3 – Create First Virtual Network

```bash
# Create VNet in Australia East
az network vnet create \
  --name "$VNET1_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION1" \
  --address-prefix 10.1.0.0/16 \
  --subnet-name "$SUBNET1_NAME" \
  --subnet-prefix 10.1.1.0/24

echo "VNet 1 created in $LOCATION1"
```

---

## Step 4 – Create Second Virtual Network

```bash
# Create VNet in Australia Southeast
az network vnet create \
  --name "$VNET2_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION2" \
  --address-prefix 10.2.0.0/16 \
  --subnet-name "$SUBNET2_NAME" \
  --subnet-prefix 10.2.1.0/24

echo "VNet 2 created in $LOCATION2"
```

---

## Step 5 – Get Admin Password

```bash
# Read admin password securely
read -s -p "Enter admin password: " ADMIN_PASSWORD
echo
```

---

## Step 6 – Create First Virtual Machine

```bash
# Create VM in first VNet
az vm create \
  --name "$VM1_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION1" \
  --vnet-name "$VNET1_NAME" \
  --subnet "$SUBNET1_NAME" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username "$ADMIN_USER" \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-address "${VM1_NAME}-pip" \
  --nsg "${VM1_NAME}-nsg"

echo "VM 1 created"
```

---

## Step 7 – Get First VM Private IP

```bash
# Get private IP of first VM
VM1_PRIVATE_IP=$(az vm show \
  --name "$VM1_NAME" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query privateIps \
  --output tsv)

echo "$VM1_PRIVATE_IP"
```

---

## Step 8 – Create Second Virtual Machine

```bash
# Create VM in second VNet
az vm create \
  --name "$VM2_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION2" \
  --vnet-name "$VNET2_NAME" \
  --subnet "$SUBNET2_NAME" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username "$ADMIN_USER" \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-address "${VM2_NAME}-pip" \
  --nsg "${VM2_NAME}-nsg"

echo "VM 2 created"
```

---

## Step 9 – Get Second VM Private IP

```bash
# Get private IP of second VM
VM2_PRIVATE_IP=$(az vm show \
  --name "$VM2_NAME" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query privateIps \
  --output tsv)

echo "$VM2_PRIVATE_IP"
```

---

## Step 10 – Configure NSG Rules

```bash
# Allow SSH on first NSG
az network nsg rule create \
  --name AllowSSH \
  --nsg-name "${VM1_NAME}-nsg" \
  --resource-group "$RG_NAME" \
  --priority 1000 \
  --source-address-prefixes '*' \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow

# Allow SSH on second NSG
az network nsg rule create \
  --name AllowSSH \
  --nsg-name "${VM2_NAME}-nsg" \
  --resource-group "$RG_NAME" \
  --priority 1000 \
  --source-address-prefixes '*' \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow

echo "NSG rules configured"
```

---

## Step 11 – Test Connectivity Before Peering

```bash
# Get public IP of second VM
VM2_PUBLIC_IP=$(az vm show \
  --name "$VM2_NAME" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "$VM2_PUBLIC_IP"

# Try to ping VM1 from VM2 (should fail - no peering yet)
echo "Testing connectivity before peering (should timeout)..."
ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 "${ADMIN_USER}@${VM2_PUBLIC_IP}" \
  "ping -c 3 -W 2 $VM1_PRIVATE_IP" 2>/dev/null || echo "No connectivity - as expected"
```

---

## Step 12 – Create VNet Peering (East to Southeast)

```bash
# Get VNet resource IDs
VNET1_ID=$(az network vnet show \
  --name "$VNET1_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

VNET2_ID=$(az network vnet show \
  --name "$VNET2_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$VNET1_ID"
echo "$VNET2_ID"

# Create peering from VNet1 to VNet2
az network vnet peering create \
  --name "peer-east-to-southeast" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET1_NAME" \
  --remote-vnet "$VNET2_ID" \
  --allow-vnet-access

echo "Peering created: East to Southeast"
```

---

## Step 13 – Create VNet Peering (Southeast to East)

```bash
# Create peering from VNet2 to VNet1
az network vnet peering create \
  --name "peer-southeast-to-east" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET2_NAME" \
  --remote-vnet "$VNET1_ID" \
  --allow-vnet-access

echo "Peering created: Southeast to East"
```

---

## Step 14 – Verify Peering Status

```bash
# Check peering status on VNet1
az network vnet peering show \
  --name "peer-east-to-southeast" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET1_NAME" \
  --query "{Name:name, Status:peeringState, RemoteVNet:remoteVirtualNetwork.id}" \
  --output table

# Check peering status on VNet2
az network vnet peering show \
  --name "peer-southeast-to-east" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET2_NAME" \
  --query "{Name:name, Status:peeringState, RemoteVNet:remoteVirtualNetwork.id}" \
  --output table
```

---

## Step 15 – Test Connectivity After Peering

```bash
# Test ping from VM2 to VM1 (should succeed now)
echo "Testing connectivity after peering..."
ssh -o StrictHostKeyChecking=no "${ADMIN_USER}@${VM2_PUBLIC_IP}" \
  "ping -c 4 $VM1_PRIVATE_IP"

echo "Connectivity test complete"
```

---

## Step 16 – Test Reverse Connectivity

```bash
# Get public IP of first VM
VM1_PUBLIC_IP=$(az vm show \
  --name "$VM1_NAME" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "$VM1_PUBLIC_IP"

# Test ping from VM1 to VM2
echo "Testing reverse connectivity..."
ssh -o StrictHostKeyChecking=no "${ADMIN_USER}@${VM1_PUBLIC_IP}" \
  "ping -c 4 $VM2_PRIVATE_IP"

echo "Reverse connectivity verified"
```

---

## Step 17 – Install Web Server on VM1

```bash
# Install nginx on VM1
ssh -o StrictHostKeyChecking=no "${ADMIN_USER}@${VM1_PUBLIC_IP}" << 'EOF'
sudo apt-get update -y
sudo apt-get install -y nginx
echo "<h1>VM in Australia East</h1>" | sudo tee /var/www/html/index.html
sudo systemctl start nginx
sudo systemctl enable nginx
EOF

echo "Web server installed on VM1"
```

---

## Step 18 – Configure NSG for HTTP

```bash
# Allow HTTP on VM1 NSG
az network nsg rule create \
  --name AllowHTTP \
  --nsg-name "${VM1_NAME}-nsg" \
  --resource-group "$RG_NAME" \
  --priority 1010 \
  --source-address-prefixes '*' \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow

echo "HTTP allowed on VM1"
```

---

## Step 19 – Test HTTP Across Peering

```bash
# Test HTTP from VM2 to VM1 private IP
echo "Testing HTTP across VNet peering..."
ssh -o StrictHostKeyChecking=no "${ADMIN_USER}@${VM2_PUBLIC_IP}" \
  "curl -s http://$VM1_PRIVATE_IP"

echo "HTTP test complete"
```

---

## Step 20 – List All Peerings

```bash
# List all peerings in VNet1
az network vnet peering list \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET1_NAME" \
  --query "[].{Name:name, State:peeringState, AllowAccess:allowVirtualNetworkAccess}" \
  --output table

# List all peerings in VNet2
az network vnet peering list \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET2_NAME" \
  --query "[].{Name:name, State:peeringState, AllowAccess:allowVirtualNetworkAccess}" \
  --output table
```

---

## Step 21 – Verify Address Spaces

```bash
# Show VNet address spaces
az network vnet list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Location:location, AddressSpace:addressSpace.addressPrefixes[0]}" \
  --output table
```

---

## Step 22 – Test Network Isolation

```bash
# Create test VM in VNet1 different subnet to verify isolation
az network vnet subnet create \
  --name "subnet-isolated" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET1_NAME" \
  --address-prefix 10.1.2.0/24

echo "Isolated subnet created"
```

---

## Step 23 – Show Effective Routes

```bash
# Get NIC name for VM2
VM2_NIC=$(az vm show \
  --name "$VM2_NAME" \
  --resource-group "$RG_NAME" \
  --query "networkProfile.networkInterfaces[0].id" \
  --output tsv | grep -o '[^/]*$')

echo "$VM2_NIC"

# Show effective routes
az network nic show-effective-route-table \
  --name "$VM2_NIC" \
  --resource-group "$RG_NAME" \
  --output table
```

---

## Step 24 – Cleanup

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

You created two virtual networks in different Azure regions with non-overlapping address spaces, deployed virtual machines in each network, configured bidirectional VNet peering to enable private connectivity, verified cross-region communication using ping and HTTP, and examined effective routes showing peering configuration.
