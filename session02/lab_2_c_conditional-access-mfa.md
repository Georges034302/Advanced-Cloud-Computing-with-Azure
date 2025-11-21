# Lab 2.C: Conditional Access & MFA

## Overview
This lab demonstrates implementing Conditional Access policies and Multi-Factor Authentication (MFA) in Azure AD. You'll configure location-based access, device compliance requirements, and enforce MFA for specific users and applications.

---

## Objectives
- Enable Azure AD Multi-Factor Authentication
- Create Conditional Access policies
- Configure location-based access controls
- Test MFA enrollment and authentication
- Implement risk-based policies
- Monitor sign-in logs and audit events

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure AD Premium P1 or P2 license (required for Conditional Access)
- Global Administrator or Conditional Access Administrator role
- Azure AD tenant with test users
- Mobile device for MFA testing

---

## Step 1 – Set Variables

```bash
# Set Azure AD configuration
TENANT_DOMAIN=$(az ad signed-in-user show --query 'userPrincipalName' -o tsv | cut -d'@' -f2)
TEST_USER_UPN="mfatest@$TENANT_DOMAIN"
CA_POLICY_NAME="CA-Require-MFA-All-Users"

# Display configuration
echo "TENANT_DOMAIN=$TENANT_DOMAIN"
echo "TEST_USER_UPN=$TEST_USER_UPN"
echo "CA_POLICY_NAME=$CA_POLICY_NAME"
```

---

## Step 2 – Check Azure AD License

```bash
# Verify Azure AD Premium license is available
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/subscribedSkus" \
  --query "value[?contains(skuPartNumber, 'AAD_PREMIUM')].{SKU:skuPartNumber, Enabled:prepaidUnits.enabled, Consumed:consumedUnits}" \
  --output table

# Note: Conditional Access requires Azure AD Premium P1 or higher
echo "⚠️  If no Premium license shown, CA policies cannot be created"
```

---

## Step 3 – Create Test User for MFA

```bash
# Create test user
TEST_PASSWORD="P@ssw0rd$(date +%s)"

az ad user create \
  --display-name "MFA Test User" \
  --user-principal-name "$TEST_USER_UPN" \
  --password "$TEST_PASSWORD" \
  --force-change-password-next-sign-in false

# Get user object ID
TEST_USER_ID=$(az ad user show \
  --id "$TEST_USER_UPN" \
  --query id \
  --output tsv)

echo "Test User ID: $TEST_USER_ID"
echo "Test User Password: $TEST_PASSWORD"
echo "⚠️  Save these credentials for testing"
```

---

## Step 4 – Enable MFA Per-User (Legacy Method)

```bash
# Note: Per-user MFA is managed via Microsoft Graph API
# Enable MFA for specific user
az rest --method PATCH \
  --url "https://graph.microsoft.com/v1.0/users/$TEST_USER_ID" \
  --headers "Content-Type=application/json" \
  --body '{
    "strongAuthenticationRequirements": [
      {
        "state": "Enabled"
      }
    ]
  }'

# Note: Modern approach uses Conditional Access instead
echo "✅ Per-user MFA enabled (legacy method)"
echo "👉 Conditional Access policies recommended for production"
```

---

## Step 5 – Create Conditional Access Policy (MFA for All Users)

```bash
# Note: Conditional Access policies are best created via Azure Portal
# CLI/API support is limited, here's the REST API approach

# Create CA policy requiring MFA for all users
cat > ca-policy.json << EOF
{
  "displayName": "$CA_POLICY_NAME",
  "state": "enabledForReportingButNotEnforced",
  "conditions": {
    "users": {
      "includeUsers": ["All"],
      "excludeUsers": [],
      "includeGroups": [],
      "excludeGroups": []
    },
    "applications": {
      "includeApplications": ["All"]
    },
    "locations": {
      "includeLocations": ["All"],
      "excludeLocations": ["AllTrusted"]
    }
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["mfa"]
  }
}
EOF

# Create policy via Microsoft Graph API
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --headers "Content-Type=application/json" \
  --body @ca-policy.json

echo "✅ Conditional Access policy created (report-only mode)"
echo "⚠️  Policy is in report-only mode - enable via Azure Portal after testing"
```

---

## Step 6 – List Conditional Access Policies

```bash
# List all Conditional Access policies
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --query "value[].{Name:displayName, State:state, ID:id}" \
  --output table
```

---

## Step 7 – Create Named Location

```bash
# Create named location for trusted IPs
cat > named-location.json << EOF
{
  "@odata.type": "#microsoft.graph.ipNamedLocation",
  "displayName": "Trusted Office IPs",
  "isTrusted": true,
  "ipRanges": [
    {
      "@odata.type": "#microsoft.graph.iPv4CidrRange",
      "cidrAddress": "203.0.113.0/24"
    }
  ]
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/namedLocations" \
  --headers "Content-Type=application/json" \
  --body @named-location.json

echo "✅ Named location created"
```

---

## Step 8 – Create CA Policy for Specific Group

