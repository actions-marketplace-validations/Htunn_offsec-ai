# offsec-ai — OIDC / OAuth 2.0 / SAML Auth Protocol Security

`offsec-ai` ships a passive **Auth Scanner** and an authorization-gated **Auth Attacker** for
assessing identity-provider and authorization-server endpoints across the three major auth
protocols used in modern applications: **OIDC**, **OAuth 2.0**, and **SAML 2.0**.

All passive probes are read-only HTTP requests — no credentials required. Active probes
require the `--i-have-authorization` flag.

---

## Quick Start

```bash
# Auto-detect protocol and scan
offsec-ai auth-scan https://auth.example.com

# Explicitly scan SAML IdP metadata
offsec-ai auth-scan https://idp.example.com --protocol saml

# Public test IdP (mocksaml.com)
offsec-ai auth-scan https://mocksaml.com/api/saml/metadata --protocol saml

# Enable LLM judge (shows "LLM Judge: gemini" or "openai" / "anthropic" in output)
offsec-ai auth-scan https://accounts.google.com --llm-judge

# Custom headers, skip TLS verification
offsec-ai auth-scan https://internal-idp.corp.example.com \
  --header "Authorization: Bearer token" --no-tls-verify

# JSON output
offsec-ai auth-scan https://auth.example.com --format json --output auth-scan.json

# Active attack — safe mode (open redirect, state bypass, PKCE bypass)
offsec-ai auth-attack https://auth.example.com --i-have-authorization

# Deep mode (JWT alg=none, scope escalation, token replay, SAML XSW, JWKS confusion)
offsec-ai auth-attack https://auth.example.com \
  --i-have-authorization --mode deep --llm-judge

# Export attack report
offsec-ai auth-attack https://auth.example.com \
  --i-have-authorization --mode deep --format json --output auth-attack.json
```

---

## Scan Phases

The scanner runs five sequential phases for OIDC/OAuth2 targets, or three phases for SAML.

| Phase | Description |
|-------|-------------|
| 1 — Discovery | Probe `/.well-known/openid-configuration` and `/.well-known/oauth-authorization-server` for OIDC; `/saml/metadata`, `/saml2/metadata`, `/metadata`, `/FederationMetadata/2007-06/FederationMetadata.xml` for SAML |
| 2 — Fingerprint | Match response headers and body against provider signatures (Keycloak, Auth0, Okta, Microsoft Entra ID, Google, AWS Cognito, Shibboleth, Spring, GitLab, Ping Identity, GitHub) |
| 3 — Security Posture | Check PKCE enforcement, implicit flow, state parameter, alg=none, JWKS cache headers, SAML signing certificates, XML Signature Wrapping surface |
| 4 — CVE Matching | Cross-reference findings against the built-in `AUTH_CVE_DB` (14 entries) |
| 5 — LLM Triage | Optional — enriches MEDIUM/LOW findings; upgrades LOW→MEDIUM if LLM confidence > 0.7 |

---

## Security Checks

| Check ID | Protocol | Severity | Title |
|----------|----------|----------|-------|
| OFFSEC-AUTH-PKCE-001 | OIDC/OAuth2 | HIGH | PKCE not supported |
| OFFSEC-AUTH-PKCE-002 | OIDC/OAuth2 | MEDIUM | PKCE supported but not required |
| OFFSEC-AUTH-IMPL-001 | OIDC/OAuth2 | HIGH | Implicit flow enabled |
| OFFSEC-AUTH-JWTALGN-001 | OIDC | HIGH | alg=none in supported algorithms |
| OFFSEC-AUTH-STATE-001 | OIDC/OAuth2 | MEDIUM | State parameter not enforced |
| OFFSEC-AUTH-JWKS-001 | OIDC | LOW | JWKS endpoint lacks cache-control header |
| OFFSEC-AUTH-SAML-NOSIG | SAML | HIGH | No signing certificate in SAML metadata |
| OFFSEC-AUTH-SAML-NOACS | SAML | MEDIUM | No AssertionConsumerService endpoint |
| OFFSEC-AUTH-SAML-XSW | SAML | INFO | XML Signature Wrapping attack surface present |

---

## CVE / Advisory Database

