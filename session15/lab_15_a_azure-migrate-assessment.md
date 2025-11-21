# Lab 15.A: Azure Migrate Assessment

## Objectives
- Create Azure Migrate project
- Set up migration assessment tools
- Assess on-premises workload readiness
- Review sizing recommendations
- Analyze cost estimates
- Identify migration blockers
- Export assessment reports
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
RG_NAME="rg-lab15a-migrate"
MIGRATE_PROJECT="migrate-project-$RANDOM"
ASSESSMENT_NAME="assessment-workloads"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$MIGRATE_PROJECT"
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

## Step 3 – Create Azure Migrate Project

```bash
# Create Azure Migrate project
az migrate project create \
  --name "$MIGRATE_PROJECT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION"

echo "Azure Migrate project created"
```

---

## Step 4 – Show Azure Migrate Overview

```bash
# Display Azure Migrate capabilities
cat << 'EOF'

Azure Migrate Overview:
=======================

Discovery and Assessment:
  - VMware VMs (vSphere environments)
  - Hyper-V VMs (Windows Server Hyper-V)
  - Physical servers (Windows and Linux)
  - AWS EC2 instances
  - GCP VM instances

Assessment Types:
  - Azure VM assessment
  - Azure SQL assessment
  - Azure App Service assessment
  - Azure VMware Solution (AVS) assessment

Key Capabilities:
  - Agentless discovery
  - Performance-based sizing
  - Cost estimation
  - Dependency mapping
  - Readiness analysis
  - Migration planning

Tools:
  - Azure Migrate: Discovery and assessment
  - Azure Migrate: Server Migration
  - Data Migration Assistant (DMA)
  - Database Migration Service (DMS)
  - Web app migration assistant

EOF
```

---

## Step 5 – Show Assessment Workflow

```bash
# Display assessment workflow
cat << 'EOF'

Azure Migrate Assessment Workflow:
===================================

Phase 1: Discover
  1. Download Azure Migrate appliance
  2. Deploy appliance (VMware/Hyper-V/Physical)
  3. Configure appliance credentials
  4. Start continuous discovery
  5. Wait for inventory collection

Phase 2: Assess
  1. Create assessment
  2. Select discovered servers
  3. Configure assessment properties:
     - Target location
     - VM sizing criteria
     - Pricing tier
     - Reserved instances
     - Comfort factor
  4. Run assessment
  5. Review results

Phase 3: Plan
  1. Review readiness status
  2. Analyze cost estimates
  3. Identify dependencies
  4. Group workloads
  5. Plan migration waves

Phase 4: Migrate
  1. Test migration
  2. Production migration
  3. Post-migration optimization

EOF
```

---

## Step 6 – Register Resource Providers

```bash
# Register required resource providers
az provider register --namespace Microsoft.Migrate
az provider register --namespace Microsoft.OffAzure

# Wait for registration
sleep 30

# Verify registration
az provider show \
  --namespace Microsoft.Migrate \
  --query "registrationState" \
  --output tsv

echo "Resource providers registered"
```

---

## Step 7 – Show Appliance Requirements

```bash
# Display appliance requirements
cat << 'EOF'

Azure Migrate Appliance Requirements:
======================================

VMware Environment:
  - vCenter Server 6.7, 7.0, or 8.0
  - ESXi host 5.5 or later
  - Appliance VM: 32 GB RAM, 8 vCPUs, 80 GB disk

Hyper-V Environment:
  - Windows Server 2019, 2016, or 2012 R2
  - Hyper-V role enabled
  - Appliance VM: 16 GB RAM, 8 vCPUs, 80 GB disk

Physical Servers:
  - Windows Server 2012 or later
  - Linux: RHEL 6+, Ubuntu 14.04+, CentOS 6+
  - Appliance: 16 GB RAM, 8 vCPUs, 80 GB disk

Network Requirements:
  - Internet access (direct or proxy)
  - Access to Azure endpoints
  - Access to vCenter/Hyper-V hosts
  - Port 443 (HTTPS) outbound

Permissions:
  - vCenter: Read-only account
  - Hyper-V: Administrator on hosts
  - Physical: Root/Administrator access

EOF
```

