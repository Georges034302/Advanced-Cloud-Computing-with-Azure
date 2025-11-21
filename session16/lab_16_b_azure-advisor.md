# Lab 16.B: Azure Advisor

## Objectives
- Access Azure Advisor recommendations
- Review cost optimization opportunities
- Analyze security recommendations
- Implement performance improvements
- Review reliability suggestions
- Apply operational excellence best practices
- Configure Advisor alerts
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
RG_NAME="rg-lab16b-advisor"
VM_NAME="vm-unoptimized"
VNET_NAME="vnet-lab"

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

## Step 3 – Show Azure Advisor Overview

```bash
# Display Advisor capabilities
cat << 'EOF'

Azure Advisor:
==============

Overview:
  Azure Advisor is a personalized cloud consultant that helps you follow
  best practices to optimize your Azure deployments. It analyzes your
  resource configuration and usage telemetry and recommends solutions
  to improve cost effectiveness, performance, reliability, security, and
  operational excellence.

Five Pillars:

1. Cost:
   - Eliminate unused resources
   - Right-size underutilized resources
   - Reserved instances recommendations
   - Optimize data transfer
   - Reduce spending

2. Security:
   - Enable MFA
   - Restrict network access
   - Enable encryption
   - Apply security patches
   - Protect sensitive data

3. Reliability:
   - Improve application resilience
   - Configure backup
   - Implement redundancy
   - Set up disaster recovery
   - Monitor health

4. Performance:
   - Optimize application speed
   - Improve database performance
   - Reduce latency
   - Scale appropriately
   - Cache optimization

5. Operational Excellence:
   - Implement best practices
   - Improve deployment reliability
   - Monitor and diagnose
   - Automate operations
   - Service health awareness

How It Works:
  1. Continuous analysis of resources
  2. Machine learning algorithms
  3. Best practice comparison
  4. Personalized recommendations
  5. Impact and effort scoring
  6. One-click remediation (when available)

Recommendation Attributes:
  - Category (Cost, Security, etc.)
  - Impact (High, Medium, Low)
  - Affected resource
  - Recommended action
  - Potential benefit
  - Implementation effort

Access Methods:
  - Azure Portal (Advisor dashboard)
  - Azure CLI
  - PowerShell
  - REST API
  - Azure mobile app

EOF
```

---

## Step 4 – Create Unoptimized VM

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

# Create oversized VM (Standard_D8s_v3) without optimization
read -s -p "Enter VM admin password: " ADMIN_PASSWORD
echo

az vm create \
  --name "$VM_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --image Ubuntu2204 \
  --size Standard_D8s_v3 \
  --vnet-name "$VNET_NAME" \
  --subnet default \
  --admin-username azureuser \
  --admin-password "$ADMIN_PASSWORD" \
  --public-ip-sku Standard \
  --nsg-rule NONE

echo "Unoptimized VM created (oversized for light workload)"
```

---

## Step 5 – Show Cost Recommendations

```bash
# Display cost optimization patterns
cat << 'EOF'

Cost Optimization Recommendations:
===================================

Common Recommendations:

1. Shut Down or Resize Underutilized VMs:
   Detection:
     - CPU utilization < 5% for 7+ days
     - Network utilization < 10 MB/day
   
   Recommendation:
     - Shut down if not needed
     - Downsize to smaller SKU
   
   Potential Savings:
     - 20-60% of VM cost
   
   Example:
     "VM 'web-server' has been underutilized.
      Current: Standard_D4s_v3 ($175/mo)
      Recommended: Standard_B2s ($40/mo)
      Savings: $135/mo"

2. Delete or Deallocate Unused VMs:
   Detection:
     - VM stopped for 30+ days
     - No activity logged
   
   Recommendation:
     - Delete if no longer needed
     - Deallocate to save compute costs
   
   Note:
     - Deallocated VMs still incur storage costs
     - Complete deletion saves 100%

3. Purchase Reserved Instances:
   Detection:
     - Consistent usage pattern
     - VMs running 24/7
     - Predictable workload
   
   Recommendation:
     - 1-year or 3-year reservation
     - Save 40-60% vs pay-as-you-go
   
   Example:
     "You could save $12,000/year with reserved instances
      for VMs running in production environment"

4. Delete Unattached Managed Disks:
   Detection:
     - Disk not attached to any VM
     - Created > 30 days ago
   
   Potential Savings:
     - 100% of disk cost
   
   Example:
     "Premium SSD P30 (1TB) unattached
      Cost: $135/mo
      Action: Delete if not needed"

5. Delete or Archive Unused Public IP Addresses:
   Detection:
     - Public IP not associated
     - No traffic for 90 days
   
   Savings:
     - $3-4/mo per IP
   
   Note:
     - Small individual savings
     - Adds up with many IPs

6. Configure Storage Account Lifecycle:
   Detection:
     - Blobs in Hot tier > 90 days old
     - No access for extended period
   
   Recommendation:
     - Move to Cool tier (50% savings)
     - Move to Archive tier (80% savings)
   
   Example:
     "100 GB of data in Hot tier
      Older than 90 days
      Move to Cool: Save $1/mo per GB"

7. Right-Size Database Resources:
   Detection:
     - DTU or vCore utilization < 20%
     - Consistent low usage
   
   Recommendation:
     - Reduce tier or compute size
     - Consider serverless tier
   
   Savings:
     - 30-70% of database cost

