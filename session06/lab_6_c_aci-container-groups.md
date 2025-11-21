# Lab 6.C: ACI Container Groups

## Overview
This lab demonstrates Azure Container Instances with multi-container groups. You'll create an ACR, build multiple container images, deploy a container group with shared volumes, environment variables, and networking, then validate inter-container communication.

---

## Objectives
- Create Azure Container Registry
- Build multiple Docker images
- Deploy ACI Container Group with multiple containers
- Configure shared volumes between containers
- Set environment variables
- Configure container networking
- Validate inter-container communication
- Output container group IP
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
RG_NAME="rg-lab6c-aci-group"
ACR_NAME="acr$RANDOM$RANDOM"
API_IMAGE="api-service"
WEB_IMAGE="web-service"
IMAGE_TAG="v1"
CONTAINER_GROUP="container-group-app"
DNS_LABEL="aci-group-$RANDOM"
STORAGE_ACCOUNT="storage$RANDOM$RANDOM"
SHARE_NAME="shared-data"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$ACR_NAME"
echo "$DNS_LABEL"
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
# Create ACR
az acr create \
  --name "$ACR_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Basic \
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

## Step 4 – Create API Service Container

```bash
# Create API service directory
mkdir -p api-service
cd api-service

# Create Flask API
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
import socket
import time

app = Flask(__name__)

@app.route('/api/status')
def status():
    return jsonify({
        'service': 'api',
        'hostname': socket.gethostname(),
        'timestamp': time.time(),
        'environment': os.getenv('SERVICE_ENV', 'development')
    })

@app.route('/api/data')
def data():
    # Read from shared volume
    try:
        with open('/data/shared.txt', 'r') as f:
            content = f.read()
    except:
        content = 'No shared data'
    
    return jsonify({
        'service': 'api',
        'shared_data': content
    })

@app.route('/api/write')
def write():
    # Write to shared volume
    with open('/data/shared.txt', 'w') as f:
        f.write(f'Data from API service at {time.time()}')
    
    return jsonify({'message': 'Data written to shared volume'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Create requirements
cat > requirements.txt << 'EOF'
Flask==3.0.0
Werkzeug==3.0.1
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN mkdir -p /data

EXPOSE 5000

CMD ["python", "app.py"]
EOF

cd ..
echo "API service created"
```

---

## Step 5 – Create Web Service Container

```bash
# Create web service directory
mkdir -p web-service
cd web-service

# Create Node.js web service
cat > package.json << 'EOF'
{
  "name": "web-service",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2",
    "axios": "^1.6.0"
  }
}
EOF

# Create server
cat > server.js << 'EOF'
const express = require('express');
const axios = require('axios');
const app = express();
const port = 8080;

const API_URL = process.env.API_URL || 'http://localhost:5000';

app.get('/', async (req, res) => {
  try {
    const response = await axios.get(`${API_URL}/api/status`);
    res.send(`
      <html>
        <head><title>Container Group</title></head>
        <body>
          <h1>Container Group Communication</h1>
          <h2>Web Service (Port 8080)</h2>
          <p>Hostname: ${process.env.HOSTNAME}</p>
          <h2>API Service Response:</h2>
          <pre>${JSON.stringify(response.data, null, 2)}</pre>
          <p><a href="/data">View Shared Data</a></p>
          <p><a href="/write">Write to Shared Volume</a></p>
        </body>
      </html>
    `);
  } catch (error) {
    res.status(500).send(`Error communicating with API: ${error.message}`);
  }
});

app.get('/data', async (req, res) => {
  try {
    const response = await axios.get(`${API_URL}/api/data`);
    res.json(response.data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/write', async (req, res) => {
  try {
    const response = await axios.get(`${API_URL}/api/write`);
    res.json(response.data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(port, () => {
  console.log('Web service running on port ' + port);
});
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package.json .
RUN npm install --production

COPY server.js .

EXPOSE 8080

CMD ["node", "server.js"]
EOF

cd ..
echo "Web service created"
```

---

## Step 6 – Build and Push Images

```bash
# Build API service image
az acr build \
  --registry "$ACR_NAME" \
  --image "${API_IMAGE}:${IMAGE_TAG}" \
  --file api-service/Dockerfile \
  api-service/

echo "API image built"

# Build web service image
az acr build \
  --registry "$ACR_NAME" \
  --image "${WEB_IMAGE}:${IMAGE_TAG}" \
  --file web-service/Dockerfile \
  web-service/

echo "Web image built"
```

---

## Step 7 – Verify Images in ACR

```bash
# List repositories
az acr repository list \
  --name "$ACR_NAME" \
  --output table
```

---

## Step 8 – Create Storage Account for Shared Volume

```bash
# Create storage account
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "[0].value" \
  --output tsv)

echo "Storage account created"
```

---

## Step 9 – Create Azure File Share

```bash
# Create file share
az storage share create \
  --name "$SHARE_NAME" \
  --account-name "$STORAGE_ACCOUNT" \
  --account-key "$STORAGE_KEY"

echo "File share created"
```

