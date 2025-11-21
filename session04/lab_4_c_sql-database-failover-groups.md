# Lab 4.C: SQL Database Failover Groups

## Overview
This lab demonstrates Azure SQL Database geo-replication and automatic failover groups for high availability and disaster recovery. You'll create SQL Servers in two regions, configure a failover group, and test failover scenarios.

---

## Objectives
- Create primary SQL Server and database
- Create secondary SQL Server in different region
- Configure auto-failover group
- Test manual and automatic failover
- Validate failover using read-write listener endpoint
- Monitor replication and failover status
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Basic SQL and disaster recovery knowledge
- Locations: Australia East (primary), Australia Southeast (secondary)

---

## Step 1 – Set Variables

```bash
# Set Azure regions and resource naming
PRIMARY_LOCATION="australiaeast"
SECONDARY_LOCATION="australiasoutheast"
RG_NAME="rg-lab4c-sqlfailover"
PRIMARY_SERVER="sqlprimary-lab4c-$RANDOM"
SECONDARY_SERVER="sqlsecondary-lab4c-$RANDOM"
SQL_DB="FailoverDB"
SQL_ADMIN="sqladmin"
SQL_PASSWORD="P@ssw0rd$(date +%s | tail -c 6)!"
FAILOVER_GROUP="fg-lab4c"

# Display configuration
echo "PRIMARY_LOCATION=$PRIMARY_LOCATION"
echo "SECONDARY_LOCATION=$SECONDARY_LOCATION"
echo "RG_NAME=$RG_NAME"
echo "PRIMARY_SERVER=$PRIMARY_SERVER"
echo "SECONDARY_SERVER=$SECONDARY_SERVER"
echo "SQL_ADMIN=$SQL_ADMIN"
echo "SQL_PASSWORD=$SQL_PASSWORD"
echo "FAILOVER_GROUP=$FAILOVER_GROUP"
echo ""
echo "⚠️  Save SQL_PASSWORD - you'll need it later"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group in primary region
az group create \
  --name "$RG_NAME" \
  --location "$PRIMARY_LOCATION"
```

---

## Step 3 – Create Primary SQL Server

```bash
# Create primary SQL Server
az sql server create \
  --name "$PRIMARY_SERVER" \
  --resource-group "$RG_NAME" \
  --location "$PRIMARY_LOCATION" \
  --admin-user "$SQL_ADMIN" \
  --admin-password "$SQL_PASSWORD"

# Verify primary server creation
az sql server show \
  --name "$PRIMARY_SERVER" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, State:state, FQDN:fullyQualifiedDomainName}" \
  --output table
```

---

## Step 4 – Create Primary Database

```bash
# Create database on primary server
az sql db create \
  --name "$SQL_DB" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --edition Standard \
  --capacity 10 \
  --zone-redundant false

# Verify database creation
az sql db show \
  --name "$SQL_DB" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --query "{Name:name, Status:status, Edition:edition, Location:location}" \
  --output table
```

---

## Step 5 – Enable Firewall Rule for Testing

```bash
# Get your public IP
MY_IP=$(curl -s https://api.ipify.org)
echo "Your IP: $MY_IP"

# Add firewall rule to primary server
az sql server firewall-rule create \
  --name "AllowMyIP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --start-ip-address "$MY_IP" \
  --end-ip-address "$MY_IP"

# List firewall rules
az sql server firewall-rule list \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --output table
```

---

## Step 6 – Create Secondary SQL Server

```bash
# Create secondary SQL Server in different region
az sql server create \
  --name "$SECONDARY_SERVER" \
  --resource-group "$RG_NAME" \
  --location "$SECONDARY_LOCATION" \
  --admin-user "$SQL_ADMIN" \
  --admin-password "$SQL_PASSWORD"

# Verify secondary server creation
az sql server show \
  --name "$SECONDARY_SERVER" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, State:state, FQDN:fullyQualifiedDomainName}" \
  --output table

# Add firewall rule to secondary server
az sql server firewall-rule create \
  --name "AllowMyIP" \
  --resource-group "$RG_NAME" \
  --server "$SECONDARY_SERVER" \
  --start-ip-address "$MY_IP" \
  --end-ip-address "$MY_IP"
```

