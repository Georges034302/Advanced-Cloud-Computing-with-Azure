# Lab 14.D: Multi-Region Architecture

## Objectives
- Design multi-region deployment architecture
- Deploy resources in multiple Azure regions
- Configure Traffic Manager for global load balancing
- Implement database replication across regions
- Test failover scenarios
- Monitor multi-region health
- Optimize for performance and availability
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Locations: Australia East and Australia Southeast

---

## Step 1 – Set Variables

```bash
# Set Azure regions and resource naming
PRIMARY_LOCATION="australiaeast"
SECONDARY_LOCATION="australiasoutheast"
PRIMARY_RG="rg-lab14d-primary"
SECONDARY_RG="rg-lab14d-secondary"
TM_RG="rg-lab14d-traffic"
TM_PROFILE="tm-global-$RANDOM"
PRIMARY_VNET="vnet-primary"
SECONDARY_VNET="vnet-secondary"
PRIMARY_APP="app-primary-$RANDOM"
SECONDARY_APP="app-secondary-$RANDOM"

# Display configuration
echo "$PRIMARY_LOCATION"
echo "$SECONDARY_LOCATION"
echo "$TM_PROFILE"
```

---

## Step 2 – Create Resource Groups

```bash
# Create primary region resource group
az group create \
  --name "$PRIMARY_RG" \
  --location "$PRIMARY_LOCATION"

# Create secondary region resource group
az group create \
  --name "$SECONDARY_RG" \
  --location "$SECONDARY_LOCATION"

# Create Traffic Manager resource group
az group create \
  --name "$TM_RG" \
  --location "$PRIMARY_LOCATION"

echo "Resource groups created"
```

---

## Step 3 – Create Primary Virtual Network

```bash
# Create VNet in primary region
az network vnet create \
  --name "$PRIMARY_VNET" \
  --resource-group "$PRIMARY_RG" \
  --location "$PRIMARY_LOCATION" \
  --address-prefix 10.100.0.0/16 \
  --subnet-name "subnet-web" \
  --subnet-prefix 10.100.1.0/24

echo "Primary VNet created"
```

---

## Step 4 – Create Secondary Virtual Network

```bash
# Create VNet in secondary region
az network vnet create \
  --name "$SECONDARY_VNET" \
  --resource-group "$SECONDARY_RG" \
  --location "$SECONDARY_LOCATION" \
  --address-prefix 10.200.0.0/16 \
  --subnet-name "subnet-web" \
  --subnet-prefix 10.200.1.0/24

echo "Secondary VNet created"
```

---

## Step 5 – Create Primary App Service Plan

```bash
# Create App Service Plan in primary region
az appservice plan create \
  --name "plan-primary" \
  --resource-group "$PRIMARY_RG" \
  --location "$PRIMARY_LOCATION" \
  --sku S1 \
  --is-linux

echo "Primary App Service Plan created"
```

---

## Step 6 – Create Secondary App Service Plan

```bash
# Create App Service Plan in secondary region
az appservice plan create \
  --name "plan-secondary" \
  --resource-group "$SECONDARY_RG" \
  --location "$SECONDARY_LOCATION" \
  --sku S1 \
  --is-linux

echo "Secondary App Service Plan created"
```

---

## Step 7 – Create Primary Web App

```bash
# Create Web App in primary region
az webapp create \
  --name "$PRIMARY_APP" \
  --resource-group "$PRIMARY_RG" \
  --plan "plan-primary" \
  --runtime "NODE:18-lts"

echo "Primary Web App created"
```

---

## Step 8 – Create Secondary Web App

```bash
# Create Web App in secondary region
az webapp create \
  --name "$SECONDARY_APP" \
  --resource-group "$SECONDARY_RG" \
  --plan "plan-secondary" \
  --runtime "NODE:18-lts"

echo "Secondary Web App created"
```

---

## Step 9 – Deploy Application to Primary