---

## Step 10 – Get ACR Credentials

```bash
# Get ACR username
ACR_USERNAME=$(az acr credential show \
  --name "$ACR_NAME" \
  --query username \
  --output tsv)

echo "$ACR_USERNAME"

# Get ACR password
ACR_PASSWORD=$(az acr credential show \
  --name "$ACR_NAME" \
  --query "passwords[0].value" \
  --output tsv)

echo "ACR credentials retrieved"
```

---

## Step 11 – Create Container Group YAML

```bash
# Create container group configuration
cat > container-group.yaml << EOF
apiVersion: 2023-05-01
location: $LOCATION
name: $CONTAINER_GROUP
properties:
  containers:
  - name: api-service
    properties:
      image: ${ACR_LOGIN_SERVER}/${API_IMAGE}:${IMAGE_TAG}
      resources:
        requests:
          cpu: 1
          memoryInGb: 1
      ports:
      - port: 5000
        protocol: TCP
      environmentVariables:
      - name: SERVICE_ENV
        value: production
      volumeMounts:
      - name: shared-volume
        mountPath: /data
  - name: web-service
    properties:
      image: ${ACR_LOGIN_SERVER}/${WEB_IMAGE}:${IMAGE_TAG}
      resources:
        requests:
          cpu: 1
          memoryInGb: 1
      ports:
      - port: 8080
        protocol: TCP
      environmentVariables:
      - name: API_URL
        value: http://localhost:5000
      - name: SERVICE_ENV
        value: production
  imageRegistryCredentials:
  - server: ${ACR_LOGIN_SERVER}
    username: ${ACR_USERNAME}
    password: ${ACR_PASSWORD}
  ipAddress:
    type: Public
    ports:
    - port: 8080
      protocol: TCP
    dnsNameLabel: ${DNS_LABEL}
  osType: Linux
  restartPolicy: Always
  volumes:
  - name: shared-volume
    azureFile:
      shareName: ${SHARE_NAME}
      storageAccountName: ${STORAGE_ACCOUNT}
      storageAccountKey: ${STORAGE_KEY}
EOF

echo "Container group YAML created"
```

---

## Step 12 – Deploy Container Group

```bash
# Deploy container group using YAML
az container create \
  --resource-group "$RG_NAME" \
  --file container-group.yaml

echo "Container group deployed"
```

---

## Step 13 – Get Container Group IP and FQDN

```bash
# Get FQDN
CONTAINER_FQDN=$(az container show \
  --name "$CONTAINER_GROUP" \
  --resource-group "$RG_NAME" \
  --query ipAddress.fqdn \
  --output tsv)

echo "$CONTAINER_FQDN"

# Get IP address
CONTAINER_IP=$(az container show \
  --name "$CONTAINER_GROUP" \
  --resource-group "$RG_NAME" \
  --query ipAddress.ip \
  --output tsv)

echo "$CONTAINER_IP"
```

---

## Step 14 – Wait for Container Group to Start

```bash
# Wait for containers to be running
echo "Waiting for containers to start..."
sleep 30

# Check container states
az container show \
  --name "$CONTAINER_GROUP" \
  --resource-group "$RG_NAME" \
  --query "containers[].{Name:name, State:instanceView.currentState.state}" \
  --output table
```

---

## Step 15 – Validate Inter-Container Communication

```bash
# Test web service (calls API service internally)
echo "Testing web service:"
curl -s "http://${CONTAINER_FQDN}:8080/" | grep -o '<h1>.*</h1>'

# View full response with API communication
echo "Full response showing inter-container communication:"
curl -s "http://${CONTAINER_FQDN}:8080/"
```

---

## Step 16 – Test Shared Volume

```bash
# Write data to shared volume via API
echo "Writing data to shared volume:"
curl -s "http://${CONTAINER_FQDN}:8080/write" | python3 -m json.tool

# Wait for write to complete
sleep 2

# Read data from shared volume
echo "Reading data from shared volume:"
curl -s "http://${CONTAINER_FQDN}:8080/data" | python3 -m json.tool
```

---

## Step 17 – View Container Logs

```bash
# View API service logs
echo "API service logs:"
az container logs \
  --name "$CONTAINER_GROUP" \
  --resource-group "$RG_NAME" \
  --container-name api-service

echo ""
echo "Web service logs:"
az container logs \
  --name "$CONTAINER_GROUP" \
  --resource-group "$RG_NAME" \
  --container-name web-service
```

---

## Step 18 – View Container Group Events

```bash
# Show container group events
az container show \
  --name "$CONTAINER_GROUP" \
  --resource-group "$RG_NAME" \
  --query "containers[].instanceView.events[]" \
  --output table
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
rm -rf api-service web-service container-group.yaml

echo "Cleanup complete"
```

---

## Summary

You created an Azure Container Registry, built two container images for API and web services, deployed a multi-container group with shared Azure File volume, configured environment variables for inter-container communication, and validated communication between containers and shared volume access.
