# Lab 5.A: Load Balancer Deployment

## Overview
This lab demonstrates Azure Load Balancer for distributing traffic across multiple VMs. You'll create a zone-redundant load balancer, configure backend pools and health probes, and test automatic failover.

---

## Objectives
- Create zone-redundant Azure Load Balancer
- Deploy multiple VMs with web servers
- Configure backend pool and health probes
- Set up load-balancing rules
- Test automatic failover when VM stops
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Basic networking knowledge
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab5a-lb"
VNET_NAME="vnet-lb"
SUBNET_NAME="subnet-vms"
LB_NAME="lb-web"
LB_FRONTEND="lb-frontend"
LB_BACKEND="lb-backend"
LB_PROBE="http-probe"
LB_RULE="http-rule"
NSG_NAME="nsg-web"
ADMIN_USER="azureuser"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$LB_NAME"
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

## Step 3 – Create Virtual Network

```bash
# Create virtual network
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.0.0.0/16

# Create subnet
az network vnet subnet create \
  --name "$SUBNET_NAME" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --address-prefix 10.0.1.0/24
```

---

## Step 4 – Create Network Security Group

```bash
# Create NSG
az network nsg create \
  --name "$NSG_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION"

# Allow HTTP traffic
az network nsg rule create \
  --name "AllowHTTP" \
  --resource-group "$RG_NAME" \
  --nsg-name "$NSG_NAME" \
  --priority 100 \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow

# Allow SSH traffic
az network nsg rule create \
  --name "AllowSSH" \
  --resource-group "$RG_NAME" \
  --nsg-name "$NSG_NAME" \
  --priority 110 \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow
```

---

## Step 5 – Create Public IP for Load Balancer

```bash
# Create zone-redundant public IP
az network public-ip create \
  --name "${LB_NAME}-pip" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard \
  --zone 1 2 3

# Get public IP address
LB_PUBLIC_IP=$(az network public-ip show \
  --name "${LB_NAME}-pip" \
  --resource-group "$RG_NAME" \
  --query ipAddress \
  --output tsv)

echo "$LB_PUBLIC_IP"
```

---

## Step 6 – Create Load Balancer

```bash
# Create zone-redundant load balancer
az network lb create \
  --name "$LB_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard \
  --public-ip-address "${LB_NAME}-pip" \
  --frontend-ip-name "$LB_FRONTEND" \
  --backend-pool-name "$LB_BACKEND"

# Verify load balancer creation
az network lb show \
  --name "$LB_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Sku:sku.name}" \
  --output table
```

---

## Step 7 – Create Health Probe

```bash
# Create HTTP health probe
az network lb probe create \
  --name "$LB_PROBE" \
  --resource-group "$RG_NAME" \
  --lb-name "$LB_NAME" \
  --protocol Http \
  --port 80 \
  --path "/" \
  --interval 15 \
  --threshold 2

# Verify health probe
az network lb probe show \
  --name "$LB_PROBE" \
  --resource-group "$RG_NAME" \
  --lb-name "$LB_NAME" \
  --query "{Name:name, Protocol:protocol, Port:port}" \
  --output table
```

---

## Step 8 – Create Load Balancing Rule

```bash
# Create load balancing rule for HTTP
az network lb rule create \
  --name "$LB_RULE" \
  --resource-group "$RG_NAME" \
  --lb-name "$LB_NAME" \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name "$LB_FRONTEND" \
  --backend-pool-name "$LB_BACKEND" \
  --probe-name "$LB_PROBE"

# Verify load balancing rule
az network lb rule show \
  --name "$LB_RULE" \
  --resource-group "$RG_NAME" \
  --lb-name "$LB_NAME" \
  --query "{Name:name, FrontendPort:frontendPort, BackendPort:backendPort}" \
  --output table
```

---

## Step 9 – Create VMs with Web Servers

