# Lab 9.C: Bicep Serverless API

## Objectives
- Create Bicep template for serverless API
- Deploy Azure Function App
- Deploy API Management service
- Configure API policies
- Validate serverless API
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab9c-serverless"
DEPLOYMENT_NAME="serverless-api-deployment"
BICEP_FILE="serverless-api.bicep"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$DEPLOYMENT_NAME"
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

## Step 3 – Create Bicep Template

```bash
# Create Bicep template for serverless API
cat > "$BICEP_FILE" << 'EOF'
@description('Location for all resources')
param location string = resourceGroup().location

@description('Function App name')
param functionAppName string = 'func-${uniqueString(resourceGroup().id)}'

@description('Storage account name for Function App')
param storageAccountName string = 'stfunc${uniqueString(resourceGroup().id)}'

@description('API Management service name')
param apimServiceName string = 'apim-${uniqueString(resourceGroup().id)}'

@description('Publisher email for API Management')
param publisherEmail string

@description('Publisher name for API Management')
param publisherName string = 'API Publisher'

var hostingPlanName = 'plan-${functionAppName}'
var applicationInsightsName = 'ai-${functionAppName}'
var functionWorkerRuntime = 'python'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
  }
}

resource hostingPlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: hostingPlanName
  location: location
  sku: {
    name: 'Y1'
    tier: 'Dynamic'
  }
  properties: {}
}

resource applicationInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: applicationInsightsName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    Request_Source: 'rest'
  }
}

resource functionApp 'Microsoft.Web/sites@2022-09-01' = {
  name: functionAppName
  location: location
  kind: 'functionapp'
  properties: {
    serverFarmId: hostingPlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTSHARE'
          value: toLower(functionAppName)
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        {
          name: 'APPINSIGHTS_INSTRUMENTATIONKEY'
          value: applicationInsights.properties.InstrumentationKey
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: functionWorkerRuntime
        }
      ]
      ftpsState: 'FtpsOnly'
      minTlsVersion: '1.2'
    }
    httpsOnly: true
  }
}

resource apimService 'Microsoft.ApiManagement/service@2023-03-01-preview' = {
  name: apimServiceName
  location: location
  sku: {
    name: 'Consumption'
    capacity: 0
  }
  properties: {
    publisherEmail: publisherEmail
    publisherName: publisherName
  }
}

resource apimApi 'Microsoft.ApiManagement/service/apis@2023-03-01-preview' = {
  parent: apimService
  name: 'serverless-api'
  properties: {
    displayName: 'Serverless API'
    apiRevision: '1'
    subscriptionRequired: true
    serviceUrl: 'https://${functionApp.properties.defaultHostName}/api'
    path: 'api'
    protocols: [
      'https'
    ]
  }
}

resource apimApiOperation 'Microsoft.ApiManagement/service/apis/operations@2023-03-01-preview' = {
  parent: apimApi
  name: 'get-status'
  properties: {
    displayName: 'Get Status'
    method: 'GET'
    urlTemplate: '/status'
    templateParameters: []
    responses: []
  }
}

resource apimPolicy 'Microsoft.ApiManagement/service/apis/policies@2023-03-01-preview' = {
  parent: apimApi
  name: 'policy'
  properties: {
    value: '''
      <policies>
        <inbound>
          <base />
          <rate-limit calls="10" renewal-period="60" />
          <cors>
            <allowed-origins>
              <origin>*</origin>
            </allowed-origins>
            <allowed-methods>
              <method>GET</method>
              <method>POST</method>
            </allowed-methods>
          </cors>
        </inbound>
        <backend>
          <base />
        </backend>
        <outbound>
          <base />
        </outbound>
        <on-error>
          <base />
        </on-error>
      </policies>
    '''
    format: 'xml'
  }
}

output functionAppName string = functionApp.name
output functionAppUrl string = 'https://${functionApp.properties.defaultHostName}'
output apimServiceName string = apimService.name
output apimGatewayUrl string = apimService.properties.gatewayUrl
output storageAccountName string = storageAccount.name
EOF

echo "Bicep template created"
```

---

## Step 4 – Create Parameters File

```bash
# Create parameters file
cat > parameters.json << 'EOF'
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "publisherEmail": {
      "value": "admin@example.com"
    },
    "publisherName": {
      "value": "Lab API Publisher"
    }
  }
}
EOF

echo "Parameters file created"
```

---

## Step 5 – Validate Bicep Template

```bash
# Build Bicep template to check syntax
az bicep build --file "$BICEP_FILE"

echo "Bicep template validated"
```

---

## Step 6 – Deploy Bicep Template

