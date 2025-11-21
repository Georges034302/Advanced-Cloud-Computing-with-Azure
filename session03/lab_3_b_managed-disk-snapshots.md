# Lab 3.B: Managed Disk Snapshots

## Overview
This lab demonstrates creating and managing Azure Managed Disk snapshots for backup and disaster recovery. You'll create VMs with managed disks, take snapshots, restore disks from snapshots, and implement disk encryption.

---

## Objectives
- Create VM with managed disks (OS and data disks)
- Create disk snapshots for backup
- Restore disks from snapshots
- Attach additional data disks to VMs
- Copy snapshots across regions
- Implement disk encryption
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- SSH client available
- Basic Linux knowledge
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="eastus"
LOCATION_SECONDARY="westus"
RG_NAME="rg-lab3b-disks"
VM_NAME="vm-disk-test"
ADMIN_USER="azureuser"
DATA_DISK_NAME="data-disk-01"
SNAPSHOT_NAME="snapshot-os-disk"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "VM_NAME=$VM_NAME"
echo "DATA_DISK_NAME=$DATA_DISK_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for disk resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create VM with Managed Disks

```bash
# Generate SSH key
SSH_KEY_PATH="$HOME/.ssh/id_rsa_lab3b"
if [ ! -f "$SSH_KEY_PATH" ]; then
  ssh-keygen -t rsa -b 4096 -f "$SSH_KEY_PATH" -N "" -C "$ADMIN_USER@lab3b"
fi

# Create VM with 30GB OS disk
az vm create \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --location "$LOCATION" \
  --image "Ubuntu2204" \
  --size "Standard_B2s" \
  --admin-username "$ADMIN_USER" \
  --ssh-key-values "${SSH_KEY_PATH}.pub" \
  --os-disk-size-gb 30 \
  --os-disk-name "${VM_NAME}-osdisk" \
  --public-ip-sku Standard \
  --output table

# Get VM details
VM_ID=$(az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query id \
  --output tsv)

echo "VM ID: $VM_ID"
```

---

## Step 4 – Create and Attach Data Disk

```bash
# Create managed data disk (128 GB, Premium SSD)
az disk create \
  --resource-group "$RG_NAME" \
  --name "$DATA_DISK_NAME" \
  --size-gb 128 \
  --sku Premium_LRS \
  --location "$LOCATION"

# Attach data disk to VM
az vm disk attach \
  --resource-group "$RG_NAME" \
  --vm-name "$VM_NAME" \
  --name "$DATA_DISK_NAME"

# Verify disk attachment
az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query "storageProfile.dataDisks[].{Name:name, Lun:lun, Size:diskSizeGb, Caching:caching}" \
  --output table
```

---

## Step 5 – Format and Mount Data Disk on VM

```bash
# Get VM public IP
VM_PUBLIC_IP=$(az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "VM Public IP: $VM_PUBLIC_IP"

# SSH to VM and configure data disk
ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no "$ADMIN_USER@$VM_PUBLIC_IP" << 'EOFVM'
# List disks
echo "=== Available Disks ==="
lsblk

# Partition the disk (assuming /dev/sdc is the data disk)
echo "Partitioning /dev/sdc..."
sudo parted /dev/sdc --script mklabel gpt mkpart primary ext4 0% 100%

# Format partition
echo "Formatting partition..."
sudo mkfs.ext4 /dev/sdc1

# Create mount point
sudo mkdir -p /mnt/data

# Mount the disk
sudo mount /dev/sdc1 /mnt/data

# Add to fstab for persistent mount
UUID=$(sudo blkid -s UUID -o value /dev/sdc1)
echo "UUID=$UUID /mnt/data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab

# Verify mount
df -h /mnt/data

# Create test data
sudo mkdir -p /mnt/data/important
echo "Critical data - $(date)" | sudo tee /mnt/data/important/file1.txt
echo "More data - $(date)" | sudo tee /mnt/data/important/file2.txt
sudo ls -lh /mnt/data/important/
EOFVM
```

---

## Step 6 – Create OS Disk Snapshot

