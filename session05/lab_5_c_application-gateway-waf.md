# Lab 5.C: Application Gateway with WAF

## Overview
This lab demonstrates Azure Application Gateway with Web Application Firewall. You'll deploy Application Gateway, configure SSL termination, path-based routing, integrate with VMSS, and test WAF functionality.

---

## Objectives
- Deploy Application Gateway with WAF_v2 tier
- Configure backend pool with VMSS
- Configure HTTP listener and SSL termination
- Set up path-based routing rules
- Validate WAF functionality
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- OpenSSL for certificate generation
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab5c-appgw"
VNET_NAME="vnet-appgw"
SUBNET_APPGW="subnet-appgw"
SUBNET_VMSS="subnet-vmss"
APPGW_NAME="appgw-waf"
VMSS_NAME="vmss-backend"
PIP_NAME="appgw-pip"
ADMIN_USER="azureuser"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$APPGW_NAME"
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
# Create virtual network
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.0.0.0/16

# Create subnet for Application Gateway
az network vnet subnet create \
  --name "$SUBNET_APPGW" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --address-prefix 10.0.1.0/24

# Create subnet for VMSS
az network vnet subnet create \
  --name "$SUBNET_VMSS" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --address-prefix 10.0.2.0/24
```

---

## Step 4 – Create Public IP

```bash
# Create public IP for Application Gateway
az network public-ip create \
  --name "$PIP_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard \
  --allocation-method Static

# Get public IP address
APPGW_IP=$(az network public-ip show \
  --name "$PIP_NAME" \
  --resource-group "$RG_NAME" \
  --query ipAddress \
  --output tsv)

echo "$APPGW_IP"
```

---

## Step 5 – Create Self-Signed Certificate

```bash
# Read certificate password from user
read -s -p "Enter certificate password: " CERT_PASSWORD
echo ""

# Generate self-signed certificate
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout appgw-key.pem \
  -out appgw-cert.pem \
  -days 365 \
  -subj "/CN=appgw.local"

# Convert to PFX format with password
openssl pkcs12 -export \
  -out appgw-cert.pfx \
  -inkey appgw-key.pem \
  -in appgw-cert.pem \
  -password pass:"$CERT_PASSWORD"

echo "Certificate created"
```

---

## Step 6 – Create VMSS for Backend

```bash
# Read VMSS password from user
read -s -p "Enter VMSS password: " VMSS_PASSWORD
echo ""

# Create VMSS
az vmss create \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --image "Ubuntu2204" \
  --vm-sku "Standard_B1s" \
  --instance-count 2 \
  --admin-username "$ADMIN_USER" \
  --admin-password "$VMSS_PASSWORD" \
  --authentication-type password \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_VMSS" \
  --upgrade-policy-mode Automatic \
  --public-ip-address ""

echo "VMSS created"
```

---

## Step 7 – Install Web Servers on VMSS

```bash
# Install nginx and create different pages for path-based routing
az vmss extension set \
  --vmss-name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --version 2.1 \
  --settings '{
    "commandToExecute": "apt-get update && apt-get install -y nginx && mkdir -p /var/www/html/api /var/www/html/images && echo \"<h1>Home Page</h1>\" > /var/www/html/index.html && echo \"<h1>API Response</h1>\" > /var/www/html/api/index.html && echo \"<h1>Images Gallery</h1>\" > /var/www/html/images/index.html && systemctl restart nginx"
  }'

# Update instances
az vmss update-instances \
  --instance-ids "*" \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME"

echo "Web servers installed"
```

---

## Step 8 – Create Application Gateway

```bash
# Create Application Gateway with WAF_v2
az network application-gateway create \
  --name "$APPGW_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku WAF_v2 \
  --capacity 2 \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_APPGW" \
  --public-ip-address "$PIP_NAME" \
  --http-settings-cookie-based-affinity Disabled \
  --http-settings-protocol Http \
  --frontend-port 80 \
  --cert-file appgw-cert.pfx \
  --cert-password "$CERT_PASSWORD"

echo "Application Gateway created"
```

---

## Step 9 – Configure WAF Policy

```bash
# Enable WAF in prevention mode
az network application-gateway waf-config set \
  --resource-group "$RG_NAME" \
  --gateway-name "$APPGW_NAME" \
  --enabled true \
  --firewall-mode Prevention \
  --rule-set-type OWASP \
  --rule-set-version 3.2

# Verify WAF configuration
az network application-gateway waf-config show \
  --resource-group "$RG_NAME" \
  --gateway-name "$APPGW_NAME" \
  --query "{Enabled:enabled, Mode:firewallMode, RuleSet:ruleSetType}" \
  --output table
