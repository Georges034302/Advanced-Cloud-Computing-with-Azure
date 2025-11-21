# Lab 15.B: Database Migration Service

## Objectives
- Create Azure Database Migration Service
- Set up source and target databases
- Configure migration project
- Perform online database migration
- Monitor migration progress
- Validate data migration
- Complete cutover process
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
RG_NAME="rg-lab15b-dms"
VNET_NAME="vnet-dms"
SUBNET_NAME="subnet-dms"
DMS_NAME="dms-migration-$RANDOM"
SOURCE_SQL="sql-source-$RANDOM"
TARGET_SQL="sql-target-$RANDOM"
DB_NAME="ContosoRetail"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$DMS_NAME"
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
# Create VNet for DMS
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.50.0.0/16 \
  --subnet-name "$SUBNET_NAME" \
  --subnet-prefix 10.50.1.0/24

echo "Virtual network created"
```

---

## Step 4 – Get Subnet ID

```bash
# Get subnet resource ID
SUBNET_ID=$(az network vnet subnet show \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --name "$SUBNET_NAME" \
  --query id \
  --output tsv)

echo "$SUBNET_ID"
```

---

## Step 5 – Show DMS Overview

```bash
# Display DMS capabilities
cat << 'EOF'

Azure Database Migration Service Overview:
===========================================

Supported Sources:
  - SQL Server (on-premises, VM, RDS)
  - MySQL
  - PostgreSQL
  - MongoDB
  - Oracle (via partner tools)

Supported Targets:
  - Azure SQL Database
  - Azure SQL Managed Instance
  - Azure Database for MySQL
  - Azure Database for PostgreSQL
  - Azure Cosmos DB (MongoDB API)

Migration Types:

Offline Migration:
  - Database unavailable during migration
  - One-time data copy
  - Faster for small databases
  - Simpler process
  - Use for: Dev/test, small DBs, scheduled downtime

Online Migration:
  - Minimal downtime
  - Continuous data sync
  - Cutover when ready
  - More complex
  - Use for: Production, large DBs, 24/7 systems

Features:
  - Schema migration
  - Data migration
  - Validation and testing
  - Minimal downtime
  - Automated process
  - Progress monitoring

EOF
```

---

## Step 6 – Register Resource Provider

```bash
# Register DataMigration provider
az provider register --namespace Microsoft.DataMigration

# Wait for registration
sleep 30

# Verify registration
az provider show \
  --namespace Microsoft.DataMigration \
  --query "registrationState" \
  --output tsv

echo "Resource provider registered"
```

---

## Step 7 – Create DMS Instance

```bash
# Create Database Migration Service
az dms create \
  --name "$DMS_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku-name Premium_4vCores \
  --subnet "$SUBNET_ID"

echo "DMS instance created"
```

---

## Step 8 – Show DMS SKUs

```bash
# Display DMS pricing tiers
cat << 'EOF'

DMS SKU Options:
================

Standard (Offline only):
  - Standard_1vCore: $0.12/hour
  - Standard_2vCores: $0.24/hour
  - Standard_4vCores: $0.48/hour
  
  Features:
    - Offline migrations only
    - Schema and data migration
    - Multiple database migration
    - Good for: Scheduled maintenance windows

Premium (Online + Offline):
  - Premium_4vCores: $1.00/hour
  - Premium_8vCores: $2.00/hour
  
  Features:
    - Online migrations (minimal downtime)
    - Offline migrations
    - Continuous data sync
    - Business continuity during migration
    - Good for: Production databases

Recommendations:
  - Development: Standard tier
  - Production: Premium tier
  - Large databases: Premium with more vCores

EOF
```

---

## Step 9 – Create Source SQL Server

```bash
# Read admin password
read -s -p "Enter SQL admin password: " SQL_PASSWORD
echo

