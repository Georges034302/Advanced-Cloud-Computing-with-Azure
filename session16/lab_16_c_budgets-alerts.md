# Lab 16.C: Budgets and Alerts

## Objectives
- Create Azure budgets for cost control
- Configure budget alert thresholds
- Set up action groups for notifications
- Create cost anomaly alerts
- Configure department spending quotas
- Implement automated responses to alerts
- Track budget vs actual spending
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
RG_NAME="rg-lab16c-budget"
STORAGE_NAME="storage$RANDOM"
BUDGET_NAME="monthly-budget-$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$BUDGET_NAME"
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

## Step 3 – Show Budgets Overview

```bash
# Display budget concepts
cat << 'EOF'

Azure Budgets:
==============

Overview:
  Azure Budgets help you plan and drive organizational accountability.
  With budgets, you can account for the Azure services you consume or
  subscribe to during a specific period. They help you proactively
  manage costs and monitor spending trends over time.

Budget Types:

1. Cost Budget:
   - Based on actual spending
   - Tracks consumption charges
   - Most common type
   - Monthly or custom period

2. Usage Budget (EA only):
   - Based on prepaid commitment
   - Tracks against monetary commitment
   - Enterprise Agreements
   - Shows credit consumption

Budget Scopes:

1. Subscription:
   - Entire subscription spending
   - All resource groups
   - Cross-team visibility

2. Resource Group:
   - Specific resource group only
   - Team or project budgets
   - Granular control

3. Management Group (EA):
   - Multiple subscriptions
   - Department or division level
   - Hierarchical budgets

Budget Periods:

1. Monthly:
   - Resets first of each month
   - Most common
   - Aligns with billing

2. Quarterly:
   - 3-month period
   - Aligns with fiscal quarters
   - Strategic planning

3. Annual:
   - 12-month period
   - Yearly budgets
   - Long-term planning

4. Custom:
   - Specific start and end dates
   - Project-based budgets
   - One-time initiatives

Alert Conditions:

1. Actual Cost:
   - Based on incurred charges
   - Real spending
   - Already consumed
   - Historical data

2. Forecasted Cost:
   - Predicted future spending
   - Based on trends
   - Early warning
   - Proactive alerts

Alert Thresholds:

Typical Configuration:
  - 50% - Early warning
  - 75% - Attention needed
  - 90% - Urgent action required
  - 100% - Budget exceeded
  - 110% - Overspend alert

Multiple Thresholds:
  - Up to 5 alerts per budget
  - Different recipients per threshold
  - Different actions per threshold
  - Percentage or absolute amount

Notification Methods:

1. Email:
   - Up to 5 email addresses per alert
   - Includes budget details
   - Spending summary
   - Link to Cost Management

2. Action Groups:
   - Email/SMS
   - Webhook
   - Azure Function
   - Logic App
   - Runbook
   - ITSM integration

Best Practices:

✓ Set realistic budgets
✓ Start with actual cost alerts
✓ Add forecast alerts for planning
✓ Multiple thresholds (50%, 75%, 90%, 100%)
✓ Include relevant stakeholders
✓ Review and adjust monthly
✓ Align with fiscal calendar
✓ Document budget owners
✓ Automate responses when possible
✓ Track budget vs actual trends

Common Budget Scenarios:

Development Team:
  Budget: $2,000/month
  Alerts: 75%, 90%, 100%
  Recipients: Team lead, dev manager
  Action: Email notification

Production Environment:
  Budget: $10,000/month
  Alerts: 50%, 75%, 90%, 100%, 110%
  Recipients: Engineering, finance, executives
  Action: Email + automated shutdown of non-critical resources at 100%

Project-Based:
  Budget: $50,000 (6 months)
  Alerts: 25%, 50%, 75%, 90%, 100%
  Recipients: Project manager, sponsor
  Action: Monthly review meetings

EOF
```

---

## Step 4 – Get Subscription ID

```bash
# Get current subscription ID
SUBSCRIPTION_ID=$(az account show --query id --output tsv)

echo "$SUBSCRIPTION_ID"
```

---

## Step 5 – Create Sample Resources

```bash
# Create storage account to generate some costs
az storage account create \
  --name "$STORAGE_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

echo "Storage account created"

# Tag resource group with budget information
az group update \
  --name "$RG_NAME" \
  --tags \
    BudgetOwner="IT Department" \
    BudgetPeriod="Monthly" \
    BudgetAmount="1000" \
    CostCenter="CC-1001"

echo "Tags applied"
```

