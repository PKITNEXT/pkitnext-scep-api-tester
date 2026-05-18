# PKITNEXT LABS SCEP API Tester

> Free Windows & macOS diagnostic tool for SCEP certificate enrollment endpoints

[![Latest Release](https://img.shields.io/github/v/release/PKITNEXT/pkitnext-scep-api-tester?label=latest%20release&color=0052a0)](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest)
[![License: Free](https://img.shields.io/badge/license-free-brightgreen)](#license)

---

## Download

Go to [**Releases**](https://github.com/PKITNEXT/pkitnext-scep-api-tester/releases/latest) and pick your platform:

| Platform | File | Notes |
|---|---|---|
| **Windows x64** | `scep-tester.exe` | Run directly — no install required |
| **macOS Apple Silicon** | `scep-tester-macos-arm64.zip` | Unzip → run `scep-tester.app` |
| **macOS Intel** | `scep-tester-macos-intel.zip` | Unzip → run `scep-tester.app` |

No installation, no dependencies — just download and run.

---

## What It Does

The PKITNEXT LABS SCEP API Tester runs a complete SCEP certificate enrollment cycle against any
SCEP endpoint and shows you exactly what happens at each step:

```
Step 1/4  GetCACaps   ─── queries server capabilities
Step 2/4  GetCACert   ─── retrieves CA / RA certificate chain
Step 3/4  PKCSReq     ─── builds and submits the certificate request
Step 4/4  CertRep     ─── receives and decodes the CA response
```

### Features

- **Challenge password** support
- **TLS verification bypass** for dev and lab environments
- **Verbose debug log** — step-by-step SCEP protocol trace, saveable as `.log`
- **Mitel Phone Mode** — CSR attributes in BER order (non-canonical), required by some PBX systems
- **CA certificate chain viewer** — inspect Subject, Issuer, Serial, validity and download as PEM
- **Certificate output** — view and save the issued certificate as PEM
- Professional blue title bar with About dialog and full EULA

---

## Use Cases

- Verify SCEP connectivity before deploying agents or MDM profiles
- Debug challenge password and CA configuration issues
- Test SCEP compatibility with VoIP phones (Mitel, Cisco, Yealink, etc.)
- Validate custom or third-party SCEP server implementations
- Quick sanity-check after CA, NDES, or firewall reconfiguration

---

## About PKITNEXT LABS

**PKITNEXT LABS** is a specialized provider of PKI and certificate lifecycle management solutions.
We develop tools, agents, and integrations for automated certificate enrollment via SCEP and ACME —
for enterprise environments, embedded devices, and cloud-native infrastructure.

**Typical use cases:**
- VoIP & SIP/TLS — certificates for phones, SBCs, and gateways
- Industrial & IoT — scalable for thousands of devices, offline-capable
- Internal IT & APIs — REST integration, no manual ticket processes

🌐 [pkitnext.de](https://www.pkitnext.de)

---

## License

This tool is provided **free of charge** for diagnostic and testing purposes only.  
PKITNEXT LABS assumes no liability for damages arising from its use.  
See the About dialog inside the application for the full EULA.

---

*© 2026 PKITNEXT LABS*
