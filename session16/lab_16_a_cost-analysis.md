# Lab 16.A: Cost Analysis

## Objectives
- Access Azure Cost Management and Billing
- Analyze current Azure spending
- Review cost breakdown by resource and service
- Create custom cost views
- Export cost data for reporting
- Identify cost optimization opportunities
- Understand cost allocation tags
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
RG_NAME="rg-lab16a-cost"
VNET_NAME="vnet-sample"
STORAGE_NAME="storage$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$STORAGE_NAME"
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

## Step 3 – Show Cost Management Overview

```bash
# Display Cost Management capabilities
cat << 'EOF'

Azure Cost Management + Billing:
=================================

Key Features:
  - Cost analysis and reporting
  - Budget creation and alerts
  - Cost recommendations
  - Export scheduling
  - Cost allocation
  - Showback/chargeback reporting
  - Anomaly detection

Cost Analysis Views:
  - Accumulated costs
  - Daily costs
  - Forecast
  - Cost by service
  - Cost by resource
  - Cost by location
  - Cost by resource group
  - Cost by tag

Cost Dimensions:
  - Service name (e.g., Virtual Machines)
  - Resource location
  - Resource group
  - Resource type
  - Meter category
  - Publisher type
  - Pricing model
  - Reservation

Scope Levels:
  - Subscription
  - Resource group
  - Management group
  - Billing account
  - Department
  - Enrollment account

EOF
```

---

## Step 4 – Create Sample Resources

```bash
# Create VNet
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.0.0.0/16 \
  --subnet-name default \
  --subnet-prefix 10.0.1.0/24

echo "VNet created"

# Create storage account
az storage account create \
  --name "$STORAGE_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

echo "Storage account created"
```

---

## Step 5 – Tag Resources

```bash
# Add tags to resource group for cost tracking
az group update \
  --name "$RG_NAME" \
  --tags \
    Environment=Production \
    Department=IT \
    CostCenter=CC-1001 \
    Project=CostAnalysis \
    Owner="finance@company.com"

echo "Tags applied"
```

---

## Step 6 – Show Tagging Strategy

```bash
# Display tagging best practices
cat << 'EOF'

Cost Allocation Tagging Strategy:
==================================

Essential Tags:

1. Environment:
   - Production
   - Staging
   - Development
   - Testing

2. Department/Team:
   - IT
   - Finance
   - Marketing
   - Engineering

3. Cost Center:
   - CC-1001 (IT Operations)
   - CC-2001 (Marketing)
   - CC-3001 (R&D)

4. Project:
   - Project name or code
   - Enables project-based reporting

5. Owner:
   - Email of responsible person
   - Contact for cost queries

6. Application:
   - App name or service
   - Group related resources

7. Compliance:
   - Regulatory requirements
   - Data classification

Tagging Best Practices:
  ✓ Define tagging policy
  ✓ Use Azure Policy to enforce tags
  ✓ Apply tags at resource group level
  ✓ Inherit tags where possible
  ✓ Regular tag audits
  ✓ Standardize tag names
  ✓ Document tag purposes

Benefits:
  - Accurate cost allocation
  - Department chargebacks
  - Project tracking
  - Resource organization
  - Compliance reporting

EOF
```

---

## Step 7 – Get Subscription ID

```bash
# Get current subscription ID
SUBSCRIPTION_ID=$(az account show --query id --output tsv)

echo "$SUBSCRIPTION_ID"
```

---

## Step 8 – Show Cost Query

```bash
# Display cost query structure
cat << EOF

Querying Azure Costs:
=====================

Azure CLI Cost Query:
  az costmanagement query \\
    --type Usage \\
    --scope "/subscriptions/$SUBSCRIPTION_ID" \\
    --timeframe MonthToDate

Query Types:
  - Usage: Detailed usage and charges
  - ActualCost: Amortized cost
  - AmortizedCost: Spread reservation costs

Timeframes:
  - MonthToDate
  - BillingMonthToDate
  - TheLastMonth
  - TheLastBillingMonth
  - WeekToDate
  - Custom (specify start/end dates)

Grouping Options:
  - ResourceGroupName
  - ResourceType
  - ResourceLocation
  - ServiceName
  - MeterCategory
  - Tags

EOF
```

