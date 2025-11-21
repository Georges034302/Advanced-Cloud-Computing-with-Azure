# Lab 1.D: Deploy Python App to Azure App Service

## Overview
This lab demonstrates deploying a Python Flask web application to Azure App Service, a fully managed platform-as-a-service (PaaS). You'll create an App Service plan, deploy a Flask application with database connectivity, configure application settings, and enable monitoring.

---

## Objectives
- Create Azure App Service Plan and Web App
- Deploy Python Flask application to App Service
- Configure application settings and environment variables
- Enable Application Insights for monitoring
- Configure custom domain and SSL (optional)
- Test and scale the application
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Git installed and configured
- Python 3.8+ installed locally
- Basic Flask knowledge
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="eastus"
RG_NAME="rg-lab1d-appservice"
APP_SERVICE_PLAN="plan-lab1d"
WEB_APP_NAME="webapp-lab1d-$RANDOM"
# App Service names must be globally unique

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "APP_SERVICE_PLAN=$APP_SERVICE_PLAN"
echo "WEB_APP_NAME=$WEB_APP_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for App Service resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create App Service Plan

```bash
# Create Linux App Service Plan (B1 tier)
az appservice plan create \
  --name "$APP_SERVICE_PLAN" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --is-linux \
  --sku B1

# Verify plan creation
az appservice plan show \
  --name "$APP_SERVICE_PLAN" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, Sku:sku.name, Kind:kind, NumberOfSites:numberOfSites}" \
  --output table
```

---

## Step 4 – Create Web App

```bash
# Create Web App with Python 3.11 runtime
az webapp create \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --plan "$APP_SERVICE_PLAN" \
  --runtime "PYTHON:3.11" \
  --output table

# Get Web App URL
WEB_APP_URL=$(az webapp show \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --query "defaultHostName" \
  --output tsv)

echo "Web App URL: https://$WEB_APP_URL"
```

---

## Step 5 – Create Flask Application Locally

