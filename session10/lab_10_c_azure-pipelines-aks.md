# Lab 10.C: Azure Pipelines AKS Deployment

## Objectives
- Create Azure Kubernetes Service cluster
- Create Azure Container Registry
- Configure Azure Pipelines for AKS
- Deploy microservices to AKS
- Implement rolling updates
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- GitHub account for code repository
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab10c-aks"
AKS_NAME="aks-pipeline"
ACR_NAME="acrpipeline$RANDOM"
REPO_NAME="azure-pipelines-aks"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$AKS_NAME"
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

## Step 3 – Create Container Registry

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

## Step 4 – Create AKS Cluster

```bash
# Create AKS cluster
az aks create \
  --name "$AKS_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --network-plugin azure \
  --enable-managed-identity \
  --attach-acr "$ACR_NAME" \
  --generate-ssh-keys

echo "AKS cluster created"
```

---

## Step 5 – Get AKS Credentials

```bash
# Get AKS credentials
az aks get-credentials \
  --name "$AKS_NAME" \
  --resource-group "$RG_NAME" \
  --overwrite-existing

# Verify connection
kubectl get nodes
```

---

## Step 6 – Create Application Code

```bash
# Create application directory
mkdir -p "$REPO_NAME"
cd "$REPO_NAME"

# Create frontend application
mkdir -p frontend
cat > frontend/app.js << 'EOF'
const express = require('express');
const axios = require('axios');
const app = express();
const port = 8080;

const backendUrl = process.env.BACKEND_URL || 'http://backend:8081';

app.get('/', async (req, res) => {
  try {
    const response = await axios.get(`${backendUrl}/api`);
    res.json({
      service: 'frontend',
      version: '1.0',
      backend: response.data
    });
  } catch (error) {
    res.json({
      service: 'frontend',
      version: '1.0',
      error: 'Backend unavailable'
    });
  }
});

app.listen(port, () => {
  console.log(`Frontend running on port ${port}`);
});
EOF

cat > frontend/package.json << 'EOF'
{
  "name": "frontend",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2",
    "axios": "^1.6.0"
  },
  "scripts": {
    "start": "node app.js"
  }
}
EOF

cat > frontend/Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install --production
COPY app.js .
EXPOSE 8080
CMD ["npm", "start"]
EOF

# Create backend application
mkdir -p backend
cat > backend/app.js << 'EOF'
const express = require('express');
const app = express();
const port = 8081;

app.get('/api', (req, res) => {
  res.json({
    service: 'backend',
    version: '1.0',
    timestamp: new Date().toISOString()
  });
});

app.listen(port, () => {
  console.log(`Backend running on port ${port}`);
});
EOF

cat > backend/package.json << 'EOF'
{
  "name": "backend",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2"
  },
  "scripts": {
    "start": "node app.js"
  }
}
EOF

cat > backend/Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install --production
COPY app.js .
EXPOSE 8081
CMD ["npm", "start"]
EOF

echo "Application code created"
```

---

## Step 7 – Create Kubernetes Manifests

```bash
# Create manifests directory
mkdir -p k8s

# Create backend deployment
cat > k8s/backend-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: ACR_LOGIN_SERVER/backend:BUILD_ID
        ports:
        - containerPort: 8081
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8081
    targetPort: 8081
EOF

# Create frontend deployment
cat > k8s/frontend-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: ACR_LOGIN_SERVER/frontend:BUILD_ID
        ports:
        - containerPort: 8080
        env:
        - name: BACKEND_URL
          value: "http://backend:8081"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
EOF

echo "Kubernetes manifests created"
```

---

## Step 8 – Create Azure Pipeline