```bash
# Create sample Node.js application
mkdir -p multi-region-app
cat > multi-region-app/index.js << 'EOF'
const http = require('http');
const os = require('os');

const server = http.createServer((req, res) => {
  const region = process.env.REGION || 'Unknown';
  const hostname = os.hostname();
  
  res.writeHead(200, { 'Content-Type': 'text/html' });
  res.end(`
    <html>
      <head><title>Multi-Region App</title></head>
      <body style="font-family: Arial; padding: 50px; text-align: center;">
        <h1>Multi-Region Application</h1>
        <h2 style="color: #0078D4;">Region: ${region}</h2>
        <p>Server: ${hostname}</p>
        <p>Timestamp: ${new Date().toISOString()}</p>
      </body>
    </html>
  `);
});

const port = process.env.PORT || 8080;
server.listen(port, () => {
  console.log(`Server running on port ${port} in ${process.env.REGION}`);
});
EOF

cat > multi-region-app/package.json << 'EOF'
{
  "name": "multi-region-app",
  "version": "1.0.0",
  "description": "Multi-region web application",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
EOF

echo "Application code created"
```

---

## Step 10 – Configure Primary App Settings

```bash
# Set environment variable for primary region
az webapp config appsettings set \
  --name "$PRIMARY_APP" \
  --resource-group "$PRIMARY_RG" \
  --settings REGION="Australia East (Primary)"

echo "Primary app settings configured"
```

---

## Step 11 – Configure Secondary App Settings

```bash
# Set environment variable for secondary region
az webapp config appsettings set \
  --name "$SECONDARY_APP" \
  --resource-group "$SECONDARY_RG" \
  --settings REGION="Australia Southeast (Secondary)"

echo "Secondary app settings configured"
```

---

## Step 12 – Deploy to Primary Region

```bash
# Create deployment package
cd multi-region-app
zip -r ../app.zip . >/dev/null 2>&1
cd ..

# Deploy to primary Web App
az webapp deployment source config-zip \
  --name "$PRIMARY_APP" \
  --resource-group "$PRIMARY_RG" \
  --src app.zip

echo "Deployed to primary region"
```

---

## Step 13 – Deploy to Secondary Region

```bash
# Deploy to secondary Web App
az webapp deployment source config-zip \
  --name "$SECONDARY_APP" \
  --resource-group "$SECONDARY_RG" \
  --src app.zip

echo "Deployed to secondary region"
```

---

## Step 14 – Get Web App URLs

```bash
# Get primary Web App URL
PRIMARY_URL=$(az webapp show \
  --name "$PRIMARY_APP" \
  --resource-group "$PRIMARY_RG" \
  --query defaultHostName \
  --output tsv)

echo "$PRIMARY_URL"

# Get secondary Web App URL
SECONDARY_URL=$(az webapp show \
  --name "$SECONDARY_APP" \
  --resource-group "$SECONDARY_RG" \
  --query defaultHostName \
  --output tsv)

echo "$SECONDARY_URL"
```

---

## Step 15 – Test Primary Application

```bash
# Test primary Web App
echo "Testing primary region..."
curl -s "https://${PRIMARY_URL}" | grep -o '<h2.*</h2>'

echo "Primary application tested"
```

---

## Step 16 – Test Secondary Application

```bash
# Test secondary Web App
echo "Testing secondary region..."
curl -s "https://${SECONDARY_URL}" | grep -o '<h2.*</h2>'

echo "Secondary application tested"
```

---

## Step 17 – Create Traffic Manager Profile

```bash
# Create Traffic Manager profile with performance routing
az network traffic-manager profile create \
  --name "$TM_PROFILE" \
  --resource-group "$TM_RG" \
  --routing-method Performance \
  --unique-dns-name "$TM_PROFILE" \
  --ttl 30 \
  --protocol HTTPS \
  --port 443 \
  --path "/"

echo "Traffic Manager profile created"
```

---

## Step 18 – Add Primary Endpoint

```bash
# Get primary Web App resource ID
PRIMARY_APP_ID=$(az webapp show \
  --name "$PRIMARY_APP" \
  --resource-group "$PRIMARY_RG" \
  --query id \
  --output tsv)

echo "$PRIMARY_APP_ID"

# Add primary endpoint to Traffic Manager
az network traffic-manager endpoint create \
  --name "endpoint-primary" \
  --profile-name "$TM_PROFILE" \
  --resource-group "$TM_RG" \
  --type azureEndpoints \
  --target-resource-id "$PRIMARY_APP_ID" \
  --endpoint-status Enabled \
  --priority 1

echo "Primary endpoint added"
```

---

## Step 19 – Add Secondary Endpoint

