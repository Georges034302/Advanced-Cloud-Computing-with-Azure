# Lab 4.B: Cosmos DB SQL API

## Overview
This lab demonstrates Azure Cosmos DB with SQL API for NoSQL database operations. You'll create a Cosmos DB account, database, and container, then perform CRUD operations using Azure CLI and Data Explorer.

---

## Objectives
- Create Azure Cosmos DB account with SQL API
- Create database and container with partition key
- Insert, query, update, and delete items
- Validate throughput (RU/s) settings
- Monitor Cosmos DB metrics
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Basic JSON and NoSQL knowledge
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab4b-cosmos"
COSMOS_ACCOUNT="cosmos-lab4b-$RANDOM"
COSMOS_DB="ProductsDB"
COSMOS_CONTAINER="Products"
PARTITION_KEY="/category"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "COSMOS_ACCOUNT=$COSMOS_ACCOUNT"
echo "COSMOS_DB=$COSMOS_DB"
echo "COSMOS_CONTAINER=$COSMOS_CONTAINER"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for Cosmos DB resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Cosmos DB Account

```bash
# Create Cosmos DB account with SQL API
az cosmosdb create \
  --name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --locations regionName="$LOCATION" \
  --kind GlobalDocumentDB \
  --default-consistency-level Session

# Verify Cosmos DB account creation
az cosmosdb show \
  --name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, Kind:kind, ConsistencyLevel:consistencyPolicy.defaultConsistencyLevel}" \
  --output table

echo "⏳ Cosmos DB account creation can take 3-5 minutes"
```

---

## Step 4 – Get Cosmos DB Connection Details

```bash
# Get Cosmos DB endpoint
COSMOS_ENDPOINT=$(az cosmosdb show \
  --name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query documentEndpoint \
  --output tsv)

echo "Cosmos DB Endpoint: $COSMOS_ENDPOINT"

# Get primary key
COSMOS_KEY=$(az cosmosdb keys list \
  --name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query primaryMasterKey \
  --output tsv)

echo "Primary Key: ${COSMOS_KEY:0:20}..."
echo "⚠️  Keep this key secure"
```

---

## Step 5 – Create Database

```bash
# Create SQL API database with 400 RU/s throughput
az cosmosdb sql database create \
  --account-name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --name "$COSMOS_DB" \
  --throughput 400

# Verify database creation
az cosmosdb sql database show \
  --account-name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --name "$COSMOS_DB" \
  --query "{Name:id, Throughput:options.throughput}" \
  --output table
```

---

## Step 6 – Create Container with Partition Key

```bash
# Create container with partition key
az cosmosdb sql container create \
  --account-name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --database-name "$COSMOS_DB" \
  --name "$COSMOS_CONTAINER" \
  --partition-key-path "$PARTITION_KEY"

# Verify container creation
az cosmosdb sql container show \
  --account-name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --database-name "$COSMOS_DB" \
  --name "$COSMOS_CONTAINER" \
  --query "{Name:id, PartitionKey:partitionKey.paths[0]}" \
  --output table
```

---

## Step 7 – Install Cosmos DB Python SDK (Optional)

```bash
# Install Python and Cosmos DB SDK for advanced operations
pip3 install azure-cosmos --quiet

# Verify installation
python3 -c "import azure.cosmos; print('Cosmos SDK Version:', azure.cosmos.__version__)"
```

---

## Step 8 – Insert Items Using Python

