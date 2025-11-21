# Lab 6.B: Web App for Containers

## Overview
This lab demonstrates Azure Web App for Containers with continuous deployment. You'll create an ACR, build and push a container image, deploy a Web App for Containers, enable continuous deployment, configure health monitoring and scaling.

---

## Objectives
- Create Azure Container Registry
- Build and push Docker image
- Deploy Web App for Containers
- Enable continuous deployment from ACR
- Configure health monitoring
- Configure auto-scaling
- Validate container application
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Docker installed (`docker --version`)
- Azure subscription with contributor access
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab6b-webapp"
ACR_NAME="acr$RANDOM$RANDOM"
IMAGE_NAME="web-app"
IMAGE_TAG="v1"
PLAN_NAME="plan-containers"
WEBAPP_NAME="webapp-container-$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$ACR_NAME"
echo "$WEBAPP_NAME"
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

## Step 3 – Create Azure Container Registry

```bash
# Create ACR with Standard SKU for webhooks
az acr create \
  --name "$ACR_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard \
  --admin-enabled true

# Get ACR login server
ACR_LOGIN_SERVER=$(az acr show \
  --name "$ACR_NAME" \
  --resource-group "$RG_NAME" \
  --query loginServer \
  --output tsv)

echo "$ACR_LOGIN_SERVER"
```

---

## Step 4 – Create Web Application

```bash
# Create application directory
mkdir -p webapp-container
cd webapp-container

# Create Node.js application
cat > package.json << 'EOF'
{
  "name": "webapp-container",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

# Create Express server
cat > server.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;

let requestCount = 0;

app.get('/', (req, res) => {
  requestCount++;
  res.send(`
    <html>
      <head><title>Web App Container</title></head>
      <body>
        <h1>Web App for Containers</h1>
        <p>Container: ${process.env.HOSTNAME}</p>
        <p>Environment: ${process.env.APP_ENV || 'development'}</p>
        <p>Version: ${process.env.APP_VERSION || '1.0.0'}</p>
        <p>Request Count: ${requestCount}</p>
        <p>Date: ${new Date().toISOString()}</p>
      </body>
    </html>
  `);
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy', uptime: process.uptime() });
});

app.listen(port, () => {
  console.log('Server running on port ' + port);
});
EOF

echo "Web application created"
```

---

## Step 5 – Create Dockerfile

```bash
# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package.json .
RUN npm install --production

COPY server.js .

EXPOSE 8080

CMD ["npm", "start"]
EOF

echo "Dockerfile created"
```

---

## Step 6 – Build and Push Image to ACR

```bash
# Build image using ACR build
az acr build \
  --registry "$ACR_NAME" \
  --image "${IMAGE_NAME}:${IMAGE_TAG}" \
  --image "${IMAGE_NAME}:latest" \
  --file Dockerfile \
  .

cd ..
echo "Image built and pushed"
```

---

## Step 7 – Verify Image in ACR

```bash
# List repositories
az acr repository list \
  --name "$ACR_NAME" \
  --output table

# Show tags
az acr repository show-tags \
  --name "$ACR_NAME" \
  --repository "$IMAGE_NAME" \
  --output table
```

---

## Step 8 – Create App Service Plan

```bash
# Create Linux App Service plan
az appservice plan create \
  --name "$PLAN_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --is-linux \
  --sku B1

# Verify plan creation
az appservice plan show \
  --name "$PLAN_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Sku:sku.name, IsLinux:reserved}" \
  --output table
```

---

## Step 9 – Deploy Web App for Containers

```bash
# Get ACR credentials
ACR_USERNAME=$(az acr credential show \
  --name "$ACR_NAME" \
  --query username \
  --output tsv)

ACR_PASSWORD=$(az acr credential show \
  --name "$ACR_NAME" \
  --query "passwords[0].value" \
  --output tsv)

# Create Web App for Containers
az webapp create \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --plan "$PLAN_NAME" \
  --deployment-container-image-name "${ACR_LOGIN_SERVER}/${IMAGE_NAME}:latest"

# Configure container registry credentials
az webapp config container set \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --docker-custom-image-name "${ACR_LOGIN_SERVER}/${IMAGE_NAME}:latest" \
  --docker-registry-server-url "https://${ACR_LOGIN_SERVER}" \
  --docker-registry-server-user "$ACR_USERNAME" \
  --docker-registry-server-password "$ACR_PASSWORD"

echo "Web App created"
```

