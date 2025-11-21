# Lab 3.A: Blob Storage Lifecycle Management

## Overview
This lab demonstrates Azure Blob Storage lifecycle management policies to automatically transition blobs between access tiers and delete old data. You'll create storage accounts, upload blobs, configure lifecycle policies, and verify automatic tier transitions.

---

## Objectives
- Create Azure Storage Account with different access tiers
- Upload blobs to hot, cool, and archive tiers
- Configure lifecycle management policies
- Implement automatic tier transitions
- Set blob deletion rules based on age
- Monitor and verify lifecycle policy execution
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Basic understanding of blob storage tiers
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="eastus"
RG_NAME="rg-lab3a-lifecycle"
STORAGE_ACCOUNT="lab3alifecycle$RANDOM"
CONTAINER_NAME="data-archive"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "STORAGE_ACCOUNT=$STORAGE_ACCOUNT"
echo "CONTAINER_NAME=$CONTAINER_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for storage resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Storage Account

```bash
# Create storage account with hot access tier (default)
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot \
  --allow-blob-public-access false

# Verify storage account creation
az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Tier:accessTier, Kind:kind, Sku:sku.name}" \
  --output table
```

---

## Step 4 – Create Blob Container

```bash
# Create container for blob storage
az storage container create \
  --name "$CONTAINER_NAME" \
  --account-name "$STORAGE_ACCOUNT" \
  --auth-mode login

# Verify container creation
az storage container list \
  --account-name "$STORAGE_ACCOUNT" \
  --auth-mode login \
  --query "[].{Name:name, PublicAccess:properties.publicAccess}" \
  --output table
```

---

## Step 5 – Upload Test Blobs

```bash
# Create test files with different timestamps
for i in {1..5}; do
  echo "Test data - File $i created on $(date)" > "testfile-$i.txt"
done

# Upload files to hot tier (default)
for i in {1..5}; do
  az storage blob upload \
    --account-name "$STORAGE_ACCOUNT" \
    --container-name "$CONTAINER_NAME" \
    --name "hot/testfile-$i.txt" \
    --file "testfile-$i.txt" \
    --auth-mode login
done

# Upload files with cool tier
for i in {1..3}; do
  az storage blob upload \
    --account-name "$STORAGE_ACCOUNT" \
    --container-name "$CONTAINER_NAME" \
    --name "cool/testfile-$i.txt" \
    --file "testfile-$i.txt" \
    --tier Cool \
    --auth-mode login
done

# List uploaded blobs
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --auth-mode login \
  --query "[].{Name:name, Tier:properties.blobTier, Size:properties.contentLength}" \
  --output table
```

---

## Step 6 – Set Blob Metadata for Lifecycle Tracking

```bash
# Set last modified time metadata on specific blobs
az storage blob metadata update \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-1.txt" \
  --metadata category=logs environment=production \
  --auth-mode login

# View blob properties including metadata
az storage blob show \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-1.txt" \
  --auth-mode login \
  --query "{Name:name, Tier:properties.blobTier, Metadata:metadata, LastModified:properties.lastModified}" \
  --output json
```

---

## Step 7 – Create Lifecycle Management Policy

```bash
# Create lifecycle policy JSON
cat > lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "enabled": true,
      "name": "move-to-cool-after-30-days",
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
          },
          "snapshot": {
            "delete": {
              "daysAfterCreationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["data-archive/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "delete-old-logs",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 7
            },
            "delete": {
              "daysAfterModificationGreaterThan": 30
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "archive-hot-tier-blobs",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 60
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["hot/"]
        }
      }
    }
  ]
}
EOF

# Apply lifecycle management policy
az storage account management-policy create \
  --account-name "$STORAGE_ACCOUNT" \
  --policy @lifecycle-policy.json \
  --resource-group "$RG_NAME"

echo "✅ Lifecycle management policy created"
```

---

## Step 8 – Verify Lifecycle Policy

```bash
# Get current lifecycle management policy
az storage account management-policy show \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "policy.rules[].{Name:name, Enabled:enabled}" \
  --output table

# View full policy details
az storage account management-policy show \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --output json | python3 -m json.tool
```

---

## Step 9 – Manually Change Blob Tier