# Create source SQL Server (simulating on-premises)
az sql server create \
  --name "$SOURCE_SQL" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --admin-user sqladmin \
  --admin-password "$SQL_PASSWORD"

echo "Source SQL Server created"
```

---

## Step 10 – Create Target SQL Server

```bash
# Create target SQL Server in Azure
az sql server create \
  --name "$TARGET_SQL" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --admin-user sqladmin \
  --admin-password "$SQL_PASSWORD"

echo "Target SQL Server created"
```

---

## Step 11 – Configure Source Firewall

```bash
# Allow Azure services to access source SQL
az sql server firewall-rule create \
  --resource-group "$RG_NAME" \
  --server "$SOURCE_SQL" \
  --name "AllowAzureServices" \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Allow DMS subnet
az sql server firewall-rule create \
  --resource-group "$RG_NAME" \
  --server "$SOURCE_SQL" \
  --name "AllowDMS" \
  --start-ip-address 10.50.1.0 \
  --end-ip-address 10.50.1.255

echo "Source firewall configured"
```

---

## Step 12 – Configure Target Firewall

```bash
# Allow Azure services to access target SQL
az sql server firewall-rule create \
  --resource-group "$RG_NAME" \
  --server "$TARGET_SQL" \
  --name "AllowAzureServices" \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Allow DMS subnet
az sql server firewall-rule create \
  --resource-group "$RG_NAME" \
  --server "$TARGET_SQL" \
  --name "AllowDMS" \
  --start-ip-address 10.50.1.0 \
  --end-ip-address 10.50.1.255

echo "Target firewall configured"
```

---

## Step 13 – Create Source Database

```bash
# Create source database with sample data
az sql db create \
  --resource-group "$RG_NAME" \
  --server "$SOURCE_SQL" \
  --name "$DB_NAME" \
  --edition GeneralPurpose \
  --compute-model Serverless \
  --family Gen5 \
  --capacity 2 \
  --auto-pause-delay 60

echo "Source database created"
```

---

## Step 14 – Create Target Database

```bash
# Create target database in Azure
az sql db create \
  --resource-group "$RG_NAME" \
  --server "$TARGET_SQL" \
  --name "$DB_NAME" \
  --edition GeneralPurpose \
  --compute-model Serverless \
  --family Gen5 \
  --capacity 2 \
  --auto-pause-delay 60

echo "Target database created"
```

---

## Step 15 – Get Connection Strings

```bash
# Get source connection string
SOURCE_CONN=$(az sql db show-connection-string \
  --client ado.net \
  --name "$DB_NAME" \
  --server "$SOURCE_SQL" \
  --output tsv | sed "s/<username>/sqladmin/g")

echo "$SOURCE_CONN"

# Get target connection string
TARGET_CONN=$(az sql db show-connection-string \
  --client ado.net \
  --name "$DB_NAME" \
  --server "$TARGET_SQL" \
  --output tsv | sed "s/<username>/sqladmin/g")

echo "$TARGET_CONN"
```

---

## Step 16 – Create Sample Schema

```bash
# Create SQL script for sample schema
cat > create-schema.sql << 'EOF'
-- Create Customers table
CREATE TABLE Customers (
    CustomerID INT PRIMARY KEY IDENTITY(1,1),
    FirstName NVARCHAR(50) NOT NULL,
    LastName NVARCHAR(50) NOT NULL,
    Email NVARCHAR(100) UNIQUE,
    Phone NVARCHAR(20),
    CreatedDate DATETIME DEFAULT GETDATE()
);

-- Create Products table
CREATE TABLE Products (
    ProductID INT PRIMARY KEY IDENTITY(1,1),
    ProductName NVARCHAR(100) NOT NULL,
    Category NVARCHAR(50),
    Price DECIMAL(10,2),
    StockQuantity INT DEFAULT 0,
    CreatedDate DATETIME DEFAULT GETDATE()
);

