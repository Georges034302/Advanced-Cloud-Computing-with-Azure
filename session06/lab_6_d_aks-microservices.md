# Lab 6.D: AKS Microservices

## Overview
This lab demonstrates Azure Kubernetes Service with microservices architecture. You'll create an AKS cluster with Azure CNI, deploy multiple microservices, expose services, install an ingress controller, configure ingress rules, and validate service-to-service communication.

---

## Objectives
- Create AKS cluster with Azure CNI networking
- Get kubeconfig credentials
- Deploy microservices applications
- Expose services with ClusterIP and LoadBalancer
- Install NGINX ingress controller
- Deploy ingress rules for routing
- Validate service-to-service communication
- Output ingress public IP
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- kubectl installed (`kubectl version --client`)
- Azure subscription with contributor access
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab6d-aks"
AKS_NAME="aks-microservices"
NODE_COUNT=2
NODE_SIZE="Standard_B2s"
VNET_NAME="vnet-aks"
SUBNET_NAME="subnet-aks"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$AKS_NAME"
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
# Create virtual network for AKS
az network vnet create \
  --name "$VNET_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --address-prefix 10.0.0.0/16

# Create subnet for AKS nodes
az network vnet subnet create \
  --name "$SUBNET_NAME" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --address-prefix 10.0.1.0/24

# Get subnet ID
SUBNET_ID=$(az network vnet subnet show \
  --name "$SUBNET_NAME" \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --query id \
  --output tsv)

echo "$SUBNET_ID"
```

---

## Step 4 – Create AKS Cluster

```bash
# Create AKS cluster with Azure CNI
az aks create \
  --name "$AKS_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --node-count "$NODE_COUNT" \
  --node-vm-size "$NODE_SIZE" \
  --network-plugin azure \
  --vnet-subnet-id "$SUBNET_ID" \
  --service-cidr 10.1.0.0/16 \
  --dns-service-ip 10.1.0.10 \
  --generate-ssh-keys \
  --enable-managed-identity

echo "AKS cluster created"
```

---

## Step 5 – Get kubeconfig Credentials

```bash
# Get AKS credentials
az aks get-credentials \
  --name "$AKS_NAME" \
  --resource-group "$RG_NAME" \
  --overwrite-existing

# Verify connection
kubectl get nodes
```

---

## Step 6 – Create Backend API Service

```bash
# Create namespace for microservices
kubectl create namespace microservices

# Create backend API deployment
cat > backend-api.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: microservices
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:latest
        args:
          - -text=Backend API Response
          - -listen=:8080
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: backend-api
---
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: microservices
spec:
  type: ClusterIP
  selector:
    app: backend-api
  ports:
  - port: 8080
    targetPort: 8080
EOF

# Deploy backend API
kubectl apply -f backend-api.yaml

echo "Backend API deployed"
```

---

## Step 7 – Create Data Service

```bash
# Create data service deployment
cat > data-service.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-service
  namespace: microservices
spec:
  replicas: 2
  selector:
    matchLabels:
      app: data-service
  template:
    metadata:
      labels:
        app: data-service
    spec:
      containers:
      - name: data
        image: hashicorp/http-echo:latest
        args:
          - -text={"service":"data","status":"available","data":["item1","item2","item3"]}
          - -listen=:8080
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: data-service
---
apiVersion: v1
kind: Service
metadata:
  name: data-service
  namespace: microservices
spec:
  type: ClusterIP
  selector:
    app: data-service
  ports:
  - port: 8080
    targetPort: 8080
EOF

# Deploy data service
kubectl apply -f data-service.yaml

echo "Data service deployed"
```

---

## Step 8 – Create Frontend Service

```bash
# Create frontend service deployment
cat > frontend.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: microservices
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: setup
        image: busybox
        command:
        - sh
        - -c
        - |
          cat > /html/index.html << 'HTMLEOF'
          <!DOCTYPE html>
          <html>
          <head>
              <title>Microservices Frontend</title>
          </head>
          <body>
              <h1>Microservices Architecture on AKS</h1>
              <h2>Frontend Service</h2>
              <p>This frontend communicates with backend services</p>
              <ul>
                  <li><a href="/api">Backend API</a></li>
                  <li><a href="/data">Data Service</a></li>
              </ul>
          </body>
          </html>
          HTMLEOF
        volumeMounts:
        - name: html
          mountPath: /html
      volumes:
      - name: html
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: microservices
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
EOF

# Deploy frontend
kubectl apply -f frontend.yaml

