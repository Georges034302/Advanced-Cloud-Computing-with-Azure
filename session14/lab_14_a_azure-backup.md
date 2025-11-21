# Lab 14.A: Azure Backup

## Objectives
- Create Recovery Services vault
- Configure backup for virtual machines
- Perform on-demand backup
- Restore VM from backup
- Configure backup policies
- Monitor backup jobs
- Test backup retention
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
RG_NAME="rg-lab14a-backup"
VAULT_NAME="vault-backup-$RANDOM"
VNET_NAME="vnet-backup"
VM_NAME="vm-backup-test"
POLICY_NAME="DailyBackupPolicy"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$VAULT_NAME"
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

## Step 3 – Create Recovery Services Vault

```bash
# Create Recovery Services vault
az backup vault create \
  --name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION"

echo "Recovery Services vault created"
```

---

## Step 4 – Configure Vault Storage Redundancy

```bash
# Set storage redundancy to locally redundant
az backup vault backup-properties set \
  --name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --backup-storage-redundancy LocallyRedundant

echo "Storage redundancy configured"
```

---

## Step 5 – Create Virtual Network

```bash
# Create VNet for test VM
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.70.0.0/16 \
  --subnet-name "subnet-vms" \
  --subnet-prefix 10.70.1.0/24

echo "Virtual network created"
```

---

## Step 6 – Get Admin Password

```bash
# Read admin password securely
read -s -p "Enter VM admin password: " ADMIN_PASSWORD
echo
```

---

## Step 7 – Create Virtual Machine

```bash
# Create VM for backup testing
az vm create \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --vnet-name "$VNET_NAME" \
  --subnet "subnet-vms" \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username "azureuser" \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-address "${VM_NAME}-pip" \
  --nsg "${VM_NAME}-nsg"

echo "Virtual machine created"
```

---

## Step 8 – Get VM Resource ID

```bash
# Get VM resource ID
VM_ID=$(az vm show \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$VM_ID"
```

---

## Step 9 – Enable Backup for VM

```bash
# Enable backup protection for VM with default policy
az backup protection enable-for-vm \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --vm "$VM_ID" \
  --policy-name DefaultPolicy

echo "Backup protection enabled"
```

---

## Step 10 – List Backup Policies

```bash
# List available backup policies
az backup policy list \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Type:backupManagementType, RetentionDays:properties.instantRpRetentionRangeInDays}" \
  --output table
```

---

## Step 11 – Show Default Policy Details

```bash
# Display default policy configuration
az backup policy show \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --name DefaultPolicy \
  --query "{Name:name, Type:backupManagementType, ScheduleTime:properties.schedulePolicy.schedulePolicyType}" \
  --output table
```

---

## Step 12 – Create Custom Backup Policy

```bash
# Get default policy to use as template
az backup policy show \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --name DefaultPolicy \
  --output json > policy-template.json

# Create custom policy (simplified version)
cat > custom-policy.json << 'EOF'
{
  "name": "CustomDailyPolicy",
  "properties": {
    "backupManagementType": "AzureIaasVM",
    "instantRpRetentionRangeInDays": 2,
    "schedulePolicy": {
      "schedulePolicyType": "SimpleSchedulePolicy",
      "scheduleRunFrequency": "Daily",
      "scheduleRunTimes": [
        "2025-01-01T02:00:00Z"
      ]
    },
    "retentionPolicy": {
      "retentionPolicyType": "LongTermRetentionPolicy",
      "dailySchedule": {
        "retentionTimes": [
          "2025-01-01T02:00:00Z"
        ],
        "retentionDuration": {
          "count": 30,
          "durationType": "Days"
        }
      }
    },
    "timeZone": "UTC"
  }
}
EOF

echo "Custom policy definition created"
```

---

## Step 13 – List Protected Items

```bash
# List all protected VMs
az backup item list \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --query "[].{Name:properties.friendlyName, ProtectionState:properties.protectionState, LastBackupTime:properties.lastBackupTime}" \
  --output table
```

---

## Step 14 – Create Test File on VM

```bash
# Get VM public IP
VM_PUBLIC_IP=$(az vm show \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "$VM_PUBLIC_IP"

# Configure NSG to allow SSH
az network nsg rule create \
  --name AllowSSH \
  --nsg-name "${VM_NAME}-nsg" \
  --resource-group "$RG_NAME" \
  --priority 1000 \
  --source-address-prefixes '*' \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow

# Create test file
ssh -o StrictHostKeyChecking=no "azureuser@${VM_PUBLIC_IP}" \
  'echo "Backup test data - created before backup" > ~/test-backup.txt && cat ~/test-backup.txt'

echo "Test file created on VM"
```

---

## Step 15 – Trigger On-Demand Backup

```bash
# Get container name
CONTAINER_NAME=$(az backup container list \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --backup-management-type AzureIaasVM \
  --query "[0].name" \
  --output tsv)

echo "$CONTAINER_NAME"

# Get item name
ITEM_NAME=$(az backup item list \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --query "[0].name" \
  --output tsv)

echo "$ITEM_NAME"

# Trigger backup
az backup protection backup-now \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --container-name "$CONTAINER_NAME" \
  --item-name "$ITEM_NAME" \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --retain-until "$(date -u -d '+30 days' +%d-%m-%Y)"

echo "On-demand backup triggered"
```

