# Lab 4.A: SQL Database with Private Endpoint

## Overview
This lab demonstrates deploying Azure SQL Database with private endpoint connectivity for secure, private access. You'll create a SQL Server and database, disable public access, configure private endpoints with private DNS zones, and validate connectivity from a VM.

---

## Objectives
- Create Azure SQL Server and database
- Disable public network access to SQL Server
- Configure VNet and subnet for private connectivity
- Create private endpoint for SQL Database
- Configure Private DNS zone for name resolution
- Validate private-only connectivity from a VM
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Basic SQL and networking knowledge
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab4a-sqlpe"
VNET_NAME="vnet-sql"
SUBNET_NAME="subnet-sql"
SQL_SERVER="sqlserver-lab4a-$RANDOM"
SQL_DB="sqldb-lab4a"
SQL_ADMIN="sqladmin"
SQL_PASSWORD="P@ssw0rd$(date +%s | tail -c 6)!"
PE_NAME="pe-sql"
VM_NAME="vm-sql-test"
ADMIN_USER="azureuser"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "SQL_SERVER=$SQL_SERVER"
echo "SQL_ADMIN=$SQL_ADMIN"
echo "SQL_PASSWORD=$SQL_PASSWORD"
echo ""
echo "⚠️  Save SQL_PASSWORD - you'll need it later"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for SQL and networking resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Virtual Network and Subnet

```bash
# Create virtual network
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.0.0.0/16

# Create subnet for private endpoint
az network vnet subnet create \
  --name "$SUBNET_NAME" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --address-prefix 10.0.1.0/24

# Disable private endpoint network policies on subnet
az network vnet subnet update \
  --name "$SUBNET_NAME" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --disable-private-endpoint-network-policies true

# Verify VNet and subnet
az network vnet subnet show \
  --name "$SUBNET_NAME" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --query "{Name:name, AddressPrefix:addressPrefix, PrivateEndpointNetworkPolicies:privateEndpointNetworkPolicies}" \
  --output table
```

---

## Step 4 – Create Azure SQL Server

```bash
# Create SQL Server
az sql server create \
  --name "$SQL_SERVER" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --admin-user "$SQL_ADMIN" \
  --admin-password "$SQL_PASSWORD"

# Verify SQL Server creation
az sql server show \
  --name "$SQL_SERVER" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, State:state, FQDN:fullyQualifiedDomainName}" \
  --output table
```

---

## Step 5 – Create SQL Database

```bash
# Create SQL Database (Basic tier)
az sql db create \
  --name "$SQL_DB" \
  --resource-group "$RG_NAME" \
  --server "$SQL_SERVER" \
  --edition Basic \
  --capacity 5 \
  --zone-redundant false

# Verify database creation
az sql db show \
  --name "$SQL_DB" \
  --resource-group "$RG_NAME" \
  --server "$SQL_SERVER" \
  --query "{Name:name, Status:status, Edition:edition, Size:maxSizeBytes}" \
  --output table
```

---

## Step 6 – Disable Public Network Access

```bash
# Disable public network access to SQL Server
az sql server update \
  --name "$SQL_SERVER" \
  --resource-group "$RG_NAME" \
  --set publicNetworkAccess="Disabled"

# Verify public access is disabled
az sql server show \
  --name "$SQL_SERVER" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, PublicNetworkAccess:publicNetworkAccess}" \
  --output table

echo "✅ Public access disabled - SQL Server accessible only via private endpoint"
```

---

## Step 7 – Create Private Endpoint

```bash
# Get SQL Server resource ID
SQL_SERVER_ID=$(az sql server show \
  --name "$SQL_SERVER" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "SQL Server ID: $SQL_SERVER_ID"

# Create private endpoint for SQL Server
az network private-endpoint create \
  --name "$PE_NAME" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_NAME" \
  --private-connection-resource-id "$SQL_SERVER_ID" \
  --group-id sqlServer \
  --connection-name "sql-pe-connection" \
  --location "$LOCATION"

# Get private endpoint details
az network private-endpoint show \
  --name "$PE_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, ProvisioningState:provisioningState}" \
  --output table
```

