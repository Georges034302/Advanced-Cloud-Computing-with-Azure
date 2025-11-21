# Lab 2.D: Service Principals & RBAC

## Overview
This lab demonstrates creating and managing Azure Service Principals for application authentication. You'll create service principals with password and certificate authentication, assign RBAC roles, and use them to access Azure resources programmatically.

---

## Objectives
- Create service principals with password authentication
- Create service principals with certificate authentication
- Assign RBAC roles to service principals
- Authenticate and access Azure resources using service principals
- Manage service principal credentials and secrets
- Clean up service principals and role assignments

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Application Administrator or Global Administrator role
- OpenSSL installed for certificate generation
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure configuration
LOCATION="eastus"
RG_NAME="rg-lab2d-sp"
SP_NAME="sp-lab2d-password"
SP_CERT_NAME="sp-lab2d-cert"
STORAGE_ACCOUNT="lab2dsp$RANDOM"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "SP_NAME=$SP_NAME"
echo "SP_CERT_NAME=$SP_CERT_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for testing
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"

# Get subscription ID
SUBSCRIPTION_ID=$(az account show --query id --output tsv)
echo "Subscription ID: $SUBSCRIPTION_ID"
```

---

## Step 3 – Create Service Principal with Password

```bash
# Create service principal with auto-generated password
SP_OUTPUT=$(az ad sp create-for-rbac \
  --name "$SP_NAME" \
  --role "Reader" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME" \
  --output json)

# Extract credentials
SP_APP_ID=$(echo $SP_OUTPUT | jq -r '.appId')
SP_PASSWORD=$(echo $SP_OUTPUT | jq -r '.password')
SP_TENANT=$(echo $SP_OUTPUT | jq -r '.tenant')

echo "=== Service Principal Created ==="
echo "Application (client) ID: $SP_APP_ID"
echo "Client Secret: $SP_PASSWORD"
echo "Tenant ID: $SP_TENANT"
echo "⚠️  Save these credentials - password cannot be retrieved later"
```

---

## Step 4 – Verify Service Principal

```bash
# Get service principal details
az ad sp show \
  --id "$SP_APP_ID" \
  --query "{DisplayName:displayName, AppId:appId, ObjectId:id}" \
  --output table

# List service principal credentials
az ad sp credential list \
  --id "$SP_APP_ID" \
  --query "[].{KeyId:keyId, Type:type, StartDate:startDateTime, EndDate:endDateTime}" \
  --output table
```

---

## Step 5 – Test Service Principal Authentication

```bash
# Login with service principal
az login --service-principal \
  --username "$SP_APP_ID" \
  --password "$SP_PASSWORD" \
  --tenant "$SP_TENANT"

# Verify authentication
az account show \
  --query "{Subscription:name, User:user.name, Type:user.type}" \
  --output table

# Test permissions - list resources in resource group
az resource list \
  --resource-group "$RG_NAME" \
  --output table

# Logout from service principal
az logout

# Login back with your user account
az login
```

---

## Step 6 – Create Storage Account for Testing

```bash
# Create storage account
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

# Get storage account resource ID
STORAGE_ID=$(az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query id \
  --output tsv)

echo "Storage Account ID: $STORAGE_ID"
```

---

## Step 7 – Assign Additional RBAC Role to Service Principal

```bash
# Get service principal object ID
SP_OBJECT_ID=$(az ad sp show \
  --id "$SP_APP_ID" \
  --query id \
  --output tsv)

# Assign Storage Blob Data Contributor role at storage account scope
az role assignment create \
  --assignee "$SP_OBJECT_ID" \
  --role "Storage Blob Data Contributor" \
  --scope "$STORAGE_ID"

# List all role assignments for service principal
az role assignment list \
  --assignee "$SP_OBJECT_ID" \
  --all \
  --query "[].{Role:roleDefinitionName, Scope:scope}" \
  --output table
```

---

## Step 8 – Create Certificate for Service Principal Authentication

```bash
# Create self-signed certificate
CERT_NAME="sp-cert-lab2d"
openssl req -x509 -newkey rsa:4096 -keyout "${CERT_NAME}.key" -out "${CERT_NAME}.crt" -sha256 -days 365 -nodes -subj "/CN=SPCertAuth"

# Create PEM file combining key and cert
cat "${CERT_NAME}.key" "${CERT_NAME}.crt" > "${CERT_NAME}.pem"

# Verify certificate created
ls -lh ${CERT_NAME}.*
openssl x509 -in "${CERT_NAME}.crt" -noout -subject -dates
```

---

## Step 9 – Create Service Principal with Certificate

```bash
# Create service principal with certificate
az ad sp create-for-rbac \
  --name "$SP_CERT_NAME" \
  --role "Contributor" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME" \
  --cert "@${CERT_NAME}.pem" \
  --create-cert

# Get certificate-based SP details
SP_CERT_APP_ID=$(az ad sp list \
  --display-name "$SP_CERT_NAME" \
  --query "[0].appId" \
  --output tsv)

echo "Certificate-based SP App ID: $SP_CERT_APP_ID"
```

---

## Step 10 – Test Certificate-Based Authentication

```bash
# Login with service principal using certificate
az login --service-principal \
  --username "$SP_CERT_APP_ID" \
  --tenant "$SP_TENANT" \
  --password "${CERT_NAME}.pem"

# Verify authentication
az account show \
  --query "{Name:name, User:user.name, Type:user.type}" \
  --output table

# Test contributor permissions - create a resource
az storage container create \
  --name "test-container" \
  --account-name "$STORAGE_ACCOUNT" \
  --auth-mode login

# Logout and return to user account
az logout
az login
```

---

## Step 11 – Create Application Registration Separately

```bash
# Create Azure AD application
APP_NAME="app-lab2d-manual"
az ad app create \
  --display-name "$APP_NAME"

# Get application ID
APP_ID=$(az ad app list \
  --display-name "$APP_NAME" \
  --query "[0].appId" \
  --output tsv)

echo "Application ID: $APP_ID"

# Create service principal for the application
az ad sp create \
  --id "$APP_ID"

# Get service principal object ID
SP_MANUAL_OBJECT_ID=$(az ad sp show \
  --id "$APP_ID" \
  --query id \
  --output tsv)

echo "Service Principal Object ID: $SP_MANUAL_OBJECT_ID"
```

---

## Step 12 – Create Client Secret for Application

```bash
# Add client secret to application
SECRET_OUTPUT=$(az ad app credential reset \
  --id "$APP_ID" \
  --append \
  --display-name "lab2d-secret" \
  --years 1 \
  --output json)

APP_SECRET=$(echo $SECRET_OUTPUT | jq -r '.password')

echo "Application Client Secret: $APP_SECRET"
echo "⚠️  Save this secret securely"

# Assign role to the service principal
az role assignment create \
  --assignee "$SP_MANUAL_OBJECT_ID" \
  --role "Reader" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME"
```

---

## Step 13 – Use Service Principal in Scripts

```bash
# Create script that uses service principal
cat > sp-test-script.sh << 'EOFSCRIPT'
#!/bin/bash
set -e

# Service principal credentials (would be from environment variables in production)
SP_APP_ID="$1"
SP_PASSWORD="$2"
SP_TENANT="$3"
RG_NAME="$4"

echo "Authenticating with service principal..."
az login --service-principal \
  --username "$SP_APP_ID" \
  --password "$SP_PASSWORD" \
  --tenant "$SP_TENANT" \
  --output none

echo "Listing resources in resource group..."
az resource list \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Type:type, Location:location}" \
  --output table

