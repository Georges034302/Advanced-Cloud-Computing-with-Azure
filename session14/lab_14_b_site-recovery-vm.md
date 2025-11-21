# Lab 14.B: Azure Site Recovery

## Objectives
- Create Recovery Services vault for disaster recovery
- Configure source and target regions
- Deploy virtual machine in source region
- Enable replication with Site Recovery
- Configure replication settings
- Monitor replication health
- Perform test failover
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Locations: Australia East (source) and Australia Southeast (target)

---

## Step 1 – Set Variables

```bash
# Set Azure regions and resource naming
SOURCE_LOCATION="australiaeast"
TARGET_LOCATION="australiasoutheast"
SOURCE_RG="rg-lab14b-source"
TARGET_RG="rg-lab14b-target"
VAULT_NAME="vault-asr-$RANDOM"
SOURCE_VNET="vnet-source"
TARGET_VNET="vnet-target"
VM_NAME="vm-production"
CACHE_STORAGE="asrcache$RANDOM"

# Display configuration
echo "$SOURCE_LOCATION"
echo "$TARGET_LOCATION"
echo "$SOURCE_RG"
echo "$VAULT_NAME"
```

---

## Step 2 – Create Source Resource Group

```bash
# Create source resource group
az group create \
  --name "$SOURCE_RG" \
  --location "$SOURCE_LOCATION"
```

---

## Step 3 – Create Target Resource Group

```bash
# Create target resource group for failover
az group create \
  --name "$TARGET_RG" \
  --location "$TARGET_LOCATION"
```

---

## Step 4 – Create Recovery Services Vault

```bash
# Create Recovery Services vault in target region
az backup vault create \
  --name "$VAULT_NAME" \
  --resource-group "$TARGET_RG" \
  --location "$TARGET_LOCATION"

echo "Recovery Services vault created"
```

---

## Step 5 – Create Source Virtual Network

```bash
# Create VNet in source region
az network vnet create \
  --name "$SOURCE_VNET" \
  --resource-group "$SOURCE_RG" \
  --location "$SOURCE_LOCATION" \
  --address-prefix 10.80.0.0/16 \
  --subnet-name "subnet-app" \
  --subnet-prefix 10.80.1.0/24

echo "Source virtual network created"
```

---

## Step 6 – Create Target Virtual Network

```bash
# Create VNet in target region
az network vnet create \
  --name "$TARGET_VNET" \
  --resource-group "$TARGET_RG" \
  --location "$TARGET_LOCATION" \
  --address-prefix 10.90.0.0/16 \
  --subnet-name "subnet-app" \
  --subnet-prefix 10.90.1.0/24

echo "Target virtual network created"
```

---

## Step 7 – Get Admin Password

```bash
# Read admin password securely
read -s -p "Enter VM admin password: " ADMIN_PASSWORD
echo
```

---

## Step 8 – Create Source Virtual Machine

```bash
# Create VM in source region
az vm create \
  --name "$VM_NAME" \
  --resource-group "$SOURCE_RG" \
  --location "$SOURCE_LOCATION" \
  --vnet-name "$SOURCE_VNET" \
  --subnet "subnet-app" \
  --image Ubuntu2204 \
  --size Standard_DS1_v2 \
  --admin-username "azureuser" \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-address "${VM_NAME}-pip" \
  --nsg "${VM_NAME}-nsg"

echo "Source VM created"
```

---

## Step 9 – Create Cache Storage Account

```bash
# Create cache storage account in source region
az storage account create \
  --name "$CACHE_STORAGE" \
  --resource-group "$SOURCE_RG" \
  --location "$SOURCE_LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2

echo "Cache storage account created"
```

---

## Step 10 – Get VM Resource ID

```bash
# Get VM resource ID
VM_ID=$(az vm show \
  --name "$VM_NAME" \
  --resource-group "$SOURCE_RG" \
  --query id \
  --output tsv)

echo "$VM_ID"
```

---

## Step 11 – Get Source VNet ID

```bash
# Get source VNet ID
SOURCE_VNET_ID=$(az network vnet show \
  --name "$SOURCE_VNET" \
  --resource-group "$SOURCE_RG" \
  --query id \
  --output tsv)

echo "$SOURCE_VNET_ID"
```

---

## Step 12 – Get Target VNet ID

```bash
# Get target VNet ID
TARGET_VNET_ID=$(az network vnet show \
  --name "$TARGET_VNET" \
  --resource-group "$TARGET_RG" \
  --query id \
  --output tsv)

echo "$TARGET_VNET_ID"
```

---

## Step 13 – Show Site Recovery Concepts

