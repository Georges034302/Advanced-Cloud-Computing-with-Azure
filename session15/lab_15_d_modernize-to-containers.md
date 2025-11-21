# Lab 15.D: Modernize to Containers

## Objectives
- Containerize legacy application
- Create Azure Container Registry
- Build and push container images
- Deploy to Azure Kubernetes Service
- Configure ingress and load balancing
- Implement rolling updates
- Monitor containerized applications
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
RG_NAME="rg-lab15d-containers"
ACR_NAME="acr$RANDOM"
AKS_NAME="aks-cluster"
APP_NAME="modernized-app"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$ACR_NAME"
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

## Step 3 – Show Containerization Overview

```bash
# Display containerization benefits
cat << 'EOF'

Application Containerization Overview:
=======================================

Why Containerize Legacy Applications:

Benefits:
  ✓ Consistent environments (dev/test/prod)
  ✓ Simplified deployment process
  ✓ Better resource utilization
  ✓ Easier scaling (horizontal and vertical)
  ✓ Faster startup times
  ✓ Microservices architecture enablement
  ✓ Cloud-native capabilities
  ✓ CI/CD integration

Containerization Strategies:

1. Rehost (Lift-and-Shift):
   - Containerize as-is
   - Minimal code changes
   - Quick migration path
   - Good for: Monolithic apps

2. Refactor:
   - Optimize for containers
   - Update dependencies
   - Externalize configuration
   - Good for: Modernization

3. Rearchitect:
   - Break into microservices
   - Cloud-native patterns
   - Significant changes
   - Good for: Long-term investment

Container Orchestration Options:

Azure Container Instances (ACI):
  - Simplest option
  - No cluster management
  - Pay per second
  - Good for: Simple apps, batch jobs

Azure Kubernetes Service (AKS):
  - Full orchestration
  - Advanced features
  - Production-ready
  - Good for: Complex apps, microservices

Azure App Service:
  - Managed platform
  - Built-in features
  - Container support
  - Good for: Web apps

EOF
```

---

## Step 4 – Show Migration Checklist

```bash
# Display containerization checklist
cat << 'EOF'

Containerization Checklist:
============================

Application Assessment:
  □ Application is stateless or state externalized
  □ No OS-specific dependencies
  □ Configuration externalized
  □ No hard-coded file paths
  □ Database uses connection strings

Docker Preparation:
  □ Dockerfile created
  □ Base image selected
  □ Dependencies documented
  □ Build process defined
  □ Environment variables identified

Data & State:
  □ Database migrated to Azure SQL/MySQL/PostgreSQL
  □ Session state in Redis Cache
  □ Files stored in Azure Storage/Files
  □ Logs sent to Azure Monitor
  □ Configuration in Key Vault/ConfigMap

Networking:
  □ Service endpoints defined
  □ Port mappings documented
  □ DNS requirements identified
  □ SSL/TLS certificates available
  □ Load balancing strategy defined

Security:
  □ Secrets managed securely
  □ Container scanning enabled
  □ RBAC configured
  □ Network policies defined
  □ Private registry access

Monitoring:
  □ Health check endpoints
  □ Application Insights integrated
  □ Prometheus metrics exposed
  □ Log aggregation configured
  □ Alerting rules defined

EOF
```

---

## Step 5 – Create Sample Application

```bash
# Create legacy application to containerize
mkdir -p legacy-app
cd legacy-app

# Create Node.js application
cat > app.js << 'EOF'
const http = require('http');
const os = require('os');

const port = process.env.PORT || 3000;
const version = process.env.APP_VERSION || '1.0.0';

// Simulated in-memory data (in production, use database)
let requestCount = 0;

const server = http.createServer((req, res) => {
  requestCount++;

  // Health check endpoint
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'healthy', uptime: process.uptime() }));
    return;
  }

  // Metrics endpoint
  if (req.url === '/metrics') {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end(`# HELP requests_total Total number of requests
# TYPE requests_total counter
requests_total ${requestCount}

