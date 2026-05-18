# PKITNEXT LABS SCEP API Tester

> Free Windows & macOS diagnostic tool for SCEP certificate enrollment endpoints

[![Latest Release](https://img.shields.io/github/v/release/PKITNEXT/pkitnext-scep-api-tester?label=download&color=0052a0)](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest)

## Download

Go to [**Releases**](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest) and download:

| Platform | File |
|---|---|
| Windows x64 | `scep-tester.exe` |
| macOS Apple Silicon | `scep-tester-macos-arm64.zip` |
| macOS Intel | `scep-tester-macos-intel.zip` |

No installation required — just run the executable.

## What it does

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

## About PKITNEXT LABS

PKITNEXT LABS develops PKI and certificate lifecycle management solutions and automated certificate enrollment.

🌐 [pkitnext.de](https://www.pkitnext.de)

---
© 2026 PKITNEXT LABS
