# Implementation Summary: Hybrid Identity and ADFS Detection

## Overview
Successfully implemented hybrid identity and ADFS endpoint detection features for the Simple Port Checker CLI tool.

## What Was Implemented

### 1. New Core Module: `hybrid_identity_checker.py`
**Location:** `src/offsec_ai/core/hybrid_identity_checker.py`

**Key Features:**
- `HybridIdentityResult` class to store detection results
- `HybridIdentityChecker` class with comprehensive detection methods

**Detection Methods:**

#### Primary Method: Azure AD Login Flow Detection 🎯
- **Most reliable method** - uses the same approach as Azure Portal
- Queries `https://login.microsoftonline.com/common/userrealm/test@domain.com`
- Checks if domain is configured as "Federated" in Azure AD
- Extracts actual ADFS endpoint from `AuthURL` field
- Falls back to modern `GetCredentialType` API
- **Advantage:** Works even if ADFS is behind firewall (gets config from Azure AD)

#### Fallback Methods:
1. **Direct ADFS Endpoint Probing**
   - Checks `/adfs/ls` and related paths
   - Tests common subdomains: `adfs`, `sts`, `federation`, `fs`

2. **Federation Metadata Discovery**
   - `/FederationMetadata/2007-06/FederationMetadata.xml`
   - WS-Federation trust endpoints

3. **Azure AD Integration Detection**
   - Checks for redirects to `login.microsoftonline.com`
   - Analyzes response headers

4. **OpenID Connect Configuration**
   - `/.well-known/openid-configuration`
   - ADFS OpenID discovery endpoints

5. **DNS Record Analysis**
   - A, CNAME, TXT, MX records
   - Microsoft verification records
   - Microsoft 365 mail server detection
   - ADFS subdomain discovery

### 2. CLI Integration
**Location:** `src/offsec_ai/cli.py`

**New Command:** `offsec-ai hybrid-identity`

**Options:**
```bash
offsec-ai hybrid-identity TARGET [TARGET...] [OPTIONS]

Options:
  -t, --timeout INTEGER           Request timeout in seconds (default: 10)
  -o, --output TEXT               Output file (JSON format)
  -v, --verbose                   Enable verbose output with DNS details
  -c, --concurrent INTEGER        Maximum concurrent checks (default: 10)
```

**Features:**
- Progress bar for scanning multiple domains
- Detailed per-domain results in verbose mode
- Summary table with statistics
- JSON output support
- Concurrent checking with configurable limit

**Display Functions:**
- `_run_hybrid_identity_check()` - Main execution function with progress tracking
- `_display_hybrid_identity_result()` - Per-domain detailed results
- `_display_hybrid_identity_summary()` - Overall summary with statistics

### 3. Documentation

#### Main Documentation: `docs/hybrid-identity.md`
Comprehensive documentation including:
- Feature overview
- Detection method details
- Usage examples
- Output format specifications
- Use cases (security auditing, migration planning, troubleshooting)
- Technical details
- Security considerations
- Troubleshooting guide
- Future enhancements

#### README Updates: `README.md`
- Added to features list with 🔑 emoji
- Added to Quick Start examples
- New command documentation section
- Highlighted Azure AD login flow method

### 4. Test Script
**Location:** `test_hybrid_identity.py`

Simple test script to verify functionality with common domains.

## Key Innovation: Azure AD Login Flow Method

This implementation uses the **same technique as Azure Portal** to discover ADFS endpoints:

1. User tries to login at portal.azure.com with `user@domain.com`
2. Azure Portal queries user realm API to check domain configuration
3. If domain is federated, Azure AD returns the ADFS endpoint URL
4. Portal redirects user to that ADFS endpoint

**Why this is superior:**
- ✅ Gets the **actual configured** ADFS endpoint from Azure AD
- ✅ Works even if ADFS is not publicly accessible
- ✅ No need to guess subdomains or paths
- ✅ Same reliability as Microsoft's own portal
- ✅ Works for any domain configured in Azure AD

## Usage Examples

### Basic Check
```bash
offsec-ai hybrid-identity example.com
```

### Multiple Domains
```bash
offsec-ai hybrid-identity domain1.com domain2.com domain3.com
```

### Batch Processing
```bash
offsec-ai hybrid-identity $(cat domains.txt) --output results.json
```

### Verbose Mode
```bash
offsec-ai hybrid-identity example.com --verbose
```

## Output Example