8. Use Azure Hybrid Benefit:
   Detection:
     - Windows VMs or SQL databases
     - Eligible existing licenses
   
   Recommendation:
     - Apply existing Windows Server licenses
     - Apply SQL Server licenses
   
   Savings:
     - Up to 40% on Windows VMs
     - Up to 80% on SQL workloads

9. Optimize Application Gateway:
   Detection:
     - Underutilized capacity units
     - V1 SKU in use
   
   Recommendation:
     - Migrate to V2 with autoscaling
     - Right-size capacity
   
   Savings:
     - 20-40% with proper sizing

10. Consider Spot VMs:
    Detection:
      - Interruptible workloads
      - Batch processing
      - Dev/test environments
    
    Recommendation:
      - Use Spot VMs for suitable workloads
    
    Savings:
      - Up to 90% vs standard pricing

Implementation Priority:
  1. High impact, low effort (delete unused resources)
  2. Medium impact, medium effort (right-sizing)
  3. High impact, high effort (reserved instances)
  4. Low impact, low effort (quick wins)

EOF
```

---

## Step 6 – Show Security Recommendations

```bash
# Display security patterns
cat << 'EOF'

Security Recommendations:
==========================

Common Recommendations:

1. Enable Multi-Factor Authentication:
   Risk: High
   Detection:
     - User accounts without MFA
     - Admin accounts unprotected
   
   Recommendation:
     - Enable MFA for all users
     - Enforce for admins
   
   Implementation:
     - Azure AD MFA configuration
     - Conditional Access policies

2. Restrict Network Access:
   Risk: High
   Detection:
     - NSG rules allowing 0.0.0.0/0
     - RDP/SSH open to internet
     - Database public access
   
   Recommendation:
     - Limit source IP ranges
     - Use Just-In-Time access
     - Implement private endpoints
   
   Example:
     "NSG rule allows RDP from any source
      Recommendation: Restrict to corporate IPs"

3. Enable Disk Encryption:
   Risk: Medium
   Detection:
     - VM disks without Azure Disk Encryption
     - Unencrypted managed disks
   
   Recommendation:
     - Enable ADE for Windows/Linux VMs
     - Use customer-managed keys
   
   Benefit:
     - Protect data at rest
     - Compliance requirement

4. Enable Diagnostic Logging:
   Risk: Medium
   Detection:
     - Resources without diagnostics
     - No log retention configured
   
   Recommendation:
     - Send logs to Log Analytics
     - Configure retention
     - Enable key resource logs
   
   Critical Logs:
     - NSG flow logs
     - Azure AD sign-ins
     - Key Vault access
     - SQL audit logs

5. Install Security Updates:
   Risk: High
   Detection:
     - VMs with pending updates
     - Critical patches missing
   
   Recommendation:
     - Enable Update Management
     - Configure maintenance windows
     - Automate patch deployment

6. Enable Azure Defender:
   Risk: Medium
   Detection:
     - Subscriptions without Defender
     - Partial coverage
   
   Recommendation:
     - Enable Defender for Cloud
     - Cover all resource types
   
   Coverage:
     - Servers
     - App Service
     - Databases
     - Storage
     - Containers
     - Key Vault

7. Rotate Secrets and Keys:
   Risk: Medium
   Detection:
     - Keys/secrets older than 90 days
     - No rotation policy
   
   Recommendation:
     - Regular rotation schedule
     - Use Key Vault expiration
     - Automate rotation

8. Configure HTTPS Only:
   Risk: Medium
   Detection:
     - App Service allowing HTTP
     - Storage without secure transfer
   
   Recommendation:
     - Enforce HTTPS only
     - Disable HTTP traffic
   
   Benefit:
     - Encrypt data in transit
     - Prevent man-in-the-middle attacks

9. Enable Azure AD Integration:
   Risk: Low
   Detection:
     - SQL using SQL authentication
     - Resources with local accounts
   
   Recommendation:
     - Use Azure AD authentication
     - Disable local accounts
   
   Benefit:
     - Centralized identity
     - MFA support
     - Conditional Access

10. Implement Network Segmentation:
    Risk: Medium
    Detection:
      - All resources in same subnet
      - No NSG rules
    
    Recommendation:
      - Separate tiers (web, app, data)
      - Apply NSG rules
      - Use Application Security Groups

EOF
```

---

## Step 7 – Show Reliability Recommendations

```bash
# Display reliability patterns
cat << 'EOF'

Reliability Recommendations:
=============================

Common Recommendations:

1. Enable Backup for VMs:
   Detection:
     - VMs without backup configured
     - Critical workloads unprotected
   
   Recommendation:
     - Configure Azure Backup
     - Set retention policy
     - Test restore procedure
   
   RPO/RTO:
     - Daily backups (24-hour RPO)
     - Restore time varies by size

2. Use Availability Sets/Zones:
   Detection:
     - Single-instance VMs
     - No redundancy configured
   
   Recommendation:
     - Availability Set (99.95% SLA)
     - Availability Zone (99.99% SLA)
   
   Example:
     "VM 'app-server' has no redundancy
      Current SLA: 99.9%
      With AZ: 99.99%"

3. Configure Geo-Redundant Storage:
   Detection:
     - Critical data in LRS
     - No cross-region replication
   
   Recommendation:
     - Use GRS or GZRS
     - Enable read access (RA-GRS)
   
   Benefit:
     - Protect against regional outage
     - 16-nines durability

4. Enable Auto-Scaling:
   Detection:
     - Fixed capacity deployments
     - Manual scaling processes
   
   Recommendation:
     - VMSS autoscaling
     - App Service autoscaling
     - AKS cluster autoscaler
   
   Benefit:
     - Handle traffic spikes
     - Reduce manual intervention