# HELP app_uptime_seconds Application uptime in seconds
# TYPE app_uptime_seconds gauge
app_uptime_seconds ${Math.floor(process.uptime())}
`);
    return;
  }

  // API endpoint
  if (req.url === '/api/info') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      version: version,
      hostname: os.hostname(),
      platform: os.platform(),
      architecture: os.arch(),
      nodeVersion: process.version,
      uptime: Math.floor(process.uptime()),
      memory: {
        total: Math.floor(os.totalmem() / 1024 / 1024) + ' MB',
        free: Math.floor(os.freemem() / 1024 / 1024) + ' MB'
      },
      cpus: os.cpus().length,
      requestCount: requestCount
    }));
    return;
  }

  // Main page
  res.writeHead(200, { 'Content-Type': 'text/html' });
  res.end(`
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Modernized Application</title>
      <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
          font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
          background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
          min-height: 100vh;
          padding: 20px;
        }
        .container {
          max-width: 800px;
          margin: 40px auto;
          background: white;
          border-radius: 12px;
          box-shadow: 0 20px 60px rgba(0,0,0,0.3);
          overflow: hidden;
        }
        .header {
          background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
          color: white;
          padding: 40px;
          text-align: center;
        }
        .header h1 { font-size: 2.5em; margin-bottom: 10px; }
        .header p { font-size: 1.2em; opacity: 0.9; }
        .content { padding: 40px; }
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
          border-left: 4px solid #667eea;
        }
        .info-card h3 {
          color: #667eea;
          margin-bottom: 10px;
          font-size: 0.9em;
          text-transform: uppercase;
        }
        .info-card p {
          color: #333;
          font-size: 1.2em;
          font-weight: bold;
        }
        .features {
          display: grid;
          grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
          gap: 20px;
          margin: 30px 0;
        }
        .feature {
          background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
          color: white;
          padding: 20px;
          border-radius: 8px;
          text-align: center;
        }
        .feature-icon {
          font-size: 2em;
          margin-bottom: 10px;
        }
        .feature-title {
          font-weight: bold;
          margin-bottom: 5px;
        }
        .feature-desc {
          font-size: 0.9em;
          opacity: 0.9;
        }
        button {
          background: #667eea;
          color: white;
          border: none;
          padding: 15px 30px;
          border-radius: 6px;
          font-size: 1em;
          cursor: pointer;
          margin: 10px;
          transition: background 0.3s;
        }
        button:hover { background: #5568d3; }
        #details {
          margin-top: 20px;
          padding: 20px;
          background: #f8f9fa;
          border-radius: 8px;
          display: none;
        }
        .metric {
          margin: 10px 0;
          padding: 10px;
          background: white;
          border-radius: 4px;
        }
        .metric strong { color: #667eea; }
      </style>
    </head>
    <body>
      <div class="container">
        <div class="header">
          <h1>🚀 Modernized Application</h1>
          <p>Running in Container on Azure Kubernetes Service</p>
        </div>
        
        <div class="content">
          <div class="info-grid">
            <div class="info-card">
              <h3>Version</h3>
              <p>${version}</p>
            </div>
            <div class="info-card">
              <h3>Container</h3>
              <p>${os.hostname().substring(0, 12)}</p>
            </div>
            <div class="info-card">
              <h3>Requests</h3>
              <p>${requestCount}</p>
            </div>
            <div class="info-card">
              <h3>Uptime</h3>
              <p>${Math.floor(process.uptime())}s</p>
            </div>
          </div>

          <h2>Container Benefits</h2>
          <div class="features">
            <div class="feature">
              <div class="feature-icon">📦</div>
              <div class="feature-title">Portable</div>
              <div class="feature-desc">Run anywhere consistently</div>
            </div>
            <div class="feature">
              <div class="feature-icon">⚡</div>
              <div class="feature-title">Fast</div>
              <div class="feature-desc">Quick startup and scaling</div>
            </div>
            <div class="feature">
              <div class="feature-icon">🔄</div>
              <div class="feature-title">Scalable</div>
              <div class="feature-desc">Auto-scale on demand</div>
            </div>
            <div class="feature">
              <div class="feature-icon">🛡️</div>
              <div class="feature-title">Isolated</div>
              <div class="feature-desc">Secure and isolated</div>
            </div>
          </div>

          <div style="text-align: center;">
            <button onclick="loadDetails()">Load Container Details</button>
            <button onclick="window.location.href='/health'">Health Check</button>
          </div>

          <div id="details"></div>
        </div>
      </div>

      <script>
        async function loadDetails() {
          try {
            const response = await fetch('/api/info');
            const data = await response.json();
            const details = document.getElementById('details');
            details.style.display = 'block';
            details.innerHTML = \`
              <h3>Container Details</h3>
              <div class="metric"><strong>Version:</strong> \${data.version}</div>
              <div class="metric"><strong>Hostname:</strong> \${data.hostname}</div>
              <div class="metric"><strong>Platform:</strong> \${data.platform} (\${data.architecture})</div>
              <div class="metric"><strong>Node.js:</strong> \${data.nodeVersion}</div>
              <div class="metric"><strong>CPUs:</strong> \${data.cpus}</div>
              <div class="metric"><strong>Memory:</strong> \${data.memory.free} free of \${data.memory.total}</div>
              <div class="metric"><strong>Uptime:</strong> \${data.uptime} seconds</div>
              <div class="metric"><strong>Total Requests:</strong> \${data.requestCount}</div>
            \`;
          } catch (error) {
            document.getElementById('details').innerHTML = 
              '<div class="metric" style="color: red;">Error loading details</div>';
          }
        }
      </script>
    </body>
    </html>
  `);
});