### Console Output
```
🔍 Checking hybrid identity for 1 domain(s)
Timeout: 10s

🔐 Hybrid Identity Check - example.com
┏━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Property              ┃ Value                            ┃
┡━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ Status                │ ✅ Hybrid Identity Detected      │
│ ADFS Endpoint         │ ✅ Found                         │
│   Endpoint URL        │ https://adfs.example.com/adfs/ls │
│   Status Code         │ 200                              │
│ Federation Metadata   │ ✅ Found                         │
│ Azure AD Integration  │ ✅ Detected                      │
│ OpenID Configuration  │ ✅ Found                         │
│ DNS Records           │ A: 2 records                     │
│                       │ MS Verification ✓                │
│                       │ ADFS subdomains: adfs, sts       │
│ Response Time         │ 5.12s                            │
└───────────────────────┴──────────────────────────────────┘
```

### JSON Output
```json
{
  "scan_time": "2025-10-05T10:30:00",
  "total_time": 5.23,
  "total_targets": 1,
  "results": [
    {
      "fqdn": "example.com",
      "has_hybrid_identity": true,
      "has_adfs": true,
      "adfs_endpoint": "https://adfs.example.com/adfs/ls",
      "adfs_status_code": 200,
      "federation_metadata_found": true,
      "azure_ad_detected": true,
      "openid_config_found": true,
      "dns_records": {
        "A": ["192.0.2.1", "192.0.2.2"],
        "microsoft_verification": true,
        "adfs_subdomains": ["adfs", "sts"]
      },
      "error": null,
      "response_time": 5.12
    }
  ]
}
```

## Files Modified/Created

### New Files
1. `src/offsec_ai/core/hybrid_identity_checker.py` - Core detection logic (478 lines)
2. `docs/hybrid-identity.md` - Comprehensive documentation (400+ lines)
3. `test_hybrid_identity.py` - Test script

### Modified Files
1. `src/offsec_ai/cli.py` - Added command and display functions
2. `README.md` - Added feature, examples, and command documentation

## Technical Details

### Dependencies
No new dependencies required! Uses existing packages:
- `aiohttp` - For async HTTP requests
- `dns.resolver` - For DNS queries
- `click` - For CLI (already used)
- `rich` - For console output (already used)

### Async Implementation
- All checks are async for performance
- Supports batch checking with concurrent limits
- Uses connection pooling via aiohttp

### Error Handling
- Graceful handling of network errors
- Timeout protection
- Per-domain error reporting
- Continues scan even if individual domains fail

## Testing Recommendations

### Test with Known Domains
```bash
# Microsoft domains (will have hybrid identity)
offsec-ai hybrid-identity microsoft.com login.microsoftonline.com

# Regular domains (should not have hybrid identity)
offsec-ai hybrid-identity google.com amazon.com

# Your organization's domains
offsec-ai hybrid-identity yourcompany.com
```

### Test Error Handling
```bash
# Non-existent domain
offsec-ai hybrid-identity nonexistent-domain-12345.com

# Invalid domain format
offsec-ai hybrid-identity "not a domain"
```

### Test Batch Processing
```bash
# Create test file
echo "microsoft.com" > test-domains.txt
echo "google.com" >> test-domains.txt
echo "example.com" >> test-domains.txt

# Run batch check
offsec-ai hybrid-identity $(cat test-domains.txt) --output results.json --verbose
```

## Integration with Existing Features

Works alongside existing features:
- Can be combined with `l7-check` for complete domain analysis
- Complements `dns-trace` with hybrid identity context
- Works with `cert-check` for ADFS certificate analysis

## Future Enhancements

Possible improvements:
1. SAML endpoint detection
2. OAuth 2.0 flow analysis
3. Kerberos realm detection
4. Certificate validation for ADFS
5. Historical tracking of ADFS endpoint changes
6. Integration with Azure AD Graph API (with authentication)

## Security & Compliance

### What the Tool Does
- ✅ Passive reconnaissance only
- ✅ Standard HTTP/HTTPS requests
- ✅ Public DNS queries
- ✅ No authentication attempts

### What it Does NOT Do
- ❌ No exploitation attempts
- ❌ No brute force attacks
- ❌ No unauthorized access
- ❌ No credential testing

### Best Practices
- Only scan domains you own or have permission to test
- Be aware scans may be logged
- Use appropriate rate limiting
- Consider legal and compliance requirements

## Status
✅ **Implementation Complete**
✅ **Documentation Complete**
✅ **CLI Integration Complete**
✅ **Ready for Testing**

## Next Steps
1. Install dependencies if needed: `pip install -e .`
2. Test the new command: `offsec-ai hybrid-identity example.com`
3. Try with your organization's domains
4. Report any issues or enhancement requests