---

## Step 7 – Create Failover Group

```bash
# Create failover group with automatic failover policy
az sql failover-group create \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --partner-server "$SECONDARY_SERVER" \
  --partner-resource-group "$RG_NAME" \
  --failover-policy Automatic \
  --grace-period 1 \
  --add-db "$SQL_DB"

echo "⏳ Failover group creation and initial sync can take 5-10 minutes"

# Verify failover group creation
az sql failover-group show \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --query "{Name:name, ReplicationRole:replicationRole, ReplicationState:replicationState}" \
  --output table
```

---

## Step 8 – Get Failover Group Endpoints

```bash
# Get read-write listener endpoint
RW_LISTENER=$(az sql failover-group show \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --query "readWriteEndpoint.endpoint" \
  --output tsv)

# Get read-only listener endpoint
RO_LISTENER=$(az sql failover-group show \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --query "readOnlyEndpoint.endpoint" \
  --output tsv)

echo "Read-Write Listener: $RW_LISTENER"
echo "Read-Only Listener: $RO_LISTENER"
echo ""
echo "ℹ️  Use read-write listener in application connection strings"
```

---

## Step 9 – Create Sample Data in Primary Database

```bash
# Install sqlcmd if not available
if ! command -v sqlcmd &> /dev/null; then
  echo "Installing SQL tools..."
  # For Ubuntu/Debian
  curl https://packages.microsoft.com/keys/microsoft.asc | sudo tee /etc/apt/trusted.gpg.d/microsoft.asc
  curl https://packages.microsoft.com/config/ubuntu/22.04/prod.list | sudo tee /etc/apt/sources.list.d/mssql-release.list
  sudo apt-get update -y
  sudo ACCEPT_EULA=Y apt-get install -y mssql-tools18 unixodbc-dev
  export PATH="$PATH:/opt/mssql-tools18/bin"
fi

# Create table and insert data
sqlcmd -S "${PRIMARY_SERVER}.database.windows.net" -d "$SQL_DB" -U "$SQL_ADMIN" -P "$SQL_PASSWORD" -C << 'EOFSQL'
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY IDENTITY(1,1),
    CustomerName NVARCHAR(100),
    Product NVARCHAR(100),
    Quantity INT,
    OrderDate DATETIME DEFAULT GETDATE(),
    Region NVARCHAR(50)
);

INSERT INTO Orders (CustomerName, Product, Quantity, Region)
VALUES 
    ('John Smith', 'Laptop', 2, 'Australia East'),
    ('Sarah Jones', 'Monitor', 3, 'Australia East'),
    ('Mike Wilson', 'Keyboard', 5, 'Australia East'),
    ('Emma Brown', 'Mouse', 10, 'Australia East'),
    ('David Lee', 'Headphones', 4, 'Australia East');

SELECT * FROM Orders;
GO
EOFSQL

echo "✅ Sample data created in primary database"
```

---

## Step 10 – Verify Replication to Secondary

```bash
# Wait for replication (30 seconds)
echo "⏳ Waiting for replication to complete..."
sleep 30

# Query secondary database (read-only)
echo "=== Data on Secondary Server ==="
sqlcmd -S "${SECONDARY_SERVER}.database.windows.net" -d "$SQL_DB" -U "$SQL_ADMIN" -P "$SQL_PASSWORD" -C -Q "SELECT COUNT(*) AS TotalOrders FROM Orders;"

# Verify replication state
az sql failover-group show \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --query "{Role:replicationRole, State:replicationState, Databases:databases[0].name}" \
  --output table
```

---

## Step 11 – Test Failover Using Listener Endpoint