```bash
# Create security group for MFA testing
az ad group create \
  --display-name "MFA-Required-Users" \
  --mail-nickname "mfarequired" \
  --description "Users requiring MFA"

MFA_GROUP_ID=$(az ad group show \
  --group "MFA-Required-Users" \
  --query id \
  --output tsv)

# Add test user to group
az ad group member add \
  --group "MFA-Required-Users" \
  --member-id "$TEST_USER_ID"

# Create CA policy for group
cat > ca-policy-group.json << EOF
{
  "displayName": "CA-Require-MFA-Specific-Group",
  "state": "enabledForReportingButNotEnforced",
  "conditions": {
    "users": {
      "includeGroups": ["$MFA_GROUP_ID"]
    },
    "applications": {
      "includeApplications": ["All"]
    }
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["mfa"]
  }
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --headers "Content-Type=application/json" \
  --body @ca-policy-group.json

echo "✅ Group-based CA policy created"
```

---

## Step 9 – Create Device Compliance Policy

```bash
# Create CA policy requiring compliant devices
cat > ca-policy-device.json << EOF
{
  "displayName": "CA-Require-Compliant-Device",
  "state": "enabledForReportingButNotEnforced",
  "conditions": {
    "users": {
      "includeUsers": ["All"]
    },
    "applications": {
      "includeApplications": ["Office365"]
    },
    "platforms": {
      "includePlatforms": ["android", "iOS", "windows", "macOS"]
    }
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["compliantDevice", "domainJoinedDevice"]
  }
}
EOF

az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --headers "Content-Type=application/json" \
  --body @ca-policy-device.json

echo "✅ Device compliance CA policy created"
```

---

## Step 10 – Configure MFA Settings

```bash
# Get current MFA settings (via Graph API)
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy" \
  --query "{ID:id, DisplayName:displayName}" \
  --output table

# List authentication methods
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy/authenticationMethodConfigurations" \
  --query "value[].{ID:id, State:state}" \
  --output table
```

---

## Step 11 – Test MFA Enrollment

```bash
echo "=== MFA Testing Instructions ==="
echo ""
echo "1. Open browser in private/incognito mode"
echo "2. Navigate to: https://aka.ms/mfasetup"
echo "3. Sign in with:"
echo "   Username: $TEST_USER_UPN"
echo "   Password: $TEST_PASSWORD"
echo "4. Follow MFA enrollment wizard"
echo "5. Configure Microsoft Authenticator app or SMS"
echo ""
echo "After enrollment, sign in to test MFA:"
echo "https://portal.azure.com"
echo ""
read -p "Press Enter after completing MFA enrollment..."
```

---

## Step 12 – View Sign-In Logs

```bash
# Get recent sign-in logs for test user
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/auditLogs/signIns?\$filter=userPrincipalName eq '$TEST_USER_UPN'&\$top=10" \
  --query "value[].{Time:createdDateTime, User:userPrincipalName, App:appDisplayName, Status:status.errorCode, MFA:authenticationRequirement}" \
  --output table

# Get sign-ins requiring MFA
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/auditLogs/signIns?\$filter=authenticationRequirement eq 'multiFactorAuthentication'&\$top=10" \
  --query "value[].{Time:createdDateTime, User:userPrincipalName, Location:location.city, Status:status.errorCode}" \
  --output table
```

---

## Step 13 – View Conditional Access Policy Report

```bash
# Get CA policy details
POLICY_ID=$(az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --query "value[?displayName=='$CA_POLICY_NAME'].id" \
  --output tsv | head -1)

if [ -n "$POLICY_ID" ]; then
  az rest --method GET \
    --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$POLICY_ID" \
    --output json | python3 -m json.tool
else
  echo "Policy not found"
fi
```

---

## Step 14 – Enable Policy (Change from Report-Only)

```bash
# Update policy state to enabled
if [ -n "$POLICY_ID" ]; then
  az rest --method PATCH \
    --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$POLICY_ID" \
    --headers "Content-Type=application/json" \
    --body '{"state": "enabled"}'
  
  echo "✅ Conditional Access policy enabled"
  echo "⚠️  MFA now required for all users"
else
  echo "Cannot enable - policy ID not found"
fi
```

---

## Step 15 – Cleanup

```bash
# Disable and delete Conditional Access policies
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --query "value[?contains(displayName, 'CA-')].id" \
  --output tsv | while read POLICY_ID; do
  echo "Deleting policy: $POLICY_ID"
  az rest --method DELETE \
    --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$POLICY_ID"
done

# Delete named locations
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/namedLocations" \
  --query "value[].id" \
  --output tsv | while read LOC_ID; do
  az rest --method DELETE \
    --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/namedLocations/$LOC_ID"
done

# Remove user from MFA group
az ad group member remove \
  --group "MFA-Required-Users" \
  --member-id "$TEST_USER_ID" 2>/dev/null || true

# Delete test group
az ad group delete \
  --group "MFA-Required-Users" 2>/dev/null || true

# Delete test user
az ad user delete \
  --id "$TEST_USER_UPN"

# Cleanup local files
rm -f ca-policy*.json named-location.json

echo "✅ Cleanup complete"
```

