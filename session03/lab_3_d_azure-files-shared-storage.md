# Lab 3.D: Azure Files Shared Storage

## Overview
This lab demonstrates Azure Files for creating cloud-based file shares accessible via SMB and NFS protocols. You'll create file shares, mount them on Linux and Windows VMs, configure snapshots, and implement access controls.

---

## Objectives
- Create Azure Storage Account with Azure Files
- Create SMB file shares
- Mount file shares on Linux VM
- Configure file share snapshots
- Implement Azure File Sync (overview)
- Set access tiers and quotas
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- SSH client available
- Basic Linux and Windows knowledge
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="eastus"
RG_NAME="rg-lab3d-files"
STORAGE_ACCOUNT="lab3dfiles$RANDOM"
FILE_SHARE_NAME="shared-data"
VM_NAME="vm-files-test"
ADMIN_USER="azureuser"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "STORAGE_ACCOUNT=$STORAGE_ACCOUNT"
echo "FILE_SHARE_NAME=$FILE_SHARE_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for file storage resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Storage Account for Azure Files

```bash
# Create storage account (supports SMB and NFS)
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --enable-large-file-share

# Verify storage account creation
az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Kind:kind, LargeFileShares:largeFileSharesState}" \
  --output table
```

---

## Step 4 – Create Azure File Share

```bash
# Create file share with 100GB quota
az storage share create \
  --name "$FILE_SHARE_NAME" \
  --account-name "$STORAGE_ACCOUNT" \
  --quota 100 \
  --auth-mode login

# Create additional file share for different tier
az storage share create \
  --name "archive-data" \
  --account-name "$STORAGE_ACCOUNT" \
  --quota 50 \
  --auth-mode login

# List file shares
az storage share list \
  --account-name "$STORAGE_ACCOUNT" \
  --auth-mode login \
  --query "[].{Name:name, Quota:properties.quota, AccessTier:properties.accessTier}" \
  --output table
```

---

## Step 5 – Upload Files to File Share

```bash
# Create test files locally
mkdir -p test-files
for i in {1..5}; do
  echo "Test file $i - $(date)" > "test-files/document-$i.txt"
done

# Upload files to file share
for i in {1..5}; do
  az storage file upload \
    --account-name "$STORAGE_ACCOUNT" \
    --share-name "$FILE_SHARE_NAME" \
    --source "test-files/document-$i.txt" \
    --path "documents/document-$i.txt" \
    --auth-mode login
done

# List files in share
az storage file list \
  --account-name "$STORAGE_ACCOUNT" \
  --share-name "$FILE_SHARE_NAME" \
  --path "documents" \
  --auth-mode login \
  --query "[].{Name:name, Size:properties.contentLength}" \
  --output table
```

---

## Step 6 – Create Directory Structure

```bash
# Create directories in file share
az storage directory create \
  --account-name "$STORAGE_ACCOUNT" \
  --share-name "$FILE_SHARE_NAME" \
  --name "projects" \
  --auth-mode login

az storage directory create \
  --account-name "$STORAGE_ACCOUNT" \
  --share-name "$FILE_SHARE_NAME" \
  --name "backups" \
  --auth-mode login

az storage directory create \
  --account-name "$STORAGE_ACCOUNT" \
  --share-name "$FILE_SHARE_NAME" \
  --name "projects/team-a" \
  --auth-mode login

# List directories
az storage directory list \
  --account-name "$STORAGE_ACCOUNT" \
  --share-name "$FILE_SHARE_NAME" \
  --auth-mode login \
  --query "[].name" \
  --output table
```

---

## Step 7 – Get Storage Account Keys

```bash
# Get storage account key for mounting
STORAGE_KEY=$(az storage account keys list \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "[0].value" \
  --output tsv)

echo "Storage Account Key (first 20 chars): ${STORAGE_KEY:0:20}..."
echo "⚠️  Keep this key secure - needed for mounting file share"
```

---

## Step 8 – Create VM for Testing File Share Mount