```bash
# Create Python script to insert items
cat > insert_items.py << 'EOFPY'
import os
import sys
from azure.cosmos import CosmosClient, PartitionKey, exceptions

# Get credentials from environment
endpoint = os.environ['COSMOS_ENDPOINT']
key = os.environ['COSMOS_KEY']
database_name = os.environ['COSMOS_DB']
container_name = os.environ['COSMOS_CONTAINER']

# Create client
client = CosmosClient(endpoint, key)
database = client.get_database_client(database_name)
container = database.get_container_client(container_name)

# Sample products data
products = [
    {
        "id": "1",
        "name": "Laptop Pro 15",
        "category": "Electronics",
        "price": 1299.99,
        "stock": 45,
        "description": "High-performance laptop with 15-inch display",
        "brand": "TechCorp"
    },
    {
        "id": "2",
        "name": "Wireless Mouse",
        "category": "Electronics",
        "price": 29.99,
        "stock": 200,
        "description": "Ergonomic wireless mouse",
        "brand": "ClickPro"
    },
    {
        "id": "3",
        "name": "Office Desk",
        "category": "Furniture",
        "price": 349.99,
        "stock": 15,
        "description": "Adjustable standing desk",
        "brand": "ErgoOffice"
    },
    {
        "id": "4",
        "name": "Gaming Keyboard",
        "category": "Electronics",
        "price": 89.99,
        "stock": 78,
        "description": "RGB mechanical keyboard",
        "brand": "GamePro"
    },
    {
        "id": "5",
        "name": "Office Chair",
        "category": "Furniture",
        "price": 249.99,
        "stock": 30,
        "description": "Ergonomic office chair with lumbar support",
        "brand": "ComfortSeating"
    }
]

# Insert items
for product in products:
    try:
        container.create_item(body=product)
        print(f"✅ Inserted: {product['name']}")
    except exceptions.CosmosResourceExistsError:
        print(f"⚠️  Item {product['id']} already exists")
    except Exception as e:
        print(f"❌ Error inserting {product['name']}: {str(e)}")

print(f"\n✅ Inserted {len(products)} products into Cosmos DB")
EOFPY

# Set environment variables and run script
export COSMOS_ENDPOINT="$COSMOS_ENDPOINT"
export COSMOS_KEY="$COSMOS_KEY"
export COSMOS_DB="$COSMOS_DB"
export COSMOS_CONTAINER="$COSMOS_CONTAINER"

python3 insert_items.py
```

---

## Step 9 – Query Items

```bash
# Create Python script to query items
cat > query_items.py << 'EOFPY'
import os
from azure.cosmos import CosmosClient

# Get credentials from environment
endpoint = os.environ['COSMOS_ENDPOINT']
key = os.environ['COSMOS_KEY']
database_name = os.environ['COSMOS_DB']
container_name = os.environ['COSMOS_CONTAINER']

# Create client
client = CosmosClient(endpoint, key)
database = client.get_database_client(database_name)
container = database.get_container_client(container_name)

print("=== All Products ===")
query = "SELECT * FROM c"
items = list(container.query_items(query=query, enable_cross_partition_query=True))
for item in items:
    print(f"ID: {item['id']}, Name: {item['name']}, Category: {item['category']}, Price: ${item['price']}")

print(f"\nTotal items: {len(items)}")

print("\n=== Electronics Only ===")
query = "SELECT * FROM c WHERE c.category = 'Electronics'"
items = list(container.query_items(query=query, enable_cross_partition_query=True))
for item in items:
    print(f"{item['name']} - ${item['price']}")

print("\n=== Products Under $100 ===")
query = "SELECT c.name, c.price FROM c WHERE c.price < 100 ORDER BY c.price"
items = list(container.query_items(query=query, enable_cross_partition_query=True))
for item in items:
    print(f"{item['name']}: ${item['price']}")

print("\n=== High Stock Items (>50) ===")
query = "SELECT c.name, c.stock FROM c WHERE c.stock > 50"
items = list(container.query_items(query=query, enable_cross_partition_query=True))
for item in items:
    print(f"{item['name']}: {item['stock']} units")
EOFPY

python3 query_items.py
```

---

## Step 10 – Update Items

```bash
# Create Python script to update items
cat > update_items.py << 'EOFPY'
import os
from azure.cosmos import CosmosClient

# Get credentials from environment
endpoint = os.environ['COSMOS_ENDPOINT']
key = os.environ['COSMOS_KEY']
database_name = os.environ['COSMOS_DB']
container_name = os.environ['COSMOS_CONTAINER']

# Create client
client = CosmosClient(endpoint, key)
database = client.get_database_client(database_name)
container = database.get_container_client(container_name)

# Read item
item_id = "1"
partition_key = "Electronics"
item = container.read_item(item=item_id, partition_key=partition_key)

print(f"Original: {item['name']} - Price: ${item['price']}, Stock: {item['stock']}")

# Update price and stock
item['price'] = 1199.99
item['stock'] = 50
item['onSale'] = True

# Replace item
container.replace_item(item=item_id, body=item)

print(f"Updated: {item['name']} - Price: ${item['price']}, Stock: {item['stock']}, On Sale: {item['onSale']}")
print("✅ Item updated successfully")
EOFPY

python3 update_items.py
```

