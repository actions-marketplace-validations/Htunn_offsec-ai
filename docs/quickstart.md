# offsec-ai — Quick Start Guide

## Installation

```bash
pip install offsec-ai
```

## Basic Usage

### Command Line

```bash
# Infrastructure scanning
offsec-ai scan example.com
offsec-ai scan example.com --ports 80,443,8080
offsec-ai l7-check example.com
offsec-ai full-scan example.com --output results.json
offsec-ai cert-check example.com
offsec-ai mtls-check example.com

# AI / LLM OWASP Top 10 black-box scanner
offsec-ai ai-owasp-scan https://api.example.com/v1/chat/completions
offsec-ai ai-owasp-scan https://api.example.com/v1/chat/completions \
    --api-key "$MY_KEY" --llm-judge

# MCP endpoint security scanner (passive)
offsec-ai mcp-scan https://mcp.example.com/mcp
offsec-ai mcp-scan https://mcp.example.com/mcp --output report.json

# MCP attacker (requires --authorized flag)
offsec-ai mcp-attack https://mcp.example.com/mcp --authorized \
    --auth-token "$TOKEN"
```

### Python API

```python
import asyncio
from offsec_ai import (
    PortChecker, L7Detector, CertificateAnalyzer, MTLSChecker,
    LLMOwaspScanner, MCPScanner, MCPAttacker, LLMJudge,
)

async def main():
    # Infrastructure: port scanning
    scanner = PortChecker()
    result = await scanner.scan_host("example.com", [80, 443, 22])
    print(f"Open ports: {[p.port for p in result.open_ports]}")

    # L7 / WAF detection
    detector = L7Detector()
    l7 = await detector.detect("example.com")
    if l7.is_protected:
        print(f"WAF/CDN: {l7.primary_protection.service.value}")

    # AI / LLM OWASP Top 10 scanner
    llm_scanner = LLMOwaspScanner(
        endpoint="https://api.example.com/v1/chat/completions",
        api_key="sk-...",
    )
    report = await llm_scanner.scan()
    print(f"LLM findings: {len(report.findings)}")

    # MCP security scan (passive)
    mcp = MCPScanner("https://mcp.example.com/mcp")
    mcp_result = await mcp.scan()
    print(f"MCP tools exposed: {len(mcp_result.tools)}")
    print(f"Auth required: {mcp_result.auth_posture.requires_auth}")

asyncio.run(main())
```

## Configuration

Create `~/.offsec-ai.yaml`:

```yaml
default_ports: [80, 443, 8080, 8443, 22, 21, 25, 53]
timeout: 5
concurrent_limit: 50
user_agent: "offsec-ai/2.0.0"
```

## Advanced Features

### Batch Scanning

```python
hosts = ["example.com", "google.com", "github.com"]
results = await scanner.scan_multiple_hosts(hosts, [80, 443])

for result in results:
    print(f"{result.host}: {len(result.open_ports)} open ports")
```

### Service Detection

```python
service_info = await scanner.check_service_version("example.com", 80)
print(f"Server: {service_info['version']}")
```

### WAF Testing

```python
waf_results = await detector.test_waf_bypass("example.com")
print(f"WAF detected: {waf_results['waf_detected']}")
```

## Output Formats

### JSON Output

```bash
offsec-ai scan example.com --output scan_results.json
```

### Programmatic Access

```python
# Save to file
result.save_to_file("results.json")

# Convert to dict
data = result.to_dict()

# Get JSON string
json_str = result.to_json(indent=2)
```

## Security Considerations

- Only scan systems you own or have explicit permission to test
- Be mindful of rate limiting and network policies
- Some scans may trigger security alerts
- Use responsibly for legitimate security testing only

## Troubleshooting

### Common Issues

1. **DNS Resolution Errors**
   ```bash
   # Check if hostname resolves
   nslookup example.com
   ```

2. **Connection Timeouts**
   ```bash
   # Increase timeout
   offsec-ai scan example.com --timeout 10
   ```

3. **Rate Limiting**
   ```bash
   # Reduce concurrency
   offsec-ai scan example.com --concurrent 10
   ```

### Debug Mode

```bash
offsec-ai scan example.com --verbose
```

## Examples

See the `examples/` directory for complete usage examples:
- `usage_examples.py` - Comprehensive examples
- `batch_scanning.py` - Batch scanning multiple targets
- `custom_config.py` - Custom configuration examples