```bash
# Create Azure Pipelines YAML
cat > azure-pipelines.yml << EOF
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  azureSubscription: 'AzureServiceConnection'
  acrName: '$ACR_NAME'
  aksName: '$AKS_NAME'
  resourceGroup: '$RG_NAME'
  acrLoginServer: '$ACR_LOGIN_SERVER'

stages:
  - stage: Build
    displayName: 'Build and Push Images'
    jobs:
      - job: BuildImages
        displayName: 'Build Docker Images'
        steps:
          - task: Docker@2
            displayName: 'Build Backend Image'
            inputs:
              containerRegistry: 'ACRServiceConnection'
              repository: 'backend'
              command: 'build'
              Dockerfile: 'backend/Dockerfile'
              buildContext: 'backend'
              tags: |
                \$(Build.BuildId)
                latest

          - task: Docker@2
            displayName: 'Push Backend Image'
            inputs:
              containerRegistry: 'ACRServiceConnection'
              repository: 'backend'
              command: 'push'
              tags: |
                \$(Build.BuildId)
                latest

          - task: Docker@2
            displayName: 'Build Frontend Image'
            inputs:
              containerRegistry: 'ACRServiceConnection'
              repository: 'frontend'
              command: 'build'
              Dockerfile: 'frontend/Dockerfile'
              buildContext: 'frontend'
              tags: |
                \$(Build.BuildId)
                latest

          - task: Docker@2
            displayName: 'Push Frontend Image'
            inputs:
              containerRegistry: 'ACRServiceConnection'
              repository: 'frontend'
              command: 'push'
              tags: |
                \$(Build.BuildId)
                latest

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: 'k8s'
              ArtifactName: 'manifests'

  - stage: Deploy
    displayName: 'Deploy to AKS'
    dependsOn: Build
    jobs:
      - deployment: DeployAKS
        displayName: 'Deploy to Kubernetes'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: 'Deploy Backend'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'AKSServiceConnection'
                    manifests: |
                      \$(Pipeline.Workspace)/manifests/backend-deployment.yaml

                - task: KubernetesManifest@0
                  displayName: 'Deploy Frontend'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'AKSServiceConnection'
                    manifests: |
                      \$(Pipeline.Workspace)/manifests/frontend-deployment.yaml
EOF

echo "Azure Pipeline created"
```

---

## Step 9 – Initialize Git Repository

```bash
# Initialize git
git init
git add .
git commit -m "Initial commit with AKS pipeline"

echo "Git repository initialized"
```

---

## Step 10 – Build and Push Images Manually

```bash
# Login to ACR
az acr login --name "$ACR_NAME"

# Build and push backend
cd backend
docker build -t "$ACR_LOGIN_SERVER/backend:1.0" .
docker push "$ACR_LOGIN_SERVER/backend:1.0"
cd ..

# Build and push frontend
cd frontend
docker build -t "$ACR_LOGIN_SERVER/frontend:1.0" .
docker push "$ACR_LOGIN_SERVER/frontend:1.0"
cd ..

echo "Images pushed to ACR"
```

---

## Step 11 – Deploy to AKS Manually

```bash
# Update manifests with ACR server and build ID
sed -i "s|ACR_LOGIN_SERVER|$ACR_LOGIN_SERVER|g" k8s/*.yaml
sed -i "s|BUILD_ID|1.0|g" k8s/*.yaml

# Deploy to AKS
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/frontend-deployment.yaml

echo "Application deployed to AKS"
```

---

## Step 12 – Wait for Load Balancer

```bash
# Wait for external IP
echo "Waiting for external IP..."
sleep 60

# Get frontend service external IP
FRONTEND_IP=$(kubectl get service frontend -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "$FRONTEND_IP"
```

---

## Step 13 – Test Application

```bash
# Test frontend
curl -s "http://$FRONTEND_IP/" | python3 -m json.tool
```

---

## Step 14 – Monitor Deployment

```bash
# Check deployments
kubectl get deployments

# Check pods
kubectl get pods

# Check services
kubectl get services
```

---

## Step 15 – View Logs

```bash
# Get frontend pod logs
kubectl logs -l app=frontend --tail=50

# Get backend pod logs
kubectl logs -l app=backend --tail=50
```

---

## Step 16 – Cleanup

```bash
# Delete Kubernetes resources
kubectl delete -f k8s/

# Delete resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local repository
cd ..
rm -rf "$REPO_NAME"

echo "Cleanup complete"
```

---

## Summary

You created an AKS cluster with ACR integration, built and deployed microservices to Kubernetes, configured Azure Pipelines for container builds and deployments, and implemented a complete CI/CD workflow for Kubernetes applications.
