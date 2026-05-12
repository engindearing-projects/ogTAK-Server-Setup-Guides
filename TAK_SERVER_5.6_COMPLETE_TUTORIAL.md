# TAK Server 5.6 Installation Guide for Ubuntu 24.04

**Complete step-by-step tutorial for installing TAK Server 5.6 on Ubuntu 24.04**

*Tested release tag: `5.6-RELEASE-35` (published 2026-04-03 on TAK-Product-Center/Server)*

> **Migrating from 5.5?** The OS-level prerequisites (Java, PostgreSQL, ports) are unchanged from 5.5. If you've followed the 5.5 guide before, you mostly just swap the `.deb` filename and re-apply the script-path fixes. See the [5.5 → 5.6 delta](#whats-new-vs-55) section for the short list.

---

## IMPORTANT GOTCHAS - READ FIRST!

The same three gotchas from 5.5 still bite on 5.6:

### PostgreSQL Version: MUST Use Version 15

- TAK Server 5.6 still pins to **PostgreSQL 15** (verified in `takserver-package/database/build.gradle` at tag `5.6-RELEASE-35`: `requires('postgresql-15')`, `requires('postgresql-15-postgis-3')`)
- Ubuntu 24.04's default `apt install postgresql` installs 16 — that **will** break TAK Server
- Install 15 explicitly from the PGDG APT repo (steps below)

### Java: OpenJDK 17 is REQUIRED

- The `.deb` declares `requires('openjdk-17-jdk')` — do not remove it
- We install Temurin 17 as the runtime AND keep OpenJDK 17 to satisfy the `.deb` dependency
- This dual-install pattern is the same as 5.5

### Federation Hub is a Separate Package

- `takserver-fed-hub_5.6-RELEASE35_all.deb` is a separate download from tak.gov
- Federation Hub scripts still need PATH fixes for Ubuntu 24.04 (full paths to `/usr/bin/awk` and `/usr/bin/java`)
- See [FEDERATION_SETUP.md](FEDERATION_SETUP.md) — the steps are identical, just bump the filename

### New for 5.6: Heads up on the Spring Security 6.5.x bump

5.6 ships with `spring_security_version = 6.5.9`. If you have any custom OAuth2/SSO config carried over from older TAK installs, re-test it after upgrade — Spring 6.5 tightened defaults around redirect URI matching.

---

## What's new vs 5.5

Pulled from `gradle.properties` and source diffs between tags `5.5-RELEASE-82` and `5.6-RELEASE-35`:

| Component | 5.5 | 5.6 | Notes |
|-----------|-----|-----|-------|
| `commons_codec` | 1.15 | 1.17.1 | Apache Commons bumped |
| `httpcomponents` | 5.4.4 | 5.4.3 | Slight downgrade — intentional, keep as-is |
| `spring_ldap` | 3.1.6 | 3.2.0 | Re-test LDAP/AD bind if customized |
| `commons_beanutils` | 1.11.0 | (removed) | No longer pulled in by core |
| OS deps (Java, PostgreSQL, OpenSSL) | unchanged | unchanged | Same install steps |
| `setenv.sh` JDK opens | unchanged | unchanged | Same `--add-opens` set |
| Federation Hub | required for fed | required for fed | Same package layout |

