# Lab 14.C: Geo-Redundant Storage

## Objectives
- Create storage accounts with different redundancy options
- Upload data to GRS storage
- Test data replication
- Configure RA-GRS for read access
- Perform failover scenarios
- Monitor replication status
- Compare storage redundancy types
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Location: Australia East (primary)

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab14c-storage"
LRS_STORAGE="lrsstorage$RANDOM"
GRS_STORAGE="grsstorage$RANDOM"
RAGRS_STORAGE="ragrsstorage$RANDOM"
GZRS_STORAGE="gzrsstorage$RANDOM"
CONTAINER_NAME="data"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$GRS_STORAGE"
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

## Step 3 – Create LRS Storage Account

```bash
# Create Locally Redundant Storage account
az storage account create \
  --name "$LRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2

echo "LRS storage account created"
```

---

## Step 4 – Create GRS Storage Account

```bash
# Create Geo-Redundant Storage account
az storage account create \
  --name "$GRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_GRS \
  --kind StorageV2

echo "GRS storage account created"
```

---

## Step 5 – Create RA-GRS Storage Account

```bash
# Create Read-Access Geo-Redundant Storage account
az storage account create \
  --name "$RAGRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_RAGRS \
  --kind StorageV2

echo "RA-GRS storage account created"
```

---

## Step 6 – Create GZRS Storage Account

```bash
# Create Geo-Zone-Redundant Storage account
az storage account create \
  --name "$GZRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_GZRS \
  --kind StorageV2

echo "GZRS storage account created"
```

---

## Step 7 – Show Storage Redundancy Types

```bash
# Display storage redundancy options
cat << 'EOF'

Azure Storage Redundancy Options:
==================================

LRS (Locally Redundant Storage):
  - 3 copies in single datacenter
  - Protects: Hardware failures
  - Durability: 99.999999999% (11 nines)
  - Lowest cost

ZRS (Zone-Redundant Storage):
  - 3 copies across availability zones
  - Protects: Datacenter failures
  - Durability: 99.9999999999% (12 nines)
  - Same region only

GRS (Geo-Redundant Storage):
  - 6 copies (3 local + 3 in paired region)
  - Protects: Regional disasters
  - Durability: 99.99999999999999% (16 nines)
  - No read access to secondary

RA-GRS (Read-Access GRS):
  - Same as GRS + read access to secondary
  - Enables read from paired region
  - Higher availability for reads

GZRS (Geo-Zone-Redundant Storage):
  - Combines ZRS + GRS
  - Maximum durability and availability
  - Highest cost

RA-GZRS (Read-Access GZRS):
  - GZRS + read access to secondary
  - Ultimate protection

EOF
```

---

## Step 8 – Get GRS Storage Key

```bash
# Get primary key for GRS storage
GRS_KEY=$(az storage account keys list \
  --account-name "$GRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --query "[0].value" \
  --output tsv)

echo "Storage key retrieved"
```

---

## Step 9 – Create Container in GRS Storage

```bash
# Create blob container
az storage container create \
  --name "$CONTAINER_NAME" \
  --account-name "$GRS_STORAGE" \
  --account-key "$GRS_KEY"

echo "Container created"
```

---

## Step 10 – Create Test Files

```bash
# Create test data files
mkdir -p test-data

echo "Critical business data - File 1" > test-data/file1.txt
echo "Important customer information - File 2" > test-data/file2.txt
echo "Financial records - File 3" > test-data/file3.txt

# Create larger file
dd if=/dev/urandom of=test-data/largefile.bin bs=1M count=10 2>/dev/null

echo "Test files created"
```

---

## Step 11 – Upload Files to GRS Storage

```bash
# Upload files to blob storage
az storage blob upload-batch \
  --account-name "$GRS_STORAGE" \
  --account-key "$GRS_KEY" \
  --destination "$CONTAINER_NAME" \
  --source test-data \
  --pattern "*"

echo "Files uploaded to GRS storage"
```

---

## Step 12 – List Uploaded Blobs

```bash
# List all blobs in container
az storage blob list \
  --account-name "$GRS_STORAGE" \
  --account-key "$GRS_KEY" \
  --container-name "$CONTAINER_NAME" \
  --query "[].{Name:name, Size:properties.contentLength, LastModified:properties.lastModified}" \
  --output table
```

---

## Step 13 – Get RA-GRS Storage Key