```bash
# Display Site Recovery architecture
cat << 'EOF'

Azure Site Recovery Concepts:
==============================

Replication Components:
  - Source Region: Primary location (Australia East)
  - Target Region: Secondary location (Australia Southeast)
  - Recovery Services Vault: Manages replication
  - Cache Storage: Temporary storage for replication data
  - Replication Policy: RPO, retention settings

Failover Types:
  - Test Failover: Non-disruptive testing
  - Planned Failover: Scheduled migration
  - Unplanned Failover: Disaster recovery

Recovery Objectives:
  - RPO (Recovery Point Objective): Data loss tolerance
  - RTO (Recovery Time Objective): Downtime tolerance

EOF
```

---

## Step 14 – Install VM Extension for Monitoring

```bash
# Install Azure Monitor agent
az vm extension set \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --resource-group "$SOURCE_RG" \
  --vm-name "$VM_NAME" \
  --enable-auto-upgrade true

echo "Monitoring agent installed"
```

---

## Step 15 – Create Application on Source VM

```bash
# Get VM public IP
VM_PUBLIC_IP=$(az vm show \
  --name "$VM_NAME" \
  --resource-group "$SOURCE_RG" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "$VM_PUBLIC_IP"

# Configure NSG
az network nsg rule create \
  --name AllowSSH \
  --nsg-name "${VM_NAME}-nsg" \
  --resource-group "$SOURCE_RG" \
  --priority 1000 \
  --source-address-prefixes '*' \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow

# Install web server
ssh -o StrictHostKeyChecking=no "azureuser@${VM_PUBLIC_IP}" << 'EOF'
sudo apt-get update -y
sudo apt-get install -y nginx
echo "<h1>Production VM in Australia East</h1>" | sudo tee /var/www/html/index.html
sudo systemctl start nginx
sudo systemctl enable nginx
EOF

echo "Application installed on source VM"
```

---

## Step 16 – Allow HTTP Traffic

```bash
# Allow HTTP on source VM
az network nsg rule create \
  --name AllowHTTP \
  --nsg-name "${VM_NAME}-nsg" \
  --resource-group "$SOURCE_RG" \
  --priority 1010 \
  --source-address-prefixes '*' \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow

echo "HTTP traffic allowed"
```

---

## Step 17 – Test Source Application

```bash
# Test web application
curl -s "http://${VM_PUBLIC_IP}"

echo "Application tested successfully"
```

---

## Step 18 – Show Replication Setup Process

```bash
# Display manual replication steps
cat << EOF

Site Recovery Replication Setup:
=================================

Portal Configuration Required:
  1. Navigate to Recovery Services Vault: $VAULT_NAME
  2. Select "Site Recovery" > "Replicate Application"
  3. Configure Source:
     - Region: $SOURCE_LOCATION
     - Resource Group: $SOURCE_RG
     - Virtual Network: $SOURCE_VNET
  4. Configure Target:
     - Region: $TARGET_LOCATION
     - Resource Group: $TARGET_RG
     - Virtual Network: $TARGET_VNET
  5. Select VM: $VM_NAME
  6. Configure Replication Settings:
     - Cache storage: $CACHE_STORAGE
     - Replication policy: 24-hour-retention-policy
  7. Enable Replication

Note: Full replication setup requires Azure Portal
CLI support for Site Recovery is limited

EOF
```

---

## Step 19 – Create Recovery Plan Template

```bash
# Create recovery plan configuration
cat > recovery-plan.json << EOF
{
  "name": "ProductionRecoveryPlan",
  "properties": {
    "primaryFabric": "$SOURCE_LOCATION",
    "recoveryFabric": "$TARGET_LOCATION",
    "failoverDeploymentModel": "ResourceManager",
    "groups": [
      {
        "groupType": "Boot",
        "replicationProtectedItems": [
          {
            "id": "$VM_ID"
          }
        ]
      }
    ]
  }
}
EOF

echo "Recovery plan template created"
```

---

## Step 20 – Show Replication Monitoring

```bash
# Display monitoring information
cat << 'EOF'

Replication Monitoring:
=======================

Key Metrics to Monitor:
  - Replication Health: Healthy/Warning/Critical
  - RPO Status: Current recovery point objective
  - Data Change Rate: GB/day being replicated
  - Missing Recovery Points: Any gaps in replication

Monitoring Tools:
  - Azure Portal: Recovery Services Vault > Replicated Items
  - Azure Monitor: Custom alerts on replication health
  - Site Recovery Logs: Detailed replication events

Alert Configuration:
  - Replication health critical
  - Test failover overdue
  - High data change rate
  - Replication lag exceeds threshold

EOF
```

---

## Step 21 – Show Test Failover Process

```bash
# Display test failover steps
cat << EOF

Test Failover Procedure:
========================

Pre-requisites:
  - Replication status: Protected
  - Latest recovery point available
  - Target network configured: $TARGET_VNET

Steps:
  1. Portal > Recovery Services Vault > Replicated Items
  2. Select VM: $VM_NAME
  3. Click "Test Failover"
  4. Choose recovery point (Latest or specific)
  5. Select target virtual network: $TARGET_VNET
  6. Click "OK" to start test failover

Validation:
  - Verify VM boots in target region
  - Test application functionality
  - Validate network connectivity
  - Check data consistency

Cleanup:
  - Click "Cleanup test failover"
  - Document test results
  - Delete test resources

EOF
```