---

## Step 8 – Create Sample Assessment Data

```bash
# Create sample on-premises inventory
cat > sample-inventory.json << 'EOF'
{
  "servers": [
    {
      "name": "web-server-01",
      "os": "Windows Server 2019",
      "cores": 4,
      "memory_gb": 16,
      "disk_gb": 500,
      "workload": "IIS Web Server"
    },
    {
      "name": "web-server-02",
      "os": "Windows Server 2019",
      "cores": 4,
      "memory_gb": 16,
      "disk_gb": 500,
      "workload": "IIS Web Server"
    },
    {
      "name": "app-server-01",
      "os": "Windows Server 2022",
      "cores": 8,
      "memory_gb": 32,
      "disk_gb": 1024,
      "workload": "Application Server"
    },
    {
      "name": "db-server-01",
      "os": "Windows Server 2019",
      "cores": 16,
      "memory_gb": 64,
      "disk_gb": 2048,
      "workload": "SQL Server 2019"
    },
    {
      "name": "linux-web-01",
      "os": "Ubuntu 20.04",
      "cores": 2,
      "memory_gb": 8,
      "disk_gb": 200,
      "workload": "Apache Web Server"
    },
    {
      "name": "file-server-01",
      "os": "Windows Server 2016",
      "cores": 2,
      "memory_gb": 8,
      "disk_gb": 4096,
      "workload": "File Server"
    }
  ]
}
EOF

echo "Sample inventory created"
```

---

## Step 9 – Show Assessment Properties

```bash
# Display assessment properties
cat << 'EOF'

Assessment Properties Configuration:
=====================================

Target Settings:
  - Target location: Azure region
  - Storage type: Standard HDD, Standard SSD, Premium SSD
  - Reserved instances: None, 1 year, 3 year
  - Offer/Licensing: Pay-as-you-go, Dev/Test, Enterprise Agreement

Sizing Criteria:
  - As on-premises: Use current allocation
  - Performance-based: Use performance data
  - Performance history: 1 day to 1 month
  - Percentile utilization: 50th, 95th, 99th

VM Series:
  - All series (recommended)
  - Specific series (D, E, F, etc.)

Comfort Factor:
  - 1.0x (no buffer)
  - 1.3x (default - 30% buffer)
  - 2.0x (100% buffer)

Pricing:
  - Currency: USD, AUD, EUR, etc.
  - Discount percentage
  - VM uptime (hours/month)
  - Azure Hybrid Benefit

EOF
```

---

## Step 10 – Analyze Sample Workloads

```bash
# Display analysis of sample inventory
cat << 'EOF'

Sample Workload Analysis:
=========================

Web Servers (2):
  - Current: 4 cores, 16 GB RAM, 500 GB disk each
  - Azure recommendation: Standard_D4s_v5
  - Features: Load balanced, stateless
  - Migration: Azure VM or App Service

Application Server (1):
  - Current: 8 cores, 32 GB RAM, 1 TB disk
  - Azure recommendation: Standard_E8s_v5
  - Features: Business logic tier
  - Migration: Azure VM

Database Server (1):
  - Current: 16 cores, 64 GB RAM, 2 TB disk
  - Azure recommendation: Azure SQL MI or Standard_E16s_v5
  - Features: SQL Server 2019
  - Migration: DMS to Azure SQL or VM

Linux Web Server (1):
  - Current: 2 cores, 8 GB RAM, 200 GB disk
  - Azure recommendation: Standard_D2s_v5
  - Features: Apache web server
  - Migration: Azure VM or App Service

File Server (1):
  - Current: 2 cores, 8 GB RAM, 4 TB disk
  - Azure recommendation: Azure Files
  - Features: SMB file shares
  - Migration: Storage Migration Service

Total: 6 servers, 36 cores, 144 GB RAM, 8.3 TB storage

EOF
```

---

