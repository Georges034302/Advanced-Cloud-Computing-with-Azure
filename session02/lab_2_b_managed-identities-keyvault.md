# Lab 2.B: Managed Identities & Key Vault

## Overview
This lab demonstrates using Azure Managed Identities to securely access Azure Key Vault secrets without storing credentials in code. You'll create a VM with managed identity, configure Key Vault access policies, and retrieve secrets programmatically.

---

## Objectives
- Create Azure Key Vault with secrets
- Deploy VM with system-assigned managed identity
- Configure Key Vault access policies for managed identity
- Retrieve secrets using managed identity from VM
- Test user-assigned managed identity
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- SSH client available
- Basic understanding of managed identities
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="eastus"
RG_NAME="rg-lab2b-identity"
KEYVAULT_NAME="kv-lab2b-$RANDOM"
VM_NAME="vm-identity-test"
ADMIN_USER="azureuser"
MANAGED_IDENTITY_NAME="mi-lab2b"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "KEYVAULT_NAME=$KEYVAULT_NAME"
echo "VM_NAME=$VM_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for managed identity resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Key Vault

```bash
# Create Key Vault (RBAC authorization model)
az keyvault create \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --enable-rbac-authorization false

# Get Key Vault resource ID
KV_ID=$(az keyvault show \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "Key Vault ID: $KV_ID"
```

---

## Step 4 – Add Secrets to Key Vault

```bash
# Add database connection string secret
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "db-connection-string" \
  --value "Server=myserver.database.windows.net;Database=mydb;User=admin;Password=P@ssw0rd123"

# Add API key secret
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "api-key" \
  --value "sk-1234567890abcdef"

# Add storage account key
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "storage-key" \
  --value "DefaultEndpointsProtocol=https;AccountName=mystorage;AccountKey=key123"

# List secrets
az keyvault secret list \
  --vault-name "$KEYVAULT_NAME" \
  --query "[].{Name:name, Enabled:attributes.enabled}" \
  --output table
```

---

## Step 5 – Create VM with System-Assigned Managed Identity

```bash
# Generate SSH key
SSH_KEY_PATH="$HOME/.ssh/id_rsa_lab2b"
if [ ! -f "$SSH_KEY_PATH" ]; then
  ssh-keygen -t rsa -b 4096 -f "$SSH_KEY_PATH" -N "" -C "$ADMIN_USER@lab2b"
fi

# Create VM with system-assigned managed identity enabled
az vm create \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --location "$LOCATION" \
  --image "Ubuntu2204" \
  --size "Standard_B2s" \
  --admin-username "$ADMIN_USER" \
  --ssh-key-values "${SSH_KEY_PATH}.pub" \
  --assign-identity \
  --public-ip-sku Standard \
  --output table

# Get VM managed identity principal ID
VM_IDENTITY_ID=$(az vm identity show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query principalId \
  --output tsv)

echo "VM Managed Identity Principal ID: $VM_IDENTITY_ID"
```

---

## Step 6 – Grant Key Vault Access to Managed Identity

```bash
# Set Key Vault access policy for VM's managed identity
az keyvault set-policy \
  --name "$KEYVAULT_NAME" \
  --object-id "$VM_IDENTITY_ID" \
  --secret-permissions get list

# Verify access policy
az keyvault show \
  --name "$KEYVAULT_NAME" \
  --query "properties.accessPolicies[?objectId=='$VM_IDENTITY_ID'].{ObjectId:objectId, Permissions:permissions}" \
  --output table
```

---

## Step 7 – Get VM Public IP

```bash
# Retrieve VM public IP address
VM_PUBLIC_IP=$(az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --show-details \
  --query publicIps \
  --output tsv)

echo "VM Public IP: $VM_PUBLIC_IP"
```

---

## Step 8 – Install Azure CLI on VM

```bash
# SSH to VM and install Azure CLI
ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no "$ADMIN_USER@$VM_PUBLIC_IP" << 'EOFVM'
# Update package list
sudo apt-get update -y

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verify installation
az version
EOFVM
```

---

## Step 9 – Retrieve Secrets Using Managed Identity

```bash
# SSH to VM and use managed identity to access Key Vault
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << EOFVM
# Login using managed identity
az login --identity

# Verify logged in with managed identity
az account show --query "{Subscription:name, User:user.name, Type:user.type}" --output table

# Retrieve database connection string secret
echo "=== Retrieving db-connection-string ==="
az keyvault secret show \
  --vault-name "$KEYVAULT_NAME" \
  --name "db-connection-string" \
  --query "value" \
  --output tsv

# Retrieve API key secret
echo -e "\n=== Retrieving api-key ==="
az keyvault secret show \
  --vault-name "$KEYVAULT_NAME" \
  --name "api-key" \
  --query "value" \
  --output tsv

# List all accessible secrets
echo -e "\n=== Listing all secrets ==="
az keyvault secret list \
  --vault-name "$KEYVAULT_NAME" \
  --query "[].name" \
  --output table
EOFVM
```