-- Create Orders table
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY IDENTITY(1,1),
    CustomerID INT FOREIGN KEY REFERENCES Customers(CustomerID),
    OrderDate DATETIME DEFAULT GETDATE(),
    TotalAmount DECIMAL(10,2),
    Status NVARCHAR(20)
);

-- Create OrderItems table
CREATE TABLE OrderItems (
    OrderItemID INT PRIMARY KEY IDENTITY(1,1),
    OrderID INT FOREIGN KEY REFERENCES Orders(OrderID),
    ProductID INT FOREIGN KEY REFERENCES Products(ProductID),
    Quantity INT,
    UnitPrice DECIMAL(10,2)
);

-- Create index on frequently queried columns
CREATE INDEX IX_Customers_Email ON Customers(Email);
CREATE INDEX IX_Orders_CustomerID ON Orders(CustomerID);
CREATE INDEX IX_Orders_OrderDate ON Orders(OrderDate);
CREATE INDEX IX_OrderItems_OrderID ON OrderItems(OrderID);
CREATE INDEX IX_OrderItems_ProductID ON OrderItems(ProductID);
EOF

echo "Schema script created"
```

---

## Step 17 – Create Sample Data

```bash
# Create SQL script for sample data
cat > insert-data.sql << 'EOF'
-- Insert sample customers
INSERT INTO Customers (FirstName, LastName, Email, Phone) VALUES
('John', 'Smith', 'john.smith@email.com', '0412345678'),
('Sarah', 'Johnson', 'sarah.j@email.com', '0423456789'),
('Michael', 'Brown', 'michael.b@email.com', '0434567890'),
('Emily', 'Davis', 'emily.d@email.com', '0445678901'),
('David', 'Wilson', 'david.w@email.com', '0456789012'),
('Lisa', 'Anderson', 'lisa.a@email.com', '0467890123'),
('James', 'Taylor', 'james.t@email.com', '0478901234'),
('Jennifer', 'Thomas', 'jennifer.t@email.com', '0489012345'),
('Robert', 'Martinez', 'robert.m@email.com', '0490123456'),
('Maria', 'Garcia', 'maria.g@email.com', '0401234567');

-- Insert sample products
INSERT INTO Products (ProductName, Category, Price, StockQuantity) VALUES
('Laptop Pro 15', 'Electronics', 1999.99, 50),
('Wireless Mouse', 'Accessories', 29.99, 200),
('USB-C Cable', 'Accessories', 19.99, 300),
('External Monitor', 'Electronics', 399.99, 75),
('Keyboard Mechanical', 'Accessories', 149.99, 100),
('Laptop Bag', 'Accessories', 79.99, 150),
('Webcam HD', 'Electronics', 89.99, 80),
('Headphones Bluetooth', 'Electronics', 199.99, 120),
('Docking Station', 'Accessories', 249.99, 60),
('Portable SSD 1TB', 'Storage', 159.99, 90);

-- Insert sample orders
INSERT INTO Orders (CustomerID, OrderDate, TotalAmount, Status) VALUES
(1, DATEADD(day, -30, GETDATE()), 2029.98, 'Completed'),
(2, DATEADD(day, -25, GETDATE()), 399.99, 'Completed'),
(3, DATEADD(day, -20, GETDATE()), 179.98, 'Completed'),
(4, DATEADD(day, -15, GETDATE()), 2149.98, 'Shipped'),
(5, DATEADD(day, -10, GETDATE()), 249.99, 'Shipped'),
(6, DATEADD(day, -5, GETDATE()), 89.99, 'Processing'),
(7, DATEADD(day, -3, GETDATE()), 459.98, 'Processing'),
(8, DATEADD(day, -2, GETDATE()), 199.99, 'Pending'),
(9, DATEADD(day, -1, GETDATE()), 1999.99, 'Pending'),
(10, GETDATE(), 329.98, 'Pending');