5. Implement Health Probes:
   Detection:
     - Load balancers without health probes
     - No health monitoring
   
   Recommendation:
     - Configure HTTP/TCP probes
     - Set appropriate intervals
   
   Example:
     "Load balancer has no health probe
      Add HTTP probe: GET /health"

6. Use Traffic Manager for DR:
   Detection:
     - Single-region deployment
     - No failover capability
   
   Recommendation:
     - Deploy to secondary region
     - Configure Traffic Manager
     - Priority or weighted routing

7. Configure Service Health Alerts:
   Detection:
     - No service health monitoring
     - Unaware of platform issues
   
   Recommendation:
     - Set up Service Health alerts
     - Monitor resource health
     - Subscribe to maintenance notifications

8. Implement Database Failover:
   Detection:
     - Single-region databases
     - No HA configuration
   
   Recommendation:
     - SQL Database failover groups
     - Cosmos DB multi-region writes
     - Read replicas for read scaling

9. Enable Soft Delete:
   Detection:
     - Storage without soft delete
     - Key Vault without protection
   
   Recommendation:
     - Enable soft delete for blobs
     - Configure retention (7-365 days)
     - Enable Key Vault soft delete

10. Use Managed Disks:
    Detection:
      - VMs with unmanaged disks
      - Storage account limits
    
    Recommendation:
      - Migrate to managed disks
      - Benefit from platform management
    
    Advantage:
      - Higher reliability
      - Better SLA
      - Automatic placement

EOF
```

---

## Step 8 – Show Performance Recommendations

```bash
# Display performance patterns
cat << 'EOF'

Performance Recommendations:
============================

Common Recommendations:

1. Enable Azure CDN:
   Detection:
     - Static content served from origin
     - High latency for global users
   
   Recommendation:
     - Implement Azure CDN
     - Cache static assets
     - Reduce origin load
   
   Benefit:
     - 50-90% latency reduction
     - Offload origin traffic

2. Use Premium Storage for VMs:
   Detection:
     - I/O-intensive workloads on Standard HDD
     - High disk latency
   
   Recommendation:
     - Migrate to Premium SSD
     - Use Ultra Disks for extreme IOPS
   
   Performance Gain:
     - 10-100x IOPS improvement
     - <1ms latency

3. Optimize Database Performance:
   Detection:
     - DTU/vCore at 80%+ consistently
     - Long-running queries
     - Missing indexes
   
   Recommendation:
     - Increase compute tier
     - Add missing indexes
     - Query optimization
     - Enable read replicas

4. Configure Connection Pooling:
   Detection:
     - High connection churn
     - Connection timeouts
   
   Recommendation:
     - Implement connection pooling
     - Reuse connections
     - Set appropriate pool size

5. Enable Application Gateway Caching:
   Detection:
     - Dynamic content cacheable
     - Repeated backend requests
   
   Recommendation:
     - Configure response caching
     - Set cache duration
     - Use cache keys

6. Use Accelerated Networking:
   Detection:
     - Network-intensive VMs
     - High latency workloads
   
   Recommendation:
     - Enable accelerated networking
     - SR-IOV technology
   
   Benefit:
     - Up to 25 Gbps throughput
     - Lower latency
     - Lower CPU utilization

7. Implement Redis Cache:
   Detection:
     - Repeated database queries
     - Slow data access
   
   Recommendation:
     - Add Redis cache layer
     - Cache frequent queries
     - Session state storage

8. Configure App Service Always On:
   Detection:
     - App Service cold starts
     - First request slow
   
   Recommendation:
     - Enable Always On
     - Keep app loaded
   
   Trade-off:
     - Better performance
     - Slightly higher cost

9. Use Azure Front Door:
   Detection:
     - Global application
     - Multi-region deployment
   
   Recommendation:
     - Implement Front Door
     - Smart routing
     - Global load balancing
   
   Benefit:
     - Lowest latency routing
     - SSL offload
     - WAF integration

10. Optimize Cosmos DB:
    Detection:
      - High RU consumption
      - Hot partitions
    
    Recommendation:
      - Better partition key design
      - Increase RU/s for workload
      - Use regional replicas
      - Implement caching

EOF
```

---

## Step 9 – Show Operational Excellence

```bash
# Display operational best practices
cat << 'EOF'

Operational Excellence Recommendations:
========================================

Common Recommendations:

1. Implement Infrastructure as Code:
   Detection:
     - Manual resource creation
     - Configuration drift
   
   Recommendation:
     - Use ARM templates or Bicep
     - Terraform for multi-cloud
     - Version control templates
   
   Benefits:
     - Repeatable deployments
     - Reduced errors
     - Faster recovery

2. Enable Diagnostic Settings:
   Detection:
     - Resources without logging
     - No centralized monitoring
   
   Recommendation:
     - Send logs to Log Analytics
     - Archive to storage
     - Stream to Event Hub
   
   Key Metrics:
     - Performance counters
     - Error rates
     - Resource utilization

3. Configure Alert Rules:
   Detection:
     - No monitoring alerts
     - Reactive troubleshooting
   
   Recommendation:
     - Metric alerts (CPU, memory)
     - Log alerts (errors, exceptions)
     - Activity log alerts (changes)
   
   Alert Categories:
     - Availability
     - Performance
     - Security
     - Cost

