# How Azure AD Discovers ADFS Endpoints

## The Problem
When a user tries to login to Azure Portal with their corporate email (e.g., `user@company.com`), how does Azure know where to redirect them for authentication?

## The Solution: User Realm API

Azure AD has a public API that returns authentication information for any domain:

```
GET https://login.microsoftonline.com/common/userrealm/user@domain.com?api-version=2.0
```

## Example Flow

### Scenario: User logs into Azure Portal

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User Journey                                  │
└─────────────────────────────────────────────────────────────────────┘

1. User navigates to portal.azure.com
   └─> Redirected to login.microsoftonline.com

2. User enters: john.doe@contoso.com
   
3. Azure AD checks domain configuration
   ┌──────────────────────────────────────────────────────────────┐
   │ GET /common/userrealm/john.doe@contoso.com?api-version=2.0  │
   └──────────────────────────────────────────────────────────────┘

4. Azure AD responds with domain info:
   ┌──────────────────────────────────────────────────────────────┐
   │ {                                                            │
   │   "NameSpaceType": "Federated",                            │
   │   "AuthURL": "https://adfs.contoso.com/adfs/ls",           │
   │   "FederationBrandName": "Contoso Corporation",             │
   │   "CloudInstanceName": "microsoftonline.com"                │
   │ }                                                            │
   └──────────────────────────────────────────────────────────────┘

5. Azure Portal redirects to ADFS endpoint
   └─> User lands on: https://adfs.contoso.com/adfs/ls

6. User authenticates with corporate credentials on ADFS

7. ADFS redirects back to Azure with authentication token

8. User is logged into Azure Portal
```

## Response Types

### Federated Domain (Has Hybrid Identity)
```json
{
  "NameSpaceType": "Federated",
  "AuthURL": "https://adfs.company.com/adfs/ls",
  "FederationBrandName": "Company Corp",
  "FederationGlobalVersion": -1,
  "CloudInstanceName": "microsoftonline.com"
}
```
**Meaning:** Domain uses on-premises ADFS for authentication

### Managed Domain (Cloud-Only)
```json
{
  "NameSpaceType": "Managed",
  "DomainName": "company.com",
  "CloudInstanceName": "microsoftonline.com",
  "State": 3
}
```
**Meaning:** Domain uses Azure AD for authentication (no ADFS)

### Unknown Domain
```json
{
  "NameSpaceType": "Unknown",
  "State": 1
}
```
**Meaning:** Domain not registered with Azure AD

## What Our Tool Does

```python
# Simplified version of what the tool does

async def discover_adfs_endpoint(domain):
    # Query Azure AD's user realm API
    url = f"https://login.microsoftonline.com/common/userrealm/test@{domain}"
    response = await http_get(url, params={"api-version": "2.0"})
    data = response.json()
    
    # Check if domain is federated
    if data.get("NameSpaceType") == "Federated":
        # Extract ADFS endpoint
        adfs_url = data.get("AuthURL")
        return {
            "has_adfs": True,
            "endpoint": adfs_url,
            "brand": data.get("FederationBrandName")
        }
    
    return {"has_adfs": False}
```

## Advantages of This Method

### ✅ Accurate
- Gets the **actual configured** ADFS endpoint from Azure AD
- No guessing of subdomains or paths
- Same data that Azure Portal uses

### ✅ Reliable
- Works even if ADFS is behind a firewall
- Doesn't require ADFS to be publicly accessible
- Information is stored in Azure AD configuration

### ✅ Complete
- Returns federation brand name
- Shows if domain is managed vs. federated
- Provides additional context

### ✅ Official
- Uses Microsoft's official API
- Same method as Azure Portal
- Well-documented behavior

## Alternative API: GetCredentialType

Azure AD also has a modern API for the new login experience:

```
POST https://login.microsoftonline.com/common/GetCredentialType
Content-Type: application/json

{
  "username": "user@domain.com",
  "isOtherIdpSupported": true,
  "checkPhones": false
}
```

Response for federated domain:
```json
{
  "IfExistsResult": 0,
  "Credentials": {
    "FederationRedirectUrl": "https://adfs.company.com/adfs/ls"
  }
}
```

Our tool checks **both APIs** for maximum compatibility!

## Real-World Example

### Test with Microsoft's Own Domain

```bash
offsec-ai hybrid-identity microsoft.com
```

Expected Result:
```
🔐 Hybrid Identity Check - microsoft.com
┏━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Property              ┃ Value                  ┃
┡━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│ Status                │ ✅ Hybrid Identity     │
│ ADFS Endpoint         │ ✅ Found               │
│   Endpoint URL        │ https://msft.sts...    │
└───────────────────────┴────────────────────────┘
```

### Test with Google (No Federation)

```bash
offsec-ai hybrid-identity google.com
```

Expected Result:
```
🔐 Hybrid Identity Check - google.com
┏━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Property              ┃ Value                  ┃
┡━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│ Status                │ ⚠️  No Hybrid Identity │
│ ADFS Endpoint         │ ❌ Not Found           │
└───────────────────────┴────────────────────────┘
```

## Why This Matters

### For Security Teams
- Discover federation infrastructure
- Audit identity configurations
- Identify shadow IT

### For IT Operations
- Troubleshoot login issues
- Verify ADFS connectivity
- Plan migrations

### For Penetration Testers
- Reconnaissance phase
- Identify authentication mechanisms
- Map attack surface (ethically!)

### For Developers
- Test authentication flows
- Validate configurations
- Debug integration issues

## Summary

**Traditional Method (Guessing):**
```
❌ Check adfs.company.com
❌ Check sts.company.com
❌ Check federation.company.com
❌ Try /adfs/ls
❌ Try /adfs/services/trust
❌ Maybe find it, maybe not
```

**Our Method (Azure AD API):**
```
✅ Ask Azure AD: "Where's the ADFS for company.com?"
✅ Get exact answer: "https://adfs.company.com/adfs/ls"
✅ Done! 🎉
```

## References

- [Azure AD User Realm API](https://login.microsoftonline.com/common/userrealm/)
- [Microsoft Identity Platform](https://learn.microsoft.com/en-us/azure/active-directory/develop/)
- [ADFS Documentation](https://learn.microsoft.com/en-us/windows-server/identity/ad-fs/)
- [Hybrid Identity Overview](https://learn.microsoft.com/en-us/azure/active-directory/hybrid/)