## Step 11 – Show Sizing Recommendations

```bash
# Display VM sizing recommendations
cat << 'EOF'

Azure VM Sizing Recommendations:
=================================

Server              Current     Azure VM          Monthly Cost (Est)
--------------------------------------------------------------------------------
web-server-01       4C/16GB     Standard_D4s_v5   ~$175 AUD
web-server-02       4C/16GB     Standard_D4s_v5   ~$175 AUD
app-server-01       8C/32GB     Standard_E8s_v5   ~$480 AUD
db-server-01        16C/64GB    Standard_E16s_v5  ~$960 AUD
linux-web-01        2C/8GB      Standard_D2s_v5   ~$90 AUD
file-server-01      2C/8GB      Azure Files       ~$500 AUD

Total Monthly Estimate: ~$2,380 AUD

Cost Optimization Options:
  - Reserved Instances (1 year): Save 40%
  - Reserved Instances (3 year): Save 60%
  - Azure Hybrid Benefit: Save up to 80% on Windows
  - Spot VMs: Save up to 90% (interruption possible)
  - Right-sizing: Reduce oversized VMs
  - Auto-shutdown: Development environments

Optimized Monthly (3-year RI + Hybrid Benefit): ~$850 AUD

EOF
```

---

## Step 12 – Show Readiness Assessment

```bash
# Display readiness categories
cat << 'EOF'

Readiness Assessment Results:
==============================

Ready for Azure (4 servers):
  ✓ web-server-01: Standard_D4s_v5
  ✓ web-server-02: Standard_D4s_v5
  ✓ app-server-01: Standard_E8s_v5
  ✓ linux-web-01: Standard_D2s_v5

Ready with Conditions (1 server):
  ⚠ db-server-01: Consider Azure SQL Managed Instance
    - Current: SQL Server 2019
    - Options: Azure SQL MI (PaaS) or IaaS VM
    - Recommendation: Assess with DMA first

Conditionally Ready (1 server):
  ⚠ file-server-01: Migrate to Azure Files
    - Current: File server with 4 TB data
    - Recommendation: Azure Files Premium tier
    - Alternative: Azure NetApp Files

Not Ready (0 servers):
  - No servers with blocking issues

Unknown (0 servers):
  - No servers requiring additional discovery

EOF
```

---

## Step 13 – Show Migration Strategies

```bash
# Display migration strategies
cat << 'EOF'

Migration Strategies (6 Rs):
=============================

1. Rehost (Lift-and-Shift):
   - Move VMs as-is to Azure
   - Minimal changes required
   - Fastest migration path
   - Example: web-server-01, web-server-02

2. Refactor (Re-architect):
   - Modernize to PaaS services
   - Code changes required
   - Better long-term benefits
   - Example: linux-web-01 → App Service

3. Rearchitect (Rebuild):
   - Significant application changes
   - Cloud-native architecture
   - Containers, microservices
   - Example: app-server-01 → AKS

4. Replace (SaaS):
   - Use existing SaaS solution
   - No custom code
   - Fastest time to value
   - Example: file-server-01 → SharePoint Online

5. Retire:
   - Decommission unused systems
   - Reduce migration scope
   - Cost savings

6. Retain:
   - Keep on-premises
   - Not suitable for cloud
   - Compliance requirements

EOF
```

---

## Step 14 – Show Dependency Mapping

```bash
# Display dependency analysis
cat << 'EOF'

Dependency Mapping Results:
============================

Application Tier Dependencies:

Internet → Load Balancer → Web Servers
  ├─ web-server-01 (Port 443)
  └─ web-server-02 (Port 443)
       ↓
  Application Server
  └─ app-server-01 (Port 8080)
       ↓
  Database Server
  └─ db-server-01 (Port 1433)
       ↓
  File Server
  └─ file-server-01 (Port 445)

External Dependencies:
  - Active Directory (authentication)
  - DNS servers
  - Monitoring systems
  - Backup infrastructure

Migration Groups:
  Group 1: File server (foundational)
  Group 2: Database server
  Group 3: Application server
  Group 4: Web servers

Recommendation:
  Migrate in reverse-dependency order
  Test each tier before proceeding

EOF
```