-- Insert order items
INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice) VALUES
(1, 1, 1, 1999.99), (1, 2, 1, 29.99),
(2, 4, 1, 399.99),
(3, 5, 1, 149.99), (3, 2, 1, 29.99),
(4, 1, 1, 1999.99), (4, 5, 1, 149.99),
(5, 9, 1, 249.99),
(6, 7, 1, 89.99),
(7, 8, 2, 199.99), (7, 3, 3, 19.99),
(8, 8, 1, 199.99),
(9, 1, 1, 1999.99),
(10, 10, 2, 159.99), (10, 3, 1, 19.99);
EOF

echo "Data script created"
```

---

## Step 18 – Show Migration Workflow

```bash
# Display migration process
cat << 'EOF'

Database Migration Workflow:
============================

Phase 1: Pre-Migration Assessment
  1. Run Data Migration Assistant (DMA)
  2. Identify compatibility issues
  3. Assess breaking changes
  4. Review deprecated features
  5. Generate assessment report

Phase 2: Schema Migration
  1. Export source schema
  2. Review and modify for Azure SQL
  3. Deploy schema to target
  4. Verify objects created
  5. Test constraints and indexes

Phase 3: Data Migration Setup
  1. Create DMS project
  2. Configure source connection
  3. Configure target connection
  4. Select databases and tables
  5. Map schema if needed

Phase 4: Migration Execution
  Online Migration:
    - Start continuous sync
    - Monitor replication lag
    - Validate data consistency
    - Plan cutover window
    - Stop source writes
    - Complete final sync
    - Switch applications to target
  
  Offline Migration:
    - Put source in read-only mode
    - Copy all data
    - Validate data
    - Switch applications to target

Phase 5: Post-Migration
  1. Validate application functionality
  2. Monitor performance
  3. Update connection strings
  4. Optimize indexes
  5. Update statistics
  6. Decommission source (after validation)

EOF
```

---

## Step 19 – Show Prerequisites

```bash
# Display migration prerequisites
cat << 'EOF'

Migration Prerequisites:
========================

Source Database:
  ✓ SQL Server 2012 or later
  ✓ Full backup capability
  ✓ TCP/IP protocol enabled
  ✓ SQL Server and Windows authentication
  ✓ Firewall rules for DMS
  ✓ sa or sysadmin account access

For Online Migration (additional):
  ✓ Full recovery model
  ✓ No filestream enabled
  ✓ No SQL Agent jobs migration
  ✓ Source accessible during migration

Target Database:
  ✓ Azure SQL Database or Managed Instance
  ✓ Sufficient DTUs/vCores
  ✓ Compatible service tier
  ✓ Firewall configured
  ✓ Admin account access

Network:
  ✓ DMS can reach source database
  ✓ DMS can reach target database
  ✓ Network bandwidth adequate
  ✓ Latency acceptable (<100ms recommended)

Permissions:
  Source:
    - CONTROL SERVER or sysadmin
    - db_owner on databases
  
  Target:
    - db_owner on target databases

EOF
```

---

## Step 20 – Show DMA Assessment

```bash
# Display DMA usage
cat << 'EOF'

Data Migration Assistant (DMA):
================================

Purpose:
  - Assess SQL Server compatibility with Azure SQL
  - Identify blocking issues
  - Discover new feature opportunities
  - Generate assessment report

Installation:
  Download from: https://aka.ms/dma

Assessment Process:
  1. Launch DMA
  2. Create new Assessment project
  3. Select source SQL Server version
  4. Select target: Azure SQL Database
  5. Choose assessment types:
     - Compatibility issues
     - Feature recommendations
  6. Connect to source server
  7. Select databases
  8. Start assessment
  9. Review results
  10. Export report (JSON/CSV)

Common Issues Found:
  - Deprecated features (e.g., SQL Mail)
  - Unsupported features (e.g., CLR)
  - Breaking changes in syntax
  - Login migration requirements
  - Agent jobs (not supported in SQL DB)
  - Full-text search differences