echo "Frontend deployed"
```

---

## Step 9 – Verify Deployments

```bash
# Wait for deployments to be ready
echo "Waiting for deployments..."
kubectl wait --for=condition=available --timeout=120s deployment/backend-api -n microservices
kubectl wait --for=condition=available --timeout=120s deployment/data-service -n microservices
kubectl wait --for=condition=available --timeout=120s deployment/frontend -n microservices

# List all resources
kubectl get all -n microservices
```

---

## Step 10 – Get Frontend LoadBalancer IP

```bash
# Wait for external IP assignment
echo "Waiting for external IP..."
sleep 30

# Get frontend LoadBalancer IP
FRONTEND_IP=$(kubectl get service frontend -n microservices \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "$FRONTEND_IP"

# Test frontend service
echo "Testing frontend:"
curl -s "http://${FRONTEND_IP}/" | grep -o '<h1>.*</h1>'
```

---

## Step 11 – Test Service-to-Service Communication

```bash
# Create test pod for internal communication
kubectl run test-pod --image=busybox -n microservices -- sleep 3600

# Wait for pod to be ready
kubectl wait --for=condition=ready pod/test-pod -n microservices --timeout=60s

# Test communication to backend-api
echo "Testing internal communication to backend-api:"
kubectl exec -n microservices test-pod -- wget -qO- http://backend-api:8080

echo ""
echo "Testing internal communication to data-service:"
kubectl exec -n microservices test-pod -- wget -qO- http://data-service:8080
```

---

## Step 12 – Install NGINX Ingress Controller

```bash
# Add NGINX ingress Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

echo "NGINX ingress controller installed"
```

---

## Step 13 – Wait for Ingress Controller

```bash
# Wait for ingress controller to be ready
echo "Waiting for ingress controller..."
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s

# Get ingress controller service
kubectl get service -n ingress-nginx
```

---

## Step 14 – Get Ingress Public IP

```bash
# Wait for external IP
echo "Waiting for ingress IP assignment..."
sleep 30

# Get ingress public IP
INGRESS_IP=$(kubectl get service ingress-nginx-controller -n ingress-nginx \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "$INGRESS_IP"
```

---

## Step 15 – Create Ingress Rules

```bash
# Create ingress resource with path-based routing
cat > ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: microservices
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-api
            port:
              number: 8080
      - path: /data
        pathType: Prefix
        backend:
          service:
            name: data-service
            port:
              number: 8080
EOF

# Deploy ingress
kubectl apply -f ingress.yaml

echo "Ingress rules deployed"
```

---

## Step 16 – Verify Ingress Configuration

```bash
# Check ingress status
kubectl get ingress -n microservices

# Describe ingress
kubectl describe ingress microservices-ingress -n microservices
```

---

## Step 17 – Test Ingress Routing

```bash
# Wait for ingress to be ready
echo "Waiting for ingress configuration..."
sleep 30

# Test frontend via ingress
echo "Testing frontend via ingress:"
curl -s "http://${INGRESS_IP}/" | grep -o '<h1>.*</h1>'

# Test backend API via ingress
echo "Testing backend API via ingress:"
curl -s "http://${INGRESS_IP}/api"

echo ""
echo "Testing data service via ingress:"
curl -s "http://${INGRESS_IP}/data"
```

---

## Step 18 – View AKS Cluster Info

```bash
# Show cluster info
kubectl cluster-info

# Show node details
kubectl get nodes -o wide

# Show pods across all namespaces
kubectl get pods --all-namespaces -o wide
```

---

## Step 19 – View Service Endpoints

```bash
# Show service endpoints
kubectl get endpoints -n microservices

# Describe backend API service
kubectl describe service backend-api -n microservices
```

---

## Step 20 – Scale Microservices

```bash
# Scale backend API
kubectl scale deployment backend-api -n microservices --replicas=3

# Scale data service
kubectl scale deployment data-service -n microservices --replicas=3

# Verify scaling
kubectl get deployments -n microservices

# Wait for scaling to complete
kubectl wait --for=condition=available --timeout=60s deployment/backend-api -n microservices
kubectl wait --for=condition=available --timeout=60s deployment/data-service -n microservices

echo "Microservices scaled"
```

---

## Step 21 – Cleanup

```bash
# Delete Kubernetes resources
kubectl delete -f ingress.yaml
kubectl delete -f frontend.yaml
kubectl delete -f data-service.yaml
kubectl delete -f backend-api.yaml
kubectl delete namespace microservices
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx

# Delete AKS cluster and resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f backend-api.yaml data-service.yaml frontend.yaml ingress.yaml

echo "Cleanup complete"
```

---

## Summary

You created an AKS cluster with Azure CNI networking, deployed multiple microservices with ClusterIP and LoadBalancer services, installed NGINX ingress controller, configured path-based routing, validated service-to-service communication within the cluster, and exposed services via ingress with a public IP.
