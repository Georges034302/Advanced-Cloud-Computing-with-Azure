# Lab 1.B: Deploy Secure Multi-VM Python API

## Overview
This lab demonstrates deploying a multi-tier Python Flask API across multiple Azure VMs with secure networking. You'll create a backend API VM and a frontend web VM, configure Network Security Groups for isolation, and enable communication between tiers while maintaining security.

---

## Objectives
- Deploy multi-tier architecture with separate VMs
- Create isolated subnets for frontend and backend
- Configure NSG rules for secure tier communication
- Deploy Python Flask API on backend VM
- Deploy frontend web application
- Test end-to-end connectivity
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- SSH client available
- Basic Python and networking knowledge
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="eastus"
RG_NAME="rg-lab1b-multivm"
VNET_NAME="vnet-lab1b"
SUBNET_FRONTEND="subnet-frontend"
SUBNET_BACKEND="subnet-backend"
NSG_FRONTEND="nsg-frontend"
NSG_BACKEND="nsg-backend"
VM_FRONTEND="vm-frontend"
VM_BACKEND="vm-backend"
ADMIN_USER="azureuser"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "VNET_NAME=$VNET_NAME"
echo "VM_FRONTEND=$VM_FRONTEND"
echo "VM_BACKEND=$VM_BACKEND"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for multi-VM deployment
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Virtual Network with Subnets

```bash
# Create VNet with address space 10.1.0.0/16
az network vnet create \
  --resource-group "$RG_NAME" \
  --name "$VNET_NAME" \
  --address-prefix 10.1.0.0/16 \
  --subnet-name "$SUBNET_FRONTEND" \
  --subnet-prefix 10.1.1.0/24 \
  --location "$LOCATION"

# Create backend subnet
az network vnet subnet create \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --name "$SUBNET_BACKEND" \
  --address-prefix 10.1.2.0/24
```

---

## Step 4 – Create Network Security Groups

```bash
# Get your public IP for SSH access
MY_IP=$(curl -s https://api.ipify.org)
echo "Your public IP: $MY_IP"

# Create NSG for frontend subnet
az network nsg create \
  --resource-group "$RG_NAME" \
  --name "$NSG_FRONTEND" \
  --location "$LOCATION"

# Allow SSH from your IP to frontend
az network nsg rule create \
  --resource-group "$RG_NAME" \
  --nsg-name "$NSG_FRONTEND" \
  --name "AllowSSH" \
  --priority 100 \
  --source-address-prefixes "$MY_IP/32" \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp

# Allow HTTP from internet to frontend
az network nsg rule create \
  --resource-group "$RG_NAME" \
  --nsg-name "$NSG_FRONTEND" \
  --name "AllowHTTP" \
  --priority 110 \
  --source-address-prefixes "Internet" \
  --destination-port-ranges 80 \
  --access Allow \
  --protocol Tcp

# Associate NSG with frontend subnet
az network vnet subnet update \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --name "$SUBNET_FRONTEND" \
  --network-security-group "$NSG_FRONTEND"

# Create NSG for backend subnet
az network nsg create \
  --resource-group "$RG_NAME" \
  --name "$NSG_BACKEND" \
  --location "$LOCATION"

# Allow API traffic from frontend subnet only
az network nsg rule create \
  --resource-group "$RG_NAME" \
  --nsg-name "$NSG_BACKEND" \
  --name "AllowAPIFromFrontend" \
  --priority 100 \
  --source-address-prefixes "10.1.1.0/24" \
  --destination-port-ranges 5000 \
  --access Allow \
  --protocol Tcp

# Allow SSH from frontend subnet for management
az network nsg rule create \
  --resource-group "$RG_NAME" \
  --nsg-name "$NSG_BACKEND" \
  --name "AllowSSHFromFrontend" \
  --priority 110 \
  --source-address-prefixes "10.1.1.0/24" \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp

# Associate NSG with backend subnet
az network vnet subnet update \
  --resource-group "$RG_NAME" \
  --vnet-name "$VNET_NAME" \
  --name "$SUBNET_BACKEND" \
  --network-security-group "$NSG_BACKEND"
```

---

## Step 5 – Generate SSH Key

```bash
# Generate SSH key pair for VM access
SSH_KEY_PATH="$HOME/.ssh/id_rsa_lab1b"

if [ ! -f "$SSH_KEY_PATH" ]; then
  ssh-keygen -t rsa -b 4096 -f "$SSH_KEY_PATH" -N "" -C "$ADMIN_USER@lab1b"
  echo "✅ SSH key pair created"
else
  echo "✅ SSH key already exists"
fi
```

---

## Step 6 – Deploy Backend API VM

