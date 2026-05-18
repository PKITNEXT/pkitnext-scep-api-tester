# PKITNEXT LABS – Tools & Agents

> Free Windows & macOS tools for SCEP certificate enrollment and automated certificate lifecycle management

[![Latest Release](https://img.shields.io/github/v/release/PKITNEXT/pkitnext-scep-api-tester?label=download&color=0052a0)](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest)

---

## SCEP API Tester

Diagnostic tool to test a SCEP server endpoint through the full enrollment cycle.

### Download

Go to [**Releases**](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest) and download:

| Platform | File |
|---|---|
| Windows x64 | `scep-tester.exe` |
| macOS Apple Silicon | `scep-tester-macos-arm64.zip` |
| macOS Intel | `scep-tester-macos-intel.zip` |

No installation required — just run the executable.

### What it does

Tests a SCEP server endpoint through the full enrollment cycle:

```
GetCACaps → GetCACert → PKCSReq → CertRep
```

**Features:**
- Challenge password support
- TLS verification bypass (dev/lab mode)
- Verbose debug log with step-by-step SCEP trace
- Mitel Phone Mode (BER-ordered CSR attributes)
- Save CA certificate chain as PEM
- About dialog with EULA

---

## Windows Certificate Agent

Automated certificate enrollment and renewal via SCEP as a Windows service.

### Download

Go to [**Releases**](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest) and download:

| File | Description |
|------|-------------|
| `windows-scep-client-*.msi` | MSI installer (recommended) – installs to `C:\Program Files\PKITNEXT\SCEP Agent\` |
| `windows-scep-client-*-x64.zip` | Portable ZIP – EXE + example config |

### What it does

- Enrolls a certificate from a SCEP server (PKCSReq)
- Stores the certificate and private key as PEM files
- Optionally imports the certificate into the Windows Certificate Store
- Runs as a Windows service and automatically renews the certificate before expiry
- Configurable renewal threshold (default: 30 days before expiry)

### Quickstart

1. Edit `windows-scep-client.yaml` for your environment:

```yaml
scep_url: https://your-scep-server/scep/api/your-profile/
common_name: myhost.example.com
cert_output: C:\ProgramData\PKITNEXT\cert.pem
key_output:  C:\ProgramData\PKITNEXT\key.pem
```

2. Run as **Administrator**:

```powershell
# Enroll certificate (one-time test)
windows-scep-client.exe --config windows-scep-client.yaml enroll

# Install and start as Windows service
windows-scep-client.exe --config windows-scep-client.yaml install-service
windows-scep-client.exe start-service

# Check certificate status
windows-scep-client.exe --config windows-scep-client.yaml status

# Remove service
windows-scep-client.exe stop-service
windows-scep-client.exe uninstall-service
```

**Requirements:**
- Windows 10 / Windows Server 2016 or newer (x64)
- Administrator rights for service installation
- Network access to the SCEP server

---

## About PKITNEXT LABS

PKITNEXT LABS develops PKI and certificate lifecycle management solutions and automated certificate enrollment.

🌐 [pkitnext.de](https://www.pkitnext.de)

---
© 2026 PKITNEXT LABS
