# Lab 15.C: App Service Migration

## Objectives
- Assess web application compatibility
- Create App Service plan
- Deploy web application to App Service
- Configure custom domain and SSL
- Migrate application settings
- Test application functionality
- Monitor application performance
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
RG_NAME="rg-lab15c-appservice"
APP_PLAN="plan-webapp-$RANDOM"
WEB_APP="webapp-migrate-$RANDOM"
APP_INSIGHTS="ai-webapp-$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$WEB_APP"
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

## Step 3 – Show Migration Overview

```bash
# Display App Service migration overview
cat << 'EOF'

Azure App Service Migration Overview:
======================================

Supported Application Types:
  - ASP.NET (Framework and Core)
  - Node.js
  - PHP
  - Python
  - Java
  - Ruby
  - Static HTML/JavaScript

Migration Methods:

1. Azure App Service Migration Assistant:
   - Windows tool for IIS applications
   - Automated assessment and migration
   - ASP.NET and PHP support
   - Download from: https://appmigration.microsoft.com

2. Manual Deployment:
   - ZIP deploy
   - Git deployment
   - FTP upload
   - Container deployment
   - CI/CD pipelines

3. Containerized Migration:
   - Docker container
   - Deploy to Web App for Containers
   - Lift-and-shift with dependencies

Benefits:
  ✓ Fully managed platform (PaaS)
  ✓ Built-in scaling and load balancing
  ✓ SSL certificates included
  ✓ Deployment slots for staging
  ✓ Continuous deployment support
  ✓ Integrated monitoring and diagnostics

Considerations:
  - Application must be web-based
  - No direct file system access (use Azure Storage)
  - No Windows services (use WebJobs/Functions)
  - Session state externalization required
  - Connection string updates needed

EOF
```

---

## Step 4 – Show Assessment Checklist

```bash
# Display pre-migration assessment
cat << 'EOF'

Pre-Migration Assessment Checklist:
====================================

Application Compatibility:
  □ Web application (not desktop or service)
  □ Supported runtime version available
  □ No hard dependencies on OS features
  □ Stateless or externalized state
  □ No file system persistence

Technical Requirements:
  □ .NET Framework 4.7+ or .NET Core/6+
  □ Node.js LTS version
  □ PHP 7.4+, 8.0+
  □ Python 3.7+
  □ Java 8, 11, 17
  □ Database connection strings configurable

Dependencies:
  □ Database compatible (Azure SQL, MySQL, PostgreSQL)
  □ No Windows services required
  □ No COM/DCOM components
  □ No registry dependencies
  □ External cache (Redis) for session state

Security:
  □ Authentication method compatible
  □ SSL certificate available
  □ Secrets in Key Vault or App Settings
  □ Network isolation requirements known

Performance:
  □ Current resource usage documented
  □ Traffic patterns understood
  □ Scaling requirements defined
  □ Always On needed for 24/7 apps

Limitations to Address:
  - No IIS modules (use web.config transforms)
  - No direct file writes (use Blob Storage)
  - No scheduled tasks (use Azure Functions)
  - No machine-wide GAC assemblies
  - Limited custom domain support on Free tier

EOF
```

---

## Step 5 – Create Sample Application

