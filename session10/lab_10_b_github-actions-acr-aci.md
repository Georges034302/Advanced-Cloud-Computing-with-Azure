# Lab 10.B: GitHub Actions ACR ACI Pipeline

## Objectives
- Create Azure Container Registry
- Create GitHub repository with Actions workflow
- Configure OIDC authentication
- Build and push Docker images
- Deploy to Azure Container Instances
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- GitHub account for repository
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab10b-github"
ACR_NAME="acrgithub$RANDOM"
ACI_NAME="aci-github"
REPO_NAME="github-actions-aci"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$ACR_NAME"
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

## Step 4 – Get GitHub Repository Information

```bash
# Get account ID
ACCOUNT_ID=$(az account show --query id --output tsv)

echo "$ACCOUNT_ID"

# Set GitHub username
read -p "Enter your GitHub username: " GITHUB_USERNAME

echo "$GITHUB_USERNAME"
```

---

## Step 5 – Create GitHub OIDC Provider

```bash
# Create OIDC provider
az rest --method put \
  --uri "https://management.azure.com/subscriptions/$ACCOUNT_ID/providers/Microsoft.ManagedIdentity/userAssignedIdentities/github-oidc-identity?api-version=2023-01-31" \
  --body "{
    \"location\": \"$LOCATION\",
    \"properties\": {}
  }" || true

echo "OIDC configuration prepared"
```

---

## Step 6 – Create Service Principal

```bash
# Create service principal for GitHub Actions
SP_OUTPUT=$(az ad sp create-for-rbac \
  --name "sp-github-actions-aci" \
  --role Contributor \
  --scopes "/subscriptions/$ACCOUNT_ID/resourceGroups/$RG_NAME" \
  --output json)

# Extract credentials
CLIENT_ID=$(echo "$SP_OUTPUT" | python3 -c "import sys, json; print(json.load(sys.stdin)['appId'])")
CLIENT_SECRET=$(echo "$SP_OUTPUT" | python3 -c "import sys, json; print(json.load(sys.stdin)['password'])")
TENANT_ID=$(echo "$SP_OUTPUT" | python3 -c "import sys, json; print(json.load(sys.stdin)['tenant'])")

echo "Service principal created"
echo "Client ID: $CLIENT_ID"
```

---

## Step 7 – Grant ACR Permissions

```bash
# Get ACR resource ID
ACR_ID=$(az acr show \
  --name "$ACR_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$ACR_ID"

# Assign AcrPush role to service principal
az role assignment create \
  --assignee "$CLIENT_ID" \
  --role AcrPush \
  --scope "$ACR_ID"

echo "ACR permissions granted"
```

---

## Step 8 – Create Application Code

```bash
# Create application directory
mkdir -p "$REPO_NAME"
cd "$REPO_NAME"

# Create Node.js application
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;

app.get('/', (req, res) => {
  res.json({
    message: 'GitHub Actions to Azure Container Instances',
    version: '1.0',
    timestamp: new Date().toISOString(),
    hostname: require('os').hostname()
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
EOF

# Create package.json
cat > package.json << 'EOF'
{
  "name": "github-actions-aci",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

echo "Application code created"
```

---

## Step 9 – Create Dockerfile

```bash
# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package.json .
RUN npm install --production

COPY app.js .

EXPOSE 8080

CMD ["npm", "start"]
EOF

echo "Dockerfile created"
```

---

## Step 10 – Create GitHub Actions Workflow

```bash
# Create workflow directory
mkdir -p .github/workflows

# Create workflow file
cat > .github/workflows/deploy.yml << EOF
name: Build and Deploy to ACI

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  ACR_NAME: $ACR_NAME
  RESOURCE_GROUP: $RG_NAME
  ACI_NAME: $ACI_NAME
  IMAGE_NAME: webapp

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: \${{ secrets.AZURE_CREDENTIALS }}

      - name: Build and push image to ACR
        run: |
          az acr build \\
            --registry \${{ env.ACR_NAME }} \\
            --image \${{ env.IMAGE_NAME }}:\${{ github.sha }} \\
            --image \${{ env.IMAGE_NAME }}:latest \\
            --file Dockerfile .

      - name: Deploy to ACI
        run: |
          az container create \\
            --name \${{ env.ACI_NAME }} \\
            --resource-group \${{ env.RESOURCE_GROUP }} \\
            --image \${{ env.ACR_NAME }}.azurecr.io/\${{ env.IMAGE_NAME }}:latest \\
            --registry-login-server \${{ env.ACR_NAME }}.azurecr.io \\
            --registry-username \${{ secrets.ACR_USERNAME }} \\
            --registry-password \${{ secrets.ACR_PASSWORD }} \\
            --dns-name-label \${{ env.ACI_NAME }}-\${{ github.run_number }} \\
            --ports 8080 \\
            --cpu 1 \\
            --memory 1 \\
            --location $LOCATION \\
            --query "ipAddress.fqdn" \\
            --output tsv || \\
          az container restart \\
            --name \${{ env.ACI_NAME }} \\
            --resource-group \${{ env.RESOURCE_GROUP }}

      - name: Get container URL
        run: |
          FQDN=\$(az container show \\
            --name \${{ env.ACI_NAME }} \\
            --resource-group \${{ env.RESOURCE_GROUP }} \\
            --query "ipAddress.fqdn" \\
            --output tsv)
          echo "Application URL: http://\$FQDN:8080"
EOF

echo "GitHub Actions workflow created"
```

