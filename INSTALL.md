# PKITNEXT Linux SCEP Agent - Installation Guide

This guide describes the full installation and initial configuration of the
PKITNEXT Linux SCEP Agent on SuSE (SLES / openSUSE) and RHEL-based systems
(RHEL, Rocky Linux, AlmaLinux).

---

## Contents

1. [System Requirements](#1-system-requirements)
2. [Install the RPM Package](#2-install-the-rpm-package)
3. [Configure the Apache Agent](#3-configure-the-apache-agent)
4. [Configure the Tomcat Agent](#4-configure-the-tomcat-agent)
5. [Connectivity Test](#5-connectivity-test)
6. [Run Initial Enrollment](#6-run-initial-enrollment)
7. [Enable Automatic Renewal](#7-enable-automatic-renewal)
8. [Operations and Troubleshooting](#8-operations-and-troubleshooting)
9. [UC Certificate Installer (OpenScape UC Application V11)](#9-uc-certificate-installer-openscape-uc-application-v11)
10. [Uninstallation](#10-uninstallation)

---

## 1. System Requirements

| Requirement | Minimum Version |
| --- | --- |
| Operating system | SLES 15 SP4 / openSUSE Leap 15.4 / RHEL 9 / Rocky 9 |
| Python | 3.11 or newer |
| Network | HTTPS access to the PKITNEXT server |
| Permissions | `root` or `sudo` for installation and Apache/Tomcat access |

Check your Python version:

```bash
python3 --version
```

---

## 2. Install the RPM Package

### 2.1 Obtain the RPM package

You can obtain the finished RPM package from PKITNEXT support or via the
PKITNEXT customer portal. Copy the file to the target system, for example:

```bash
scp pkitnext-agent-<version>-1.<dist>.x86_64.rpm user@target-system:/tmp/
```

### 2.2 Install the RPM

```bash
# SuSE
zypper install ~/rpmbuild/RPMS/x86_64/pkitnext-agent-*.x86_64.rpm

# RHEL / Rocky
dnf install ~/rpmbuild/RPMS/x86_64/pkitnext-agent-*.x86_64.rpm
```

### 2.3 Verify the installation

```bash
pkitnext-agent --help
pkitnext-tomcat-agent --help
```

---

## 3. Configure the Apache Agent

### 3.1 Create the configuration file

```bash
cp /opt/pkitnext-agent/examples/pkitnext-agent.yaml /etc/pkitnext-agent/agent.yaml
chmod 600 /etc/pkitnext-agent/agent.yaml
```

### 3.2 Update the configuration

```bash
vi /etc/pkitnext-agent/agent.yaml
```

The following values must be customized:

```yaml
agent:
  instance_name: webserver-prod          # Unique name of this instance
  log_file: /var/log/pkitnext-agent/agent.log

apache:
  service_name: apache2                  # or: httpd (RHEL)
  configtest_command: apachectl configtest
  reload_command: systemctl reload apache2

certificate:
  cert_path: /etc/apache2/ssl/cert.pem   # Path to the SSL certificate
  key_path:  /etc/apache2/ssl/key.pem    # Path to the private key
  backup_directory: /etc/apache2/ssl/backup
  # key_passphrase_file: /etc/apache2/ssl/key.passphrase
  # Optional: passphrase protection for the private key (mode b).
  # Created automatically on first enrollment (0600).

subject:
  common_name: webserver.your-domain.com # Server FQDN
  organization: Your Company LLC
  country: DE
  san:
    - webserver.your-domain.com          # Subject Alternative Names

renewal:
  renew_before_days: 30                  # Renew 30 days before expiry
  key_size: 2048

scep:
  url: https://your-pkitnext-server/scep/api/<config_name>/
  verify_tls: true
  timeout_seconds: 30
  # challenge_password: secret           # Only if required by the CA
```

> Note: You receive the SCEP URL and `config_name` from your PKITNEXT administrator.

### 3.3 Create target directories

```bash
mkdir -p /etc/apache2/ssl/backup
chown root:root /etc/apache2/ssl
chmod 750 /etc/apache2/ssl
```

---

## 4. Configure the Tomcat Agent

If you do not run Tomcat, skip this section.

### 4.1 Create the configuration file

```bash
cp /opt/pkitnext-agent/examples/pkitnext-tomcat-agent.yaml \
   /etc/pkitnext-agent/tomcat-agent.yaml
chmod 600 /etc/pkitnext-agent/tomcat-agent.yaml
```

### 4.2 Update the configuration

```bash
vi /etc/pkitnext-agent/tomcat-agent.yaml
```

The following values must be customized:

```yaml
agent:
  instance_name: tomcat-prod
  log_path: /var/log/pkitnext-agent/tomcat-agent.log

tomcat:
  service_name: tomcat9                  # or tomcat10
  restart_command: systemctl restart tomcat9
  keystore_path: /etc/tomcat9/ssl/keystore.p12
  keystore_password: changeit            # Keystore password (min. 6 chars)
  keystore_alias: tomcat

certificate:
  cert_path:  /etc/tomcat9/ssl/cert.pem
  key_path:   /etc/tomcat9/ssl/key.pem
  chain_path: /etc/tomcat9/ssl/chain.pem
  backup_directory: /etc/tomcat9/ssl/backup
  # key_passphrase_file: /etc/tomcat9/ssl/key.passphrase
  # Optional: passphrase protection for the PEM key file (mode b).
  # Does not apply to the PKCS#12 keystore (uses keystore_password).

subject:
  common_name: tomcat.your-domain.com
  san:
    - tomcat.your-domain.com

scep:
  url: https://your-pkitnext-server/scep/api/<config_name>/
  verify_tls: true
  timeout_seconds: 30
```

### 4.3 Update Tomcat server.xml

In Tomcat `server.xml`, configure the keystore path:

```xml
<!-- Tomcat 9+ with SSLHostConfig (recommended) -->
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           SSLEnabled="true">
  <SSLHostConfig>
    <Certificate certificateKeystoreFile="/etc/tomcat9/ssl/keystore.p12"
                 certificateKeystorePassword="changeit"
                 certificateKeystoreType="PKCS12" />
  </SSLHostConfig>
</Connector>
```

### 4.4 Create target directories

```bash
mkdir -p /etc/tomcat9/ssl/backup
chown tomcat:tomcat /etc/tomcat9/ssl
chmod 750 /etc/tomcat9/ssl
```

---

## 5. Connectivity Test

Before initial enrollment, test the SCEP connection. This fetches a test
certificate, but installs nothing.

Apache Agent:

```bash
pkitnext-agent test-scep --config /etc/pkitnext-agent/agent.yaml
```

Tomcat Agent:

```bash
pkitnext-tomcat-agent test-scep --config /etc/pkitnext-agent/tomcat-agent.yaml
```

Expected output (excerpt):

```text
[1/3] GetCACaps ...
      POSTPKIOperation
      SHA-256
      AES

[2/3] GetCACert ...
      Subject    : CN=PKITNEXT SYSTEMCA,...
      Valid until: 2031-10-27 19:22:01 UTC

[3/3] PKCSReq (Enrollment Test) ...
      Result     : SUCCESS
      Valid until: 2026-08-12 10:00:00 UTC

Test completed with status: SUCCESS (Exit code 0)
```

If the test returns an error:

| Error message | Cause | Fix |
| --- | --- | --- |
| `Connection refused` | SCEP server is unreachable | Check network / firewall |
| `SSL: CERTIFICATE_VERIFY_FAILED` | TLS certificate is not trusted | Import CA cert or set `verify_tls: false` |
| `FAILURE / PENDING` | Challenge password missing or wrong | Set `challenge_password` in config |
| `Timeout` | Server overloaded or URL is wrong | Check URL and `timeout_seconds` |

---

## 6. Run Initial Enrollment

> Warning: Initial enrollment briefly stops Apache / Tomcat, installs the new
> certificate, and restarts the service. Plan a short maintenance window.

Apache Agent:

```bash
pkitnext-agent enroll --config /etc/pkitnext-agent/agent.yaml
```

With one-time passphrase (mode a - not stored):

```bash
pkitnext-agent enroll --config /etc/pkitnext-agent/agent.yaml \
    --key-passphrase "my-passphrase"
```

Tomcat Agent:

```bash
pkitnext-tomcat-agent enroll --config /etc/pkitnext-agent/tomcat-agent.yaml
```

Successful output ends with:

```text
Certificate installed successfully.
Apache/Tomcat restarted.
```

Verify certificate:

```bash
# Apache
openssl x509 -in /etc/apache2/ssl/cert.pem -noout -subject -enddate

# Tomcat
keytool -list -v -keystore /etc/tomcat9/ssl/keystore.p12 \
        -storetype PKCS12 -storepass changeit | grep -E '(Alias|Valid)'
```

---

## 7. Enable Automatic Renewal

The agent renews certificates automatically once per day when the configured
window (`renew_before_days`) is reached.

Apache Agent:

```bash
systemctl enable --now pkitnext-agent.timer
systemctl status pkitnext-agent.timer
```

Tomcat Agent:

```bash
systemctl enable --now pkitnext-tomcat-agent.timer
systemctl status pkitnext-tomcat-agent.timer
```

Show next scheduled run:

```bash
systemctl list-timers pkitnext*
```

---

## 8. Operations and Troubleshooting

### Check status

```bash
pkitnext-agent        status --config /etc/pkitnext-agent/agent.yaml
pkitnext-tomcat-agent status --config /etc/pkitnext-agent/tomcat-agent.yaml
```

### Logs

```bash
# Real-time log
tail -f /var/log/pkitnext-agent/agent.log
tail -f /var/log/pkitnext-agent/tomcat-agent.log

# systemd journal
journalctl -u pkitnext-agent.service -f
journalctl -u pkitnext-tomcat-agent.service -f
```

### Force manual renewal

```bash
# Renew only if due
pkitnext-agent renew --config /etc/pkitnext-agent/agent.yaml

# Force renewal immediately (start timer service manually)
systemctl start pkitnext-agent.service
```

### Backup and rollback

The agent creates a backup before each renewal:

```text
/etc/apache2/ssl/backup/cert.pem.<timestamp>
/etc/apache2/ssl/backup/key.pem.<timestamp>
```

If an error occurs, rollback is automatic. There is no dedicated `rollback`
command; the agent restores backup files automatically on `enroll` or `renew`
failures.

### System check

```bash
# Check certificate, permissions, SCEP reachability, and Apache
pkitnext-agent check --config /etc/pkitnext-agent/agent.yaml
```

---

## 9. UC Certificate Installer (OpenScape UC Application V11)

For Mitel / Unify OpenScape UC Application V11, the RPM additionally contains
`uc_cert_installer.sh`. It automates UC-specific certificate renewal steps
(PKCS#12 keystore build, config patching, service restart, rollback) and
requires `pkitnext-agent` for the SCEP part.

```bash
# Initial setup (interactive)
uc_cert_installer.sh --init \
    --mode application_server \
    --scep-config /etc/pkitnext-agent/agent.yaml

# Simulation run (no system changes, real SCEP request)
uc_cert_installer.sh --simulate-all

# Production run
uc_cert_installer.sh

# Log
tail -f /var/log/cert_agent/uc_cert_installer.log
```

Full usage guide: [scripts/README.md](scripts/README.md)

---

## 10. Uninstallation

```bash
# Stop timers and services
systemctl disable --now pkitnext-agent.timer pkitnext-tomcat-agent.timer
systemctl disable --now uc-cert-installer.timer

# Remove RPM
# SuSE
zypper remove pkitnext-agent

# RHEL / Rocky
dnf remove pkitnext-agent
```

Configuration files and logs remain and must be removed manually:

```bash
rm -rf /etc/pkitnext-agent/
rm -rf /etc/pkitnext/
rm -rf /var/log/pkitnext-agent/
rm -rf /var/log/cert_agent/
rm -rf /var/lib/pkitnext-agent/
```

---

*PKITNEXT Labs - [support@pkitnext.de](mailto:support@pkitnext.de)*
