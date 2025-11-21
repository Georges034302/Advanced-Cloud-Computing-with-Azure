# Lab 10.A: Azure DevOps Web App Pipeline

## Objectives
- Create Azure DevOps organization and project
- Create Azure App Service
- Configure service connection
- Create CI/CD pipeline
- Deploy web application
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
RG_NAME="rg-lab10a-devops"
PLAN_NAME="plan-devops"
WEBAPP_NAME="webapp-devops-$RANDOM"
REPO_NAME="devops-webapp"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
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

## Step 3 – Create App Service Plan

```bash
# Create App Service plan
az appservice plan create \
  --name "$PLAN_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku B1 \
  --is-linux

echo "App Service plan created"
```

---

## Step 4 – Create Web App

```bash
# Create web app
az webapp create \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --plan "$PLAN_NAME" \
  --runtime "PYTHON:3.11"

# Get web app URL
WEBAPP_URL=$(az webapp show \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --query defaultHostName \
  --output tsv)

echo "$WEBAPP_URL"
```

---

## Step 5 – Create Application Code

```bash
# Create application directory
mkdir -p "$REPO_NAME"
cd "$REPO_NAME"

# Create Flask application
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
import datetime

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'message': 'Azure DevOps CI/CD Pipeline',
        'version': '1.0',
        'timestamp': datetime.datetime.utcnow().isoformat(),
        'environment': os.getenv('ENVIRONMENT', 'production')
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF

# Create requirements.txt
cat > requirements.txt << 'EOF'
Flask==3.0.0
gunicorn==21.2.0
EOF

echo "Application code created"
```

---

## Step 6 – Create Azure Pipeline YAML

```bash
# Create Azure Pipelines configuration
cat > azure-pipelines.yml << EOF
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  azureSubscription: 'AzureServiceConnection'
  webAppName: '$WEBAPP_NAME'
  resourceGroup: '$RG_NAME'

stages:
  - stage: Build
    displayName: 'Build Application'
    jobs:
      - job: BuildJob
        displayName: 'Build'
        steps:
          - task: UsePythonVersion@0
            inputs:
              versionSpec: '3.11'
            displayName: 'Use Python 3.11'

          - script: |
              python -m pip install --upgrade pip
              pip install -r requirements.txt
            displayName: 'Install dependencies'

          - script: |
              pip install pytest
              pytest --version
            displayName: 'Run tests'

          - task: ArchiveFiles@2
            inputs:
              rootFolderOrFile: '\$(Build.SourcesDirectory)'
              includeRootFolder: false
              archiveType: 'zip'
              archiveFile: '\$(Build.ArtifactStagingDirectory)/\$(Build.BuildId).zip'
            displayName: 'Archive application'

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '\$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'drop'
            displayName: 'Publish artifact'

  - stage: Deploy
    displayName: 'Deploy to Azure'
    dependsOn: Build
    jobs:
      - deployment: DeployWeb
        displayName: 'Deploy Web App'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: '\$(azureSubscription)'
                    appType: 'webAppLinux'
                    appName: '\$(webAppName)'
                    package: '\$(Pipeline.Workspace)/drop/\$(Build.BuildId).zip'
                    runtimeStack: 'PYTHON|3.11'
                    startUpCommand: 'gunicorn --bind=0.0.0.0 --timeout 600 app:app'
                  displayName: 'Deploy to App Service'
EOF

echo "Azure Pipeline YAML created"
```

---

## Step 7 – Initialize Git Repository

```bash
# Initialize git repository
git init
git add .
git commit -m "Initial commit with Azure DevOps pipeline"

echo "Git repository initialized"
```

---

## Step 8 – Create GitHub Repository

```bash
# Create GitHub repository (requires gh CLI)
if command -v gh &> /dev/null; then
  gh repo create "$REPO_NAME" --public --source=. --remote=origin --push
  echo "GitHub repository created"
else
  echo "GitHub CLI not installed. Create repository manually at https://github.com/new"
  echo "Then run: git remote add origin https://github.com/YOUR_USERNAME/$REPO_NAME.git"
  echo "Then run: git push -u origin main"
fi
```

---

## Step 9 – Create Service Principal