| ID | CVE | Affected Provider | Severity | Description |
|----|-----|-------------------|----------|-------------|
| AUTH-ADV-SPRING-REDIRECT | CVE-2019-3778 | Spring Security OAuth | CRITICAL | Open redirect via malformed redirect_uri before OAuth token grant |
| AUTH-ADV-SAML-SHIBBOLETH | CVE-2018-0489 | Shibboleth SP | HIGH | SAML XML Signature Wrapping — unsigned assertion accepted |
| AUTH-ADV-SAML-ONELOGIN | CVE-2017-11427 | OneLogin / generic SAML | HIGH | SAML XSW — signature validation bypass via comment injection |
| AUTH-ADV-KEYCLOAK-SESSION | CVE-2023-41900 | Keycloak | HIGH | Session fixation via OIDC back-channel logout endpoint |
| AUTH-ADV-PKCE | — | universal | HIGH | Missing PKCE enforcement enables authorization code interception (PKCE Downgrade) |
| AUTH-ADV-IMPLICIT | — | universal | HIGH | Implicit flow exposes access tokens in browser history and referrer headers |
| AUTH-ADV-STATE | — | universal | HIGH | Missing state parameter enables CSRF on authorization code flow |
| AUTH-ADV-ALGNONE | — | universal | CRITICAL | JWT alg=none accepted — complete authentication bypass |
| AUTH-ADV-CODE-REUSE | — | universal | HIGH | Authorization code accepted more than once (replay) |
| AUTH-ADV-SAML-REPLAY | — | universal | HIGH | SAML assertion accepted after NotOnOrAfter expiry |
| AUTH-ADV-SCOPE | — | universal | MEDIUM | Scope escalation — server grants more scopes than requested |
| AUTH-ADV-JWKS | — | universal | LOW | JWKS endpoint lacks cache-control (JWKS confusion surface) |
| AUTH-ADV-OPEN-REDIR | — | universal | HIGH | Open redirect via redirect_uri without strict allowlist |
| AUTH-ADV-PKCE-REQUIRED | — | universal | MEDIUM | PKCE supported but not required — PKCE downgrade possible |

---

## CLI Reference

### `auth-scan`

```
offsec-ai auth-scan [OPTIONS] TARGET
```

| Option | Default | Description |
|--------|---------|-------------|
| `--protocol` | `auto` | Protocol to probe: `auto`, `oidc`, `oauth2`, `saml` |
| `--header` | — | Extra request header (`Key: Value`). Repeatable. |
| `--timeout` | `15.0` | Per-request timeout in seconds |
| `--no-tls-verify` | off | Disable TLS certificate verification |
| `--format` | `console` | Output format: `console` or `json` |
| `--output`/`-o` | — | Write output to file |
| `--llm-judge` | off | Enable LLM judge for MEDIUM/LOW triage |

### `auth-attack`

```
offsec-ai auth-attack [OPTIONS] TARGET
```

| Option | Default | Description |
|--------|---------|-------------|
| `--i-have-authorization` | required | Confirm you have written authorization to test the target |
| `--protocol` | `auto` | Protocol hint for Phase 1 scan |
| `--mode` | `safe` | Attack depth: `safe` (redirect/state/PKCE) or `deep` (all attacks) |
| `--header` | — | Extra request header. Repeatable. |
| `--timeout` | `15.0` | Per-request timeout |
| `--format` | `console` | Output format: `console` or `json` |
| `--output`/`-o` | — | Write output to file |
| `--llm-judge` | off | Enable LLM judge for attack-path narrative |

---

## Attack Suite

### Safe Mode

| Attack | Payloads | Description |
|--------|----------|-------------|
| Open Redirect | 5 | Tests `redirect_uri` parameter for open-redirect acceptance |
| State Bypass | 3 | Sends requests with missing, invalid, or replayed `state` values |
| PKCE Bypass | 3 | Omits or downgrades `code_challenge` / `code_challenge_method` |

### Deep Mode (adds)

| Attack | Payloads | Description |
|--------|----------|-------------|
| Scope Escalation | 3 | Requests `admin`, `offline_access`, and over-scoped grants |
| JWT alg=none | 2 | Forges JWTs with `"alg":"none"` and `"alg":"None"` (case variation) |
| Token Replay | 1 | Re-submits a previously seen authorization code |
| SAML XSW | 3 | Sends XSW1 (moved Signature), XSW2 (cloned element), comment_inject variants to ACS |
| JWKS Confusion | 1 | RS256 → HS256 key confusion probe |

---

## Provider Fingerprinting

The scanner identifies the following providers automatically:

| Provider | Detection method |
|----------|-----------------|
| **Keycloak** | `issuer` URL contains `/realms/` or response body contains `keycloak` |
| **Auth0** | Issuer matches `*.auth0.com` or `*.us.auth0.com` |
| **Okta** | Issuer matches `*.okta.com` or `*.oktapreview.com` |
| **Microsoft Entra ID** | Issuer matches `login.microsoftonline.com` |
| **Google** | Issuer `accounts.google.com` |
| **AWS Cognito** | Issuer matches `cognito-idp.*.amazonaws.com` |
| **Shibboleth** | Server header or body contains `Shibboleth` |
| **Spring Security** | Response body or issuer path contains `spring` |
| **GitLab** | Issuer matches GitLab hostname pattern |
| **Ping Identity** | Server header or issuer contains `ping` |
| **GitHub** | Issuer matches `token.actions.githubusercontent.com` |

