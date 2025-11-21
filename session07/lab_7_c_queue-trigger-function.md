# Lab 7.C: Queue Trigger Function

## Objectives
- Create Azure Function App
- Create storage queue
- Deploy queue trigger function
- Send messages to queue
- View function execution logs
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
RG_NAME="rg-lab7c-queue"
STORAGE_ACCOUNT="storage$RANDOM$RANDOM"
FUNCTION_APP="func-queue-$RANDOM"
QUEUE_NAME="tasks"

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

# Get connection string
STORAGE_CONNECTION=$(az storage account show-connection-string \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query connectionString \
  --output tsv)

echo "Storage account created"
```

---

## Step 4 – Create Storage Queue

```bash
# Create queue
az storage queue create \
  --name "$QUEUE_NAME" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION"

echo "Queue created"
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
mkdir -p queue-function
cd queue-function

# Initialize function project
func init --worker-runtime python --model V2

echo "Function project initialized"
```

---

## Step 7 – Create Queue Trigger Function

```bash
# Create function code
cat > function_app.py << 'EOF'
import azure.functions as func
import logging
import json
from datetime import datetime

app = func.FunctionApp()

@app.function_name(name="QueueTrigger")
@app.queue_trigger(arg_name="msg", 
                   queue_name="tasks",
                   connection="AzureWebJobsStorage")
@app.queue_output(arg_name="outputmsg",
                 queue_name="tasks-processed",
                 connection="AzureWebJobsStorage")
def queue_trigger(msg: func.QueueMessage, outputmsg: func.Out[str]):
    logging.info('Processing queue message')
    
    # Get message content
    message_body = msg.get_body().decode('utf-8')
    logging.info(f'Message content: {message_body}')
    
    try:
        # Parse JSON message
        data = json.loads(message_body)
        
        # Process the task
        task_type = data.get('type', 'unknown')
        task_data = data.get('data', {})
        
        logging.info(f'Processing task type: {task_type}')
        
        # Create result
        result = {
            'original_message': data,
            'processed_at': datetime.utcnow().isoformat(),
            'status': 'completed',
            'message_id': msg.id,
            'dequeue_count': msg.dequeue_count
        }
        
        # Send result to output queue
        outputmsg.set(json.dumps(result))
        
        logging.info(f'Successfully processed message: {msg.id}')
        
    except json.JSONDecodeError:
        logging.warning(f'Message is not valid JSON: {message_body}')
        # Process as plain text
        result = {
            'message': message_body,
            'processed_at': datetime.utcnow().isoformat(),
            'status': 'processed_as_text'
        }
        outputmsg.set(json.dumps(result))
    except Exception as e:
        logging.error(f'Error processing message: {str(e)}')
        raise
EOF

echo "Queue trigger function created"
```

---

## Step 8 – Update Requirements

```bash
# Update requirements.txt
cat > requirements.txt << 'EOF'
azure-functions
azure-storage-queue
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

## Step 10 – Create Output Queue

```bash
# Create output queue for processed messages
az storage queue create \
  --name "tasks-processed" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION"

echo "Output queue created"
```

---

## Step 11 – Send Messages to Queue

```bash
# Send JSON message 1
az storage message put \
  --queue-name "$QUEUE_NAME" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --content '{"type":"email","data":{"to":"user@example.com","subject":"Test"}}'

echo "Sent message 1"

# Wait for processing
sleep 5

# Send JSON message 2
az storage message put \
  --queue-name "$QUEUE_NAME" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --content '{"type":"notification","data":{"user":"admin","message":"System update"}}'

echo "Sent message 2"

# Wait for processing
sleep 5

# Send plain text message
az storage message put \
  --queue-name "$QUEUE_NAME" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --content "Plain text message for processing"

echo "Sent message 3"

# Wait for processing
sleep 5
```

---

## Step 12 – Validate Queue Processing

```bash
# Check input queue (should be empty)
echo "Input queue message count:"
az storage queue stats \
  --queue-name "$QUEUE_NAME" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --query approximateMessageCount

# Check output queue
echo "Output queue message count:"
az storage queue stats \
  --queue-name "tasks-processed" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --query approximateMessageCount
```

---

## Step 13 – View Processed Messages

```bash
# Peek messages from output queue
echo "Processed messages:"
az storage message peek \
  --queue-name "tasks-processed" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --num-messages 10 \
  --query "[].content" \
  --output tsv | while read msg; do
    echo "$msg" | python3 -m json.tool 2>/dev/null || echo "$msg"
    echo "---"
  done
```

---

## Step 14 – View Function Logs

```bash
# Stream function logs
echo "Function execution logs:"
az functionapp log tail \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" &

sleep 15
kill %1 2>/dev/null
```

---

## Step 15 – Send Batch Messages

```bash
# Send multiple messages in batch
echo "Sending batch messages:"
for i in {1..5}; do
  az storage message put \
    --queue-name "$QUEUE_NAME" \
    --account-name "$STORAGE_ACCOUNT" \
    --connection-string "$STORAGE_CONNECTION" \
    --content "{\"type\":\"batch\",\"data\":{\"index\":$i,\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}}" \
    --output none
  echo "Sent batch message $i"
done

# Wait for processing
sleep 10

# Check queue stats
echo "Final queue stats:"
az storage queue stats \
  --queue-name "tasks-processed" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --query approximateMessageCount
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
rm -rf queue-function

echo "Cleanup complete"
```

---

## Summary

You created an Azure Function with queue trigger, sent JSON and plain text messages to storage queue, automatically processed messages with the function, and sent results to an output queue.
