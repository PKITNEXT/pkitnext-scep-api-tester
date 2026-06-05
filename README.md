# PKITNEXT LABS – SCEP Tools for Windows & macOS

> Modern SCEP enrollment testing and certificate lifecycle automation toolkit for enterprise environments.

[![Latest Release](https://img.shields.io/github/v/release/PKITNEXT/pkitnext-scep-api-tester?label=latest%20release&color=0052a0)](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest)
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS-blue)](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest)
[![Built with Rust](https://img.shields.io/badge/built%20with-Rust-orange)](https://www.rust-lang.org/)

---

## What's in this repo?

| Tool | Purpose | Platforms |
|---|---|---|
| [**SCEP API Tester**](#scep-api-tester) | GUI diagnostic tool — test a SCEP server in one click | Windows, macOS |
| [**Windows Certificate Agent**](#windows-certificate-agent) | Automated enrollment & renewal as a Windows service | Windows |

Both tools are built on a shared library that implements RFC 8894.

---

## Why PKITNEXT?

| Problem | Existing Tools | PKITNEXT |
|---|---|---|
| **SCEP server debugging** | Difficult — raw logs, Wireshark, trial and error | One click — full trace with step-by-step SCEP debug log |
| **Certificate renewal** | Manual — someone has to remember and act | Automated — Windows service renews before expiry, zero downtime |
| **Mitel phone support** | Broken — standard tools send wrong CSR attribute order | Built-in Mitel Phone Mode with correct BER-ordered attributes |
| **Cross-platform testing** | Limited — most tools Windows-only | Windows x64 + macOS (Apple Silicon and Intel) |
| **Deployment** | Complex — dependencies, runtimes, installers | Portable single executable — download and run |
| **Private key safety** | Often unclear | Private key generated locally, never transmitted, never logged |
| **Audit trail** | None | Verbose debug log exportable to file |

---

## Architecture

```
┌───────────────────────────────────────────────────────────┐
│                      Windows / macOS                      │
│                                                           │
│   ┌─────────────────────┐   ┌─────────────────────────┐   │
│   │   SCEP API Tester   │   │  Windows Cert Agent     │   │
│   │   (GUI · eframe)    │   │ (CLI + Windows Service) │   │
│   └──────────┬──────────┘   └───────────┬─────────────┘   │
│              └──────────┬───────────────┘                 │
│              ┌──────────▼──────────┐                      │
│              │      scep-core      │                      │
│              │  (Rust library)     │                      │
│              │  RFC 8894 / CMS     │                      │
│              └──────────┬──────────┘                      │
└─────────────────────────┼─────────────────────────────────┘
                          │  HTTPS  (RFC 8894)
          ┌───────────────▼──────────────────┐
          │     SCEP Certificate Authority   │
          │  PKITNEXT  │  MS NDES  │  EJBCA  │
          │  OpenXPKI  │  any RFC 8894 CA    │
          └──────────────────────────────────┘
```

### SCEP Protocol Flow

```
Client                              CA
  │                                  │
  │──── GET  GetCACaps ─────────────►│  Capabilities (SHA-256, AES, …)
  │                                  │
  │──── GET  GetCACert ─────────────►│  CA certificate chain (DER / PKCS#7)
  │                                  │
  │  [generate RSA key pair + CSR]   │
  │                                  │
  │──── POST PKIOperation ──────────►│  PKCSReq (CMS EnvelopedData → SignedData → CSR)
  │                                  │
  │◄─── CertRep ─────────────────────│  pkiStatus: SUCCESS / PENDING / FAILURE
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
| macOS Apple Silicon (M1/M2/M3) | `scep-tester-macos-arm64.zip` |
| macOS Intel (x86_64) | `scep-tester-macos-intel.zip` |

### Install in under 5 minutes

**Windows:**
1. Download `scep-tester.exe`
2. Double-click — no installer, no runtime required
3. Enter your SCEP URL and click **Test**

**macOS:**
1. Download and unzip the archive for your chip (Apple Silicon or Intel)
2. The app is **ad-hoc signed** but not notarized by Apple. On first launch macOS Gatekeeper will show a warning.

   **Option A — via Finder:** Right-click `scep-tester.app` → **Open** → confirm **Open** in the dialog.

   **Option B — via Terminal:**
   ```bash
   xattr -d com.apple.quarantine scep-tester.app
   open scep-tester.app
   ```
3. After the first approval the app starts normally every time.

### Screenshots

![PKITNEXT SCEP API Tester – successful test run](screenshots/Screenshot__Tester1.jpg)

![PKITNEXT SCEP API Tester – NDES mode with auto OTP fetch](screenshots/Screenshot__Tester_NDES.jpg)

**NDES Mode** enables automatic enrollment against Microsoft AD CS / NDES (`mscep.dll`) endpoints.
Enable it by checking *NDES Mode*, enter the Windows account credentials for `mscep_admin`,
and click **Test** — the tester fetches the one-time challenge password automatically,
builds and encrypts the PKCSReq, and decrypts the issued certificate from the CertRep response.
No manual OTP copy-paste required.
If the OTP fetch fails, the tester falls back to the manually entered challenge password (configurable).

### What the tester checks

Runs the full SCEP enrollment cycle and shows the result live in the UI:

| Step | Operation | What you see |
|---|---|---|
| 1 | `GetCACaps` | Server capabilities (algorithms, POST support, …) |
| 2 | `GetCACert` | CA certificate chain with subject and validity |
| 3 | `PKCSReq` | RSA key pair + CSR generated on the fly, sent to CA |
| 4 | `CertRep` | Issued certificate: subject, SANs, validity, PEM |

### Features

**Connectivity & authentication**
- Any SCEP server — PKITNEXT, Microsoft NDES / AD CS, EJBCA, OpenXPKI
- **NDES Mode** — auto-fetch one-time challenge password from `mscep_admin` using Windows credentials; automatic fallback to manual challenge on fetch failure
- Manual challenge password
- TLS certificate verification bypass (dev / lab mode)

**Certificate request configuration** *(expandable panel, all fields pre-filled with defaults)*
- DNS SAN — editable (default: built-in test name)
- IP SAN — IPv4 and IPv6
- Email SAN — rfc822Name in SubjectAltName extension
- Subject O — Organization
- Subject OU — Organizational Unit
- Subject emailAddress — email in the subject DN
- SAN extension Critical flag

**Crypto profiles — selected automatically from `GetCACaps`**
- Modern: SHA-256 + AES-128-CBC
- Legacy MSCEP: SHA-1 + 3DES-CBC
- Key transport: RSA PKCS#1 v1.5 and **RSAES-OAEP** (required by some NDES configurations)

**Results & export**
- Issued certificate details: subject, issuer, serial, validity, SANs
- Save end-entity certificate as PEM file
- Save CA / RA certificate chain as PEM file
- **Copy to clipboard** button on every panel
- **Show / Hide** toggle on every panel to manage screen space
- If certificate extraction fails: **Show hex raw** — displays the raw CertRep response in hex for protocol-level debugging

**Debug log**
- Verbose step-by-step protocol trace (enable *Debug mode*)
- Shows every HTTP request/response, crypto parameters, parsed fields
- NDES OTP value is always redacted — safe to share with support
- Save log to file for support tickets

**Mitel Phone Mode** — BER-ordered CSR attributes for Mitel SIP phones

**CA Compatibility**

| CA | Status |
|---|---|
| PKITNEXT | ✅ Full support |
| Microsoft NDES / AD CS | ✅ Full support incl. RSAES-OAEP, auto OTP fetch |
| EJBCA | ✅ Standard RFC 8894 |
| OpenXPKI | ✅ Standard RFC 8894 |
| Mitel PBX | ✅ Mitel Phone Mode |

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

---

## Security Design

Security software requires trust. Here is what these tools do — and what they deliberately do not do:

| Property | Detail |
|---|---|
| **No telemetry** | No data is sent anywhere except to your configured SCEP server |
| **No cloud dependency** | Fully offline-capable — no license server, no update check, no analytics |
| **Local key generation** | RSA key pairs are generated on the machine, never transmitted |
| **Private key never leaves the machine** | Only the CSR (public key + subject) is sent to the CA |
| **TLS validation** | TLS certificate verification is enabled by default; bypass is opt-in (lab/dev only) |
| **RFC 8894 compliant** | Standard SCEP protocol — no proprietary extensions required |
| **Memory-safe implementation** | Written in Rust — no buffer overflows, no use-after-free, no null pointer dereferences |

---

## CA Compatibility

| CA | SCEP API Tester | Windows Cert Agent |
|---|---|---|
| **PKITNEXT** | ✅ Full support | ✅ Full support |
| **Microsoft NDES / AD CS** | ✅ Full support incl. RSAES-OAEP, auto OTP fetch | ✅ Standard RFC 8894 |
| **EJBCA** | ✅ Standard RFC 8894 | ✅ Standard RFC 8894 |
| **OpenXPKI** | ✅ Standard RFC 8894 | ✅ Standard RFC 8894 |
| **Mitel PBX** | ✅ Mitel Phone Mode | — |

---

## Roadmap

### ✅ Phase 1 — Windows & macOS Tools (released)

| Tool | Description |
|---|---|
| SCEP API Tester | GUI diagnostic tool for Windows and macOS |
| Windows Certificate Agent | CLI + Windows Service — MSI installer + portable ZIP |

---

### 🔄 Phase 2 — Linux Certificate Agent (next release)

Automated certificate lifecycle management for Linux servers — packaged as RPM, deployed via systemd.

**Apache Agent** (`pkitnext-agent`) — Scans Apache config directories, enrolls via SCEP, writes PEM files, runs `apachectl configtest`, then `systemctl reload`. Rolls back to backup on any error.

**Tomcat Agent** (`pkitnext-tomcat-agent`) — Scans `server.xml` for SSL keystore references, builds PKCS#12 keystore, writes atomically, then `systemctl restart tomcat`.

**UC Certificate Installer** — Specialized for Mitel (former Unify) OpenScape UC Application V10/V11.

---

### 📋 Phase 3 — Broader Linux Support

| Feature | Description |
|---|---|
| DEB / APT packaging | Ubuntu and Debian support alongside RPM |
| Nginx Agent | SSL cert management for Nginx virtual hosts |
| HAProxy Agent | Certificate updates for HAProxy `bind` directives |

---

### 📋 Phase 4 — Enterprise & Cloud

| Feature | Description |
|---|---|
| macOS Certificate Agent | LaunchDaemon-based service for macOS servers |
| Multi-CA failover | Multiple SCEP endpoints with automatic failover |
| Ansible role | Deploy and configure the Linux agent via Ansible |
| EST protocol (RFC 7030) | Modern alternative to SCEP for EST-capable CAs |

---

## About PKITNEXT LABS

PKITNEXT LABS develops PKI and certificate lifecycle management solutions.

🌐 [pkitnext.de](https://www.pkitnext.de)

---

© 2026 PKITNEXT LABS · All rights reserved
