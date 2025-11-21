# Lab 7.A: HTTP Trigger Function

## Objectives
- Create Azure Function App
- Deploy HTTP trigger function
- Configure function settings
- Test HTTP endpoints
- View function logs
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure Functions Core Tools installed (`func --version`)
- Azure subscription with contributor access
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab7a-functions"
STORAGE_ACCOUNT="storage$RANDOM$RANDOM"
FUNCTION_APP="func-http-$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$FUNCTION_APP"
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

## Step 3 – Create Storage Account

```bash
# Create storage account for function app
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

echo "Storage account created"
```

---

## Step 4 – Create Function App

```bash
# Create Python function app
az functionapp create \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" \
  --storage-account "$STORAGE_ACCOUNT" \
  --consumption-plan-location "$LOCATION" \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --os-type Linux

echo "Function app created"
```

---

## Step 5 – Create Local Function Project

```bash
# Create project directory
mkdir -p http-function
cd http-function

# Initialize function project
func init --worker-runtime python --model V2

echo "Function project initialized"
```

---

## Step 6 – Create HTTP Trigger Function

```bash
# Create HTTP trigger function
func new --name HttpTrigger --template "HTTP trigger" --authlevel "function"

echo "HTTP trigger function created"
```

---

## Step 7 – Update Function Code

```bash
# Create custom function code
cat > function_app.py << 'EOF'
import azure.functions as func
import logging
import json
from datetime import datetime

app = func.FunctionApp()

@app.function_name(name="HttpTrigger")
@app.route(route="hello", auth_level=func.AuthLevel.FUNCTION)
def http_trigger(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Processing HTTP request')
    
    name = req.params.get('name')
    if not name:
        try:
            req_body = req.get_json()
            name = req_body.get('name')
        except ValueError:
            pass
    
    if name:
        response_data = {
            'message': f'Hello, {name}!',
            'timestamp': datetime.utcnow().isoformat(),
            'status': 'success'
        }
        return func.HttpResponse(
            json.dumps(response_data),
            mimetype="application/json",
            status_code=200
        )
    else:
        return func.HttpResponse(
            json.dumps({'error': 'Please provide a name parameter'}),
            mimetype="application/json",
            status_code=400
        )

@app.function_name(name="HealthCheck")
@app.route(route="health", auth_level=func.AuthLevel.ANONYMOUS)
def health_check(req: func.HttpRequest) -> func.HttpResponse:
    return func.HttpResponse(
        json.dumps({'status': 'healthy', 'timestamp': datetime.utcnow().isoformat()}),
        mimetype="application/json",
        status_code=200
    )
EOF

echo "Function code updated"
```

---

## Step 8 – Update Requirements

```bash
# Update requirements.txt
cat > requirements.txt << 'EOF'
azure-functions
EOF

echo "Requirements updated"
```

---

## Step 9 – Deploy Function to Azure

```bash
# Deploy function app
func azure functionapp publish "$FUNCTION_APP" --python

cd ..
echo "Function deployed"
```

---

## Step 10 – Get Function URL

```bash
# Get function app hostname
FUNCTION_HOST=$(az functionapp show \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" \
  --query defaultHostName \
  --output tsv)

echo "$FUNCTION_HOST"

# Get function key
FUNCTION_KEY=$(az functionapp function keys list \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" \
  --function-name HttpTrigger \
  --query default \
  --output tsv)

echo "Function key retrieved"
```

---

## Step 11 – Validate HTTP Trigger

```bash
# Test health endpoint
echo "Testing health endpoint:"
curl -s "https://${FUNCTION_HOST}/api/health" | python3 -m json.tool

# Test HTTP trigger with GET request
echo "Testing GET request:"
curl -s "https://${FUNCTION_HOST}/api/hello?code=${FUNCTION_KEY}&name=Azure" | python3 -m json.tool

# Test HTTP trigger with POST request
echo "Testing POST request:"
curl -s -X POST "https://${FUNCTION_HOST}/api/hello?code=${FUNCTION_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"name":"Functions"}' | python3 -m json.tool
```

---

## Step 12 – View Function Logs

```bash
# Enable application insights
az functionapp config appsettings set \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" \
  --settings FUNCTIONS_WORKER_RUNTIME=python

# Stream logs
echo "Streaming logs (press Ctrl+C to stop):"
az functionapp log tail \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" &

sleep 15
kill %1 2>/dev/null
```

---

## Step 13 – Test Error Handling

```bash
# Test without name parameter
echo "Testing error handling:"
curl -s "https://${FUNCTION_HOST}/api/hello?code=${FUNCTION_KEY}" | python3 -m json.tool
```

---

## Step 14 – View Function Configuration

```bash
# Show function app settings
az functionapp config appsettings list \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" \
  --output table

# Show function app details
az functionapp show \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, State:state, Runtime:siteConfig.linuxFxVersion}" \
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
rm -rf http-function

echo "Cleanup complete"
```

---

## Summary

You created an Azure Function App with HTTP trigger, deployed custom Python functions with GET and POST support, tested endpoints with query parameters and JSON body, and validated error handling.