---

## Step 9 – List Resources with Tags

```bash
# List all resources in resource group with tags
az resource list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Type:type, Tags:tags}" \
  --output table
```

---

## Step 10 – Show Cost Breakdown

```bash
# Display typical Azure cost breakdown
cat << 'EOF'

Typical Azure Cost Breakdown:
==============================

Compute (40-50%):
  - Virtual Machines
  - App Services
  - Azure Kubernetes Service
  - Function Apps
  - Container Instances

Storage (15-20%):
  - Blob Storage
  - File Storage
  - Managed Disks
  - Backup storage
  - Archive storage

Networking (10-15%):
  - Load Balancers
  - Application Gateway
  - VPN Gateway
  - ExpressRoute
  - Data transfer (egress)

Databases (15-25%):
  - Azure SQL Database
  - Cosmos DB
  - MySQL/PostgreSQL
  - Redis Cache

Other Services (5-15%):
  - Monitoring and diagnostics
  - Key Vault
  - Active Directory
  - Backup and Site Recovery
  - AI/ML services

Hidden Costs:
  - Data transfer between regions
  - Public IP addresses
  - Load balancer data processing
  - Snapshot storage
  - Diagnostic logs
  - Reserved IP addresses not in use

EOF
```

---

## Step 11 – Show Cost Optimization Tips

```bash
# Display cost optimization strategies
cat << 'EOF'

Cost Optimization Strategies:
==============================

Compute Optimization:
  ✓ Right-size VMs based on utilization
  ✓ Use B-series for variable workloads
  ✓ Enable auto-shutdown for dev/test VMs
  ✓ Use spot VMs for fault-tolerant workloads
  ✓ Reserved instances (1 or 3 year)
  ✓ Azure Hybrid Benefit (existing licenses)
  ✓ Use Serverless (Functions, Logic Apps)

Storage Optimization:
  ✓ Use appropriate access tiers
  ✓ Lifecycle management policies
  ✓ Delete unused disks and snapshots
  ✓ Use managed disks efficiently
  ✓ Enable soft delete carefully
  ✓ Compress data before storage
  ✓ Use cool/archive for old data

Networking Optimization:
  ✓ Minimize cross-region traffic
  ✓ Use Azure CDN for static content
  ✓ Delete unused public IPs
  ✓ Consolidate VPN gateways
  ✓ Use private endpoints
  ✓ Monitor bandwidth usage

Database Optimization:
  ✓ Right-size DTUs/vCores
  ✓ Use serverless tier when appropriate
  ✓ Enable auto-pause for dev databases
  ✓ Reserved capacity for production
  ✓ Optimize queries and indexes
  ✓ Use read replicas efficiently

General Tips:
  ✓ Delete unused resources
  ✓ Use dev/test pricing
  ✓ Implement governance policies
  ✓ Regular cost reviews
  ✓ Set up budgets and alerts
  ✓ Use Azure Advisor
  ✓ Optimize resource placement

Potential Savings:
  - Reserved Instances: 40-60%
  - Spot VMs: Up to 90%
  - Azure Hybrid Benefit: 40-80%
  - Right-sizing: 20-30%
  - Storage tiering: 50-70%

EOF
```

---

## Step 12 – Show Pricing Models

```bash
# Display Azure pricing models
cat << 'EOF'

Azure Pricing Models:
=====================

Pay-As-You-Go:
  - No upfront commitment
  - Pay only for what you use
  - Flexible scaling
  - Highest per-hour cost
  - Good for: Variable workloads

Reserved Instances (1-year):
  - 30-40% discount vs PAYG
  - Fixed capacity commitment
  - Can exchange or cancel
  - Good for: Predictable workloads

Reserved Instances (3-year):
  - 60-70% discount vs PAYG
  - Longer commitment
  - Better for stable production
  - Good for: Long-term workloads

Spot Instances:
  - Up to 90% discount
  - Can be evicted anytime
  - 30-second warning
  - Good for: Batch jobs, testing

Dev/Test Pricing:
  - Discounted rates
  - Must have Visual Studio subscription
  - For non-production only
  - Good for: Development, testing

Enterprise Agreement:
  - Volume discounts
  - Prepaid commitment
  - Custom pricing
  - Good for: Large organizations

CSP (Cloud Solution Provider):
  - Through Microsoft partners
  - Bundled support
  - Partner margin included
  - Good for: Managed services

EOF
```