---

## Step 22 – Show Failover Types

```bash
# Display failover options
cat << 'EOF'

Failover Options:
=================

1. Test Failover:
   - Non-disruptive
   - Creates isolated test environment
   - Source VM continues running
   - Used for DR drills

2. Planned Failover:
   - Zero data loss
   - Source VM shutdown cleanly
   - Final replication sync
   - Used for planned migrations

3. Unplanned Failover:
   - Used during disasters
   - Source region unavailable
   - May have data loss (last sync point)
   - Choose recovery point

4. Failback:
   - Return to source region
   - After disaster resolved
   - Re-protect in original direction
   - Can be planned or unplanned

EOF
```

---

## Step 23 – Show Replication Policy Settings

```bash
# Display policy configuration
cat << 'EOF'

Replication Policy Settings:
============================

Default Policy:
  - App-consistent snapshot: Every 4 hours
  - Crash-consistent snapshot: Every 5 minutes
  - Recovery point retention: 24 hours
  - Multi-VM consistency: Optional

Custom Policy Options:
  - RPO threshold: 15/30/60/120 minutes
  - Snapshot frequency: 1-12 hours
  - Retention: 1-15 days
  - Compression: Enabled by default

Best Practices:
  - Enable app-consistent snapshots
  - Set RPO based on business requirements
  - Use multi-VM consistency for multi-tier apps
  - Regular test failovers (quarterly)

EOF
```

---

## Step 24 – Create Automation Script

```bash
# Create PowerShell automation template
cat > asr-automation.ps1 << 'EOF'
# Azure Site Recovery Automation Example

# Variables
$vaultName = "vault-asr"
$resourceGroupName = "rg-asr"
$vmName = "vm-production"

# Get vault
$vault = Get-AzRecoveryServicesVault -Name $vaultName

# Set vault context
Set-AzRecoveryServicesAsrVaultContext -Vault $vault

# Get replication protected item
$protectedItem = Get-AzRecoveryServicesAsrReplicationProtectedItem `
  -ProtectionContainer $container `
  -Name $vmName

# Test failover
$TFOJob = Start-AzRecoveryServicesAsrTestFailoverJob `
  -ReplicationProtectedItem $protectedItem `
  -Direction PrimaryToRecovery `
  -AzureVMNetworkId $targetNetworkId

# Monitor job
while (($TFOJob.State -eq "InProgress") -or ($TFOJob.State -eq "NotStarted")) {
  Start-Sleep -Seconds 10
  $TFOJob = Get-AzRecoveryServicesAsrJob -Job $TFOJob
}

Write-Output "Test Failover Status: $($TFOJob.State)"
EOF

echo "Automation script created"
```

---

## Step 25 – Show Cost Optimization

```bash
# Display cost considerations
cat << 'EOF'

Site Recovery Cost Optimization:
=================================

Licensing:
  - Protected Instance: Per VM per month
  - Standard Tier: Full features
  - No charge for failed-over VMs during testing

Storage Costs:
  - Cache storage in source region
  - Replicated data in target region
  - Snapshots for recovery points

Network Costs:
  - Replication bandwidth (outbound data)
  - Failover data transfer
  - Consider ExpressRoute for large workloads

Optimization Tips:
  - Use Standard HDD for cache storage
  - Optimize replication frequency
  - Compress data during replication
  - Clean up old recovery points
  - Deallocate test failover VMs

EOF
```

---

## Step 26 – List Recovery Services Vaults

```bash
# List all Recovery Services vaults
az backup vault list \
  --query "[].{Name:name, Location:location, ResourceGroup:resourceGroup}" \
  --output table
```

---

## Step 27 – Show VM Details

```bash
# Display source VM information
az vm show \
  --name "$VM_NAME" \
  --resource-group "$SOURCE_RG" \
  --query "{Name:name, Size:hardwareProfile.vmSize, Location:location, ProvisioningState:provisioningState}" \
  --output table
```

---

## Step 28 – Cleanup

```bash
# Delete source resource group
az group delete \
  --name "$SOURCE_RG" \
  --yes \
  --no-wait

# Delete target resource group
az group delete \
  --name "$TARGET_RG" \
  --yes \
  --no-wait

# Remove local files
rm -f recovery-plan.json asr-automation.ps1

echo "Cleanup complete"
```

---

## Summary

You created Recovery Services vault for disaster recovery in the target region, configured source and target virtual networks in different Azure regions, deployed a production VM with a web application in the source region, created cache storage account for replication data, explored Site Recovery concepts including RPO and RTO, learned about test failover procedures for non-disruptive disaster recovery testing, reviewed replication policies with app-consistent and crash-consistent snapshots, and understood failover types including planned, unplanned, and failback operations for comprehensive business continuity planning.