echo "Logging out..."
az logout
EOFSCRIPT

chmod +x sp-test-script.sh

# Run script
./sp-test-script.sh "$SP_APP_ID" "$SP_PASSWORD" "$SP_TENANT" "$RG_NAME"
```

---

## Step 14 – Rotate Service Principal Credentials

```bash
# Reset service principal password
NEW_CREDENTIALS=$(az ad sp credential reset \
  --id "$SP_APP_ID" \
  --output json)

NEW_PASSWORD=$(echo $NEW_CREDENTIALS | jq -r '.password')

echo "New Service Principal Password: $NEW_PASSWORD"
echo "⚠️  Old password is now invalid"

# Verify new credentials work
az login --service-principal \
  --username "$SP_APP_ID" \
  --password "$NEW_PASSWORD" \
  --tenant "$SP_TENANT"

az account show --query name
az logout
az login
```

---

## Step 15 – List All Service Principals and Role Assignments

```bash
# List all service principals created in this lab
az ad sp list \
  --display-name "sp-lab2d" \
  --query "[].{DisplayName:displayName, AppId:appId, ObjectId:id}" \
  --output table

# List all role assignments in resource group
az role assignment list \
  --resource-group "$RG_NAME" \
  --query "[].{Principal:principalName, Role:roleDefinitionName, Type:principalType}" \
  --output table
```

---

## Step 16 – Cleanup

```bash
# Remove role assignments
az role assignment delete \
  --assignee "$SP_OBJECT_ID" \
  --resource-group "$RG_NAME"

az role assignment delete \
  --assignee "$SP_OBJECT_ID" \
  --scope "$STORAGE_ID"

az role assignment delete \
  --assignee "$SP_MANUAL_OBJECT_ID" \
  --resource-group "$RG_NAME"

# Delete service principals
az ad sp delete --id "$SP_APP_ID"
az ad sp delete --id "$SP_CERT_APP_ID" 2>/dev/null || true
az ad sp delete --id "$APP_ID"

# Delete application registrations
az ad app delete --id "$APP_ID"

# Delete resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove certificate files
rm -f ${CERT_NAME}.* sp-test-script.sh