---

## Step 11 – Delete Items

```bash
# Create Python script to delete items
cat > delete_items.py << 'EOFPY'
import os
from azure.cosmos import CosmosClient, exceptions

# Get credentials from environment
endpoint = os.environ['COSMOS_ENDPOINT']
key = os.environ['COSMOS_KEY']
database_name = os.environ['COSMOS_DB']
container_name = os.environ['COSMOS_CONTAINER']

# Create client
client = CosmosClient(endpoint, key)
database = client.get_database_client(database_name)
container = database.get_container_client(container_name)

# Delete item
item_id = "2"
partition_key = "Electronics"

try:
    container.delete_item(item=item_id, partition_key=partition_key)
    print(f"✅ Deleted item with ID: {item_id}")
except exceptions.CosmosResourceNotFoundError:
    print(f"❌ Item {item_id} not found")
except Exception as e:
    print(f"❌ Error: {str(e)}")

# Verify deletion
query = "SELECT * FROM c"
items = list(container.query_items(query=query, enable_cross_partition_query=True))
print(f"\nRemaining items: {len(items)}")
for item in items:
    print(f"  - {item['name']}")
EOFPY

python3 delete_items.py
```

---

## Step 12 – Check Throughput Settings

```bash
# Get database throughput
az cosmosdb sql database throughput show \
  --account-name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --name "$COSMOS_DB" \
  --query "resource.throughput" \
  --output tsv

# Get container details
az cosmosdb sql container show \
  --account-name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --database-name "$COSMOS_DB" \
  --name "$COSMOS_CONTAINER" \
  --query "{Container:id, PartitionKey:partitionKey.paths[0], IndexingMode:indexingPolicy.indexingMode}" \
  --output table
```

---

## Step 13 – Monitor Request Units (RUs)

```bash
# Create script to monitor RU consumption
cat > monitor_rus.py << 'EOFPY'
import os
from azure.cosmos import CosmosClient

# Get credentials from environment
endpoint = os.environ['COSMOS_ENDPOINT']
key = os.environ['COSMOS_KEY']
database_name = os.environ['COSMOS_DB']
container_name = os.environ['COSMOS_CONTAINER']

# Create client
client = CosmosClient(endpoint, key)
database = client.get_database_client(database_name)
container = database.get_container_client(container_name)

# Execute query and check RU charge
query = "SELECT * FROM c WHERE c.category = 'Electronics'"
query_iterable = container.query_items(query=query, enable_cross_partition_query=True)

# Get request charge from response headers
items = list(query_iterable)
request_charge = container.client_connection.last_response_headers['x-ms-request-charge']

print(f"Query returned {len(items)} items")
print(f"Request charge: {request_charge} RUs")
print(f"\nTip: Lower RU consumption = lower costs")
EOFPY

python3 monitor_rus.py
```

---

## Step 14 – List All Databases and Containers

```bash
# List all databases in account
az cosmosdb sql database list \
  --account-name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Throughput:options.throughput}" \
  --output table

# List all containers in database
az cosmosdb sql container list \
  --account-name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --database-name "$COSMOS_DB" \
  --query "[].{Name:name}" \
  --output table
```

---

## Step 15 – Get Cosmos DB Metrics

```bash
# Get Cosmos DB account resource ID
COSMOS_ID=$(az cosmosdb show \
  --name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

# Get total request units consumed
az monitor metrics list \
  --resource "$COSMOS_ID" \
  --metric "TotalRequestUnits" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT5M \
  --output table

# List available metrics
az monitor metrics list-definitions \
  --resource "$COSMOS_ID" \
  --query "[].{Metric:name.value, Unit:unit}" \
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

# Remove local Python scripts
rm -f insert_items.py query_items.py update_items.py delete_items.py monitor_rus.py

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Cosmos DB account with SQL API
- Database with provisioned throughput
- Container with partition key
- CRUD operations on JSON documents
- RU consumption monitoring

**Architecture:**
```
Client → Cosmos DB Account → Database (400 RU/s) → Container (Partition: /category)
                                                         ↓
                                                   JSON Documents