---

## Step 10 – Create Python Script to Access Secrets

```bash
# Create Python script on VM that uses managed identity
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << 'EOFVM'
# Install Python packages
sudo apt-get install -y python3-pip
pip3 install azure-identity azure-keyvault-secrets

# Create Python script
cat > get_secrets.py << 'EOFPY'
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
import os

# Key Vault name from environment or hardcode
vault_name = os.getenv('KEY_VAULT_NAME', '$KEYVAULT_NAME')
vault_url = f"https://{vault_name}.vault.azure.net"

print(f"Connecting to Key Vault: {vault_url}")

# Authenticate using managed identity (DefaultAzureCredential)
credential = DefaultAzureCredential()
client = SecretClient(vault_url=vault_url, credential=credential)

# Retrieve secrets
try:
    # Get database connection string
    db_secret = client.get_secret("db-connection-string")
    print(f"\n✅ Retrieved secret: db-connection-string")
    print(f"Value: {db_secret.value[:50]}...")
    
    # Get API key
    api_secret = client.get_secret("api-key")
    print(f"\n✅ Retrieved secret: api-key")
    print(f"Value: {api_secret.value}")
    
    # List all secrets
    print(f"\n📋 Available secrets:")
    for secret in client.list_properties_of_secrets():
        print(f"  - {secret.name}")
        
except Exception as e:
    print(f"❌ Error: {e}")
EOFPY

# Replace placeholder with actual vault name
sed -i "s/\$KEYVAULT_NAME/$KEYVAULT_NAME/g" get_secrets.py

# Run Python script
echo "=== Running Python script ==="
python3 get_secrets.py
EOFVM
```

---

## Step 11 – Create User-Assigned Managed Identity

```bash
# Create user-assigned managed identity
az identity create \
  --resource-group "$RG_NAME" \
  --name "$MANAGED_IDENTITY_NAME" \
  --location "$LOCATION"

# Get user-assigned identity details
USER_IDENTITY_ID=$(az identity show \
  --resource-group "$RG_NAME" \
  --name "$MANAGED_IDENTITY_NAME" \
  --query id \
  --output tsv)

USER_IDENTITY_PRINCIPAL_ID=$(az identity show \
  --resource-group "$RG_NAME" \
  --name "$MANAGED_IDENTITY_NAME" \
  --query principalId \
  --output tsv)

echo "User-Assigned Identity ID: $USER_IDENTITY_ID"
echo "User-Assigned Identity Principal ID: $USER_IDENTITY_PRINCIPAL_ID"
```

---

## Step 12 – Assign User-Assigned Identity to VM

```bash
# Assign user-assigned identity to VM
az vm identity assign \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --identities "$USER_IDENTITY_ID"

# Grant Key Vault access to user-assigned identity
az keyvault set-policy \
  --name "$KEYVAULT_NAME" \
  --object-id "$USER_IDENTITY_PRINCIPAL_ID" \
  --secret-permissions get list

# List all identities assigned to VM
az vm identity show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query "{Type:type, SystemAssigned:principalId, UserAssigned:userAssignedIdentities}" \
  --output json
```

---

## Step 13 – Test User-Assigned Identity

```bash
# SSH to VM and test user-assigned identity
ssh -i "$SSH_KEY_PATH" "$ADMIN_USER@$VM_PUBLIC_IP" << EOFVM
# Get client ID of user-assigned identity
USER_CLIENT_ID=\$(curl -s 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net&client_id=$USER_IDENTITY_PRINCIPAL_ID' -H Metadata:true | python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'][:50])")

echo "User-assigned identity token retrieved (first 50 chars): \$USER_CLIENT_ID..."

# Login with user-assigned managed identity
az login --identity --username "$USER_IDENTITY_ID"

# Retrieve secret using user-assigned identity
az keyvault secret show \
  --vault-name "$KEYVAULT_NAME" \
  --name "storage-key" \
  --query "value" \
  --output tsv
EOFVM
```

---

## Step 14 – View Key Vault Access Policies

```bash
# List all access policies on Key Vault
az keyvault show \
  --name "$KEYVAULT_NAME" \
  --query "properties.accessPolicies[].{ObjectId:objectId, Permissions:permissions.secrets}" \
  --output table

# Get Key Vault audit logs (requires diagnostic settings)
az monitor activity-log list \
  --resource-group "$RG_NAME" \
  --query "[?contains(resourceId, '$KEYVAULT_NAME')].{Time:eventTimestamp, Operation:operationName.localizedValue, Status:status.localizedValue}" \
  --output table
```

---

