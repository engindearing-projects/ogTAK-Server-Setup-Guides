# TAK Server 5.7 Installation Guide for Ubuntu 24.04

**Complete step-by-step tutorial for installing TAK Server 5.7 on Ubuntu 24.04**

*Tested release tag: `5.7-RELEASE-14` (latest at time of writing — published 2026-04-03 on TAK-Product-Center/Server). Filename in this guide assumes `RELEASE-14`; substitute whatever build you downloaded.*

> **Migrating from 5.6?** Same OS prerequisites, same install flow. The big news in 5.7 is server-side messaging performance and a much faster WebTAK — see [What's new in 5.7](#whats-new-in-57).
>
> **Migrating from 5.5?** Read the 5.6 guide's delta first, then this one. Or just follow this guide top-to-bottom — it's the cleanest path.

---

## What's new in 5.7

Captured from the official TAK Product Center 5.7 release notes (5.7-RELEASE-8) plus source diff vs `5.6-RELEASE-35`:

### WebTAK
- **WebTAK bumped to 4.10.5**
- **Performance**: client-side bottlenecks above ~500 connected clients are gone. The TPC tested up to **2800 simultaneous clients on a c5.2xlarge (8 vCPU)** server. If you have a busy server, this is the upgrade.

### Messaging
- **takproto over UDP inputs** — UDP inputs now accept binary takproto messages (previously TCP/TLS only)
- **Optional broadcast filter** — server admins can prevent specific users from broadcasting map items (added to `CoreConfig.xml`)
- **Websocket compression off by default** — compression added latency at scale; you can re-enable it in `CoreConfig.xml` if you need it for low-bandwidth clients

### What's not changed (carries over from 5.5/5.6)
- **OS deps**: still Java 17 + PostgreSQL 15 + PostGIS 3
- **`setenv.sh` JDK opens**: identical
- **Federation Hub**: still a separate `.deb`, same scripts and ports
- **Cert generation**: same `makeRootCa.sh` / `makeCert.sh` flow
- **Database schema**: handled by `SchemaManager.jar upgrade` — works in-place from 5.5 or 5.6

If you upgrade in-place from 5.6, the only careful step is re-applying the script-path fixes (Section 8) after `dpkg -i` overwrites the launcher scripts.

---

## IMPORTANT GOTCHAS - READ FIRST!

The original gotchas from 5.5 still apply on 5.7. None of these have been fixed upstream because they're really Ubuntu 24.04 + tak.gov packaging quirks, not TAK Server bugs.

### PostgreSQL Version: MUST Use Version 15

- TAK Server 5.7 still pins to PostgreSQL 15 (verified in `takserver-package/database/build.gradle` at tag `5.7-RELEASE-14`: `requires('postgresql-15')`, `requires('postgresql-15-postgis-3')`)
- Ubuntu 24.04's default `apt install postgresql` installs 16 — that **will** break TAK Server 5.7
- Install 15 explicitly from the PGDG repo (steps below)

### Java: OpenJDK 17 is REQUIRED

- The `.deb` declares `requires('openjdk-17-jdk')` — keep it installed
- Run Temurin 17 as the runtime AND keep OpenJDK 17 to satisfy the `.deb` dependency
- Same dual-install pattern as 5.5 and 5.6

### Federation Hub is a Separate Package

- `takserver-fed-hub_5.7-RELEASE14_all.deb` — separate download from tak.gov
- Federation Hub scripts still need PATH fixes for Ubuntu 24.04
- See [FEDERATION_SETUP.md](FEDERATION_SETUP.md) — same procedure, just bump the filename

### New for 5.7: Websocket compression default flipped

- WebTAK clients used to negotiate per-message-deflate by default — now they don't
- If you have a custom proxy/CDN in front of TAK Server expecting compressed websocket frames, plan for uncompressed traffic by default
- To re-enable, find the `<websocket>` element in `CoreConfig.xml` and set `compressionEnabled="true"` (verify the exact attribute name in your installed `CoreConfig.example.xml` — XSD has the authoritative list)

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Java Installation (Temurin 17 + OpenJDK 17)](#java-installation)
3. [PostgreSQL 15 Installation](#postgresql-15-installation)
4. [TAK Server Installation](#tak-server-installation)
5. [Database Configuration](#database-configuration)
6. [Certificate Generation](#certificate-generation)
7. [CoreConfig.xml Configuration](#coreconfig-configuration)
8. [Fix Startup Scripts](#fix-startup-scripts)
9. [Firewall Configuration](#firewall-configuration)
10. [Starting and Testing TAK Server](#starting-and-testing)
11. [Accessing TAK Server](#accessing-tak-server)
12. [Verify with OmniTAK](#verify-with-omnitak)
13. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Ubuntu 24.04 LTS with sudo access
- Minimum 8GB RAM (16GB+ for production)
- 40GB+ disk space
- Internet connection
- TAK Server 5.7 `.deb` from [tak.gov](https://tak.gov) — registration required
  - Filename: `takserver_5.7-RELEASE14_all.deb` (or current 5.7 RELEASE-N)

> If you plan to use 5.7's WebTAK at scale (1000+ clients), spec your VM at **8 vCPU / 16GB RAM minimum** — the AWS c5.2xlarge benchmark is the published reference for 2800 clients.

---

## Java Installation

TAK Server 5.7 requires Java 17. Install BOTH Temurin 17 (runtime) AND OpenJDK 17 (`.deb` dependency).

### 1. Remove Java 21 if present (DO NOT remove Java 17)

```bash
sudo apt-get remove -y openjdk-21-jdk openjdk-21-jre openjdk-21-jdk-headless openjdk-21-jre-headless
sudo apt-get autoremove -y
```

### 2. Install Temurin Java 17

```bash
sudo apt-get update
sudo apt-get install -y wget apt-transport-https gpg

wget -qO - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo gpg --dearmor -o /usr/share/keyrings/adoptium-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/adoptium-archive-keyring.gpg] https://packages.adoptium.net/artifactory/deb noble main" | sudo tee /etc/apt/sources.list.d/adoptium.list

sudo apt-get update
sudo apt-get install -y temurin-17-jdk
```

### 3. Set JAVA_HOME

```bash
echo 'export JAVA_HOME=/usr/lib/jvm/temurin-17-jdk-amd64' | sudo tee -a /etc/environment
echo 'export PATH=$JAVA_HOME/bin:$PATH' | sudo tee -a /etc/environment
source /etc/environment
```

### 4. Verify

```bash
java -version
echo $JAVA_HOME
```

### 5. Install OpenJDK 17 (required for `.deb` dependency)

```bash
sudo apt-get install -y openjdk-17-jdk openjdk-17-jre
```

---

## PostgreSQL 15 Installation

### 1. Add PostgreSQL APT Repository

```bash
sudo apt-get install -y curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

### 2. Install PostgreSQL 15 + PostGIS 3

```bash
sudo apt-get update
sudo apt-get install -y postgresql-15 postgresql-client-15 postgresql-contrib-15 postgresql-15-postgis-3
```

### 3. Pin port 5432

```bash
sudo sed -i 's/port = 5433/port = 5432/' /etc/postgresql/15/main/postgresql.conf
sudo systemctl restart postgresql@15-main
```

### 4. Verify

```bash
psql --version    # MUST show 15.x
sudo systemctl status postgresql@15-main
sudo -u postgres psql -c "\l"
```

---

## TAK Server Installation

### 1. Install the 5.7 `.deb`

```bash
cd ~/Downloads
sudo dpkg --force-depends -i takserver_5.7-RELEASE14_all.deb
```

### 2. Verify

```bash
ls -la /opt/tak/
```

You should see `CoreConfig.example.xml`, `takserver.sh`, `setenv.sh`, `certs/`, `db-utils/`, etc.

---

## Database Configuration

### 1. Initialize CoreConfig

```bash
cd /opt/tak
sudo cp CoreConfig.example.xml CoreConfig.xml
```

### 2. Set DB password and host

```bash
sudo sed -i 's/password=""/password="takserver123"/' /opt/tak/CoreConfig.xml
sudo sed -i 's/tak-database/localhost/' /opt/tak/CoreConfig.xml
```

> **Production:** use a strong password and update `pg_hba.conf` accordingly.

### 3. Verify

```bash
sudo grep "connection url" /opt/tak/CoreConfig.xml
```

Expected: `<connection url="jdbc:postgresql://localhost:5432/cot" username="martiuser" password="takserver123" />`

### 4. Create DB role + database

```bash
sudo -u postgres psql -c "CREATE ROLE martiuser LOGIN ENCRYPTED PASSWORD 'takserver123' SUPERUSER INHERIT CREATEDB NOCREATEROLE;"
sudo -u postgres psql -c "CREATE DATABASE cot OWNER martiuser;"
```

### 5. Apply schema

```bash
cd /opt/tak/db-utils
sudo java -jar SchemaManager.jar upgrade
```

Expected: `TAK server database schema is up to date.`

> **Upgrading in place from 5.5 or 5.6?** This same `SchemaManager.jar upgrade` command handles the migration. Take a `pg_dump` first.

---

## Certificate Generation

### 1. Configure metadata

```bash
cd /opt/tak/certs
sudo sed -i 's/STATE=${STATE}/STATE=YourState/' /opt/tak/certs/cert-metadata.sh
sudo sed -i 's/CITY=${CITY}/CITY=YourCity/' /opt/tak/certs/cert-metadata.sh
sudo sed -i 's/ORGANIZATIONAL_UNIT=${ORGANIZATIONAL_UNIT}/ORGANIZATIONAL_UNIT=TAKServer/' /opt/tak/certs/cert-metadata.sh
```

### 2. Root CA

```bash
cd /opt/tak/certs
sudo ./makeRootCa.sh
```

Pick a unique CA name (e.g., `MyTAKServer57`).

### 3. Server cert

```bash
sudo ./makeCert.sh server takserver
```

### 4. Admin + user client certs

```bash
sudo ./makeCert.sh client admin
sudo ./makeCert.sh client user1
```

### 5. Verify

```bash
ls -lh /opt/tak/certs/files/
```

---

## CoreConfig Configuration

### 1. Multicast SA discovery

```bash
sudo sed -i 's/<!-- <input _name="SAproxy" protocol="mcast" group="239.2.3.1" port="6969" proxy="true" auth="anonymous" \/> -->/<input _name="SAproxy" protocol="mcast" group="239.2.3.1" port="6969" proxy="true" auth="anonymous" \/>/' /opt/tak/CoreConfig.xml
```

### 2. Server announcement (optional)

```bash
SERVER_IP=$(hostname -I | awk '{print $1}')
sudo sed -i "s/ip=\"192.168.1.1\"/ip=\"$SERVER_IP\"/" /opt/tak/CoreConfig.xml
sudo sed -i 's/<!--<announce enable="true" uid="Marti1"/<announce enable="true" uid="TakServer1"/' /opt/tak/CoreConfig.xml
sudo sed -i 's/<\/announce>-->/<\/announce>/' /opt/tak/CoreConfig.xml
```

### 3. (Optional, 5.7-specific) Re-enable websocket compression

If you serve clients on constrained links and want the old behavior back:

```bash
# Inspect the example config to find the right attribute for your build
grep -A2 -i "websocket" /opt/tak/CoreConfig.example.xml
# Then edit /opt/tak/CoreConfig.xml accordingly
```

The XSD at `/opt/tak/TAKIgniteConfig.example.xml` and the schemas under `/opt/tak/docs/` are the authoritative reference for available attributes.

### 4. (Optional, 5.7-specific) takproto over UDP

To accept binary takproto messages on a UDP input, add a UDP `<input>` block with `protocol="udp"` and `proto="takproto"` (verify the attribute name in your installed `CoreConfig.example.xml`). This is most useful for low-power radio gateways that prefer UDP.

### 5. Verify

```bash
grep -E "(SAproxy|announce)" /opt/tak/CoreConfig.xml
```

---

## Fix Startup Scripts

Same fixes as 5.5/5.6 — Ubuntu 24.04 needs absolute paths in the systemd-launched scripts:

```bash
sudo sed -i 's|TOTALRAMBYTES=`awk|TOTALRAMBYTES=`/usr/bin/awk|' /opt/tak/setenv.sh
sudo sed -i 's|^rm -rf|/usr/bin/rm -rf|' /opt/tak/takserver-messaging.sh
sudo sed -i 's|^java |/usr/bin/java |' /opt/tak/takserver-messaging.sh
sudo sed -i 's|^java |/usr/bin/java |' /opt/tak/takserver-api.sh
sudo sed -i 's|^java |/usr/bin/java |' /opt/tak/takserver-config.sh
```

Create the Ignite config:

```bash
sudo cp /opt/tak/TAKIgniteConfig.example.xml /opt/tak/TAKIgniteConfig.xml
```

> **Re-apply this every time you install or reinstall the `.deb`** — `dpkg` overwrites these scripts.

---

## Firewall Configuration

```bash
sudo ufw allow 8089/tcp comment 'TAK Server TLS'
sudo ufw allow 8443/tcp comment 'TAK Server HTTPS'
sudo ufw allow 8444/tcp comment 'TAK Server Federation HTTPS'
sudo ufw allow 8446/tcp comment 'TAK Server Cert Enrollment HTTPS'
sudo ufw allow 6969/udp comment 'TAK Server Multicast'
sudo ufw status
```

If you're enabling takproto over UDP for a custom port, open that too.

---

## Starting and Testing

```bash
sudo systemctl enable takserver
sudo systemctl start takserver

sudo systemctl status takserver
sudo systemctl status takserver-messaging
sudo systemctl status takserver-api
sudo systemctl status takserver-config

ps aux | grep "tak.server" | grep -v grep
sudo netstat -plnt | grep -E "(8089|8443|8444|8446)"
curl -k https://localhost:8446/
```

---

## Accessing TAK Server

| URL | Purpose |
|-----|---------|
| `https://YOUR_SERVER_IP:8443` | Web admin / WebTAK 4.10.5 (requires client cert) |
| `https://YOUR_SERVER_IP:8446` | Cert enrollment |
| `https://YOUR_SERVER_IP:8444` | Federation |

Copy admin cert:

```bash
sudo cp /opt/tak/certs/files/admin.p12 ~/admin.p12
sudo chown $USER:$USER ~/admin.p12
```

Default cert password: `atakatak` — change it for production.

---

## Verify with OmniTAK

See [OMNITAK_TEST_GUIDE.md](OMNITAK_TEST_GUIDE.md). Quick version:

1. `sudo /opt/tak/certs/makeCert.sh client engie-test`
2. Copy `/opt/tak/certs/files/engie-test.p12` to your phone
3. OmniTAK-Android / OmniTAK-iOS → **Settings → TAK Servers → +**, IP + port `8089`, SSL/TLS, attach `.p12` (password `atakatak`)
4. Drop a marker; verify it appears in the WebTAK UI at `https://YOUR_SERVER_IP:8443`

For 5.7 specifically, also try the takproto-UDP input if you configured one — point a client (or `nc`) at the new UDP port and confirm the messaging service logs the inbound CoT.

---

## Troubleshooting

### Service won't start

```bash
cat /opt/tak/logs/takserver-messaging-console.log
cat /opt/tak/logs/takserver-api-console.log
cat /opt/tak/logs/takserver-config-console.log
```

### `java: not found` / `awk: not found`

Re-run [Fix Startup Scripts](#fix-startup-scripts).

### WebTAK feels slow even on 5.7

- Confirm you're actually on 5.7: check `/opt/tak/version.txt` or the bottom-right of WebTAK
- If you upgraded via `dpkg`, restart all services: `sudo systemctl restart takserver`
- Browser cache: hard-refresh (Ctrl+Shift+R) — old WebTAK assets may be cached
- Verify websocket compression isn't being forced on by an intermediate proxy

### PostgreSQL connection errors

```bash
sudo netstat -plnt | grep 5432
sudo -u postgres psql -c "\l"
sudo -u postgres psql -U martiuser -d cot -c "SELECT version();"
```

### Cert issues

```bash
cd /opt/tak/certs
sudo rm -rf files/
sudo ./makeRootCa.sh
sudo ./makeCert.sh server takserver
sudo ./makeCert.sh client admin
```

### Live logs

```bash
tail -f /opt/tak/logs/takserver-messaging-console.log
tail -f /opt/tak/logs/takserver-api-console.log
```

---

## Port Reference

| Port | Protocol | Service |
|------|----------|---------|
| 5432 | TCP | PostgreSQL 15 |
| 8089 | TCP/TLS | ATAK/WinTAK/OmniTAK client connection |
| 8443 | TCP/HTTPS | Web admin / WebTAK 4.10.5 |
| 8444 | TCP/HTTPS | Federation |
| 8446 | TCP/HTTPS | Cert enrollment |
| 6969 | UDP | Multicast SA discovery |
| custom | UDP | (5.7) takproto-over-UDP input, if enabled |

---

## Next Steps

1. Stress-test WebTAK at scale — 5.7's headline feature
2. Stand up Federation Hub if connecting to other servers — see [FEDERATION_SETUP.md](FEDERATION_SETUP.md)
3. Verify with OmniTAK — see [OMNITAK_TEST_GUIDE.md](OMNITAK_TEST_GUIDE.md)
4. Consider takproto-over-UDP if you're integrating with Meshtastic/LoRa gateways

---

**Tested on:** Ubuntu 24.04.3 LTS
**TAK Server version:** 5.7-RELEASE-14
**WebTAK:** 4.10.5
**Java:** Temurin 17 + OpenJDK 17
**PostgreSQL:** 15.x