Recommendations:
  - Run DMA before creating DMS project
  - Address blocking issues first
  - Document required code changes
  - Test in non-production first

EOF
```

---

## Step 21 – Show Migration Project Types

```bash
# Display DMS project types
cat << 'EOF'

DMS Migration Project Types:
=============================

SQL Server to Azure SQL Database:
  Scenarios:
    - Single database migration
    - Multiple database migration
  
  Methods:
    - Offline: Downtime acceptable
    - Online: Minimal downtime (Premium SKU)
  
  Considerations:
    - No cross-database queries
    - No SQL Agent jobs
    - No CLR assemblies

SQL Server to Azure SQL Managed Instance:
  Scenarios:
    - Lift-and-shift migration
    - Minimal code changes
  
  Methods:
    - Offline: Faster migration
    - Online: Business continuity
  
  Features:
    - Cross-database queries supported
    - SQL Agent supported
    - Higher compatibility

MySQL to Azure Database for MySQL:
  Scenarios:
    - Open-source database migration
  
  Methods:
    - Offline migration only (via DMS)
    - Online via data-in replication

PostgreSQL to Azure Database for PostgreSQL:
  Scenarios:
    - Open-source database migration
  
  Methods:
    - Offline migration
    - pg_dump and pg_restore

MongoDB to Azure Cosmos DB:
  Scenarios:
    - NoSQL database migration
  
  Methods:
    - Offline: mongodump/mongorestore
    - Online: DMS or native tools

EOF
```

---

## Step 22 – Show Monitoring Commands

```bash
# Display DMS monitoring
cat << EOF

DMS Monitoring and Status:
==========================

Check DMS Service Status:
  az dms show \\
    --name "$DMS_NAME" \\
    --resource-group "$RG_NAME" \\
    --query "{Name:name, State:provisioningState, SKU:sku.name}" \\
    --output table

List DMS Projects:
  az dms project list \\
    --service-name "$DMS_NAME" \\
    --resource-group "$RG_NAME" \\
    --output table

Project Status:
  az dms project show \\
    --name "ProjectName" \\
    --service-name "$DMS_NAME" \\
    --resource-group "$RG_NAME"

Task Status:
  az dms project task show \\
    --name "TaskName" \\
    --project-name "ProjectName" \\
    --service-name "$DMS_NAME" \\
    --resource-group "$RG_NAME"

Metrics to Monitor:
  - Replication lag (online migration)
  - Data transfer rate
  - Errors and warnings
  - Completion percentage
  - Cutover readiness

EOF
```

---

## Step 23 – Show Cutover Process

```bash
# Display cutover steps
cat << 'EOF'

Online Migration Cutover Process:
==================================

Pre-Cutover Checklist:
  □ All data synchronized
  □ Replication lag < 1 minute
  □ No errors in migration logs
  □ Application tested on target
  □ Rollback plan documented
  □ Stakeholders notified
  □ Maintenance window scheduled

Cutover Steps:

1. Stop Application Traffic
   - Put application in maintenance mode
   - Redirect users to maintenance page
   - Stop all write operations to source

2. Wait for Final Sync
   - Monitor replication lag
   - Ensure all transactions replicated
   - Verify no pending changes
   - Typically takes 1-5 minutes

3. Verify Data Consistency
   - Compare row counts
   - Check last modified timestamps
   - Validate critical records
   - Run data validation queries

4. Complete Cutover in DMS
   - Click "Complete Cutover" in portal
   - Or use CLI command
   - DMS stops monitoring source

5. Update Connection Strings
   - Update application configuration
   - Point to new Azure SQL database
   - Update any hardcoded connections
   - Verify application.config/web.config

6. Start Application
   - Remove maintenance mode
   - Test critical functionality
   - Monitor error logs
   - Verify user access

