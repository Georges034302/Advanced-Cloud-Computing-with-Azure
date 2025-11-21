# Lab 5.D: Traffic Manager Multi-Region

## Overview
This lab demonstrates Azure Traffic Manager for multi-region deployment and failover. You'll deploy web apps in two regions, create a Traffic Manager profile with priority routing, and test failover by disabling the primary endpoint.

---

## Objectives
- Deploy web apps in multiple Azure regions
- Create Traffic Manager profile
- Configure priority-based routing
- Add endpoints from different regions
- Test automatic failover
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Basic understanding of DNS and routing
- Locations: Australia East, Southeast Asia

---

## Step 1 – Set Variables

```bash
# Set Azure regions and resource naming
PRIMARY_LOCATION="australiaeast"
SECONDARY_LOCATION="southeastasia"
RG_NAME="rg-lab5d-tm"
TM_PROFILE="tm-lab5d-$RANDOM"
APP_PRIMARY="webapp-primary-$RANDOM"
APP_SECONDARY="webapp-secondary-$RANDOM"
PLAN_PRIMARY="plan-primary"
PLAN_SECONDARY="plan-secondary"

# Display configuration
echo "$PRIMARY_LOCATION"
echo "$SECONDARY_LOCATION"
echo "$TM_PROFILE"
echo "$APP_PRIMARY"
echo "$APP_SECONDARY"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group in primary region
az group create \
  --name "$RG_NAME" \
  --location "$PRIMARY_LOCATION"
```

---

## Step 3 – Create App Service Plan in Primary Region

```bash
# Create App Service plan in Australia East
az appservice plan create \
  --name "$PLAN_PRIMARY" \
  --resource-group "$RG_NAME" \
  --location "$PRIMARY_LOCATION" \
  --sku B1 \
  --is-linux

# Verify plan creation
az appservice plan show \
  --name "$PLAN_PRIMARY" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, Sku:sku.name}" \
  --output table
```

---

## Step 4 – Create Web App in Primary Region

```bash
# Create web app in primary region
az webapp create \
  --name "$APP_PRIMARY" \
  --resource-group "$RG_NAME" \
  --plan "$PLAN_PRIMARY" \
  --runtime "NODE:18-lts"

# Get primary app URL
PRIMARY_URL=$(az webapp show \
  --name "$APP_PRIMARY" \
  --resource-group "$RG_NAME" \
  --query defaultHostName \
  --output tsv)

echo "$PRIMARY_URL"
```

---

## Step 5 – Deploy Application to Primary Region

```bash
# Create simple Node.js application
mkdir -p webapp-primary
cd webapp-primary

# Create package.json
cat > package.json << 'EOF'
{
  "name": "primary-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

# Create server.js for primary region
cat > server.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;

app.get('/', (req, res) => {
  res.send(`
    <html>
      <head><title>Primary Region</title></head>
      <body>
        <h1>PRIMARY REGION - Australia East</h1>
        <p>Server: ${process.env.WEBSITE_SITE_NAME}</p>
        <p>Region: Australia East</p>
        <p>Priority: 1 (Primary)</p>
      </body>
    </html>
  `);
});

app.listen(port, () => {
  console.log('Primary app running on port ' + port);
});
EOF

# Deploy to primary web app
zip -r app.zip .
az webapp deployment source config-zip \
  --name "$APP_PRIMARY" \
  --resource-group "$RG_NAME" \
  --src app.zip

cd ..
echo "Primary app deployed"
```

---

## Step 6 – Create App Service Plan in Secondary Region

```bash
# Create App Service plan in Southeast Asia
az appservice plan create \
  --name "$PLAN_SECONDARY" \
  --resource-group "$RG_NAME" \
  --location "$SECONDARY_LOCATION" \
  --sku B1 \
  --is-linux

# Verify plan creation
az appservice plan show \
  --name "$PLAN_SECONDARY" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, Sku:sku.name}" \
  --output table
```

---

## Step 7 – Create Web App in Secondary Region

```bash
# Create web app in secondary region
az webapp create \
  --name "$APP_SECONDARY" \
  --resource-group "$RG_NAME" \
  --plan "$PLAN_SECONDARY" \
  --runtime "NODE:18-lts"

# Get secondary app URL
SECONDARY_URL=$(az webapp show \
  --name "$APP_SECONDARY" \
  --resource-group "$RG_NAME" \
  --query defaultHostName \
  --output tsv)

echo "$SECONDARY_URL"
```

---

## Step 8 – Deploy Application to Secondary Region

```bash
# Create application for secondary region
mkdir -p webapp-secondary
cd webapp-secondary

# Create package.json
cat > package.json << 'EOF'
{
  "name": "secondary-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

# Create server.js for secondary region
cat > server.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;

app.get('/', (req, res) => {
  res.send(`
    <html>
      <head><title>Secondary Region</title></head>
      <body style="background-color: #ffe6e6;">
        <h1>SECONDARY REGION - Southeast Asia</h1>
        <p>Server: ${process.env.WEBSITE_SITE_NAME}</p>
        <p>Region: Southeast Asia</p>
        <p>Priority: 2 (Failover)</p>
      </body>
    </html>
  `);
});

app.listen(port, () => {
  console.log('Secondary app running on port ' + port);
});
EOF

# Deploy to secondary web app
zip -r app.zip .
az webapp deployment source config-zip \
  --name "$APP_SECONDARY" \
  --resource-group "$RG_NAME" \
  --src app.zip

cd ..
echo "Secondary app deployed"
```