```bash
# Get OS disk ID
OS_DISK_ID=$(az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query "storageProfile.osDisk.managedDisk.id" \
  --output tsv)

echo "OS Disk ID: $OS_DISK_ID"

# Create OS disk snapshot
az snapshot create \
  --resource-group "$RG_NAME" \
  --name "$SNAPSHOT_NAME" \
  --source "$OS_DISK_ID" \
  --location "$LOCATION"

# Verify snapshot creation
az snapshot show \
  --resource-group "$RG_NAME" \
  --name "$SNAPSHOT_NAME" \
  --query "{Name:name, Size:diskSizeGb, State:provisioningState, CreatedTime:timeCreated}" \
  --output table
```

---

## Step 7 – Create Data Disk Snapshot

```bash
# Create data disk snapshot
DATA_SNAPSHOT_NAME="snapshot-data-disk"

az snapshot create \
  --resource-group "$RG_NAME" \
  --name "$DATA_SNAPSHOT_NAME" \
  --source "$DATA_DISK_NAME" \
  --location "$LOCATION"

# List all snapshots
az snapshot list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Size:diskSizeGb, SourceDisk:creationData.sourceResourceId}" \
  --output table
```

---

## Step 8 – Create Disk from Snapshot

```bash
# Stop and deallocate VM (optional, for safety)
az vm deallocate \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME"

# Create new disk from OS snapshot
RESTORED_DISK_NAME="restored-os-disk"

az disk create \
  --resource-group "$RG_NAME" \
  --name "$RESTORED_DISK_NAME" \
  --source "$SNAPSHOT_NAME" \
  --location "$LOCATION"

# Verify restored disk
az disk show \
  --resource-group "$RG_NAME" \
  --name "$RESTORED_DISK_NAME" \
  --query "{Name:name, Size:diskSizeGb, Sku:sku.name}" \
  --output table
```

---

## Step 9 – Swap OS Disk

```bash
# Get restored disk ID
RESTORED_DISK_ID=$(az disk show \
  --resource-group "$RG_NAME" \
  --name "$RESTORED_DISK_NAME" \
  --query id \
  --output tsv)

# Note: Swapping OS disk requires VM to be deallocated
# Update VM to use restored OS disk
az vm update \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --os-disk "$RESTORED_DISK_ID"

echo "✅ OS disk swapped - VM now uses restored disk"

# Start VM with restored disk
az vm start \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME"

echo "⏳ Waiting for VM to start..."
sleep 30
```

---

## Step 10 – Copy Snapshot to Another Region

```bash
# Get snapshot ID
SNAPSHOT_ID=$(az snapshot show \
  --resource-group "$RG_NAME" \
  --name "$DATA_SNAPSHOT_NAME" \
  --query id \
  --output tsv)

# Copy snapshot to secondary region
az snapshot create \
  --resource-group "$RG_NAME" \
  --name "${DATA_SNAPSHOT_NAME}-westus" \
  --location "$LOCATION_SECONDARY" \
  --source "$SNAPSHOT_ID" \
  --copy-start

# Monitor copy progress
az snapshot show \
  --resource-group "$RG_NAME" \
  --name "${DATA_SNAPSHOT_NAME}-westus" \
  --query "{Name:name, Location:location, State:provisioningState, CompletionPercent:completionPercent}" \
  --output table

echo "⏳ Snapshot copy initiated - may take several minutes"
```

---

## Step 11 – Create Incremental Snapshot

```bash
# Create incremental snapshot (more cost-effective)
INCREMENTAL_SNAPSHOT="snapshot-data-incremental"

az snapshot create \
  --resource-group "$RG_NAME" \
  --name "$INCREMENTAL_SNAPSHOT" \
  --source "$DATA_DISK_NAME" \
  --incremental true \
  --location "$LOCATION"

# Compare snapshot types
az snapshot list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Type:incremental, Size:diskSizeGb}" \
  --output table
```

---

## Step 12 – Resize Managed Disk