---

## Step 8 – Create Private DNS Zone

```bash
# Create Private DNS Zone for SQL Database
az network private-dns zone create \
  --name "privatelink.database.windows.net" \
  --resource-group "$RG_NAME"

# Link Private DNS Zone to VNet
az network private-dns link vnet create \
  --name "sql-dns-link" \
  --resource-group "$RG_NAME" \
  --zone-name "privatelink.database.windows.net" \
  --virtual-network "$VNET_NAME" \
  --registration-enabled false

# Verify DNS zone and link
az network private-dns link vnet show \
  --name "sql-dns-link" \
  --resource-group "$RG_NAME" \
  --zone-name "privatelink.database.windows.net" \
  --query "{Name:name, VNet:virtualNetwork.id, ProvisioningState:provisioningState}" \
  --output table
```

---

## Step 9 – Create DNS Zone Group for Private Endpoint

```bash
# Create DNS zone group to automatically manage DNS records
az network private-endpoint dns-zone-group create \
  --name "sql-dns-zone-group" \
  --resource-group "$RG_NAME" \
  --endpoint-name "$PE_NAME" \
  --private-dns-zone "privatelink.database.windows.net" \
  --zone-name "sql-zone"

# List DNS records in private zone
az network private-dns record-set a list \
  --resource-group "$RG_NAME" \
  --zone-name "privatelink.database.windows.net" \
  --output table

echo "✅ Private DNS zone configured for SQL Server"
```

---

## Step 10 – Create Test VM in Same VNet

```bash
# Generate SSH key
SSH_KEY_PATH="$HOME/.ssh/id_rsa_lab4a"
if [ ! -f "$SSH_KEY_PATH" ]; then
  ssh-keygen -t rsa -b 4096 -f "$SSH_KEY_PATH" -N "" -C "$ADMIN_USER@lab4a"
fi

# Create VM in the same VNet
az vm create \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --location "$LOCATION" \
  --image "Ubuntu2204" \
  --size "Standard_B2s" \
  --admin-username "$ADMIN_USER" \
  --ssh-key-values "${SSH_KEY_PATH}.pub" \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_NAME" \
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

## Step 11 – Install SQL Tools on VM

```bash
# SSH to VM and install SQL tools
ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no "$ADMIN_USER@$VM_PUBLIC_IP" << 'EOFVM'
# Update packages
sudo apt-get update -y

# Install curl and gnupg
sudo apt-get install -y curl gnupg

# Add Microsoft repository
curl https://packages.microsoft.com/keys/microsoft.asc | sudo tee /etc/apt/trusted.gpg.d/microsoft.asc
curl https://packages.microsoft.com/config/ubuntu/22.04/prod.list | sudo tee /etc/apt/sources.list.d/mssql-release.list

# Update and install SQL tools
sudo apt-get update -y
sudo ACCEPT_EULA=Y apt-get install -y mssql-tools18 unixodbc-dev

# Add to PATH
echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> ~/.bashrc
export PATH="$PATH:/opt/mssql-tools18/bin"

# Verify installation
sqlcmd -?

echo "✅ SQL tools installed"
EOFVM
```

---

## Step 12 – Test Private Connectivity to SQL Database

```bash
# Test DNS resolution and connectivity from VM
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << EOFVM
# Test DNS resolution (should resolve to private IP)
echo "=== DNS Resolution ==="
nslookup ${SQL_SERVER}.database.windows.net

# Test connection to SQL Server
echo -e "\n=== Testing SQL Connection ==="
sqlcmd -S ${SQL_SERVER}.database.windows.net -U ${SQL_ADMIN} -P '${SQL_PASSWORD}' -C -Q "SELECT @@VERSION;"

if [ \$? -eq 0 ]; then
  echo "✅ Successfully connected to SQL Database via private endpoint"
else
  echo "❌ Connection failed"