---

## Step 13 – Show Resource Costs

```bash
# Display typical Azure resource costs (Australia East)
cat << 'EOF'

Typical Monthly Costs (Australia East):
========================================

Virtual Machines:
  B1s (1 vCPU, 1GB): $10
  B2s (2 vCPU, 4GB): $40
  D2s_v5 (2 vCPU, 8GB): $100
  D4s_v5 (4 vCPU, 16GB): $200
  E4s_v5 (4 vCPU, 32GB): $250

Storage:
  Blob Storage (LRS): $0.02/GB
  Blob Storage (GRS): $0.04/GB
  Managed Disk (Standard HDD): $5/128GB
  Managed Disk (Standard SSD): $10/128GB
  Managed Disk (Premium SSD): $25/128GB

Databases:
  SQL Database (2 vCore): $400
  SQL Database (4 vCore): $800
  Cosmos DB (400 RU/s): $25
  MySQL (2 vCore): $150
  PostgreSQL (2 vCore): $150

App Services:
  B1 (Basic): $55
  S1 (Standard): $100
  P1v3 (Premium): $220

Networking:
  VPN Gateway (Basic): $35
  VPN Gateway (VpnGw1): $150
  Load Balancer (Standard): $20
  Application Gateway: $150
  Public IP: $4
  Data transfer (first 5GB): Free
  Data transfer (per GB): $0.087

Containers:
  AKS control plane: Free
  AKS node (D2s_v3): $70
  Container Instance (1 vCPU, 1.5GB): $35
  Container Registry (Basic): $5

Monitoring:
  Log Analytics (per GB): $2.76
  Application Insights (per GB): $2.76
  Metrics (per metric): $0.25

Note: Prices are approximate and subject to change

EOF
```

---

## Step 14 – Show Cost Management API

```bash
# Display Cost Management API usage
cat << 'EOF'

Cost Management API:
====================

REST API Endpoints:

1. Usage Details:
   GET /subscriptions/{subscriptionId}/providers/Microsoft.Consumption/usageDetails

2. Cost by Resource Group:
   POST /subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.CostManagement/query

3. Forecast:
   POST /subscriptions/{subscriptionId}/providers/Microsoft.CostManagement/forecast

4. Budgets:
   GET /subscriptions/{subscriptionId}/providers/Microsoft.Consumption/budgets

5. Price Sheet:
   GET /subscriptions/{subscriptionId}/providers/Microsoft.Consumption/pricesheets/default

Query Example:
```json
{
  "type": "Usage",
  "timeframe": "MonthToDate",
  "dataset": {
    "granularity": "Daily",
    "aggregation": {
      "totalCost": {
        "name": "PreTaxCost",
        "function": "Sum"
      }
    },
    "grouping": [
      {
        "type": "Dimension",
        "name": "ServiceName"
      }
    ]
  }
}
```

Programmatic Access:
  - Azure PowerShell
  - Azure CLI
  - REST API
  - SDKs (.NET, Python, Java)
  - Power BI connector

EOF
```

---

## Step 15 – Show Export Options

```bash
# Display cost data export options
cat << 'EOF'

Cost Data Export Options:
==========================

Export Types:

1. One-time Export:
   - Manual export from portal
   - CSV format
   - Current view data

2. Scheduled Export:
   - Daily, weekly, or monthly
   - Automated delivery
   - To storage account
   - Consistent format

3. API Export:
   - Programmatic access
   - Real-time data
   - Custom applications

Export Destinations:
  - Azure Storage (Blob)
  - Download to local
  - Power BI integration
  - Custom applications

Export Formats:
  - CSV (Comma-separated)
  - JSON (for APIs)
  - Power BI dataset

Export Data Includes:
  - Usage date
  - Resource name
  - Resource group
  - Service type
  - Location
  - Quantity consumed
  - Unit price
  - Total cost
  - Tags

Best Practices:
  ✓ Export to storage account
  ✓ Set lifecycle policies
  ✓ Encrypt export data
  ✓ Regular export schedule
  ✓ Archive historical data
  ✓ Integrate with BI tools

EOF
```