---

## Step 15 – Create Migration Plan

```bash
# Create migration plan document
cat > migration-plan.md << 'EOF'
# Azure Migration Plan

## Phase 1: Preparation (Weeks 1-2)
- [ ] Complete Azure Migrate assessment
- [ ] Review readiness and dependencies
- [ ] Create Azure subscriptions and resource groups
- [ ] Set up networking (VNets, subnets, NSGs)
- [ ] Configure hybrid connectivity (VPN/ExpressRoute)
- [ ] Set up Azure AD and RBAC

## Phase 2: Foundation (Week 3)
- [ ] Migrate file-server-01 to Azure Files
- [ ] Test file share access
- [ ] Update DNS records
- [ ] Validate permissions

## Phase 3: Database (Week 4)
- [ ] Assess SQL Server with DMA
- [ ] Create Azure SQL Managed Instance
- [ ] Migrate database with DMS
- [ ] Test database connectivity
- [ ] Update connection strings

## Phase 4: Application (Week 5)
- [ ] Replicate app-server-01 with Azure Migrate
- [ ] Test failover to Azure
- [ ] Validate application functionality
- [ ] Cutover to Azure

## Phase 5: Web Tier (Week 6)
- [ ] Replicate web-server-01 and web-server-02
- [ ] Configure Azure Load Balancer
- [ ] Test failover
- [ ] Update DNS for cutover
- [ ] Migrate linux-web-01

## Phase 6: Optimization (Weeks 7-8)
- [ ] Right-size VMs based on metrics
- [ ] Enable Azure Hybrid Benefit
- [ ] Purchase reserved instances
- [ ] Configure backup and disaster recovery
- [ ] Set up monitoring and alerts
- [ ] Decommission on-premises servers

EOF

echo "Migration plan created"
```

---

## Step 16 – Show Cost Estimation

```bash
# Display detailed cost breakdown
cat << 'EOF'

Detailed Cost Estimation:
=========================

Compute Costs (Monthly):
  Web Servers:   $175 × 2 = $350
  App Server:    $480 × 1 = $480
  DB Server:     $960 × 1 = $960
  Linux Web:     $90 × 1  = $90
  Subtotal:                $1,880

Storage Costs (Monthly):
  Web Server disks:      $30 × 2 = $60
  App Server disk:       $60 × 1 = $60
  DB Server disk:        $120 × 1 = $120
  Linux disk:            $15 × 1 = $15
  Azure Files (4 TB):              $500
  Subtotal:                        $755

Networking Costs (Monthly):
  Load Balancer:                   $20
  VPN Gateway:                     $150
  Data transfer (100 GB):          $10
  Subtotal:                        $180

Backup Costs (Monthly):
  VM backups (6 servers):          $60
  SQL backup:                      $40
  Subtotal:                        $100

Other Costs (Monthly):
  Azure Monitor:                   $50
  Log Analytics:                   $30
  Subtotal:                        $80

TOTAL MONTHLY: $2,995 AUD

Annual Cost: $35,940 AUD

With Optimizations:
  - 3-year Reserved Instances (-60%)
  - Azure Hybrid Benefit (-40% Windows)
  - Right-sizing (-20%)

Optimized Annual Cost: $14,376 AUD
Total Savings: $21,564 AUD/year (60%)

EOF
```

---

## Step 17 – Show Assessment Report