---

## Step 16 – Monitor Backup Job

```bash
# List backup jobs
az backup job list \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].{JobName:name, Operation:properties.operation, Status:properties.status, StartTime:properties.startTime}" \
  --output table

echo "Backup job status displayed"
```

---

## Step 17 – Wait for Backup Completion

```bash
# Get latest job name
JOB_NAME=$(az backup job list \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --query "[0].name" \
  --output tsv)

echo "$JOB_NAME"

# Check job status (simplified - may need to wait)
echo "Waiting for backup to complete (this may take several minutes)..."
sleep 60

az backup job show \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --name "$JOB_NAME" \
  --query "{Status:properties.status, PercentComplete:properties.percentComplete}" \
  --output table
```

---

## Step 18 – List Recovery Points

```bash
# List available recovery points
az backup recoverypoint list \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --container-name "$CONTAINER_NAME" \
  --item-name "$ITEM_NAME" \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --query "[].{Name:name, Time:properties.recoveryPointTime, Type:properties.recoveryPointType}" \
  --output table 2>/dev/null || echo "Recovery points being created"
```

---

## Step 19 – Modify VM Data

```bash
# Modify file on VM to test restore
ssh -o StrictHostKeyChecking=no "azureuser@${VM_PUBLIC_IP}" \
  'echo "Modified data - should be restored from backup" > ~/test-backup.txt && cat ~/test-backup.txt'

echo "VM data modified"
```

---

## Step 20 – Show Restore Options

```bash
# Display restore options
cat << 'EOF'

Azure Backup Restore Options:
==============================

1. Create New VM:
   - Restore to new virtual machine
   - Specify new name and location

2. Replace Existing:
   - Replace disks on existing VM
   - Requires VM to be stopped

3. Restore Disks:
   - Restore only managed disks
   - Attach to any VM manually

Use Azure Portal for full restore workflow:
Portal > Recovery Services Vault > Backup Items > Restore VM

EOF
```

---

## Step 21 – Show Backup Reports

```bash
# Get backup summary
az backup item list \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --query "[].{VM:properties.friendlyName, Policy:properties.policyName, Status:properties.protectionStatus, Health:properties.healthStatus}" \
  --output table
```

---

## Step 22 – Configure Backup Alerts

```bash
# Show alert configuration information
cat << 'EOF'

Backup Alerts Configuration:
============================

Built-in Alerts (automatically enabled):
  - Backup failures
  - Restore failures
  - Protection stopped

Configure in Portal:
  Recovery Services Vault > Backup Alerts
  - Set email notifications
  - Configure severity levels
  - Integrate with Action Groups

Monitoring Options:
  - Azure Monitor integration
  - Log Analytics workspace
  - Azure Backup Reports workbook

EOF
```

---

## Step 23 – Show Backup Costs

```bash
# Display storage usage
az backup vault backup-properties show \
  --name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --query "{StorageType:storageType, StorageTypeState:storageTypeState}" \
  --output table
```

---

## Step 24 – List All Backup Containers

```bash
# Show all backup containers
az backup container list \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --backup-management-type AzureIaasVM \
  --query "[].{Name:name, Type:properties.backupManagementType, Status:properties.registrationStatus}" \
  --output table
```

---

## Step 25 – Show Backup Features

```bash
# Display Azure Backup capabilities
cat << 'EOF'

Azure Backup Features:
======================

Supported Workloads:
  - Azure VMs (Windows/Linux)
  - SQL Server in Azure VMs
  - SAP HANA in Azure VMs
  - Azure Files
  - Azure Blobs
  - Azure Disks

Key Features:
  - Application-consistent backups
  - Incremental backups
  - Instant restore (snapshot tier)
  - Long-term retention
  - Cross-region restore
  - Soft delete protection
  - Encryption at rest

Retention Options:
  - Daily: Up to 9999 days
  - Weekly: Up to 5163 weeks
  - Monthly: Up to 1188 months
  - Yearly: Up to 99 years

EOF
```

---

## Step 26 – Disable Backup Protection

```bash
# Stop backup protection (retain data)
az backup protection disable \
  --vault-name "$VAULT_NAME" \
  --resource-group "$RG_NAME" \
  --container-name "$CONTAINER_NAME" \
  --item-name "$ITEM_NAME" \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --yes

echo "Backup protection disabled"
```

---

## Step 27 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f policy-template.json custom-policy.json

echo "Cleanup complete"
```

---

## Summary

You created a Recovery Services vault with locally redundant storage, deployed a virtual machine and enabled backup protection with the default policy, triggered an on-demand backup and monitored the job status, created test data on the VM to simulate backup scenarios, listed recovery points available for restore operations, explored backup policies including retention settings, reviewed backup monitoring and alerting options, and learned about Azure Backup features including application-consistent backups, instant restore, and cross-region restore capabilities.