If you're upgrading in-place from 5.5, the database schema migration is handled by `SchemaManager.jar upgrade` (same command, same path).

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
- Minimum 8GB RAM (16GB+ recommended)
- 40GB+ disk space
- Internet connection
- TAK Server 5.6 `.deb` package downloaded from [tak.gov](https://tak.gov) — registration required
  - Filename: `takserver_5.6-RELEASE35_all.deb` (or whatever the current 5.6 RELEASE-N is when you download)

---

## Java Installation

TAK Server 5.6 still requires Java 17. Install BOTH Temurin 17 (runtime) AND OpenJDK 17 (`.deb` dependency).

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

### 4. Verify Java

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

### 3. Pin port to 5432 (default port)

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

### 1. Install the 5.6 `.deb`

```bash
cd ~/Downloads
sudo dpkg --force-depends -i takserver_5.6-RELEASE35_all.deb
```

(`--force-depends` because we have Temurin alongside OpenJDK — `dpkg` only sees the OpenJDK dependency.)

### 2. Verify

```bash
ls -la /opt/tak/
```

You should see `CoreConfig.example.xml`, `takserver.sh`, `setenv.sh`, `certs/`, `db-utils/`, `federation-hub/`, etc.

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

> **Production:** replace `takserver123` with a strong password and update `pg_hba.conf` accordingly.

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

---

## Certificate Generation

### 1. Configure metadata

```bash
cd /opt/tak/certs
sudo sed -i 's/STATE=${STATE}/STATE=YourState/' /opt/tak/certs/cert-metadata.sh
sudo sed -i 's/CITY=${CITY}/CITY=YourCity/' /opt/tak/certs/cert-metadata.sh
sudo sed -i 's/ORGANIZATIONAL_UNIT=${ORGANIZATIONAL_UNIT}/ORGANIZATIONAL_UNIT=TAKServer/' /opt/tak/certs/cert-metadata.sh
```

### 2. Generate Root CA

```bash
cd /opt/tak/certs
sudo ./makeRootCa.sh
```

Pick a unique CA name (e.g., `MyTAKServer56`).

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

You should see `.pem`, `.p12`, and `.jks` files for each cert.

---

## CoreConfig Configuration

### 1. Enable multicast SA discovery

```bash
sudo sed -i 's/<!-- <input _name="SAproxy" protocol="mcast" group="239.2.3.1" port="6969" proxy="true" auth="anonymous" \/> -->/<input _name="SAproxy" protocol="mcast" group="239.2.3.1" port="6969" proxy="true" auth="anonymous" \/>/' /opt/tak/CoreConfig.xml
```

### 2. Enable server announcement (optional)

```bash
SERVER_IP=$(hostname -I | awk '{print $1}')
sudo sed -i "s/ip=\"192.168.1.1\"/ip=\"$SERVER_IP\"/" /opt/tak/CoreConfig.xml
sudo sed -i 's/<!--<announce enable="true" uid="Marti1"/<announce enable="true" uid="TakServer1"/' /opt/tak/CoreConfig.xml
sudo sed -i 's/<\/announce>-->/<\/announce>/' /opt/tak/CoreConfig.xml
```

### 3. Verify

```bash
grep -E "(SAproxy|announce)" /opt/tak/CoreConfig.xml
```

Defaults that ship enabled in 5.6 (no change vs 5.5):
- Video streaming (`streamingbroker enable="true"`)
- File sharing
- Mission API
- HTTPS on 8443/8444/8446
- TLS on 8089

---

## Fix Startup Scripts

The systemd-launched scripts need absolute paths under Ubuntu 24.04 (carries over from 5.5):

```bash
sudo sed -i 's|TOTALRAMBYTES=`awk|TOTALRAMBYTES=`/usr/bin/awk|' /opt/tak/setenv.sh
sudo sed -i 's|^rm -rf|/usr/bin/rm -rf|' /opt/tak/takserver-messaging.sh
sudo sed -i 's|^java |/usr/bin/java |' /opt/tak/takserver-messaging.sh
sudo sed -i 's|^java |/usr/bin/java |' /opt/tak/takserver-api.sh
sudo sed -i 's|^java |/usr/bin/java |' /opt/tak/takserver-config.sh
```

Also create the Ignite config:

```bash
sudo cp /opt/tak/TAKIgniteConfig.example.xml /opt/tak/TAKIgniteConfig.xml
```

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

Expect 2-3 Java processes (messaging / API / config) and ports 8089, 8443, 8444, 8446 listening on `0.0.0.0`.

---

## Accessing TAK Server

| URL | Purpose |
|-----|---------|
| `https://YOUR_SERVER_IP:8443` | Web admin (requires client cert) |
| `https://YOUR_SERVER_IP:8446` | Certificate enrollment |
| `https://YOUR_SERVER_IP:8444` | Federation HTTPS |

Copy the admin certificate locally:

```bash
sudo cp /opt/tak/certs/files/admin.p12 ~/admin.p12
sudo chown $USER:$USER ~/admin.p12
```

Default cert password: `atakatak` — change it for production.

---

## Verify with OmniTAK

The fastest sanity test for a fresh 5.6 server is to point an OmniTAK client at it. See [OMNITAK_TEST_GUIDE.md](OMNITAK_TEST_GUIDE.md) for the full client-side walkthrough — short version:

1. Generate a client cert: `sudo ./makeCert.sh client engie-test`
2. Copy `/opt/tak/certs/files/engie-test.p12` to your phone
3. In OmniTAK-Android or OmniTAK-iOS: **Settings → TAK Servers → +**, enter your server IP, port `8089`, protocol `SSL/TLS`, attach the `.p12` (password `atakatak`)
4. Long-press the map to drop a marker — confirm it lands on the server's web UI under **Marti → COT** within a couple of seconds

If the marker round-trips, the install is healthy.

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
| 8443 | TCP/HTTPS | Web admin (client cert required) |
| 8444 | TCP/HTTPS | Federation |
| 8446 | TCP/HTTPS | Certificate enrollment |
| 6969 | UDP | Multicast SA discovery |

---

## Next Steps

1. Import admin cert into browser → access `https://YOUR_SERVER_IP:8443`
2. Create users via `UserManager.jar`
3. Stand up Federation Hub if connecting to other TAK servers — see [FEDERATION_SETUP.md](FEDERATION_SETUP.md) (use 5.6 fed-hub `.deb`)
4. Verify with OmniTAK clients — see [OMNITAK_TEST_GUIDE.md](OMNITAK_TEST_GUIDE.md)

---

**Tested on:** Ubuntu 24.04.3 LTS
**TAK Server version:** 5.6-RELEASE-35
**Java:** Temurin 17.0.16 + OpenJDK 17
**PostgreSQL:** 15.x