```bash
# Create sample Node.js web application
mkdir -p web-app
cd web-app

# Create package.json
cat > package.json << 'EOF'
{
  "name": "migration-demo-app",
  "version": "2.0.0",
  "description": "Migrated web application",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
EOF

# Create server.js
cat > server.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;

// Middleware
app.use(express.static('public'));
app.use(express.json());

// Environment info
app.get('/api/info', (req, res) => {
  res.json({
    environment: process.env.ENVIRONMENT || 'Production',
    version: '2.0.0',
    hostname: require('os').hostname(),
    platform: process.platform,
    nodeVersion: process.version,
    uptime: process.uptime(),
    timestamp: new Date().toISOString()
  });
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

// Main page
app.get('/', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Migrated Application</title>
      <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
          font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
          background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
          min-height: 100vh;
          display: flex;
          align-items: center;
          justify-content: center;
          padding: 20px;
        }
        .container {
          background: white;
          border-radius: 12px;
          box-shadow: 0 20px 60px rgba(0,0,0,0.3);
          max-width: 600px;
          width: 100%;
          padding: 40px;
        }
        h1 {
          color: #333;
          margin-bottom: 10px;
          font-size: 2em;
        }
        .subtitle {
          color: #666;
          margin-bottom: 30px;
          font-size: 1.1em;
        }
        .info-box {
          background: #f8f9fa;
          border-left: 4px solid #667eea;
          padding: 15px;
          margin: 20px 0;
          border-radius: 4px;
        }
        .feature {
          display: flex;
          align-items: center;
          margin: 15px 0;
        }
        .feature-icon {
          width: 40px;
          height: 40px;
          background: #667eea;
          border-radius: 50%;
          display: flex;
          align-items: center;
          justify-content: center;
          color: white;
          font-weight: bold;
          margin-right: 15px;
        }
        .feature-text { flex: 1; }
        .feature-title {
          font-weight: bold;
          color: #333;
          margin-bottom: 5px;
        }
        .feature-desc {
          color: #666;
          font-size: 0.9em;
        }
        button {
          background: #667eea;
          color: white;
          border: none;
          padding: 12px 30px;
          border-radius: 6px;
          font-size: 1em;
          cursor: pointer;
          margin-top: 20px;
          transition: background 0.3s;
        }
        button:hover { background: #5568d3; }
        #info { margin-top: 20px; }
        .metric {
          display: inline-block;
          margin: 5px 10px 5px 0;
          color: #555;
        }
      </style>
    </head>
    <body>
      <div class="container">
        <h1>🚀 Application Migration Successful</h1>
        <p class="subtitle">Running on Azure App Service</p>
        
        <div class="info-box">
          <strong>Migration Status:</strong> ✅ Complete<br>
          <strong>Platform:</strong> Azure App Service (PaaS)<br>
          <strong>Runtime:</strong> Node.js ${process.version}
        </div>

        <div class="feature">
          <div class="feature-icon">1</div>
          <div class="feature-text">
            <div class="feature-title">Fully Managed Platform</div>
            <div class="feature-desc">No infrastructure to manage</div>
          </div>
        </div>

        <div class="feature">
          <div class="feature-icon">2</div>
          <div class="feature-text">
            <div class="feature-title">Auto-scaling</div>
            <div class="feature-desc">Scale based on demand</div>
          </div>
        </div>

        <div class="feature">
          <div class="feature-icon">3</div>
          <div class="feature-text">
            <div class="feature-title">Built-in SSL</div>
            <div class="feature-desc">Free SSL certificates included</div>
          </div>
        </div>

        <div class="feature">
          <div class="feature-icon">4</div>
          <div class="feature-text">
            <div class="feature-title">Deployment Slots</div>
            <div class="feature-desc">Staging and production environments</div>
          </div>
        </div>

        <button onclick="loadInfo()">Load Environment Info</button>
        <div id="info"></div>
      </div>

      <script>
        async function loadInfo() {
          try {
            const response = await fetch('/api/info');
            const data = await response.json();
            document.getElementById('info').innerHTML = \`
              <div class="info-box">
                <strong>Environment Details:</strong><br>
                <div class="metric">Environment: \${data.environment}</div>
                <div class="metric">Version: \${data.version}</div><br>
                <div class="metric">Hostname: \${data.hostname}</div><br>
                <div class="metric">Platform: \${data.platform}</div>
                <div class="metric">Node: \${data.nodeVersion}</div><br>
                <div class="metric">Uptime: \${Math.floor(data.uptime)}s</div><br>
                <div class="metric">Timestamp: \${data.timestamp}</div>
              </div>
            \`;
          } catch (error) {
            document.getElementById('info').innerHTML = 
              '<div class="info-box" style="border-color: red;">Error loading info</div>';
          }
        }
      </script>
    </body>
    </html>
  `);
});