---

## Step 6 – Create Action Group

```bash
# Create action group for budget alerts
az monitor action-group create \
  --name "budget-alerts" \
  --resource-group "$RG_NAME" \
  --short-name "budget" \
  --email-receiver \
    name=Admin \
    email-address=admin@company.com \
  --email-receiver \
    name=Finance \
    email-address=finance@company.com

echo "Action group created"
```

---

## Step 7 – Show Budget CLI Commands

```bash
# Display budget management commands
cat << EOF

Budget Management via CLI:
==========================

Create Budget:

# Create monthly budget for subscription
az consumption budget create \\
  --budget-name "subscription-monthly-budget" \\
  --category Cost \\
  --amount 5000 \\
  --time-grain Monthly \\
  --start-date 2024-01-01 \\
  --end-date 2024-12-31

# Create budget for resource group
az consumption budget create \\
  --budget-name "$BUDGET_NAME" \\
  --category Cost \\
  --amount 1000 \\
  --time-grain Monthly \\
  --start-date 2024-01-01 \\
  --resource-group "$RG_NAME"

List Budgets:

# List all budgets
az consumption budget list

# Show specific budget
az consumption budget show \\
  --budget-name "$BUDGET_NAME"

Update Budget:

# Update budget amount
az consumption budget update \\
  --budget-name "$BUDGET_NAME" \\
  --amount 1500

Delete Budget:

# Remove budget
az consumption budget delete \\
  --budget-name "$BUDGET_NAME"

Note: Budget notifications are configured via Portal or ARM templates.

EOF
```

---

## Step 8 – Show Budget Tracking

```bash
# Display budget tracking example
cat << 'EOF'

Budget vs Actual Tracking:
==========================

Budget Performance Metrics:

1. Consumption Rate:
   Formula: (Current Spend / Budget Amount) × 100
   
   Example:
   Budget: $10,000
   Current: $6,500
   Rate: 65%
   Status: On track

2. Burn Rate:
   Formula: Current Spend / Days Elapsed
   
   Example:
   Current Spend: $6,500
   Days Elapsed: 20
   Burn Rate: $325/day
   
   Projected Monthly: $325 × 30 = $9,750
   Status: Slightly under budget

3. Variance:
   Formula: Actual Spend - Budgeted Spend
   
   Example at Day 20:
   Actual: $6,500
   Budgeted (pro-rata): $6,667
   Variance: -$167 (2.5% under)
   Status: Favorable

Budget Dashboard:

┌─────────────────────────────────────────────┐
│ Monthly Budget Summary - January 2024      │
├─────────────────────────────────────────────┤
│ Budget Amount:        $10,000              │
│ Current Spend:        $7,500               │
│ Remaining:            $2,500               │
│ % Consumed:           75%                  │
│                                             │
│ Days Elapsed:         20 / 31              │
│ Days Remaining:       11                   │
│                                             │
│ Daily Burn Rate:      $375                 │
│ Projected Total:      $11,625              │
│ Projected Variance:   +$1,625 (16% over)   │
│                                             │
│ Status: ⚠ Trending over budget             │
├─────────────────────────────────────────────┤
│ Alerts:                                     │
│ ✓ 50% threshold reached (Day 13)           │
│ ✓ 75% threshold reached (Day 20)           │
│ ⏰ 90% projected (Day 24)                   │
│ ⚠ 100% projected (Day 27)                  │
└─────────────────────────────────────────────┘

Cost Trend Chart:

$12k │                                    ╱─── Forecast
     │                                ╱───
$10k │────────────────────────────────────── Budget
     │                        ╱───╱
 $8k │                  ╱───╱
     │            ╱───╱
 $6k │      ╱───╱
     │ ───╱                                    
 $4k │╱                                       ─── Actual
     │
 $2k │
     │
   0 ┴─────────────────────────────────────
     1   5   10  15  20  25  30  (days)

Multi-Budget View:

Development:
  Budget: $2,000
  Actual: $1,200 (60%)
  Status: ✓ On track

Staging:
  Budget: $3,000
  Actual: $2,850 (95%)
  Status: ⚠ Near limit

Production:
  Budget: $10,000
  Actual: $7,500 (75%)
  Status: ⚠ Trending over

Total:
  Budget: $15,000
  Actual: $11,550 (77%)
  Projected: $16,200
  Variance: +$1,200 (8% over)

EOF
```