4. Implement Application Insights:
   Detection:
     - Applications without APM
     - No distributed tracing
   
   Recommendation:
     - Enable Application Insights
     - Auto-instrumentation
     - Custom telemetry
   
   Visibility:
     - Request rates
     - Response times
     - Failure rates
     - Dependencies

5. Use Azure Automation:
   Detection:
     - Manual operational tasks
     - Repetitive processes
   
   Recommendation:
     - Automate routine tasks
     - Runbooks for common operations
     - Schedule automated jobs
   
   Use Cases:
     - Start/stop VMs
     - Patching
     - Backup verification
     - Cost reports

6. Implement Resource Tagging:
   Detection:
     - Resources without tags
     - Inconsistent naming
   
   Recommendation:
     - Define tagging policy
     - Enforce with Azure Policy
     - Standard tags (env, owner, cost center)

7. Configure Resource Locks:
   Detection:
     - Critical resources unprotected
     - Risk of accidental deletion
   
   Recommendation:
     - Read-Only lock for reference data
     - Delete lock for production resources
   
   Scope:
     - Resource level
     - Resource group level
     - Subscription level

8. Enable Azure Policy:
   Detection:
     - No governance enforcement
     - Compliance drift
   
   Recommendation:
     - Assign built-in policies
     - Custom policies for requirements
     - Regular compliance reviews
   
   Common Policies:
     - Allowed locations
     - Allowed VM SKUs
     - Require tags
     - Encryption enforcement

9. Implement Change Management:
   Detection:
     - Untracked configuration changes
     - No approval process
   
   Recommendation:
     - Use Change Tracking
     - Monitor configuration drift
     - Approval workflows
   
   Track:
     - Software changes
     - File modifications
     - Registry changes (Windows)
     - Service changes

10. Configure Update Management:
    Detection:
      - Inconsistent patching
      - Security vulnerabilities
    
    Recommendation:
      - Automated update deployment
      - Maintenance windows
      - Pre/post scripts
    
    Coverage:
      - Windows updates
      - Linux package updates
      - Third-party updates

Best Practices Summary:
  ✓ Automate everything possible
  ✓ Monitor and alert proactively
  ✓ Implement governance
  ✓ Document procedures
  ✓ Regular reviews and audits
  ✓ Continuous improvement
  ✓ Test disaster recovery

EOF
```

---

## Step 10 – Check Advisor Recommendations

```bash
# List all Advisor recommendations
az advisor recommendation list \
  --query "[].{Category:category, Impact:impact, Resource:impactedField, Problem:shortDescription.problem}" \
  --output table
```

---

## Step 11 – Get Cost Recommendations

```bash
# Get cost-specific recommendations
az advisor recommendation list \
  --category Cost \
  --query "[].{Impact:impact, Resource:impactedValue, Problem:shortDescription.problem, Solution:shortDescription.solution}" \
  --output table
```

---

## Step 12 – Get Security Recommendations

```bash
# Get security-specific recommendations
az advisor recommendation list \
  --category Security \
  --query "[].{Impact:impact, Resource:impactedValue, Problem:shortDescription.problem}" \
  --output table
```

---

## Step 13 – Show Advisor Configuration

```bash
# Display Advisor configuration options
cat << 'EOF'

Advisor Configuration:
======================

Configuration Options:

1. Recommendation Filters:
   - Subscription scope
   - Resource group scope
   - Resource type filter
   - Impact level (High, Medium, Low)
   - Category filter

2. CPU Threshold (Low Utilization):
   - Default: 5% average over 7 days
   - Configurable: 5%, 10%, 15%, 20%
   - Affects VM right-sizing recommendations

3. Network Threshold:
   - Default: 7 days of low network activity
   - Triggers shutdown recommendations

4. Exclusions:
   - Dismiss specific recommendations
   - Postpone for later review
   - Exclude specific resources

5. Notification Settings:
   - Email alerts for new recommendations
   - Digest frequency (daily, weekly)
   - Recipient list

Configuration via CLI:

  # Configure CPU threshold for resource group
  az advisor configuration update \
    --low-cpu-threshold 10 \
    --resource-group myResourceGroup

  # Suppress recommendation
  az advisor recommendation suppress \
    --recommendation-id <id> \
    --duration 30

  # List suppressions
  az advisor recommendation list-suppressions

EOF
```

---

## Step 14 – Show Recommendation Workflow

```bash
# Display how to act on recommendations
cat << 'EOF'

Recommendation Action Workflow:
================================

Step 1: Review Recommendations
  - Access Azure Advisor dashboard
  - Filter by category and impact
  - Prioritize High impact items
  - Review each recommendation details

Step 2: Assess Impact
  - Understand affected resources
  - Estimate effort required
  - Calculate potential savings/benefit
  - Check dependencies

Step 3: Plan Implementation
  - Schedule changes
  - Assign ownership
  - Prepare rollback plan
  - Document changes

Step 4: Implement
  - Follow recommended actions
  - Use "Quick Fix" when available
  - Test in non-production first
  - Monitor during implementation

Step 5: Validate
  - Verify desired outcome achieved
  - Check for side effects
  - Measure improvement
  - Update documentation

Step 6: Track Progress
  - Mark recommendation as addressed
  - Monitor for recurring issues
  - Update policies to prevent recurrence
  - Report savings achieved

Quick Fix Scenarios:
  ✓ Delete unused resources (one-click)
  ✓ Resize VMs (with approval)
  ✓ Purchase reserved instances
  ✓ Enable diagnostic logging
  ✓ Configure backup

Manual Implementation:
  - Security hardening
  - Architecture changes
  - Application modifications
  - Complex migrations