---

## Summary

**What You Built:**
- Multi-Factor Authentication configuration
- Conditional Access policies for security
- Named locations for trusted networks
- Group-based access controls
- Device compliance requirements

**Architecture:**
```
User Sign-In → Azure AD → Conditional Access Evaluation → MFA Challenge → Access Granted/Denied
                              │
                              ├─ Location Check
                              ├─ Device Compliance
                              ├─ Risk Assessment
                              └─ Application Sensitivity
```

**Key Components:**
- **Multi-Factor Authentication**: Additional verification beyond passwords
- **Conditional Access**: Policy-based access control
- **Named Locations**: Define trusted network locations
- **Device Compliance**: Ensure device meets security requirements
- **Sign-In Logs**: Audit authentication events

**What You Learned:**
- Enable and configure Azure AD MFA
- Create Conditional Access policies
- Implement location-based access controls
- Configure device compliance requirements
- Monitor sign-in and audit logs
- Test MFA enrollment and authentication

---

## Best Practices

**MFA Configuration:**
- Enable MFA for all users (especially admins)
- Use Conditional Access instead of per-user MFA
- Support multiple authentication methods
- Implement passwordless authentication where possible
- Configure emergency access accounts

**Conditional Access Policies:**
- Start with report-only mode
- Test policies thoroughly before enabling
- Exclude emergency access accounts
- Use named locations for trusted networks
- Implement defense-in-depth with multiple policies

**Policy Design:**
- Apply least privilege principles
- Use groups for policy assignments
- Document policy purposes and owners
- Review and update policies regularly
- Monitor policy effectiveness with reports

**User Experience:**
- Communicate changes to users in advance
- Provide clear MFA enrollment instructions
- Offer multiple authentication methods
- Implement remember MFA on trusted devices
- Create helpdesk documentation

---

## Production Enhancements

**1. Implement Risk-Based Policies**
```json
{
  "displayName": "CA-Block-High-Risk-Sign-Ins",
  "conditions": {
    "users": {"includeUsers": ["All"]},
    "signInRiskLevels": ["high"],
    "applications": {"includeApplications": ["All"]}
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["block"]
  }
}
```

**2. Require Approved Client Apps**
```json
{
  "displayName": "CA-Require-Approved-Apps-Mobile",
  "conditions": {
    "users": {"includeUsers": ["All"]},
    "applications": {"includeApplications": ["Office365"]},
    "platforms": {"includePlatforms": ["android", "iOS"]}
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["approvedApplication"]
  }
}
```

**3. Implement Session Controls**
```json
{
  "sessionControls": {
    "signInFrequency": {
      "value": 1,
      "type": "hours",
      "isEnabled": true
    },
    "persistentBrowser": {
      "mode": "never",
      "isEnabled": true
    }
  }
}
```

**4. Configure Terms of Use**
```bash
# Create Terms of Use agreement
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/agreements" \
  --body '{
    "displayName": "Company Terms of Use",
    "isViewingBeforeAcceptanceRequired": true,
    "files": [{"fileName": "TOU.pdf", "language": "en"}]
  }'
```

---

## Troubleshooting

**Conditional Access not working:**
- Verify Azure AD Premium P1/P2 license
- Check policy is enabled (not report-only)
- Confirm user/group is included in policy
- Review exclusions in policy
- Allow time for policy propagation (up to 1 hour)

**MFA enrollment fails:**
- Verify user has permission to register MFA
- Check authentication methods are enabled
- Confirm phone number format is correct
- Try alternative authentication method
- Review Azure AD MFA settings

**Users locked out:**
- Use emergency access account
- Temporarily disable problematic CA policy
- Check sign-in logs for error details
- Verify device compliance status
- Review location detection accuracy

**Graph API errors:**
- Verify sufficient API permissions
- Check authentication token is valid
- Confirm policy JSON format is correct
- Review API version compatibility
- Check Microsoft Graph documentation for changes

**Report-only mode not showing data:**
- Wait 24-48 hours for data collection
- Check sign-in logs in Azure Portal
- Verify users are signing in
- Review policy conditions match users
- Ensure Azure AD Premium license is active

---

## Additional Resources

- [Conditional Access Documentation](https://docs.microsoft.com/azure/active-directory/conditional-access/)
- [Azure AD MFA Overview](https://docs.microsoft.com/azure/active-directory/authentication/concept-mfa-howitworks)
- [Conditional Access Policies](https://docs.microsoft.com/azure/active-directory/conditional-access/concept-conditional-access-policies)
- [Microsoft Graph API - Conditional Access](https://docs.microsoft.com/graph/api/resources/conditionalaccesspolicy)
- [Common Conditional Access Policies](https://docs.microsoft.com/azure/active-directory/conditional-access/concept-conditional-access-policy-common)