```bash
# Create application directory
APP_DIR=$(mktemp -d)
cd "$APP_DIR"

# Initialize git repository
git init

# Create Flask application
cat > app.py << 'EOF'
from flask import Flask, jsonify, render_template_string
import os
import socket
from datetime import datetime

app = Flask(__name__)

# HTML template
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Azure App Service - Python Flask</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
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
            box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
            padding: 40px;
            max-width: 800px;
            width: 100%;
        }
        h1 {
            color: #0078d4;
            margin-bottom: 10px;
            font-size: 2.5em;
        }
        .subtitle {
            color: #666;
            margin-bottom: 30px;
            font-size: 1.2em;
        }
        .info-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 20px;
            margin: 30px 0;
        }
        .info-card {
            background: #f8f9fa;
            padding: 20px;
            border-radius: 8px;
            border-left: 4px solid #0078d4;
        }
        .info-card h3 {
            color: #0078d4;
            font-size: 0.9em;
            text-transform: uppercase;
            margin-bottom: 10px;
        }
        .info-card p {
            color: #333;
            font-size: 1.1em;
            font-weight: bold;
        }
        .endpoints {
            background: #2d3748;
            color: white;
            padding: 20px;
            border-radius: 8px;
            margin: 20px 0;
        }
        .endpoints h2 {
            margin-bottom: 15px;
            color: #00bcf2;
        }
        .endpoints ul {
            list-style: none;
        }
        .endpoints li {
            margin: 10px 0;
            padding: 10px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 4px;
        }
        .endpoints a {
            color: #00bcf2;
            text-decoration: none;
        }
        .endpoints a:hover {
            text-decoration: underline;
        }
        .status {
            background: #d4edda;
            color: #155724;
            padding: 15px;
            border-radius: 8px;
            border: 1px solid #c3e6cb;
            margin: 20px 0;
        }
        .footer {
            text-align: center;
            margin-top: 30px;
            padding-top: 20px;
            border-top: 1px solid #ddd;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Azure App Service</h1>
        <p class="subtitle">Python Flask Application (PaaS)</p>
        
        <div class="status">
            ✅ Application is running successfully on Azure App Service
        </div>

        <div class="info-grid">
            <div class="info-card">
                <h3>Hostname</h3>
                <p>{{ hostname }}</p>
            </div>
            <div class="info-card">
                <h3>Environment</h3>
                <p>{{ environment }}</p>
            </div>
            <div class="info-card">
                <h3>Python Version</h3>
                <p>{{ python_version }}</p>
            </div>
            <div class="info-card">
                <h3>Timestamp</h3>
                <p>{{ timestamp }}</p>
            </div>
        </div>

        <div class="endpoints">
            <h2>API Endpoints</h2>
            <ul>
                <li><a href="/api/health">/api/health</a> - Health check endpoint</li>
                <li><a href="/api/info">/api/info</a> - Application information</li>
                <li><a href="/api/env">/api/env</a> - Environment variables</li>
            </ul>
        </div>

        <div class="footer">
            <p>Lab 1.D - Azure Cloud Computing</p>
            <p>Deployed to Azure App Service</p>
        </div>
    </div>
</body>
</html>
'''

@app.route('/')
def home():
    return render_template_string(
        HTML_TEMPLATE,
        hostname=socket.gethostname(),
        environment=os.getenv('ENVIRONMENT', 'production'),
        python_version=os.sys.version.split()[0],
        timestamp=datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    )

@app.route('/api/health')
def health():
    return jsonify({
        'status': 'healthy',
        'service': 'flask-app',
        'timestamp': datetime.now().isoformat()
    }), 200

@app.route('/api/info')
def info():
    return jsonify({
        'application': 'Azure App Service Demo',
        'version': '1.0.0',
        'runtime': 'Python Flask',
        'hostname': socket.gethostname(),
        'platform': 'Azure App Service (PaaS)',
        'python_version': os.sys.version,
        'environment': os.getenv('ENVIRONMENT', 'production')
    }), 200

@app.route('/api/env')
def env_vars():
    # Only show non-sensitive environment variables
    safe_vars = {
        'WEBSITE_SITE_NAME': os.getenv('WEBSITE_SITE_NAME', 'N/A'),
        'WEBSITE_RESOURCE_GROUP': os.getenv('WEBSITE_RESOURCE_GROUP', 'N/A'),
        'WEBSITE_INSTANCE_ID': os.getenv('WEBSITE_INSTANCE_ID', 'N/A'),
        'WEBSITE_HOSTNAME': os.getenv('WEBSITE_HOSTNAME', 'N/A'),
        'ENVIRONMENT': os.getenv('ENVIRONMENT', 'production')
    }
    return jsonify(safe_vars), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000, debug=False)
EOF

# Create requirements.txt
cat > requirements.txt << 'EOF'
Flask==3.0.0
Werkzeug==3.0.1
gunicorn==21.2.0
EOF

# Create startup script for App Service
cat > startup.sh << 'EOF'
#!/bin/bash
gunicorn --bind=0.0.0.0:8000 --workers=2 --timeout=600 app:app
EOF

chmod +x startup.sh

# Create .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
venv/
.env
EOF

# List created files
echo "Application files created:"
ls -lh
```

---

## Step 6 – Test Application Locally (Optional)

```bash
# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run Flask app locally
# python app.py
# (Press Ctrl+C to stop)

# Deactivate virtual environment
deactivate
```

---

## Step 7 – Deploy Application to App Service

```bash
# Add all files to git
git add .
git config user.email "lab@example.com"
git config user.name "Lab User"
git commit -m "Initial Flask application"

# Configure deployment source
az webapp deployment source config-local-git \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --query url \
  --output tsv

# Get deployment credentials
DEPLOY_USER=$(az webapp deployment list-publishing-credentials \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --query publishingUserName \
  --output tsv)

DEPLOY_PASS=$(az webapp deployment list-publishing-credentials \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --query publishingPassword \
  --output tsv)

# Get Git URL
GIT_URL=$(az webapp deployment source config-local-git \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --query url \
  --output tsv)

echo "Deployment URL: $GIT_URL"

# Add Azure as remote and push
git remote add azure "$GIT_URL"

# Push to Azure (will prompt for password)
echo "Pushing to Azure App Service..."
echo "Username: $DEPLOY_USER"
echo "Password: (use deployment password)"

# Alternative: Use deployment credentials in URL
GIT_URL_WITH_CREDS=$(echo "$GIT_URL" | sed "s|https://|https://$DEPLOY_USER:$DEPLOY_PASS@|")
git remote set-url azure "$GIT_URL_WITH_CREDS"
git push azure main

echo "⏳ Deployment in progress (2-3 minutes)..."
```

---

## Step 8 – Configure Application Settings