```bash
# Generate assessment summary
cat > assessment-report.txt << 'EOF'
================================================================================
                    AZURE MIGRATE ASSESSMENT REPORT
================================================================================

Project: Azure Migration Assessment
Date: 2025-11-21
Region: Australia East

EXECUTIVE SUMMARY
-----------------
Total Servers Assessed: 6
Ready for Azure: 4 (67%)
Conditionally Ready: 2 (33%)
Not Ready: 0 (0%)

CURRENT ENVIRONMENT
-------------------
Total vCPUs: 36
Total Memory: 144 GB
Total Storage: 8.3 TB
Operating Systems: Windows (5), Linux (1)

RECOMMENDED AZURE ENVIRONMENT
------------------------------
VM Instances: 5
Azure Files: 1 (4 TB)
Estimated Monthly Cost: $2,995 AUD
Optimized Monthly Cost: $1,198 AUD (with RI + Hybrid)

MIGRATION READINESS
-------------------
✓ Ready: web-server-01, web-server-02, app-server-01, linux-web-01
⚠ Conditional: db-server-01 (SQL assessment needed)
⚠ Conditional: file-server-01 (migrate to Azure Files)

TOP RECOMMENDATIONS
-------------------
1. Use Azure Hybrid Benefit for Windows VMs
2. Purchase 3-year Reserved Instances
3. Migrate file server to Azure Files
4. Assess SQL Server with Database Migration Assistant
5. Consider App Service for web workloads
6. Implement Azure Backup for all resources

MIGRATION TIMELINE
------------------
Phase 1: Preparation (2 weeks)
Phase 2: Foundation (1 week)
Phase 3: Database (1 week)
Phase 4: Application (1 week)
Phase 5: Web Tier (1 week)
Phase 6: Optimization (2 weeks)

Total Duration: 8 weeks

RISK ASSESSMENT
---------------
Low Risk: Web servers, Linux server
Medium Risk: Application server
High Risk: Database server (requires DMA assessment)

NEXT STEPS
----------
1. Review and approve assessment
2. Initiate dependency mapping
3. Run SQL Database Migration Assistant
4. Create Azure environment
5. Begin pilot migration with file server

================================================================================
EOF

cat assessment-report.txt
```

---

## Step 18 – Show SQL Assessment

```bash
# Display SQL-specific assessment
cat << 'EOF'

SQL Server Migration Assessment:
=================================

Current Environment:
  Server: db-server-01
  SQL Version: SQL Server 2019 Enterprise
  Cores: 16
  Memory: 64 GB
  Storage: 2 TB
  Databases: 5
  Availability: AlwaysOn (not configured)

Azure Options:

1. Azure SQL Managed Instance (Recommended):
   ✓ Near 100% compatibility
   ✓ Managed service (PaaS)
   ✓ Built-in high availability
   ✓ Automated backups
   ✓ Cost: ~$1,800/month (General Purpose, 16 vCores)

2. Azure SQL Database:
   ⚠ May require application changes
   ✓ Fully managed PaaS
   ✓ Hyperscale for large databases
   ✓ Cost: ~$1,200/month (General Purpose, 16 vCores)

3. SQL Server on Azure VM:
   ✓ Full compatibility
   ⚠ More management overhead
   ✓ Use existing licenses
   ✓ Cost: ~$960/month (VM) + licenses

Assessment Tools:
  - Data Migration Assistant (DMA): Compatibility check
  - Database Experimentation Assistant: Performance testing
  - Azure Data Studio: Query and manage

Recommendation:
  Run DMA to identify:
    - Breaking changes
    - Deprecated features
    - New feature opportunities
  Then choose between SQL MI or SQL Database

EOF
```

---

## Step 19 – Show Migration Tools

```bash
# Display available migration tools
cat << 'EOF'

Azure Migration Tools:
======================

Server Migration:
  Tool: Azure Migrate: Server Migration
  - VMware VM replication
  - Hyper-V VM replication
  - Physical server migration
  - AWS/GCP VM migration
  - Agentless or agent-based

Database Migration:
  Tool: Azure Database Migration Service
  - SQL Server → Azure SQL
  - MySQL → Azure Database for MySQL
  - PostgreSQL → Azure Database for PostgreSQL
  - MongoDB → Cosmos DB
  - Online and offline migration

Web Apps:
  Tool: Azure App Service Migration Assistant
  - ASP.NET applications
  - PHP applications
  - Compatibility assessment
  - Automated migration

Storage:
  Tool: Storage Migration Service
  - File server to Azure Files
  - Incremental sync
  - Cutover automation

Data:
  Tool: Azure Data Box
  - Offline data transfer
  - 80 TB per device
  - Large dataset migration

Discovery:
  Tool: Azure Migrate Appliance
  - Agentless discovery
  - Performance collection
  - Dependency mapping

EOF
```

