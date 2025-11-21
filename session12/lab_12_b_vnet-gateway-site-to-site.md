# Lab 12.B: Site-to-Site VPN Gateway

## Objectives
- Create two virtual networks simulating different sites
- Deploy VPN gateways in each VNet
- Configure site-to-site VPN connection
- Establish encrypted tunnel between sites
- Test connectivity across VPN
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Location: Australia East
- Note: VPN Gateway deployment takes 30-45 minutes

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab12b-vpn"
VNET1_NAME="vnet-site1"
VNET2_NAME="vnet-site2"
SUBNET1_NAME="subnet-site1"
SUBNET2_NAME="subnet-site2"
GATEWAY1_NAME="vpn-gateway-site1"
GATEWAY2_NAME="vpn-gateway-site2"
GATEWAY1_PIP="pip-gateway-site1"
GATEWAY2_PIP="pip-gateway-site2"
CONNECTION1_NAME="site1-to-site2"
CONNECTION2_NAME="site2-to-site1"

# Display configuration
echo "$LOCATION"
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
  --location "$LOCATION"
```

---

## Step 3 – Create First Virtual Network

```bash
# Create VNet for Site 1
az network vnet create \
  --name "$VNET1_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.10.0.0/16 \
  --subnet-name "$SUBNET1_NAME" \
  --subnet-prefix 10.10.1.0/24

echo "Site 1 VNet created"
```

---

## Step 4 – Create Gateway Subnet for Site 1

```bash
# Create gateway subnet in Site 1 VNet
az network vnet subnet create \
  --name "GatewaySubnet" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET1_NAME" \
  --address-prefix 10.10.255.0/27

echo "Gateway subnet created for Site 1"
```

---

## Step 5 – Create Second Virtual Network

```bash
# Create VNet for Site 2
az network vnet create \
  --name "$VNET2_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.20.0.0/16 \
  --subnet-name "$SUBNET2_NAME" \
  --subnet-prefix 10.20.1.0/24

echo "Site 2 VNet created"
```

---

## Step 6 – Create Gateway Subnet for Site 2

```bash
# Create gateway subnet in Site 2 VNet
az network vnet subnet create \
  --name "GatewaySubnet" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET2_NAME" \
  --address-prefix 10.20.255.0/27

echo "Gateway subnet created for Site 2"
```

---

## Step 7 – Create Public IP for First Gateway

```bash
# Create public IP for Site 1 gateway
az network public-ip create \
  --name "$GATEWAY1_PIP" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --allocation-method Dynamic \
  --sku Basic

echo "Public IP created for Site 1 gateway"
```

---

## Step 8 – Create Public IP for Second Gateway

```bash
# Create public IP for Site 2 gateway
az network public-ip create \
  --name "$GATEWAY2_PIP" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --allocation-method Dynamic \
  --sku Basic

echo "Public IP created for Site 2 gateway"
```

---

## Step 9 – Create First VPN Gateway

```bash
# Create VPN gateway for Site 1 (takes 30-45 minutes)
echo "Creating Site 1 VPN gateway (this will take 30-45 minutes)..."
az network vnet-gateway create \
  --name "$GATEWAY1_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet "$VNET1_NAME" \
  --public-ip-address "$GATEWAY1_PIP" \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1 \
  --no-wait

echo "Site 1 gateway deployment started"
```

---

## Step 10 – Create Second VPN Gateway

```bash
# Create VPN gateway for Site 2 (takes 30-45 minutes)
echo "Creating Site 2 VPN gateway (this will take 30-45 minutes)..."
az network vnet-gateway create \
  --name "$GATEWAY2_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet "$VNET2_NAME" \
  --public-ip-address "$GATEWAY2_PIP" \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1 \
  --no-wait

echo "Site 2 gateway deployment started"
```

---

## Step 11 – Wait for Gateway Deployments

```bash
# Wait for Site 1 gateway to complete
echo "Waiting for Site 1 gateway deployment..."
az network vnet-gateway wait \
  --name "$GATEWAY1_NAME" \
  --resource-group "$RG_NAME" \
  --created

echo "Site 1 gateway ready"

# Wait for Site 2 gateway to complete
echo "Waiting for Site 2 gateway deployment..."
az network vnet-gateway wait \
  --name "$GATEWAY2_NAME" \
  --resource-group "$RG_NAME" \
  --created

echo "Site 2 gateway ready"
```

---

## Step 12 – Verify Gateway Status

```bash
# Check Site 1 gateway status
az network vnet-gateway show \
  --name "$GATEWAY1_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, ProvisioningState:provisioningState, GatewayType:gatewayType, VpnType:vpnType}" \
  --output table

# Check Site 2 gateway status
az network vnet-gateway show \
  --name "$GATEWAY2_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, ProvisioningState:provisioningState, GatewayType:gatewayType, VpnType:vpnType}" \
  --output table
```

---

## Step 13 – Get Gateway Public IPs

```bash
# Get Site 1 gateway public IP
GATEWAY1_IP=$(az network public-ip show \
  --name "$GATEWAY1_PIP" \
  --resource-group "$RG_NAME" \
  --query ipAddress \
  --output tsv)

echo "$GATEWAY1_IP"

# Get Site 2 gateway public IP
GATEWAY2_IP=$(az network public-ip show \
  --name "$GATEWAY2_PIP" \
  --resource-group "$RG_NAME" \
  --query ipAddress \
  --output tsv)