```bash
# Set custom environment variable
az webapp config appsettings set \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --settings ENVIRONMENT="production" \
  --output table

# Configure startup command
az webapp config set \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --startup-file "startup.sh"

# Enable detailed error logging
az webapp log config \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --application-logging filesystem \
  --detailed-error-messages true \
  --failed-request-tracing true \
  --web-server-logging filesystem

# Restart web app to apply changes
az webapp restart \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME"

echo "⏳ Waiting for restart (30 seconds)..."
sleep 30
```

---

## Step 9 – Test Deployed Application

```bash
# Test main endpoint
echo "Testing home page..."
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" "https://$WEB_APP_URL"

# Test health endpoint
echo -e "\nTesting health endpoint..."
curl -s "https://$WEB_APP_URL/api/health" | python3 -m json.tool

# Test info endpoint
echo -e "\nTesting info endpoint..."
curl -s "https://$WEB_APP_URL/api/info" | python3 -m json.tool

# Open in browser
echo -e "\n🌐 Opening application in browser..."
if command -v xdg-open &> /dev/null; then
  xdg-open "https://$WEB_APP_URL" &
elif [ -n "$BROWSER" ]; then
  "$BROWSER" "https://$WEB_APP_URL" &
fi
```

---

## Step 10 – View Application Logs

```bash
# Stream application logs (Ctrl+C to stop)
echo "Streaming logs (press Ctrl+C to stop)..."
az webapp log tail \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME"

# Alternative: Download logs
# az webapp log download \
#   --name "$WEB_APP_NAME" \
#   --resource-group "$RG_NAME" \
#   --log-file "webapp-logs.zip"
```

---

## Step 11 – Enable Application Insights

```bash
# Create Application Insights component
APPINSIGHTS_NAME="appi-${WEB_APP_NAME}"

az monitor app-insights component create \
  --app "$APPINSIGHTS_NAME" \
  --location "$LOCATION" \
  --resource-group "$RG_NAME" \
  --application-type web \
  --kind web

# Get instrumentation key
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app "$APPINSIGHTS_NAME" \
  --resource-group "$RG_NAME" \
  --query instrumentationKey \
  --output tsv)

# Configure App Service to use Application Insights
az webapp config appsettings set \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --settings APPINSIGHTS_INSTRUMENTATIONKEY="$INSTRUMENTATION_KEY" \
  --output table

echo "✅ Application Insights enabled"
echo "Instrumentation Key: $INSTRUMENTATION_KEY"
```

---

## Step 12 – Scale Application

```bash
# Scale up: Change App Service Plan tier
az appservice plan update \
  --name "$APP_SERVICE_PLAN" \
  --resource-group "$RG_NAME" \
  --sku S1

# Scale out: Add more instances
az appservice plan update \
  --name "$APP_SERVICE_PLAN" \
  --resource-group "$RG_NAME" \
  --number-of-workers 2

# Verify scaling
az appservice plan show \
  --name "$APP_SERVICE_PLAN" \
  --resource-group "$RG_NAME" \
  --query "{Sku:sku.name, Instances:numberOfWorkers}" \
  --output table

# Scale back to B1 to save costs
az appservice plan update \
  --name "$APP_SERVICE_PLAN" \
  --resource-group "$RG_NAME" \
  --sku B1 \
  --number-of-workers 1
```

---

## Step 13 – Configure Custom Domain (Optional)

```bash
# List current hostnames
az webapp show \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --query hostNames \
  --output table

# Add custom domain (requires domain ownership)
# az webapp config hostname add \
#   --webapp-name "$WEB_APP_NAME" \
#   --resource-group "$RG_NAME" \
#   --hostname "www.yourdomain.com"

# Bind SSL certificate
# az webapp config ssl bind \
#   --name "$WEB_APP_NAME" \
#   --resource-group "$RG_NAME" \
#   --certificate-thumbprint <thumbprint> \
#   --ssl-type SNI
```

---

## Step 14 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Verify deletion initiated
az group list --query "[?name=='$RG_NAME']" --output table

# Remove application directory
cd ~
rm -rf "$APP_DIR"

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Azure App Service Plan (Linux-based)
- Python Flask web application
- Managed PaaS deployment with auto-scaling
- Application Insights monitoring
- Git-based deployment pipeline

**Architecture:**
```
Git Push → Azure App Service → Kudu Build → Flask App → HTTPS Endpoint
                              ↓
                        Application Insights
```

**Key Components:**
- **App Service Plan**: Compute resources for web apps
- **Web App**: Managed PaaS hosting environment
- **Flask Application**: Python web framework
- **Gunicorn**: Production WSGI server
- **Application Insights**: Monitoring and telemetry
- **Git Deployment**: Source control integration