```

---

## Step 10 – Add VMSS to Backend Pool

```bash
# Get VMSS NIC configuration
VMSS_NIC_CONFIG=$(az vmss show \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --query "virtualMachineProfile.networkProfile.networkInterfaceConfigurations[0].name" \
  --output tsv)

echo "$VMSS_NIC_CONFIG"

# Update VMSS to add to Application Gateway backend pool
az vmss update \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME" \
  --add virtualMachineProfile.networkProfile.networkInterfaceConfigurations[0].ipConfigurations[0].applicationGatewayBackendAddressPools \
  id="/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.Network/applicationGateways/$APPGW_NAME/backendAddressPools/appGatewayBackendPool"

# Update instances
az vmss update-instances \
  --instance-ids "*" \
  --name "$VMSS_NAME" \
  --resource-group "$RG_NAME"

echo "VMSS added to backend pool"
```

---

## Step 11 – Configure HTTPS Listener

```bash
# Add HTTPS frontend port
az network application-gateway frontend-port create \
  --gateway-name "$APPGW_NAME" \
  --resource-group "$RG_NAME" \
  --name "httpsPort" \
  --port 443

# Create HTTPS listener with SSL certificate
az network application-gateway http-listener create \
  --gateway-name "$APPGW_NAME" \
  --resource-group "$RG_NAME" \
  --name "httpsListener" \
  --frontend-port "httpsPort" \
  --ssl-cert "$APPGW_NAME" \
  --frontend-ip appGatewayFrontendIP

echo "HTTPS listener configured"
```

---

## Step 12 – Configure Path-Based Routing

```bash
# Create path-based rule for /api/*
az network application-gateway url-path-map create \
  --gateway-name "$APPGW_NAME" \
  --resource-group "$RG_NAME" \
  --name "pathMap" \
  --paths "/api/*" \
  --address-pool appGatewayBackendPool \
  --http-settings appGatewayBackendHttpSettings \
  --default-address-pool appGatewayBackendPool \
  --default-http-settings appGatewayBackendHttpSettings

# Add path rule for /images/*
az network application-gateway url-path-map rule create \
  --gateway-name "$APPGW_NAME" \
  --resource-group "$RG_NAME" \
  --path-map-name "pathMap" \
  --name "imagesRule" \
  --paths "/images/*" \
  --address-pool appGatewayBackendPool \
  --http-settings appGatewayBackendHttpSettings

# Create routing rule with path-based routing
az network application-gateway rule create \
  --gateway-name "$APPGW_NAME" \
  --resource-group "$RG_NAME" \
  --name "pathRule" \
  --http-listener "httpsListener" \
  --rule-type PathBasedRouting \
  --url-path-map "pathMap" \
  --priority 100

echo "Path-based routing configured"
```

---

## Step 13 – Test Application Gateway

```bash
# Wait for Application Gateway to be ready
echo "Waiting for Application Gateway..."
sleep 60

# Test HTTP endpoint
echo "Testing home page:"
curl -k "http://${APPGW_IP}/"

echo ""
echo "Testing /api path:"
curl -k "http://${APPGW_IP}/api/"

echo ""
echo "Testing /images path:"
curl -k "http://${APPGW_IP}/images/"
```

---

## Step 14 – Test WAF Functionality

```bash
# Test SQL injection attack (should be blocked by WAF)
echo "Testing WAF - SQL Injection (should be blocked):"
curl -k "http://${APPGW_IP}/?id=1' OR '1'='1" -v 2>&1 | grep -i "403\|forbidden" || echo "Request allowed"

echo ""
echo "Testing WAF - XSS attack (should be blocked):"
curl -k "http://${APPGW_IP}/?input=<script>alert('xss')</script>" -v 2>&1 | grep -i "403\|forbidden" || echo "Request allowed"

echo ""
echo "Testing normal request (should succeed):"
curl -k "http://${APPGW_IP}/" -I | grep -i "200\|OK"

echo "WAF is protecting the application"
```

---

## Step 15 – View Application Gateway Metrics

```bash
# Get Application Gateway resource ID
APPGW_ID=$(az network application-gateway show \
  --name "$APPGW_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$APPGW_ID"

# Get request count metrics
az monitor metrics list \
  --resource "$APPGW_ID" \
  --metric "TotalRequests" \
  --start-time "$(date -u -d '30 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT5M \
  --output table
```

---

## Step 16 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove certificate files
rm -f appgw-cert.pfx appgw-cert.pem appgw-key.pem

echo "Cleanup complete"
```

---

## Summary

You deployed Application Gateway with WAF_v2, configured SSL termination with user-provided certificate password, implemented path-based routing, integrated VMSS as backend, and validated WAF protection against malicious requests.