```

**Key Components:**
- **Cosmos DB Account**: Globally distributed database service
- **SQL API**: Query data using familiar SQL syntax
- **Partition Key**: Distributes data across partitions (/category)
- **Request Units**: Measure of throughput/performance
- **Consistency Levels**: Session consistency by default

**What You Learned:**
- Create Cosmos DB account and containers
- Perform CRUD operations with SQL API
- Query data using SQL-like syntax
- Monitor and optimize RU consumption
- Work with partition keys for scalability

---

## Best Practices

**Partition Key Selection:**
- Choose high-cardinality partition keys (many unique values)
- Distribute reads/writes evenly across partitions
- Avoid hot partitions (single partition getting most traffic)
- Consider query patterns when choosing partition key
- Use composite partition keys for complex scenarios

**Throughput Optimization:**
- Start with minimum RU/s and scale up based on metrics
- Use autoscale for unpredictable workloads
- Monitor RU consumption per operation
- Optimize queries to reduce RU costs
- Use TTL to automatically delete old data

**Data Modeling:**
- Denormalize data for read-heavy workloads
- Embed related data in single document when possible
- Use point reads (read by ID + partition key) for best performance
- Keep document sizes under 2MB
- Design for your query patterns

**Cost Management:**
- Use serverless tier for development/testing
- Scale down throughput during off-peak hours
- Enable TTL to reduce storage costs
- Monitor and optimize RU consumption
- Use free tier for learning (400 RU/s + 5GB)

---

## Production Enhancements

**1. Enable Multi-Region Replication**
```bash
# Add read region for global distribution
az cosmosdb update \
  --name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --locations regionName="australiaeast" failoverPriority=0 isZoneRedundant=False \
  --locations regionName="australiasoutheast" failoverPriority=1 isZoneRedundant=False
```

**2. Configure Automatic Failover**
```bash
# Enable automatic failover
az cosmosdb update \
  --name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --enable-automatic-failover true
```

**3. Implement Change Feed**
```bash
# Change feed is enabled by default - use for real-time processing
cat > change_feed.py << 'EOFPY'
from azure.cosmos import CosmosClient
import os

endpoint = os.environ['COSMOS_ENDPOINT']
key = os.environ['COSMOS_KEY']
client = CosmosClient(endpoint, key)

database = client.get_database_client(os.environ['COSMOS_DB'])
container = database.get_container_client(os.environ['COSMOS_CONTAINER'])

# Read change feed
for item in container.query_items_change_feed():
    print(f"Changed: {item}")
EOFPY
```

**4. Configure Time-to-Live (TTL)**
```bash
# Enable TTL on container (items expire automatically)
az cosmosdb sql container update \
  --account-name "$COSMOS_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --database-name "$COSMOS_DB" \
  --name "$COSMOS_CONTAINER" \
  --ttl 86400
```

---

## Troubleshooting

**High RU consumption:**
- Review query patterns and add indexes
- Avoid cross-partition queries when possible
- Use SELECT specific fields instead of SELECT *
- Implement pagination for large result sets
- Check for missing or inefficient indexes

**Partition key errors:**
- Ensure partition key is included in all queries
- Verify partition key value matches document
- Don't change partition key after container creation
- Use cross-partition queries sparingly
- Check partition key path is correct

**Connection failures:**
- Verify endpoint URL and access key
- Check firewall rules and network access
- Ensure Cosmos DB account is provisioned
- Review consistency level settings
- Check for regional outages

**Slow queries:**
- Add appropriate indexes for query filters
- Use partition key in WHERE clause
- Implement pagination with continuation tokens
- Monitor RU consumption per query
- Consider composite indexes for complex queries

**Python SDK errors:**
- Ensure azure-cosmos package is installed
- Check Python version compatibility (3.6+)
- Verify environment variables are set
- Review exception messages carefully
- Enable SDK logging for detailed errors

---

## Additional Resources

- [Cosmos DB Documentation](https://docs.microsoft.com/azure/cosmos-db/)
- [SQL API Guide](https://docs.microsoft.com/azure/cosmos-db/sql-query-getting-started)
- [Partition Key Selection](https://docs.microsoft.com/azure/cosmos-db/partitioning-overview)
- [Request Units](https://docs.microsoft.com/azure/cosmos-db/request-units)
- [Python SDK Reference](https://docs.microsoft.com/python/api/overview/azure/cosmos-readme)