```bash
# Deallocate VM before resizing
az vm deallocate \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME"

# Resize data disk from 128GB to 256GB
az disk update \
  --resource-group "$RG_NAME" \
  --name "$DATA_DISK_NAME" \
  --size-gb 256

# Start VM
az vm start \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME"

# SSH to VM and expand filesystem
sleep 30
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << 'EOFVM'
# Resize partition
echo "Resizing partition..."
sudo parted /dev/sdc resizepart 1 100%

# Resize filesystem
sudo resize2fs /dev/sdc1

# Verify new size
df -h /mnt/data
EOFVM
```

---

## Step 13 – Enable Disk Encryption

```bash
# Create Key Vault for disk encryption keys
KEYVAULT_NAME="kv-diskenc-$RANDOM"

az keyvault create \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --enabled-for-disk-encryption true

# Enable encryption on VM
az vm encryption enable \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --disk-encryption-keyvault "$KEYVAULT_NAME" \
  --volume-type ALL

echo "⏳ Encryption in progress (may take 10-15 minutes)..."

# Check encryption status
az vm encryption show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query "{OSDisk:disks[0].statuses[0].displayStatus, DataDisk:disks[1].statuses[0].displayStatus}" \
  --output table
```

---

## Step 14 – List All Disks and Snapshots

```bash
# List all managed disks
echo "=== Managed Disks ==="
az disk list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Size:diskSizeGb, Sku:sku.name, State:diskState}" \
  --output table

# List all snapshots
echo -e "\n=== Snapshots ==="
az snapshot list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Size:diskSizeGb, Incremental:incremental, Location:location}" \
  --output table

# Get total disk usage
TOTAL_DISK_SIZE=$(az disk list \
  --resource-group "$RG_NAME" \
  --query "sum([].diskSizeGb)" \
  --output tsv)

echo -e "\nTotal Disk Size: ${TOTAL_DISK_SIZE}GB"
```

---

## Step 15 – Cleanup

```bash
# Disable disk encryption before deletion
az vm encryption disable \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --volume-type ALL 2>/dev/null || true

# Delete VM
az vm delete \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --yes

# Delete snapshots
az snapshot delete \
  --resource-group "$RG_NAME" \
  --name "$SNAPSHOT_NAME" \
  --yes

az snapshot delete \
  --resource-group "$RG_NAME" \
  --name "$DATA_SNAPSHOT_NAME" \
  --yes

az snapshot delete \
  --resource-group "$RG_NAME" \
  --name "$INCREMENTAL_SNAPSHOT" \
  --yes

az snapshot delete \
  --resource-group "$RG_NAME" \
  --name "${DATA_SNAPSHOT_NAME}-westus" \
  --yes 2>/dev/null || true

# Delete disks
az disk delete \
  --resource-group "$RG_NAME" \
  --name "$DATA_DISK_NAME" \
  --yes

az disk delete \
  --resource-group "$RG_NAME" \
  --name "$RESTORED_DISK_NAME" \
  --yes

# Delete resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove SSH key
# rm -f "$SSH_KEY_PATH" "${SSH_KEY_PATH}.pub"

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- VM with OS and data managed disks
- Full and incremental disk snapshots
- Disk restoration and recovery procedures
- Cross-region snapshot replication
- Disk encryption with Azure Key Vault

**Architecture:**
```
VM → OS Disk (Encrypted) → Snapshot → Restored Disk
  ↓
Data Disk (256GB) → Incremental Snapshot → Cross-Region Copy
```

**Key Components:**
- **Managed Disks**: Azure-managed storage for VMs
- **Snapshots**: Point-in-time disk backups
- **Incremental Snapshots**: Cost-effective delta backups
- **Disk Encryption**: Azure Disk Encryption with Key Vault
- **Disk SKUs**: Standard HDD, Standard SSD, Premium SSD, Ultra Disk

**What You Learned:**
- Create and attach managed disks
- Take full and incremental snapshots
- Restore disks from snapshots
- Swap OS disks for recovery
- Copy snapshots across regions
- Resize disks and expand filesystems
- Enable disk encryption

---

## Best Practices

**Disk Management:**
- Use Premium SSD for production workloads
- Implement regular snapshot schedules
- Test disk restoration procedures
- Monitor disk performance metrics
- Right-size disks to avoid waste

**Backup Strategy:**
- Use incremental snapshots for cost savings
- Implement retention policies for snapshots
- Store snapshots in multiple regions
- Automate snapshot creation
- Document recovery procedures

**Performance:**
- Use Premium SSD for IOPS-intensive workloads
- Enable read/write caching appropriately
- Monitor disk IOPS and throughput
- Consider Ultra Disks for extreme performance
- Stripe multiple disks for higher throughput

**Security:**
- Enable disk encryption for sensitive data
- Use customer-managed keys when required
- Implement RBAC for disk access
- Enable soft delete for accidental deletion protection
- Audit disk operations regularly

---

## Production Enhancements

**1. Automate Snapshot Creation**
```bash
# Create snapshot schedule using Azure Automation (example script)
cat > create-snapshot.sh << 'EOF'
#!/bin/bash
DISK_ID="/subscriptions/.../disks/your-disk"
SNAPSHOT_NAME="auto-snapshot-$(date +%Y%m%d-%H%M%S)"