```bash
# Change blob from hot to cool tier manually
az storage blob set-tier \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-2.txt" \
  --tier Cool \
  --auth-mode login

# Change blob to archive tier
az storage blob set-tier \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-3.txt" \
  --tier Archive \
  --auth-mode login

# Verify tier changes
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --prefix "hot/" \
  --auth-mode login \
  --query "[].{Name:name, Tier:properties.blobTier, ArchiveStatus:properties.archiveStatus}" \
  --output table
```

---

## Step 10 – Rehydrate Archived Blob

```bash
# Rehydrate blob from archive to hot tier (high priority)
az storage blob set-tier \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-3.txt" \
  --tier Hot \
  --rehydrate-priority High \
  --auth-mode login

# Check rehydration status
az storage blob show \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-3.txt" \
  --auth-mode login \
  --query "{Name:name, Tier:properties.blobTier, ArchiveStatus:properties.archiveStatus, RehydratePriority:properties.rehydratePriority}" \
  --output table

echo "⏳ Rehydration can take up to 15 hours (Standard) or 1 hour (High priority)"
```

---

## Step 11 – Create Blob Snapshots

```bash
# Create snapshot of a blob
SNAPSHOT_TIME=$(az storage blob snapshot \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-1.txt" \
  --auth-mode login \
  --query snapshot \
  --output tsv)

echo "Snapshot created: $SNAPSHOT_TIME"

# List blobs including snapshots
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --include s \
  --auth-mode login \
  --query "[?snapshot!=null].{Name:name, Snapshot:snapshot}" \
  --output table
```

---

## Step 12 – Enable Versioning and Soft Delete

```bash
# Enable blob versioning
az storage account blob-service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --enable-versioning true

# Enable soft delete for blobs (7 days retention)
az storage account blob-service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --enable-delete-retention true \
  --delete-retention-days 7

# Enable soft delete for containers (7 days)
az storage account blob-service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --enable-container-delete-retention true \
  --container-delete-retention-days 7

# Verify settings
az storage account blob-service-properties show \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "{Versioning:isVersioningEnabled, BlobSoftDelete:deleteRetentionPolicy.enabled, ContainerSoftDelete:containerDeleteRetentionPolicy.enabled}" \
  --output table
```

---

## Step 13 – Test Soft Delete

```bash
# Delete a blob
az storage blob delete \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-4.txt" \
  --auth-mode login

# List deleted blobs (soft-deleted)
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --include d \
  --auth-mode login \
  --query "[?deleted==true].{Name:name, DeletedTime:properties.deletedTime}" \
  --output table

# Undelete the blob
az storage blob undelete \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-4.txt" \
  --auth-mode login

echo "✅ Blob restored from soft delete"
```

---

## Step 14 – Monitor Storage Metrics

```bash
# Get storage account metrics
az monitor metrics list \
  --resource "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT" \
  --metric "UsedCapacity" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT1H \
  --query "value[].{Time:timeseries[0].data[0].timeStamp, UsedGB:timeseries[0].data[0].total}" \
  --output table

# Get blob count by tier
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --auth-mode login \
  --query "[].properties.blobTier" \
  --output tsv | sort | uniq -c
```

---

## Step 15 – Update Lifecycle Policy

```bash
# Create updated policy with different rules
cat > lifecycle-policy-updated.json << 'EOF'
{
  "rules": [
    {
      "enabled": true,
      "name": "aggressive-archival",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 15
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 45
            },
            "delete": {
              "daysAfterModificationGreaterThan": 180
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"]
        }
      }
    }
  ]
}
EOF

# Update lifecycle policy
az storage account management-policy update \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --policy @lifecycle-policy-updated.json

echo "✅ Lifecycle policy updated"
```

---

## Step 16 – Cleanup

```bash
# Delete lifecycle management policy
az storage account management-policy delete \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME"

# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local test files
rm -f testfile-*.txt lifecycle-policy*.json

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Storage account with lifecycle management
- Blob containers with hot, cool, and archive tiers
- Automated tier transition policies
- Blob versioning and soft delete
- Snapshot and deletion management

**Architecture:**
```
Blob Upload → Hot Tier → (30 days) → Cool Tier → (90 days) → Archive Tier → (365 days) → Delete
                                                                ↓
                                                         Rehydration (1-15 hrs)