```bash
# Read SSH password from user
read -s -p "Enter VM password: " ADMIN_PW
echo ""

# Create 3 VMs in loop
for i in {1..3}; do
  # Create network interface
  az network nic create \
    --name "vm${i}-nic" \
    --resource-group "$RG_NAME" \
    --vnet-name "$VNET_NAME" \
    --subnet "$SUBNET_NAME" \
    --network-security-group "$NSG_NAME" \
    --lb-name "$LB_NAME" \
    --lb-address-pools "$LB_BACKEND"

  # Create VM
  az vm create \
    --name "vm-web-${i}" \
    --resource-group "$RG_NAME" \
    --location "$LOCATION" \
    --nics "vm${i}-nic" \
    --image "Ubuntu2204" \
    --size "Standard_B1s" \
    --admin-username "$ADMIN_USER" \
    --admin-password "$ADMIN_PW" \
    --authentication-type password \
    --zone "$i" \
    --no-wait

  echo "VM ${i} creation started"
done

# Wait for all VMs to be created
az vm wait \
  --created \
  --resource-group "$RG_NAME" \
  --name "vm-web-1" \
  --interval 10

echo "VMs created"
```

---

## Step 10 – Install Web Server on VMs

```bash
# Install and configure web server on each VM
for i in {1..3}; do
  az vm run-command invoke \
    --name "vm-web-${i}" \
    --resource-group "$RG_NAME" \
    --command-id RunShellScript \
    --scripts "
      sudo apt-get update -y
      sudo apt-get install -y nginx
      echo '<html><body><h1>Server: vm-web-${i}</h1><p>Zone: ${i}</p><p>Hostname: '\$(hostname)'</p></body></html>' | sudo tee /var/www/html/index.html
      sudo systemctl restart nginx
    "
  
  echo "Web server installed on VM ${i}"
done

echo "All web servers configured"
```

---

## Step 11 – Test Load Balancer

```bash
# Wait for health probes to detect healthy backends
echo "Waiting for health probes..."
sleep 30

# Test load balancer multiple times
echo "Testing load balancer:"
for i in {1..6}; do
  echo "Request ${i}:"
  curl -s "http://${LB_PUBLIC_IP}" | grep -o '<h1>.*</h1>'
  sleep 2
done
```

---

## Step 12 – Verify Backend Pool Health

```bash
# Check backend pool status
az network lb address-pool show \
  --name "$LB_BACKEND" \
  --resource-group "$RG_NAME" \
  --lb-name "$LB_NAME" \
  --query "backendIpConfigurations[].{VM:id}" \
  --output table

# List VMs in backend pool
az network nic list \
  --resource-group "$RG_NAME" \
  --query "[?ipConfigurations[0].loadBalancerBackendAddressPools!=null].{Name:name}" \
  --output table
```

---

## Step 13 – Test Automatic Failover

```bash
# Stop one VM to test failover
echo "Stopping vm-web-1 to test failover..."
az vm stop \
  --name "vm-web-1" \
  --resource-group "$RG_NAME" \
  --no-wait

# Wait for health probe to detect unhealthy VM
echo "Waiting for health probe to detect failure..."
sleep 45

# Test load balancer again (should only hit vm-web-2 and vm-web-3)
echo "Testing after failover:"
for i in {1..6}; do
  echo "Request ${i}:"
  curl -s "http://${LB_PUBLIC_IP}" | grep -o '<h1>.*</h1>'
  sleep 2
done

echo "Failover successful - traffic routed to healthy VMs only"
```

---

## Step 14 – Restart Stopped VM

```bash
# Restart the stopped VM
echo "Restarting vm-web-1..."
az vm start \
  --name "vm-web-1" \
  --resource-group "$RG_NAME" \
  --no-wait

# Wait for VM to be healthy again
echo "Waiting for VM to become healthy..."
sleep 45

# Test load balancer (should distribute across all 3 VMs)
echo "Testing after VM restart:"
for i in {1..6}; do
  echo "Request ${i}:"
  curl -s "http://${LB_PUBLIC_IP}" | grep -o '<h1>.*</h1>'
  sleep 2
done
```

---

## Step 15 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

echo "Cleanup complete"
```

---

## Summary

You deployed a zone-redundant Azure Load Balancer with multiple VMs, configured health probes and load-balancing rules, and tested automatic failover by stopping a VM.