---

## Step 9 – Show Anomaly Detection

```bash
# Display anomaly detection concepts
cat << 'EOF'

Cost Anomaly Detection:
=======================

Overview:
  Azure uses machine learning to detect unusual spending patterns
  and alert you before they appear in budgets.

Anomaly Types:

1. Spike Anomaly:
   Description: Sudden increase
   Example:
     Normal daily: $100
     Today: $500
     Anomaly: +400% spike
   
   Possible Causes:
     - New resources deployed
     - Configuration change
     - Usage spike
     - Data transfer surge

2. Drop Anomaly:
   Description: Unexpected decrease
   Example:
     Normal daily: $200
     Today: $50
     Anomaly: -75% drop
   
   Possible Causes:
     - Resources stopped
     - Service outage
     - Configuration error

3. Trend Change:
   Description: Gradual shift
   Example:
     Week 1-4: $100/day
     Week 5-8: $150/day
     Anomaly: +50% trend
   
   Possible Causes:
     - Growing usage
     - Added services
     - New projects

Sample Anomaly Alert:

───────────────────────────────────────
Cost Anomaly Detected
───────────────────────────────────────

Anomaly Details:
  Date: 2024-01-15
  Type: Spending Spike
  Severity: High
  Confidence: 92%

Impact:
  Expected Daily Cost: $150
  Actual Cost: $650
  Increase: $500 (+333%)
  Monthly Impact: +$15,000

Affected Scope:
  Resource Group: rg-web-app
  Top Services:
    1. Virtual Machines: +$300
    2. Storage: +$150
    3. Networking: +$50

Root Cause:
  - 10 new D4s_v3 instances deployed
  - Deploy time: 08:30 UTC
  - Deployed by: user@company.com

Recommended Actions:
  ☐ Verify new VMs are necessary
  ☐ Check if spot instances appropriate
  ☐ Review storage activity
  ☐ Check deployment approvals

[Investigate] [Dismiss as Expected]

───────────────────────────────────────

Benefits vs Budget Alerts:

Anomaly Detection:
  - Earlier detection (hours)
  - Identifies root causes
  - No setup required
  - Adaptive to changes

Budget Alerts:
  - Planning focused
  - Monthly tracking
  - Requires configuration
  - Strategic control

Use Both:
  ✓ Anomalies for tactical response
  ✓ Budgets for strategic planning

EOF
```

---

## Step 10 – Show Automated Responses

```bash
# Display automation examples
cat << 'EOF'

Automated Budget Responses:
===========================

Response Patterns:

1. Notification (Low Risk):
   Trigger: 75% budget
   Action:
     - Email team
     - Post to Slack/Teams
     - Log to monitoring
   
   Implementation: Action Group + Email

2. Automated Reporting (Medium):
   Trigger: 90% budget
   Action:
     - Generate cost report
     - Identify top consumers
     - Create tracking ticket
   
   Implementation: Azure Function

3. Graceful Scaling (Medium):
   Trigger: 95% budget
   Action:
     - Scale down non-prod VMs
     - Reduce VMSS instances
     - Lower App Service tier
   
   Implementation: Runbook

4. Protective Shutdown (High Risk):
   Trigger: 100% budget
   Action:
     - Stop dev/test VMs
     - Prevent new deployments
     - Escalate to management
   
   Implementation: Logic App

Example Runbook (PowerShell):

```powershell
# Stop non-production VMs when budget exceeded

param([object]$WebhookData)

# Parse budget alert
$alert = ConvertFrom-Json $WebhookData.RequestBody