```bash
# Get secondary Web App resource ID
SECONDARY_APP_ID=$(az webapp show \
  --name "$SECONDARY_APP" \
  --resource-group "$SECONDARY_RG" \
  --query id \
  --output tsv)

echo "$SECONDARY_APP_ID"

# Add secondary endpoint to Traffic Manager
az network traffic-manager endpoint create \
  --name "endpoint-secondary" \
  --profile-name "$TM_PROFILE" \
  --resource-group "$TM_RG" \
  --type azureEndpoints \
  --target-resource-id "$SECONDARY_APP_ID" \
  --endpoint-status Enabled \
  --priority 2

echo "Secondary endpoint added"
```

---

## Step 20 – Get Traffic Manager DNS

```bash
# Get Traffic Manager DNS name
TM_DNS=$(az network traffic-manager profile show \
  --name "$TM_PROFILE" \
  --resource-group "$TM_RG" \
  --query dnsConfig.fqdn \
  --output tsv)

echo "$TM_DNS"
```

---

## Step 21 – Test Traffic Manager

```bash
# Test Traffic Manager endpoint
echo "Testing Traffic Manager (may take a few minutes for DNS propagation)..."
sleep 30
curl -s "http://${TM_DNS}" | grep -o '<h2.*</h2>' || echo "Waiting for DNS propagation"

echo "Traffic Manager tested"
```

---

## Step 22 – Show Routing Methods

```bash
# Display Traffic Manager routing methods
cat << 'EOF'

Traffic Manager Routing Methods:
=================================

Performance:
  - Routes to closest endpoint (lowest latency)
  - Best for global applications
  - Uses DNS location of client

Priority:
  - Primary/failover configuration
  - Routes to highest priority available
  - Good for active-passive scenarios

Weighted:
  - Distributes traffic by weight
  - A/B testing, gradual rollouts
  - Can be even or custom distribution

Geographic:
  - Routes based on geographic location
  - Data sovereignty requirements
  - Region-specific content

MultiValue:
  - Returns multiple healthy endpoints
  - Client chooses endpoint
  - IPv4/IPv6 addresses only

Subnet:
  - Maps IP address ranges to endpoints
  - Custom routing rules
  - Advanced scenarios

EOF
```

---

## Step 23 – Create Cosmos DB Account

```bash
# Create globally distributed Cosmos DB account
COSMOS_ACCOUNT="cosmos-global-$RANDOM"

az cosmosdb create \
  --name "$COSMOS_ACCOUNT" \
  --resource-group "$PRIMARY_RG" \
  --locations regionName="$PRIMARY_LOCATION" failoverPriority=0 isZoneRedundant=False \
  --locations regionName="$SECONDARY_LOCATION" failoverPriority=1 isZoneRedundant=False \
  --enable-automatic-failover true \
  --default-consistency-level Session

echo "Cosmos DB account created"
```

---

## Step 24 – Show Cosmos DB Locations

```bash
# Display Cosmos DB replication locations
az cosmosdb show \
  --name "$COSMOS_ACCOUNT" \
  --resource-group "$PRIMARY_RG" \
  --query "{Name:name, Locations:writeLocations[*].locationName, ReadLocations:readLocations[*].locationName}" \
  --output json
```

---

## Step 25 – Monitor Endpoint Health

```bash
# Check Traffic Manager endpoint health
az network traffic-manager endpoint list \
  --profile-name "$TM_PROFILE" \
  --resource-group "$TM_RG" \
  --query "[].{Name:name, Status:endpointStatus, MonitorStatus:endpointMonitorStatus, Priority:priority}" \
  --output table
```

---

## Step 26 – Show Multi-Region Patterns

```bash
# Display architecture patterns
cat << 'EOF'

Multi-Region Architecture Patterns:
====================================

Active-Passive:
  - Primary region handles all traffic
  - Secondary on standby for failover
  - Lower cost, longer recovery time
  - Use Traffic Manager Priority routing

Active-Active:
  - Both regions serve traffic
  - Geographic distribution
  - Higher availability and performance
  - Use Traffic Manager Performance routing

Active-Active-Passive:
  - Two regions active, one standby
  - Balanced cost and resilience
  - Regional failover capability

Deployment Stamps:
  - Identical deployments per region
  - Independent scaling
  - Region isolation
  - Simplified operations

Geo-Distributed Data:
  - Data replicated across regions
  - Local reads, global writes
  - Eventual consistency considerations
  - Use Cosmos DB multi-region writes

EOF
```

---

## Step 27 – Show Best Practices

