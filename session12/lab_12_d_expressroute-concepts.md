# Lab 12.D: ExpressRoute Concepts

## Objectives
- Understand ExpressRoute architecture
- Explore ExpressRoute circuit SKUs
- Create ExpressRoute circuit (simulation)
- Configure virtual network gateway for ExpressRoute
- Review peering configurations
- Understand connectivity models
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Location: Australia East
- Note: This lab explores concepts without activating paid ExpressRoute circuit

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab12d-expressroute"
VNET_NAME="vnet-expressroute"
SUBNET_NAME="subnet-workload"
GATEWAY_NAME="ergw-gateway"
GATEWAY_PIP="pip-ergw"
CIRCUIT_NAME="er-circuit-demo"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$VNET_NAME"
echo "$CIRCUIT_NAME"
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

## Step 3 – List Available ExpressRoute Providers

```bash
# List ExpressRoute service providers in Australia
az network express-route list-service-providers \
  --query "[?contains(peeringLocations[0], 'Sydney') || contains(peeringLocations[0], 'Melbourne')].{Provider:name, Locations:peeringLocations, Bandwidths:bandwidthsOffered[*].offerName}" \
  --output table
```

---

## Step 4 – List ExpressRoute Peering Locations

```bash
# Show available peering locations
az network express-route list-service-providers \
  --query "[].peeringLocations[]" \
  --output tsv | sort -u | grep -i australia
```

---

## Step 5 – Create Virtual Network

```bash
# Create VNet for ExpressRoute connection
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.40.0.0/16 \
  --subnet-name "$SUBNET_NAME" \
  --subnet-prefix 10.40.1.0/24

echo "Virtual network created"
```

---

## Step 6 – Create Gateway Subnet

```bash
# Create gateway subnet for ExpressRoute gateway
az network vnet subnet create \
  --name "GatewaySubnet" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --address-prefix 10.40.255.0/27

echo "Gateway subnet created"
```

---

## Step 7 – Create Public IP for Gateway

```bash
# Create public IP for ExpressRoute gateway
az network public-ip create \
  --name "$GATEWAY_PIP" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --allocation-method Dynamic \
  --sku Basic

echo "Public IP created"
```

---

## Step 8 – Show ExpressRoute Gateway SKUs

```bash
# Display information about ExpressRoute gateway SKUs
cat << 'EOF'

ExpressRoute Gateway SKUs:
==========================
Standard    - Up to 1 Gbps, 4 circuits
HighPerf    - Up to 2 Gbps, 10 circuits
UltraPerf   - Up to 10 Gbps, 16 circuits
ErGw1AZ     - Zone-redundant, 1 Gbps
ErGw2AZ     - Zone-redundant, 2 Gbps
ErGw3AZ     - Zone-redundant, 10 Gbps

EOF
```

---

## Step 9 – Create ExpressRoute Gateway

```bash
# Create ExpressRoute virtual network gateway
echo "Creating ExpressRoute gateway (takes 30-45 minutes)..."
az network vnet-gateway create \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet "$VNET_NAME" \
  --public-ip-address "$GATEWAY_PIP" \
  --gateway-type ExpressRoute \
  --sku Standard \
  --no-wait

echo "ExpressRoute gateway deployment started"
```

---

## Step 10 – Show ExpressRoute Circuit SKUs

```bash
# Display ExpressRoute circuit tier and family options
cat << 'EOF'

ExpressRoute Circuit SKUs:
==========================

Tier:
  - Standard: Connect to resources in same geopolitical region
  - Premium: Global connectivity, increased route limits

Family:
  - MeteredData: Pay per GB of outbound data
  - UnlimitedData: Unlimited outbound data transfer

Bandwidth Options:
  - 50 Mbps, 100 Mbps, 200 Mbps, 500 Mbps
  - 1 Gbps, 2 Gbps, 5 Gbps, 10 Gbps, 100 Gbps

EOF
```

---

## Step 11 – Create ExpressRoute Circuit

```bash
# Create ExpressRoute circuit (not activated, no charges)
az network express-route create \
  --name "$CIRCUIT_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --peering-location "Sydney" \
  --bandwidth 50 \
  --provider "Equinix" \
  --sku-family MeteredData \
  --sku-tier Standard \
  --allow-classic-operations false

echo "ExpressRoute circuit created (not provisioned)"
```

---

## Step 12 – Get Circuit Service Key

```bash
# Get service key for circuit provisioning
SERVICE_KEY=$(az network express-route show \
  --name "$CIRCUIT_NAME" \
  --resource-group "$RG_NAME" \
  --query serviceKey \
  --output tsv)

echo "$SERVICE_KEY"
```

---

## Step 13 – Show Circuit Details

```bash
# Display circuit information
az network express-route show \
  --name "$CIRCUIT_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, Bandwidth:serviceProviderProperties.bandwidthInMbps, Provider:serviceProviderProperties.serviceProviderName, PeeringLocation:serviceProviderProperties.peeringLocation, ServiceKey:serviceKey, ProvisioningState:serviceProviderProvisioningState}" \
  --output table
```

---

## Step 14 – Show Peering Types

```bash
# Display ExpressRoute peering types information
cat << 'EOF'

ExpressRoute Peering Types:
===========================

Private Peering:
  - Connect to Azure IaaS services (VMs, Storage)
  - Private IP connectivity to VNets
  - Supports bidirectional connectivity

Microsoft Peering:
  - Connect to Azure PaaS services (SQL, Storage public endpoints)
  - Connect to Microsoft 365 services
  - Uses public IP addresses

Note: Azure Public Peering is deprecated

EOF
```

