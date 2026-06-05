# UC Certificate Installer – Nutzungsanleitung

`uc_cert_installer.sh` automatisiert die vollständige Zertifikatserneuerung für  
**Mitel / Unify OpenScape UC Application V11** (Kapitel 10 – Configuring a Certificate Strategy).

---

## Inhaltsverzeichnis

1. [Voraussetzungen](#1-voraussetzungen)
2. [Installation des Scripts](#2-installation-des-scripts)
3. [Konfiguration](#3-konfiguration)
4. [Aufruf](#4-aufruf)
5. [Install-Modi](#5-install-modi)
6. [Service-Neustart-Modi](#6-service-neustart-modi)
7. [Ablauf im Detail](#7-ablauf-im-detail)
8. [Logging und Reports](#8-logging-und-reports)
9. [Rollback](#9-rollback)
10. [Systemd-Timer / Cron](#10-systemd-timer--cron)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Voraussetzungen

| Anforderung | Details |
|---|---|
| Betriebssystem | Linux (SLES / RHEL / Debian) |
| Ausführung als | `root` |
| Shell | bash ≥ 4.0 |
| Tools | `openssl`, `curl`, optional: `kubectl` |
| Python-Paket | `pkitnext-agent` (SCEP-Enrollment) |
| OpenScape | UC Application V11, läuft auf dem Zielserver |

```bash
# Verfügbarkeit prüfen
openssl version
pkitnext-agent --version
curl --version
```

---

## 2. Installation des Scripts

```bash
# Script ablegen (ausführbar machen)
install -m 755 -o root -g root \
    scripts/uc_cert_installer.sh \
    /usr/local/sbin/uc_cert_installer.sh

# Beispiel-Config kopieren und anpassen
install -m 600 -o root -g root \
    examples/uc_cert_installer.conf \
    /etc/pkitnext/uc_cert_installer.conf

# Log-Verzeichnis anlegen
mkdir -p /var/log/cert_agent
```

---

## 3. Konfiguration

Alle Einstellungen werden in `/etc/pkitnext/uc_cert_installer.conf` gesetzt  
(oder als Umgebungsvariablen übergeben).

### Minimale Pflichteinstellungen

```bash
# /etc/pkitnext/uc_cert_installer.conf

# Install-Modus (s. Abschnitt 5)
INSTALL_MODE="application_server"

# PKCS#12 Keystore-Passwort – MUSS gesetzt werden
# Key- und Keystore-Passwort müssen identisch sein (Kapitel 10 Anforderung)
KEYSTORE_PASSWORD="Ihr_sicheres_Passwort_123!"

# SCEP-Agent Konfigdatei
PKITNEXT_AGENT_CONFIG="/etc/pkitnext-agent.yaml"

# URL für Konnektivitätstest nach Neustart
TEST_URL="https://myhost.example.com:8443/"
```

### Alle konfigurierbaren Parameter

| Variable | Standard | Beschreibung |
|---|---|---|
| `INSTALL_MODE` | `application_server` | Zielkomponente (s. Abschnitt 5) |
| `KEYSTORE_PASSWORD` | – | **Pflicht.** PKCS#12-Passwort |
| `KEYSTORE_ALIAS` | `tomcat` | Alias im Keystore |
| `SYMPHONIA_HOME` | (auto) | OpenScape-Installationspfad |
| `CA_DIST` | (auto) | HiPathCA Distribution-Verzeichnis |
| `CA_CONFIG_HTTPD` | (auto) | SSL-Config-Verzeichnis |
| `PKITNEXT_AGENT_CMD` | `pkitnext-agent` | SCEP-Agent-Befehl |
| `PKITNEXT_AGENT_CONFIG` | `/etc/pkitnext-agent.yaml` | Agent-Konfiguration |
| `PKITNEXT_SKIP_RENEWAL` | `false` | SCEP-Schritt überspringen |
| `PEM_CERT` | `/etc/pkitnext/certs/server.crt` | Zertifikat-PEM |
| `PEM_KEY` | `/etc/pkitnext/certs/server.key` | Key-PEM |
| `PEM_CHAIN` | `/etc/pkitnext/certs/chain.crt` | CA-Chain-PEM (optional) |
| `BACKUP_BASE_DIR` | `/etc/pkitnext/backup/uc_installer` | Backup-Verzeichnis |
| `SERVICE_RESTART_MODE` | `init_d_symphoniad` | Neustart-Mechanismus (s. Abschnitt 6) |
| `TEST_URL` | – | HTTPS-URL für Post-Test |
| `TEST_TIMEOUT` | `15` | Timeout in Sekunden |
| `TEST_RETRIES` | `3` | Anzahl Wiederholungen |
| `TEST_SKIP_TLS_VERIFY` | `true` | TLS-Verifizierung beim Test |
| `LOG_FILE` | `/var/log/cert_agent/uc_cert_installer.log` | Log-Datei |
| `LOG_LEVEL` | `INFO` | `DEBUG` / `INFO` / `WARN` / `ERROR` |

---

## 4. Aufruf

```
Verwendung:
  uc_cert_installer.sh [OPTIONS]            Zertifikat erneuern und installieren
  uc_cert_installer.sh --init [INIT-FLAGS]  Konfigurationsdatei generieren

Haupt-Optionen:
  -c, --config FILE         Konfigurationsdatei
                            (Standard: /etc/pkitnext/uc_cert_installer.conf)
  -m, --mode MODE           Install-Modus: application_server | facade_server | spml
  -n, --dry-run             Planung ohne Änderungen
  -v, --verbose             Debug-Ausgabe
  -h, --help                Hilfe anzeigen

--init Flags:
  --mode MODE               Install-Modus
  --password-file FILE      Keystore-Passwort aus Datei lesen (sicherer als direkte Eingabe)
  --test-url URL            URL für HTTPS-Konnektivitätstest
  --restart-mode MODE       Neustart-Modus
  --scep-config FILE        Pfad zur pkitnext-agent Konfigdatei
  --symphoniad-service SVC  Symphoniad-Service-Name
  --k8s-namespace NS        Kubernetes Namespace
  --k8s-pod-label LABEL     Kubernetes Pod-Label-Selector
  --skip-renewal            SCEP-Schritt überspringen
  --log-file FILE           Log-Datei
  --force                   Bestehende Config überschreiben
```

### Config generieren mit `--init`

`--init` erzeugt die `/etc/pkitnext/uc_cert_installer.conf` automatisch und setzt `chmod 600`.  
Das Passwort wird **nie als CLI-Argument** übergeben (nicht in `ps aux` sichtbar).

#### Option A: Passwort aus Datei (für Ansible / Cloud-Init / CI)

```bash
# Passwort-Datei vorbereiten
echo -n "MeinSicheresPasswort!" > /run/secrets/ks-password
chmod 600 /run/secrets/ks-password

# Config generieren – vollautomatisch, keine Eingabeaufforderung
uc_cert_installer.sh --init \
    --mode facade_server \
    --password-file /run/secrets/ks-password \
    --test-url https://facade.example.com:8443/ \
    --restart-mode catalinad
```

#### Option B: Interaktive Eingabe (für manuelle Installation)

```bash
# Ohne --password-file: sicherer Prompt (Passwort wird nicht angezeigt)
uc_cert_installer.sh --init \
    --mode application_server \
    --test-url https://myhost.example.com:8443/ \
    --restart-mode init_d_symphoniad
# → Passwort eingeben: ****
# → Passwort bestätigen: ****
# → Config erstellt: /etc/pkitnext/uc_cert_installer.conf
```

#### Vollständiger Workflow nach RPM-Installation

```bash
# 1. Config generieren
uc_cert_installer.sh --init \
    --mode application_server \
    --password-file /run/secrets/ks-password \
    --test-url https://myhost.example.com:8443/ \
    --restart-mode init_d_symphoniad \
    --scep-config /etc/pkitnext-agent/agent.yaml

# 2. Dry-run prüfen
uc_cert_installer.sh -n -v

# 3. Install ausführen
uc_cert_installer.sh

# 4. Timer aktivieren (monatliche Erneuerung)
systemctl enable --now uc-cert-installer.timer
```

#### Bestehende Config überschreiben

```bash
uc_cert_installer.sh --init --force \
    --mode facade_server \
    --password-file /run/secrets/ks-password \
    --test-url https://new-host.example.com:8443/
```

### Weitere Aufruf-Beispiele

```bash
# SPML-Responder
uc_cert_installer.sh --init --mode spml --password-file /run/secrets/ks-password
uc_cert_installer.sh

# SCEP-Schritt überspringen (PEM-Dateien bereits aktuell)
PKITNEXT_SKIP_RENEWAL=true uc_cert_installer.sh

# Explizite Config-Datei angeben
uc_cert_installer.sh -c /etc/pkitnext/facade_installer.conf
```

---

## 5. Install-Modi

### `application_server` (Standard)

Für den **Anwendungsserver** (Application Computer) der OpenScape UC Application.

```
Quelle:  PEM cert + key + chain  (von pkitnext-agent)
         ↓
Aufbau:  PKCS#12  →  ${SYMPHONIA_HOME}/common/conf/tomcat-keystore.p12
                  →  ${CA_DIST}/.keystore  (Owner: sym:sym)
Config:  ${CA_CONFIG_HTTPD}/Portal/ssl.cfg   ← keystorePass aktualisieren
         ${CA_CONFIG_HTTPD}/CAMgmt/ssl.cfg   ← keystorePass aktualisieren
Restart: symphoniad restart WebClient_BE
```

**Minimale Config:**
```bash
INSTALL_MODE="application_server"
KEYSTORE_PASSWORD="..."
SERVICE_RESTART_MODE="init_d_symphoniad"
SYMPHONIAD_SERVICE="WebClient_BE"
TEST_URL="https://myhost.example.com:8443/"
```

---

### `facade_server`

Für den **Facade-Server** (OpenScape Mobile Client / Tomcat).

```
Quelle:  PEM cert + key + chain
         ↓
Aufbau:  PKCS#12  →  /usr/local/tomcat6/conf/OSC-Facade-Server-Default-Cert.p12
Config:  /usr/local/tomcat6/conf/server.xml  ← keystorePass aktualisieren
Restart: /etc/init.d/catalinad stop && start
```

**Minimale Config:**
```bash
INSTALL_MODE="facade_server"
KEYSTORE_ALIAS="tomcat"          # Pflicht für Facade Server (Kapitel 10)
KEYSTORE_PASSWORD="..."
SERVICE_RESTART_MODE="catalinad"
TEST_URL="https://facade.example.com:8443/"
```

---

### `spml`

Für den **SPML-Responder** (TomcatServletContainer).

```
Quelle:  PEM cert + key + chain
         ↓
Aufbau:  PKCS#12  →  ${SYMPHONIA_HOME}/share/tomcat/conf/spmlresponder_keystore.p12
                     (Owner: sym:sym, Permissions: 644)
Config:  server.xml  ← keystorePass aktualisieren
Restart: symphoniad restart TomcatServletContainer
```

**Minimale Config:**
```bash
INSTALL_MODE="spml"
KEYSTORE_PASSWORD="..."
SERVICE_RESTART_MODE="init_d_symphoniad"
SYMPHONIAD_SERVICE="TomcatServletContainer"
```

---

## 6. Service-Neustart-Modi

| Modus | Beschreibung | Wann verwenden |
|---|---|---|
| `init_d_symphoniad` | `/etc/init.d/symphoniad restart <SERVICE>` | Klassisches On-Premise |
| `kubernetes` | `kubectl exec ... kill 1` (Pod-Neustart) | k8s-Deployment |
| `catalinad` | `/etc/init.d/catalinad stop/start` | Facade Server |
| `none` | Kein automatischer Neustart | Manueller Neustart gewünscht |

### Kubernetes-Einstellungen

```bash
SERVICE_RESTART_MODE="kubernetes"
K8S_NAMESPACE="default"
K8S_POD_LABEL="app=ucowcbe"    # kubectl get pods -l app=ucowcbe
K8S_CONTAINER="ucowcbe"
```

---

## 7. Ablauf im Detail

Das Script durchläuft **10 Phasen** – bei Fehler in einer Phase wird automatisch  
ein Rollback ausgeführt:

```
Phase 1  SCEP Renewal     pkitnext-agent renew  →  neue PEM-Dateien
         ↓
Phase 2  Backup           Snapshot aller betroffenen Dateien
         ↓
Phase 3  Build P12        openssl pkcs12 -export  (cert/key-Match wird geprüft)
         ↓
Phase 4  Install          Keystore atomisch an Zielort kopieren
         ↓
Phase 5  Config-Update    ssl.cfg / server.xml  keystorePass patchen
         ↓
Phase 6  Pre-Test         SHA-256-Fingerprint-Vergleich cert ↔ P12
         ↓
Phase 7  Restart          Service neustarten (kubectl / symphoniad / catalinad)
         ↓
Phase 8  Post-Test        HTTPS-Konnektivitätstest (curl, Retry-Logik)
         ↓
Phase 9  [Rollback]       Nur bei Fehler: Backup zurückspielen + Neustart
         ↓
Phase 10 Report           Strukturierter Log-Eintrag mit Zertifikat-Daten
```

### Dry-run

Mit `-n` werden alle Phasen simuliert. Es werden **keine Dateien** verändert,  
kein Service neugestartet und kein SCEP-Request gesendet.

```bash
uc_cert_installer.sh -n -v 2>&1 | tee /tmp/dry_run_output.txt
```

---

## 8. Logging und Reports

### Log-Datei

Alle Ausgaben gehen sowohl auf stdout als auch in die Log-Datei  
(Standard: `/var/log/cert_agent/uc_cert_installer.log`).

```
[2026-05-14T10:15:23] [INFO ] ======================================
[2026-05-14T10:15:23] [INFO ] UC CERT INSTALLER – FINAL REPORT
[2026-05-14T10:15:23] [INFO ] ======================================
[2026-05-14T10:15:23] [INFO ] Script version  : 1.0.0
[2026-05-14T10:15:23] [INFO ] Run timestamp   : 20260514_101520
[2026-05-14T10:15:23] [INFO ] Host            : myhost.example.com
[2026-05-14T10:15:23] [INFO ] Install mode    : application_server
[2026-05-14T10:15:23] [INFO ] Status          : SUCCESS
[2026-05-14T10:15:23] [INFO ] Certificate subject : CN=myhost.example.com, OU=...
[2026-05-14T10:15:23] [INFO ] Certificate expiry  : May 14 10:15:00 2027 GMT
[2026-05-14T10:15:23] [INFO ] Keystore target     : /opt/siemens/common/conf/tomcat-keystore.p12
```

### Log-Rotation (logrotate)

```
# /etc/logrotate.d/pkitnext-cert-installer
/var/log/cert_agent/uc_cert_installer.log {
    monthly
    rotate 12
    compress
    missingok
    notifempty
}
```

---

## 9. Rollback

Der Rollback erfolgt **automatisch** bei Fehler in einer der Phasen:

1. Alle gesicherten Dateien werden aus dem Backup zurückgespielt
2. Der Service wird neu gestartet (mit der alten Konfiguration)
3. Ein Konnektivitätstest prüft, ob der Dienst wieder erreichbar ist

Der Backup-Pfad hat das Format:
```
/etc/pkitnext/backup/uc_installer/20260514_101520/
├── MANIFEST.txt
├── opt/siemens/common/conf/tomcat-keystore.p12
├── opt/siemens/HiPathCA/config/services/default/Httpd/Portal/ssl.cfg
└── etc/pkitnext/certs/server.crt
```

**Manueller Rollback** (falls das automatische Rollback fehlschlägt):

```bash
# Backup-Verzeichnis anzeigen
ls /etc/pkitnext/backup/uc_installer/

# Manuell wiederherstellen (Beispiel für application_server)
BACKUP="/etc/pkitnext/backup/uc_installer/20260514_101520"

cp "${BACKUP}/opt/siemens/common/conf/tomcat-keystore.p12" \
   /opt/siemens/common/conf/tomcat-keystore.p12

cp "${BACKUP}/opt/siemens/HiPathCA/distribution/.keystore" \
   /opt/siemens/HiPathCA/distribution/.keystore

cp "${BACKUP}/opt/siemens/HiPathCA/config/services/default/Httpd/Portal/ssl.cfg" \
   /opt/siemens/HiPathCA/config/services/default/Httpd/Portal/ssl.cfg

chown sym:sym /opt/siemens/HiPathCA/distribution/.keystore
/etc/init.d/symphoniad restart WebClient_BE
```

---

## 10. Systemd-Timer / Cron

### Vollständiger Setup-Workflow nach `rpm -ivh`

```bash
# 1. Config generieren (Passwort-Datei vorbereiten)
echo -n "MeinSicheresPasswort!" > /run/secrets/ks-password
chmod 600 /run/secrets/ks-password

uc_cert_installer.sh --init \
    --mode application_server \
    --password-file /run/secrets/ks-password \
    --test-url https://$(hostname -f):8443/ \
    --restart-mode init_d_symphoniad

# 2. Config prüfen
cat /etc/pkitnext/uc_cert_installer.conf

# 3. Dry-run
uc_cert_installer.sh -n -v

# 4. Manuellen Lauf testen
uc_cert_installer.sh

# 5. Monatlichen Timer aktivieren
systemctl enable --now uc-cert-installer.timer
systemctl status uc-cert-installer.timer
```

```cron
# /etc/cron.d/pkitnext-cert-installer
0 2 1 * * root /usr/local/sbin/uc_cert_installer.sh \
    -c /etc/pkitnext/uc_cert_installer.conf \
    >> /var/log/cert_agent/uc_cert_installer.log 2>&1
```

### systemd-Timer

```ini
# /etc/systemd/system/uc-cert-installer.service
[Unit]
Description=UC Application Certificate Installer (pkitnext-agent SCEP)
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/uc_cert_installer.sh -c /etc/pkitnext/uc_cert_installer.conf
StandardOutput=append:/var/log/cert_agent/uc_cert_installer.log
StandardError=append:/var/log/cert_agent/uc_cert_installer.log
```

```ini
# /etc/systemd/system/uc-cert-installer.timer
[Unit]
Description=UC Application Certificate Installer – monatlicher Trigger

[Timer]
OnCalendar=monthly
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now uc-cert-installer.timer
systemctl list-timers uc-cert-installer.timer
```

---

## 11. Troubleshooting

### Problem: `KEYSTORE_PASSWORD is not set`

```bash
# Passwort in der Config-Datei setzen (chmod 600 sicherstellen)
chmod 600 /etc/pkitnext/uc_cert_installer.conf
echo 'KEYSTORE_PASSWORD="MeinSicheresPasswort!"' >> /etc/pkitnext/uc_cert_installer.conf
```

### Problem: `Certificate and private key do NOT match`

Der pkitnext-agent hat neue PEM-Dateien geschrieben, aber `PEM_KEY` zeigt  
noch auf einen alten Key. Pfade in der Config prüfen:

```bash
# Cert und Key manuell prüfen
openssl x509 -noout -pubkey -in /etc/pkitnext/certs/server.crt | sha256sum
openssl pkey -in /etc/pkitnext/certs/server.key -pubout | sha256sum
# → beide Hashes müssen identisch sein
```

### Problem: `SYMPHONIA_HOME is required`

OpenScape-Sysconfig nicht gefunden. Manuell setzen:

```bash
# In der Config-Datei
SYMPHONIA_HOME="/opt/siemens"

# Oder: aus Sysconfig lesen
cat /etc/sysconfig/siemens/symphonia
```

### Problem: Konnektivitätstest schlägt fehl

```bash
# Manuell testen
curl -k -v https://myhost.example.com:8443/

# TLS-Zertifikat des Servers prüfen
openssl s_client -connect myhost.example.com:8443 -showcerts < /dev/null 2>/dev/null \
    | openssl x509 -noout -subject -enddate

# Dienststatus prüfen
/etc/init.d/symphoniad status WebClient_BE
```

### Problem: Rollback fehlgeschlagen

```bash
# Backup-Inhalt prüfen
ls -la /etc/pkitnext/backup/uc_installer/
cat /etc/pkitnext/backup/uc_installer/<TIMESTAMP>/MANIFEST.txt

# Manuell wiederherstellen (s. Abschnitt 9)
```

### Debug-Modus aktivieren

```bash
# Vollständige Ausgabe aller Befehle
uc_cert_installer.sh -v -n 2>&1 | tee /tmp/debug.log
```