```bash
# Create backend VM without public IP
az vm create \
  --resource-group "$RG_NAME" \
  --name "$VM_BACKEND" \
  --location "$LOCATION" \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_BACKEND" \
  --nsg "" \
  --image "Ubuntu2204" \
  --size "Standard_B2s" \
  --admin-username "$ADMIN_USER" \
  --ssh-key-values "${SSH_KEY_PATH}.pub" \
  --public-ip-address "" \
  --output table

# Get backend VM private IP
BACKEND_PRIVATE_IP=$(az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_BACKEND" \
  --show-details \
  --query privateIps \
  --output tsv)

echo "Backend Private IP: $BACKEND_PRIVATE_IP"
```

---

## Step 7 – Deploy Frontend Web VM

```bash
# Create frontend VM with public IP
az vm create \
  --resource-group "$RG_NAME" \
  --name "$VM_FRONTEND" \
  --location "$LOCATION" \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_FRONTEND" \
  --nsg "" \
  --image "Ubuntu2204" \
  --size "Standard_B2s" \
  --admin-username "$ADMIN_USER" \
  --ssh-key-values "${SSH_KEY_PATH}.pub" \
  --public-ip-sku Standard \
  --output table

# Get frontend VM public IP
FRONTEND_PUBLIC_IP=$(az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_FRONTEND" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "Frontend Public IP: $FRONTEND_PUBLIC_IP"
```

---

## Step 8 – Install Python on Backend VM

```bash
# SSH to frontend VM, then jump to backend VM
# First, copy SSH key to frontend VM for jump host access
scp -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no \
  "$SSH_KEY_PATH" \
  "$ADMIN_USER@$FRONTEND_PUBLIC_IP:~/.ssh/id_rsa"

# Set correct permissions on frontend VM
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$FRONTEND_PUBLIC_IP" "chmod 600 ~/.ssh/id_rsa"

# Install Python and Flask on backend VM via frontend jump host
ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no "$ADMIN_USER@$FRONTEND_PUBLIC_IP" << JUMPEOF
# Connect to backend VM and install Python
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no $ADMIN_USER@$BACKEND_PRIVATE_IP << 'BACKENDEOF'
# Update packages
sudo apt-get update -y

# Install Python and pip
sudo apt-get install -y python3 python3-pip python3-venv

# Verify installation
python3 --version
pip3 --version
BACKENDEOF
JUMPEOF
```

---

## Step 9 – Deploy Flask API on Backend VM

```bash
# Create and deploy Flask API application on backend VM
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$FRONTEND_PUBLIC_IP" << 'JUMPEOF'
ssh -i ~/.ssh/id_rsa $ADMIN_USER@$BACKEND_PRIVATE_IP << 'BACKENDEOF'
# Create application directory
mkdir -p ~/api-app
cd ~/api-app

# Create Flask API application
cat > app.py << 'PYEOF'
from flask import Flask, jsonify
import socket
import os

app = Flask(__name__)

@app.route('/api/health')
def health():
    return jsonify({
        'status': 'healthy',
        'service': 'backend-api',
        'hostname': socket.gethostname()
    }), 200

@app.route('/api/data')
def get_data():
    return jsonify({
        'message': 'Data from secure backend API',
        'hostname': socket.gethostname(),
        'items': [
            {'id': 1, 'name': 'Item 1', 'value': 100},
            {'id': 2, 'name': 'Item 2', 'value': 200},
            {'id': 3, 'name': 'Item 3', 'value': 300}
        ]
    }), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
PYEOF

# Install Flask
pip3 install flask

# Create systemd service for API
sudo tee /etc/systemd/system/api-app.service > /dev/null << 'SERVICEEOF'
[Unit]
Description=Flask API Application
After=network.target

[Service]
Type=simple
User=azureuser
WorkingDirectory=/home/azureuser/api-app
ExecStart=/usr/bin/python3 /home/azureuser/api-app/app.py
Restart=always

[Install]
WantedBy=multi-user.target
SERVICEEOF

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable api-app
sudo systemctl start api-app

# Wait for service to start
sleep 3

# Check service status
sudo systemctl status api-app --no-pager

# Test API locally
curl -s http://localhost:5000/api/health | python3 -m json.tool
BACKENDEOF
JUMPEOF
```

---

## Step 10 – Install Nginx on Frontend VM