7. Monitor Post-Cutover
   - Watch database performance
   - Check application metrics
   - Monitor user feedback
   - Keep source database for 24-48 hours

Rollback Plan:
  If issues occur:
    1. Stop application
    2. Revert connection strings to source
    3. Restart application
    4. Investigate and fix issues
    5. Plan next cutover attempt

EOF
```

---

## Step 24 – Show Best Practices

```bash
# Display migration best practices
cat << 'EOF'

Database Migration Best Practices:
===================================

Planning:
  ✓ Run DMA assessment first
  ✓ Test migration in non-production
  ✓ Document dependencies
  ✓ Plan for adequate downtime window
  ✓ Communicate with stakeholders

Performance:
  ✓ Ensure adequate network bandwidth
  ✓ Use Premium DMS SKU for large DBs
  ✓ Migrate during off-peak hours
  ✓ Disable replication triggers if possible
  ✓ Remove unnecessary indexes before migration

Security:
  ✓ Use private endpoints for DMS
  ✓ Encrypt connections (TLS 1.2+)
  ✓ Use Azure Key Vault for credentials
  ✓ Implement RBAC for DMS access
  ✓ Enable Azure AD authentication

Data Validation:
  ✓ Compare row counts per table
  ✓ Validate critical business data
  ✓ Check foreign key relationships
  ✓ Verify computed columns
  ✓ Test stored procedures

Post-Migration:
  ✓ Update statistics on all tables
  ✓ Rebuild or reorganize indexes
  ✓ Enable query store
  ✓ Configure auto-tuning
  ✓ Set up monitoring and alerts
  ✓ Implement backup strategy

Common Pitfalls to Avoid:
  ✗ Not testing in non-production
  ✗ Insufficient target database size
  ✗ Ignoring DMA warnings
  ✗ Not validating data post-migration
  ✗ Inadequate rollback planning
  ✗ Migrating during peak hours

EOF
```

---

## Step 25 – Show Troubleshooting

```bash
# Display troubleshooting guide
cat << 'EOF'

Migration Troubleshooting:
==========================

Connection Issues:

Problem: Cannot connect to source
  ✓ Check firewall rules
  ✓ Verify SQL Server allows remote connections
  ✓ Test connectivity from DMS subnet
  ✓ Check SQL Server authentication mode
  ✓ Verify credentials

Problem: Cannot connect to target
  ✓ Check Azure SQL firewall
  ✓ Verify DMS subnet allowed
  ✓ Test with sqlcmd or SSMS
  ✓ Check target server exists

Performance Issues:

Problem: Slow migration speed
  ✓ Increase DMS vCores
  ✓ Check network bandwidth
  ✓ Reduce load on source database
  ✓ Migrate during off-peak hours
  ✓ Remove unnecessary indexes

Problem: High replication lag
  ✓ Reduce write load on source
  ✓ Increase target DTUs/vCores
  ✓ Check for blocking queries
  ✓ Optimize slow queries

Data Issues:

Problem: Missing data
  ✓ Check for filtered migrations
  ✓ Verify all tables selected
  ✓ Check for case sensitivity
  ✓ Review migration logs

Problem: Data type mismatches
  ✓ Run DMA compatibility check
  ✓ Adjust schema on target
  ✓ Use appropriate data types
  ✓ Handle deprecated types

Problem: Foreign key violations
  ✓ Disable foreign keys before migration
  ✓ Migrate in correct order
  ✓ Re-enable after validation

Error Messages:

"Network path was not found"
  → Check firewall and network connectivity

"Login failed for user"
  → Verify credentials and permissions

"Database is in read-only mode"
  → Check source database status

"Insufficient DTUs"
  → Increase target database tier

EOF
```

---

## Step 26 – Show Cost Optimization

```bash
# Display cost optimization tips
cat << 'EOF'

Migration Cost Optimization:
============================

