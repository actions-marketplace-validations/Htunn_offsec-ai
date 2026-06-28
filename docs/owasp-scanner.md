# offsec-ai — OWASP Security Scanners

offsec-ai ships **two** complementary OWASP scanners:

| Scanner | Command | Target |
|---------|---------|--------|
| Web OWASP Top 10 2021 | `offsec-ai owasp-scan` | HTTP/HTTPS web hosts |
| AI/LLM OWASP Top 10 2025 | `offsec-ai ai-owasp-scan` | LLM API endpoints (OpenAI-compatible) |

---

## AI / LLM OWASP Top 10 Scanner (LLM01–LLM10)

Black-box scanner for LLM inference endpoints, aligned with the
[OWASP Top 10 for Large Language Model Applications 2025](https://owasp.org/www-project-top-10-for-large-language-model-applications/).

### Quick Start

```bash
# Scan a public OpenAI-compatible endpoint
offsec-ai ai-owasp-scan https://api.example.com/v1/chat/completions

# With API key + LLM judge (uses OPENAI_API_KEY / ANTHROPIC_API_KEY / GEMINI_API_KEY)
offsec-ai ai-owasp-scan https://api.example.com/v1/chat/completions \
    --api-key "$MY_KEY" --llm-judge

# Scan specific categories only
offsec-ai ai-owasp-scan https://api.example.com/v1/chat/completions \
    --categories LLM01,LLM06,LLM07

# Export to JSON
offsec-ai ai-owasp-scan https://api.example.com/v1/chat/completions \
    --output ai-report.json
```

### LLM OWASP Categories

| ID | Category | Description |
|----|----------|-------------|
| LLM01 | Prompt Injection | Direct/indirect prompt injection bypassing guardrails |
| LLM02 | Insecure Output Handling | Unvalidated LLM output passed downstream |
| LLM03 | Training Data Poisoning | Data integrity and supply chain issues |
| LLM04 | Model Denial of Service | Resource exhaustion via adversarial inputs |
| LLM05 | Supply Chain Vulnerabilities | Third-party model/plugin risks |
| LLM06 | Sensitive Info Disclosure | PII, credentials, or system info leakage |
| LLM07 | Insecure Plugin Design | Insufficient plugin/tool sandboxing |
| LLM08 | Excessive Agency | Over-permissioned autonomous actions |
| LLM09 | Overreliance | Hallucinations and factual inaccuracies |
| LLM10 | Model Theft | Extraction of proprietary model behavior |

### LLM Judge

An optional second LLM evaluates each finding for a higher-confidence verdict:

```bash
# Auto-detects from env: OPENAI_API_KEY > ANTHROPIC_API_KEY > GEMINI_API_KEY
export GEMINI_API_KEY="your-key"
offsec-ai ai-owasp-scan https://api.example.com/v1/chat/completions --llm-judge
```

Gemini requires **no extra package** (stdlib `urllib` only).
OpenAI/Anthropic require `pip install offsec-ai[ai]`.

### Ethics & Authorization

> **Only scan LLM endpoints you own or have explicit written authorization to test.**
> AI OWASP scanning sends adversarial prompts that may trigger security alerts, consume
> API credits, or violate Terms of Service if used without authorization.

---

## OWASP Top 10 2021 Security Scanner

Comprehensive OWASP Top 10 2021 vulnerability scanning with automated detection, remediation guidance, and multi-format reporting.

### Quick Start

```bash
# Basic security scan (safe-mode with passive checks)
offsec-ai owasp-scan example.com

# Deep scan with active probing
offsec-ai owasp-scan example.com --deep

# Generate PDF report
offsec-ai owasp-scan example.com -f pdf -o security-report.pdf

# Scan with Nginx-specific remediation
offsec-ai owasp-scan example.com -t nginx --verbose

# Multiple targets with JSON export
offsec-ai owasp-scan site1.com site2.com -f json -o results.json
```

### Scan Modes

#### Safe Mode (Default)
- **Passive checks only** - No active probing or payload injection
- **Default categories**: A02, A05, A06, A07
- **Safe for production** environments
- Analyzes HTTP headers, TLS configuration, cookies, and banners

```bash
offsec-ai owasp-scan example.com
# or explicitly
offsec-ai owasp-scan example.com --safe-mode
```

#### Deep Mode
- **Active probing** enabled - Tests with payloads
- **All categories** scanned (A01-A10)
- **Higher detection accuracy**
- Includes HTTP method enumeration, path fuzzing, error detection

```bash
offsec-ai owasp-scan example.com --deep
```

### OWASP Top 10 2021 Categories

| ID | Category | Testable | Safe Mode | Deep Mode |
|----|----------|----------|-----------|-----------|
| A01 | Broken Access Control | ✅ | Limited | Full |
| A02 | Cryptographic Failures | ✅ | ✅ Full | ✅ Full |
| A03 | Injection | ✅ | ❌ | ✅ Limited |
| A04 | Insecure Design | ✅ | Limited | Limited |
| A05 | Security Misconfiguration | ✅ | ✅ Full | ✅ Full |
| A06 | Vulnerable & Outdated Components | ✅ | ✅ Full | ✅ Full |
| A07 | Authentication Failures | ✅ | ✅ Full | ✅ Full |
| A08 | Software/Data Integrity Failures | ✅ | Limited | Limited |
| A09 | Logging & Monitoring Failures | ❌ Not Testable | N/A | N/A |
| A10 | Server-Side Request Forgery (SSRF) | ✅ | ❌ | Limited |

**Note**: A09 (Logging & Monitoring) cannot be tested externally and will be marked as "Not Testable" in reports.

### What Gets Scanned

#### A02: Cryptographic Failures
- ✅ HSTS header presence and configuration
- ✅ TLS protocol versions (weak: TLS 1.0, 1.1)
- ✅ Cipher suite strength
- ✅ Certificate key size and algorithms
- ✅ Certificate expiration
- ✅ Cookie security flags (Secure, HttpOnly, SameSite)

#### A05: Security Misconfiguration  
- ✅ Content-Security-Policy (CSP)
- ✅ X-Content-Type-Options
- ✅ X-Frame-Options (clickjacking)
- ✅ Referrer-Policy
- ✅ Permissions-Policy
- ✅ Server version disclosure
- ✅ X-Powered-By disclosure
- ✅ CORS misconfiguration

#### A06: Vulnerable Components
- ✅ Server version detection
- ✅ Framework version disclosure
- ✅ Outdated software identification

#### A07: Authentication Failures
- ✅ Session cookie security
- ✅ Cookie flags analysis
- ✅ Session management indicators

### Output Formats

#### Console (Default)
Interactive terminal output with color-coded grades and findings.

```bash
offsec-ai owasp-scan example.com
offsec-ai owasp-scan example.com --verbose  # Detailed findings
offsec-ai owasp-scan example.com --quiet    # Grade summary only
```

#### JSON Export
Complete scan results with optional remediation details.

```bash
offsec-ai owasp-scan example.com -f json -o report.json
```

**JSON Structure:**
```json
{
  "target": "https://example.com",
  "scan_mode": "safe",
  "overall_grade": "B",
  "overall_score": 15,
  "categories": [
    {
      "category_id": "A02",
      "category_name": "Cryptographic Failures",
      "grade": "B",
      "findings": [...]
    }
  ]
}
```

#### CSV Export
Flat table format for spreadsheet analysis.

```bash
offsec-ai owasp-scan example.com -f csv -o findings.csv
```

**CSV Columns:**
- Category ID, Category Name, Severity, Title, Description, CWE ID, Score, Evidence

#### PDF Report
Professional security assessment report with remediation guidance.

```bash
offsec-ai owasp-scan example.com -f pdf -o security-report.pdf
```

**PDF Sections:**
1. Cover page with overall grade
2. Executive summary with severity breakdown
3. Category-by-category findings
4. Remediation steps with code examples
5. References and documentation links

### Technology-Specific Remediation

Use `--tech-stack` to get remediation code examples for your infrastructure:

```bash
# Apache web server
offsec-ai owasp-scan example.com -t apache -f pdf -o report.pdf

# Nginx
offsec-ai owasp-scan example.com -t nginx --verbose

# Microsoft IIS
offsec-ai owasp-scan example.com -t iis -f json -o results.json

# Cloudflare
offsec-ai owasp-scan example.com -t cloudflare

# Generic (default)
offsec-ai owasp-scan example.com -t generic
```

### Category Filtering

Scan specific OWASP categories only:

```bash
# Crypto and misconfiguration only
offsec-ai owasp-scan example.com -c A02,A05

# All authentication and access control issues
offsec-ai owasp-scan example.com -c A01,A07 --deep

# Single category deep dive
offsec-ai owasp-scan example.com -c A02 --verbose
```

### Severity Filtering

Filter findings by minimum severity:

```bash
# Critical findings only
offsec-ai owasp-scan example.com --severity CRITICAL

# High and critical
offsec-ai owasp-scan example.com --severity HIGH

# All findings (default)
offsec-ai owasp-scan example.com
```

### Grading System

#### Letter Grades (A-F)
- **A** (0-10 points): Excellent security posture
- **B** (11-25 points): Good security with minor issues
- **C** (26-50 points): Moderate security concerns
- **D** (51-100 points): Significant security issues
- **F** (>100 points or critical crypto failures): Failed security assessment

#### Severity Scoring
- **CRITICAL**: 15 points (automatic F grade for A02 crypto failures)
- **HIGH**: 10 points
- **MEDIUM**: 5 points
- **LOW**: 1 point

### Example Workflows

#### 1. Quick Security Check
```bash
# Fast passive scan for most common issues
offsec-ai owasp-scan example.com
```

#### 2. Compliance Report
```bash
# Full scan with PDF report for stakeholders
offsec-ai owasp-scan example.com \
  --deep \
  -f pdf \
  -o compliance-report.pdf \
  -t nginx
```

#### 3. CI/CD Integration
```bash
# JSON output for automated processing
offsec-ai owasp-scan staging.example.com \
  -f json \
  -o owasp-results.json \
  --severity HIGH
  
# Parse JSON in your CI/CD pipeline
# Fail build if critical findings exist
```

#### 4. Multi-Target Assessment
```bash
# Scan multiple domains
offsec-ai owasp-scan \
  app1.example.com \
  app2.example.com \
  api.example.com \
  -f csv \
  -o multi-target-scan.csv
```

#### 5. Focused Category Audit
```bash
# Deep dive into cryptographic security
offsec-ai owasp-scan example.com \
  -c A02 \
  --deep \
  --verbose \
  -f pdf \
  -o crypto-audit.pdf
```

### Programmatic Usage

```python
from offsec_ai import OwaspScanner
import asyncio

async def scan_security():
    # Initialize scanner
    scanner = OwaspScanner(
        mode="safe",  # or "deep"
        categories=["A02", "A05", "A06", "A07"],
        timeout=10.0,
    )
    
    # Scan target
    result = await scanner.scan("https://example.com")
    
    # Access results
    print(f"Overall Grade: {result.overall_grade}")
    print(f"Total Findings: {len(result.all_findings)}")
    print(f"Critical Findings: {len(result.critical_findings)}")
    
    # Export to PDF
    from offsec_ai.utils.exporters import OwaspPdfExporter
    exporter = OwaspPdfExporter(tech_stack="nginx")
    exporter.export(result, "security-report.pdf")
    
    # Or export to JSON
    from offsec_ai.utils.exporters import export_to_json
    export_to_json(result, "results.json", include_remediation=True)

# Run scan
asyncio.run(scan_security())
```

### Best Practices

1. **Start with Safe Mode**: Test passive scanning first before enabling --deep mode
2. **Use Tech-Stack**: Always specify your technology stack for relevant remediation
3. **Regular Scanning**: Integrate into CI/CD for continuous security monitoring
4. **Review Findings**: Not all findings may apply to your specific use case
5. **Remediate Prioritize**: Focus on CRITICAL and HIGH severity findings first
6. **Document Results**: Use PDF reports for audits and compliance documentation
7. **Test Remediations**: Re-scan after applying fixes to verify effectiveness

### Limitations

- **External Scanning Only**: Cannot detect internal vulnerabilities requiring application access
- **No Exploit Execution**: Detects potential vulnerabilities but doesn't exploit them
- **Context-Dependent**: Some findings may be false positives depending on architecture
- **A09 Not Testable**: Logging/monitoring cannot be assessed externally
- **Safe Mode Limited**: Passive checks miss vulnerabilities requiring active testing

### References

- [OWASP Top 10 2021](https://owasp.org/Top10/2021/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [Mozilla Web Security Guidelines](https://infosec.mozilla.org/guidelines/web_security)
- [SSL Labs](https://www.ssllabs.com/)

### Troubleshooting

#### "Cannot connect to target"
- Ensure target is accessible and responds to HTTPS
- Check firewall rules allow outbound connections
- Verify target URL format (include https:// if needed)

#### "No findings detected"
- This is good! Your security posture may be excellent
- Try --deep mode for more thorough scanning
- Verify target is actually a web application

#### "PDF generation failed"
- Ensure reportlab is installed: `pip install reportlab`
- Check output path has write permissions
- Verify sufficient disk space

#### "Timeout errors"
- Increase timeout: `--timeout 30`
- Check network latency to target
- Target may be rate-limiting requests