```bash
# Install and configure Nginx on frontend VM
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$FRONTEND_PUBLIC_IP" << FRONTENDEOF
# Update packages
sudo apt-get update -y

# Install Nginx
sudo apt-get install -y nginx

# Create simple HTML page that calls backend API
sudo tee /var/www/html/index.html > /dev/null << 'HTMLEOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Multi-VM Python API Demo</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background: #f5f5f5;
        }
        .container {
            background: white;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        h1 { color: #0078d4; }
        button {
            background: #0078d4;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
        }
        button:hover { background: #005a9e; }
        #result {
            margin-top: 20px;
            padding: 15px;
            background: #f0f0f0;
            border-radius: 4px;
            white-space: pre-wrap;
            font-family: monospace;
        }
        .status { color: #107c10; font-weight: bold; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Multi-VM Python API Demo</h1>
        <p>Frontend VM calling Backend API VM securely</p>
        
        <button onclick="testHealth()">Test Backend Health</button>
        <button onclick="getData()">Get Data from Backend</button>
        
        <div id="result"></div>
    </div>

    <script>
        const BACKEND_IP = '$BACKEND_PRIVATE_IP';
        
        async function testHealth() {
            try {
                const response = await fetch(\`http://\${BACKEND_IP}:5000/api/health\`);
                const data = await response.json();
                document.getElementById('result').textContent = 
                    '✅ Backend Health Check:\\n' + JSON.stringify(data, null, 2);
            } catch (error) {
                document.getElementById('result').textContent = 
                    '❌ Error: ' + error.message;
            }
        }

        async function getData() {
            try {
                const response = await fetch(\`http://\${BACKEND_IP}:5000/api/data\`);
                const data = await response.json();
                document.getElementById('result').textContent = 
                    '✅ Backend Data Response:\\n' + JSON.stringify(data, null, 2);
            } catch (error) {
                document.getElementById('result').textContent = 
                    '❌ Error: ' + error.message;
            }
        }
    </script>
</body>
</html>
HTMLEOF

# Configure Nginx as reverse proxy to backend API
sudo tee /etc/nginx/sites-available/default > /dev/null << 'NGINXEOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    
    root /var/www/html;
    index index.html;
    
    server_name _;
    
    location / {
        try_files \$uri \$uri/ =404;
    }
    
    # Proxy API requests to backend
    location /api/ {
        proxy_pass http://$BACKEND_PRIVATE_IP:5000;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}
NGINXEOF

# Test Nginx configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
sudo systemctl status nginx --no-pager
FRONTENDEOF
```

---

## Step 11 – Test Multi-VM Setup

```bash
# Test frontend web interface
echo "🌐 Frontend URL: http://$FRONTEND_PUBLIC_IP"

# Test direct API access from frontend VM
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$FRONTEND_PUBLIC_IP" \
  "curl -s http://$BACKEND_PRIVATE_IP:5000/api/health | python3 -m json.tool"

# Test via Nginx proxy
curl -s "http://$FRONTEND_PUBLIC_IP/api/health"
echo ""
curl -s "http://$FRONTEND_PUBLIC_IP/api/data"

# Open in browser
if command -v xdg-open &> /dev/null; then
  xdg-open "http://$FRONTEND_PUBLIC_IP" &
elif [ -n "$BROWSER" ]; then
  "$BROWSER" "http://$FRONTEND_PUBLIC_IP" &
fi
```

---

## Step 12 – Verify Network Security

```bash
# Verify backend VM has no public IP (should fail)
echo "Testing backend VM isolation (should timeout)..."
timeout 5 ssh -i "$SSH_KEY_PATH" -o ConnectTimeout=5 \
  "$ADMIN_USER@$BACKEND_PRIVATE_IP" "echo 'Should not connect'" 2>&1 || echo "✅ Backend properly isolated"

# Test NSG rules
az network nsg rule list \
  --resource-group "$RG_NAME" \
  --nsg-name "$NSG_BACKEND" \
  --query "[].{Name:name, Priority:priority, Source:sourceAddressPrefix, DestPort:destinationPortRange, Access:access}" \
  --output table

# List VMs and their network configuration
az vm list \
  --resource-group "$RG_NAME" \
  --show-details \
  --query "[].{Name:name, PrivateIP:privateIps, PublicIP:publicIps, Size:hardwareProfile.vmSize}" \
  --output table
```

---

## Step 13 – View Application Logs

```bash
# View backend API logs
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$FRONTEND_PUBLIC_IP" << 'JUMPEOF'
echo "=== Backend API Service Logs ==="
ssh -i ~/.ssh/id_rsa $ADMIN_USER@$BACKEND_PRIVATE_IP \
  "sudo journalctl -u api-app -n 20 --no-pager"
JUMPEOF

# View frontend Nginx access logs
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$FRONTEND_PUBLIC_IP" \
  "sudo tail -20 /var/log/nginx/access.log"
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

# Optional: Remove SSH key
# rm -f "$SSH_KEY_PATH" "${SSH_KEY_PATH}.pub"

echo "✅ Cleanup initiated - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Multi-tier architecture with frontend and backend VMs
- Isolated subnets with Network Security Groups
- Python Flask REST API on backend VM
- Nginx reverse proxy on frontend VM
- Secure inter-tier communication

**Architecture:**
```
Internet → Frontend VM (Nginx) → NSG → Backend VM (Flask API)
         (10.1.1.0/24)              (10.1.2.0/24)
         Public IP                   Private IP only