```bash
# Generate SSH key
SSH_KEY_PATH="$HOME/.ssh/id_rsa_lab3d"
if [ ! -f "$SSH_KEY_PATH" ]; then
  ssh-keygen -t rsa -b 4096 -f "$SSH_KEY_PATH" -N "" -C "$ADMIN_USER@lab3d"
fi

# Create VM
az vm create \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --location "$LOCATION" \
  --image "Ubuntu2204" \
  --size "Standard_B2s" \
  --admin-username "$ADMIN_USER" \
  --ssh-key-values "${SSH_KEY_PATH}.pub" \
  --public-ip-sku Standard \
  --output table

# Get VM public IP
VM_PUBLIC_IP=$(az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "VM Public IP: $VM_PUBLIC_IP"
```

---

## Step 9 – Mount File Share on Linux VM (SMB)

```bash
# SSH to VM and mount Azure Files
ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no "$ADMIN_USER@$VM_PUBLIC_IP" << EOFVM
# Install CIFS utilities
sudo apt-get update -y
sudo apt-get install -y cifs-utils

# Create mount point
sudo mkdir -p /mnt/azure-files

# Create credentials file
sudo bash -c 'cat > /etc/smbcredentials.txt << EOF
username=$STORAGE_ACCOUNT
password=$STORAGE_KEY
EOF'

# Secure credentials file
sudo chmod 600 /etc/smbcredentials.txt

# Mount the file share
sudo mount -t cifs \
  //${STORAGE_ACCOUNT}.file.core.windows.net/${FILE_SHARE_NAME} \
  /mnt/azure-files \
  -o credentials=/etc/smbcredentials.txt,dir_mode=0777,file_mode=0777,serverino,nosharesock,actimeo=30

# Verify mount
df -h /mnt/azure-files

# List files
ls -lh /mnt/azure-files/documents/

# Create test file on mounted share
echo "Test from VM - \$(date)" | sudo tee /mnt/azure-files/vm-test.txt

# Add to fstab for persistent mount
echo "//${STORAGE_ACCOUNT}.file.core.windows.net/${FILE_SHARE_NAME} /mnt/azure-files cifs credentials=/etc/smbcredentials.txt,dir_mode=0777,file_mode=0777,serverino,nosharesock,actimeo=30 0 0" | sudo tee -a /etc/fstab

echo "✅ File share mounted successfully"
EOFVM
```

---

## Step 10 – Test File Share Operations

```bash
# Perform file operations on mounted share
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << 'EOFVM'
# Create directory
sudo mkdir -p /mnt/azure-files/vm-data

# Create multiple files
for i in {1..3}; do
  echo "VM file $i - $(date)" | sudo tee /mnt/azure-files/vm-data/file-$i.txt
done

# List all files
echo "=== Files in Azure File Share ==="
sudo find /mnt/azure-files -type f -ls

# Check disk usage
echo -e "\n=== Disk Usage ==="
sudo du -sh /mnt/azure-files/*
EOFVM
```

---

## Step 11 – Create File Share Snapshot

```bash
# Create snapshot of file share
SNAPSHOT_NAME=$(az storage share snapshot \
  --account-name "$STORAGE_ACCOUNT" \
  --name "$FILE_SHARE_NAME" \
  --auth-mode login \
  --query "snapshot" \
  --output tsv)

echo "Snapshot created: $SNAPSHOT_NAME"

# List all snapshots
az storage share list \
  --account-name "$STORAGE_ACCOUNT" \
  --include-snapshots \
  --auth-mode login \
  --query "[?snapshot!=null].{Share:name, Snapshot:snapshot}" \
  --output table
```

---

## Step 12 – Modify Files and Restore from Snapshot

```bash
# Modify file on VM
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << 'EOFVM'
# Delete a file
sudo rm /mnt/azure-files/documents/document-3.txt

# Modify another file
echo "MODIFIED - $(date)" | sudo tee /mnt/azure-files/documents/document-1.txt

echo "Files modified/deleted"
EOFVM

# List files in snapshot (from Azure CLI)
echo "=== Files in Snapshot ==="
az storage file list \
  --account-name "$STORAGE_ACCOUNT" \
  --share-name "$FILE_SHARE_NAME" \
  --snapshot "$SNAPSHOT_NAME" \
  --path "documents" \
  --auth-mode login \
  --query "[].name" \
  --output table

# Restore deleted file from snapshot
az storage file download \
  --account-name "$STORAGE_ACCOUNT" \
  --share-name "$FILE_SHARE_NAME" \
  --snapshot "$SNAPSHOT_NAME" \
  --path "documents/document-3.txt" \
  --dest "restored-document-3.txt" \
  --auth-mode login

# Upload restored file back to share
az storage file upload \
  --account-name "$STORAGE_ACCOUNT" \
  --share-name "$FILE_SHARE_NAME" \
  --source "restored-document-3.txt" \
  --path "documents/document-3.txt" \
  --auth-mode login

echo "✅ File restored from snapshot"
```

