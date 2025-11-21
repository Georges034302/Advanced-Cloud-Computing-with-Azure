# Lab 7.D: Event Grid Function

## Objectives
- Create Azure Function App
- Create Event Grid topic
- Deploy Event Grid trigger function
- Subscribe function to Event Grid events
- Send custom events
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
RG_NAME="rg-lab7d-eventgrid"
STORAGE_ACCOUNT="storage$RANDOM$RANDOM"
FUNCTION_APP="func-events-$RANDOM"
TOPIC_NAME="topic-events"
SUBSCRIPTION_NAME="sub-function"

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
# Create storage account
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

echo "Storage account created"
```

---

## Step 4 – Create Event Grid Topic

```bash
# Create Event Grid topic
az eventgrid topic create \
  --name "$TOPIC_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION"

# Get topic endpoint
TOPIC_ENDPOINT=$(az eventgrid topic show \
  --name "$TOPIC_NAME" \
  --resource-group "$RG_NAME" \
  --query endpoint \
  --output tsv)

echo "$TOPIC_ENDPOINT"

# Get topic key
TOPIC_KEY=$(az eventgrid topic key list \
  --name "$TOPIC_NAME" \
  --resource-group "$RG_NAME" \
  --query key1 \
  --output tsv)

echo "Event Grid topic created"
```

---

## Step 5 – Create Function App

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

## Step 6 – Create Local Function Project

```bash
# Create project directory
mkdir -p eventgrid-function
cd eventgrid-function

# Initialize function project
func init --worker-runtime python --model V2

echo "Function project initialized"
```

---

## Step 7 – Create Event Grid Trigger Function

```bash
# Create function code
cat > function_app.py << 'EOF'
import azure.functions as func
import logging
import json
from datetime import datetime

app = func.FunctionApp()

@app.function_name(name="EventGridTrigger")
@app.event_grid_trigger(arg_name="event")
def eventgrid_trigger(event: func.EventGridEvent):
    logging.info('Processing Event Grid event')
    
    # Get event properties
    event_id = event.id
    event_type = event.event_type
    event_subject = event.subject
    event_time = event.event_time
    
    logging.info(f'Event ID: {event_id}')
    logging.info(f'Event Type: {event_type}')
    logging.info(f'Event Subject: {event_subject}')
    logging.info(f'Event Time: {event_time}')
    
    # Get event data
    event_data = event.get_json()
    logging.info(f'Event Data: {json.dumps(event_data, indent=2)}')
    
    # Process based on event type
    if event_type == 'Custom.Alert.Critical':
        logging.warning(f'CRITICAL ALERT: {event_data.get("message", "No message")}')
        # Here you would send notifications, create tickets, etc.
    elif event_type == 'Custom.Deployment.Completed':
        logging.info(f'Deployment completed: {event_data.get("service", "unknown")}')
    elif event_type == 'Custom.User.Action':
        logging.info(f'User action: {event_data.get("action", "unknown")}')
    else:
        logging.info(f'Generic event processed: {event_type}')
    
    # Log processing completion
    logging.info(f'Successfully processed event: {event_id}')
EOF

echo "Event Grid trigger function created"
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

## Step 10 – Get Function Endpoint

```bash
# Get function app resource ID
FUNCTION_ID=$(az functionapp show \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$FUNCTION_ID"
```

---

## Step 11 – Create Event Grid Subscription

```bash
# Create event subscription
az eventgrid event-subscription create \
  --name "$SUBSCRIPTION_NAME" \
  --source-resource-id "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.EventGrid/topics/$TOPIC_NAME" \
  --endpoint "${FUNCTION_ID}/functions/EventGridTrigger" \
  --endpoint-type azurefunction

echo "Event subscription created"
```

---

## Step 12 – Send Custom Events

```bash
# Send critical alert event
az eventgrid event publish \
  --topic-name "$TOPIC_NAME" \
  --resource-group "$RG_NAME" \
  --events "[
    {
      \"id\": \"$(uuidgen)\",
      \"eventType\": \"Custom.Alert.Critical\",
      \"subject\": \"monitoring/server01\",
      \"eventTime\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
      \"data\": {
        \"message\": \"CPU usage exceeded 90%\",
        \"server\": \"server01\",
        \"metric\": \"cpu\",
        \"value\": 95
      },
      \"dataVersion\": \"1.0\"
    }
  ]"

echo "Sent critical alert event"
sleep 5

# Send deployment event
az eventgrid event publish \
  --topic-name "$TOPIC_NAME" \
  --resource-group "$RG_NAME" \
  --events "[
    {
      \"id\": \"$(uuidgen)\",
      \"eventType\": \"Custom.Deployment.Completed\",
      \"subject\": \"deployment/web-app\",
      \"eventTime\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
      \"data\": {
        \"service\": \"web-app\",
        \"version\": \"2.1.0\",
        \"status\": \"success\",
        \"duration\": 120
      },
      \"dataVersion\": \"1.0\"
    }
  ]"

echo "Sent deployment event"
sleep 5

# Send user action event
az eventgrid event publish \
  --topic-name "$TOPIC_NAME" \
  --resource-group "$RG_NAME" \
  --events "[
    {
      \"id\": \"$(uuidgen)\",
      \"eventType\": \"Custom.User.Action\",
      \"subject\": \"user/admin\",
      \"eventTime\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
      \"data\": {
        \"user\": \"admin\",
        \"action\": \"create_resource\",
        \"resource\": \"storage_account\",
        \"result\": \"success\"
      },
      \"dataVersion\": \"1.0\"
    }
  ]"

echo "Sent user action event"
sleep 5
```

---

## Step 13 – Validate Event Processing

```bash
# View function logs
echo "Function execution logs:"
az functionapp log tail \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" &

sleep 15
kill %1 2>/dev/null
```

---

## Step 14 – Send Batch Events

```bash
# Send multiple events
echo "Sending batch events:"
for i in {1..3}; do
  az eventgrid event publish \
    --topic-name "$TOPIC_NAME" \
    --resource-group "$RG_NAME" \
    --events "[
      {
        \"id\": \"$(uuidgen)\",
        \"eventType\": \"Custom.Alert.Critical\",
        \"subject\": \"monitoring/server0${i}\",
        \"eventTime\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
        \"data\": {
          \"message\": \"Memory usage high\",
          \"server\": \"server0${i}\",
          \"metric\": \"memory\",
          \"value\": 85
        },
        \"dataVersion\": \"1.0\"
      }
    ]" \
    --output none
  echo "Sent event $i"
  sleep 2
done
```

---

## Step 15 – View Event Subscription

```bash
# Show event subscription details
az eventgrid event-subscription show \
  --name "$SUBSCRIPTION_NAME" \
  --source-resource-id "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.EventGrid/topics/$TOPIC_NAME" \
  --query "{Name:name, Endpoint:destination.endpointType, Status:provisioningState}" \
  --output table

# List all event subscriptions
az eventgrid event-subscription list \
  --source-resource-id "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.EventGrid/topics/$TOPIC_NAME" \
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
rm -rf eventgrid-function

echo "Cleanup complete"
```

---

## Summary

You created an Azure Function with Event Grid trigger, created an Event Grid topic, subscribed the function to the topic, sent custom events with different types, and validated event processing through function logs.
