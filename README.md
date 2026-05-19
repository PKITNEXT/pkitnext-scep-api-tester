# PKITNEXT LABS – SCEP Tools for Windows & macOS

> Free tools for SCEP certificate enrollment and automated certificate lifecycle management

[![Latest Release](https://img.shields.io/github/v/release/PKITNEXT/pkitnext-scep-api-tester?label=latest%20release&color=0052a0)](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest)
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS-blue)](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest)
[![Built with Rust](https://img.shields.io/badge/built%20with-Rust-orange)](https://www.rust-lang.org/)

---

## What's in this repo?

| Tool | Purpose | Platforms |
|---|---|---|
| [**SCEP API Tester**](#scep-api-tester) | GUI diagnostic tool — test a SCEP server in one click | Windows, macOS |
| [**Windows Certificate Agent**](#windows-certificate-agent) | Automated enrollment & renewal as a Windows service | Windows |

Both tools are built in Rust on top of a shared `scep-core` library that implements RFC 8894.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Windows / macOS                        │
│                                                             │
│   ┌─────────────────────┐   ┌─────────────────────────┐    │
│   │   SCEP API Tester   │   │  Windows Cert Agent     │    │
│   │   (GUI · eframe)    │   │  (CLI + Windows Service)│    │
│   └──────────┬──────────┘   └───────────┬─────────────┘    │
│              │                          │                   │
│              └──────────┬───────────────┘                   │
│                         │                                   │
│              ┌──────────▼──────────┐                        │
│              │      scep-core      │                        │
│              │  (Rust library)     │                        │
│              │  RFC 8894 / CMS     │                        │
│              └──────────┬──────────┘                        │
└─────────────────────────┼───────────────────────────────────┘
                          │  HTTPS  (RFC 8894)
          ┌───────────────▼──────────────────┐
          │     SCEP Certificate Authority    │
          │                                  │
          │  PKITNEXT  │  MS NDES  │  EJBCA  │
          │  OpenXPKI  │  any RFC 8894 CA    │
          └──────────────────────────────────┘
```

### SCEP Protocol Flow

```
Client                              CA
  │                                  │
  │──── GET  GetCACaps ─────────────►│  Capabilities (SHA-256, AES, POSTPKIOperation …)
  │                                  │
  │──── GET  GetCACert ─────────────►│  CA certificate chain (DER / PKCS#7)
  │                                  │
  │  [generate RSA key pair + CSR]   │
  │                                  │
  │──── POST PKIOperation ──────────►│  PKCSReq  (CMS EnvelopedData → SignedData → CSR)
  │                                  │
  │◄─── CertRep ────────────────────│  pkiStatus: SUCCESS / PENDING / FAILURE
  │                                  │
  │  [decrypt → extract certificate] │
```

---

## SCEP API Tester

A graphical desktop tool to run a full SCEP enrollment test against any SCEP server endpoint.  
No installation required. Download, double-click, test.

### Download

Go to [**Releases**](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest) and download:

| Platform | File |
|---|---|
| Windows x64 | `scep-tester.exe` |
| macOS Apple Silicon | `scep-tester-macos-arm64.zip` |
| macOS Intel | `scep-tester-macos-intel.zip` |

### Install in under 5 minutes

**Windows:**
1. Download `scep-tester.exe`
2. Double-click — no installer, no runtime required
3. Enter your SCEP URL and click **Test**

**macOS:**
1. Download and unzip the archive for your chip (Apple Silicon or Intel)
2. Remove the quarantine flag and make executable:
   ```bash
   xattr -d com.apple.quarantine scep-tester
   chmod +x scep-tester
   ./scep-tester
   ```
3. Enter your SCEP URL and click **Test**

### What the tester checks

Runs the full SCEP enrollment cycle and shows the result live in the UI:

| Step | Operation | What you see |
|---|---|---|
| 1 | `GetCACaps` | Server capabilities (algorithms, POST support, …) |
| 2 | `GetCACert` | CA certificate chain with subject and validity |
| 3 | `PKCSReq` | RSA key pair + CSR generated on the fly, sent to CA |
| 4 | `CertRep` | Issued certificate: subject, SANs, validity, fingerprint |

**Features:**
- Challenge password support
- TLS verification bypass (dev / lab mode)
- Verbose step-by-step debug log — save to file for support tickets
- **Mitel Phone Mode** — BER-ordered CSR attributes for Mitel SIP phones
- Save the full CA certificate chain as PEM
- About dialog with EULA

---

## Windows Certificate Agent

A command-line tool and Windows service for automated certificate enrollment and renewal via SCEP.  
Set it up once, and it keeps your certificate valid forever.

### Download

Go to [**Releases**](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest) and download:

| File | Description |
|---|---|
| `windows-scep-client-*.msi` | MSI installer (recommended) — installs to `C:\Program Files\PKITNEXT\SCEP Agent\` |
| `windows-scep-client-*-x64.zip` | Portable ZIP — EXE + example config |

**Requirements:** Windows 10 / Server 2016 or newer (x64) · Administrator rights for service installation

### Install in under 5 minutes

```powershell
# 1. Generate a ready-to-use config file
windows-scep-client.exe --create-conf

# 2. Edit the config (Notepad or any editor)
notepad windows-scep-client.yaml

# 3. Test a one-time enrollment
windows-scep-client.exe --config windows-scep-client.yaml enroll

# 4. Install as auto-start Windows service (run PowerShell as Administrator)
windows-scep-client.exe --config C:\ProgramData\pkitnext\client.yaml install-service
windows-scep-client.exe start-service
```

Done. The agent checks the certificate every 24 hours and renews automatically 30 days before expiry.

### Configuration file

Generated by `--create-conf`. All options explained:

```yaml
# SCEP server URL
scep_url: "https://ca.example.com/scep/api/your-profile/"

# Certificate subject
common_name: "myhost.example.com"

# Subject Alternative Names (optional)
san_dns: "myhost.example.com"
# san_ip: "192.168.1.1"

# Output paths (PEM format)
cert_output: "C:/ProgramData/pkitnext/cert.pem"
key_output:  "C:/ProgramData/pkitnext/key.pem"

# SCEP challenge password
challenge_password: "your-challenge-secret"

# Renewal threshold: renew this many days before expiry (default: 30)
days_before_expiry: 30

# Service check interval in hours (default: 24)
check_interval_hours: 24

# Windows Certificate Store — also import into certstore (optional)
# cert_store: "LocalMachine\\MY"

# Log file for service mode (optional)
# log_file: "C:/ProgramData/pkitnext/windows-scep-client.log"
```

### All commands

```powershell
# Enrollment & renewal
windows-scep-client.exe --config client.yaml enroll               # Request a new certificate
windows-scep-client.exe --config client.yaml renew                # Renew if near expiry
windows-scep-client.exe --config client.yaml renew --force-renew  # Renew immediately

# Certificate status
windows-scep-client.exe --config client.yaml status               # Subject, expiry, path

# Windows Certificate Store test
windows-scep-client.exe --config client.yaml test-certstore       # Import → verify → remove

# Windows Service (requires Administrator)
windows-scep-client.exe --config client.yaml install-service
windows-scep-client.exe start-service
windows-scep-client.exe stop-service
windows-scep-client.exe uninstall-service

# Config helper
windows-scep-client.exe --create-conf   # Write default config to disk
windows-scep-client.exe --help          # Full help text
```

### How the service works

```
Service starts
      │
      ▼
Load config.yaml
      │
      ▼
┌──────────────────────────┐
│  Certificate exists?     │
│  Valid? > threshold?     │──── Yes ──► sleep check_interval_hours ──┐
└──────────┬───────────────┘                                          │
           │ No / expiry within threshold                             │
           ▼                                                          │
    SCEP Enrollment                                                   │
    PKCSReq → CertRep                                                 │
           │                                                          │
           ▼                                                          │
    Write cert.pem + key.pem                                          │
    (+ import to Windows Cert Store if cert_store is configured)     │
           │                                                          │
           └──────────────────────────────────────────────────────────┘
```

---

## CA Compatibility

| CA | Notes |
|---|---|
| **PKITNEXT** | Full support including scep_v2 server |
| **Microsoft NDES** | Standard RFC 8894 enrollment |
| **EJBCA** | Standard RFC 8894 enrollment |
| **OpenXPKI** | Standard RFC 8894 enrollment |
| **Mitel PBX** | Use Mitel Phone Mode in the SCEP API Tester |

---

## About PKITNEXT LABS

PKITNEXT LABS develops PKI and certificate lifecycle management solutions.

🌐 [pkitnext.de](https://www.pkitnext.de)

---

© 2026 PKITNEXT LABS · All rights reserved
