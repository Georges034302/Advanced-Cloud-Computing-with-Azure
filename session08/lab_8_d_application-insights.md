# Lab 8.D: Application Insights

## Objectives
- Create Application Insights resource
- Deploy web application
- Configure application monitoring
- Generate application traffic
- Query telemetry data
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
RG_NAME="rg-lab8d-insights"
APP_INSIGHTS="appinsights-monitoring"
PLAN_NAME="plan-insights"
WEBAPP_NAME="webapp-insights-$RANDOM"
WORKSPACE_NAME="law-insights"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$WEBAPP_NAME"
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

## Step 3 – Create Log Analytics Workspace

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --workspace-name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku PerGB2018

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --workspace-name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$WORKSPACE_ID"
```

---

## Step 4 – Create Application Insights

```bash
# Create Application Insights resource
az monitor app-insights component create \
  --app "$APP_INSIGHTS" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --workspace "$WORKSPACE_ID" \
  --application-type web

# Get instrumentation key
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app "$APP_INSIGHTS" \
  --resource-group "$RG_NAME" \
  --query instrumentationKey \
  --output tsv)

echo "$INSTRUMENTATION_KEY"

# Get connection string
CONNECTION_STRING=$(az monitor app-insights component show \
  --app "$APP_INSIGHTS" \
  --resource-group "$RG_NAME" \
  --query connectionString \
  --output tsv)

echo "Application Insights created"
```

---

## Step 5 – Create App Service Plan

```bash
# Create App Service plan
az appservice plan create \
  --name "$PLAN_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku B1 \
  --is-linux

echo "App Service plan created"
```

---

## Step 6 – Create Web App

```bash
# Create web app
az webapp create \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --plan "$PLAN_NAME" \
  --runtime "NODE:18-lts"

echo "Web app created"
```

---

## Step 7 – Configure Application Insights

```bash
# Enable Application Insights for web app
az webapp config appsettings set \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --settings \
    APPINSIGHTS_INSTRUMENTATIONKEY="$INSTRUMENTATION_KEY" \
    APPLICATIONINSIGHTS_CONNECTION_STRING="$CONNECTION_STRING" \
    ApplicationInsightsAgent_EXTENSION_VERSION="~3"

echo "Application Insights configured"
```

---

## Step 8 – Create Application Code

```bash
# Create application directory
mkdir -p webapp-insights
cd webapp-insights

# Create package.json
cat > package.json << 'EOF'
{
  "name": "webapp-insights",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "applicationinsights": "^2.9.0"
  }
}
EOF

# Create server with Application Insights
cat > server.js << 'EOF'
const appInsights = require('applicationinsights');
appInsights.setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING)
  .setAutoCollectRequests(true)
  .setAutoCollectPerformance(true)
  .setAutoCollectExceptions(true)
  .setAutoCollectDependencies(true)
  .start();

const client = appInsights.defaultClient;
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;

let requestCount = 0;

app.get('/', (req, res) => {
  requestCount++;
  client.trackEvent({ name: 'HomePageVisit' });
  res.send(`
    <html>
      <head><title>App Insights Demo</title></head>
      <body>
        <h1>Application Insights Demo</h1>
        <p>Request Count: ${requestCount}</p>
        <ul>
          <li><a href="/api/data">Get Data</a></li>
          <li><a href="/api/slow">Slow Endpoint</a></li>
          <li><a href="/api/error">Error Endpoint</a></li>
        </ul>
      </body>
    </html>
  `);
});

app.get('/api/data', (req, res) => {
  client.trackEvent({ name: 'DataAPICall' });
  res.json({ message: 'Data retrieved successfully', timestamp: new Date() });
});

app.get('/api/slow', async (req, res) => {
  const start = Date.now();
  client.trackEvent({ name: 'SlowAPICall' });
  await new Promise(resolve => setTimeout(resolve, 2000));
  const duration = Date.now() - start;
  client.trackMetric({ name: 'SlowAPIResponseTime', value: duration });
  res.json({ message: 'Slow response', duration });
});