echo "✅ Cleanup complete"
```

---

## Summary

**What You Built:**
- Service principals with password authentication
- Service principals with certificate authentication
- RBAC role assignments for service principals
- Automated authentication scripts
- Credential rotation mechanisms

**Architecture:**
```
Application/Script → Service Principal (Client ID + Secret/Cert) → Azure AD → Azure Resources
                                                                      │
                                                                  RBAC Roles
```

**Key Components:**
- **Service Principal**: Identity for applications/services
- **Application Registration**: Azure AD app representation
- **Client Secret**: Password-based authentication
- **Client Certificate**: Certificate-based authentication
- **RBAC Roles**: Permissions assigned to service principals

**What You Learned:**
- Create service principals with different authentication methods
- Assign RBAC roles to service principals
- Authenticate programmatically using service principals
- Manage and rotate credentials securely
- Use service principals in automation scripts
- Separate application registration from service principal

---

## Best Practices

**Service Principal Management:**
- Use certificate authentication for production
- Implement credential rotation policies
- Store secrets in Azure Key Vault
- Apply least privilege RBAC assignments
- Document service principal purposes

**Security:**
- Never commit credentials to source control
- Use managed identities when possible (instead of SPs)
- Set credential expiration dates
- Monitor service principal usage
- Implement secret scanning in CI/CD pipelines

**Authentication:**
- Prefer certificate over password authentication
- Use strong, randomly generated secrets
- Implement conditional access for service principals
- Rotate credentials regularly (90 days recommended)
- Use different SPs for different environments

**Automation:**
- Store credentials in environment variables
- Use Azure Key Vault for secret management
- Implement proper error handling
- Log authentication events
- Use Azure Pipelines service connections

---

## Production Enhancements

**1. Store Secrets in Key Vault**
```bash
# Create Key Vault
az keyvault create \
  --name "kv-sp-secrets" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION"

# Store service principal credentials
az keyvault secret set \
  --vault-name "kv-sp-secrets" \
  --name "sp-client-id" \
  --value "$SP_APP_ID"

az keyvault secret set \
  --vault-name "kv-sp-secrets" \
  --name "sp-client-secret" \
  --value "$SP_PASSWORD"

# Retrieve from Key Vault in scripts
SP_ID=$(az keyvault secret show --vault-name "kv-sp-secrets" --name "sp-client-id" --query value -o tsv)
SP_SECRET=$(az keyvault secret show --vault-name "kv-sp-secrets" --name "sp-client-secret" --query value -o tsv)
```

**2. Implement Federated Credentials (Workload Identity)**
```bash
# Add federated credential for GitHub Actions
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-federated",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:org/repo:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

**3. Configure Application Permissions (Graph API)**
```bash
# Grant Microsoft Graph API permissions
az ad app permission add \
  --id "$APP_ID" \
  --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope

# Admin consent required
az ad app permission admin-consent --id "$APP_ID"
```

**4. Monitor Service Principal Activity**
```bash
# Query sign-in logs for service principal
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/auditLogs/signIns?\$filter=servicePrincipalId eq '$SP_OBJECT_ID'&\$top=50" \
  --query "value[].{Time:createdDateTime, App:appDisplayName, IP:ipAddress, Status:status.errorCode}" \
  --output table
```

---

## Troubleshooting

**Service principal creation fails:**
- Verify Application Administrator role
- Check naming conflicts (must be unique)
- Ensure sufficient permissions in subscription
- Review Azure AD quota limits
- Check certificate format if using cert authentication

**Authentication failures:**
- Verify client ID, secret, and tenant ID are correct
- Check credentials haven't expired
- Ensure service principal exists in correct tenant
- Wait for credential propagation (up to 5 minutes)
- Review sign-in logs for error details

**Permission denied errors:**
- Verify RBAC role assignments are active
- Check scope of role assignment
- Ensure service principal has required permissions
- Allow time for role propagation
- Review deny assignments

**Certificate authentication issues:**
- Verify certificate format (PEM)
- Check certificate expiration date
- Ensure certificate is properly encoded
- Confirm certificate is uploaded to app registration
- Test certificate validity with OpenSSL

**Credential rotation problems:**
- Update all applications using the credentials
- Use gradual rollover (add new, remove old)
- Test new credentials before removing old ones
- Document rotation procedures
- Implement automated rotation where possible

---

## Additional Resources

- [Service Principals Documentation](https://docs.microsoft.com/azure/active-directory/develop/app-objects-and-service-principals)
- [Azure AD Application Registration](https://docs.microsoft.com/azure/active-directory/develop/quickstart-register-app)
- [Certificate Credentials](https://docs.microsoft.com/azure/active-directory/develop/howto-create-service-principal-portal#option-1-upload-a-certificate)
- [Managed Identities vs Service Principals](https://docs.microsoft.com/azure/active-directory/managed-identities-azure-resources/managed-identities-faq)
- [Azure CLI Service Principal Commands](https://docs.microsoft.com/cli/azure/ad/sp)