---

## Step 20 – Show Best Practices

```bash
# Display migration best practices
cat << 'EOF'

Azure Migration Best Practices:
================================

Planning:
  ✓ Complete thorough assessment
  ✓ Identify dependencies
  ✓ Classify applications by criticality
  ✓ Define success criteria
  ✓ Plan migration waves

Testing:
  ✓ Test migrations in non-production
  ✓ Validate functionality
  ✓ Performance testing
  ✓ Disaster recovery testing
  ✓ Document test results

Security:
  ✓ Use Azure Key Vault for secrets
  ✓ Enable encryption at rest
  ✓ Configure network security groups
  ✓ Implement Azure AD authentication
  ✓ Enable Microsoft Defender for Cloud

Networking:
  ✓ Design hub-and-spoke topology
  ✓ Use private endpoints
  ✓ Configure DNS properly
  ✓ Plan for hybrid connectivity
  ✓ Implement ExpressRoute for production

Optimization:
  ✓ Right-size resources
  ✓ Use Azure Advisor recommendations
  ✓ Enable auto-scaling
  ✓ Implement cost management
  ✓ Use tags for organization

Governance:
  ✓ Define naming conventions
  ✓ Implement Azure Policy
  ✓ Set up RBAC properly
  ✓ Enable resource locks
  ✓ Configure management groups

EOF
```

---

## Step 21 – Show Performance Baselines

```bash
# Display performance baseline requirements
cat << 'EOF'

Performance Baseline Collection:
=================================

Metrics to Collect:
  - CPU utilization (%)
  - Memory utilization (%)
  - Disk IOPS (read/write)
  - Disk throughput (MB/s)
  - Network throughput (Mbps)
  - Application response time

Collection Duration:
  - Minimum: 1 day
  - Recommended: 1 week
  - Ideal: 1 month
  - Include: Peak business hours

Tools:
  - Azure Migrate appliance (continuous)
  - Performance Monitor (Windows)
  - SAR, top, iostat (Linux)
  - Application-specific monitoring

Percentile Usage:
  - 50th percentile: Average usage
  - 95th percentile: Most scenarios
  - 99th percentile: Peak capacity

Sample Results:
  web-server-01:
    CPU: 45% average, 75% (95th percentile)
    Memory: 60% average, 80% (95th percentile)
    Disk: 500 IOPS average, 1200 (95th percentile)
  
  Recommendation: Standard_D4s_v5 with 1.3x comfort factor

EOF
```

---

## Step 22 – Export Assessment Summary

```bash
# Create CSV export of assessment
cat > assessment-export.csv << 'EOF'
Server Name,OS,Cores,Memory GB,Disk GB,Azure VM,Monthly Cost,Readiness
web-server-01,Windows Server 2019,4,16,500,Standard_D4s_v5,$175,Ready
web-server-02,Windows Server 2019,4,16,500,Standard_D4s_v5,$175,Ready
app-server-01,Windows Server 2022,8,32,1024,Standard_E8s_v5,$480,Ready
db-server-01,Windows Server 2019,16,64,2048,Standard_E16s_v5,$960,Conditional
linux-web-01,Ubuntu 20.04,2,8,200,Standard_D2s_v5,$90,Ready
file-server-01,Windows Server 2016,2,8,4096,Azure Files,$500,Conditional
EOF

echo "Assessment exported to CSV"
cat assessment-export.csv
```

---

## Step 23 – Show Compliance Considerations