echo "$GATEWAY2_IP"
```

---

## Step 14 – Get Shared Key for VPN

```bash
# Read shared key securely
read -s -p "Enter shared key for VPN (min 8 chars): " SHARED_KEY
echo
```

---

## Step 15 – Create Site 1 to Site 2 Connection

```bash
# Get Site 2 gateway resource ID
GATEWAY2_ID=$(az network vnet-gateway show \
  --name "$GATEWAY2_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$GATEWAY2_ID"

# Create connection from Site 1 to Site 2
az network vpn-connection create \
  --name "$CONNECTION1_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet-gateway1 "$GATEWAY1_NAME" \
  --vnet-gateway2 "$GATEWAY2_ID" \
  --shared-key "$SHARED_KEY" \
  --connection-type Vnet2Vnet

echo "Connection created: Site 1 to Site 2"
```

---

## Step 16 – Create Site 2 to Site 1 Connection

```bash
# Get Site 1 gateway resource ID
GATEWAY1_ID=$(az network vnet-gateway show \
  --name "$GATEWAY1_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$GATEWAY1_ID"

# Create connection from Site 2 to Site 1
az network vpn-connection create \
  --name "$CONNECTION2_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet-gateway1 "$GATEWAY2_NAME" \
  --vnet-gateway2 "$GATEWAY1_ID" \
  --shared-key "$SHARED_KEY" \
  --connection-type Vnet2Vnet

echo "Connection created: Site 2 to Site 1"
```

---

## Step 17 – Verify Connection Status

```bash
# Check Site 1 to Site 2 connection
az network vpn-connection show \
  --name "$CONNECTION1_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, ConnectionStatus:connectionStatus, ConnectionType:connectionType}" \
  --output table

# Check Site 2 to Site 1 connection
az network vpn-connection show \
  --name "$CONNECTION2_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, ConnectionStatus:connectionStatus, ConnectionType:connectionType}" \
  --output table
```

---

## Step 18 – Deploy Test VM in Site 1

```bash
# Read admin password
read -s -p "Enter VM admin password: " ADMIN_PASSWORD
echo

# Create VM in Site 1
az vm create \
  --name "vm-site1" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet-name "$VNET1_NAME" \
  --subnet "$SUBNET1_NAME" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username "azureuser" \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-address "pip-vm-site1" \
  --nsg "nsg-vm-site1"

echo "VM created in Site 1"
```

---

## Step 19 – Get Site 1 VM Private IP

```bash
# Get private IP of Site 1 VM
VM1_PRIVATE_IP=$(az vm show \
  --name "vm-site1" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query privateIps \
  --output tsv)

echo "$VM1_PRIVATE_IP"
```

---

## Step 20 – Deploy Test VM in Site 2

```bash
# Create VM in Site 2
az vm create \
  --name "vm-site2" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet-name "$VNET2_NAME" \
  --subnet "$SUBNET2_NAME" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username "azureuser" \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-address "pip-vm-site2" \
  --nsg "nsg-vm-site2"

echo "VM created in Site 2"
```

---

## Step 21 – Get Site 2 VM Private IP

```bash
# Get private IP of Site 2 VM
VM2_PRIVATE_IP=$(az vm show \
  --name "vm-site2" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query privateIps \
  --output tsv)

echo "$VM2_PRIVATE_IP"
```

---

## Step 22 – Configure NSG Rules

```bash
# Allow SSH on Site 1 VM
az network nsg rule create \
  --name AllowSSH \
  --nsg-name "nsg-vm-site1" \
  --resource-group "$RG_NAME" \
  --priority 1000 \
  --source-address-prefixes '*' \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow

# Allow SSH on Site 2 VM
az network nsg rule create \
  --name AllowSSH \
  --nsg-name "nsg-vm-site2" \
  --resource-group "$RG_NAME" \
  --priority 1000 \
  --source-address-prefixes '*' \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow

echo "NSG rules configured"
```

---

## Step 23 – Test Connectivity Across VPN

```bash
# Get public IP of Site 2 VM
VM2_PUBLIC_IP=$(az vm show \
  --name "vm-site2" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "$VM2_PUBLIC_IP"

# Test ping from Site 2 to Site 1 across VPN tunnel
echo "Testing connectivity across VPN tunnel..."
ssh -o StrictHostKeyChecking=no "azureuser@${VM2_PUBLIC_IP}" \
  "ping -c 4 $VM1_PRIVATE_IP"

echo "VPN connectivity verified"
```

---

## Step 24 – Check Connection Statistics

```bash
# Get connection statistics for Site 1 to Site 2
az network vpn-connection show \
  --name "$CONNECTION1_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Status:connectionStatus, EgressBytes:egressBytesTransferred, IngressBytes:ingressBytesTransferred}" \
  --output table
```

---

## Step 25 – List All VPN Connections

```bash
# List all VPN connections
az network vpn-connection list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Status:connectionStatus, Type:connectionType}" \
  --output table
```

---

## Step 26 – Show Gateway Configuration

```bash
# Show Site 1 gateway configuration
az network vnet-gateway show \
  --name "$GATEWAY1_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, SKU:sku.name, VpnType:vpnType, ActiveActive:activeActive, BgpEnabled:enableBgp}" \
  --output table
```

---

## Step 27 – Cleanup

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

You created two virtual networks simulating different sites, deployed VPN gateways in each network with gateway subnets and public IPs, configured bidirectional site-to-site VPN connections with shared key authentication, established encrypted IPsec tunnels between sites, deployed test virtual machines, and verified connectivity across the VPN tunnel.
