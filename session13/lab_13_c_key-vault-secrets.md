# Lab 13.C: Azure Key Vault Secrets

## Objectives
- Create Azure Key Vault
- Store and retrieve secrets
- Generate encryption keys
- Create certificates
- Configure access policies
- Enable soft delete and purge protection
- Use Key Vault with applications
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- OpenSSL installed
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab13c-keyvault"
KEYVAULT_NAME="kv-lab-$RANDOM"
STORAGE_ACCOUNT="kvstorage$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$KEYVAULT_NAME"
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

## Step 3 – Get Current User Object ID

```bash
# Get current user object ID for access policies
USER_OBJECT_ID=$(az ad signed-in-user show --query id --output tsv)

echo "$USER_OBJECT_ID"
```

---

## Step 4 – Create Key Vault

```bash
# Create Key Vault with soft delete and purge protection
az keyvault create \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --enable-soft-delete true \
  --enable-purge-protection true \
  --retention-days 90

echo "Key Vault created"
```

---

## Step 5 – Set Access Policy for Current User

```bash
# Grant full permissions to current user
az keyvault set-policy \
  --name "$KEYVAULT_NAME" \
  --object-id "$USER_OBJECT_ID" \
  --secret-permissions get list set delete recover backup restore purge \
  --key-permissions get list create delete recover backup restore purge encrypt decrypt \
  --certificate-permissions get list create delete recover backup restore purge

echo "Access policy configured"
```

---

## Step 6 – Create Database Password Secret

```bash
# Read database password
read -s -p "Enter database password to store: " DB_PASSWORD
echo

# Store secret in Key Vault
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "DatabasePassword" \
  --value "$DB_PASSWORD"

echo "Database password stored"
```

---

## Step 7 – Create API Key Secret

```bash
# Store API key secret
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "ApiKey" \
  --value "sk-prod-$(openssl rand -hex 16)" \
  --tags environment=production application=api

echo "API key stored"
```

---

## Step 8 – Create Connection String Secret

```bash
# Store connection string
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "StorageConnectionString" \
  --value "DefaultEndpointsProtocol=https;AccountName=myaccount;AccountKey=secret123" \
  --content-type "text/plain"

echo "Connection string stored"
```

---

## Step 9 – List All Secrets

```bash
# List all secrets in vault
az keyvault secret list \
  --vault-name "$KEYVAULT_NAME" \
  --query "[].{Name:name, Enabled:attributes.enabled, Created:attributes.created}" \
  --output table
```

---

## Step 10 – Retrieve Secret Value

```bash
# Get API key secret value
API_KEY=$(az keyvault secret show \
  --vault-name "$KEYVAULT_NAME" \
  --name "ApiKey" \
  --query value \
  --output tsv)

echo "$API_KEY"
```

---

## Step 11 – Create Encryption Key

```bash
# Create RSA encryption key
az keyvault key create \
  --vault-name "$KEYVAULT_NAME" \
  --name "DataEncryptionKey" \
  --kty RSA \
  --size 2048 \
  --ops encrypt decrypt

echo "Encryption key created"
```

---

## Step 12 – Create Signing Key

```bash
# Create key for signing operations
az keyvault key create \
  --vault-name "$KEYVAULT_NAME" \
  --name "SigningKey" \
  --kty RSA \
  --size 4096 \
  --ops sign verify

echo "Signing key created"
```

---

## Step 13 – List All Keys

```bash
# List all keys in vault
az keyvault key list \
  --vault-name "$KEYVAULT_NAME" \
  --query "[].{Name:name, KeyType:keyType, Enabled:attributes.enabled}" \
  --output table
```

---

## Step 14 – Show Key Details

```bash
# Get encryption key details
az keyvault key show \
  --vault-name "$KEYVAULT_NAME" \
  --name "DataEncryptionKey" \
  --query "{Name:key.kid, KeyType:key.kty, KeySize:key.n}" \
  --output table
```

---

## Step 15 – Create Self-Signed Certificate

```bash
# Create certificate policy file
cat > cert-policy.json << 'EOF'
{
  "issuerParameters": {
    "name": "Self"
  },
  "keyProperties": {
    "exportable": true,
    "keySize": 2048,
    "keyType": "RSA",
    "reuseKey": false
  },
  "secretProperties": {
    "contentType": "application/x-pkcs12"
  },
  "x509CertificateProperties": {
    "subject": "CN=lab.example.com",
    "validityInMonths": 12,
    "subjectAlternativeNames": {
      "dnsNames": ["lab.example.com", "www.lab.example.com"]
    }
  }
}
EOF

# Create certificate
az keyvault certificate create \
  --vault-name "$KEYVAULT_NAME" \
  --name "WebServerCert" \
  --policy @cert-policy.json

echo "Certificate created"
```