## Step 15 – Cleanup

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

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Azure Key Vault with multiple secrets
- VM with system-assigned managed identity
- User-assigned managed identity
- Key Vault access policies for managed identities
- Python application using managed identity authentication

**Architecture:**
```
VM (Managed Identity) → Azure AD Authentication → Key Vault Access Policy → Secrets
```

**Key Components:**
- **Managed Identity**: Azure AD identity for resources (no credentials needed)
- **System-Assigned Identity**: Lifecycle tied to resource
- **User-Assigned Identity**: Independent lifecycle, reusable
- **Key Vault**: Secure storage for secrets, keys, certificates
- **Access Policies**: Grant permissions to managed identities

**What You Learned:**
- Create and configure Azure Key Vault
- Enable system-assigned managed identities
- Create and assign user-assigned managed identities
- Configure Key Vault access policies
- Retrieve secrets using managed identity from code
- Use DefaultAzureCredential for authentication

---

## Best Practices

**Managed Identity Usage:**
- Prefer managed identities over service principals
- Use system-assigned for single-resource scenarios
- Use user-assigned for multi-resource scenarios
- Never hardcode credentials in application code
- Use DefaultAzureCredential for local development

**Key Vault Security:**
- Enable soft-delete and purge protection
- Use separate Key Vaults for different environments
- Implement least privilege access policies
- Enable Key Vault firewall and private endpoints
- Monitor Key Vault access logs

**Secret Management:**
- Rotate secrets regularly
- Use secret versioning
- Set expiration dates on secrets
- Never log or display secret values
- Use Key Vault references in App Service

**Access Control:**
- Grant minimal required permissions
- Use RBAC instead of access policies when possible
- Review access policies regularly
- Implement conditional access for users
- Enable Azure AD Privileged Identity Management

---

## Production Enhancements

**1. Enable Key Vault Firewall**
```bash
# Restrict Key Vault access to specific networks
az keyvault network-rule add \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RG_NAME" \
  --vnet-name "your-vnet" \
  --subnet "your-subnet"

az keyvault update \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RG_NAME" \
  --default-action Deny
```

**2. Enable Soft-Delete and Purge Protection**
```bash
# Enable soft-delete (90-day retention)
az keyvault update \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RG_NAME" \
  --enable-soft-delete true \
  --retention-days 90

# Enable purge protection
az keyvault update \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RG_NAME" \
  --enable-purge-protection true
```

**3. Configure Diagnostic Logs**
```bash
# Send Key Vault logs to Log Analytics
az monitor diagnostic-settings create \
  --name "kv-diagnostics" \
  --resource "$KV_ID" \
  --logs '[{"category":"AuditEvent","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --workspace "/subscriptions/.../workspaces/your-workspace"
```

**4. Use Private Endpoint**
```bash
# Create private endpoint for Key Vault
az network private-endpoint create \
  --resource-group "$RG_NAME" \
  --name "pe-keyvault" \
  --vnet-name "your-vnet" \
  --subnet "your-subnet" \
  --private-connection-resource-id "$KV_ID" \
  --group-id vault \
  --connection-name "kv-connection"
```

---

## Troubleshooting

**Cannot access Key Vault from VM:**
- Verify managed identity is enabled on VM
- Check Key Vault access policy includes managed identity
- Confirm network connectivity to Key Vault
- Verify Key Vault firewall rules
- Check for service endpoint or private endpoint requirements

**Access denied errors:**
- Verify access policy permissions (get, list, etc.)
- Check managed identity principal ID is correct
- Wait for access policy propagation (up to 5 minutes)
- Confirm Key Vault name is correct
- Review Azure AD authentication token

**Managed identity authentication fails:**
- Verify VM has managed identity enabled
- Check DefaultAzureCredential is properly configured
- Ensure azure-identity SDK is installed
- Confirm IMDS endpoint is accessible (169.254.169.254)
- Review application error messages

**Python script errors:**
- Install required packages: azure-identity, azure-keyvault-secrets
- Check Python version (3.6+)
- Verify Key Vault URL format
- Review exception messages for details
- Test Azure CLI access first

**User-assigned identity not working:**
- Verify identity is assigned to VM
- Check access policy includes user-assigned identity
- Confirm client ID is correct
- Allow time for identity propagation
- Review identity assignment in portal

---

## Additional Resources

- [Azure Managed Identities Documentation](https://docs.microsoft.com/azure/active-directory/managed-identities-azure-resources/)
- [Azure Key Vault Overview](https://docs.microsoft.com/azure/key-vault/general/overview)
- [Key Vault Access Policies](https://docs.microsoft.com/azure/key-vault/general/assign-access-policy)
- [Azure Identity SDK for Python](https://docs.microsoft.com/python/api/overview/azure/identity-readme)
- [DefaultAzureCredential Documentation](https://docs.microsoft.com/python/api/azure-identity/azure.identity.defaultazurecredential)
