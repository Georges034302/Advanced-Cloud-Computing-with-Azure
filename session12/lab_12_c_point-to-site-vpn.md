# Lab 12.C: Point-to-Site VPN

## Objectives
- Create virtual network with gateway subnet
- Generate self-signed root certificate
- Create client certificate
- Deploy VPN gateway with P2S configuration
- Configure VPN client address pool
- Download and test VPN client
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- OpenSSL installed for certificate generation
- Location: Australia East
- Note: VPN Gateway deployment takes 30-45 minutes

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab12c-p2s"
VNET_NAME="vnet-p2s"
SUBNET_NAME="subnet-workload"
GATEWAY_NAME="vpn-gateway-p2s"
GATEWAY_PIP="pip-gateway-p2s"
CLIENT_ADDRESS_POOL="172.16.0.0/24"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$VNET_NAME"
echo "$GATEWAY_NAME"
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

## Step 3 – Create Virtual Network

```bash
# Create VNet with workload subnet
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.30.0.0/16 \
  --subnet-name "$SUBNET_NAME" \
  --subnet-prefix 10.30.1.0/24

echo "Virtual network created"
```

---

## Step 4 – Create Gateway Subnet

```bash
# Create gateway subnet
az network vnet subnet create \
  --name "GatewaySubnet" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --address-prefix 10.30.255.0/27

echo "Gateway subnet created"
```

---

## Step 5 – Create Public IP for Gateway

```bash
# Create public IP for VPN gateway
az network public-ip create \
  --name "$GATEWAY_PIP" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --allocation-method Dynamic \
  --sku Basic

echo "Public IP created"
```

---

## Step 6 – Create Certificate Directory

```bash
# Create directory for certificates
mkdir -p vpn-certs
cd vpn-certs

echo "Certificate directory created"
```

---

## Step 7 – Generate Root Certificate

```bash
# Generate root certificate private key
openssl genrsa -out rootCA.key 2048

# Generate root certificate
openssl req -x509 -new -nodes -key rootCA.key \
  -sha256 -days 1024 -out rootCA.crt \
  -subj "/C=AU/ST=NSW/L=Sydney/O=Lab/OU=IT/CN=P2S-Root-Cert"

echo "Root certificate generated"
```

---

## Step 8 – Extract Root Certificate Data

```bash
# Extract base64 encoded certificate (remove header/footer)
ROOT_CERT_DATA=$(openssl x509 -in rootCA.crt -outform PEM | \
  grep -v "BEGIN CERTIFICATE" | \
  grep -v "END CERTIFICATE" | \
  tr -d '\n')

echo "Root certificate data extracted"
```

---

## Step 9 – Generate Client Certificate

```bash
# Generate client certificate private key
openssl genrsa -out client.key 2048

# Create certificate signing request
openssl req -new -key client.key -out client.csr \
  -subj "/C=AU/ST=NSW/L=Sydney/O=Lab/OU=IT/CN=VPN-Client"

# Sign client certificate with root CA
openssl x509 -req -in client.csr -CA rootCA.crt -CAkey rootCA.key \
  -CAcreateserial -out client.crt -days 365 -sha256

echo "Client certificate generated"
```

---

## Step 10 – Create Client PFX Certificate

```bash
# Read PFX password securely
read -s -p "Enter password for client certificate: " CERT_PASSWORD
echo

# Create PFX file for client
openssl pkcs12 -export -out client.pfx \
  -inkey client.key -in client.crt -certfile rootCA.crt \
  -password pass:"$CERT_PASSWORD"

echo "Client PFX certificate created"
```

---

## Step 11 – Return to Working Directory

```bash
# Return to parent directory
cd ..

echo "$(pwd)"
```

---

## Step 12 – Create VPN Gateway

```bash
# Create VPN gateway (takes 30-45 minutes)
echo "Creating VPN gateway (this will take 30-45 minutes)..."
az network vnet-gateway create \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet "$VNET_NAME" \
  --public-ip-address "$GATEWAY_PIP" \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1 \
  --no-wait

echo "Gateway deployment started"
```

---

## Step 13 – Wait for Gateway Deployment

```bash
# Wait for gateway deployment to complete
echo "Waiting for gateway deployment to complete..."
az network vnet-gateway wait \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --created

echo "Gateway deployment complete"
```

---

## Step 14 – Verify Gateway Status

```bash
# Check gateway status
az network vnet-gateway show \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, ProvisioningState:provisioningState, GatewayType:gatewayType, VpnType:vpnType, SKU:sku.name}" \
  --output table
```