---

## Step 10 – Configure Application Settings

```bash
# Set environment variables
az webapp config appsettings set \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --settings \
    APP_ENV=production \
    APP_VERSION=1.0.0 \
    WEBSITES_PORT=8080

echo "Application settings configured"
```

---

## Step 11 – Enable Continuous Deployment

```bash
# Enable continuous deployment from ACR
az webapp deployment container config \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --enable-cd true

# Get webhook URL
WEBHOOK_URL=$(az webapp deployment container show-cd-url \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --query CI_CD_URL \
  --output tsv)

echo "Continuous deployment enabled"

# Create ACR webhook for continuous deployment
az acr webhook create \
  --name "webapp${RANDOM}" \
  --registry "$ACR_NAME" \
  --uri "$WEBHOOK_URL" \
  --actions push \
  --scope "${IMAGE_NAME}:*"

echo "ACR webhook created"
```

---

## Step 12 – Configure Health Check

```bash
# Enable health check monitoring
az webapp config set \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --health-check-path "/health"

echo "Health check configured"
```

---

## Step 13 – Configure Auto-Scaling

```bash
# Get App Service plan ID
PLAN_ID=$(az appservice plan show \
  --name "$PLAN_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$PLAN_ID"

# Create autoscale settings
az monitor autoscale create \
  --name "autoscale-webapp" \
  --resource-group "$RG_NAME" \
  --resource "$PLAN_ID" \
  --min-count 1 \
  --max-count 3 \
  --count 1

# Add scale-out rule based on CPU
az monitor autoscale rule create \
  --resource-group "$RG_NAME" \
  --autoscale-name "autoscale-webapp" \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 1

# Add scale-in rule based on CPU
az monitor autoscale rule create \
  --resource-group "$RG_NAME" \
  --autoscale-name "autoscale-webapp" \
  --condition "CpuPercentage < 30 avg 5m" \
  --scale in 1

echo "Auto-scaling configured"
```

---

## Step 14 – Get Web App URL

```bash
# Get default hostname
WEBAPP_URL=$(az webapp show \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --query defaultHostName \
  --output tsv)

echo "$WEBAPP_URL"
```

---

## Step 15 – Wait for Deployment

```bash
# Wait for container to start
echo "Waiting for container to start..."
sleep 60

# Check webapp state
az webapp show \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --query "state" \
  --output tsv
```

---

## Step 16 – Validate Container Application

```bash
# Test web application
echo "Testing web application:"
curl -s "https://${WEBAPP_URL}/" | grep -o '<h1>.*</h1>'

# Test health endpoint
echo "Testing health endpoint:"
curl -s "https://${WEBAPP_URL}/health" | python3 -m json.tool

# Test multiple requests
echo "Testing multiple requests:"
for i in {1..5}; do
  echo "Request ${i}:"
  curl -s "https://${WEBAPP_URL}/" | grep -o '<p>Request Count:.*</p>'
  sleep 1
done
```

---

## Step 17 – View Container Logs

```bash
# Enable container logging
az webapp log config \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --docker-container-logging filesystem

# View logs
az webapp log tail \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --only-show-errors &

sleep 10
kill %1 2>/dev/null
```

---

## Step 18 – Test Continuous Deployment

```bash
# Build new version of image
cd webapp-container

# Update version in package.json
sed -i 's/"version": "1.0.0"/"version": "1.1.0"/' package.json

# Build and push new version
az acr build \
  --registry "$ACR_NAME" \
  --image "${IMAGE_NAME}:v2" \
  --image "${IMAGE_NAME}:latest" \
  --file Dockerfile \
  .

cd ..
echo "New version pushed - webhook will trigger redeployment"

# Wait for webhook to trigger redeployment
echo "Waiting for redeployment..."
sleep 90

# Verify new deployment
curl -s "https://${WEBAPP_URL}/"
```

---

## Step 19 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf webapp-container

echo "Cleanup complete"
```

---

## Summary

You created an Azure Container Registry, built and pushed a web application container image, deployed Web App for Containers, enabled continuous deployment with ACR webhooks, configured health monitoring and auto-scaling, and validated the application.