---

## Step 13 – Configure File Share Access Tier

```bash
# Change file share to cool tier (lower cost, higher latency)
az storage share update \
  --account-name "$STORAGE_ACCOUNT" \
  --name "archive-data" \
  --access-tier Cool \
  --auth-mode login

# Verify tier change
az storage share show \
  --account-name "$STORAGE_ACCOUNT" \
  --name "archive-data" \
  --auth-mode login \
  --query "{Name:name, Tier:accessTier, Quota:quota}" \
  --output table
```

---

## Step 14 – Set File Share Quota and Metadata

```bash
# Update file share quota
az storage share update \
  --account-name "$STORAGE_ACCOUNT" \
  --name "$FILE_SHARE_NAME" \
  --quota 200 \
  --auth-mode login

# Set metadata on file share
az storage share metadata update \
  --account-name "$STORAGE_ACCOUNT" \
  --name "$FILE_SHARE_NAME" \
  --metadata department=engineering project=lab3d \
  --auth-mode login

# View file share properties
az storage share show \
  --account-name "$STORAGE_ACCOUNT" \
  --name "$FILE_SHARE_NAME" \
  --auth-mode login \
  --query "{Name:name, Quota:quota, Metadata:metadata, Tier:accessTier}" \
  --output json
```

---

## Step 15 – Monitor File Share Metrics

```bash
# Get file share capacity
az storage share stats \
  --account-name "$STORAGE_ACCOUNT" \
  --name "$FILE_SHARE_NAME" \
  --auth-mode login

# Get storage account metrics
az monitor metrics list \
  --resource "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT/fileServices/default" \
  --metric "FileCapacity" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT1H \
  --output table
```

---

## Step 16 – Download Files from File Share

```bash
# Download file from share
az storage file download \
  --account-name "$STORAGE_ACCOUNT" \
  --share-name "$FILE_SHARE_NAME" \
  --path "documents/document-1.txt" \
  --dest "downloaded-document.txt" \
  --auth-mode login

# View downloaded file
cat downloaded-document.txt

# Download entire directory
az storage file download-batch \
  --account-name "$STORAGE_ACCOUNT" \
  --source "$FILE_SHARE_NAME/documents" \
  --destination "downloaded-documents" \
  --auth-mode login

ls -lh downloaded-documents/
```

---

## Step 17 – Unmount File Share

```bash
# SSH to VM and unmount
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << 'EOFVM'
# Unmount file share
sudo umount /mnt/azure-files

# Remove from fstab
sudo sed -i "/${STORAGE_ACCOUNT}.file.core.windows.net/d" /etc/fstab

# Verify unmounted
df -h | grep azure-files || echo "File share unmounted successfully"
EOFVM
```

---

## Step 18 – Cleanup

```bash
# Delete snapshots
az storage share delete \
  --account-name "$STORAGE_ACCOUNT" \
  --name "$FILE_SHARE_NAME" \
  --snapshot "$SNAPSHOT_NAME" \
  --auth-mode login

# Delete file shares
az storage share delete \
  --account-name "$STORAGE_ACCOUNT" \
  --name "$FILE_SHARE_NAME" \
  --auth-mode login

az storage share delete \
  --account-name "$STORAGE_ACCOUNT" \
  --name "archive-data" \
  --auth-mode login

# Delete resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf test-files downloaded-documents downloaded-document.txt restored-document-3.txt
# rm -f "$SSH_KEY_PATH" "${SSH_KEY_PATH}.pub"

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Azure Files file shares with SMB protocol
- Linux VM with mounted file share
- File share snapshots for point-in-time recovery
- Directory structure and file operations
- Access tier configuration for cost optimization

**Architecture:**
```
Linux VM → SMB Mount → Azure Files Share → Storage Account
                              ↓
                          Snapshots (Backup)
