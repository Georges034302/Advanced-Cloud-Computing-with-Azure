# Lab 1.A: VM on Custom VNet

## Overview
This lab demonstrates how to deploy an Azure Virtual Machine on a custom Virtual Network with isolated subnets and Network Security Groups. You'll create the networking infrastructure, deploy a Linux VM, and configure secure SSH access.

---

## Objectives
- Create a custom Virtual Network with subnets
- Configure Network Security Groups for traffic control
- Deploy a Linux VM with managed identity
- Secure SSH access to the VM
- Test connectivity and verify configuration
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- SSH client (built-in on Linux/Mac, or use Azure Cloud Shell)
- Basic understanding of networking concepts
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="eastus"
RG_NAME="rg-lab1a-vm"
VNET_NAME="vnet-lab1a"
SUBNET_NAME="subnet-vm"
NSG_NAME="nsg-lab1a-vm"
VM_NAME="vm-lab1a-linux"
ADMIN_USER="azureuser"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "VNET_NAME=$VNET_NAME"
echo "VM_NAME=$VM_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group to contain all lab resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Virtual Network and Subnet

```bash
# Create VNet with address space 10.0.0.0/16
az network vnet create \
  --resource-group "$RG_NAME" \
  --name "$VNET_NAME" \
  --address-prefix 10.0.0.0/16 \
  --subnet-name "$SUBNET_NAME" \
  --subnet-prefix 10.0.1.0/24 \
  --location "$LOCATION"

# Verify VNet creation
az network vnet show \
  --resource-group "$RG_NAME" \
  --name "$VNET_NAME" \
  --query "{Name:name, AddressSpace:addressSpace.addressPrefixes, Location:location}" \
  --output table
```

---

## Step 4 – Create Network Security Group

```bash
# Create NSG for VM subnet
az network nsg create \
  --resource-group "$RG_NAME" \
  --name "$NSG_NAME" \
  --location "$LOCATION"

# Add rule to allow SSH from your IP (port 22)
# Get your current public IP
MY_IP=$(curl -s https://api.ipify.org)
echo "Your public IP: $MY_IP"

# Create SSH allow rule
az network nsg rule create \
  --resource-group "$RG_NAME" \
  --nsg-name "$NSG_NAME" \
  --name "AllowSSH" \
  --priority 100 \
  --source-address-prefixes "$MY_IP/32" \
  --source-port-ranges "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp \
  --description "Allow SSH from my IP"

# Associate NSG with subnet
az network vnet subnet update \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --name "$SUBNET_NAME" \
  --network-security-group "$NSG_NAME"
```

---

## Step 5 – Generate SSH Key Pair

```bash
# Generate SSH key pair (skip if you already have one)
SSH_KEY_PATH="$HOME/.ssh/id_rsa_lab1a"

if [ ! -f "$SSH_KEY_PATH" ]; then
  ssh-keygen -t rsa -b 4096 -f "$SSH_KEY_PATH" -N "" -C "$ADMIN_USER@lab1a"
  echo "✅ SSH key pair created at $SSH_KEY_PATH"
else
  echo "✅ SSH key already exists at $SSH_KEY_PATH"
fi

# Display public key
echo "Public key:"
cat "${SSH_KEY_PATH}.pub"
```

---

## Step 6 – Create Virtual Machine

```bash
# Create Ubuntu VM with managed identity
az vm create \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --location "$LOCATION" \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_NAME" \
  --nsg "" \
  --image "Ubuntu2204" \
  --size "Standard_B2s" \
  --admin-username "$ADMIN_USER" \
  --ssh-key-values "${SSH_KEY_PATH}.pub" \
  --public-ip-sku Standard \
  --assign-identity \
  --output table

# VM creation takes 2-3 minutes
echo "⏳ VM deployment in progress..."
```

---

## Step 7 – Get VM Public IP

```bash
# Retrieve VM public IP address
VM_PUBLIC_IP=$(az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "VM Public IP: $VM_PUBLIC_IP"
```

---

## Step 8 – Test SSH Connection

```bash
# Test SSH connection to VM
ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no "$ADMIN_USER@$VM_PUBLIC_IP" "echo '✅ SSH connection successful'; hostname; uname -a"

# Interactive SSH session (optional)
# ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP"
```

---

## Step 9 – Verify VM Configuration

```bash
# Get VM details
az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query "{Name:name, Location:location, Size:hardwareProfile.vmSize, OS:storageProfile.imageReference.offer, PrivateIP:privateIps, PublicIP:'$VM_PUBLIC_IP'}" \
  --output table

# Get VM identity
az vm identity show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query "{Type:type, PrincipalId:principalId}" \
  --output table

# List network interfaces
az vm nic list \
  --resource-group "$RG_NAME" \
  --vm-name "$VM_NAME" \
  --query "[].{Name:id, Primary:primary}" \
  --output table
```

---

## Step 10 – Test Network Connectivity

```bash
# Run network diagnostics from VM
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << 'SSHEOF'
echo "=== Network Configuration ==="
ip addr show
echo ""
echo "=== Default Route ==="
ip route
echo ""
echo "=== DNS Resolution ==="
nslookup microsoft.com
echo ""
echo "=== Internet Connectivity ==="
curl -s https://api.ipify.org
SSHEOF
```

---

## Step 11 – View NSG Rules

```bash
# List effective NSG rules on VM
az network nsg show \
  --resource-group "$RG_NAME" \
  --name "$NSG_NAME" \
  --query "securityRules[].{Name:name, Priority:priority, Direction:direction, Access:access, Protocol:protocol, SourcePort:sourcePortRange, DestPort:destinationPortRange}" \
  --output table
```

