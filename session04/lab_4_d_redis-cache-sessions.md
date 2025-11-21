# Lab 4.D: Redis Cache for Session Management

## Overview
This lab demonstrates Azure Cache for Redis for managing web application sessions. You'll create a Redis cache, deploy a simple Node.js web app, configure Redis session storage, and validate session persistence across requests.

---

## Objectives
- Create Azure Cache for Redis
- Deploy Node.js web application
- Configure Redis for session storage
- Test session persistence
- Monitor Redis cache metrics
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Node.js and npm installed
- Azure subscription with contributor access
- Basic web development knowledge
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab4d-redis"
REDIS_NAME="redis-lab4d-$RANDOM"
APP_NAME="webapp-redis-$RANDOM"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "REDIS_NAME=$REDIS_NAME"
echo "APP_NAME=$APP_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for Redis and web app
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Azure Cache for Redis

```bash
# Create Redis cache (Basic tier, C0 size)
az redis create \
  --name "$REDIS_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Basic \
  --vm-size C0 \
  --enable-non-ssl-port false

echo "⏳ Redis cache creation takes 15-20 minutes"
echo "You can continue to next steps while it provisions"

# Check provisioning status
az redis show \
  --name "$REDIS_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Status:provisioningState, Tier:sku.name}" \
  --output table
```

---

## Step 4 – Wait for Redis to be Ready

```bash
# Wait for Redis to complete provisioning
echo "Waiting for Redis cache to be ready..."
while true; do
  STATUS=$(az redis show \
    --name "$REDIS_NAME" \
    --resource-group "$RG_NAME" \
    --query provisioningState \
    --output tsv)
  
  if [ "$STATUS" = "Succeeded" ]; then
    echo "✅ Redis cache is ready"
    break
  else
    echo "Status: $STATUS - waiting..."
    sleep 60
  fi
done
```

---

## Step 5 – Get Redis Connection Details

```bash
# Get Redis hostname
REDIS_HOST=$(az redis show \
  --name "$REDIS_NAME" \
  --resource-group "$RG_NAME" \
  --query hostName \
  --output tsv)

# Get Redis access key
REDIS_KEY=$(az redis list-keys \
  --name "$REDIS_NAME" \
  --resource-group "$RG_NAME" \
  --query primaryKey \
  --output tsv)

# Get Redis port
REDIS_PORT=$(az redis show \
  --name "$REDIS_NAME" \
  --resource-group "$RG_NAME" \
  --query sslPort \
  --output tsv)

echo "Redis Host: $REDIS_HOST"
echo "Redis Port: $REDIS_PORT"
echo "Redis Key: ${REDIS_KEY:0:20}..."
echo "⚠️  Keep these credentials secure"
```

---

## Step 6 – Create Node.js Web Application

```bash
# Create app directory
mkdir -p redis-session-app
cd redis-session-app

# Initialize Node.js project
cat > package.json << 'EOF'
{
  "name": "redis-session-app",
  "version": "1.0.0",
  "description": "Web app with Redis session management",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "express-session": "^1.17.3",
    "connect-redis": "^7.1.0",
    "redis": "^4.6.5",
    "ejs": "^3.1.9"
  }
}
EOF

# Install dependencies
npm install

echo "✅ Node.js dependencies installed"
```

---

## Step 7 – Create Express Server with Redis Sessions

```bash
# Create server.js
cat > server.js << 'EOFJS'
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const app = express();
const PORT = process.env.PORT || 3000;

// Redis client configuration
const redisClient = createClient({
  socket: {
    host: process.env.REDIS_HOST,
    port: process.env.REDIS_PORT || 6380,
    tls: true
  },
  password: process.env.REDIS_KEY
});

// Connect to Redis
redisClient.connect().then(() => {
  console.log('✅ Connected to Redis');
}).catch((err) => {
  console.error('❌ Redis connection error:', err);
});

// Session configuration with Redis store
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'my-secret-key-change-in-production',
  resave: false,
  saveUninitialized: false,
  cookie: {
    maxAge: 1000 * 60 * 30, // 30 minutes
    httpOnly: true,
    secure: false // Set to true in production with HTTPS
  }
}));

// Set view engine
app.set('view engine', 'ejs');

// Middleware to parse request body
app.use(express.urlencoded({ extended: true }));

// Routes
app.get('/', (req, res) => {
  if (!req.session.visits) {
    req.session.visits = 1;
    req.session.userName = 'Guest';
    req.session.loginTime = new Date().toLocaleString();
  } else {
    req.session.visits++;
  }

  res.render('index', {
    visits: req.session.visits,
    userName: req.session.userName,
    loginTime: req.session.loginTime,
    sessionId: req.sessionID
  });
});

app.post('/login', (req, res) => {
  req.session.userName = req.body.userName || 'Guest';
  req.session.loginTime = new Date().toLocaleString();
  res.redirect('/');
});

app.get('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      console.error('Session destroy error:', err);
    }
    res.redirect('/');
  });
});

app.get('/info', (req, res) => {
  res.json({
    sessionId: req.sessionID,
    visits: req.session.visits || 0,
    userName: req.session.userName || 'Not logged in',
    loginTime: req.session.loginTime || 'N/A',
    redisHost: process.env.REDIS_HOST
  });
});

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', redis: redisClient.isOpen });
});

// Start server
app.listen(PORT, () => {
  console.log(`🚀 Server running on http://localhost:${PORT}`);
  console.log(`📊 Sessions stored in Redis: ${process.env.REDIS_HOST}`);
});
EOFJS
```

---

## Step 8 – Create Web UI Template

```bash
# Create views directory
mkdir -p views