```

**Key Components:**
- **Access Tiers**: Hot (frequent access), Cool (infrequent), Archive (rare)
- **Lifecycle Policies**: Automated tier transitions and deletions
- **Blob Versioning**: Automatic version history
- **Soft Delete**: Recovery window for deleted blobs
- **Snapshots**: Point-in-time blob copies

**What You Learned:**
- Configure blob storage access tiers
- Create lifecycle management policies
- Implement automatic tier transitions
- Use blob versioning and soft delete
- Rehydrate archived blobs
- Monitor storage metrics and costs

---

## Best Practices

**Lifecycle Management:**
- Analyze access patterns before setting policies
- Start with conservative transition periods
- Use prefixes to apply different policies to different data
- Test policies in non-production first
- Monitor policy execution and adjust as needed

**Cost Optimization:**
- Move infrequently accessed data to cool tier
- Archive data that must be retained but rarely accessed
- Set deletion policies for temporary data
- Use blob inventory to analyze tier distribution
- Monitor early deletion fees (cool/archive)

**Data Protection:**
- Enable soft delete for accidental deletion recovery
- Use blob versioning for critical data
- Create snapshots before major changes
- Implement backup strategies for important data
- Test restore procedures regularly

**Performance:**
- Keep hot data in hot tier for best performance
- Plan for rehydration time when accessing archive
- Use high-priority rehydration for urgent needs
- Consider cool tier for data accessed monthly
- Monitor access patterns and adjust tiers

---

## Production Enhancements

**1. Implement Last Access Time Tracking**
```bash
# Enable last access time tracking
az storage account blob-service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --enable-last-access-tracking true

# Create policy based on last access time
cat > lifecycle-last-access.json << 'EOF'
{
  "rules": [{
    "enabled": true,
    "name": "move-based-on-access",
    "type": "Lifecycle",
    "definition": {
      "actions": {
        "baseBlob": {
          "tierToCool": {
            "daysAfterLastAccessTimeGreaterThan": 30
          }
        }
      },
      "filters": {"blobTypes": ["blockBlob"]}
    }
  }]
}
EOF
```

**2. Configure Blob Index Tags for Filtering**
```bash
# Set blob index tags
az storage blob tag set \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --name "hot/testfile-1.txt" \
  --tags "Project=Alpha" "Department=Engineering" \
  --auth-mode login

# Query blobs by tags
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --filter-expression "Department='Engineering'" \
  --auth-mode login
```

**3. Enable Change Feed**
```bash
# Enable change feed for audit trail
az storage account blob-service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --enable-change-feed true \
  --change-feed-retention-days 30
```

**4. Configure Immutable Storage**
```bash
# Enable versioning-level immutability
az storage account blob-service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --enable-versioning true

# Set immutability policy on container
az storage container immutability-policy create \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER_NAME" \
  --period 30 \
  --auth-mode login
```

---

## Troubleshooting

**Lifecycle policy not executing:**
- Policies run once per day (not immediate)
- Check policy is enabled
- Verify blob matches filter criteria (prefix, blob type)
- Wait 24-48 hours for first execution
- Review storage account activity logs

**Cannot change blob tier:**
- Verify blob is not in archive tier (must rehydrate first)
- Check storage account supports tier changes
- Ensure account is StorageV2 or BlobStorage
- Confirm blob type is block blob
- Review RBAC permissions

**Rehydration taking too long:**
- Standard priority can take up to 15 hours
- Use high priority for 1-hour rehydration
- Check rehydration status with `archiveStatus` property
- Consider copying to new blob instead
- Plan ahead for archive access needs

**Soft delete not working:**
- Verify soft delete is enabled
- Check retention period hasn't expired
- Use `--include d` flag to list deleted blobs
- Ensure blob wasn't permanently deleted
- Review blob service properties settings

**High storage costs:**
- Review tier distribution of blobs
- Check for early deletion fees (cool/archive)
- Analyze lifecycle policy effectiveness
- Use blob inventory for cost analysis
- Consider aggressive archival for old data

---

## Additional Resources

- [Blob Storage Tiers](https://docs.microsoft.com/azure/storage/blobs/access-tiers-overview)
- [Lifecycle Management Policies](https://docs.microsoft.com/azure/storage/blobs/lifecycle-management-overview)
- [Blob Versioning](https://docs.microsoft.com/azure/storage/blobs/versioning-overview)
- [Soft Delete for Blobs](https://docs.microsoft.com/azure/storage/blobs/soft-delete-blob-overview)
- [Rehydrate Blobs from Archive](https://docs.microsoft.com/azure/storage/blobs/archive-rehydrate-overview)