---

## Step 15 – Configure Private Peering

```bash
# Create private peering configuration
az network express-route peering create \
  --circuit-name "$CIRCUIT_NAME" \
  --resource-group "$RG_NAME" \
  --peering-type AzurePrivatePeering \
  --peer-asn 65001 \
  --primary-peer-subnet 192.168.1.0/30 \
  --secondary-peer-subnet 192.168.2.0/30 \
  --vlan-id 100

echo "Private peering configured"
```

---

## Step 16 – Show Peering Configuration

```bash
# Display private peering details
az network express-route peering show \
  --circuit-name "$CIRCUIT_NAME" \
  --resource-group "$RG_NAME" \
  --name AzurePrivatePeering \
  --query "{PeeringType:peeringType, VlanId:vlanId, PrimarySubnet:primaryPeerAddressPrefix, SecondarySubnet:secondaryPeerAddressPrefix, State:state}" \
  --output table
```

---

## Step 17 – Wait for Gateway Deployment

```bash
# Wait for ExpressRoute gateway creation
echo "Waiting for gateway deployment (if running)..."
az network vnet-gateway wait \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --created \
  --timeout 3600 2>/dev/null || echo "Gateway deployment in progress"

echo "Gateway check complete"
```

---

## Step 18 – Show Gateway Configuration

```bash
# Display gateway details
az network vnet-gateway show \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, GatewayType:gatewayType, SKU:sku.name, ProvisioningState:provisioningState}" \
  --output table 2>/dev/null || echo "Gateway still provisioning"
```

---

## Step 19 – Show Connection Models

```bash
# Display ExpressRoute connectivity models
cat << 'EOF'

ExpressRoute Connectivity Models:
==================================

1. CloudExchange Co-location:
   - Facility with cloud exchange
   - Virtual cross-connection to Azure
   - Layer 2 and Layer 3 connectivity

2. Point-to-Point Ethernet:
   - Dedicated Ethernet connection
   - Direct connection to Azure
   - Typically Layer 3 MPLS

3. Any-to-Any (IPVPN):
   - Integrate Azure with WAN
   - Azure as another branch
   - MPLS VPN connectivity

4. ExpressRoute Direct:
   - Direct connection to Microsoft global network
   - 10 Gbps or 100 Gbps ports
   - Dual connectivity for high availability

EOF
```

---

## Step 20 – List ExpressRoute Circuits

```bash
# List all ExpressRoute circuits in resource group
az network express-route list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Bandwidth:serviceProviderProperties.bandwidthInMbps, Location:location, ProvisioningState:serviceProviderProvisioningState}" \
  --output table
```

---

## Step 21 – Show Circuit Peerings

```bash
# List all peerings on the circuit
az network express-route peering list \
  --circuit-name "$CIRCUIT_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Type:peeringType, State:state, VlanId:vlanId}" \
  --output table
```

---

## Step 22 – Show Bandwidth Options

```bash
# Query available bandwidth options
az network express-route list-service-providers \
  --query "[?name=='Equinix'].bandwidthsOffered[*].offerName" \
  --output table
```

---

## Step 23 – Show Circuit Stats

```bash
# Display circuit statistics
az network express-route show \
  --name "$CIRCUIT_NAME" \
  --resource-group "$RG_NAME" \
  --query "{CircuitName:name, ServiceKey:serviceKey, SKUTier:sku.tier, SKUFamily:sku.family, Bandwidth:serviceProviderProperties.bandwidthInMbps, AllowClassicOps:allowClassicOperations}" \
  --output table
```

---

## Step 24 – Show HA Architecture

```bash
# Display high availability information
cat << 'EOF'

ExpressRoute High Availability:
================================

Built-in Redundancy:
  - Dual physical connections to Microsoft edge routers
  - Primary and secondary circuits
  - Active-Active BGP sessions
  - 99.95% availability SLA

Best Practices:
  - Use zone-redundant gateway SKUs (ErGw1AZ, ErGw2AZ, ErGw3AZ)
  - Configure dual ExpressRoute circuits (different providers)
  - Use ExpressRoute with VPN as backup
  - Monitor circuit health with Network Performance Monitor

EOF
```

---

## Step 25 – Show Routing Requirements

```bash
# Display BGP routing requirements
cat << 'EOF'

ExpressRoute Routing Requirements:
===================================

BGP Configuration:
  - Public AS number or Private AS number (65001-65535)
  - /30 subnet for primary link
  - /30 subnet for secondary link
  - VLAN ID for each peering

Route Filters:
  - Control which routes advertised over Microsoft peering
  - Apply to specific services (Storage, SQL, etc.)
  - Reduce unnecessary route advertisements

Maximum Routes:
  - Standard: 4,000 routes (private), 200 routes (Microsoft)
  - Premium: 10,000 routes (private), 200 routes (Microsoft)

EOF
```

---

## Step 26 – Cleanup

```bash
# Delete ExpressRoute circuit
az network express-route delete \
  --name "$CIRCUIT_NAME" \
  --resource-group "$RG_NAME" \
  --yes

# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

echo "Cleanup complete"
```

---

## Summary

You explored ExpressRoute architecture and connectivity models, examined service providers and peering locations in Australia, created an ExpressRoute circuit with service key for provider provisioning, configured private peering with BGP settings, deployed an ExpressRoute virtual network gateway, reviewed SKU options for circuits and gateways, and learned about high availability design patterns and routing requirements for dedicated private connectivity to Azure.