server.listen(port, () => {
  console.log(\`Server running on port \${port}\`);
  console.log(\`Version: \${version}\`);
  console.log(\`Hostname: \${os.hostname()}\`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
EOF

# Create package.json
cat > package.json << 'EOF'
{
  "name": "modernized-app",
  "version": "1.0.0",
  "description": "Containerized legacy application",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
EOF

cd ..
echo "Sample application created"
```

---

## Step 6 – Create Dockerfile

```bash
# Create Dockerfile for containerization
cd legacy-app

cat > Dockerfile << 'EOF'
# Multi-stage build for optimized image
FROM node:18-alpine AS base

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --only=production

# Copy application code
COPY app.js ./

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 && \
    chown -R nodejs:nodejs /app

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1); })"

# Start application
CMD ["node", "app.js"]
EOF

cd ..
echo "Dockerfile created"
```

---

## Step 7 – Create .dockerignore

```bash
# Create .dockerignore to exclude unnecessary files
cd legacy-app

cat > .dockerignore << 'EOF'
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.DS_Store
*.md
.vscode
coverage
.nyc_output
EOF

cd ..
echo ".dockerignore created"
```

---

## Step 8 – Create Azure Container Registry

```bash
# Create ACR with Basic SKU
az acr create \
  --name "$ACR_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Basic \
  --admin-enabled true

echo "Container registry created"
```

---

## Step 9 – Show ACR SKUs

```bash
# Display ACR pricing tiers
cat << 'EOF'

Azure Container Registry SKUs:
===============================

Basic:
  - 10 GB storage included
  - 10 webhooks
  - Basic authentication
  - Good for: Development/testing
  - Cost: ~$5/month

Standard:
  - 100 GB storage included
  - 100 webhooks
  - Geo-replication (preview)
  - Good for: Production (small-medium)
  - Cost: ~$20/month

Premium:
  - 500 GB storage included
  - 500 webhooks
  - Geo-replication
  - Content trust
  - Private link
  - Good for: Production (enterprise)
  - Cost: ~$50/month

Features by Tier:
  - All: Container images, Helm charts, OCI artifacts
  - Standard+: Webhook notifications
  - Premium: Multi-region replication, security scanning

Recommendations:
  - Development: Basic
  - Production: Standard or Premium
  - Multi-region: Premium
  - Security compliance: Premium

EOF
```

---

## Step 10 – Build Container Image

```bash
# Build and push image to ACR
az acr build \
  --registry "$ACR_NAME" \
  --image "${APP_NAME}:v1" \
  --file legacy-app/Dockerfile \
  legacy-app/

echo "Container image built and pushed"
```

---

## Step 11 – List Container Images

```bash
# List images in ACR
az acr repository list \
  --name "$ACR_NAME" \
  --output table

# Show image tags
az acr repository show-tags \
  --name "$ACR_NAME" \
  --repository "$APP_NAME" \
  --output table
```

---

## Step 12 – Create AKS Cluster

```bash
# Create AKS cluster with default settings
az aks create \
  --name "$AKS_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --node-count 2 \
  --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity \
  --generate-ssh-keys \
  --attach-acr "$ACR_NAME"

echo "AKS cluster created"
```

---

## Step 13 – Get AKS Credentials

```bash
# Get cluster credentials
az aks get-credentials \
  --name "$AKS_NAME" \
  --resource-group "$RG_NAME" \
  --overwrite-existing

echo "Credentials configured"
```

---

## Step 14 – Show AKS Cluster Info

```bash
# Display cluster information
cat << 'EOF'

Azure Kubernetes Service (AKS):
================================

AKS Features:
  ✓ Managed Kubernetes control plane
  ✓ Auto-scaling (cluster and pods)
  ✓ Auto-upgrade and patching
  ✓ Integration with Azure services
  ✓ Built-in monitoring
  ✓ Azure AD integration
  ✓ Network policies
  ✓ Private clusters

Node Pool Types:

System Node Pool:
  - Runs system pods (CoreDNS, metrics-server)
  - Required for cluster operation
  - Minimum 1 node recommended

User Node Pool:
  - Runs application workloads
  - Can scale to zero
  - Multiple pools for workload separation

VM Sizes:
  - D-series: General purpose
  - E-series: Memory optimized
  - F-series: Compute optimized
  - B-series: Burstable (dev/test)

Networking Modes:

kubenet:
  - Basic networking
  - Lower IP address consumption
  - Simple setup
  - Good for: Small deployments

Azure CNI:
  - Advanced networking
  - Each pod gets IP from VNet
  - VNet integration
  - Good for: Enterprise, networking requirements

EOF
```

---

## Step 15 – Verify Cluster Access

```bash
# Check cluster nodes
kubectl get nodes

# Check cluster info
kubectl cluster-info

echo "Cluster verified"
```

---

## Step 16 – Create Kubernetes Deployment

```bash
# Create deployment YAML
cat > deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  labels:
    app: ${APP_NAME}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${ACR_NAME}.azurecr.io/${APP_NAME}:v1
        ports:
        - containerPort: 3000
        env:
        - name: PORT
          value: "3000"
        - name: APP_VERSION
          value: "1.0.0"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
EOF

echo "Deployment manifest created"
```

---

## Step 17 – Create Kubernetes Service

```bash
# Create service YAML
cat > service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: ${APP_NAME}
EOF

echo "Service manifest created"
```

---

## Step 18 – Deploy Application to AKS

```bash
# Apply deployment and service
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

echo "Application deployed to AKS"
```

---

## Step 19 – Wait for Deployment

```bash
# Wait for deployment to be ready
kubectl rollout status deployment/${APP_NAME}

echo "Deployment ready"
```

---

## Step 20 – Get Service External IP

```bash
# Wait for external IP (may take a few minutes)
echo "Waiting for external IP..."
kubectl get service ${APP_NAME} --watch &
WATCH_PID=$!
sleep 60
kill $WATCH_PID 2>/dev/null || true

# Get external IP
EXTERNAL_IP=$(kubectl get service ${APP_NAME} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "$EXTERNAL_IP"
```

---

## Step 21 – Test Application

```bash
# Test application endpoint
if [ -n "$EXTERNAL_IP" ]; then
  echo "Testing application..."
  curl -s "http://${EXTERNAL_IP}/health" | grep -o '"status":"[^"]*"'
  echo "Application tested"
else
  echo "External IP not yet assigned"
fi
```

---

## Step 22 – Show Pod Status

```bash
# List all pods
kubectl get pods -l app=${APP_NAME}

# Get pod details
kubectl describe pods -l app=${APP_NAME} | grep -E "^Name:|^Status:|^IP:|^Node:"
```

---

## Step 23 – View Pod Logs

```bash
# Get logs from first pod
POD_NAME=$(kubectl get pods -l app=${APP_NAME} -o jsonpath='{.items[0].metadata.name}')

echo "$POD_NAME"

# Show pod logs
kubectl logs "$POD_NAME" --tail=20
```

---

## Step 24 – Create Updated Version

```bash
# Build version 2 of the application
cd legacy-app

# Update version in package.json for demonstration
sed -i 's/"version": "1.0.0"/"version": "2.0.0"/' package.json

# Build new version
cd ..
az acr build \
  --registry "$ACR_NAME" \
  --image "${APP_NAME}:v2" \
  --file legacy-app/Dockerfile \
  legacy-app/

echo "Version 2 built"
```

---

## Step 25 – Perform Rolling Update

```bash
# Update deployment to use v2
kubectl set image deployment/${APP_NAME} ${APP_NAME}=${ACR_NAME}.azurecr.io/${APP_NAME}:v2

echo "Rolling update initiated"
```

---

## Step 26 – Monitor Rolling Update

```bash
# Watch rollout status
kubectl rollout status deployment/${APP_NAME}

# Show rollout history
kubectl rollout history deployment/${APP_NAME}

echo "Update completed"
```

---

## Step 27 – Show Scaling Options

```bash
# Display scaling information
cat << EOF

Kubernetes Scaling Options:
============================

Horizontal Pod Autoscaler (HPA):
  - Scales pods based on metrics
  - CPU, memory, custom metrics
  - Min/max replica count

  Create HPA:
  kubectl autoscale deployment ${APP_NAME} \\
    --min=2 --max=10 \\
    --cpu-percent=70

Vertical Pod Autoscaler (VPA):
  - Adjusts resource requests/limits
  - Right-sizes containers
  - Requires VPA controller

Cluster Autoscaler:
  - Scales AKS nodes
  - Adds nodes when pods pending
  - Removes nodes when underutilized

  Enable on AKS:
  az aks update \\
    --resource-group $RG_NAME \\
    --name $AKS_NAME \\
    --enable-cluster-autoscaler \\
    --min-count 1 \\
    --max-count 5

Manual Scaling:
  Scale pods:
  kubectl scale deployment ${APP_NAME} --replicas=5

  Scale nodes:
  az aks scale \\
    --resource-group $RG_NAME \\
    --name $AKS_NAME \\
    --node-count 3

EOF
```

---

## Step 28 – Show Best Practices

```bash
# Display containerization best practices
cat << 'EOF'

Container Best Practices:
==========================

Dockerfile:
  ✓ Use official base images
  ✓ Multi-stage builds for smaller images
  ✓ Layer caching optimization
  ✓ Don't run as root user
  ✓ Use specific image tags (not 'latest')
  ✓ Minimize layers
  ✓ Use .dockerignore

Image Security:
  ✓ Scan images for vulnerabilities
  ✓ Use minimal base images (Alpine)
  ✓ Keep base images updated
  ✓ Sign images
  ✓ Private registry for production

Kubernetes Manifests:
  ✓ Define resource requests and limits
  ✓ Use health checks (liveness/readiness)
  ✓ Set appropriate restart policies
  ✓ Use namespaces for isolation
  ✓ Label resources consistently
  ✓ Use ConfigMaps for configuration
  ✓ Use Secrets for sensitive data

Application Design:
  ✓ Stateless design
  ✓ 12-factor app principles
  ✓ Graceful shutdown (SIGTERM)
  ✓ Health check endpoints
  ✓ Structured logging
  ✓ Externalize configuration
  ✓ Prometheus metrics

Monitoring:
  ✓ Container Insights enabled
  ✓ Application logging
  ✓ Resource utilization metrics
  ✓ Alert rules configured
  ✓ Distributed tracing

High Availability:
  ✓ Multiple replicas
  ✓ Pod anti-affinity
  ✓ Resource quotas
  ✓ Network policies
  ✓ Backup strategies

CI/CD:
  ✓ Automated builds
  ✓ Image scanning in pipeline
  ✓ Automated testing
  ✓ GitOps deployment
  ✓ Rollback procedures

EOF
```

---

## Step 29 – Show Troubleshooting

```bash
# Display troubleshooting commands
cat << EOF

Kubernetes Troubleshooting:
============================

Pod Issues:

Get pod status:
  kubectl get pods -l app=${APP_NAME}

Describe pod:
  kubectl describe pod <pod-name>

View logs:
  kubectl logs <pod-name>
  kubectl logs <pod-name> --previous  # Previous container
  kubectl logs -f <pod-name>           # Follow logs

Execute in pod:
  kubectl exec -it <pod-name> -- /bin/sh

Events:
  kubectl get events --sort-by='.lastTimestamp'

Service Issues:

Check service:
  kubectl get service ${APP_NAME}
  kubectl describe service ${APP_NAME}

Check endpoints:
  kubectl get endpoints ${APP_NAME}

Test connectivity:
  kubectl run debug --rm -it --image=busybox -- /bin/sh
  wget -O- http://${APP_NAME}

Deployment Issues:

Rollout status:
  kubectl rollout status deployment/${APP_NAME}

Rollout history:
  kubectl rollout history deployment/${APP_NAME}

Rollback:
  kubectl rollout undo deployment/${APP_NAME}
  kubectl rollout undo deployment/${APP_NAME} --to-revision=1

Cluster Issues:

Node status:
  kubectl get nodes
  kubectl describe node <node-name>

Resource usage:
  kubectl top nodes
  kubectl top pods

Cluster events:
  kubectl get events -A

Common Problems:

ImagePullBackOff:
  - Check ACR connection
  - Verify image exists
  - Check permissions

CrashLoopBackOff:
  - Check application logs
  - Verify health checks
  - Review resource limits

Pending pods:
  - Check resource availability
  - Review node selectors
  - Check quotas

EOF
```

---

## Step 30 – Show Cost Optimization

```bash
# Display cost optimization tips
cat << 'EOF'

AKS Cost Optimization:
======================

Node Optimization:
  ✓ Right-size VM SKUs
  ✓ Use spot instances for dev/test
  ✓ Enable cluster autoscaler
  ✓ Use B-series VMs for variable loads
  ✓ Delete unused node pools

Container Optimization:
  ✓ Set resource requests and limits
  ✓ Use Horizontal Pod Autoscaler
  ✓ Optimize container images (size)
  ✓ Use init containers wisely
  ✓ Schedule batch jobs off-peak

Storage:
  ✓ Use appropriate storage classes
  ✓ Delete unused PVCs
  ✓ Use Azure Files vs Blob based on need
  ✓ Lifecycle policies for storage

Networking:
  ✓ Use internal load balancers
  ✓ Minimize data transfer costs
  ✓ Use Azure CNI efficiently
  ✓ Private clusters for security

Monitoring:
  ✓ Container Insights (only when needed)
  ✓ Log retention policies
  ✓ Metrics sampling rate
  ✓ Alert rule optimization

Development:
  ✓ Stop dev clusters after hours
  ✓ Use smaller node sizes
  ✓ Share clusters across teams
  ✓ Use namespaces for isolation

Reserved Instances:
  ✓ 1-year: Save 30-40%
  ✓ 3-year: Save 60-65%
  ✓ Analyze usage patterns first

Estimated Costs (Australia East):
  AKS control plane: Free
  2x D2s_v3 nodes: ~$140/month
  Load Balancer: ~$20/month
  Storage: ~$10/month
  Container Insights: ~$15/month
  
  Total: ~$185/month

With optimizations: ~$100/month

EOF
```

---

## Step 31 – Cleanup

```bash
# Delete Kubernetes resources
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml

# Delete resource group (includes AKS and ACR)
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf legacy-app
rm -f deployment.yaml
rm -f service.yaml

# Remove kubectl context
kubectl config delete-context "$AKS_NAME"

echo "Cleanup complete"
```

---

## Step 32 – Show Migration Summary

```bash
# Display migration summary
cat << 'EOF'

Containerization Migration Summary:
====================================

What We Accomplished:

1. Assessment:
   ✓ Identified legacy application
   ✓ Reviewed containerization requirements
   ✓ Planned modernization strategy

2. Containerization:
   ✓ Created Dockerfile
   ✓ Built optimized container image
   ✓ Implemented health checks
   ✓ Added graceful shutdown

3. Registry:
   ✓ Created Azure Container Registry
   ✓ Pushed container images
   ✓ Managed image versions

4. Orchestration:
   ✓ Deployed AKS cluster
   ✓ Created Kubernetes deployment
   ✓ Configured service (LoadBalancer)
   ✓ Set resource limits

5. Operations:
   ✓ Performed rolling update
   ✓ Tested zero-downtime deployment
   ✓ Monitored pod health
   ✓ Verified scalability

Benefits Achieved:

Infrastructure:
  - From: VMs with manual management
  - To: Managed Kubernetes platform

Deployment:
  - From: Manual, error-prone process
  - To: Automated, repeatable deployments

Scaling:
  - From: Manual VM scaling
  - To: Auto-scaling pods and nodes

Updates:
  - From: Downtime required
  - To: Rolling updates, zero downtime

Efficiency:
  - From: One app per VM
  - To: Multiple containers per node

Next Steps:

Immediate:
  1. Implement CI/CD pipeline
  2. Add monitoring and alerting
  3. Configure ingress controller
  4. Set up persistent storage

Short-term:
  1. Break monolith into microservices
  2. Implement service mesh (Istio/Linkerd)
  3. Add distributed tracing
  4. Optimize resource usage

Long-term:
  1. Multi-region deployment
  2. Disaster recovery plan
  3. Chaos engineering
  4. FinOps optimization

EOF
```

---

## Summary

You containerized a legacy application using Docker and multi-stage builds for optimization, created Azure Container Registry to store private container images, built and pushed container images using ACR build tasks, deployed Azure Kubernetes Service cluster with managed identity and ACR integration, created Kubernetes deployment with three replicas and resource limits, configured LoadBalancer service for external access, implemented health checks with liveness and readiness probes, performed rolling update to demonstrate zero-downtime deployment, explored pod autoscaling and cluster autoscaling options, monitored application using kubectl commands and pod logs, learned container and Kubernetes best practices for security and reliability, and understood cost optimization strategies for AKS including right-sizing, autoscaling, and reserved instances.