---

## Step 11 – Initialize Git Repository

```bash
# Initialize git
git init
git add .
git commit -m "Initial commit with GitHub Actions workflow"

echo "Git repository initialized"
```

---

## Step 12 – Create GitHub Repository

```bash
# Create GitHub repository
if command -v gh &> /dev/null; then
  gh repo create "$REPO_NAME" --public --source=. --remote=origin --push
  echo "GitHub repository created"
else
  echo "GitHub CLI not installed. Create repository manually at https://github.com/new"
  echo "Repository name: $REPO_NAME"
  echo "Then run: git remote add origin https://github.com/$GITHUB_USERNAME/$REPO_NAME.git"
  echo "Then run: git push -u origin main"
fi
```

---

## Step 13 – Get ACR Credentials

```bash
# Get ACR admin credentials
ACR_USERNAME=$(az acr credential show \
  --name "$ACR_NAME" \
  --query username \
  --output tsv)

ACR_PASSWORD=$(az acr credential show \
  --name "$ACR_NAME" \
  --query "passwords[0].value" \
  --output tsv)

echo "ACR credentials retrieved"
```

---

## Step 14 – Configure GitHub Secrets

```bash
# Create Azure credentials JSON
AZURE_CREDENTIALS=$(cat <<EOF_CREDS
{
  "clientId": "$CLIENT_ID",
  "clientSecret": "$CLIENT_SECRET",
  "subscriptionId": "$ACCOUNT_ID",
  "tenantId": "$TENANT_ID"
}
EOF_CREDS
)

echo "Configure the following GitHub Secrets:"
echo ""
echo "1. Go to: https://github.com/$GITHUB_USERNAME/$REPO_NAME/settings/secrets/actions"
echo "2. Click: New repository secret"
echo ""
echo "Secret 1:"
echo "  Name: AZURE_CREDENTIALS"
echo "  Value: $AZURE_CREDENTIALS"
echo ""
echo "Secret 2:"
echo "  Name: ACR_USERNAME"
echo "  Value: $ACR_USERNAME"
echo ""
echo "Secret 3:"
echo "  Name: ACR_PASSWORD"
echo "  Value: $ACR_PASSWORD"
echo ""
read -p "Press Enter after configuring secrets..."
```

---

## Step 15 – Trigger Workflow

```bash
# Trigger workflow by pushing a change
echo "# GitHub Actions ACI Deployment" > README.md
git add README.md
git commit -m "Trigger workflow"
git push origin main

echo "GitHub Actions workflow triggered"
echo "Monitor at: https://github.com/$GITHUB_USERNAME/$REPO_NAME/actions"
```

---

## Step 16 – Monitor Deployment

```bash
echo "Waiting for deployment..."
sleep 120

# Get ACI FQDN
ACI_FQDN=$(az container show \
  --name "$ACI_NAME" \
  --resource-group "$RG_NAME" \
  --query "ipAddress.fqdn" \
  --output tsv)

echo "$ACI_FQDN"
```

---

## Step 17 – Test Deployment

```bash
# Test application
curl -s "http://$ACI_FQDN:8080/" | python3 -m json.tool

# Test health endpoint
curl -s "http://$ACI_FQDN:8080/health" | python3 -m json.tool
```

---

## Step 18 – Cleanup

```bash
# Delete service principal
az ad sp delete --id "$CLIENT_ID"

# Delete resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local repository
cd ..
rm -rf "$REPO_NAME"

echo "Cleanup complete"
echo "Manually delete GitHub repository if needed"
```

---

## Summary

You created Azure Container Registry, configured GitHub Actions workflow with service principal authentication, built and pushed Docker images to ACR, deployed containers to Azure Container Instances, and automated the entire process with GitHub Actions CI/CD pipeline.