app.get('/api/error', (req, res) => {
  client.trackEvent({ name: 'ErrorAPICall' });
  try {
    throw new Error('Simulated error for testing');
  } catch (error) {
    client.trackException({ exception: error });
    res.status(500).json({ error: error.message });
  }
});

app.listen(port, () => {
  console.log('Server running on port ' + port);
  client.trackEvent({ name: 'ServerStarted' });
});
EOF

echo "Application code created"
```

---

## Step 9 – Deploy Application

```bash
# Create deployment package
zip -r app.zip .

# Deploy to web app
az webapp deployment source config-zip \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --src app.zip

cd ..
echo "Application deployed"
```

---

## Step 10 – Get Web App URL

```bash
# Get web app URL
WEBAPP_URL=$(az webapp show \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG_NAME" \
  --query defaultHostName \
  --output tsv)

echo "$WEBAPP_URL"

# Wait for deployment
echo "Waiting for application to start..."
sleep 60
```

---

## Step 11 – Generate Application Traffic

```bash
# Generate normal traffic
echo "Generating normal traffic:"
for i in {1..10}; do
  curl -s "https://${WEBAPP_URL}/" > /dev/null && echo "Request $i completed"
  sleep 1
done

# Call data API
echo "Calling data API:"
for i in {1..5}; do
  curl -s "https://${WEBAPP_URL}/api/data" | python3 -m json.tool
  sleep 1
done

# Call slow API
echo "Calling slow API:"
for i in {1..3}; do
  curl -s "https://${WEBAPP_URL}/api/slow" | python3 -m json.tool
done

# Generate errors
echo "Generating errors:"
for i in {1..5}; do
  curl -s "https://${WEBAPP_URL}/api/error"
  sleep 1
done
```

---

## Step 12 – Wait for Telemetry

```bash
# Wait for telemetry to be ingested
echo "Waiting for telemetry ingestion..."
sleep 120
```

---

## Step 13 – Validate Query Requests

```bash
# Get workspace customer ID
WORKSPACE_CUSTOMER_ID=$(az monitor log-analytics workspace show \
  --workspace-name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --query customerId \
  --output tsv)

echo "$WORKSPACE_CUSTOMER_ID"

# Query requests
az monitor app-insights query \
  --app "$APP_INSIGHTS" \
  --resource-group "$RG_NAME" \
  --analytics-query "requests | where timestamp > ago(1h) | summarize Count=count() by name | order by Count desc" \
  --output table
```

---

## Step 14 – Query Exceptions

```bash
# Query exceptions
az monitor app-insights query \
  --app "$APP_INSIGHTS" \
  --resource-group "$RG_NAME" \
  --analytics-query "exceptions | where timestamp > ago(1h) | project timestamp, type, outerMessage | order by timestamp desc" \
  --output table
```

---

## Step 15 – Query Performance

```bash
# Query request duration
az monitor app-insights query \
  --app "$APP_INSIGHTS" \
  --resource-group "$RG_NAME" \
  --analytics-query "requests | where timestamp > ago(1h) | summarize AvgDuration=avg(duration), MaxDuration=max(duration) by name" \
  --output table
```

---

## Step 16 – Query Custom Events

```bash
# Query custom events
az monitor app-insights query \
  --app "$APP_INSIGHTS" \
  --resource-group "$RG_NAME" \
  --analytics-query "customEvents | where timestamp > ago(1h) | summarize Count=count() by name | order by Count desc" \
  --output table
```

---

## Step 17 – Query Custom Metrics

```bash
# Query custom metrics
az monitor app-insights query \
  --app "$APP_INSIGHTS" \
  --resource-group "$RG_NAME" \
  --analytics-query "customMetrics | where timestamp > ago(1h) | project timestamp, name, value | order by timestamp desc" \
  --output table
```

---

## Step 18 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf webapp-insights

echo "Cleanup complete"
```

---

## Summary

You created an Application Insights resource, deployed a Node.js web application with Application Insights SDK integration, generated various types of telemetry including requests, exceptions, custom events and metrics, and queried telemetry data using Log Analytics.