---

## Step 15 – Configure Point-to-Site

```bash
# Add root certificate to gateway
az network vnet-gateway root-cert create \
  --name "P2S-Root-Cert" \
  --gateway-name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --public-cert-data "$ROOT_CERT_DATA"

echo "Root certificate added to gateway"
```

---

## Step 16 – Set VPN Client Address Pool

```bash
# Configure P2S VPN with client address pool
az network vnet-gateway update \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --address-prefixes "$CLIENT_ADDRESS_POOL" \
  --client-protocol OpenVPN

echo "VPN client address pool configured"
```

---

## Step 17 – Get Gateway Public IP

```bash
# Get gateway public IP
GATEWAY_IP=$(az network public-ip show \
  --name "$GATEWAY_PIP" \
  --resource-group "$RG_NAME" \
  --query ipAddress \
  --output tsv)

echo "$GATEWAY_IP"
```

---

## Step 18 – List Root Certificates

```bash
# List all root certificates on gateway
az network vnet-gateway root-cert list \
  --gateway-name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, ProvisioningState:provisioningState}" \
  --output table
```

---

## Step 19 – Download VPN Client Package

```bash
# Generate VPN client configuration package URL
VPN_CLIENT_URL=$(az network vnet-gateway vpn-client generate \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --processor-architecture Amd64 \
  --output tsv)

echo "$VPN_CLIENT_URL"
```

---

## Step 20 – Download VPN Client Configuration

```bash
# Download VPN client package
curl -o vpn-client.zip "$VPN_CLIENT_URL"

echo "VPN client package downloaded"
```

---

## Step 21 – Extract VPN Client Package

```bash
# Extract VPN client configuration
unzip -q vpn-client.zip -d vpn-client

# List extracted contents
ls -la vpn-client/

echo "VPN client package extracted"
```

---

## Step 22 – Deploy Test VM

```bash
# Read VM admin password
read -s -p "Enter VM admin password: " ADMIN_PASSWORD
echo

# Create test VM in VNet
az vm create \
  --name "vm-internal" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_NAME" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username "azureuser" \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-address "" \
  --nsg ""

echo "Test VM created"
```

---

## Step 23 – Get VM Private IP

```bash
# Get private IP of test VM
VM_PRIVATE_IP=$(az vm show \
  --name "vm-internal" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query privateIps \
  --output tsv)

echo "$VM_PRIVATE_IP"
```

---

## Step 24 – Show P2S Configuration

```bash
# Show Point-to-Site configuration
az network vnet-gateway show \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --query "{VpnClientAddressPool:vpnClientConfiguration.vpnClientAddressPool.addressPrefixes, VpnClientProtocols:vpnClientConfiguration.vpnClientProtocols}" \
  --output table
```

---

## Step 25 – Verify Certificate Configuration

```bash
# Show certificate details
az network vnet-gateway root-cert show \
  --name "P2S-Root-Cert" \
  --gateway-name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, ProvisioningState:provisioningState}" \
  --output table
```

---

## Step 26 – Show Connection Instructions

```bash
# Display connection instructions
cat << EOF

Point-to-Site VPN Connection Instructions:
===========================================

1. Install the client certificate (client.pfx) on your device
   - Location: vpn-certs/client.pfx
   - Password: (the one you entered earlier)

2. Extract and install VPN client profile:
   - Windows: Run the installer from vpn-client/WindowsAmd64/
   - Linux: Use OpenVPN with config from vpn-client/OpenVPN/
   - Mac: Use config from vpn-client/Generic/

3. Connect using the VPN client

4. Once connected, you can access internal resources:
   - Test VM: $VM_PRIVATE_IP

Gateway Public IP: $GATEWAY_IP
Client Address Pool: $CLIENT_ADDRESS_POOL

EOF
```

---

## Step 27 – List VPN Client Protocols

```bash
# Show supported VPN client protocols
az network vnet-gateway show \
  --name "$GATEWAY_NAME" \
  --resource-group "$RG_NAME" \
  --query "vpnClientConfiguration.vpnClientProtocols" \
  --output table
```

---

## Step 28 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf vpn-certs vpn-client vpn-client.zip

echo "Cleanup complete"
```

---

## Summary

You created a virtual network with gateway subnet, generated self-signed root and client certificates using OpenSSL, deployed a VPN gateway with Point-to-Site configuration, configured client address pool for remote users, uploaded root certificate for authentication, downloaded VPN client configuration package, and deployed an internal VM accessible only through the VPN tunnel.