DMS Costs:
  - Standard: $0.12-0.48/hour
  - Premium: $1.00-2.00/hour
  
  Tips:
    ✓ Delete DMS after migration
    ✓ Use Standard for offline migrations
    ✓ Use Premium only for online migrations
    ✓ Right-size vCores based on database size

Target Database Costs:

Azure SQL Database:
  - DTU model: Predictable workloads
  - vCore model: More control
  
  Optimization:
    ✓ Use Serverless for variable workloads
    ✓ Enable auto-pause for dev/test
    ✓ Start with General Purpose tier
    ✓ Monitor DTU/CPU usage
    ✓ Right-size after migration

Azure SQL Managed Instance:
  - More expensive than SQL Database
  - Better for lift-and-shift
  
  Optimization:
    ✓ Use General Purpose for most workloads
    ✓ Consider Business Critical only if needed
    ✓ Use reserved capacity for 1-3 years
    ✓ Monitor vCore utilization

General Tips:
  ✓ Delete test/development migrations
  ✓ Use Azure Hybrid Benefit (save 40%)
  ✓ Reserved instances for production (save 33-64%)
  ✓ Geo-replication only if required
  ✓ Long-term retention only for compliance
  ✓ Use lifecycle policies for backups

Estimated Costs (Australia East):
  DMS Premium (migration): $24/day
  SQL Database (2 vCore GP): $400/month
  SQL Managed Instance (4 vCore GP): $1,200/month

EOF
```

---

## Step 27 – List DMS Services

```bash
# List all DMS services
az dms list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Location:location, SKU:sku.name, State:provisioningState}" \
  --output table
```

---

## Step 28 – Show DMS Details

```bash
# Get DMS service details
az dms show \
  --name "$DMS_NAME" \
  --resource-group "$RG_NAME" \
  --output json
```

---

## Step 29 – Show Validation Queries

```bash
# Create validation queries
cat > validation-queries.sql << 'EOF'
-- Pre-Migration Validation Queries
-- Run on SOURCE database

-- 1. Get row counts for all tables
SELECT 
    t.NAME AS TableName,
    SUM(p.rows) AS RowCount
FROM 
    sys.tables t
INNER JOIN 
    sys.partitions p ON t.object_id = p.object_id
WHERE 
    p.index_id IN (0,1)
GROUP BY 
    t.NAME
ORDER BY 
    t.NAME;

-- 2. Get database size
EXEC sp_spaceused;

-- 3. Check for deprecated features
SELECT 
    name,
    compatibility_level
FROM 
    sys.databases
WHERE 
    name = DB_NAME();

-- 4. List all constraints
SELECT 
    OBJECT_NAME(parent_object_id) AS TableName,
    name AS ConstraintName,
    type_desc AS ConstraintType
FROM 
    sys.foreign_keys
ORDER BY 
    TableName;

-- Post-Migration Validation Queries
-- Run on TARGET database after migration

-- 5. Compare row counts
SELECT 
    t.NAME AS TableName,
    SUM(p.rows) AS RowCount
FROM 
    sys.tables t
INNER JOIN 
    sys.partitions p ON t.object_id = p.object_id
WHERE 
    p.index_id IN (0,1)
GROUP BY 
    t.NAME
ORDER BY 
    t.NAME;

-- 6. Verify indexes
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType
FROM 
    sys.indexes i
WHERE 
    i.is_primary_key = 0 
    AND i.type > 0
ORDER BY 
    TableName, IndexName;

-- 7. Check for missing foreign keys
SELECT 
    OBJECT_NAME(parent_object_id) AS TableName,
    name AS ConstraintName
FROM 
    sys.foreign_keys
ORDER BY 
    TableName;

-- 8. Verify data integrity
SELECT 
    'Customers' AS TableName, 
    COUNT(*) AS RecordCount,
    MAX(CreatedDate) AS LatestRecord
FROM Customers
UNION ALL
SELECT 
    'Products', 
    COUNT(*), 
    MAX(CreatedDate)