# Create index.ejs template
cat > views/index.ejs << 'EOFHTML'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Redis Session Demo</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 40px 20px;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background: white;
            border-radius: 12px;
            padding: 40px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
        }
        h1 {
            color: #0078d4;
            margin-bottom: 10px;
        }
        .subtitle {
            color: #666;
            margin-bottom: 30px;
        }
        .info-box {
            background: #f0f8ff;
            border-left: 4px solid #0078d4;
            padding: 20px;
            margin: 20px 0;
            border-radius: 4px;
        }
        .info-box h2 {
            color: #0078d4;
            margin-bottom: 15px;
            font-size: 1.3em;
        }
        .info-row {
            margin: 10px 0;
            padding: 10px;
            background: white;
            border-radius: 4px;
        }
        .label {
            font-weight: bold;
            color: #333;
        }
        .value {
            color: #0078d4;
            font-family: monospace;
        }
        form {
            margin: 30px 0;
        }
        input[type="text"] {
            width: 100%;
            padding: 12px;
            border: 2px solid #ddd;
            border-radius: 6px;
            font-size: 16px;
            margin: 10px 0;
        }
        button {
            background: #0078d4;
            color: white;
            border: none;
            padding: 12px 30px;
            border-radius: 6px;
            font-size: 16px;
            cursor: pointer;
            margin: 5px;
            transition: background 0.3s;
        }
        button:hover {
            background: #005a9e;
        }
        .logout-btn {
            background: #d32f2f;
        }
        .logout-btn:hover {
            background: #9a0007;
        }
        .feature {
            background: #e8f5e9;
            padding: 15px;
            margin: 10px 0;
            border-radius: 4px;
            border-left: 4px solid #4caf50;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🔴 Redis Session Management</h1>
        <p class="subtitle">Azure Cache for Redis Demo</p>

        <div class="info-box">
            <h2>📊 Session Information</h2>
            <div class="info-row">
                <span class="label">User:</span>
                <span class="value"><%= userName %></span>
            </div>
            <div class="info-row">
                <span class="label">Page Visits:</span>
                <span class="value"><%= visits %></span>
            </div>
            <div class="info-row">
                <span class="label">Login Time:</span>
                <span class="value"><%= loginTime %></span>
            </div>
            <div class="info-row">
                <span class="label">Session ID:</span>
                <span class="value"><%= sessionId.substring(0, 20) %>...</span>
            </div>
        </div>

        <form method="POST" action="/login">
            <h2>👤 Change Username</h2>
            <input type="text" name="userName" placeholder="Enter your name" required>
            <button type="submit">Update Name</button>
        </form>

        <div>
            <button onclick="location.reload()">🔄 Refresh Page</button>
            <button class="logout-btn" onclick="location.href='/logout'">🚪 Logout (Clear Session)</button>
        </div>

        <div class="info-box" style="margin-top: 30px;">
            <h2>✨ Features Demonstrated</h2>
            <div class="feature">✅ Session data stored in Redis</div>
            <div class="feature">✅ Session persists across page refreshes</div>
            <div class="feature">✅ Automatic session expiration (30 minutes)</div>
            <div class="feature">✅ Scalable across multiple app instances</div>
        </div>
    </div>
</body>
</html>
EOFHTML

echo "✅ Application files created"
```

---

## Step 9 – Test Application Locally

```bash
# Set environment variables
export REDIS_HOST="$REDIS_HOST"
export REDIS_PORT="$REDIS_PORT"
export REDIS_KEY="$REDIS_KEY"
export PORT=3000

# Start the application
echo "Starting Node.js application..."
node server.js &
APP_PID=$!

sleep 5

# Test the application
echo -e "\n=== Testing Local Application ==="
curl -s http://localhost:3000/health | python3 -m json.tool

# Open in browser if available
if [ -n "$BROWSER" ]; then
  "$BROWSER" http://localhost:3000 &
fi

echo "✅ Application running at http://localhost:3000"
echo "Press Ctrl+C when ready to continue"

# Wait for user
sleep 10

# Stop local server
kill $APP_PID 2>/dev/null
```

---

## Step 10 – Deploy to Azure App Service

```bash
# Return to parent directory
cd ..

# Create App Service plan
az appservice plan create \
  --name "${APP_NAME}-plan" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku B1 \
  --is-linux

# Create Web App
az webapp create \
  --name "$APP_NAME" \
  --resource-group "$RG_NAME" \
  --plan "${APP_NAME}-plan" \
  --runtime "NODE:18-lts"

# Configure app settings (environment variables)
az webapp config appsettings set \
  --name "$APP_NAME" \
  --resource-group "$RG_NAME" \
  --settings \
    REDIS_HOST="$REDIS_HOST" \
    REDIS_PORT="$REDIS_PORT" \
    REDIS_KEY="$REDIS_KEY" \
    PORT="8080"

echo "✅ Web App created"
```

---

## Step 11 – Deploy Application Code

```bash
# Initialize git repository in app folder
cd redis-session-app
git init
git add .
git commit -m "Initial commit - Redis session app"

# Configure deployment user (if not already configured)
az webapp deployment user set \
  --user-name "deployuser$RANDOM" \
  --password "Deploy@Pass$(date +%s)!" 2>/dev/null || true

# Get Git deployment URL
GIT_URL=$(az webapp deployment source config-local-git \
  --name "$APP_NAME" \
  --resource-group "$RG_NAME" \
  --query url \
  --output tsv)

echo "Git URL: $GIT_URL"

# Deploy via git (alternative: use ZIP deployment)
echo "Deploying application..."
az webapp deployment source config-zip \
  --name "$APP_NAME" \
  --resource-group "$RG_NAME" \
  --src <(cd .. && zip -r - redis-session-app -x "*.git*" -x "*node_modules*")

echo "⏳ Deployment in progress..."
sleep 30
```

---

## Step 12 – Test Deployed Application

```bash
# Get application URL
APP_URL="https://${APP_NAME}.azurewebsites.net"
echo "Application URL: $APP_URL"

# Test health endpoint
echo -e "\n=== Health Check ==="
curl -s "${APP_URL}/health" | python3 -m json.tool

# Test session info endpoint
echo -e "\n=== Session Info ==="
curl -s "${APP_URL}/info" | python3 -m json.tool

# Open in browser
if [ -n "$BROWSER" ]; then
  "$BROWSER" "$APP_URL" &
fi

echo "✅ Application deployed and accessible"
```

---

## Step 13 – Verify Session Persistence

```bash
# Make multiple requests to verify session persistence
echo "=== Testing Session Persistence ==="

# First request (creates session)
echo "Request 1:"
curl -s -c cookies.txt -b cookies.txt "${APP_URL}/info" | python3 -m json.tool

sleep 2

# Second request (same session)
echo -e "\nRequest 2 (should show visits=2):"
curl -s -c cookies.txt -b cookies.txt "${APP_URL}/info" | python3 -m json.tool

sleep 2

# Third request (same session)
echo -e "\nRequest 3 (should show visits=3):"
curl -s -c cookies.txt -b cookies.txt "${APP_URL}/info" | python3 -m json.tool

rm -f cookies.txt

echo "✅ Session data persisted across requests via Redis"
```

---

## Step 14 – Monitor Redis Cache

```bash
# Get Redis metrics
REDIS_ID=$(az redis show \
  --name "$REDIS_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

# Check cache hits and misses
az monitor metrics list \
  --resource "$REDIS_ID" \
  --metric "cachehits" "cachemisses" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT5M \
  --output table

# Check connected clients
az monitor metrics list \
  --resource "$REDIS_ID" \
  --metric "connectedclients" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT5M \
  --output table
```

---

## Step 15 – View Application Logs

```bash
# Enable application logging
az webapp log config \
  --name "$APP_NAME" \
  --resource-group "$RG_NAME" \
  --application-logging filesystem \
  --level information

# Stream logs
echo "Streaming application logs (Ctrl+C to stop)..."
az webapp log tail \
  --name "$APP_NAME" \
  --resource-group "$RG_NAME" &

LOG_PID=$!
sleep 10
kill $LOG_PID 2>/dev/null
```

---

## Step 16 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
cd ..
rm -rf redis-session-app

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Azure Cache for Redis instance
- Node.js Express web application
- Redis-backed session management
- Deployed web app to Azure App Service
- Session persistence across requests

**Architecture:**
```
User Browser → Azure App Service (Node.js) → Azure Cache for Redis
                        ↓                           ↓
                  Session Cookie              Session Data Storage
```

**Key Components:**
- **Azure Cache for Redis**: Managed Redis cache service
- **Express Session**: Session middleware for Express.js
- **Connect-Redis**: Redis session store adapter
- **App Service**: Managed platform for web applications

**What You Learned:**
- Create and configure Azure Cache for Redis
- Build Node.js app with Redis session storage
- Deploy web applications to Azure App Service
- Configure application settings and environment variables
- Monitor Redis cache performance metrics

---

## Best Practices

**Redis Configuration:**
- Use SSL/TLS for all connections (port 6380)
- Store access keys in Azure Key Vault
- Enable AOF persistence for data durability
- Use Premium tier for production workloads
- Configure appropriate maxmemory policy

**Session Management:**
- Set appropriate session timeout (30 min typical)
- Use secure, httpOnly cookies
- Implement session regeneration on login
- Clean up expired sessions regularly
- Use short session IDs

**Application Design:**
- Implement connection retry logic
- Handle Redis connection failures gracefully
- Use connection pooling
- Serialize session data efficiently
- Monitor session store performance

**Security:**
- Never commit Redis keys to source control
- Use environment variables for credentials
- Enable firewall rules to restrict access
- Use VNet integration for private connectivity
- Rotate access keys regularly

---

## Production Enhancements

**1. Enable Redis Persistence**
```bash
# Upgrade to Premium tier with persistence
az redis update \
  --name "$REDIS_NAME" \
  --resource-group "$RG_NAME" \
  --sku Premium \
  --vm-size P1

# Enable RDB persistence
az redis patch-schedule create \
  --name "$REDIS_NAME" \
  --resource-group "$RG_NAME" \
  --schedule-entries '[{"dayOfWeek":"Sunday","startHourUtc":3,"maintenanceWindow":"PT5H"}]'
```

**2. Configure Clustering (Premium)**
```bash
# Enable Redis clustering for scale
az redis create \
  --name "redis-clustered" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Premium \
  --vm-size P1 \
  --shard-count 3
```

**3. Implement VNet Integration**
```bash
# Create Redis in VNet (Premium tier)
az redis create \
  --name "redis-vnet" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Premium \
  --vm-size P1 \
  --subnet "/subscriptions/.../subnets/redis-subnet"
```

**4. Set Up Geo-Replication**
```bash
# Link two Premium Redis instances for geo-replication
az redis server-link create \
  --name "$REDIS_NAME" \
  --resource-group "$RG_NAME" \
  --server-to-link "redis-secondary" \
  --replication-role Secondary
```

---

## Troubleshooting

**Cannot connect to Redis:**
- Verify Redis is provisioned (not creating)
- Check firewall rules allow your IP
- Ensure using SSL port (6380, not 6379)
- Verify access key is correct
- Check Redis cache status in portal

**Application errors:**
- Review application logs in App Service
- Verify environment variables are set correctly
- Check Node.js version compatibility
- Ensure all npm packages installed
- Review Redis client connection settings

**Sessions not persisting:**
- Verify Redis connection is successful
- Check session middleware configuration
- Ensure cookies are enabled in browser
- Review cookie settings (secure, httpOnly)
- Check session timeout settings

**Slow performance:**
- Monitor Redis CPU and memory usage
- Check for high latency between app and Redis
- Review connection pool settings
- Consider upgrading Redis tier
- Optimize session data size

**Deployment fails:**
- Check App Service deployment logs
- Verify package.json is correct
- Ensure Node.js version is supported
- Review build output for errors
- Check for missing dependencies

---

## Additional Resources

- [Azure Cache for Redis Documentation](https://docs.microsoft.com/azure/azure-cache-for-redis/)
- [Redis Best Practices](https://docs.microsoft.com/azure/azure-cache-for-redis/cache-best-practices)
- [Express Session Documentation](https://www.npmjs.com/package/express-session)
- [Connect-Redis Documentation](https://www.npmjs.com/package/connect-redis)
- [App Service Node.js Guide](https://docs.microsoft.com/azure/app-service/quickstart-nodejs)