**What You Learned:**
- Create App Service plans and web apps
- Deploy Python applications to App Service
- Configure application settings and environment variables
- Enable logging and monitoring with Application Insights
- Scale applications vertically and horizontally
- Use Git-based deployment workflow

---

## Best Practices

**Application Development:**
- Use production WSGI server (Gunicorn, uWSGI)
- Never commit secrets to Git repository
- Use environment variables for configuration
- Implement health check endpoints
- Enable detailed logging for troubleshooting

**Deployment:**
- Use deployment slots for staging/production
- Test in staging before swapping to production
- Enable auto-swap for zero-downtime deployments
- Use CI/CD pipelines for automation
- Tag releases with version numbers

**Security:**
- Enable HTTPS only (redirect HTTP to HTTPS)
- Use managed identities for Azure resource access
- Store secrets in Azure Key Vault
- Implement authentication (Azure AD, OAuth)
- Keep Python runtime and dependencies updated

**Performance:**
- Enable Application Insights for monitoring
- Use caching for frequently accessed data
- Optimize database queries
- Enable compression for responses
- Configure appropriate scaling rules

**Cost Optimization:**
- Use B-series plans for dev/test
- Scale down during off-hours
- Use consumption-based pricing when possible
- Monitor usage with Azure Cost Management
- Delete unused deployment slots

---

## Production Enhancements

**1. Enable Deployment Slots**
```bash
# Create staging slot
az webapp deployment slot create \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --slot staging

# Deploy to staging
git push azure-staging main

# Swap staging to production
az webapp deployment slot swap \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --slot staging
```

**2. Configure Auto-Scaling**
```bash
# Create autoscale rule
az monitor autoscale create \
  --resource-group "$RG_NAME" \
  --resource "$APP_SERVICE_PLAN" \
  --resource-type Microsoft.Web/serverfarms \
  --name "autoscale-${APP_SERVICE_PLAN}" \
  --min-count 1 \
  --max-count 5 \
  --count 1

# Add scale-out rule
az monitor autoscale rule create \
  --resource-group "$RG_NAME" \
  --autoscale-name "autoscale-${APP_SERVICE_PLAN}" \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 1
```

**3. Enable Managed Identity**
```bash
# Enable system-assigned managed identity
az webapp identity assign \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME"

# Get identity principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --query principalId \
  --output tsv)

echo "Managed Identity Principal ID: $PRINCIPAL_ID"
```

**4. Add Database Connection**
```python
# Update app.py to connect to Azure SQL or PostgreSQL
from flask_sqlalchemy import SQLAlchemy

app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv('DATABASE_URL')
db = SQLAlchemy(app)

# Configure connection string in App Settings
az webapp config connection-string set \
  --name "$WEB_APP_NAME" \
  --resource-group "$RG_NAME" \
  --settings DATABASE_URL="your-connection-string" \
  --connection-string-type SQLAzure
```

---

## Troubleshooting

**Application not starting:**
- Check application logs: `az webapp log tail`
- Verify requirements.txt has all dependencies
- Confirm startup.sh has correct permissions
- Check Python version compatibility
- Review deployment logs in Kudu console

**Git push fails:**
- Verify deployment credentials are correct
- Check Git remote URL is correct
- Ensure repository is initialized
- Try resetting deployment credentials
- Check Azure Portal deployment center

**Application errors (500):**
- Enable detailed error messages
- Check Flask app configuration
- Review application logs
- Test locally before deploying
- Verify all required environment variables

**Slow performance:**
- Enable Application Insights profiling
- Check App Service Plan tier and scaling
- Optimize database queries
- Enable caching where appropriate
- Review application code for bottlenecks

**Cannot access endpoints:**
- Verify application is running
- Check firewall and IP restrictions
- Confirm routes are defined correctly
- Test with curl or Postman
- Review network security settings

---

## Additional Resources

- [Azure App Service Documentation](https://docs.microsoft.com/azure/app-service/)
- [Deploy Python to App Service](https://docs.microsoft.com/azure/app-service/quickstart-python)
- [App Service Pricing](https://azure.microsoft.com/pricing/details/app-service/)
- [Application Insights for Python](https://docs.microsoft.com/azure/azure-monitor/app/opencensus-python)
- [Flask Documentation](https://flask.palletsprojects.com/)