---

## Step 16 – Show Billing Periods

```bash
# Display billing period information
cat << 'EOF'

Azure Billing Periods:
======================

Billing Cycle:
  - Monthly billing
  - Aligned with calendar month
  - Invoice generated 5-10 days after month end
  - Payment due 30-60 days (depends on agreement)

Pay-As-You-Go:
  - Credit card charged monthly
  - Invoice available in portal
  - Automatic payment

Enterprise Agreement:
  - Prepaid commitment
  - Annual or multi-year terms
  - Overage billed separately
  - Quarterly reconciliation

Microsoft Customer Agreement:
  - Flexible billing
  - Invoice sections
  - Multiple payment profiles
  - Modern commerce experience

Billing Documents:
  - Invoice (monthly)
  - Usage details (daily)
  - Price sheet
  - Marketplace charges
  - Credits and adjustments

Important Dates:
  - Month end: Billing period closes
  - Day 5-10: Invoice generated
  - Day 10-15: Invoice available
  - Day 30-60: Payment due

EOF
```

---

## Step 17 – Show Cost Alerts

```bash
# Display cost alert types
cat << 'EOF'

Cost Alert Types:
==================

Budget Alerts:
  - Percentage-based (50%, 75%, 90%, 100%)
  - Absolute amount
  - Actual or forecasted
  - Email notifications
  - Action groups integration

Anomaly Alerts:
  - Machine learning detection
  - Unusual spending patterns
  - Automatic notifications
  - Root cause analysis

Credit Alerts:
  - Prepaid credit balance
  - Credit expiration warnings
  - EA commitments

Department Spending Alerts:
  - EA department quotas
  - Spending thresholds
  - Department managers notified

Alert Configuration:
  1. Set budget amount
  2. Define alert threshold (e.g., 80%)
  3. Add email recipients
  4. Configure action groups
  5. Enable notifications

Action Groups:
  - Email/SMS
  - Azure Function
  - Logic App
  - Webhook
  - ITSM integration
  - Runbook

Best Practices:
  ✓ Set multiple thresholds (50%, 80%, 100%)
  ✓ Alert before overspend
  ✓ Include relevant stakeholders
  ✓ Review alerts regularly
  ✓ Automate remediation
  ✓ Document alert procedures

EOF
```

---

## Step 18 – Show FinOps Principles

```bash
# Display FinOps best practices
cat << 'EOF'

FinOps (Cloud Financial Operations):
====================================

Core Principles:

1. Teams Collaborate:
   - Finance, engineering, business
   - Shared responsibility
   - Regular sync meetings

2. Decisions Driven by Value:
   - Cost vs performance tradeoff
   - Business value alignment
   - ROI measurement

3. Everyone Takes Ownership:
   - Engineers own usage
   - Finance owns process
   - Business owns budget

FinOps Lifecycle:

1. Inform:
   - Visibility into costs
   - Accurate allocation
   - Trend analysis
   - Forecasting

2. Optimize:
   - Right-sizing resources
   - Reserved capacity
   - Spot instances
   - Architecture optimization

3. Operate:
   - Continuous improvement
   - Policy enforcement
   - Automation
   - Culture building

Key Metrics:
  - Unit economics (cost per transaction)
  - Cost per customer
  - Cost trends
  - Waste percentage
  - Reserved instance coverage
  - Savings achieved

FinOps Team Structure:
  - FinOps Lead/Manager
  - Cloud Architects
  - Finance Analysts
  - Engineering Representatives
  - Product Owners

Tools and Practices:
  ✓ Cost Management dashboards
  ✓ Tagging standards
  ✓ Showback reports
  ✓ Monthly cost reviews
  ✓ Optimization recommendations
  ✓ Budget accountability
  ✓ Training programs

EOF
```

---

## Step 19 – Show Showback vs Chargeback