Recommendation States:
  - Active: Needs attention
  - Postponed: Defer to later
  - Dismissed: Not applicable
  - Resolved: Action taken

Best Practices:
  ✓ Review Advisor weekly
  ✓ Act on High impact first
  ✓ Track savings achieved
  ✓ Automate common fixes
  ✓ Regular team reviews
  ✓ Integrate with change management

EOF
```

---

## Step 15 – Show Impact Assessment

```bash
# Display how to evaluate recommendation impact
cat << 'EOF'

Recommendation Impact Assessment:
==================================

Impact Levels:

High Impact:
  Characteristics:
    - Significant cost savings (>$1000/month)
    - Critical security vulnerability
    - Major performance improvement
    - High availability risk
  
  Examples:
    - Reserved instance savings $5000/year
    - Internet-exposed database
    - Database at 95% DTU consistently
    - No backup for production VMs
  
  Priority: Immediate action
  Timeline: Within 1 week

Medium Impact:
  Characteristics:
    - Moderate cost savings ($100-1000/month)
    - Important security best practice
    - Noticeable performance gain
    - Reliability improvement
  
  Examples:
    - Underutilized VM (save $50/month)
    - Missing MFA on admin accounts
    - Enable CDN for static content
    - Configure availability set
  
  Priority: Schedule within sprint
  Timeline: Within 1 month

Low Impact:
  Characteristics:
    - Minor cost savings (<$100/month)
    - Security hardening
    - Incremental improvement
    - Nice-to-have optimization
  
  Examples:
    - Delete unused public IP ($4/month)
    - Enable soft delete on storage
    - Optimize cache TTL
    - Update SDK version
  
  Priority: Backlog
  Timeline: Next quarter

Effort Estimation:

Low Effort (< 1 hour):
  - Delete resources
  - Enable settings
  - One-click fixes
  - Configuration changes

Medium Effort (1-8 hours):
  - Resize resources
  - Implement caching
  - Configure monitoring
  - Simple automation

High Effort (1-5 days):
  - Architecture changes
  - Application modifications
  - Multi-resource migrations
  - Complex implementations

Impact vs Effort Matrix:

  High Impact + Low Effort   = DO NOW (Quick wins)
  High Impact + High Effort  = PLAN CAREFULLY (Major projects)
  Low Impact + Low Effort    = FILL-IN WORK (When time permits)
  Low Impact + High Effort   = RECONSIDER (May not be worth it)

Cost-Benefit Analysis:

Example 1 - Right-size VM:
  Current cost: $200/month
  New cost: $100/month
  Monthly savings: $100
  Annual savings: $1,200
  Effort: 1 hour
  Risk: Low (can easily revert)
  Decision: Implement immediately

Example 2 - Reserved Instances:
  Current cost: $500/month (PAYG)
  RI cost: $300/month (3-year)
  Monthly savings: $200
  Annual savings: $2,400
  Upfront: $10,800 (3 years)
  Effort: 2 hours
  Risk: Medium (commitment)
  Decision: Analyze usage pattern, then proceed

Example 3 - Delete unused disk:
  Current cost: $10/month
  New cost: $0
  Monthly savings: $10
  Annual savings: $120
  Effort: 15 minutes
  Risk: Very low (if truly unused)
  Decision: Verify not needed, then delete

Recommendation Scoring:

Financial Value:
  - Monthly savings amount
  - Annual savings
  - ROI calculation

Technical Value:
  - Security improvement
  - Performance gain
  - Reliability increase
  - Operational efficiency

Strategic Value:
  - Compliance requirement
  - Business enablement
  - Competitive advantage
  - Innovation potential

EOF
```

---

## Step 16 – Configure Advisor Alerts

```bash
# Show alert configuration
cat << 'EOF'

Configuring Advisor Alerts:
===========================

Alert Types:

1. New Recommendations:
   - Triggered: When new recommendations appear
   - Frequency: As they occur
   - Action: Email notification

2. Recommendation Digest:
   - Triggered: Scheduled
   - Frequency: Daily or weekly
   - Content: Summary of all active recommendations

Configuration Steps:

Via Portal:
  1. Open Azure Advisor
  2. Go to "Monitoring" > "Alerts"
  3. Click "+ New alert rule"
  4. Select scope (subscription/resource group)
  5. Choose condition (new recommendation)
  6. Select action group (email/SMS)
  7. Provide alert name and severity
  8. Create alert rule

Via CLI:

# Example (conceptual - actual implementation via portal or ARM)
  az monitor action-group create \
    --name advisor-alerts \
    --resource-group monitoring-rg \
    --short-name adv-alert \
    --email-receiver admin email@company.com

Action Group Configuration:
  - Email recipients
  - SMS recipients
  - Voice call
  - Azure Function
  - Logic App
  - Webhook
  - ITSM connector

Alert Conditions:
  - Category = Cost
  - Impact = High
  - Specific resource groups
  - Specific recommendation types

Notification Examples:

Subject: [Azure Advisor] New High Impact Cost Recommendation
Body:
  Resource: vm-production-app
  Category: Cost
  Impact: High
  Problem: Underutilized VM
  Potential Savings: $150/month
  Recommended Action: Resize to Standard_B2s
  
  View in Portal: [Link]

Weekly Digest:
  Summary: 12 active recommendations
  New this week: 3
  - High Impact: 1 (Cost - Unused disk)
  - Medium Impact: 2 (Security - Enable MFA)
  - Low Impact: 0
  
  Potential Savings: $2,500/month

