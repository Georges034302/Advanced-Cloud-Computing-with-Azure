# Lab 2.A: Azure AD Users & RBAC

## Overview
This lab demonstrates Azure Active Directory (Azure AD) user and group management with Role-Based Access Control (RBAC). You'll create users, organize them into groups, assign Azure RBAC roles at different scopes, and verify permissions.

---

## Objectives
- Create Azure AD users with different attributes
- Create Azure AD groups and add members
- Assign RBAC roles at subscription, resource group, and resource scopes
- Test user permissions and access control
- Review role assignments and effective permissions
- Clean up users and groups

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with Global Administrator or User Administrator role
- Owner or User Access Administrator role for RBAC assignments
- Valid Azure AD tenant
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure region and naming
LOCATION="eastus"
RG_NAME="rg-lab2a-rbac"
STORAGE_ACCOUNT="lab2arbac$RANDOM"

# User and group names
USER1_UPN="developer1@$(az ad signed-in-user show --query 'userPrincipalName' -o tsv | cut -d'@' -f2)"
USER2_UPN="operator1@$(az ad signed-in-user show --query 'userPrincipalName' -o tsv | cut -d'@' -f2)"
GROUP_NAME="DevTeam"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "USER1_UPN=$USER1_UPN"
echo "USER2_UPN=$USER2_UPN"
echo "GROUP_NAME=$GROUP_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for testing RBAC
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Azure AD Users

```bash
# Create first user (Developer)
USER1_PASSWORD="P@ssw0rd$(date +%s)"

az ad user create \
  --display-name "Developer User 1" \
  --user-principal-name "$USER1_UPN" \
  --password "$USER1_PASSWORD" \
  --force-change-password-next-sign-in false

# Get user1 object ID
USER1_ID=$(az ad user show \
  --id "$USER1_UPN" \
  --query id \
  --output tsv)

echo "User 1 ID: $USER1_ID"
echo "User 1 Password: $USER1_PASSWORD"

# Create second user (Operator)
USER2_PASSWORD="P@ssw0rd$(date +%s)"

az ad user create \
  --display-name "Operator User 1" \
  --user-principal-name "$USER2_UPN" \
  --password "$USER2_PASSWORD" \
  --force-change-password-next-sign-in false

# Get user2 object ID
USER2_ID=$(az ad user show \
  --id "$USER2_UPN" \
  --query id \
  --output tsv)

echo "User 2 ID: $USER2_ID"
echo "User 2 Password: $USER2_PASSWORD"

# Verify users created
az ad user list \
  --filter "startswith(userPrincipalName,'developer') or startswith(userPrincipalName,'operator')" \
  --query "[].{Name:displayName, UPN:userPrincipalName}" \
  --output table
```

---

## Step 4 – Create Azure AD Group

```bash
# Create security group
az ad group create \
  --display-name "$GROUP_NAME" \
  --mail-nickname "devteam" \
  --description "Development Team Security Group"

# Get group object ID
GROUP_ID=$(az ad group show \
  --group "$GROUP_NAME" \
  --query id \
  --output tsv)

echo "Group ID: $GROUP_ID"

# Add users to group
az ad group member add \
  --group "$GROUP_NAME" \
  --member-id "$USER1_ID"

az ad group member add \
  --group "$GROUP_NAME" \
  --member-id "$USER2_ID"

# Verify group membership
az ad group member list \
  --group "$GROUP_NAME" \
  --query "[].{Name:displayName, UPN:userPrincipalName}" \
  --output table
```

---

## Step 5 – Create Test Resources

```bash
# Create storage account for RBAC testing
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

## Step 6 – Assign RBAC Role to User (Resource Group Scope)

```bash
# Assign Contributor role to User1 at resource group scope
az role assignment create \
  --assignee "$USER1_ID" \
  --role "Contributor" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME"

# Verify role assignment
az role assignment list \
  --assignee "$USER1_ID" \
  --resource-group "$RG_NAME" \
  --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" \
  --output table
```

---

## Step 7 – Assign RBAC Role to User (Resource Scope)

```bash
# Assign Storage Blob Data Reader to User2 at storage account scope
az role assignment create \
  --assignee "$USER2_ID" \
  --role "Storage Blob Data Reader" \
  --scope "$STORAGE_ID"

# Verify role assignment
az role assignment list \
  --assignee "$USER2_ID" \
  --scope "$STORAGE_ID" \
  --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" \
  --output table
```

---

## Step 8 – Assign RBAC Role to Group

```bash
# Assign Reader role to group at resource group scope
az role assignment create \
  --assignee "$GROUP_ID" \
  --role "Reader" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME"