---

## Step 12 – Cleanup

```bash
# Delete resource group and all contained resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Verify deletion started
az group list --query "[?name=='$RG_NAME']" --output table

# Optional: Remove SSH key pair
# rm -f "$SSH_KEY_PATH" "${SSH_KEY_PATH}.pub"

echo "✅ Cleanup initiated - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Custom Virtual Network with isolated subnet
- Network Security Group with SSH access rules
- Linux Virtual Machine with managed identity
- Secure SSH access from your IP address

**Architecture:**
```
Internet → NSG (SSH Rule) → VNet (10.0.0.0/16) → Subnet (10.0.1.0/24) → VM (Ubuntu)
```

**Key Components:**
- **Virtual Network**: Isolated network environment in Azure
- **Subnet**: Logical network segment within VNet
- **NSG**: Firewall rules for inbound/outbound traffic
- **Virtual Machine**: Ubuntu Linux compute instance
- **Managed Identity**: Azure AD identity for the VM
- **Public IP**: Internet-accessible endpoint

**What You Learned:**
- Create Azure Virtual Networks with custom addressing
- Configure Network Security Groups for traffic filtering
- Deploy Linux VMs with SSH authentication
- Use Azure managed identities
- Verify network connectivity and configuration

---

## Best Practices

**Networking Security:**
- Always restrict NSG rules to specific source IPs
- Use SSH keys instead of passwords
- Disable password authentication on VMs
- Apply NSGs at subnet level for centralized control
- Use private IPs for internal communication

**VM Configuration:**
- Enable managed identities for Azure resource access
- Use Standard SKU for public IPs (required for availability zones)
- Choose appropriate VM sizes based on workload
- Enable boot diagnostics for troubleshooting
- Use managed disks (default)

**Resource Organization:**
- Use consistent naming conventions
- Tag resources for cost tracking and organization
- Keep related resources in same resource group
- Document custom configurations
- Use resource locks for production environments

**Cost Optimization:**
- Deallocate VMs when not in use (`az vm deallocate`)
- Use B-series VMs for dev/test workloads
- Delete unused public IPs
- Review and optimize VM sizes regularly
- Use Azure Hybrid Benefit for Windows VMs

---

## Production Enhancements

**1. Add Application Security Groups**
```bash
# Create ASG for web tier
az network asg create \
  --resource-group "$RG_NAME" \
  --name "asg-web" \
  --location "$LOCATION"

# Update NSG rule to use ASG
az network nsg rule create \
  --nsg-name "$NSG_NAME" \
  --resource-group "$RG_NAME" \
  --name "AllowWebTraffic" \
  --priority 200 \
  --source-asgs "asg-web" \
  --destination-port-ranges 80 443
```

**2. Enable VM Auto-Shutdown**
```bash
# Schedule automatic shutdown at 7 PM EST
az vm auto-shutdown \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --time "1900" \
  --timezone "Eastern Standard Time"
```

**3. Add Additional Subnet**
```bash
# Create subnet for database tier
az network vnet subnet create \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --name "subnet-db" \
  --address-prefix 10.0.2.0/24
```

**4. Enable Azure Bastion**
```bash
# Create Bastion subnet (must be named AzureBastionSubnet)
az network vnet subnet create \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --name "AzureBastionSubnet" \
  --address-prefix 10.0.255.0/27

# Create Bastion public IP
az network public-ip create \
  --resource-group "$RG_NAME" \
  --name "pip-bastion" \
  --sku Standard \
  --location "$LOCATION"

# Create Bastion host
az network bastion create \
  --resource-group "$RG_NAME" \
  --name "bastion-lab1a" \
  --vnet-name "$VNET_NAME" \
  --public-ip-address "pip-bastion" \
  --location "$LOCATION"
```

---

## Troubleshooting

**Cannot SSH to VM:**
- Verify NSG rule allows SSH from your IP
- Check VM is running: `az vm get-instance-view`
- Confirm public IP is attached to VM
- Test SSH key permissions: `chmod 600 $SSH_KEY_PATH`
- Check Azure Network Watcher for connectivity issues

**VM creation fails:**
- Verify quota availability for VM size
- Check region supports selected VM size
- Ensure VNet and subnet exist
- Verify SSH public key format is correct
- Review error message for specific issues

**NSG rules not working:**
- Check rule priority (lower number = higher priority)
- Verify rule direction (Inbound/Outbound)
- Confirm NSG is associated with subnet or NIC
- Test with Network Watcher IP flow verify
- Remember Azure platform rules have priority 65000+

**Managed identity not working:**
- Verify identity is enabled: `az vm identity show`
- Check RBAC role assignments
- Allow time for role propagation (up to 5 minutes)
- Use correct identity scope and permissions
- Test with Azure CLI using managed identity

**Public IP not accessible:**
- Confirm VM has Standard SKU public IP
- Verify NSG allows required ports
- Check VM is in running state
- Test from different source IP
- Review Azure DDoS protection settings

---

## Additional Resources

- [Azure Virtual Networks Documentation](https://docs.microsoft.com/azure/virtual-network/)
- [Network Security Groups Overview](https://docs.microsoft.com/azure/virtual-network/network-security-groups-overview)
- [Azure Linux VMs Documentation](https://docs.microsoft.com/azure/virtual-machines/linux/)
- [Azure CLI VM Commands](https://docs.microsoft.com/cli/azure/vm)
- [Azure Managed Identities](https://docs.microsoft.com/azure/active-directory/managed-identities-azure-resources/)