Best Practices:
  ✓ Set up alerts for High impact only
  ✓ Weekly digest for team review
  ✓ Include relevant stakeholders
  ✓ Separate alerts by category
  ✓ Test alert delivery
  ✓ Review and tune thresholds

EOF
```

---

## Step 17 – Integration with Azure Policy

```bash
# Display Advisor and Policy integration
cat << 'EOF'

Advisor + Azure Policy Integration:
====================================

Preventive vs Detective:

Advisor (Detective):
  - Analyzes existing resources
  - Identifies optimization opportunities
  - Recommends improvements
  - Reactive approach

Azure Policy (Preventive):
  - Prevents non-compliant resources
  - Enforces standards at creation
  - Blocks violations
  - Proactive approach

Combined Strategy:

1. Use Advisor to Identify Patterns:
   Example:
     Advisor finds: 20 VMs without backup
   
   Response:
     Create Policy: "Require backup on all production VMs"
     Tag production VMs: Environment=Production
     Policy auto-configures backup on new VMs

2. Policy Based on Recommendations:
   
   Cost Example:
     Advisor: "Many VMs using expensive SKUs"
     Policy: "Restrict VM SKUs to approved list"
     
   Security Example:
     Advisor: "Public IPs on database VMs"
     Policy: "Deny public IP on VMs tagged Database"
   
   Reliability Example:
     Advisor: "Single-instance VMs"
     Policy: "Require availability set or zone"

3. Automated Remediation:
   
   Advisor Finds Issue:
     - Unencrypted storage account
   
   Policy Remediates:
     - DeployIfNotExists policy
     - Enables encryption automatically
     - Brings to compliance

Implementation Workflow:

Step 1: Review Advisor recommendations
Step 2: Identify recurring patterns
Step 3: Create Azure Policy to prevent
Step 4: Assign policy to scope
Step 5: Monitor compliance
Step 6: Advisor validates effectiveness

Example Policies from Advisor:

1. Cost Optimization:
   Policy: "Allowed VM SKUs"
   Prevents: Oversized VMs
   Based on: VM right-sizing recommendations

2. Security:
   Policy: "Require HTTPS for storage"
   Prevents: Insecure data transfer
   Based on: Security recommendations

3. Reliability:
   Policy: "Require backup for VMs"
   Prevents: Unprotected resources
   Based on: Backup recommendations

4. Operational:
   Policy: "Require specific tags"
   Prevents: Untagged resources
   Based on: Resource organization recommendations

Benefits:
  ✓ Shift left (prevent issues)
  ✓ Reduce recurring recommendations
  ✓ Enforce best practices
  ✓ Lower operational overhead
  ✓ Consistent compliance

EOF
```

---

## Step 18 – Show Advisor Scores

```bash
# Display Advisor score concept
cat << 'EOF'

Azure Advisor Score:
====================

Overview:
  Advisor Score represents how well you're following Azure best
  practices. It's calculated based on the percentage of resources
  that have no active Advisor recommendations.

Score Calculation:

  Score = (Compliant Resources / Total Resources) × 100

  For each pillar:
    - Cost Score: 0-100%
    - Security Score: 0-100%
    - Reliability Score: 0-100%
    - Performance Score: 0-100%
    - Operational Excellence Score: 0-100%

  Overall Score: Average of all five pillars

Score Breakdown Example:

  Overall Advisor Score: 72%

  Cost:                  65%
    ✓ 20 resources optimized
    ✗ 11 resources with recommendations
    Issues: Underutilized VMs, unused disks

  Security:              85%
    ✓ 28 resources secure
    ✗ 5 resources with recommendations
    Issues: MFA not enabled, public access

  Reliability:           70%
    ✓ 18 resources resilient
    ✗ 8 resources with recommendations
    Issues: No backup, single instance

  Performance:           75%
    ✓ 22 resources optimized
    ✗ 7 resources with recommendations
    Issues: Standard HDD for database

  Operational Excellence: 68%
    ✓ 19 resources following practices
    ✗ 9 resources with recommendations
    Issues: No logging, manual processes

Score Trends:
  - Track over time
  - Month-over-month comparison
  - Goal: Steady improvement
  - Target: 80%+ overall

Score by Subscription:
  Production:    85% ⬆ (target: 90%)
  Staging:       75% ⬌ (target: 80%)
  Development:   60% ⬇ (target: 70%)

Score by Resource Group:
  rg-prod-web:   90% ✓ Excellent
  rg-prod-data:  85% ✓ Good
  rg-dev-test:   55% ⚠ Needs attention

Improving Your Score:

1. Address High Impact Recommendations:
   - Biggest score improvement
   - Most critical issues
   - Quick wins

2. Systematic Approach:
   - One category at a time
   - Start with easiest fixes
   - Build momentum

3. Prevent New Issues:
   - Implement Azure Policy
   - Enforce standards
   - Automate compliance

4. Regular Reviews:
   - Weekly score check
   - Monthly deep dive
   - Quarterly strategy

Score Reporting:

Dashboard View:
  - Overall score gauge
  - Trend chart (30/60/90 days)
  - Score by category
  - Top recommendations

Executive Summary:
  "Advisor Score improved from 65% to 72% this month
   by implementing 15 high-impact recommendations,
   resulting in $3,200/month savings and improved
   security posture."

Team KPIs:
  - Target score: 80%
  - Current score: 72%
  - Monthly improvement: +7%
  - Recommendations addressed: 15/32
  - Estimated savings realized: $3,200/month

Gamification:
  - Team score competitions
  - Recognize improvements
  - Celebrate milestones
  - Share success stories