fi
EOFVM
```

---

## Step 13 – Create Table and Insert Data

```bash
# Create sample table and insert data
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << EOFVM
# Connect and create table
sqlcmd -S ${SQL_SERVER}.database.windows.net -d ${SQL_DB} -U ${SQL_ADMIN} -P '${SQL_PASSWORD}' -C << 'EOFSQL'
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY IDENTITY(1,1),
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50),
    Department NVARCHAR(50),
    Salary DECIMAL(10,2),
    HireDate DATE
);

INSERT INTO Employees (FirstName, LastName, Department, Salary, HireDate)
VALUES 
    ('John', 'Smith', 'Engineering', 85000.00, '2023-01-15'),
    ('Sarah', 'Johnson', 'Marketing', 72000.00, '2023-03-20'),
    ('Michael', 'Brown', 'Sales', 68000.00, '2023-05-10'),
    ('Emily', 'Davis', 'Engineering', 92000.00, '2023-02-01'),
    ('David', 'Wilson', 'HR', 65000.00, '2023-04-15');

SELECT * FROM Employees;
GO
EOFSQL

echo "✅ Table created and data inserted"
EOFVM
```

---

## Step 14 – Query Data from SQL Database

```bash
# Run queries on the database
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << EOFVM
echo "=== All Employees ==="
sqlcmd -S ${SQL_SERVER}.database.windows.net -d ${SQL_DB} -U ${SQL_ADMIN} -P '${SQL_PASSWORD}' -C -Q "SELECT * FROM Employees;"

echo -e "\n=== Engineering Department ==="
sqlcmd -S ${SQL_SERVER}.database.windows.net -d ${SQL_DB} -U ${SQL_ADMIN} -P '${SQL_PASSWORD}' -C -Q "SELECT FirstName, LastName, Salary FROM Employees WHERE Department='Engineering';"

echo -e "\n=== Average Salary by Department ==="
sqlcmd -S ${SQL_SERVER}.database.windows.net -d ${SQL_DB} -U ${SQL_ADMIN} -P '${SQL_PASSWORD}' -C -Q "SELECT Department, AVG(Salary) AS AvgSalary FROM Employees GROUP BY Department;"
EOFVM
```

---

## Step 15 – Verify Private Endpoint Connection

```bash
# List private endpoint connections
az network private-endpoint-connection list \
  --name "$SQL_SERVER" \
  --resource-group "$RG_NAME" \
  --type Microsoft.Sql/servers \
  --query "[].{Name:name, Status:properties.privateLinkServiceConnectionState.status, PrivateIP:properties.privateEndpoint.id}" \
  --output table

# Get private IP address of endpoint
az network private-endpoint show \
  --name "$PE_NAME" \
  --resource-group "$RG_NAME" \
  --query "customDnsConfigs[0].ipAddresses[0]" \
  --output tsv
```

---

## Step 16 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove SSH key (optional)
# rm -f "$SSH_KEY_PATH" "${SSH_KEY_PATH}.pub"

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Azure SQL Server with database
- Private endpoint for secure connectivity
- Private DNS zone for name resolution
- VNet with subnet for private networking
- Test VM to validate private connectivity

**Architecture:**
```
VM (VNet) → Private Endpoint → SQL Server (Public Access Disabled)
              ↓
       Private DNS Zone (privatelink.database.windows.net)