---

## Python API

```python
import asyncio
from offsec_ai import AuthScanner, AuthAttacker, LLMJudge
from offsec_ai.exceptions import AuthorizationRequired

async def main():
    judge = LLMJudge.from_env()   # reads GEMINI_API_KEY / ANTHROPIC_API_KEY / OPENAI_API_KEY

    # --- Passive scan ---
    scanner = AuthScanner(
        target="https://accounts.google.com",
        protocol="auto",
        judge=judge,
        timeout=15.0,
        verify_tls=True,
    )
    result = await scanner.scan()

    print(f"Protocol  : {result.protocol.value}")
    print(f"Provider  : {result.provider_info.name}")
    print(f"Issuer    : {result.provider_info.issuer}")
    print(f"PKCE req  : {result.provider_info.pkce_required}")
    print(f"Implicit  : {result.provider_info.implicit_flow_enabled}")
    print(f"Vulns     : {len(result.all_vulns)}")
    print(f"CVE hits  : {len(result.cve_matches)}")
    for vuln in result.all_vulns:
        print(f"  [{vuln.severity.value:8}] {vuln.vuln_id}: {vuln.title}")
        if vuln.llm_reasoning:
            print(f"    LLM: {vuln.llm_reasoning[:120]}")

    # --- SAML scan ---
    saml = AuthScanner(
        target="https://mocksaml.com/api/saml/metadata",
        protocol="saml",
    )
    saml_result = await saml.scan()
    print(f"SAML entityID   : {saml_result.provider_info.issuer}")
    print(f"Signing certs   : {saml_result.provider_info.raw.get('signing_cert_count', 0)}")
    print(f"ACS endpoints   : {saml_result.provider_info.endpoints.get('acs', 'n/a')}")

    # --- Authorized attack ---
    try:
        attacker = AuthAttacker(authorized=True)
        report = await attacker.attack(
            target="https://auth.example.com",
            mode="safe",
            judge=judge,
        )
        print(f"Attacks run      : {report.attacks_run}")
        print(f"Attacks triggered: {report.attacks_triggered}")
        for r in report.triggered_results:
            print(f"  [{r.severity.value}] {r.title}")
            print(f"    evidence: {r.evidence}")
    except AuthorizationRequired as exc:
        print(exc)

asyncio.run(main())
```

---

## Remediation Guidance

### PKCE (OFFSEC-AUTH-PKCE-001 / 002)

Require `code_challenge_method=S256` for **all** public clients. Reject requests without
a `code_challenge`. See [RFC 7636](https://tools.ietf.org/html/rfc7636).

### Implicit Flow (OFFSEC-AUTH-IMPL-001)

Disable implicit (`response_type=token`) and hybrid flows. Use the authorization code flow
with PKCE for SPAs. See [OAuth 2.0 Security BCP §2.1.2](https://www.rfc-editor.org/rfc/rfc9700#section-2.1.2).

### State Parameter (OFFSEC-AUTH-STATE-001)

Generate a cryptographically random `state` value (≥ 128 bits) per authorization request.
Verify it matches on the callback before processing the code. Use `nonce` additionally for
OIDC to prevent replay.

### JWT alg=none (OFFSEC-AUTH-JWTALGN-001)

Hard-code the expected algorithm (`RS256` or `ES256`) on the resource-server side.
Never rely on the `alg` header from an untrusted JWT. Reject tokens with `alg=none`.

### SAML XML Signature Wrapping (OFFSEC-AUTH-SAML-NOSIG / SAML-XSW)

- Ensure every assertion is signed (not just the response envelope).
- Validate that the `ID` attribute in the `<Assertion>` matches the `AssertionID` referenced
  by the `<Reference>` inside the `<Signature>`.
- Use a SAML library that has explicit XSW mitigations (e.g., python-saml ≥ 2.6, OpenSAML ≥ 3.4).

### JWKS (OFFSEC-AUTH-JWKS-001)

Serve the JWKS endpoint with `Cache-Control: max-age=86400, public` to reduce the number of
inbound requests and mitigate cache-poisoning surfaces.

---

## Public Test Endpoints

| Endpoint | Protocol | Notes |
|----------|----------|-------|
| `https://accounts.google.com` | OIDC/OAuth2 | Google's production IdP — safe to scan passively |
| `https://mocksaml.com/api/saml/metadata` | SAML | Dedicated public SAML test service |
| `https://login.microsoftonline.com/{tenant-id}/v2.0` | OIDC | Microsoft Entra ID (requires tenant ID) |

> **Note**: Only run the `auth-attack` command against endpoints you own or have written
> authorization to test. Never run attacks against production identity providers you do not control.