EOF
```

---

## Step 19 – Show Advisor API Usage

```bash
# Display API usage examples
cat << 'EOF'

Advisor REST API Usage:
=======================

API Endpoints:

1. List Recommendations:
   GET https://management.azure.com/subscriptions/{subscriptionId}
       /providers/Microsoft.Advisor/recommendations?api-version=2020-01-01

2. Get Recommendation Detail:
   GET https://management.azure.com/{resourceUri}
       /providers/Microsoft.Advisor/recommendations/{recommendationId}
       ?api-version=2020-01-01

3. Suppress Recommendation:
   PUT https://management.azure.com/{resourceUri}
       /providers/Microsoft.Advisor/recommendations/{recommendationId}
       /suppressions/{name}?api-version=2020-01-01

4. Generate Recommendations:
   POST https://management.azure.com/subscriptions/{subscriptionId}
        /providers/Microsoft.Advisor/generateRecommendations
        ?api-version=2020-01-01

5. Get Advisor Score:
   GET https://management.azure.com/subscriptions/{subscriptionId}
       /providers/Microsoft.Advisor/advisorScore?api-version=2020-01-01

Response Example:

```json
{
  "value": [
    {
      "id": "/subscriptions/.../recommendations/abc123",
      "type": "Microsoft.Advisor/recommendations",
      "name": "abc123",
      "properties": {
        "category": "Cost",
        "impact": "High",
        "impactedField": "Microsoft.Compute/virtualMachines",
        "impactedValue": "vm-production",
        "lastUpdated": "2024-01-15T10:30:00Z",
        "shortDescription": {
          "problem": "Virtual machine is underutilized",
          "solution": "Resize to smaller SKU"
        },
        "extendedProperties": {
          "currentSku": "Standard_D8s_v3",
          "targetSku": "Standard_D4s_v3",
          "savingsAmount": "150",
          "savingsCurrency": "USD"
        }
      }
    }
  ]
}
```

Automation Examples:

1. Daily Cost Report (PowerShell):
```powershell
# Get all cost recommendations
$recommendations = Get-AzAdvisorRecommendation -Category Cost

# Filter high impact
$highImpact = $recommendations | Where-Object {$_.Impact -eq "High"}

# Calculate total potential savings
$totalSavings = ($highImpact | Measure-Object -Property SavingsAmount -Sum).Sum

# Send email report
Send-MailMessage -Subject "Daily Advisor Cost Report" -Body "Potential savings: $$totalSavings"
```

2. Auto-Suppress Development Resources (Python):
```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.advisor import AdvisorManagementClient

credential = DefaultAzureCredential()
advisor_client = AdvisorManagementClient(credential, subscription_id)

# Get recommendations
recommendations = advisor_client.recommendations.list()

# Suppress dev environment recommendations
for rec in recommendations:
    if 'dev' in rec.impacted_value.lower():
        advisor_client.suppressions.create(
            resource_uri=rec.id,
            recommendation_id=rec.name,
            name='dev-suppression',
            ttl='30.00:00:00'  # 30 days
        )
```

3. Track Score Over Time (Bash):
```bash
#!/bin/bash
# Daily Advisor score tracker

SUBSCRIPTION_ID=$(az account show --query id -o tsv)
DATE=$(date +%Y-%m-%d)

# Get current score (via portal or custom implementation)
SCORE=$(az advisor recommendation list \
  --query "length([?impact=='High'])" -o tsv)

# Log to file
echo "$DATE,$SCORE" >> advisor-score-history.csv

# Alert if score drops
if [ $SCORE -lt 70 ]; then
  echo "Advisor score below threshold!" | mail -s "Advisor Alert" admin@company.com
fi
```

Integration Scenarios:

1. CI/CD Pipeline:
   - Check Advisor score before deployment
   - Fail build if critical recommendations
   - Automated compliance gates

2. ServiceNow Integration:
   - Create tickets from recommendations
   - Track remediation
   - Close loop automation

3. Slack/Teams Notifications:
   - New high-impact recommendations
   - Daily digest
   - Score changes

4. Custom Dashboards:
   - Power BI integration
   - Grafana visualization
   - Real-time monitoring

EOF
```

---

## Step 20 – Show Best Practices Summary

```bash
# Display comprehensive best practices
cat << 'EOF'

Advisor Best Practices Summary:
================================

Regular Review Cadence:

Daily:
  ✓ Check for new High impact recommendations
  ✓ Review anomaly alerts
  ✓ Monitor Advisor score changes

Weekly:
  ✓ Team review of all active recommendations
  ✓ Prioritize upcoming work
  ✓ Track remediation progress
  ✓ Update suppressions if needed

Monthly:
  ✓ Deep dive into each category
  ✓ Analyze trends
  ✓ Report savings achieved
  ✓ Update policies based on patterns
  ✓ Review and adjust configurations

Quarterly:
  ✓ Strategic review
  ✓ Set improvement targets
  ✓ Evaluate tool effectiveness
  ✓ Training and knowledge sharing
  ✓ Budget planning with projected savings

Organizational Structure:

1. Cloud Governance Team:
   - Sets policies based on Advisor
   - Defines standards
   - Monitors compliance

2. FinOps Team:
   - Acts on cost recommendations
   - Tracks savings
   - Manages reserved instances
   - Chargeback/showback

3. Security Team:
   - Implements security recommendations
   - Validates fixes
   - Coordinates with Advisor for Security

4. Engineering Teams:
   - Own their resources
   - Implement technical fixes
   - Optimize applications
   - Follow best practices