```bash
# Display compliance requirements
cat << 'EOF'

Azure Compliance Considerations:
=================================

Data Residency:
  - Choose appropriate Azure region
  - Australia East for Australian data
  - Use geo-redundancy within country
  - Consider sovereignty requirements

Certifications:
  - ISO 27001, 27018
  - SOC 1, 2, 3
  - PCI DSS
  - HIPAA (US healthcare)
  - IRAP (Australian government)

Industry Standards:
  - APRA (Australian financial services)
  - GDPR (European data protection)
  - PDPA (Singapore)

Azure Tools:
  - Azure Policy for compliance
  - Microsoft Purview for governance
  - Compliance Manager
  - Service Trust Portal

Best Practices:
  - Document compliance requirements
  - Map controls to Azure services
  - Regular compliance assessments
  - Maintain audit trails
  - Encrypt sensitive data

EOF
```

---

## Step 24 – Show Risk Mitigation

```bash
# Display risk mitigation strategies
cat << 'EOF'

Migration Risk Mitigation:
==========================

Technical Risks:

1. Data Loss:
   Mitigation:
   - Test backup and restore
   - Use DMS for online migration
   - Keep source systems until validation
   - Implement checkpoints

2. Downtime:
   Mitigation:
   - Plan maintenance windows
   - Use replication for cutover
   - Test failover procedures
   - Have rollback plan

3. Performance Issues:
   Mitigation:
   - Baseline current performance
   - Right-size Azure resources
   - Load testing before cutover
   - Monitor during migration

4. Compatibility:
   Mitigation:
   - Run assessment tools (DMA)
   - Test in Azure dev environment
   - Document dependencies
   - Remediate issues before production

Business Risks:

1. Budget Overruns:
   Mitigation:
   - Detailed cost estimation
   - Use Azure Cost Management
   - Set budget alerts
   - Plan for reserved instances

2. Timeline Delays:
   Mitigation:
   - Build in buffer time
   - Identify critical path
   - Parallel migration tracks
   - Clear milestone definitions

3. Skills Gap:
   Mitigation:
   - Azure training for team
   - Partner with Azure experts
   - Documentation and runbooks
   - Managed services consideration

EOF
```

---

## Step 25 – Show Post-Migration Tasks

```bash
# Display post-migration checklist
cat << 'EOF'

Post-Migration Checklist:
=========================

Immediate (Day 1-7):
  □ Verify all applications functional
  □ Check performance metrics
  □ Validate data integrity
  □ Test disaster recovery
  □ Update documentation
  □ Monitor costs daily
  □ Address any issues immediately

Short-term (Week 2-4):
  □ Optimize resource sizing
  □ Configure autoscaling
  □ Enable Azure Backup
  □ Set up monitoring alerts
  □ Review security settings
  □ Implement cost optimization
  □ Decommission on-premises (after validation)

Long-term (Month 2-6):
  □ Purchase reserved instances
  □ Implement Azure Policy
  □ Modernize to PaaS services
  □ Enable advanced security features
  □ Regular cost reviews
  □ Continuous optimization
  □ Team training on Azure

EOF
```

---

## Step 26 – List Azure Migrate Projects

```bash
# List all Azure Migrate projects
az migrate project list \
  --resource-group "$RG_NAME" \
  --output table
```

---

## Step 27 – Show Project Details

```bash
# Get project details
az migrate project show \
  --name "$MIGRATE_PROJECT" \
  --resource-group "$RG_NAME" \
  --output json
```

---

## Step 28 – Show Azure Regions

```bash
# List all Azure regions
az account list-locations \
  --query "[?metadata.regionType=='Physical'].{Name:name, DisplayName:displayName, Geography:metadata.geography}" \
  --output table | head -20

echo "Regions listed"
```

---

## Step 29 – Show Assessment Resources

```bash
# Display assessment resources
cat << 'EOF'

Additional Assessment Resources:
=================================

Documentation:
  - Azure Migrate documentation
  - Azure Architecture Center
  - Cloud Adoption Framework
  - Well-Architected Framework

Tools:
  - Azure Migrate hub (portal)
  - Total Cost of Ownership (TCO) Calculator
  - Azure Pricing Calculator
  - Azure Advisor

Training:
  - Microsoft Learn: Azure Migrate
  - Microsoft Learn: Azure Fundamentals
  - Cloud Adoption Framework training

Support:
  - Azure Migration Program (AMP)
  - FastTrack for Azure
  - Azure support plans
  - Partner network

Community:
  - Azure forums
  - Stack Overflow
  - GitHub repositories
  - Azure blog

EOF
```