```bash
# Connect via read-write listener (always points to primary)
echo "=== Connecting via Read-Write Listener ==="
sqlcmd -S "$RW_LISTENER" -d "$SQL_DB" -U "$SQL_ADMIN" -P "$SQL_PASSWORD" -C -Q "SELECT @@SERVERNAME AS CurrentServer, COUNT(*) AS TotalOrders FROM Orders;"

echo "ℹ️  Listener automatically routes to current primary server"
```

---

## Step 12 – Perform Manual Failover

```bash
# Initiate manual failover to secondary region
echo "=== Starting Manual Failover ==="
az sql failover-group set-primary \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$SECONDARY_SERVER"

echo "⏳ Failover in progress (1-2 minutes)..."
sleep 60

# Verify new primary
az sql failover-group show \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$SECONDARY_SERVER" \
  --query "{Name:name, Role:replicationRole, State:replicationState}" \
  --output table
```

---

## Step 13 – Verify Failover via Listener

```bash
# Connect again via listener (now points to secondary as new primary)
echo "=== After Failover - Connecting via Read-Write Listener ==="
sqlcmd -S "$RW_LISTENER" -d "$SQL_DB" -U "$SQL_ADMIN" -P "$SQL_PASSWORD" -C -Q "SELECT @@SERVERNAME AS CurrentPrimaryServer, COUNT(*) AS TotalOrders FROM Orders;"

echo "✅ Failover successful - listener automatically updated"
```

---

## Step 14 – Insert Data After Failover

```bash
# Insert new data via listener (writes to new primary)
sqlcmd -S "$RW_LISTENER" -d "$SQL_DB" -U "$SQL_ADMIN" -P "$SQL_PASSWORD" -C << 'EOFSQL'
INSERT INTO Orders (CustomerName, Product, Quantity, Region)
VALUES 
    ('Alice Cooper', 'Tablet', 2, 'Australia Southeast'),
    ('Bob Taylor', 'Webcam', 1, 'Australia Southeast');

SELECT * FROM Orders ORDER BY OrderID DESC;
GO
EOFSQL

echo "✅ Data inserted after failover"
```

---

## Step 15 – Failback to Original Primary

```bash
# Failback to original primary server
echo "=== Failing Back to Original Primary ==="
az sql failover-group set-primary \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER"

echo "⏳ Failback in progress..."
sleep 60

# Verify failback
az sql failover-group show \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --query "{Name:name, Role:replicationRole, State:replicationState}" \
  --output table

# Verify all data is present
sqlcmd -S "$RW_LISTENER" -d "$SQL_DB" -U "$SQL_ADMIN" -P "$SQL_PASSWORD" -C -Q "SELECT COUNT(*) AS TotalOrders FROM Orders;"

echo "✅ Failback complete"
```

---

## Step 16 – Monitor Failover Group Health

```bash
# Get detailed failover group information
az sql failover-group show \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --output json | python3 -m json.tool

# List databases in failover group
az sql failover-group show \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --query "databases[]" \
  --output table
```

---

## Step 17 – Cleanup

```bash
# Remove database from failover group (required before deletion)
az sql failover-group update \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --remove databases "$SQL_DB"

# Delete failover group
az sql failover-group delete \
  --name "$FAILOVER_GROUP" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER"

# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Primary SQL Server and database
- Secondary SQL Server in different region
- Auto-failover group with geo-replication
- Read-write and read-only listener endpoints
- Tested manual failover and failback

**Architecture:**
```
Application → RW Listener → Primary Server (Australia East)
                                ↓ (Geo-Replication)
                              Secondary Server (Australia Southeast)
                                ↓ (Failover)
               RW Listener → New Primary (Australia Southeast)