// Start server
app.listen(port, () => {
  console.log(\`Server running on port \${port}\`);
  console.log(\`Environment: \${process.env.ENVIRONMENT || 'Production'}\`);
});
EOF

cd ..
echo "Sample application created"
```

---

## Step 6 – Create App Service Plan

```bash
# Create App Service Plan with Linux
az appservice plan create \
  --name "$APP_PLAN" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --is-linux \
  --sku S1

echo "App Service Plan created"
```

---

## Step 7 – Show App Service SKUs

```bash
# Display App Service pricing tiers
cat << 'EOF'

App Service Pricing Tiers:
==========================

Free (F1):
  - Shared compute
  - 1 GB disk space
  - 60 CPU minutes/day
  - No custom domains
  - Good for: Development/testing

Shared (D1):
  - Shared compute
  - 1 GB disk space
  - 240 CPU minutes/day
  - Custom domains supported
  - Good for: Low-traffic sites

Basic (B1, B2, B3):
  - Dedicated compute
  - Custom domains and SSL
  - Manual scaling
  - No deployment slots
  - Good for: Development, low-traffic production

Standard (S1, S2, S3):
  - Auto-scaling (up to 10 instances)
  - Deployment slots (5)
  - Daily backups
  - Traffic Manager
  - Good for: Production workloads

Premium (P1v2, P2v2, P3v2, P1v3, P2v3, P3v3):
  - Enhanced performance
  - Deployment slots (20)
  - VNet integration
  - Higher scale (up to 30 instances)
  - Good for: High-scale production

Isolated (I1v2, I2v2, I3v2):
  - Dedicated environment (ASE)
  - Network isolation
  - Internal load balancing
  - Maximum scale
  - Good for: Mission-critical, compliance

Recommendations:
  - Development: Free or B1
  - Staging/Testing: B1 or S1
  - Production: S1 or higher
  - High-traffic: P-series
  - Compliance: Isolated

EOF
```

---

## Step 8 – Create Web App

```bash
# Create Web App with Node.js runtime
az webapp create \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --plan "$APP_PLAN" \
  --runtime "NODE:18-lts"

echo "Web App created"
```

---

## Step 9 – Configure App Settings

```bash
# Set application settings
az webapp config appsettings set \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --settings \
    ENVIRONMENT="Production" \
    NODE_ENV="production" \
    WEBSITE_NODE_DEFAULT_VERSION="18-lts"

echo "App settings configured"
```

---

## Step 10 – Enable Application Insights

```bash
# Create Application Insights
az monitor app-insights component create \
  --app "$APP_INSIGHTS" \
  --location "$LOCATION" \
  --resource-group "$RG_NAME" \
  --application-type web

# Get instrumentation key
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app "$APP_INSIGHTS" \
  --resource-group "$RG_NAME" \
  --query instrumentationKey \
  --output tsv)

echo "$INSTRUMENTATION_KEY"

# Configure Application Insights for Web App
az webapp config appsettings set \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --settings APPINSIGHTS_INSTRUMENTATIONKEY="$INSTRUMENTATION_KEY"

echo "Application Insights enabled"
```

---

## Step 11 – Deploy Application

```bash
# Create deployment package
cd web-app
zip -r ../app.zip . >/dev/null 2>&1
cd ..

# Deploy to App Service
az webapp deployment source config-zip \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --src app.zip

echo "Application deployed"
```

---

## Step 12 – Get Web App URL

```bash
# Get Web App URL
WEB_APP_URL=$(az webapp show \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --query defaultHostName \
  --output tsv)

echo "$WEB_APP_URL"
```

---

## Step 13 – Test Application

```bash
# Test Web App endpoint
echo "Testing application..."
curl -s "https://${WEB_APP_URL}/health" | grep -o '"status":"[^"]*"'

echo "Application tested"
```

---

## Step 14 – Show Deployment Methods

```bash
# Display deployment options
cat << 'EOF'

App Service Deployment Methods:
================================

1. ZIP Deploy (Quick):
   az webapp deployment source config-zip \
     --src app.zip

2. Local Git:
   az webapp deployment source config-local-git
   git remote add azure <git-url>
   git push azure main

3. GitHub Actions (CI/CD):
   - Connect GitHub repository
   - Auto-deploy on push
   - Build and test pipeline

4. Azure DevOps:
   - Azure Pipelines
   - Release management
   - Multi-stage deployments

5. FTP/FTPS:
   - Get credentials from portal
   - Upload files via FTP client
   - Legacy method

6. Container Deploy:
   az webapp create --deployment-container-image-name <image>

7. Visual Studio:
   - Right-click → Publish
   - Integrated deployment
   - Configuration management

8. VS Code:
   - Azure App Service extension
   - One-click deployment
   - Remote debugging

Best Practices:
  ✓ Use CI/CD for production
  ✓ Deploy to staging slot first
  ✓ Run tests before swap
  ✓ Keep deployment packages small
  ✓ Use .deployment file for custom builds

EOF
```

---

## Step 15 – Create Deployment Slot

```bash
# Create staging slot
az webapp deployment slot create \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --slot staging

echo "Staging slot created"
```

---

## Step 16 – Deploy to Staging

```bash
# Deploy application to staging slot
az webapp deployment source config-zip \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --slot staging \
  --src app.zip

echo "Deployed to staging"
```

---

## Step 17 – Get Staging URL

```bash
# Get staging slot URL
STAGING_URL=$(az webapp deployment slot list \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --query "[?name=='staging'].defaultHostName" \
  --output tsv)

echo "$STAGING_URL"
```

---

## Step 18 – Test Staging Slot

```bash
# Test staging environment
echo "Testing staging slot..."
curl -s "https://${STAGING_URL}/health" | grep -o '"status":"[^"]*"'

echo "Staging tested"
```

---

## Step 19 – Show Slot Swap Process

```bash
# Display slot swap information
cat << 'EOF'

Deployment Slot Swap Process:
==============================

Why Use Slots:
  - Zero-downtime deployments
  - Warm-up before production
  - Easy rollback
  - A/B testing capability
  - Traffic routing for canary releases

Swap Process:

1. Deploy to Staging:
   - Upload new version
   - Verify functionality
   - Run smoke tests

2. Warm-up Phase:
   - Azure triggers app initialization
   - Loads app settings
   - Makes HTTP requests to root
   - Ensures app is ready

3. Perform Swap:
   - Atomic operation
   - Swaps hostnames
   - Updates load balancer
   - Takes seconds to complete

4. Verify Production:
   - Check application health
   - Monitor for errors
   - Validate functionality

5. Rollback if Needed:
   - Swap back to previous version
   - Instant rollback
   - No data loss

Swap Command:
  az webapp deployment slot swap \
    --slot staging \
    --resource-group $RG \
    --name $APP

Slot Settings:
  - Sticky settings: Stay with slot
  - Non-sticky: Move with swap
  - Connection strings
  - App settings
  - Handler mappings

EOF
```

---

## Step 20 – Configure Always On

```bash
# Enable Always On to prevent app from idling
az webapp config set \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --always-on true

echo "Always On enabled"
```

---

## Step 21 – Configure Health Check

```bash
# Enable health check monitoring
az webapp config set \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --generic-configurations '{"healthCheckPath":"/health"}'

echo "Health check configured"
```

---

## Step 22 – Show Configuration Options

```bash
# Display App Service configuration
cat << 'EOF'

App Service Configuration Options:
===================================

Application Settings:
  - Environment variables
  - Connection strings (marked as secret)
  - App-specific configuration
  - Azure Key Vault references

Runtime Configuration:
  - Stack and version
  - Startup command
  - Platform bitness (32/64)
  - HTTP version
  - Always On
  - ARR affinity

Security:
  - HTTPS only
  - Minimum TLS version
  - Client certificates
  - Authentication/authorization
  - Managed identity

Networking:
  - VNet integration
  - Private endpoints
  - Hybrid connections
  - Access restrictions (IP filtering)

Monitoring:
  - Application Insights
  - Diagnostic logs
  - Log streaming
  - Metrics

Deployment:
  - Deployment slots
  - Deployment credentials
  - Deployment center
  - Continuous deployment

Scaling:
  - Manual scale
  - Auto-scale rules
  - Scale-out (instances)
  - Scale-up (tier change)

Backup:
  - Scheduled backups
  - On-demand backups
  - Backup retention
  - Restore options

EOF
```

---

## Step 23 – Enable Diagnostic Logging

```bash
# Enable application logging
az webapp log config \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --application-logging filesystem \
  --detailed-error-messages true \
  --failed-request-tracing true \
  --web-server-logging filesystem

echo "Diagnostic logging enabled"
```

---

## Step 24 – Configure Auto-scaling

```bash
# Create autoscale rule based on CPU
az monitor autoscale create \
  --resource-group "$RG_NAME" \
  --resource "$APP_PLAN" \
  --resource-type "Microsoft.Web/serverfarms" \
  --name autoscale-cpu \
  --min-count 1 \
  --max-count 3 \
  --count 1

# Add scale-out rule (CPU > 70%)
az monitor autoscale rule create \
  --resource-group "$RG_NAME" \
  --autoscale-name autoscale-cpu \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 1

# Add scale-in rule (CPU < 30%)
az monitor autoscale rule create \
  --resource-group "$RG_NAME" \
  --autoscale-name autoscale-cpu \
  --condition "Percentage CPU < 30 avg 5m" \
  --scale in 1

echo "Auto-scaling configured"
```

---

## Step 25 – Show Migration Best Practices

```bash
# Display best practices
cat << 'EOF'

Web App Migration Best Practices:
==================================

Pre-Migration:
  ✓ Run migration assistant tool
  ✓ Test application locally
  ✓ Review dependencies
  ✓ Externalize configuration
  ✓ Document current architecture

Code Changes:
  ✓ Use environment variables for config
  ✓ Externalize session state (Redis)
  ✓ Use Azure Storage for files
  ✓ Update database connection strings
  ✓ Handle graceful shutdown

Configuration:
  ✓ Set all app settings in portal
  ✓ Use Key Vault for secrets
  ✓ Enable managed identity
  ✓ Configure custom domains
  ✓ Set up SSL certificates

Deployment:
  ✓ Use deployment slots
  ✓ Deploy to staging first
  ✓ Warm up application
  ✓ Verify functionality
  ✓ Swap to production

Post-Migration:
  ✓ Monitor application metrics
  ✓ Check error logs
  ✓ Validate all features
  ✓ Test under load
  ✓ Optimize performance

Security:
  ✓ Enable HTTPS only
  ✓ Use TLS 1.2+
  ✓ Configure authentication
  ✓ Set up access restrictions
  ✓ Enable managed identity

Performance:
  ✓ Enable Always On
  ✓ Configure auto-scaling
  ✓ Use CDN for static content
  ✓ Enable response compression
  ✓ Optimize database queries

Reliability:
  ✓ Set up health checks
  ✓ Configure auto-heal
  ✓ Enable backup
  ✓ Plan disaster recovery
  ✓ Use multiple regions

EOF
```

---

## Step 26 – Show Troubleshooting Tips

```bash
# Display troubleshooting guide
cat << 'EOF'

App Service Troubleshooting:
============================

Application Won't Start:

Problem: Application error on startup
  ✓ Check application logs
  ✓ Verify runtime version
  ✓ Check package.json/requirements.txt
  ✓ Validate startup command
  ✓ Review app settings

Problem: Timeout during deployment
  ✓ Increase deployment timeout
  ✓ Reduce package size
  ✓ Use build in production
  ✓ Check network connectivity

Performance Issues:

Problem: Slow response times
  ✓ Enable Application Insights
  ✓ Check App Service Plan tier
  ✓ Review database performance
  ✓ Optimize queries
  ✓ Enable CDN for static files

Problem: High CPU usage
  ✓ Scale up to higher tier
  ✓ Enable auto-scaling
  ✓ Profile application
  ✓ Optimize code
  ✓ Check for infinite loops

Configuration Issues:

Problem: Connection string errors
  ✓ Verify connection strings in portal
  ✓ Check firewall rules
  ✓ Validate credentials
  ✓ Test connectivity
  ✓ Review SSL requirements

Problem: Missing environment variables
  ✓ Add to app settings
  ✓ Check slot settings
  ✓ Restart application
  ✓ Verify variable names

Deployment Issues:

Problem: Deployment fails
  ✓ Check deployment logs
  ✓ Validate credentials
  ✓ Review file permissions
  ✓ Check disk quota
  ✓ Verify deployment method

Problem: Old version still running
  ✓ Restart web app
  ✓ Clear browser cache
  ✓ Check deployment slot
  ✓ Verify swap completed

Diagnostic Tools:
  - Kudu console (Advanced Tools)
  - Log streaming
  - App Insights
  - Diagnose and solve problems
  - Process explorer

EOF
```

---

## Step 27 – Show Custom Domain Setup

```bash
# Display custom domain configuration
cat << EOF

Custom Domain Configuration:
=============================

Prerequisites:
  - Domain name registered
  - DNS provider access
  - App Service Plan: Basic or higher

Steps to Add Custom Domain:

1. Get verification ID:
   VERIFY_ID=\$(az webapp show \\
     --name "$WEB_APP" \\
     --resource-group "$RG_NAME" \\
     --query customDomainVerificationId \\
     --output tsv)

2. Add DNS records:
   
   For apex domain (example.com):
     Type: A
     Host: @
     Value: <App IP address>
     
     Type: TXT
     Host: asuid
     Value: \$VERIFY_ID

   For subdomain (www.example.com):
     Type: CNAME
     Host: www
     Value: $WEB_APP.azurewebsites.net
     
     Type: TXT
     Host: asuid.www
     Value: \$VERIFY_ID

3. Add custom domain:
   az webapp config hostname add \\
     --webapp-name "$WEB_APP" \\
     --resource-group "$RG_NAME" \\
     --hostname www.example.com

4. Enable SSL:
   az webapp config ssl bind \\
     --name "$WEB_APP" \\
     --resource-group "$RG_NAME" \\
     --certificate-thumbprint <thumbprint> \\
     --ssl-type SNI

Free Managed Certificate:
  - Available for custom domains
  - Automatically renewed
  - SNI SSL only
  - Enabled in portal

EOF
```

---

## Step 28 – View Application Logs

```bash
# Stream live application logs
echo "Streaming logs for 30 seconds..."
timeout 30s az webapp log tail \
  --name "$WEB_APP" \
  --resource-group "$RG_NAME" 2>/dev/null || echo "Log streaming completed"
```

---

## Step 29 – Show Metrics

```bash
# Get Web App metrics
az monitor metrics list \
  --resource "$WEB_APP" \
  --resource-group "$RG_NAME" \
  --resource-type "Microsoft.Web/sites" \
  --metric-names "CpuTime" "Requests" "Http5xx" \
  --output table 2>/dev/null || echo "Metrics available in Azure Monitor"
```

---

## Step 30 – Show Migration Checklist

```bash
# Display final migration checklist
cat << 'EOF'

Post-Migration Validation Checklist:
=====================================

Functionality:
  □ Home page loads correctly
  □ All routes/pages accessible
  □ Forms submit successfully
  □ File uploads work
  □ Downloads function properly
  □ Search functionality works
  □ Authentication successful
  □ Authorization rules applied

Performance:
  □ Response times acceptable
  □ Database queries optimized
  □ Static files cached
  □ CDN configured (if needed)
  □ Compression enabled
  □ Always On enabled

Security:
  □ HTTPS enforced
  □ SSL certificate valid
  □ Authentication configured
  □ Secrets in Key Vault
  □ Access restrictions set
  □ Managed identity enabled

Monitoring:
  □ Application Insights working
  □ Logs being generated
  □ Metrics visible
  □ Alerts configured
  □ Health checks passing

Configuration:
  □ All app settings migrated
  □ Connection strings updated
  □ Environment variables set
  □ Feature flags configured
  □ Scaling rules defined

Backup & DR:
  □ Backup enabled
  □ Restore tested
  □ Deployment slots configured
  □ Rollback plan documented
  □ Multi-region considered

Business Validation:
  □ Stakeholders notified
  □ Users can access
  □ No critical errors
  □ Performance acceptable
  □ All features working

Decommission Old Environment:
  □ Wait period completed (2-4 weeks)
  □ Backup old environment
  □ Document lessons learned
  □ Update documentation
  □ Remove old resources

EOF
```

---

## Step 31 – Show Cost Optimization

```bash
# Display cost optimization tips
cat << 'EOF'

App Service Cost Optimization:
===============================

Right-Sizing:
  ✓ Start with lower tier (S1)
  ✓ Monitor resource usage
  ✓ Scale up only when needed
  ✓ Use Serverless for variable loads

Auto-scaling:
  ✓ Scale based on metrics
  ✓ Set appropriate min/max
  ✓ Use schedule-based scaling
  ✓ Scale in during off-hours

Reserved Instances:
  ✓ 1-year: Save 30%
  ✓ 3-year: Save 55%
  ✓ Good for production workloads

Development Environments:
  ✓ Use Free or Basic tier
  ✓ Auto-shutdown during off-hours
  ✓ Share App Service Plans
  ✓ Delete when not needed

Deployment Slots:
  ✓ Only available on Standard+
  ✓ Consider if needed
  ✓ Each slot consumes resources
  ✓ Use CI/CD instead for dev

Storage:
  ✓ Use Azure Storage for files
  ✓ Enable CDN for static content
  ✓ Compress responses
  ✓ Optimize images

Monitoring:
  ✓ Application Insights sampling
  ✓ Log retention policies
  ✓ Delete old logs
  ✓ Use workspace-based App Insights

Multi-tenancy:
  ✓ Host multiple apps on one plan
  ✓ Share resources efficiently
  ✓ Separate by environment
  ✓ Monitor resource usage

Estimated Costs (Australia East):
  Free: $0/month
  Basic B1: $55/month
  Standard S1: $100/month
  Premium P1v3: $220/month

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
rm -rf web-app
rm -f app.zip

echo "Cleanup complete"
```

---

## Summary

You assessed web application compatibility for Azure App Service migration, created App Service plan with Linux runtime support, deployed Node.js web application using ZIP deployment method, configured application settings and environment variables, enabled Application Insights for monitoring and diagnostics, created deployment slot for staging environment, tested blue-green deployment with slot swap capability, configured Always On and health check endpoints, enabled auto-scaling based on CPU metrics, explored diagnostic logging and live log streaming, reviewed deployment methods including ZIP, Git, GitHub Actions, and Azure DevOps, configured custom domain and SSL certificate options, implemented best practices for security, performance, and reliability, and learned troubleshooting techniques for common App Service issues including startup failures, performance problems, and configuration errors.