```bash
# Display cost allocation models
cat << 'EOF'

Showback vs Chargeback:
========================

Showback:
  - Informational reporting
  - No actual billing
  - Cost visibility
  - Awareness building

  Benefits:
    ✓ Low friction
    ✓ Cost awareness
    ✓ Easy to implement
    ✓ Foster optimization

  Example:
    "Your department used $5,000 of Azure this month"
    (No money changes hands)

Chargeback:
  - Actual cost transfer
  - Internal billing
  - Budget enforcement
  - Accountability

  Benefits:
    ✓ True accountability
    ✓ Budget discipline
    ✓ ROI measurement
    ✓ Waste reduction

  Challenges:
    - Complex implementation
    - Finance overhead
    - Can slow innovation
    - Requires accurate allocation

  Example:
    "Your department is charged $5,000"
    (Budget is actually transferred)

Hybrid Approach:
  - Showback for most teams
  - Chargeback for large consumers
  - Transition from showback to chargeback
  - Tiered responsibility

Implementation Steps:

1. Start with Showback:
   - Build cost visibility
   - Establish tagging
   - Create reports
   - Build culture

2. Refine Allocation:
   - Improve accuracy
   - Handle shared costs
   - Define rules
   - Get stakeholder buy-in

3. Move to Chargeback:
   - Pilot with one department
   - Automate billing
   - Establish processes
   - Scale gradually

EOF
```

---

## Step 20 – Get Resource Group Cost

```bash
# Display resource group information
az group show \
  --name "$RG_NAME" \
  --query "{Name:name, Location:location, Tags:tags}" \
  --output json
```

---

## Step 21 – List All Tags

```bash
# List all tags used in subscription
az tag list --output table
```

---

## Step 22 – Show Cost Reporting

```bash
# Display cost reporting best practices
cat << 'EOF'

Cost Reporting Best Practices:
===============================

Report Types:

1. Executive Dashboard:
   - Total spend trends
   - Month-over-month comparison
   - Budget vs actual
   - Top 10 services
   - Forecast
   - Audience: Leadership

2. Department Report:
   - Department allocation
   - Project breakdown
   - Cost per application
   - Optimization opportunities
   - Audience: Department heads

3. Engineering Report:
   - Resource-level details
   - Unused resources
   - Right-sizing recommendations
   - Reservation coverage
   - Audience: Engineers

4. Finance Report:
   - Invoice reconciliation
   - Commitment usage
   - Credits and discounts
   - Variance analysis
   - Audience: Finance team

Reporting Frequency:
  - Daily: Anomaly alerts
  - Weekly: Trend monitoring
  - Monthly: Full analysis
  - Quarterly: Strategic review
  - Annually: Planning

Key Metrics to Track:
  - Total monthly spend
  - Growth rate
  - Cost per environment
  - Cost per customer (if applicable)
  - Waste percentage
  - Optimization savings
  - Budget variance
  - Forecast accuracy

Visualization Tools:
  - Power BI
  - Azure Workbooks
  - Grafana
  - Custom dashboards
  - Excel/Google Sheets

Report Distribution:
  - Email digest
  - Shared dashboard
  - Slack/Teams notifications
  - Regular meetings
  - Self-service portal

EOF
```

---

## Step 23 – Show Common Cost Issues

```bash
# Display common cost problems
cat << 'EOF'

Common Azure Cost Issues:
==========================

1. Zombie Resources:
   Problem: Unused resources still running
   Examples:
     - Stopped VMs with attached disks
     - Orphaned NICs and public IPs
     - Empty App Service plans
     - Unused storage accounts
   
   Solution:
     ✓ Regular resource audits
     ✓ Automation to detect unused
     ✓ Auto-shutdown policies
     ✓ Tag resources with expiry dates

2. Oversized Resources:
   Problem: Resources larger than needed
   Examples:
     - VM with 32GB when 8GB sufficient
     - Premium SSD for archival data
     - High DTU database under-utilized
   
   Solution:
     ✓ Monitor utilization metrics
     ✓ Right-size based on usage
     ✓ Use Azure Advisor
     ✓ Review quarterly

3. Missing Reserved Instances:
   Problem: Paying PAYG for steady workloads
   Impact: 40-60% cost increase
   
   Solution:
     ✓ Analyze usage patterns
     ✓ Purchase 1 or 3-year RIs
     ✓ Start with 1-year commitment
     ✓ Review RI coverage monthly

4. Data Transfer Costs:
   Problem: Unexpected egress charges
   Examples:
     - Cross-region replication
     - Public downloads
     - Inter-service traffic
   
   Solution:
     ✓ Keep resources in same region
     ✓ Use private endpoints
     ✓ Implement CDN
     ✓ Monitor bandwidth

5. Dev/Test in Production:
   Problem: Development using prod pricing
   Impact: Missing 40-60% discounts
   
   Solution:
     ✓ Use dev/test subscriptions
     ✓ Separate environments
     ✓ Tag appropriately
     ✓ Apply policies

6. No Budget Alerts:
   Problem: Overspend discovered too late
   Impact: Budget overruns
   
   Solution:
     ✓ Set budgets at multiple levels
     ✓ Configure alerts
     ✓ Monitor daily
     ✓ Automated responses

7. Inefficient Storage:
   Problem: Wrong storage tier
   Examples:
     - Hot tier for archives
     - No lifecycle policies
     - Redundant backups
   
   Solution:
     ✓ Use appropriate access tiers
     ✓ Lifecycle management
     ✓ Compression
     ✓ Deduplication

8. Shadow IT:
   Problem: Ungoverned resource creation
   Impact: Lost visibility and control
   
   Solution:
     ✓ Azure Policy enforcement
     ✓ Require tags
     ✓ Approval workflows
     ✓ Regular audits

EOF
```