---

## Step 30 – Create Handoff Document

```bash
# Create migration handoff document
cat > handoff-document.md << 'EOF'
# Migration Handoff Document

## Assessment Summary
- **Servers Assessed**: 6
- **Azure Ready**: 4 servers
- **Conditional**: 2 servers
- **Estimated Monthly Cost**: $2,995 AUD
- **Optimized Cost**: $1,198 AUD

## Migration Approach
- **Strategy**: Phased migration over 8 weeks
- **Method**: Azure Migrate for VMs, Azure Files for file server
- **Priority**: Foundation → Database → Application → Web

## Next Steps
1. Approve assessment and budget
2. Provision Azure environment
3. Configure hybrid networking
4. Begin Phase 1: Foundation

## Key Contacts
- **Project Sponsor**: [Name]
- **Technical Lead**: [Name]
- **Azure Architect**: [Name]

## Success Criteria
- Zero data loss
- <4 hours downtime per system
- Performance meets or exceeds baseline
- Cost within approved budget

## Documentation Locations
- Assessment Report: `assessment-report.txt`
- Migration Plan: `migration-plan.md`
- Inventory: `sample-inventory.json`
- Cost Export: `assessment-export.csv`

## Risks and Mitigations
See `migration-plan.md` for detailed risk register

EOF

echo "Handoff document created"
```

---

## Step 31 – Show Final Recommendations

```bash
# Display final recommendations
cat << 'EOF'

Final Recommendations:
======================

Priority 1 - Must Do:
  ✓ Complete SQL Server assessment with DMA
  ✓ Test pilot migration with file server
  ✓ Set up Azure networking and connectivity
  ✓ Configure backup and disaster recovery
  ✓ Document rollback procedures

Priority 2 - Should Do:
  ✓ Enable Azure Hybrid Benefit
  ✓ Plan for reserved instance purchases
  ✓ Implement dependency mapping
  ✓ Set up monitoring and alerting
  ✓ Train operations team on Azure

Priority 3 - Consider:
  ✓ Modernize web apps to App Service
  ✓ Evaluate Azure SQL Managed Instance
  ✓ Implement Azure DevOps pipelines
  ✓ Plan for multi-region deployment
  ✓ Consider containerization (AKS)

Quick Wins:
  - Start with file server migration
  - Enable Azure Advisor recommendations
  - Use Azure Cost Management
  - Implement tagging strategy
  - Enable Azure Monitor

Success Metrics:
  - Migration timeline adherence
  - Zero data loss incidents
  - Minimal unplanned downtime
  - Cost within 10% of estimate
  - User satisfaction scores

EOF
```

---

## Step 32 – Cleanup

```bash
# Delete resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f sample-inventory.json
rm -f migration-plan.md
rm -f assessment-report.txt
rm -f assessment-export.csv
rm -f handoff-document.md

echo "Cleanup complete"
```

---

## Summary

You created an Azure Migrate project for workload assessment, explored Azure Migrate capabilities for discovering on-premises VMware, Hyper-V, physical servers, and cloud instances, analyzed sample inventory of six servers with varying workloads and resource requirements, reviewed Azure VM sizing recommendations and cost estimates with optimization strategies, examined readiness assessment results and identified migration blockers, understood migration strategies using the 6 Rs framework, explored dependency mapping for application tiers, created detailed migration plan with phased approach over eight weeks, generated cost estimates showing potential 60% savings with reserved instances and hybrid benefits, reviewed SQL Server migration options including Managed Instance and Azure SQL Database, documented post-migration tasks and success criteria, and prepared comprehensive assessment reports for stakeholder review and migration planning.
