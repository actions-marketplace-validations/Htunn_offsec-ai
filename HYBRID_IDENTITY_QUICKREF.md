# Hybrid Identity Check - Quick Reference

## Command
```bash
port-checker hybrid-identity [DOMAINS...] [OPTIONS]
```

## Quick Examples
```bash
# Single domain
port-checker hybrid-identity example.com

# Multiple domains
port-checker hybrid-identity domain1.com domain2.com domain3.com

# From file
port-checker hybrid-identity $(cat domains.txt)

# With output
port-checker hybrid-identity example.com -o results.json

# Verbose mode
port-checker hybrid-identity example.com -v

# Custom settings
port-checker hybrid-identity example.com -t 15 -c 5
```

## What It Checks

| Check | Description |
|-------|-------------|
| 🎯 **Azure AD Flow** | Queries Azure AD realm API (most reliable) |
| 🔒 **ADFS Endpoints** | Checks `/adfs/ls` and related paths |
| 📜 **Federation Metadata** | WS-Federation metadata endpoints |
| ☁️ **Azure AD Integration** | Redirects to Microsoft login |
| 🔑 **OpenID Config** | `.well-known/openid-configuration` |
| 🌐 **DNS Records** | Microsoft verification, MX, subdomains |

## Key Innovation
**Uses same method as Azure Portal to discover ADFS endpoints!**

When you login to Azure Portal with `user@domain.com`:
1. Portal checks: `login.microsoftonline.com/common/userrealm/user@domain.com`
2. Gets ADFS endpoint from Azure AD configuration
3. Redirects to that ADFS endpoint

This tool does the **exact same thing** ✨

## Options
| Option | Description | Default |
|--------|-------------|---------|
| `-t, --timeout` | Request timeout (seconds) | 10 |
| `-o, --output` | Save results to JSON file | - |
| `-v, --verbose` | Show detailed information | False |
| `-c, --concurrent` | Max concurrent checks | 10 |

## Output Indicators
- ✅ **Hybrid Identity Detected** - Domain has federation configured
- 🔒 **ADFS Endpoint Found** - ADFS URL discovered
- ☁️ **Azure AD Integration** - Uses Microsoft authentication
- 📜 **Federation Metadata** - WS-Federation configured
- 🔑 **OpenID Config** - OpenID Connect available

## Use Cases
- 🔍 **Security Audit** - Identify federation services
- 📋 **Migration Planning** - Assess current identity setup
- 🛠️ **Troubleshooting** - Verify ADFS accessibility
- 🎯 **Reconnaissance** - Discover authentication mechanisms (ethical only!)

## Common Results

### Hybrid Identity Found
```
Status: ✅ Hybrid Identity Detected
ADFS Endpoint: ✅ Found
  Endpoint URL: https://adfs.company.com/adfs/ls
  Status Code: 200
```

### Cloud-Only (No Hybrid)
```
Status: ⚠️  No Hybrid Identity Found
ADFS Endpoint: ❌ Not Found
Federation Metadata: ❌ Not Found
```

### Error
```
Status: ❌ Error: Connection timeout
```

## Tips
- Use `-v` for DNS details and full analysis
- Use `-o` to save results for later analysis
- Increase `-t` if getting timeouts
- Reduce `-c` if overwhelming targets
- Check multiple related domains together

## Related Commands
```bash
# Full domain analysis workflow
port-checker hybrid-identity example.com -v
port-checker dns-trace example.com
port-checker l7-check example.com
port-checker cert-check adfs.example.com
```

## Documentation
- Full docs: `docs/hybrid-identity.md`
- Implementation: `IMPLEMENTATION_HYBRID_IDENTITY.md`
- README: `README.md`