---

## Step 16 – List Certificates

```bash
# List all certificates
az keyvault certificate list \
  --vault-name "$KEYVAULT_NAME" \
  --query "[].{Name:name, Enabled:attributes.enabled, Expires:attributes.expires}" \
  --output table
```

---

## Step 17 – Download Certificate

```bash
# Download certificate in PEM format
az keyvault certificate download \
  --vault-name "$KEYVAULT_NAME" \
  --name "WebServerCert" \
  --file webserver.pem \
  --encoding PEM

echo "Certificate downloaded"
```

---

## Step 18 – Create Secret Version

```bash
# Update existing secret (creates new version)
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "ApiKey" \
  --value "sk-prod-$(openssl rand -hex 16)"

echo "New secret version created"
```

---

## Step 19 – List Secret Versions

```bash
# Show all versions of a secret
az keyvault secret list-versions \
  --vault-name "$KEYVAULT_NAME" \
  --name "ApiKey" \
  --query "[].{Version:id, Created:attributes.created, Enabled:attributes.enabled}" \
  --output table
```

---

## Step 20 – Enable Secret Expiration

```bash
# Set expiration date for secret
EXPIRY_DATE=$(date -u -d "+30 days" +"%Y-%m-%dT%H:%M:%SZ")

az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "TemporaryToken" \
  --value "temp-token-12345" \
  --expires "$EXPIRY_DATE"

echo "Secret with expiration created"
```

---

## Step 21 – Create Storage Account with Key Vault

```bash
# Create storage account
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

echo "Storage account created"
```

---

## Step 22 – Store Storage Key in Vault

```bash
# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --account-name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "[0].value" \
  --output tsv)

# Store in Key Vault
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "StorageAccountKey" \
  --value "$STORAGE_KEY"

echo "Storage key stored in vault"
```

---

## Step 23 – Create Application to Use Key Vault

```bash
# Create Python script to access Key Vault
cat > use-keyvault.py << EOF
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

# Key Vault URL
vault_url = "https://${KEYVAULT_NAME}.vault.azure.net"

# Authenticate
credential = DefaultAzureCredential()
client = SecretClient(vault_url=vault_url, credential=credential)

# Retrieve secret
try:
    secret = client.get_secret("ApiKey")
    print(f"Secret value: {secret.value}")
    print(f"Secret version: {secret.properties.version}")
except Exception as e:
    print(f"Error: {e}")
EOF

echo "Application script created"
```

---

## Step 24 – Enable Logging and Monitoring

```bash
# Create Log Analytics workspace
LOG_WORKSPACE="log-keyvault-$RANDOM"

az monitor log-analytics workspace create \
  --name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION"

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --name "$LOG_WORKSPACE" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "$WORKSPACE_ID"

# Enable diagnostic logs
az monitor diagnostic-settings create \
  --name "keyvault-diagnostics" \
  --resource "$KEYVAULT_NAME" \
  --resource-group "$RG_NAME" \
  --resource-type "Microsoft.KeyVault/vaults" \
  --workspace "$WORKSPACE_ID" \
  --logs '[{"category":"AuditEvent","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

echo "Diagnostics enabled"
```

---

## Step 25 – Show Key Vault Properties

```bash
# Display Key Vault configuration
az keyvault show \
  --name "$KEYVAULT_NAME" \
  --query "{Name:name, Location:location, SKU:sku.name, SoftDelete:properties.enableSoftDelete, PurgeProtection:properties.enablePurgeProtection, VaultUri:properties.vaultUri}" \
  --output table
```

---

## Step 26 – Backup Secret

```bash
# Backup a secret
az keyvault secret backup \
  --vault-name "$KEYVAULT_NAME" \
  --name "DatabasePassword" \
  --file db-password-backup.blob

echo "Secret backed up"
```

---

## Step 27 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f cert-policy.json webserver.pem use-keyvault.py db-password-backup.blob

echo "Cleanup complete"
```

---

## Summary

You created Azure Key Vault with soft delete and purge protection enabled, stored multiple types of secrets including passwords, API keys, and connection strings, generated RSA encryption and signing keys, created self-signed certificates with custom policies, configured access policies for secure access, enabled secret versioning and expiration, integrated storage account keys with Key Vault, created application code to retrieve secrets programmatically, and enabled diagnostic logging for audit and compliance.