```

**Key Components:**
- **Failover Group**: Manages replication and failover
- **Read-Write Listener**: Always routes to current primary
- **Read-Only Listener**: Routes to secondary for read scaling
- **Geo-Replication**: Asynchronous data replication across regions
- **Automatic Failover**: Triggers failover on outage (1-hour grace period)

**What You Learned:**
- Configure SQL Database failover groups
- Set up geo-replication across regions
- Perform manual and automatic failover
- Use listener endpoints for automatic routing
- Test disaster recovery scenarios
- Monitor replication health

---

## Best Practices

**Failover Group Configuration:**
- Use automatic failover for production workloads
- Set appropriate grace period (1-2 hours typical)
- Include all related databases in same failover group
- Use listener endpoints in connection strings
- Test failover procedures regularly

**High Availability:**
- Deploy primary and secondary in different regions
- Enable zone redundancy for both servers
- Monitor replication lag and health
- Implement retry logic in applications
- Use read-only listener for read scaling

**Application Design:**
- Always use listener endpoints, not server names
- Implement connection retry logic
- Handle brief connection interruptions during failover
- Use read-only listener for reporting workloads
- Cache non-critical data to reduce database load

**Testing:**
- Perform regular DR drills
- Test failover during maintenance windows
- Verify application behavior post-failover
- Document failover procedures
- Monitor metrics during and after failover

---

## Production Enhancements

**1. Add More Regions**
```bash
# Failover groups support 1 primary + 1 secondary
# For more regions, use active geo-replication
az sql db replica create \
  --name "$SQL_DB" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --partner-server "third-server" \
  --partner-resource-group "$RG_NAME"
```

**2. Configure Alerts**
```bash
# Create alert for replication lag
az monitor metrics alert create \
  --name "ReplicationLagAlert" \
  --resource-group "$RG_NAME" \
  --scopes "/subscriptions/.../resourceGroups/$RG_NAME/providers/Microsoft.Sql/servers/$PRIMARY_SERVER" \
  --condition "avg replication_lag_sec > 300" \
  --description "Alert when replication lag exceeds 5 minutes"
```

**3. Enable Zone Redundancy**
```bash
# Create zone-redundant database
az sql db create \
  --name "ZoneRedundantDB" \
  --resource-group "$RG_NAME" \
  --server "$PRIMARY_SERVER" \
  --edition Premium \
  --zone-redundant true
```

**4. Implement Read Scale-Out**
```bash
# Use read-only listener for read replicas
# Connection string for read-only workloads:
# Server=$RO_LISTENER;Database=$SQL_DB;ApplicationIntent=ReadOnly
```

---

## Troubleshooting

**Failover group creation fails:**
- Verify both servers exist in same subscription
- Check servers are in different regions
- Ensure database tier supports geo-replication (S1+)
- Verify network connectivity between regions
- Review Azure service health

**Replication lag high:**
- Check network connectivity between regions
- Monitor primary server load
- Review transaction log size
- Consider upgrading database tier
- Check for long-running transactions

**Failover not completing:**
- Verify failover policy is set correctly
- Check grace period configuration
- Review replication state (must be SYNCHRONIZED)
- Ensure no active transactions blocking failover
- Check Azure service health in both regions

**Connection fails after failover:**
- Verify using listener endpoint, not server name
- Check firewall rules on new primary
- Review connection string configuration
- Ensure application implements retry logic
- Verify DNS propagation (may take 1-2 minutes)

**Cannot connect to secondary:**
- Secondary is read-only (use read-only listener)
- Check firewall rules on secondary server
- Verify replication is complete
- Use ApplicationIntent=ReadOnly in connection string
- Review server state and availability

---

## Additional Resources

- [Failover Groups Overview](https://docs.microsoft.com/azure/azure-sql/database/auto-failover-group-overview)
- [Active Geo-Replication](https://docs.microsoft.com/azure/azure-sql/database/active-geo-replication-overview)
- [Business Continuity](https://docs.microsoft.com/azure/azure-sql/database/business-continuity-high-availability-disaster-recover-hadr-overview)
- [Connection Strategies](https://docs.microsoft.com/azure/azure-sql/database/troubleshoot-common-connectivity-issues)
- [SQL Database Best Practices](https://docs.microsoft.com/azure/azure-sql/database/best-practices)