az snapshot create \
  --resource-group "$RG_NAME" \
  --name "$SNAPSHOT_NAME" \
  --source "$DISK_ID" \
  --incremental true
EOF
```

**2. Implement Snapshot Retention**
```bash
# Delete snapshots older than 30 days
CUTOFF_DATE=$(date -u -d '30 days ago' '+%Y-%m-%dT%H:%M:%SZ')

az snapshot list \
  --resource-group "$RG_NAME" \
  --query "[?timeCreated<'$CUTOFF_DATE'].name" \
  --output tsv | while read SNAP; do
  echo "Deleting old snapshot: $SNAP"
  az snapshot delete --resource-group "$RG_NAME" --name "$SNAP" --yes
done
```

**3. Configure Disk Metrics and Alerts**
```bash
# Create alert for high disk IOPS
az monitor metrics alert create \
  --name "high-disk-iops" \
  --resource-group "$RG_NAME" \
  --scopes "$OS_DISK_ID" \
  --condition "avg Percentage CPU > 80" \
  --description "Alert when disk IOPS exceed threshold"
```

**4. Use Azure Backup**
```bash
# Create Recovery Services Vault
az backup vault create \
  --resource-group "$RG_NAME" \
  --name "vault-backup" \
  --location "$LOCATION"

# Enable backup for VM
az backup protection enable-for-vm \
  --resource-group "$RG_NAME" \
  --vault-name "vault-backup" \
  --vm "$VM_NAME" \
  --policy-name DefaultPolicy
```

---

## Troubleshooting

**Snapshot creation fails:**
- Verify sufficient permissions (Contributor role)
- Check disk is in available state
- Ensure quota limits aren't exceeded
- Confirm region supports snapshots
- Review error message for specific issues

**Cannot resize disk:**
- VM must be deallocated first
- Cannot shrink disk, only expand
- Check SKU supports target size
- Verify filesystem supports expansion
- Ensure no active connections to disk

**Disk encryption fails:**
- Verify Key Vault exists and is accessible
- Check VM size supports encryption
- Ensure Key Vault has proper permissions
- Wait for encryption to complete (10-15 min)
- Review encryption extension logs

**Snapshot copy slow:**
- Large snapshots take time to copy
- Network bandwidth affects copy speed
- Use `--copy-start` for async copy
- Monitor completion percentage
- Consider incremental snapshots

**Filesystem not expanding:**
- Partition must be resized first (parted/growpart)
- Use appropriate filesystem command (resize2fs, xfs_growfs)
- Verify partition table type (GPT recommended)
- Unmount filesystem before resizing (if required)
- Check for filesystem errors first

---

## Additional Resources

- [Managed Disks Overview](https://docs.microsoft.com/azure/virtual-machines/managed-disks-overview)
- [Disk Snapshots](https://docs.microsoft.com/azure/virtual-machines/snapshot-copy-managed-disk)
- [Incremental Snapshots](https://docs.microsoft.com/azure/virtual-machines/disks-incremental-snapshots)
- [Azure Disk Encryption](https://docs.microsoft.com/azure/virtual-machines/linux/disk-encryption-overview)
- [Disk Performance Tiers](https://docs.microsoft.com/azure/virtual-machines/disks-performance-tiers)