```bash
# Get subscription ID
SUBSCRIPTION_ID=$(az account show --query id --output tsv)

echo "$SUBSCRIPTION_ID"

# Create service principal for Azure DevOps
SP_OUTPUT=$(az ad sp create-for-rbac \
  --name "sp-devops-webapp" \
  --role Contributor \
  --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME" \
  --output json)

# Extract credentials
CLIENT_ID=$(echo "$SP_OUTPUT" | python3 -c "import sys, json; print(json.load(sys.stdin)['appId'])")
CLIENT_SECRET=$(echo "$SP_OUTPUT" | python3 -c "import sys, json; print(json.load(sys.stdin)['password'])")
TENANT_ID=$(echo "$SP_OUTPUT" | python3 -c "import sys, json; print(json.load(sys.stdin)['tenant'])")

echo "Service principal created"
echo "Client ID: $CLIENT_ID"
echo "Tenant ID: $TENANT_ID"
```

---

## Step 10 – Setup Azure DevOps

```bash
# Install Azure DevOps extension
az extension add --name azure-devops

# Configure Azure DevOps defaults
echo "Visit https://dev.azure.com to create an organization and project"
echo ""
echo "Organization name: Choose a unique name"
echo "Project name: devops-webapp-project"
echo ""
read -p "Press Enter after creating organization and project..."

# Set organization URL
read -p "Enter your Azure DevOps organization name: " DEVOPS_ORG
DEVOPS_ORG_URL="https://dev.azure.com/$DEVOPS_ORG"

echo "$DEVOPS_ORG_URL"
```

---

## Step 11 – Configure Service Connection

```bash
echo "Configure Service Connection in Azure DevOps:"
echo ""
echo "1. Go to: $DEVOPS_ORG_URL/devops-webapp-project/_settings/adminservices"
echo "2. Click: New service connection"
echo "3. Select: Azure Resource Manager"
echo "4. Choose: Service principal (manual)"
echo "5. Enter the following:"
echo "   - Subscription ID: $SUBSCRIPTION_ID"
echo "   - Subscription Name: (your subscription name)"
echo "   - Service Principal ID: $CLIENT_ID"
echo "   - Service Principal Key: $CLIENT_SECRET"
echo "   - Tenant ID: $TENANT_ID"
echo "   - Service connection name: AzureServiceConnection"
echo "6. Click: Verify and save"
echo ""
read -p "Press Enter after configuring service connection..."
```

---

## Step 12 – Create Azure Pipeline

```bash
echo "Create Pipeline in Azure DevOps:"
echo ""
echo "1. Go to: $DEVOPS_ORG_URL/devops-webapp-project/_build"
echo "2. Click: New pipeline"
echo "3. Select: GitHub"
echo "4. Authorize Azure Pipelines to access your GitHub account"
echo "5. Select repository: $REPO_NAME"
echo "6. Select: Existing Azure Pipelines YAML file"
echo "7. Path: /azure-pipelines.yml"
echo "8. Click: Run"
echo ""
read -p "Press Enter after pipeline is created..."
```

---

## Step 13 – Monitor Pipeline Execution

```bash
echo "Monitor pipeline at:"
echo "$DEVOPS_ORG_URL/devops-webapp-project/_build"
echo ""
echo "Pipeline stages:"
echo "1. Build - Install dependencies and create artifact"
echo "2. Deploy - Deploy to Azure App Service"
echo ""
read -p "Press Enter after pipeline completes..."
```

---

## Step 14 – Test Deployed Application

```bash
# Wait for deployment
echo "Waiting for deployment..."
sleep 30

# Test application
curl -s "https://$WEBAPP_URL/" | python3 -m json.tool

# Test health endpoint
curl -s "https://$WEBAPP_URL/health" | python3 -m json.tool
```

---

## Step 15 – Update Application

```bash
# Update application version
sed -i "s/'version': '1.0'/'version': '2.0'/" app.py

# Commit and push changes
git add app.py
git commit -m "Update to version 2.0"
git push origin main

echo "Pipeline will automatically trigger"
```

---

## Step 16 – Cleanup

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
echo "Manually delete Azure DevOps project and GitHub repository if needed"
```

---

## Summary

You created an Azure DevOps organization and project, configured service connection with service principal, created a CI/CD pipeline using YAML, deployed a Python web application to Azure App Service, and automated deployment on code changes.