```bash
# Deploy Bicep template
az deployment group create \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --template-file "$BICEP_FILE" \
  --parameters parameters.json

echo "Deployment started"
```

---

## Step 7 – Monitor Deployment

```bash
# Check deployment status
az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.provisioningState" \
  --output tsv

# List deployment operations
az deployment operation group list \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].{Resource:properties.targetResource.resourceType, State:properties.provisioningState}" \
  --output table
```

---

## Step 8 – Get Deployment Outputs

```bash
# Get Function App name
FUNCTION_APP_NAME=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.functionAppName.value" \
  --output tsv)

echo "$FUNCTION_APP_NAME"

# Get Function App URL
FUNCTION_APP_URL=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.functionAppUrl.value" \
  --output tsv)

echo "$FUNCTION_APP_URL"

# Get APIM service name
APIM_SERVICE_NAME=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.apimServiceName.value" \
  --output tsv)

echo "$APIM_SERVICE_NAME"

# Get APIM gateway URL
APIM_GATEWAY_URL=$(az deployment group show \
  --name "$DEPLOYMENT_NAME" \
  --resource-group "$RG_NAME" \
  --query "properties.outputs.apimGatewayUrl.value" \
  --output tsv)

echo "$APIM_GATEWAY_URL"
```

---

## Step 9 – Create Function Code

```bash
# Create function directory
mkdir -p function-code
cd function-code

# Create host.json
cat > host.json << 'EOF'
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
EOF

# Create requirements.txt
cat > requirements.txt << 'EOF'
azure-functions
EOF

# Create function directory
mkdir -p StatusFunction

# Create function.json
cat > StatusFunction/function.json << 'EOF'
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post"],
      "route": "status"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ]
}
EOF

# Create function code
cat > StatusFunction/__init__.py << 'EOF'
import azure.functions as func
import json
import datetime

def main(req: func.HttpRequest) -> func.HttpResponse:
    return func.HttpResponse(
        json.dumps({
            'status': 'healthy',
            'timestamp': datetime.datetime.utcnow().isoformat(),
            'service': 'Serverless API',
            'version': '1.0'
        }),
        mimetype='application/json',
        status_code=200
    )
EOF

echo "Function code created"
```

---

## Step 10 – Deploy Function Code

```bash
# Create deployment package
zip -r function-app.zip . -x "*.git*"

# Deploy function code
az functionapp deployment source config-zip \
  --name "$FUNCTION_APP_NAME" \
  --resource-group "$RG_NAME" \
  --src function-app.zip

cd ..
echo "Function code deployed"
```

---

## Step 11 – Get Function Key

```bash
# Wait for function to be ready
echo "Waiting for function to be ready..."
sleep 60

# Get function key
FUNCTION_KEY=$(az functionapp keys list \
  --name "$FUNCTION_APP_NAME" \
  --resource-group "$RG_NAME" \
  --query "functionKeys.default" \
  --output tsv)

echo "$FUNCTION_KEY"
```

---

## Step 12 – Test Function Directly

```bash
# Test function endpoint
curl -s "${FUNCTION_APP_URL}/api/status?code=${FUNCTION_KEY}" | python3 -m json.tool
```

---

## Step 13 – Get APIM Subscription Key

```bash
# Get APIM subscription key
APIM_SUBSCRIPTION_KEY=$(az rest \
  --method post \
  --uri "https://management.azure.com/subscriptions/$(az account show --query id -o tsv)/resourceGroups/${RG_NAME}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/subscriptions/master/listSecrets?api-version=2023-03-01-preview" \
  --query "primaryKey" \
  --output tsv)

echo "$APIM_SUBSCRIPTION_KEY"
```

---

## Step 14 – Test API Through APIM

```bash
# Test API through API Management
curl -s -H "Ocp-Apim-Subscription-Key: ${APIM_SUBSCRIPTION_KEY}" \
  "${APIM_GATEWAY_URL}/api/status" | python3 -m json.tool
```

---

## Step 15 – Validate APIM Configuration

```bash
# List APIs in APIM
az apim api list \
  --service-name "$APIM_SERVICE_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, DisplayName:displayName, Path:path}" \
  --output table

# Show API operations
az apim api operation list \
  --service-name "$APIM_SERVICE_NAME" \
  --resource-group "$RG_NAME" \
  --api-id "serverless-api" \
  --query "[].{Name:name, Method:method, Path:urlTemplate}" \
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

# Remove local files
rm -f "$BICEP_FILE" parameters.json
rm -rf function-code

echo "Cleanup complete"
```

---

## Summary

You created a Bicep template defining a serverless API with Azure Functions and API Management, deployed the infrastructure and function code, configured API policies for rate limiting and CORS, and validated the complete serverless API stack.