5. Operations Team:
   - Reliability and performance
   - Monitoring and alerting
   - Automated remediation
   - Operational excellence

Success Metrics:

Financial:
  - Monthly savings achieved
  - Cost avoidance
  - RI coverage percentage
  - Cost per resource reduction

Technical:
  - Recommendations addressed/month
  - Mean time to remediation
  - Recurring issues eliminated
  - Automated fixes implemented

Compliance:
  - Advisor score improvement
  - Policy compliance rate
  - Security posture
  - SLA achievement

Operational:
  - Manual processes automated
  - Incident reduction
  - Performance improvements
  - Resource optimization

Maturity Model:

Level 1 - Reactive:
  - Occasional Advisor checks
  - No systematic approach
  - Manual remediation
  - Low score (<50%)

Level 2 - Managed:
  - Regular reviews
  - Documented process
  - Some automation
  - Improving score (50-70%)

Level 3 - Defined:
  - Weekly cadence
  - Cross-team coordination
  - Policy integration
  - Good score (70-85%)

Level 4 - Optimized:
  - Automated workflows
  - Preventive policies
  - Continuous improvement
  - Excellent score (85%+)

Level 5 - Innovating:
  - Proactive optimization
  - Custom automation
  - Best-in-class practices
  - Score >90%, industry leading

Common Pitfalls to Avoid:

✗ Ignoring Low impact recommendations
  - They accumulate over time
  - Set aside time for quick wins

✗ No ownership assignment
  - Recommendations linger
  - Assign clear responsibility

✗ Analysis paralysis
  - Overthinking simple fixes
  - Start with easy wins

✗ Suppressing without investigation
  - May hide real issues
  - Document suppression reasons

✗ One-time cleanup only
  - Requires ongoing effort
  - Build into regular workflow

✗ Not measuring outcomes
  - Can't prove value
  - Track savings and improvements

✗ Siloed implementation
  - Requires cross-team collaboration
  - Regular sync meetings

Success Factors:

✓ Executive sponsorship
✓ Clear ownership model
✓ Regular review cadence
✓ Automation where possible
✓ Integration with existing processes
✓ Measurement and reporting
✓ Continuous improvement mindset
✓ Training and enablement

Sample 30-Day Plan:

Week 1:
  - Assess current state
  - Review all recommendations
  - Categorize and prioritize
  - Assign owners

Week 2:
  - Implement quick wins
  - Document procedures
  - Set up alerts
  - Begin tracking

Week 3:
  - Address High impact items
  - Create remediation policies
  - Automate common fixes
  - Report initial savings

Week 4:
  - Review progress
  - Refine processes
  - Plan next priorities
  - Celebrate wins

EOF
```

---

## Step 21 – List VM Recommendations

```bash
# Get recommendations for the created VM
az advisor recommendation list \
  --resource-group "$RG_NAME" \
  --query "[?contains(impactedValue, '$VM_NAME')].{Category:category, Impact:impact, Problem:shortDescription.problem, Solution:shortDescription.solution}" \
  --output table
```

---

## Step 22 – Show VM Metrics

```bash
# Display VM utilization (would show low usage after running for a while)
cat << EOF

VM Monitoring:
==============

To view actual metrics:
  1. Navigate to Azure Portal
  2. Go to Virtual Machines > $VM_NAME
  3. Select "Metrics"
  4. Add metrics:
     - Percentage CPU
     - Network In Total
     - Network Out Total
     - Disk Read Bytes
     - Disk Write Bytes

Expected Advisor Recommendation:
  "This VM ($VM_NAME) is underutilized:
   - Current SKU: Standard_D8s_v3 (8 vCPUs, 32 GB RAM)
   - CPU usage: < 5%
   - Cost: ~$280/month
   
   Recommended: Standard_B2s (2 vCPUs, 4 GB RAM)
   Savings: ~$240/month"

Note: Recommendations may take 24-48 hours to appear after resource creation.

EOF
```

---

## Step 23 – Cleanup

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

You explored Azure Advisor as a personalized cloud consultant providing recommendations across five pillars including cost, security, reliability, performance, and operational excellence, created an intentionally oversized VM to demonstrate how Advisor detects underutilized resources and suggests right-sizing with potential savings of $240 per month, reviewed comprehensive cost optimization recommendations including reserved instances for 40-60% savings, spot VMs for up to 90% savings, and eliminating unused resources, examined security recommendations for MFA enablement, network access restrictions, disk encryption, diagnostic logging, and Azure Defender coverage, learned reliability best practices including backup configuration, availability sets and zones for higher SLA, geo-redundant storage, auto-scaling, and health probes, explored performance recommendations for Azure CDN, premium storage, accelerated networking, caching strategies, and database optimization, understood operational excellence guidance for infrastructure as code, diagnostic settings, alert rules, Application Insights, automation, tagging policies, and Azure Policy enforcement, configured Advisor alerts for new high-impact recommendations with email notifications and weekly digests, reviewed Advisor Score calculation showing compliance percentage across all pillars with trending and improvement tracking, explored integration between Advisor and Azure Policy to shift from reactive recommendations to proactive prevention of issues, learned API usage for programmatic access to recommendations enabling automation workflows, custom dashboards, and CI/CD integration, and established best practices including daily checks for high-impact items, weekly team reviews, monthly deep dives, quarterly strategic planning, and a four-level maturity model from reactive to optimized operations.", "oldString": "# Lab 16b: Optimize Azure Environment with Azure Advisor