if ($alert.data.percentage -gt 100) {
    
    # Get dev VMs that are running
    $vms = Get-AzVM -Status | Where-Object {
        $_.Tags['Environment'] -eq 'Dev' -and
        $_.PowerState -eq 'VM running'
    }
    
    # Stop each VM
    foreach ($vm in $vms) {
        Write-Output "Stopping: $($vm.Name)"
        Stop-AzVM -ResourceGroupName $vm.ResourceGroupName `
                  -Name $vm.Name -Force
    }
    
    # Notify
    Send-MailMessage -To "ops@company.com" `
        -Subject "Budget Auto-Response" `
        -Body "Stopped $($vms.Count) dev VMs"
}
```

Safety Considerations:

✓ Never auto-stop production
✓ Tag resources clearly
✓ Test in non-prod first
✓ Require approval for high-impact
✓ Audit all automated actions
✓ Rollback capability
✓ Notification before action

Gradual Approach:

Phase 1: Notification only
Phase 2: Automated reporting
Phase 3: Safe scaling (dev/test)
Phase 4: Full automation

EOF
```

---

## Step 11 – Show Forecasting

```bash
# Display forecasting concepts
cat << 'EOF'

Budget Forecasting:
===================

Forecast Methods:

1. Historical Average:
   Example:
     Past 3 months: $8K, $9K, $10K
     Average: $9K
     Forecast: $9K next month

2. Trend Analysis:
   Example:
     Month 1: $8K
     Month 2: $9K
     Month 3: $10K
     Trend: +$1K/month
     Forecast: $11K next month

3. Machine Learning (Azure):
   - Analyzes usage patterns
   - Accounts for multiple factors
   - Most accurate
   - Built into Cost Management

Current Month Forecast:

Date: January 20
MTD Spend: $7,000
Days Elapsed: 20
Days Remaining: 11

Simple Projection:
  Daily avg: $350
  Forecast: $10,850

Trend-Adjusted:
  Recent increase
  Forecast: $11,200

Azure ML:
  Considers patterns
  Forecast: $10,950
  Confidence: 85%
  Range: $10,400 - $11,500

Forecast Alerts:

Alert on Forecast:
  Threshold: 100% of budget (forecasted)
  Lead time: 7-10 days
  Action: Proactive cost reduction

Benefits:
  ✓ Early warning
  ✓ Time to act
  ✓ Prevent overruns

Using Forecasts:

If forecast < 80%:
  ✓ Review optimization
  ✓ Consider new projects

If forecast = 90-100%:
  ✓ On track
  ✓ Continue monitoring

If forecast > 110%:
  ✗ Action required
  ✓ Immediate investigation
  ✓ Implement controls

EOF
```

---

## Step 12 – Show Department Quotas

```bash
# Display EA department quotas
cat << 'EOF'

Department Quotas (EA):
=======================

Organizational Hierarchy:

  ┌─────────────────────┐
  │ Enrollment          │
  └──────────┬──────────┘
             │
    ┌────────┴────────┐
    │                 │
┌───▼──────┐    ┌────▼───────┐
│Department│    │ Department │
│ (IT)     │    │ (Marketing)│
└───┬──────┘    └────┬───────┘
    │                │
┌───▼────┐      ┌────▼─────┐
│Account │      │ Account  │
└───┬────┘      └────┬─────┘
    │                │
┌───▼──────────┐     │
│Subscription  │     │
└──────────────┘ ┌───▼──────────┐
                 │ Subscription │
                 └──────────────┘

IT Department Quota:

Annual Commitment: $500,000
Quarterly Quota: $125,000
Monthly Target: ~$42,000

Current Month:
  Spent: $38,000
  Remaining: $4,000
  % Used: 90%

Quarterly:
  Spent: $110,000
  Remaining: $15,000
  % Used: 88%

Alerts:
  ✓ 75% monthly (triggered)
  ⏰ 90% monthly (pending)
  ✓ 80% quarterly (triggered)

Department Report:

Month: January 2024
Department: IT

Accounts:
  1. Production: $18,000 (47%)
  2. Development: $8,000 (21%)
  3. Testing: $6,000 (16%)
  4. Staging: $4,000 (11%)
  5. Analytics: $2,000 (5%)

Total: $38,000

Top Services:
  1. Virtual Machines: $15,000
  2. Azure SQL: $8,000
  3. Storage: $6,000
  4. Networking: $5,000
  5. Other: $4,000

Quota Benefits:

✓ Departmental accountability
✓ Cost governance
✓ Fair distribution
✓ Chargeback support
✓ Optimization incentive

EOF
```

---

## Step 13 – Show Best Practices

```bash
# Display budget best practices
cat << 'EOF'

Budget Best Practices:
======================

Budget Strategy:

1. Hierarchical Budgets:
   
   Subscription: $50,000/month
     Production: $30,000 (60%)
     Staging: $10,000 (20%)
     Development: $10,000 (20%)

2. Alert Configuration:
   
   Standard Thresholds:
     50%: Informational
     75%: Warning
     90%: Critical
     100%: Exceeded
     110%: Emergency

3. Budget Lifecycle:
   
   Beginning (Day 1-5):
     ✓ Review previous month
     ✓ Analyze variances
     ✓ Update forecast

   Mid-Month (Day 15-20):
     ✓ Check pace (~50%)
     ✓ Review forecast
     ✓ Address issues

   End of Month (Day 25-30):
     ✓ Final forecast
     ✓ Prepare explanations
     ✓ Plan next month

   Quarterly:
     ✓ Deep dive analysis
     ✓ Budget adjustments
     ✓ Strategic planning

4. Stakeholder Management:
   
   Engineering:
     - Cost visibility
     - Budget ownership
     - Optimization guidance

   Finance:
     - Overall tracking
     - Variance analysis
     - Forecast accuracy

   Management:
     - Executive summaries
     - Strategic decisions
     - Investment priorities

Common Pitfalls:

✗ Set and forget
✗ Too many budgets
✗ Alert fatigue
✗ No ownership
✗ Unrealistic budgets
✗ No action on alerts
✗ Lack of context

Success Metrics:

Budget Adherence:
  Target: 90%+ budgets met

Forecast Accuracy:
  Target: Within 10%

Response Time:
  Target: < 24 hours

Cost Optimization:
  Target: 10-20% reduction

Maturity Levels:

Level 1: Basic
  - Few budgets
  - Manual tracking
  - Reactive

Level 2: Managed
  - Comprehensive budgets
  - Regular reviews
  - Some automation

Level 3: Defined
  - Integrated processes
  - Proactive management
  - Automated alerts

Level 4: Measured
  - Metrics-driven
  - Continuous improvement
  - Highly automated

Level 5: Optimized
  - Strategic tool
  - Predictive analytics
  - Business enablement

EOF
```

---

## Step 14 – List Action Groups

```bash
# List created action groups
az monitor action-group list \
  --query "[].{Name:name, ResourceGroup:resourceGroup, Enabled:enabled}" \
  --output table
```

---

## Step 15 – Show Budget Template

```bash
# Display budget template
cat << 'EOF'

Budget Template:
================

Budget Name: Production-App-A
Period: Monthly
Amount: $15,000
Owner: App A Team Lead
Scope: rg-prod-app-a

Alert Thresholds:
  50%: app-team@company.com
  75%: app-team@company.com, manager@company.com
  90%: Above + finance@company.com
  100%: Above + vp-engineering@company.com

Action Plan:
  At 75%: Review cost drivers
  At 90%: Identify reductions
  At 100%: Implement reductions

Review Schedule:
  Weekly: Team lead
  Monthly: Manager + Finance
  Quarterly: Strategic review

Historical Data:
  Last 3 months: $14K, $14.5K, $15.2K
  Trend: Slightly increasing
  Justification: $15K appropriate

EOF
```

---

## Step 16 – Cleanup

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

You explored Azure Budgets for proactive cost management and financial accountability enabling planning and monitoring of Azure service consumption across subscriptions, resource groups, and departments, created action groups for automated budget alert notifications via email, SMS, webhook, Azure Functions, and Logic Apps enabling multi-channel alerting and automated responses, learned budget configuration with multiple alert thresholds at 50%, 75%, 90%, and 100% for both actual and forecasted spending with escalating notifications to different stakeholders based on severity, examined cost anomaly detection using machine learning to identify unusual spending patterns with immediate alerts providing hours of advance warning before budget impacts, configured department spending quotas for Enterprise Agreements enabling hierarchical cost control and chargeback across organizational units with quarterly and annual tracking, implemented automated response strategies including graceful scaling of non-production resources, protective shutdown workflows, and escalation procedures using runbooks and Logic Apps with safety considerations, tracked budget versus actual spending using consumption rate, burn rate, variance analysis, and runway calculations to project month-end outcomes and enable proactive cost management, utilized forecasting capabilities based on historical trends, seasonal patterns, and machine learning providing 7-10 day lead time for cost reduction actions before budget is exceeded, established budget lifecycle processes with beginning of month reviews, mid-month pace checks, end of month variance analysis, plus quarterly deep dives and annual strategic planning, and developed comprehensive budget management maturity framework progressing from basic reactive tracking to optimized strategic cost enablement.