FROM Products
UNION ALL
SELECT 
    'Orders', 
    COUNT(*), 
    MAX(OrderDate)
FROM Orders
UNION ALL
SELECT 
    'OrderItems', 
    COUNT(*), 
    NULL
FROM OrderItems;
EOF

echo "Validation queries created"
```

---

## Step 30 – Show Portal Steps

```bash
# Display Azure Portal migration steps
cat << 'EOF'

Migration via Azure Portal:
===========================

Step-by-Step Process:

1. Navigate to DMS:
   - Go to Azure Portal
   - Search for "Azure Database Migration Services"
   - Select your DMS instance

2. Create Migration Project:
   - Click "+ New Migration Project"
   - Project name: Enter name
   - Source server type: SQL Server
   - Target server type: Azure SQL Database
   - Migration type: Online or Offline
   - Click "Create and run activity"

3. Configure Source:
   - Server name: Your source SQL Server
   - Authentication: SQL Authentication
   - Username: sqladmin
   - Password: ********
   - Encryption: Encrypt connection
   - Trust certificate: Check
   - Click "Save"

4. Select Target:
   - Target server: Your Azure SQL Server
   - Authentication: SQL Authentication
   - Username: sqladmin
   - Password: ********
   - Click "Save"

5. Map Databases:
   - Source database: Select from dropdown
   - Target database: Select from dropdown
   - Click "Save"

6. Configure Migration Settings:
   - Select tables to migrate (or all)
   - Configure advanced settings if needed
   - Click "Save"

7. Summary and Run:
   - Review all settings
   - Activity name: Enter name
   - Click "Run migration"

8. Monitor Progress:
   - View migration status
   - Check for errors
   - Monitor replication lag (online)

9. Complete Cutover (Online only):
   - Wait for full sync
   - Click "Start Cutover"
   - Confirm cutover
   - Wait for completion

10. Validate:
    - Connect to target database
    - Run validation queries
    - Test application
    - Verify data integrity

EOF
```

---

## Step 31 – Show Additional Resources

```bash
# Display additional resources
cat << 'EOF'

Additional Migration Resources:
================================

Microsoft Documentation:
  - Azure DMS overview and tutorials
  - SQL Server to Azure SQL migration guide
  - Data Migration Assistant documentation
  - Azure SQL best practices

Tools:
  - Data Migration Assistant (DMA)
  - SQL Server Migration Assistant (SSMA)
  - Azure Data Studio
  - SQL Server Management Studio (SSMS)

Learning Resources:
  - Microsoft Learn: Database Migration
  - Azure Migration Center
  - Cloud Adoption Framework
  - Database migration videos

Partner Solutions:
  - Attunity Replicate
  - Striim
  - Qlik Replicate
  - DBmaestro

Support:
  - Azure support plans
  - Migration program assistance
  - FastTrack for Azure
  - Partner network

Community:
  - Azure forums
  - Stack Overflow
  - GitHub samples
  - Tech Community blog

EOF
```

---

## Step 32 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f create-schema.sql
rm -f insert-data.sql
rm -f validation-queries.sql

echo "Cleanup complete"
```

---

## Summary

You created Azure Database Migration Service with Premium SKU for online migrations, set up source and target Azure SQL databases to simulate migration scenario, configured virtual network and firewall rules for DMS connectivity, explored DMS capabilities for SQL Server, MySQL, PostgreSQL, and MongoDB migrations, learned about online migration with minimal downtime and offline migration with scheduled maintenance, created sample schema with customers, products, orders, and order items tables, reviewed migration workflow from pre-migration assessment through cutover process, understood Data Migration Assistant usage for compatibility assessment, explored validation queries for pre and post-migration data integrity checks, reviewed best practices for performance optimization and security, and learned troubleshooting techniques for common migration issues including connectivity, performance, and data problems.