```bash
# Get primary key for RA-GRS storage
RAGRS_KEY=$(az storage account keys list \
  --account-name "$RAGRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --query "[0].value" \
  --output tsv)

echo "RA-GRS key retrieved"
```

---

## Step 14 – Create Container in RA-GRS Storage

```bash
# Create container in RA-GRS account
az storage container create \
  --name "$CONTAINER_NAME" \
  --account-name "$RAGRS_STORAGE" \
  --account-key "$RAGRS_KEY"

echo "RA-GRS container created"
```

---

## Step 15 – Upload to RA-GRS Storage

```bash
# Upload critical data to RA-GRS
az storage blob upload-batch \
  --account-name "$RAGRS_STORAGE" \
  --account-key "$RAGRS_KEY" \
  --destination "$CONTAINER_NAME" \
  --source test-data \
  --pattern "*"

echo "Files uploaded to RA-GRS storage"
```

---

## Step 16 – Get Primary Endpoint

```bash
# Get primary blob endpoint
PRIMARY_ENDPOINT=$(az storage account show \
  --name "$RAGRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --query primaryEndpoints.blob \
  --output tsv)

echo "$PRIMARY_ENDPOINT"
```

---

## Step 17 – Get Secondary Endpoint

```bash
# Get secondary blob endpoint for RA-GRS
SECONDARY_ENDPOINT=$(az storage account show \
  --name "$RAGRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --query secondaryEndpoints.blob \
  --output tsv)

echo "$SECONDARY_ENDPOINT"
```

---

## Step 18 – Test Read from Primary

```bash
# Read blob from primary endpoint
az storage blob download \
  --account-name "$RAGRS_STORAGE" \
  --account-key "$RAGRS_KEY" \
  --container-name "$CONTAINER_NAME" \
  --name "file1.txt" \
  --file downloaded-primary.txt

cat downloaded-primary.txt
echo "Read from primary successful"
```

---

## Step 19 – Show Secondary Region

```bash
# Display secondary region for each storage account
az storage account show \
  --name "$GRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, PrimaryLocation:primaryLocation, SecondaryLocation:secondaryLocation}" \
  --output table

az storage account show \
  --name "$RAGRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, PrimaryLocation:primaryLocation, SecondaryLocation:secondaryLocation}" \
  --output table
```

---

## Step 20 – Check Replication Status

```bash
# Get GRS replication status
az storage account show \
  --name "$GRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, SKU:sku.name, StatusOfPrimary:statusOfPrimary, StatusOfSecondary:statusOfSecondary}" \
  --output table
```

---

## Step 21 – Show Last Sync Time

```bash
# Get last sync time for geo-replication
az storage account show \
  --name "$RAGRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, GeoReplicationStats:geoReplicationStats}" \
  --output json
```

---

## Step 22 – Compare Storage Account Properties

```bash
# Compare all storage accounts
az storage account list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, SKU:sku.name, Location:primaryLocation, Secondary:secondaryLocation}" \
  --output table
```

---

## Step 23 – Show Failover Capabilities

```bash
# Display failover information
cat << EOF

Storage Account Failover:
=========================

Customer-Managed Failover:
  - Available for GRS and RA-GRS accounts
  - Switch primary to secondary region
  - Initiated manually during outages
  - Account becomes LRS after failover

Failover Process:
  1. Verify primary region is unavailable
  2. Initiate account failover:
     az storage account failover --name $GRS_STORAGE
  3. Wait for failover completion (may take hours)
  4. Update application endpoints
  5. After primary region recovers, failback available

Considerations:
  - RPO (Recovery Point Objective): ~15 minutes
  - RTO (Recovery Time Objective): ~1 hour
  - Data loss possible (last sync point)
  - Account becomes LRS in new primary region
  - Re-enable GRS after failover if needed

EOF
```

---

## Step 24 – Create Lifecycle Policy

```bash
# Create lifecycle management policy
cat > lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "enabled": true,
      "name": "MoveToArchive",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90
            },
            "delete": {
              "daysAfterModificationGreaterThan": 365
            }
          }
        },
        "filters": {
          "blobTypes": [
            "blockBlob"
          ]
        }
      }
    }
  ]
}
EOF

# Apply lifecycle policy to GRS storage
az storage account management-policy create \
  --account-name "$GRS_STORAGE" \
  --policy @lifecycle-policy.json \
  --resource-group "$RG_NAME"

echo "Lifecycle policy created"
```

---

## Step 25 – Enable Soft Delete