```

**Key Components:**
- **Frontend VM**: Public-facing web server with Nginx
- **Backend VM**: Private API server with Flask
- **Network Segmentation**: Separate subnets for each tier
- **NSG Rules**: Allow only necessary traffic between tiers
- **Jump Host Pattern**: Access backend via frontend VM

**What You Learned:**
- Deploy multi-tier applications on Azure VMs
- Configure network segmentation and isolation
- Implement jump host pattern for secure access
- Deploy Python Flask APIs as systemd services
- Configure Nginx as reverse proxy
- Apply defense-in-depth security principles

---

## Best Practices

**Network Security:**
- Never expose backend VMs to internet
- Use jump hosts for administrative access
- Apply principle of least privilege in NSG rules
- Segment networks by tier/function
- Use private IPs for inter-tier communication

**Application Architecture:**
- Separate frontend and backend concerns
- Use reverse proxy for API routing
- Implement health check endpoints
- Run applications as systemd services
- Enable application logging

**VM Management:**
- Use managed identities when possible
- Enable automatic security updates
- Configure monitoring and alerts
- Implement backup strategies
- Document network topology

**Cost Optimization:**
- Use B-series VMs for dev/test workloads
- Deallocate VMs when not needed
- Remove unused public IPs
- Consider VM scale sets for production
- Use spot VMs for non-critical workloads

---

## Production Enhancements

**1. Add Load Balancer for Frontend**
```bash
# Create load balancer for frontend tier
az network lb create \
  --resource-group "$RG_NAME" \
  --name "lb-frontend" \
  --sku Standard \
  --vnet-name "$VNET_NAME" \
  --subnet "$SUBNET_FRONTEND" \
  --frontend-ip-name "frontend-ip" \
  --backend-pool-name "backend-pool"
```

**2. Enable Azure Monitor**
```bash
# Enable VM insights
az vm extension set \
  --resource-group "$RG_NAME" \
  --vm-name "$VM_BACKEND" \
  --name "AzureMonitorLinuxAgent" \
  --publisher "Microsoft.Azure.Monitor"
```

**3. Add SSL/TLS to Nginx**
```bash
# Install Certbot for Let's Encrypt
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

**4. Implement Connection Pooling**
```python
# Update Flask app with database connection pooling
from flask_sqlalchemy import SQLAlchemy
app.config['SQLALCHEMY_POOL_SIZE'] = 10
app.config['SQLALCHEMY_POOL_RECYCLE'] = 3600
```

---

## Troubleshooting

**Cannot access frontend VM:**
- Verify NSG allows HTTP (port 80) from Internet
- Check VM is running: `az vm get-instance-view`
- Confirm public IP is attached
- Test Nginx: `sudo systemctl status nginx`
- Check firewall rules on VM

**Backend API not responding:**
- Verify Flask service is running on backend
- Check NSG allows port 5000 from frontend subnet
- Test locally on backend: `curl localhost:5000/api/health`
- Review service logs: `journalctl -u api-app`
- Confirm backend private IP is correct

**Cannot SSH to backend from frontend:**
- Verify SSH key is copied to frontend VM
- Check NSG allows SSH from frontend subnet
- Test connectivity: `ping $BACKEND_PRIVATE_IP`
- Ensure private key permissions: `chmod 600 ~/.ssh/id_rsa`
- Check backend VM is running

**Nginx proxy not working:**
- Verify Nginx configuration: `sudo nginx -t`
- Check backend IP in Nginx config
- Review Nginx error logs: `/var/log/nginx/error.log`
- Test backend API directly from frontend
- Restart Nginx: `sudo systemctl restart nginx`

**Network connectivity issues:**
- Use Azure Network Watcher for diagnostics
- Verify route tables and NSG associations
- Check effective security rules on NICs
- Test with `traceroute` and `tcpdump`
- Review Azure Network Topology

---

## Additional Resources

- [Azure Virtual Network Documentation](https://docs.microsoft.com/azure/virtual-network/)
- [Network Security Groups Best Practices](https://docs.microsoft.com/azure/virtual-network/network-security-groups-overview)
- [Flask Documentation](https://flask.palletsprojects.com/)
- [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [Azure VM Security Best Practices](https://docs.microsoft.com/azure/security/fundamentals/virtual-machines-overview)
