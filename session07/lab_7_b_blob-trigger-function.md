# Lab 7.B: Blob Trigger Function

## Objectives
- Create Azure Function App
- Create blob storage container
- Deploy blob trigger function
- Upload files to trigger function
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
RG_NAME="rg-lab7b-blob"
STORAGE_ACCOUNT="storage$RANDOM$RANDOM"
FUNCTION_APP="func-blob-$RANDOM"
CONTAINER_INPUT="uploads"
CONTAINER_OUTPUT="processed"

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

## Step 4 – Create Blob Containers

```bash
# Create input container
az storage container create \
  --name "$CONTAINER_INPUT" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION"

# Create output container
az storage container create \
  --name "$CONTAINER_OUTPUT" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION"

echo "Containers created"
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
mkdir -p blob-function
cd blob-function

# Initialize function project
func init --worker-runtime python --model V2

echo "Function project initialized"
```

---

## Step 7 – Create Blob Trigger Function

```bash
# Create function code
cat > function_app.py << 'EOF'
import azure.functions as func
import logging
from datetime import datetime

app = func.FunctionApp()

@app.function_name(name="BlobTrigger")
@app.blob_trigger(arg_name="myblob", 
                  path="uploads/{name}",
                  connection="AzureWebJobsStorage")
@app.blob_output(arg_name="outputblob",
                path="processed/{name}",
                connection="AzureWebJobsStorage")
def blob_trigger(myblob: func.InputStream, outputblob: func.Out[bytes]):
    logging.info(f"Processing blob: {myblob.name}")
    logging.info(f"Blob size: {myblob.length} bytes")
    
    # Read blob content
    content = myblob.read()
    
    # Process content (convert to uppercase as example)
    try:
        text_content = content.decode('utf-8')
        processed_content = text_content.upper()
        
        # Add metadata
        metadata = f"\n\n--- Processed at {datetime.utcnow().isoformat()} ---\n"
        final_content = processed_content + metadata
        
        # Write to output blob
        outputblob.set(final_content.encode('utf-8'))
        
        logging.info(f"Successfully processed: {myblob.name}")
    except Exception as e:
        logging.error(f"Error processing blob: {str(e)}")
        # Still write original content to output
        outputblob.set(content)
EOF

echo "Blob trigger function created"
```

---

## Step 8 – Update Requirements

```bash
# Update requirements.txt
cat > requirements.txt << 'EOF'
azure-functions
azure-storage-blob
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

## Step 10 – Create Test Files

```bash
# Create test directory
mkdir -p test-files

# Create test file 1
cat > test-files/sample1.txt << 'EOF'
Hello from Azure Blob Storage!
This file will be processed by Azure Functions.
Timestamp: $(date)
EOF

# Create test file 2
cat > test-files/sample2.txt << 'EOF'
Azure Functions can automatically process files.
Blob triggers monitor storage containers.
This is a serverless solution.
EOF

echo "Test files created"
```

---

## Step 11 – Upload Files to Trigger Function

```bash
# Upload first test file
az storage blob upload \
  --container-name "$CONTAINER_INPUT" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --file test-files/sample1.txt \
  --name sample1.txt

echo "Uploaded sample1.txt"

# Wait for processing
sleep 10

# Upload second test file
az storage blob upload \
  --container-name "$CONTAINER_INPUT" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --file test-files/sample2.txt \
  --name sample2.txt

echo "Uploaded sample2.txt"

# Wait for processing
sleep 10
```

---

## Step 12 – Validate Blob Processing

```bash
# List blobs in input container
echo "Input container blobs:"
az storage blob list \
  --container-name "$CONTAINER_INPUT" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --query "[].name" \
  --output table

# List blobs in output container
echo "Output container blobs:"
az storage blob list \
  --container-name "$CONTAINER_OUTPUT" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --query "[].name" \
  --output table
```

---

## Step 13 – Download Processed Files

```bash
# Create output directory
mkdir -p processed-files

# Download processed file 1
az storage blob download \
  --container-name "$CONTAINER_OUTPUT" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --name sample1.txt \
  --file processed-files/sample1.txt

echo "Downloaded processed sample1.txt:"
cat processed-files/sample1.txt

echo ""

# Download processed file 2
az storage blob download \
  --container-name "$CONTAINER_OUTPUT" \
  --account-name "$STORAGE_ACCOUNT" \
  --connection-string "$STORAGE_CONNECTION" \
  --name sample2.txt \
  --file processed-files/sample2.txt

echo "Downloaded processed sample2.txt:"
cat processed-files/sample2.txt
```

---

## Step 14 – View Function Logs

```bash
# Stream function logs
echo "Recent function logs:"
az functionapp log tail \
  --name "$FUNCTION_APP" \
  --resource-group "$RG_NAME" &

sleep 15
kill %1 2>/dev/null
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
rm -rf blob-function test-files processed-files

echo "Cleanup complete"
```

---

## Summary

You created an Azure Function with blob trigger, uploaded files to blob storage, automatically processed files with the function, and verified processed output in a separate container.
