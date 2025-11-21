# Lab 6.A: ACR and ACI Deployment

## Overview
This lab demonstrates Azure Container Registry and Azure Container Instances. You'll create an ACR, build a Flask API Docker image, push it to ACR, deploy to ACI with environment variables, and validate the container.

---

## Objectives
- Create Azure Container Registry
- Build Docker image for Flask API
- Push image to ACR
- Deploy container to Azure Container Instances
- Configure environment variables
- Validate container deployment
- Output ACI FQDN
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
RG_NAME="rg-lab6a-aci"
ACR_NAME="acr$RANDOM$RANDOM"
IMAGE_NAME="flask-api"
IMAGE_TAG="v1"
ACI_NAME="aci-flask-api"
DNS_LABEL="flask-api-$RANDOM"

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
# Create ACR with Basic SKU
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

## Step 4 – Create Flask Application

```bash
# Create application directory
mkdir -p flask-api
cd flask-api

# Create Flask application
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'message': 'Flask API running in Azure Container Instance',
        'hostname': socket.gethostname(),
        'environment': os.getenv('APP_ENV', 'development'),
        'version': os.getenv('APP_VERSION', '1.0.0')
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Create requirements file
cat > requirements.txt << 'EOF'
Flask==3.0.0
Werkzeug==3.0.1
EOF

echo "Flask application created"
```

---

## Step 5 – Create Dockerfile

```bash
# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF

echo "Dockerfile created"
```

---

## Step 6 – Build Docker Image in ACR

```bash
# Build image using ACR build task
az acr build \
  --registry "$ACR_NAME" \
  --image "${IMAGE_NAME}:${IMAGE_TAG}" \
  --file Dockerfile \
  .

cd ..
echo "Image built in ACR"
```

---

## Step 7 – Verify Image in ACR

```bash
# List repositories in ACR
az acr repository list \
  --name "$ACR_NAME" \
  --output table

# Show image tags
az acr repository show-tags \
  --name "$ACR_NAME" \
  --repository "$IMAGE_NAME" \
  --output table
```

---

## Step 8 – Get ACR Credentials

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

## Step 9 – Deploy to Azure Container Instances

```bash
# Create ACI with environment variables
az container create \
  --name "$ACI_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --image "${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}" \
  --registry-login-server "$ACR_LOGIN_SERVER" \
  --registry-username "$ACR_USERNAME" \
  --registry-password "$ACR_PASSWORD" \
  --dns-name-label "$DNS_LABEL" \
  --ports 5000 \
  --cpu 1 \
  --memory 1 \
  --environment-variables APP_ENV=production APP_VERSION=1.0.0

echo "Container instance created"
```

---

## Step 10 – Get ACI FQDN

```bash
# Get container FQDN
ACI_FQDN=$(az container show \
  --name "$ACI_NAME" \
  --resource-group "$RG_NAME" \
  --query ipAddress.fqdn \
  --output tsv)

echo "$ACI_FQDN"

# Get container IP address
ACI_IP=$(az container show \
  --name "$ACI_NAME" \
  --resource-group "$RG_NAME" \
  --query ipAddress.ip \
  --output tsv)

echo "$ACI_IP"
```

---

## Step 11 – Wait for Container to Start

```bash
# Wait for container to be in running state
echo "Waiting for container to start..."
az container wait \
  --name "$ACI_NAME" \
  --resource-group "$RG_NAME" \
  --condition "running" \
  --interval 10

echo "Container is running"
```

---

## Step 12 – Validate Container Deployment

```bash
# Check container state
az container show \
  --name "$ACI_NAME" \
  --resource-group "$RG_NAME" \
  --query "containers[0].instanceView.currentState.state" \
  --output tsv

# Wait for application to be ready
echo "Waiting for application to be ready..."
sleep 15

# Test API endpoint
echo "Testing Flask API:"
curl -s "http://${ACI_FQDN}:5000/" | python3 -m json.tool

# Test health endpoint
echo "Testing health endpoint:"
curl -s "http://${ACI_FQDN}:5000/health" | python3 -m json.tool
```

---

## Step 13 – View Container Logs

```bash
# Get container logs
az container logs \
  --name "$ACI_NAME" \
  --resource-group "$RG_NAME"
```

---

## Step 14 – View Container Metrics

```bash
# Get container CPU usage
az monitor metrics list \
  --resource "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.ContainerInstance/containerGroups/$ACI_NAME" \
  --metric "CpuUsage" \
  --start-time "$(date -u -d '10 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT1M \
  --output table

# Get container memory usage
az monitor metrics list \
  --resource "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.ContainerInstance/containerGroups/$ACI_NAME" \
  --metric "MemoryUsage" \
  --start-time "$(date -u -d '10 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT1M \
  --output table
```

---

## Step 15 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf flask-api

echo "Cleanup complete"
```

---

## Summary

You created an Azure Container Registry, built a Flask API Docker image using ACR build tasks, pushed it to ACR, deployed to Azure Container Instances with environment variables, and validated the deployment using the container FQDN.
