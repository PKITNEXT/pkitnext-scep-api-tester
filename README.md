# PKITNEXT LABS SCEP API Tester

> Free Windows & macOS diagnostic tool for SCEP certificate enrollment endpoints

[![Latest Release](https://img.shields.io/github/v/release/PKITNEXT/pkitnext-scep-api-tester?label=download&color=0052a0)](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest)

## Download

Go to [**Releases**](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest) and download:

| Platform | File |
|---|---|
| Windows x64 | `scep-tester.exe` |
| macOS Apple Silicon (M1/M2/M3) | `scep-tester-macos-arm64.zip` |
| macOS Intel (x86_64) | `scep-tester-macos-intel.zip` |

No installation required — just run the executable.

### macOS: First Launch

The app is **ad-hoc signed** but not notarized by Apple.
On first launch macOS Gatekeeper will show a warning. To open it:

**Option A — via Finder:**
Right-click `scep-tester.app` → **Open** → confirm **Open** in the dialog.

**Option B — via Terminal:**
```bash
xattr -d com.apple.quarantine scep-tester.app
open scep-tester.app
```

After the first approval the app starts normally every time.

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

PKITNEXT LABS develops PKI and certificate lifecycle management solutions —
automated certificate enrollment via SCEP and ACME for enterprise environments,
embedded devices, and cloud-native infrastructure.

🌐 [pkitnext.de](https://www.pkitnext.de)

---
© 2026 PKITNEXT LABS