```bash
# Display best practices
cat << 'EOF'

Multi-Region Best Practices:
=============================

Design Principles:
  - Design for failure
  - Automate deployments
  - Use managed services
  - Implement health checks
  - Monitor cross-region latency

Data Strategy:
  - Identify data residency requirements
  - Choose appropriate consistency model
  - Plan for data synchronization
  - Handle conflicts and replication lag

Application Design:
  - Stateless application tier
  - Session affinity considerations
  - Connection string management
  - Retry logic with exponential backoff

Networking:
  - VNet peering for private connectivity
  - ExpressRoute for hybrid scenarios
  - CDN for static content
  - Traffic Manager for DNS routing

Operations:
  - Consistent deployment pipelines
  - Region-specific configurations
  - Centralized monitoring
  - Regular failover testing

Cost Optimization:
  - Right-size resources per region
  - Use autoscaling
  - Optimize data transfer costs
  - Consider reserved instances

EOF
```

---

## Step 28 – Simulate Failover

```bash
# Disable primary endpoint to simulate failure
az network traffic-manager endpoint update \
  --name "endpoint-primary" \
  --profile-name "$TM_PROFILE" \
  --resource-group "$TM_RG" \
  --endpoint-status Disabled

echo "Primary endpoint disabled - simulating failure"

# Wait for health probe
sleep 60

# Test failover
echo "Testing after failover..."
curl -s "http://${TM_DNS}" | grep -o '<h2.*</h2>' || echo "Waiting for failover"
```

---

## Step 29 – Re-enable Primary

```bash
# Re-enable primary endpoint
az network traffic-manager endpoint update \
  --name "endpoint-primary" \
  --profile-name "$TM_PROFILE" \
  --resource-group "$TM_RG" \
  --endpoint-status Enabled

echo "Primary endpoint re-enabled"
```

---

## Step 30 – Show Disaster Recovery

```bash
# Display DR considerations
cat << 'EOF'

Disaster Recovery Strategy:
============================

Recovery Metrics:
  - RPO (Recovery Point Objective): Acceptable data loss
  - RTO (Recovery Time Objective): Acceptable downtime

DR Tiers:
  - Tier 1: <1 hour RTO, <15 min RPO (Mission critical)
  - Tier 2: <4 hours RTO, <1 hour RPO (Business critical)
  - Tier 3: <24 hours RTO, <4 hours RPO (Important)
  - Tier 4: >24 hours RTO (Non-critical)

Azure Services:
  - Traffic Manager: DNS-level failover
  - Azure Site Recovery: VM replication
  - Geo-redundant storage: Data replication
  - Cosmos DB: Multi-region writes
  - Azure SQL: Active geo-replication

Testing:
  - Regular DR drills
  - Document runbooks
  - Validate recovery procedures
  - Measure actual RTO/RPO
  - Update plans based on results

EOF
```

---

## Step 31 – Show Monitoring

```bash
# Display monitoring setup
cat << EOF

Multi-Region Monitoring:
========================

Key Metrics:
  - Application availability per region
  - Request latency by region
  - Traffic distribution
  - Failover events
  - Data replication lag

Azure Monitor:
  - Application Insights per region
  - Log Analytics workspace
  - Action Groups for alerts
  - Availability tests

Traffic Manager Metrics:
  - Endpoint health status
  - Query volume by endpoint
  - Latency by endpoint

Alerts to Configure:
  - Endpoint unavailable
  - High latency
  - Failover triggered
  - Replication lag exceeds threshold

Traffic Manager URL: http://${TM_DNS}
Primary App: https://${PRIMARY_URL}
Secondary App: https://${SECONDARY_URL}

EOF
```

---

## Step 32 – Cleanup

```bash
# Delete all resource groups
az group delete \
  --name "$PRIMARY_RG" \
  --yes \
  --no-wait

az group delete \
  --name "$SECONDARY_RG" \
  --yes \
  --no-wait

az group delete \
  --name "$TM_RG" \
  --yes \
  --no-wait

# Remove local files
rm -rf multi-region-app
rm -f app.zip

echo "Cleanup complete"
```

---

## Summary

You designed and implemented a multi-region architecture across Australia East and Southeast, deployed identical web applications in both regions using App Service, configured Azure Traffic Manager with performance-based routing for global load balancing, created geo-distributed Cosmos DB account with automatic failover, tested failover scenarios by disabling endpoints, monitored endpoint health and traffic distribution, explored multi-region patterns including active-active and active-passive configurations, and learned best practices for disaster recovery with RPO and RTO considerations.