---

## Step 24 – Show Cost Calculation Example

```bash
# Display cost calculation example
cat << 'EOF'

Cost Calculation Example:
==========================

Scenario: Web Application Infrastructure

Resources:
  - 2x D2s_v5 VMs (Web servers)
  - 1x D4s_v5 VM (Application server)
  - Azure SQL Database (4 vCore)
  - Standard Load Balancer
  - 500 GB Premium SSD
  - 1 TB Blob Storage (Hot tier)
  - Application Insights

Monthly Costs (Australia East, PAYG):

Compute:
  2x D2s_v5: $100 x 2      = $200
  1x D4s_v5: $200 x 1      = $200
                             -----
  Subtotal:                  $400

Database:
  SQL Database (4 vCore):    $800

Storage:
  Premium SSD (500 GB):      $80
  Blob Storage (1 TB):       $20
                             ----
  Subtotal:                  $100

Networking:
  Load Balancer:             $20
  Data processing (100GB):   $5
                             ---
  Subtotal:                  $25

Monitoring:
  App Insights (50 GB):      $140

================================
Total Monthly Cost (PAYG):   $1,465

With Optimizations:

1. Reserved Instances (3-year):
   Compute savings (60%):    -$240
   Database savings (60%):   -$480

2. Right-sizing:
   Reduce one D2s_v5 to B2s: -$60

3. Storage optimization:
   Move old blobs to Cool:   -$10

4. Monitor optimization:
   Reduce retention:         -$40

================================
Optimized Monthly Cost:      $635

Total Savings:               $830 (57%)
Annual Savings:              $9,960

ROI Analysis:
  3-year RI commitment:      $22,860
  3-year PAYG cost:          $52,740
  Total savings:             $29,880
  Break-even:                Day 1

EOF
```

---

## Step 25 – Show Cost Governance

```bash
# Display governance strategies
cat << 'EOF'

Cost Governance Framework:
===========================

Policy Enforcement:

1. Resource Naming:
   - Standardized naming convention
   - Include environment, project
   - Azure Policy validation

2. Required Tags:
   - Enforce at creation
   - Reject untagged resources
   - Inherited from resource group

3. Allowed Locations:
   - Limit to approved regions
   - Data residency compliance
   - Cost optimization

4. Allowed Resource Types:
   - Restrict to approved services
   - Prevent unauthorized resources
   - Control sprawl

5. SKU Restrictions:
   - Limit VM sizes
   - Prevent oversizing
   - Match environment tier

Approval Workflows:

1. Self-Service:
   - Pre-approved resources
   - Standard configurations
   - Automated provisioning

2. Manager Approval:
   - Medium-cost resources
   - Non-standard configurations
   - Email workflow

3. CAB Approval:
   - High-cost resources
   - Architecture changes
   - Formal review process

Cost Controls:

1. Resource Locks:
   - Prevent accidental deletion
   - Critical resources
   - Delete vs Read-only

2. Spending Limits:
   - Dev/Test subscriptions
   - Hard cap enforcement
   - Monthly reset

3. Quota Limits:
   - Regional quotas
   - VM cores
   - Prevent runaway costs

4. Auto-Shutdown:
   - Non-production VMs
   - Scheduled shutdown
   - Tag-based policies

Compliance:

1. Regular Audits:
   - Monthly resource review
   - Tag compliance
   - Policy violations

2. Cost Reviews:
   - Weekly anomaly checks
   - Monthly budget reviews
   - Quarterly optimization

3. Access Control:
   - RBAC for cost access
   - Separation of duties
   - Audit logs

4. Training:
   - Cost awareness
   - Best practices
   - Tool usage
   - FinOps culture

EOF
```