# List all role assignments for the resource group
az role assignment list \
  --resource-group "$RG_NAME" \
  --query "[].{Principal:principalName, Role:roleDefinitionName, Type:principalType}" \
  --output table
```

---

## Step 9 – List Built-in RBAC Roles

```bash
# List common built-in roles
az role definition list \
  --query "[?contains(roleName, 'Contributor') || contains(roleName, 'Reader') || contains(roleName, 'Owner')].{Name:roleName, Description:description}" \
  --output table

# Get details of specific role
az role definition list \
  --name "Contributor" \
  --output json | head -50

# List all Storage-related roles
az role definition list \
  --query "[?contains(roleName, 'Storage')].roleName" \
  --output table
```

---

## Step 10 – Test User Permissions

```bash
# Get current user context
CURRENT_USER=$(az account show --query user.name --output tsv)
echo "Currently signed in as: $CURRENT_USER"

# Note: To fully test user permissions, you would need to:
# 1. Sign in as User1: az login -u $USER1_UPN -p $USER1_PASSWORD
# 2. Try creating resources in the resource group (should succeed - Contributor)
# 3. Try deleting the resource group (should fail - Contributor cannot delete RG)
# 4. Sign in as User2: az login -u $USER2_UPN -p $USER2_PASSWORD  
# 5. Try reading storage blobs (should succeed - Reader)
# 6. Try creating resources (should fail - Reader only)

echo "⚠️  User credentials for testing:"
echo "User1 (Contributor): $USER1_UPN / $USER1_PASSWORD"
echo "User2 (Storage Reader): $USER2_UPN / $USER2_PASSWORD"
```

---

## Step 11 – View Effective Permissions

```bash
# List all role assignments for User1
echo "=== User1 Role Assignments ==="
az role assignment list \
  --assignee "$USER1_ID" \
  --all \
  --query "[].{Role:roleDefinitionName, Scope:scope}" \
  --output table

# List all role assignments for User2 (direct + group)
echo "=== User2 Role Assignments ==="
az role assignment list \
  --assignee "$USER2_ID" \
  --all \
  --query "[].{Role:roleDefinitionName, Scope:scope}" \
  --output table

# List group role assignments
echo "=== Group Role Assignments ==="
az role assignment list \
  --assignee "$GROUP_ID" \
  --all \
  --query "[].{Role:roleDefinitionName, Scope:scope}" \
  --output table
```

---

## Step 12 – Create Custom RBAC Role (Optional)

```bash
# Create custom role definition JSON
cat > custom-role.json << EOF
{
  "Name": "Storage Account Key Reader",
  "Description": "Can read storage account keys",
  "Actions": [
    "Microsoft.Storage/storageAccounts/listkeys/action",
    "Microsoft.Storage/storageAccounts/read"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/$(az account show --query id -o tsv)"
  ]
}
EOF

# Create custom role
az role definition create \
  --role-definition custom-role.json

# Verify custom role created
az role definition list \
  --custom-role-only true \
  --query "[].{Name:roleName, Type:roleType}" \
  --output table
```

---

## Step 13 – Cleanup

```bash
# Remove role assignments
az role assignment delete \
  --assignee "$USER1_ID" \
  --resource-group "$RG_NAME"

az role assignment delete \
  --assignee "$USER2_ID" \
  --scope "$STORAGE_ID"

az role assignment delete \
  --assignee "$GROUP_ID" \
  --resource-group "$RG_NAME"

# Delete custom role (if created)
az role definition delete \
  --name "Storage Account Key Reader" 2>/dev/null || true

# Delete resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove users from group
az ad group member remove \
  --group "$GROUP_NAME" \
  --member-id "$USER1_ID"

az ad group member remove \
  --group "$GROUP_NAME" \
  --member-id "$USER2_ID"

# Delete Azure AD group
az ad group delete \
  --group "$GROUP_NAME"

# Delete Azure AD users
az ad user delete \
  --id "$USER1_UPN"

az ad user delete \
  --id "$USER2_UPN"

echo "✅ Cleanup complete"
```

---

## Summary

**What You Built:**
- Azure AD users with unique credentials
- Azure AD security group with members
- RBAC role assignments at multiple scopes
- Custom RBAC role definition
- Test environment for permission validation

**Architecture:**
```
Azure AD Tenant
├── Users (User1, User2)
├── Groups (DevTeam)
└── RBAC Assignments
    ├── Resource Group Scope → User1 (Contributor)
    ├── Resource Scope → User2 (Storage Reader)
    └── Resource Group Scope → DevTeam (Reader)