```

**Key Components:**
- **Azure SQL Database**: Managed relational database service
- **Private Endpoint**: Private network interface in your VNet
- **Private DNS Zone**: Resolves SQL Server FQDN to private IP
- **VNet Integration**: Secure, private connectivity without public internet

**What You Learned:**
- Deploy Azure SQL Database with private endpoints
- Disable public network access for security
- Configure private DNS zones for name resolution
- Validate connectivity from within VNet
- Connect to SQL Database using private IP

---

## Best Practices

**Security:**
- Always disable public network access for production databases
- Use private endpoints for all database connectivity
- Implement Azure AD authentication instead of SQL authentication
- Enable Transparent Data Encryption (TDE) for data at rest
- Use firewall rules and NSGs for additional security layers

**Networking:**
- Place private endpoints in dedicated subnets
- Use Private DNS zones for automatic name resolution
- Configure DNS zone groups for automatic record management
- Plan IP address space carefully for VNet integration
- Use Network Security Groups to control traffic flow

**Database Management:**
- Use appropriate service tiers based on workload
- Enable automatic backups and point-in-time restore
- Monitor database performance with Query Performance Insight
- Implement elastic pools for multiple databases
- Use read replicas for scaling read workloads

**Cost Optimization:**
- Start with Basic tier for development/testing
- Use serverless compute tier for intermittent workloads
- Scale up/down based on actual usage patterns
- Enable auto-pause for serverless databases
- Review and optimize database size regularly

---

## Production Enhancements

**1. Enable Azure AD Authentication**
```bash
# Set Azure AD admin for SQL Server
az sql server ad-admin create \
  --resource-group "$RG_NAME" \
  --server-name "$SQL_SERVER" \
  --display-name "SQL Admin" \
  --object-id "<your-azure-ad-object-id>"
```

**2. Configure Advanced Threat Protection**
```bash
# Enable Advanced Data Security
az sql server threat-policy update \
  --resource-group "$RG_NAME" \
  --server "$SQL_SERVER" \
  --state Enabled \
  --email-account-admins true
```

**3. Implement Geo-Replication**
```bash
# Create read replica in another region
az sql db replica create \
  --name "$SQL_DB" \
  --resource-group "$RG_NAME" \
  --server "$SQL_SERVER" \
  --partner-server "sqlserver-secondary" \
  --partner-resource-group "$RG_NAME" \
  --partner-location "australiasoutheast"
```

**4. Enable Auditing and Diagnostic Logs**
```bash
# Create storage account for logs
LOG_STORAGE="sqlaudit$RANDOM"
az storage account create \
  --name "$LOG_STORAGE" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

# Enable auditing
az sql server audit-policy update \
  --resource-group "$RG_NAME" \
  --name "$SQL_SERVER" \
  --state Enabled \
  --storage-account "$LOG_STORAGE"
```

---

## Troubleshooting

**Cannot connect to SQL Database from VM:**
- Verify VM is in the same VNet as private endpoint
- Check NSG rules allow traffic on port 1433
- Confirm private endpoint is in "Approved" state
- Verify DNS resolution returns private IP (10.x.x.x)
- Check SQL Server credentials are correct

**DNS resolution fails:**
- Ensure Private DNS zone is linked to VNet
- Verify DNS zone group is created for private endpoint
- Check VM is using Azure-provided DNS (168.63.129.16)
- Wait a few minutes for DNS propagation
- Use `nslookup` to verify resolution

**Private endpoint creation fails:**
- Verify subnet has private endpoint network policies disabled
- Check sufficient IP addresses available in subnet
- Ensure correct resource ID for SQL Server
- Verify location matches between resources
- Review Azure subscription limits

**Public access still possible:**
- Confirm publicNetworkAccess is set to "Disabled"
- Remove any existing firewall rules
- Wait a few minutes for settings to propagate
- Test from external network to verify blocking
- Check for any virtual network rules

**SQL authentication fails:**
- Verify admin username and password are correct
- Check for special characters in password (escape properly)
- Ensure database name is correct
- Use `-C` flag with sqlcmd to trust server certificate
- Review SQL Server error logs

---

## Additional Resources

- [Azure SQL Database Documentation](https://docs.microsoft.com/azure/azure-sql/database/)
- [Private Endpoints](https://docs.microsoft.com/azure/private-link/private-endpoint-overview)
- [Private DNS Zones](https://docs.microsoft.com/azure/dns/private-dns-overview)
- [SQL Database Security](https://docs.microsoft.com/azure/azure-sql/database/security-overview)
- [SQL Database Connectivity](https://docs.microsoft.com/azure/azure-sql/database/connectivity-architecture)