```bash
# Enable soft delete for blobs
az storage blob service-properties delete-policy update \
  --account-name "$RAGRS_STORAGE" \
  --account-key "$RAGRS_KEY" \
  --enable true \
  --days-retained 7

echo "Soft delete enabled"
```

---

## Step 26 – Enable Versioning

```bash
# Enable blob versioning
az storage account blob-service-properties update \
  --account-name "$RAGRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --enable-versioning true

echo "Blob versioning enabled"
```

---

## Step 27 – Show Cost Comparison

```bash
# Display cost comparison
cat << 'EOF'

Storage Redundancy Cost Comparison:
====================================

Relative Costs (LRS = 1x):
  - LRS:    1.0x (baseline)
  - ZRS:    1.25x
  - GRS:    2.0x
  - RA-GRS: 2.5x
  - GZRS:   2.5x
  - RA-GZRS: 3.1x

Factors Affecting Cost:
  - Storage capacity (per GB)
  - Transactions (reads/writes)
  - Data transfer (egress)
  - Redundancy level

Best Practices:
  - Use LRS for non-critical data
  - Use ZRS for high availability in region
  - Use GRS for disaster recovery
  - Use RA-GRS for read scalability
  - Use lifecycle policies to optimize costs
  - Archive old data to cool/archive tiers

EOF
```

---

## Step 28 – Monitor Storage Metrics

```bash
# Get storage account metrics
az monitor metrics list \
  --resource "$GRS_STORAGE" \
  --resource-group "$RG_NAME" \
  --resource-type "Microsoft.Storage/storageAccounts" \
  --metric-names "UsedCapacity" \
  --output table 2>/dev/null || echo "Metrics available in Azure Monitor"
```

---

## Step 29 – Show Paired Regions

```bash
# Display Azure paired regions
cat << 'EOF'

Azure Paired Regions:
=====================

Australia Regions:
  - Australia East ↔ Australia Southeast
  - Australia Central ↔ Australia Central 2

Other Major Pairs:
  - East US ↔ West US
  - North Europe ↔ West Europe
  - Southeast Asia ↔ East Asia
  - UK South ↔ UK West

Benefits of Paired Regions:
  - Geographic separation (>300 miles)
  - Platform updates sequenced
  - Data residency for compliance
  - Prioritized recovery in outages
  - GRS replication uses pairs

EOF
```

---

## Step 30 – Download from RA-GRS Secondary

```bash
# Construct secondary endpoint URL
SECONDARY_URL="${SECONDARY_ENDPOINT}${CONTAINER_NAME}/file1.txt"

echo "$SECONDARY_URL"

# Note: Reading from secondary requires special handling
cat << EOF

Reading from Secondary Endpoint:
=================================

To read from RA-GRS secondary endpoint:
  - Use secondary endpoint URL
  - May have replication lag (~15 min)
  - Good for read scalability
  - Geographic distribution of reads

Secondary endpoint: $SECONDARY_ENDPOINT

Application code should:
  1. Try primary endpoint first
  2. Fallback to secondary on primary failure
  3. Handle eventual consistency
  4. Monitor last sync time

EOF
```

---

## Step 31 – Show Best Practices

```bash
# Display best practices
cat << 'EOF'

Storage Redundancy Best Practices:
===================================

Data Classification:
  - Critical data: RA-GRS or RA-GZRS
  - Important data: GRS or GZRS
  - Temporary data: LRS or ZRS

Application Design:
  - Implement retry logic
  - Handle secondary endpoint reads
  - Monitor replication lag
  - Test failover procedures

Compliance:
  - Consider data residency requirements
  - Use paired regions in same geography
  - Enable soft delete and versioning
  - Implement lifecycle policies

Disaster Recovery:
  - Document failover procedures
  - Test failover regularly
  - Monitor geo-replication status
  - Plan for RPO/RTO requirements

EOF
```

---

## Step 32 – Cleanup

```bash
# Delete resource group and all storage accounts
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf test-data
rm -f downloaded-primary.txt lifecycle-policy.json

echo "Cleanup complete"
```

---

## Summary

You created storage accounts with different redundancy options including LRS, GRS, RA-GRS, and GZRS, uploaded test data to geo-redundant storage accounts, explored primary and secondary endpoints for read access, reviewed geo-replication status and last sync times, learned about customer-managed failover for disaster recovery, configured lifecycle management policies for cost optimization, enabled blob versioning and soft delete for data protection, compared storage redundancy costs and use cases, and understood Azure paired regions for geographic distribution and compliance requirements.