```

**Key Components:**
- **Azure Files**: Managed file shares in the cloud
- **SMB Protocol**: Industry-standard file sharing protocol
- **File Share Snapshots**: Point-in-time backups
- **Access Tiers**: Hot, Cool for cost optimization
- **CIFS**: Common Internet File System (SMB client)

**What You Learned:**
- Create and configure Azure Files shares
- Mount file shares on Linux using SMB/CIFS
- Manage directories and files in Azure Files
- Create and restore from snapshots
- Configure quotas and access tiers
- Monitor file share usage and metrics

---

## Best Practices

**File Share Configuration:**
- Set appropriate quotas to prevent overuse
- Use access tiers based on access patterns
- Implement snapshots for backup/recovery
- Organize files with clear directory structure
- Monitor capacity and performance metrics

**Security:**
- Store storage account keys securely
- Use Azure AD authentication when possible
- Implement network security with private endpoints
- Enable encryption at rest (default)
- Restrict access with firewall rules

**Performance:**
- Use Premium file shares for high IOPS workloads
- Enable large file shares for capacity >5TB
- Monitor IOPS and throttling
- Use appropriate VM SKUs for file share workloads
- Consider Azure File Sync for hybrid scenarios

**Cost Optimization:**
- Use cool tier for infrequently accessed data
- Set appropriate quotas to avoid waste
- Delete old snapshots regularly
- Monitor and optimize file share usage
- Use lifecycle management policies

---

## Production Enhancements

**1. Enable Azure AD Authentication**
```bash
# Join storage account to Azure AD DS
az storage account update \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --enable-files-aadds true
```

**2. Configure Private Endpoint**
```bash
# Create private endpoint for file share
az network private-endpoint create \
  --resource-group "$RG_NAME" \
  --name "pe-files" \
  --vnet-name "your-vnet" \
  --subnet "your-subnet" \
  --private-connection-resource-id "/subscriptions/.../storageAccounts/$STORAGE_ACCOUNT" \
  --group-id file \
  --connection-name "file-connection"
```

**3. Enable Soft Delete**
```bash
# Enable soft delete for file shares (7 days)
az storage account file-service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --enable-delete-retention true \
  --delete-retention-days 7
```

**4. Use Premium File Shares**
```bash
# Create FileStorage account for premium shares
az storage account create \
  --name "premiumfiles$RANDOM" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Premium_LRS \
  --kind FileStorage
```

---

## Troubleshooting

**Cannot mount file share:**
- Verify storage account name and key are correct
- Check port 445 is not blocked by firewall
- Ensure CIFS utilities are installed
- Verify network connectivity to storage endpoint
- Check mount command syntax

**Permission denied errors:**
- Verify credentials file has correct username/password
- Check file permissions (dir_mode, file_mode)
- Ensure user has write permissions
- Verify storage account key is valid
- Check Azure RBAC assignments

**Slow performance:**
- Check IOPS limits for Standard vs Premium
- Monitor network bandwidth
- Verify VM and storage are in same region
- Consider Premium file shares for high IOPS
- Review file share metrics for throttling

**Snapshot restore issues:**
- Verify snapshot exists and hasn't been deleted
- Check snapshot timestamp is correct
- Ensure sufficient quota for restored files
- Verify permissions to access snapshot
- Use Azure Portal for easier snapshot browsing

**Quota exceeded errors:**
- Check current file share usage
- Increase quota if needed
- Delete unnecessary files
- Move data to cool tier
- Review and optimize storage usage

---

## Additional Resources

- [Azure Files Documentation](https://docs.microsoft.com/azure/storage/files/)
- [Mount Azure Files on Linux](https://docs.microsoft.com/azure/storage/files/storage-how-to-use-files-linux)
- [Azure Files Snapshots](https://docs.microsoft.com/azure/storage/files/storage-snapshots-files)
- [Premium File Shares](https://docs.microsoft.com/azure/storage/files/storage-files-planning#storage-tiers)
- [Azure File Sync](https://docs.microsoft.com/azure/storage/file-sync/file-sync-introduction)
