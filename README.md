# SSL Check Tool

A PowerShell HTTPS diagnostic tool for enterprise environments. Goes far beyond `Invoke-WebRequest` and `Test-NetConnection` — from DNS resolution through to full TLS inspection, real-world trust validation, certificate security analysis, HTTP response, ICMP ping, certificate export, and HTML report generation. All in a single staged run with no external modules or installs required.

## Requirements

- Windows PowerShell 5.1 or PowerShell 7+
- Windows 8 / Server 2012 or later
- No external modules or installs required
- TLS 1.3 probing requires Windows 10 1903+ or Windows Server 2022
- Traceroute requires `Test-NetConnection` (PS 4.0+ / Windows 8+)

---

## How It Works

The script runs in stages. Each stage gates the next — if DNS fails, TCP is not attempted. If TCP fails, TLS is not attempted. Every failure is caught gracefully and reported at the stage where it occurs.

```
Stage 1    DNS Resolution              Resolves hostname, shows all IPs
Stage 2    TCP Reachability            Connect test with traceroute on failure
Stage 3a   TLS Handshake              Cert inspection with bypass (full detail)
Stage 3b   Real-World Trust           Validates against Windows cert store
Stage 3c   HTTP Response              Real GET via Invoke-WebRequest
Stage 4*   Legacy TLS Audit           SSL2/3, TLS 1.0-1.3 probes
           Connection Summary         Source and destination IP / hostname
           ICMP Ping                  4-packet ping with statistics
           Warnings                   All advisories printed once at the end
```
\* Optional — enabled with `-AuditLegacyTls`

---

## Usage

```powershell
.\sslCheck.ps1 -Uri <endpoint> [options]
```

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `-Uri` | String | *(required)* | Target endpoint. `https://` prefix optional. Accepts FQDN, hostname, or IP. |
| `-Port` | Int | `443` | TCP port. Overridden by port embedded in URI. |
| `-TimeoutMs` | Int | `5000` | Connection timeout in ms. Increase for slow private endpoints. |
| `-TraceRouteHops` | Int | `15` | Max hops for traceroute on TCP failure. |
| `-AuditLegacyTls` | Switch | off | Probe SSL 2.0, SSL 3.0, TLS 1.0–1.3. Non-destructive. |
| `-SkipTraceroute` | Switch | off | Suppress traceroute when TCP fails. |
| `-RetryCount` | Int | `0` | Additional TCP attempts after first failure. |
| `-RetryDelayMs` | Int | `1000` | Wait between retries in ms. |
| `-ExportCerts` | Switch | off | Export all chain certificates to disk. |
| `-ExportPath` | String | `Documents\sslCheck-Output` | Directory for certificate export. |
| `-SaveReport` | Switch | off | Save full run output as an HTML report. |
| `-ReportPath` | String | `Documents\sslCheck-Output` | Directory for HTML report. |

---

## Examples

```powershell
# Basic check
.\sslCheck.ps1 -Uri https://example.com

# Internal / private endpoint with longer timeout
.\sslCheck.ps1 -Uri https://internal-api.corp.local -TimeoutMs 10000

# HTTPS on non-standard port
.\sslCheck.ps1 -Uri https://example.com:8443
.\sslCheck.ps1 -Uri https://example.com -Port 8443

# Full TLS audit including SSL 2.0 / 3.0
.\sslCheck.ps1 -Uri https://example.com -AuditLegacyTls

# Flaky endpoint with retries
.\sslCheck.ps1 -Uri https://example.com -RetryCount 2 -RetryDelayMs 2000

# Export certificates and save HTML report
.\sslCheck.ps1 -Uri https://example.com -ExportCerts -SaveReport

# Full run — everything
.\sslCheck.ps1 -Uri https://example.com -AuditLegacyTls -ExportCerts -SaveReport
```

---

## What Gets Checked

### Stage 1 — DNS Resolution
Resolves the hostname using an async lookup with a 3-second timeout. Displays all resolved IP addresses — useful for confirming whether a private or public address is being hit. Handles split-horizon DNS and internal zones. Raw IP addresses bypass this stage entirely.

### Stage 2 — TCP Reachability
Performs an explicit TCP connect to the target host and port with configurable timeout and retry support. Connection time is reported in milliseconds. On failure, automatically runs a fast Traceroute (capped at 15 hops with reverse DNS lookups suppressed) to identify where in the network path traffic is being dropped — firewall, routing black hole, or unreachable host. Suppressed with `-SkipTraceroute`.

### Stage 3a — TLS Handshake & Certificate Inspection
Uses a certificate bypass connection to inspect the cert regardless of whether it is trusted on the current machine. Reports:

- Negotiated TLS version — colour-coded (TLS 1.3 green, TLS 1.2 yellow, TLS 1.0/1.1 red)
- Cipher algorithm and strength — weak ciphers (RC4, DES, 3DES, RC2, NULL) flagged red
- SNI sent and HTTP version negotiated
- Certificate subject, issuer, thumbprint (lowercase), serial number
- Signature algorithm — MD5 flagged red, SHA-1 flagged yellow
- Public key type and size — RSA under 2048-bit flagged red
- Expiry with days remaining — colour-coded urgency at 90, 30, and 0 days
- Certificate type: Self-Signed / Publicly Trusted CA / Private / Internal CA
- Full certificate chain from leaf to root — subject, issuer, thumbprint, and expiry per level
- Certificate Transparency (SCT) check for publicly trusted certificates
- SANs grouped by root domain with entry counts

### Stage 3b — Real-World Trust Validation
A second TLS connection without bypass using the Windows certificate store — exactly as `Invoke-WebRequest`, browsers, and applications behave. When this fails, it tells you exactly why and what to import to fix it.

### Stage 3c — HTTP Response
A real HTTP GET with redirect following disabled to prevent browser popups on SSO / SAML protected endpoints. Detects and labels Azure AD / SAML redirect responses. Reports status code, response time, all headers, images, input fields, links, raw content preview, and body size.

### Stage 4 — Legacy TLS Audit
Non-destructive isolated probes covering SSL 2.0, SSL 3.0, TLS 1.0, 1.1, 1.2, and 1.3. SSL 2.0 and SSL 3.0 flagged as critical (POODLE / DROWN). Does not affect the primary connection.

---

## Certificate Export (`-ExportCerts`)

Exports every certificate in the chain to `Documents\sslCheck-Output` in a timestamped subfolder:

```
Documents\sslCheck-Output\
    example.com_20260506_125207\
        leaf.cer                  DER binary - import into certlm.msc
        leaf.pem                  Base64 PEM - use with OpenSSL / nginx / Apache
        intermediate_1.cer
        intermediate_1.pem
        root.cer
        root.pem
        chain_full.pem            All certs concatenated leaf to root
        chain_full.p7b            PKCS#7 bundle - Windows / Java keystores
```

Private keys are never available from a TLS handshake and are never exported.

---

## HTML Report (`-SaveReport`)

Saves a self-contained HTML file of the full run output with all colour coding preserved. No external dependencies, no CDN, no JavaScript frameworks.

```
Documents\sslCheck-Output\
    sslCheck_example.com_2026-05-06-12-52-07.html
```

The report includes a **Print / Save PDF** button in the top right corner. Clicking it opens the browser print dialog — select **Save as PDF** to produce a clean, portable PDF with no extra tools needed. A print stylesheet switches the dark terminal theme to white background with black text for clean printing.

---

## Output Folder

Both certificates and reports save to the same location:

```
C:\Users\<username>\Documents\sslCheck-Output\
```

This path works without Administrator privileges on Windows 10, 11, and Server for all standard user accounts. Override either path with `-ExportPath` or `-ReportPath` if needed.

---

## Comparison with Built-in Tools

| Capability | Test-NetConnection | Invoke-WebRequest | sslCheck.ps1 |
|---|---|---|---|
| DNS resolution | Yes | No | Yes |
| Resolved IPs | Yes | No | Yes |
| Source IP | Yes | No | Yes |
| TCP connect result | Yes | No | Yes |
| Traceroute on failure | No | No | Yes |
| TLS version | No | No | Yes |
| Cipher & hash | No | No | Yes |
| Weak cipher detection | No | No | Yes |
| Certificate details | No | No | Yes |
| Signature algorithm check | No | No | Yes |
| Key size check | No | No | Yes |
| Certificate expiry | No | No | Yes |
| Full cert chain | No | No | Yes |
| Real-world trust validation | No | Partial | Yes |
| Trust failure reason | No | No | Yes |
| HTTP response & headers | No | Yes | Yes |
| SSO / SAML detection | No | No | Yes |
| ICMP ping with statistics | No | No | Yes |
| Legacy TLS audit | No | No | Yes |
| Certificate export | No | No | Yes |
| HTML report | No | No | Yes |

---

## Enterprise & Proxy Environments

When running behind a TLS inspection proxy such as Zscaler, all TLS and certificate results reflect the **client-to-proxy** connection. This is expected behaviour and the correct trust boundary to test — if the proxy cert is not in the Windows trust store, `Invoke-WebRequest` will fail and Stage 3b will surface the exact reason.

Self-signed and private / internal CA certificates are treated as informational rather than errors. Each type displays a neutral context note appropriate to the environment.

---

## Exit Codes

| Code | Meaning |
|---|---|
| `0` | Completed successfully |
| `1` | Halted — DNS resolution failed or invalid URI |
| `2` | Halted — TCP connection failed |
| `3` | Halted — unexpected error in TLS or trust stage |

---

## Author

**Hashim Hilal**

Read-only tool. No HTTP content is stored. No changes are made to any system.