---

## Step 9 – Test Web Apps Directly

```bash
# Wait for deployments to complete
echo "Waiting for deployments..."
sleep 30

# Test primary app
echo "Testing primary app:"
curl -s "https://${PRIMARY_URL}" | grep -o '<h1>.*</h1>'

echo ""
echo "Testing secondary app:"
curl -s "https://${SECONDARY_URL}" | grep -o '<h1>.*</h1>'
```

---

## Step 10 – Create Traffic Manager Profile

```bash
# Create Traffic Manager profile with priority routing
az network traffic-manager profile create \
  --name "$TM_PROFILE" \
  --resource-group "$RG_NAME" \
  --routing-method Priority \
  --unique-dns-name "$TM_PROFILE" \
  --ttl 30 \
  --protocol HTTPS \
  --port 443 \
  --path "/"

# Get Traffic Manager DNS name
TM_DNS=$(az network traffic-manager profile show \
  --name "$TM_PROFILE" \
  --resource-group "$RG_NAME" \
  --query dnsConfig.fqdn \
  --output tsv)

echo "$TM_DNS"
```

---

## Step 11 – Add Primary Endpoint

```bash
# Get primary web app resource ID
PRIMARY_ID=$(az webapp show \
  --name "$APP_PRIMARY" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$PRIMARY_ID"

# Add primary endpoint with priority 1
az network traffic-manager endpoint create \
  --name "primary-endpoint" \
  --resource-group "$RG_NAME" \
  --profile-name "$TM_PROFILE" \
  --type azureEndpoints \
  --target-resource-id "$PRIMARY_ID" \
  --priority 1 \
  --endpoint-status Enabled

echo "Primary endpoint added"
```

---

## Step 12 – Add Secondary Endpoint

```bash
# Get secondary web app resource ID
SECONDARY_ID=$(az webapp show \
  --name "$APP_SECONDARY" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$SECONDARY_ID"

# Add secondary endpoint with priority 2
az network traffic-manager endpoint create \
  --name "secondary-endpoint" \
  --resource-group "$RG_NAME" \
  --profile-name "$TM_PROFILE" \
  --type azureEndpoints \
  --target-resource-id "$SECONDARY_ID" \
  --priority 2 \
  --endpoint-status Enabled

echo "Secondary endpoint added"
```

---

## Step 13 – Test Traffic Manager

```bash
# Wait for DNS propagation
echo "Waiting for DNS propagation..."
sleep 60

# Test Traffic Manager endpoint (should route to primary)
echo "Testing Traffic Manager (should route to PRIMARY):"
for i in {1..5}; do
  echo "Request ${i}:"
  curl -s "https://${TM_DNS}" | grep -o '<h1>.*</h1>'
  sleep 2
done
```

---

## Step 14 – Disable Primary Endpoint for Failover Test

```bash
# Disable primary endpoint to simulate failure
az network traffic-manager endpoint update \
  --name "primary-endpoint" \
  --resource-group "$RG_NAME" \
  --profile-name "$TM_PROFILE" \
  --type azureEndpoints \
  --endpoint-status Disabled

echo "Primary endpoint disabled"

# Wait for Traffic Manager to detect change
echo "Waiting for failover..."
sleep 60
```

---

## Step 15 – Test Failover

```bash
# Test Traffic Manager after failover (should route to secondary)
echo "Testing after failover (should route to SECONDARY):"
for i in {1..5}; do
  echo "Request ${i}:"
  curl -s "https://${TM_DNS}" | grep -o '<h1>.*</h1>'
  sleep 2
done

echo "Failover successful - traffic routed to secondary region"
```

---

## Step 16 – View Traffic Manager Configuration

```bash
# List all endpoints
az network traffic-manager endpoint list \
  --resource-group "$RG_NAME" \
  --profile-name "$TM_PROFILE" \
  --type azureEndpoints \
  --query "[].{Name:name, Status:endpointStatus, Priority:priority, MonitorStatus:endpointMonitorStatus}" \
  --output table

# Show Traffic Manager profile details
az network traffic-manager profile show \
  --name "$TM_PROFILE" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, DNS:dnsConfig.fqdn, Routing:trafficRoutingMethod, TTL:dnsConfig.ttl}" \
  --output table
```

---

## Step 17 – Re-enable Primary Endpoint

```bash
# Re-enable primary endpoint
az network traffic-manager endpoint update \
  --name "primary-endpoint" \
  --resource-group "$RG_NAME" \
  --profile-name "$TM_PROFILE" \
  --type azureEndpoints \
  --endpoint-status Enabled

echo "Primary endpoint re-enabled"

# Wait for Traffic Manager to update
echo "Waiting for Traffic Manager to update..."
sleep 60

# Test again (should route back to primary)
echo "Testing after re-enabling primary:"
for i in {1..3}; do
  curl -s "https://${TM_DNS}" | grep -o '<h1>.*</h1>'
  sleep 2
done
```

---

## Step 18 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf webapp-primary webapp-secondary

echo "Cleanup complete"
```

---

## Summary

You deployed web applications in Australia East and Southeast Asia, created a Traffic Manager profile with priority routing, added both endpoints, and tested automatic failover by disabling the primary endpoint.