---

## Step 26 – Cleanup

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

You explored Azure Cost Management and Billing capabilities for cost analysis and reporting, created sample resources and applied cost allocation tags for department and project tracking, learned tagging strategies for accurate showback and chargeback reporting, reviewed cost breakdown by service types and typical Azure spending patterns, understood pricing models including pay-as-you-go, reserved instances, spot instances, and dev/test pricing, explored cost optimization strategies for compute, storage, networking, and databases with potential savings of 40-60%, examined FinOps principles and cloud financial operations best practices, learned showback versus chargeback models for internal cost allocation, identified common cost issues including zombie resources, oversized resources, and missing reserved instances, and established cost governance framework with policy enforcement, approval workflows, and compliance controls.

## Objectives
- Access Azure Cost Management and Billing
- Analyze current Azure spending
- Review cost breakdown by resource and service
- Create custom cost views
- Export cost data for reporting
- Identify cost optimization opportunities
- Understand cost allocation tags
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
RG_NAME="rg-lab16a-cost"
VNET_NAME="vnet-sample"
VM_NAME="vm-sample"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
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

## Step 3 – Show Cost Management Overview

```bash
# Display Cost Management capabilities
cat << 'EOF'

Azure Cost Management + Billing:
=================================

Key Features:
  - Cost analysis and reporting
  - Budget creation and alerts
  - Cost recommendations
  - Export scheduling
  - Cost allocation
  - Showback/chargeback reporting
  - Anomaly detection

Cost Analysis Views:
  - Accumulated costs
  - Daily costs
  - Forecast
  - Cost by service
  - Cost by resource
  - Cost by location
  - Cost by resource group
  - Cost by tag

Cost Dimensions:
  - Service name (e.g., Virtual Machines)
  - Resource location
  - Resource group
  - Resource type
  - Meter category
  - Publisher type
  - Pricing model
  - Reservation

Scope Levels:
  - Subscription
  - Resource group
  - Management group
  - Billing account
  - Department
  - Enrollment account

EOF
```

---

## Step 4 – Create Sample Resources

```bash
# Create VNet
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.0.0.0/16 \
  --subnet-name default \
  --subnet-prefix 10.0.1.0/24

echo "VNet created"
```

---

## Step 5 – Tag Resources

```bash
# Add tags to resource group for cost tracking
az group update \
  --name "$RG_NAME" \
  --tags \
    Environment=Production \
    Department=IT \
    CostCenter=CC-1001 \
    Project=CostAnalysis \
    Owner="finance@company.com"

echo "Tags applied"
```

---

## Step 6 – Show Tagging Strategy

```bash
# Display tagging best practices
cat << 'EOF'

Cost Allocation Tagging Strategy:
==================================

Essential Tags:

1. Environment:
   - Production
   - Staging
   - Development
   - Testing

2. Department/Team:
   - IT
   - Finance
   - Marketing
   - Engineering

3. Cost Center:
   - CC-1001 (IT Operations)
   - CC-2001 (Marketing)
   - CC-3001 (R&D)

4. Project:
   - Project name or code
   - Enables project-based reporting

5. Owner:
   - Email of responsible person
   - Contact for cost queries

6. Application:
   - App name or service
   - Group related resources

7. Compliance:
   - Regulatory requirements
   - Data classification

Tagging Best Practices:
  ✓ Define tagging policy
  ✓ Use Azure Policy to enforce tags
  ✓ Apply tags at resource group level
  ✓ Inherit tags where possible
  ✓ Regular tag audits
  ✓ Standardize tag names
  ✓ Document tag purposes

Benefit