```

**Key Components:**
- **Azure AD Users**: Identity principals for authentication
- **Azure AD Groups**: Organize users for simplified management
- **RBAC Roles**: Define permissions (built-in or custom)
- **Role Assignments**: Link principals to roles at specific scopes
- **Scopes**: Subscription, Resource Group, or Resource level

**What You Learned:**
- Create and manage Azure AD users and groups
- Assign RBAC roles at different scope levels
- Understand permission inheritance
- Create custom RBAC roles
- Verify and test effective permissions
- Clean up identity and access resources

---

## Best Practices

**Identity Management:**
- Use groups instead of individual user assignments
- Implement least privilege principle
- Use managed identities for applications
- Enable MFA for all users
- Regularly audit user accounts and permissions

**RBAC Configuration:**
- Assign roles at appropriate scopes (least privilege)
- Use built-in roles when possible
- Document custom role definitions
- Review role assignments regularly
- Avoid using Owner role unless necessary

**Security:**
- Enforce strong password policies
- Enable Azure AD Privileged Identity Management (PIM)
- Use Conditional Access policies
- Monitor sign-in logs and audit logs
- Implement emergency access accounts

**Group Management:**
- Use descriptive group names
- Document group purposes
- Implement dynamic group membership when applicable
- Review group membership regularly
- Use nested groups carefully

---

## Production Enhancements

**1. Enable Privileged Identity Management (PIM)**
```bash
# Requires Azure AD Premium P2
# Configure eligible role assignments instead of permanent
az rest --method PUT \
  --url "https://graph.microsoft.com/v1.0/privilegedAccess/azureResources/roleAssignmentRequests" \
  --body '{"roleDefinitionId":"","resourceId":"","subjectId":"","assignmentState":"Eligible"}'
```

**2. Implement Dynamic Groups**
```bash
# Create dynamic group based on user attributes
az ad group create \
  --display-name "Dynamic-Developers" \
  --mail-nickname "dynamicdevs" \
  --group-types "DynamicMembership" \
  --membership-rule "user.department -eq 'Engineering'" \
  --membership-rule-processing-state "On"
```

**3. Enable Azure AD Audit Logging**
```bash
# Configure diagnostic settings for Azure AD
az monitor diagnostic-settings create \
  --name "aad-audit-logs" \
  --resource "/providers/Microsoft.AADIAM" \
  --logs '[{"category":"AuditLogs","enabled":true},{"category":"SignInLogs","enabled":true}]' \
  --workspace "/subscriptions/.../resourceGroups/.../providers/Microsoft.OperationalInsights/workspaces/..."
```

**4. Implement Access Reviews**
```bash
# Requires Azure AD Premium P2
# Create access review for group membership
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identityGovernance/accessReviews/definitions" \
  --body '{"displayName":"Review DevTeam Access","scope":{"@odata.type":"#microsoft.graph.accessReviewQueryScope"}}'
```

---

## Troubleshooting

**User creation fails:**
- Verify you have User Administrator role
- Check domain name is correct
- Ensure UPN doesn't already exist
- Verify password meets complexity requirements
- Check Azure AD tenant quotas

**Cannot assign RBAC roles:**
- Verify you have Owner or User Access Administrator role
- Check role assignment scope is correct
- Ensure principal ID is valid
- Allow time for role propagation (up to 5 minutes)
- Review subscription/resource group permissions

**Group membership not reflecting:**
- Wait for Azure AD synchronization (up to 15 minutes)
- Verify user was added successfully
- Check group type (security vs. Microsoft 365)
- Review dynamic membership rules if applicable
- Clear cached credentials and re-authenticate

**Permission denied errors:**
- Verify role assignments are active
- Check scope hierarchy (resource < RG < subscription)
- Review deny assignments
- Confirm user is authenticated correctly
- Check for management locks on resources

**Custom role issues:**
- Verify JSON syntax is correct
- Check assignable scopes are valid
- Ensure actions are properly formatted
- Review NotActions for conflicts
- Confirm role name is unique

---

## Additional Resources

- [Azure RBAC Documentation](https://docs.microsoft.com/azure/role-based-access-control/)
- [Azure AD User Management](https://docs.microsoft.com/azure/active-directory/fundamentals/add-users-azure-active-directory)
- [Built-in RBAC Roles](https://docs.microsoft.com/azure/role-based-access-control/built-in-roles)
- [Custom RBAC Roles](https://docs.microsoft.com/azure/role-based-access-control/custom-roles)
- [Azure AD Groups](https://docs.microsoft.com/azure/active-directory/fundamentals/active-directory-groups-create-azure-portal)
