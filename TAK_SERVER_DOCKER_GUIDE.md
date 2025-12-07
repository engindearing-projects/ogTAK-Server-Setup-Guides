# TAK Server 5.5 Docker Installation Guide

**Complete guide for running TAK Server 5.5 using Docker**

*Tested on: Ubuntu 24.04.3 LTS, macOS 14+ with Docker 24.0+*

---

## IMPORTANT - READ FIRST!

### Official TAK Server Package Required

**Download the official TAK Server Docker package from tak.gov:**

- Package: `takserver-docker-5.5-RELEASE-58.tar.gz`
- Source: https://tak.gov (requires account)
- Size: ~550MB

⚠️ **Important Note:** The official package contains Dockerfiles and TAK Server binaries but **does NOT** include `docker-compose.yml` or `.env` files. You will create these files using the templates provided in this guide.

### What the Official Package Contains

```
takserver-docker-5.5-RELEASE-58/
├── docker/
│   ├── Dockerfile.takserver
│   └── Dockerfile.takserver-db
└── tak/
    ├── takserver.war (main application)
    ├── takserver-pm.jar
    ├── takserver-retention.jar
    ├── certs/ (certificate generation scripts)
    ├── db-utils/ (database utilities)
    └── [configuration files]
```

### Prerequisites

- Docker Engine 20.10+ or Docker Desktop
- Docker Compose 2.0+
- Minimum 8GB RAM (16GB+ recommended)
- 40GB+ disk space
- Downloaded TAK Server Docker package from tak.gov

---

## Table of Contents

1. [Install Docker](#install-docker)
2. [Download TAK Server Package](#download-tak-server-package)
3. [Extract and Setup](#extract-and-setup)
4. [Create docker-compose.yml](#create-docker-composeyml)
5. [Create Environment File](#create-environment-file)
6. [Generate Certificates](#generate-certificates)
7. [Configure Database Password](#configure-database-password)
8. [Build and Start Services](#build-and-start-services)
9. [Access Web Interface](#access-web-interface)
10. [Managing Containers](#managing-containers)
11. [Troubleshooting](#troubleshooting)
12. [Backup and Restore](#backup-and-restore)

---

## Install Docker

### For Ubuntu 24.04

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

### For macOS

Install Docker Desktop from https://www.docker.com/products/docker-desktop/

---

## Download TAK Server Package

1. Go to https://tak.gov and create/login to your account
2. Navigate to Downloads section
3. Download: `takserver-docker-5.5-RELEASE-58.tar.gz`

Verify download:
```bash
ls -lh ~/Downloads/takserver-docker-*.tar.gz
```

---

## Extract and Setup

```bash
# Extract the package
cd ~/Downloads
tar -xzf takserver-docker-5.5-RELEASE-58.tar.gz
cd takserver-docker-5.5-RELEASE-58

# Verify contents
ls -la
```

You should see `docker/` and `tak/` directories.

---

## Create docker-compose.yml

The official package does not include a docker-compose.yml file. Create one:

```bash
nano docker-compose.yml
```

**Copy and paste this configuration:**

```yaml
version: '3.8'

services:
  tak-database:
    build:
      context: .
      dockerfile: ./docker/Dockerfile.takserver-db
    container_name: tak-database
    hostname: tak-database
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-atakatak}
      POSTGRES_USER: postgres
      POSTGRES_DB: cot
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./tak:/opt/tak
    networks:
      - tak-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  takserver:
    build:
      context: .
      dockerfile: ./docker/Dockerfile.takserver
    container_name: takserver
    hostname: takserver
    depends_on:
      tak-database:
        condition: service_healthy
    environment:
      DB_HOST: tak-database
      DB_PORT: 5432
      DB_NAME: cot
      DB_USERNAME: postgres
      DB_PASSWORD: ${POSTGRES_PASSWORD:-atakatak}
      CONFIG_MAX_HEAP: ${CONFIG_MAX_HEAP:-256}
      MESSAGING_MAX_HEAP: ${MESSAGING_MAX_HEAP:-512}
      API_MAX_HEAP: ${API_MAX_HEAP:-512}
      RETENTION_MAX_HEAP: ${RETENTION_MAX_HEAP:-256}
      PLUGIN_MANAGER_MAX_HEAP: ${PLUGIN_MANAGER_MAX_HEAP:-256}
    volumes:
      - ./tak:/opt/tak
    ports:
      - "8443:8443"   # HTTPS API
      - "8089:8089"   # TLS client connections
      - "9000:9000"   # Federation
      - "9001:9001"   # Federation
      - "8444:8444"   # Additional
      - "8446:8446"   # Certificate enrollment
    networks:
      - tak-network
    restart: unless-stopped

volumes:
  db-data:
    driver: local

networks:
  tak-network:
    driver: bridge
```

Save and exit (Ctrl+O, Enter, Ctrl+X in nano).

---

## Create Environment File

Create a `.env` file for configuration:

```bash
nano .env
```

**Add this content:**

```bash
# Database Configuration
POSTGRES_PASSWORD=atakatak

# TAK Server Memory Configuration (in MB)
# Adjust based on your system resources
CONFIG_MAX_HEAP=256
MESSAGING_MAX_HEAP=512
API_MAX_HEAP=512
RETENTION_MAX_HEAP=256
PLUGIN_MANAGER_MAX_HEAP=256

# Certificate Configuration
CERT_STATE=California
CERT_CITY=Sacramento
CERT_ORGANIZATION=YourOrganization
CERT_ORGANIZATIONAL_UNIT=TAKServer
```

**Important:** For production, change `POSTGRES_PASSWORD` to a strong password.

Secure the file:
```bash
chmod 600 .env
```

---

## Generate Certificates

Create a setup script to generate certificates:

```bash
nano setup.sh
```

**Add this content:**

```bash
#!/bin/bash
set -e

echo "TAK Server 5.5 Certificate Generation"
echo "====================================="

# Load environment variables
if [ -f ".env" ]; then
    export $(grep -v '^#' .env | xargs)
fi

# Check if certificates already exist
if [ -d "tak/certs/files" ] && [ -f "tak/certs/files/ca.pem" ]; then
    echo "✓ Certificates already exist"
    exit 0
fi

# Set certificate environment variables
export STATE=${CERT_STATE:-California}
export CITY=${CERT_CITY:-Sacramento}
export ORGANIZATION=${CERT_ORGANIZATION:-TAK}
export ORGANIZATIONAL_UNIT=${CERT_ORGANIZATIONAL_UNIT:-TAK}

# Generate certificates
cd tak/certs

echo "Creating Root CA..."
./makeRootCa.sh --ca-name takserver

echo "Creating server certificate..."
HOSTNAME=$(hostname -f 2>/dev/null || hostname)
./makeCert.sh server "$HOSTNAME"

echo "Creating admin client certificate..."
./makeCert.sh client admin

cd ../..

echo ""
echo "✓ Certificate generation complete!"
echo ""
echo "Certificates location: tak/certs/files/"
echo "Admin certificate: tak/certs/files/admin.p12"
echo "Password: atakatak"
```

Make it executable and run:
```bash
chmod +x setup.sh
./setup.sh
```

Verify certificates were created:
```bash
ls -la tak/certs/files/
```

You should see `admin.p12`, `ca.pem`, and server certificates.

---

## Configure Database Password

⚠️ **Critical Step:** The CoreConfig.xml file must be configured with your database settings before starting TAK Server.

### Check if CoreConfig.xml Exists

```bash
ls -la tak/CoreConfig.xml
```

**If the file exists**, open it and check if it has content:
```bash
cat tak/CoreConfig.xml
```

### If CoreConfig.xml is Missing or Empty

The official TAK Server Docker package may include a minimal or empty CoreConfig.xml. If your file is missing, empty, or doesn't contain the `<connection>` section, create it using this template:

```bash
nano tak/CoreConfig.xml
```

**Copy and paste this complete CoreConfig.xml template:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration xmlns="http://bbn.com/marti/xml/config">

    <!-- Database Connection - CRITICAL: Update password to match your .env file -->
    <repository>
        <connection url="jdbc:postgresql://tak-database:5432/cot" username="martiuser" password="atakatak"/>
    </repository>

    <!-- Network Configuration -->
    <network>
        <input name="stdssl" protocol="tls" port="8089" auth="x509"/>
        <connector port="8443" tls="true" clientAuth="true"/>
        <connector port="8444" tls="true" clientAuth="false"/>
        <connector port="8446" tls="true" clientAuth="false"/>
    </network>

    <!-- Security Settings -->
    <security>
        <tls keystore="JKS" keystoreFile="/opt/tak/certs/files/takserver.jks" keystorePass="atakatak"
             truststore="JKS" truststoreFile="/opt/tak/certs/files/truststore-root.jks" truststorePass="atakatak"
             context="TLSv1.2,TLSv1.3"/>
    </security>

    <!-- Authentication -->
    <auth>
        <File location="UserAuthenticationFile.xml"/>
    </auth>

    <!-- Federation (optional - uncomment to enable) -->
    <!--
    <federation>
        <federation-server port="9000" keyManagerType="JKS"/>
    </federation>
    -->

    <!-- Submission Service -->
    <submission/>

    <!-- Subscription Service -->
    <subscription/>

    <!-- Buffer Configuration -->
    <buffer>
        <latestSA enable="true"/>
    </buffer>

</Configuration>
```

Save the file (Ctrl+O, Enter, Ctrl+X in nano).

### If CoreConfig.xml Already Has Content

If your CoreConfig.xml file exists and has content, you just need to find and update the database password.

```bash
nano tak/CoreConfig.xml
```

Look for the `<connection>` line (usually within a `<repository>` section):
```xml
<connection url="jdbc:postgresql://tak-database:5432/cot" username="martiuser" password=""/>
```

**Change the password** to match your `.env` file (default is `atakatak`):
```xml
<connection url="jdbc:postgresql://tak-database:5432/cot" username="martiuser" password="atakatak"/>
```

### Verify Configuration

After creating or editing CoreConfig.xml, verify the database password matches your `.env` file:

```bash
# Check password in .env
grep POSTGRES_PASSWORD .env

# Check password in CoreConfig.xml
grep password tak/CoreConfig.xml
```

Both passwords **must match** for the server to connect to the database.

Save and exit.

---

## Build and Start Services

### Build Docker Images

This step downloads PostgreSQL 15, Java 17, and builds the containers (5-10 minutes):

```bash
docker compose build
```

### Start Services

```bash
# Start all services
docker compose up -d

# View logs
docker compose logs -f
```

Wait 30-60 seconds for initialization.

### Verify Services Are Running

```bash
docker compose ps
```

Expected output:
```
NAME            STATUS              PORTS
tak-database    Up (healthy)        5432/tcp
takserver       Up                  0.0.0.0:8089->8089/tcp, 0.0.0.0:8443-8444->8443-8444/tcp, ...
```

Check TAK Server logs:
```bash
docker compose logs takserver | tail -50
```

---

## Access Web Interface

### Import Admin Certificate

The admin certificate is located at: `tak/certs/files/admin.p12`
Password: `atakatak`

**Firefox:**
1. Settings → Privacy & Security → Certificates → View Certificates
2. Your Certificates tab → Import
3. Select `tak/certs/files/admin.p12`
4. Enter password: `atakatak`
5. **Restart Firefox**

**Chrome/Edge:**
1. Settings → Privacy and Security → Security → Manage certificates
2. Your certificates tab → Import
3. Select `admin.p12`
4. Enter password: `atakatak`
5. **Restart browser**

**macOS:** The certificate may import to your system keychain. Ensure it's trusted:
```bash
# View certificate
security find-identity -v -p codesigning

# Trust certificate (if needed)
security add-trusted-cert -d -r trustRoot -k ~/Library/Keychains/login.keychain-db tak/certs/files/admin.p12
```

### Access Web UI

1. Open browser (must be restarted after certificate import)
2. Navigate to: **https://localhost:8443**
3. Accept security warning (self-signed certificate)
4. When prompted, select the **admin** certificate
5. You should see the TAK Server web interface

**If using a remote server:**
- Replace `localhost` with your server's IP address
- Navigate to: `https://YOUR_SERVER_IP:8443`

---

## Managing Containers

### Common Commands

**Start services:**
```bash
docker compose up -d
```

**Stop services:**
```bash
docker compose down
```

**Restart specific service:**
```bash
docker compose restart takserver
```

**View logs:**
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f takserver

# Last 100 lines
docker compose logs --tail=100 takserver
```

**Access container shell:**
```bash
docker exec -it takserver bash
```

**Check resource usage:**
```bash
docker stats
```

**Rebuild after configuration changes:**
```bash
docker compose up -d --build
```

---

## Troubleshooting

### Services Won't Start

**Check logs:**
```bash
docker compose logs
```

**Common issues:**

**1. Port already in use:**
```bash
# Check what's using port 8443
sudo lsof -i :8443
# or
sudo netstat -tlnp | grep 8443

# Solution: Stop conflicting service or change port in docker-compose.yml
```

**2. Database connection errors:**

Verify database password matches in both `.env` and `tak/CoreConfig.xml`:
```bash
grep POSTGRES_PASSWORD .env
grep password tak/CoreConfig.xml
```

They must match!

**3. Database not ready:**
```bash
# Check database health
docker compose ps

# View database logs
docker compose logs tak-database

# Restart if needed
docker compose restart tak-database
sleep 10
docker compose restart takserver
```

### Cannot Access Web UI

**Certificate not working:**
1. Verify certificate imported in browser (check browser certificate manager)
2. **Restart browser** after importing
3. Clear browser cache and cookies
4. Try different browser
5. Verify certificate password was correct: `atakatak`

**Connection refused:**
```bash
# Verify services running
docker compose ps

# Check if port is listening
docker exec takserver netstat -tlnp | grep 8443

# Check firewall (Linux)
sudo ufw status
sudo ufw allow 8443/tcp
```

**ERR_BAD_SSL_CLIENT_AUTH_CERT error:**
This means the admin certificate isn't properly installed. Reimport the certificate and restart your browser.

### Build Failures

**Out of disk space:**
```bash
# Clean up Docker
docker system prune -a

# Check disk space
df -h
```

**Network errors during build:**
```bash
# Retry build
docker compose build --no-cache
```

### Generate Additional Certificates

```bash
# Access the certificate directory
cd tak/certs

# Set environment variables (if not already set)
export STATE="California"
export CITY="Sacramento"
export ORGANIZATIONAL_UNIT="TAKServer"

# Generate client certificate
./makeCert.sh client username1

# Generate server certificate
./makeCert.sh server servername

# List generated certificates
ls -la files/
```

Certificates are created as `.p12` files with password: `atakatak`

---

## Backup and Restore

### Create Backup Script

Create `backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR="$HOME/tak-backups/backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "Creating TAK Server backup..."

# Backup database
docker exec tak-database pg_dump -U postgres cot > "$BACKUP_DIR/database.sql"

# Backup certificates
cp -r tak/certs/files "$BACKUP_DIR/certs"

# Backup configuration
cp tak/CoreConfig.xml "$BACKUP_DIR/"
cp docker-compose.yml "$BACKUP_DIR/"
cp .env "$BACKUP_DIR/.env.backup"

# Compress
tar -czf "$BACKUP_DIR.tar.gz" -C "$HOME/tak-backups" "$(basename $BACKUP_DIR)"
rm -rf "$BACKUP_DIR"

echo "Backup complete: $BACKUP_DIR.tar.gz"
```

Run backup:
```bash
chmod +x backup.sh
./backup.sh
```

### Restore from Backup

```bash
# Stop services
docker compose down

# Extract backup
tar -xzf backup-YYYYMMDD-HHMMSS.tar.gz

# Start database
docker compose up -d tak-database
sleep 10

# Restore database
cat backup-YYYYMMDD-HHMMSS/database.sql | docker exec -i tak-database psql -U postgres cot

# Restore configuration
cp backup-YYYYMMDD-HHMMSS/CoreConfig.xml tak/
cp -r backup-YYYYMMDD-HHMMSS/certs tak/certs/files

# Start services
docker compose up -d

# Verify
docker compose logs -f
```

---

## Production Recommendations

### Security

1. **Change default passwords:**
   - Update `POSTGRES_PASSWORD` in `.env`
   - Update password in `tak/CoreConfig.xml` to match

2. **Configure firewall:**
   ```bash
   sudo ufw allow 8089/tcp   # TAK client connections
   sudo ufw allow 8443/tcp   # Web interface
   sudo ufw enable
   ```

3. **Use unique certificates:**
   - Generate separate certificates for each user
   - Never share certificate files
   - Revoke compromised certificates immediately

### Performance Tuning

For high-load environments, increase memory in `.env`:

```bash
CONFIG_MAX_HEAP=512
MESSAGING_MAX_HEAP=2048
API_MAX_HEAP=2048
RETENTION_MAX_HEAP=512
```

Ensure your system has sufficient RAM.

### Monitoring

```bash
# Check container health
docker compose ps

# Monitor resources
docker stats

# View logs
docker compose logs -f takserver
```

Set up automated monitoring with tools like Prometheus and Grafana.

---

## Port Reference

- **8443** - Web administration interface (HTTPS)
- **8089** - TAK client connections (TLS)
- **8444** - Federation HTTPS
- **8446** - Certificate enrollment
- **9000** - Federation
- **9001** - Federation

---

## Additional Resources

### Official Documentation
- TAK Server: https://tak.gov
- Docker: https://docs.docker.com

### Related Guides
- [Complete Installation Tutorial](TAK_SERVER_5.5_COMPLETE_TUTORIAL.md) - Bare metal installation
- [Federation Setup](FEDERATION_SETUP.md) - Connect multiple TAK Servers
- [LDAP Setup](LDAP_SETUP_GUIDE.md) - LDAP authentication

---

## Quick Reference

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# View logs
docker compose logs -f takserver

# Restart
docker compose restart takserver

# Access container
docker exec -it takserver bash

# Check status
docker compose ps

# Backup
./backup.sh
```

---

## License & Disclaimer

This guide is provided as-is for educational purposes. TAK Server software is subject to its own licensing terms from tak.gov.

**TAK Server is U.S. Government software.** All TAK Server components must be obtained from official sources (tak.gov).

For official support:
- Documentation: https://tak.gov
- Community forums: https://tak.gov/community

---

**Last Updated:** November 2025
**TAK Server Version:** 5.5-RELEASE-58
# TAK Server Docker Guide - Section Updates

## Updates to Existing Sections

These updates fix GUI access issues and add proper client package configuration. Insert/replace in the existing TAK_SERVER_DOCKER_GUIDE.md.

---

## UPDATE 1: Troubleshooting Section - Add After Line 625

### API Service Won't Start

**Symptoms:**
- GUI returns "site can't be reached" or connection timeout
- No response on ports 8443 or 8446
- Container shows "unhealthy" status

**Diagnosis:**
```bash
# Check if API service is running inside container
docker exec takserver ps aux | grep "api.*java"

# Check API service logs
docker exec takserver cat /opt/tak/logs/takserver-api.log

# Look for this specific error:
docker exec takserver grep "FAILED TO START" /opt/tak/logs/takserver-api.log
```

**Common Error:**
```
***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to bind properties under 'server.ssl.client-auth'
Value: "false"
Reason: No enum constant org.springframework.boot.web.server.Ssl.ClientAuth.false

Action:

Update your application's configuration. The following values are valid:
    NEED
    NONE
    WANT
```

**Root Cause:**
Invalid `clientAuth` attribute in `CoreConfig.xml`. TAK Server 5.5 does NOT accept `clientAuth="false"` - it must be removed entirely or set to valid enum values.

**Solution:**
```bash
# Edit CoreConfig.xml
vim tak/CoreConfig.xml

# Find these lines and REMOVE the clientAuth attributes:
# BEFORE (INCORRECT):
# <connector port="8443" clientAuth="false" _name="https"/>
# <connector port="8446" clientAuth="false" _name="cert_https"/>

# AFTER (CORRECT):
# <connector port="8443" _name="https"/>
# <connector port="8446" _name="cert_https"/>

# Restart TAK Server
docker restart takserver

# Wait for services to start (60 seconds)
sleep 60

# Verify API service started
docker exec takserver ps aux | grep api
```

### Database Schema Not Initialized

**Symptoms:**
- GUI accessible but shows errors
- API log shows database errors
- "relation does not exist" errors in logs

**Diagnosis:**
```bash
# Check for database schema errors
docker logs takserver 2>&1 | grep "relation.*does not exist"

# Common error:
# ERROR: relation "client_endpoint" does not exist
```

**Solution:**

TAK Server should auto-initialize the database on first startup. If schema is missing:

```bash
# Stop TAK Server
docker stop takserver

# Remove and recreate database volume (WARNING: Deletes all data!)
docker volume rm takserver-docker_tak-db-data

# Restart database first
docker compose up -d tak-database

# Wait for database to be fully ready
sleep 15

# Start TAK Server
docker compose up -d takserver

# Monitor initialization
docker logs takserver --follow

# Wait for all services to report "Started" status
# This takes 60-90 seconds
```

**Verify Schema Created:**
```bash
# Check if tables exist
docker exec tak-database psql -U martiuser -d cot -c "\dt" | grep client_endpoint
```

### "Site Can't Be Reached" - Complete Diagnostic

**Full troubleshooting workflow:**

```bash
# 1. Verify container is running
docker ps --filter name=takserver

# 2. Check if ports are exposed
docker port takserver

# 3. Test localhost connection
timeout 5 curl -k -I https://localhost:8443/

# Expected responses:
# - "bad certificate" = Good! Server is running, needs client cert
# - "SSL_ERROR_SYSCALL" = API service not bound to port
# - "Connection refused" = Service not running
# - "Connection timed out" = Firewall/network issue

# 4. Check API service status inside container
docker exec takserver netstat -tlnp | grep -E "(8443|8446)"

# Should show:
# tcp 0 0 0.0.0.0:8443 0.0.0.0:* LISTEN 11/java
# tcp 0 0 0.0.0.0:8446 0.0.0.0:* LISTEN 11/java

# 5. If ports aren't listening, check API logs
docker exec takserver tail -100 /opt/tak/logs/takserver-api.log

# 6. If no log file or very small, API never started
docker logs takserver 2>&1 | grep -i "api.*started\|api.*error"
```

---

## UPDATE 2: New Section - Add After "Access Web Interface" Section

## Client Package Generation

### Understanding TAK Client Authentication

TAK clients (ATAK/iTAK) support two authentication methods:
1. **Certificate-only** - Client authenticated by X.509 certificate alone
2. **Certificate + Username/Password** - Certificate for TLS, username for identification

For multi-user deployments, method #2 is recommended as it allows unique user identification while sharing certificates.

### Configure Username/Password Authentication

**1. Edit UserAuthenticationFile.xml**

```bash
vim tak/UserAuthenticationFile.xml
```

Add users:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<UserAuthenticationFile xmlns="http://bbn.com/marti/xml/bindings">
    <User identifier="admin" password="atakatak" passwordHashed="false"/>

    <!-- Android Users -->
    <User identifier="android1" password="atakatak" passwordHashed="false"/>
    <User identifier="android2" password="atakatak" passwordHashed="false"/>
    <User identifier="android3" password="atakatak" passwordHashed="false"/>

    <!-- iPhone Users -->
    <User identifier="iphone1" password="atakatak" passwordHashed="false"/>
    <User identifier="iphone2" password="atakatak" passwordHashed="false"/>
    <User identifier="iphone3" password="atakatak" passwordHashed="false"/>
</UserAuthenticationFile>
```

**2. Update CoreConfig.xml to enable file-based authentication**

Verify this section exists in `tak/CoreConfig.xml`:
```xml
<auth>
    <File location="UserAuthenticationFile.xml"/>
</auth>
```

**3. Restart TAK Server**
```bash
docker restart takserver
sleep 30
```

### Create Client Data Packages

**1. Create directory structure**
```bash
mkdir -p client-packages/{android,iphone}
```

**2. Copy certificates**
```bash
# Android
cp tak/certs/files/android.p12 client-packages/android/
cp tak/certs/files/ca.pem client-packages/android/
cp tak/certs/files/truststore-root.p12 client-packages/android/

# iPhone
cp tak/certs/files/iphone.p12 client-packages/iphone/
cp tak/certs/files/ca.pem client-packages/iphone/
cp tak/certs/files/truststore-root.p12 client-packages/iphone/
```

**3. Create server connection preference files**

**For Android** - Create `client-packages/android/server-connection.pref`:
```xml
<?xml version='1.0' standalone='yes'?>
<preferences>
  <preference version="1" name="cot_streams">
    <entry key="count" class="class java.lang.Integer">1</entry>
    <entry key="description0" class="class java.lang.String">TAK Server</entry>
    <entry key="enabled0" class="class java.lang.Boolean">true</entry>
    <entry key="connectString0" class="class java.lang.String">YOUR_SERVER_IP:8089:ssl</entry>
    <entry key="useAuth0" class="class java.lang.Boolean">true</entry>
    <entry key="enrollForCertificateWithTrust0" class="class java.lang.Boolean">false</entry>
  </preference>
  <preference version="1" name="com.atakmap.app_preferences">
    <entry key="displayServerConnectionWidget" class="class java.lang.Boolean">true</entry>
    <entry key="caLocation" class="class java.lang.String">/storage/emulated/0/atak/cert/ca.pem</entry>
    <entry key="caPassword" class="class java.lang.String">atakatak</entry>
    <entry key="certificateLocation" class="class java.lang.String">/storage/emulated/0/atak/cert/android.p12</entry>
    <entry key="clientPassword" class="class java.lang.String">atakatak</entry>
  </preference>
</preferences>
```

**For iPhone** - Create `client-packages/iphone/server-connection.pref`:
```xml
<?xml version='1.0' standalone='yes'?>
<preferences>
  <preference version="1" name="cot_streams">
    <entry key="count" class="class java.lang.Integer">1</entry>
    <entry key="description0" class="class java.lang.String">TAK Server</entry>
    <entry key="enabled0" class="class java.lang.Boolean">true</entry>
    <entry key="connectString0" class="class java.lang.String">YOUR_SERVER_IP:8089:ssl</entry>
    <entry key="useAuth0" class="class java.lang.Boolean">true</entry>
    <entry key="enrollForCertificateWithTrust0" class="class java.lang.Boolean">false</entry>
  </preference>
  <preference version="1" name="com.atakmap.app_preferences">
    <entry key="displayServerConnectionWidget" class="class java.lang.Boolean">true</entry>
    <entry key="caLocation" class="class java.lang.String">ca.pem</entry>
    <entry key="caPassword" class="class java.lang.String">atakatak</entry>
    <entry key="certificateLocation" class="class java.lang.String">iphone.p12</entry>
    <entry key="clientPassword" class="class java.lang.String">atakatak</entry>
  </preference>
</preferences>
```

⚠️ **CRITICAL:** Replace `YOUR_SERVER_IP` with your actual server IP or domain name!

⚠️ **IMPORTANT:** The `useAuth0` flag is **required** for username/password authentication. Without it, clients will only attempt certificate-based auth and username/password fields will be ignored.

**4. Create manifest files**

**Android manifest** - Create `client-packages/android/manifest.xml`:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<MissionPackageManifest version="2">
   <Configuration>
      <Parameter name="name" value="Android TAK Client Package"/>
      <Parameter name="onReceiveDelete" value="true"/>
   </Configuration>
   <Contents>
      <Content ignore="false" zipEntry="android.p12"/>
      <Content ignore="false" zipEntry="truststore-root.p12"/>
      <Content ignore="false" zipEntry="server-connection.pref"/>
      <Content ignore="false" zipEntry="ca.pem"/>
   </Contents>
</MissionPackageManifest>
```

**iPhone manifest** - Create `client-packages/iphone/manifest.xml`:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<MissionPackageManifest version="2">
   <Configuration>
      <Parameter name="name" value="iPhone TAK Client Package"/>
      <Parameter name="onReceiveDelete" value="true"/>
   </Configuration>
   <Contents>
      <Content ignore="false" zipEntry="iphone.p12"/>
      <Content ignore="false" zipEntry="truststore-root.p12"/>
      <Content ignore="false" zipEntry="server-connection.pref"/>
      <Content ignore="false" zipEntry="ca.pem"/>
   </Contents>
</MissionPackageManifest>
```

**5. Create data package ZIP files**

```bash
# Android CONFIG package
cd client-packages/android
zip android_CONFIG.zip android.p12 truststore-root.p12 ca.pem manifest.xml server-connection.pref
cd ../..

# iPhone CONFIG package
cd client-packages/iphone
zip iphone_CONFIG.zip iphone.p12 truststore-root.p12 ca.pem manifest.xml server-connection.pref
cd ../..

# Create distribution packages
cd client-packages
zip -r android-distribution.zip android/
zip -r iphone-distribution.zip iphone/
cd ..
```

**6. Verify packages created**
```bash
ls -lh client-packages/*.zip
ls -lh client-packages/*/*.zip

# Should show:
# android-distribution.zip
# iphone-distribution.zip
# android/android_CONFIG.zip
# iphone/iphone_CONFIG.zip
```

### Client Setup Instructions

**Android (ATAK):**
1. Install ATAK from Google Play or https://tak.gov
2. Transfer `android_CONFIG.zip` to Android device
3. In ATAK: Menu → Import → Import Data Package
4. Select `android_CONFIG.zip`
5. Go to Settings → Network Connections
6. Edit "TAK Server" connection
7. Enter username: `android1` (must be unique per user!)
8. Enter password: `atakatak`
9. Enable connection
10. Verify green connection indicator

**iPhone (iTAK):**
1. Install iTAK from App Store
2. Email `iphone_CONFIG.zip` to yourself or AirDrop
3. Open file on iPhone - iTAK imports automatically
4. In iTAK → Settings → Servers
5. Edit "TAK Server" connection
6. Enter username: `iphone1` (must be unique per user!)
7. Enter password: `atakatak`
8. Enable connection
9. Verify connected status

### Testing User-to-User Visibility

**Verify users can see each other:**

1. **User 1** (android1): Connect and drop a marker
2. **User 2** (android2 or iphone1): Connect
3. User 2 should see User 1's marker within 2-3 seconds
4. Both users should see each other in Active Contacts list

**If users can't see each other:**

```bash
# Check server logs for connections
docker logs takserver --tail 50 --follow

# Look for authentication messages
docker logs takserver 2>&1 | grep -i "auth\|login"

# Verify users are in same team/group (default: __ANON__)
# In ATAK/iTAK → Settings → Self SA → Team
```

Common issues:
- **Same username used by multiple users** - Each user MUST have unique username
- **Different teams** - All users must be on same team
- **CoT filters blocking** - Disable filters for testing
- **useAuth0 flag missing** - Clients won't send username/password

---

## UPDATE 3: Add to "Generate Additional Certificates" Section (Line 643)

### Multi-User Certificate Strategy

For deployments with multiple users:

**Option 1: Shared Certificate + Unique Usernames (Recommended)**
- All Android users share `android.p12`
- All iPhone users share `iphone.p12`
- Each user has unique username (android1, android2, iphone1, iphone2, etc.)
- Advantages: Simple distribution, easy management
- Disadvantages: Can't revoke individual users

**Option 2: Individual Certificates Per User**
```bash
# Generate certificates for each user
cd tak/certs

# Android users
./makeCert.sh client user1-android
./makeCert.sh client user2-android
./makeCert.sh client user3-android

# iPhone users
./makeCert.sh client user1-iphone
./makeCert.sh client user2-iphone

# Certificates created in tak/certs/files/
ls -lh files/*.p12
```

Then create individual data packages for each user with their specific certificate.

---

## UPDATE 4: Add to "Production Recommendations" Section

### Client Distribution

1. **Use file sharing for large deployments:**
   ```bash
   # Upload packages to secure file sharing
   # Examples: Google Drive, Dropbox, OneDrive, S3
   ```

2. **Provide setup instructions:**
   - Include PDF/markdown instructions with packages
   - List assigned usernames
   - Provide troubleshooting contact

3. **Test before deployment:**
   - Verify at least 2 clients can connect
   - Confirm users see each other
   - Test marker and chat functionality

4. **Monitor connections:**
   ```bash
   # Check active connections
   docker exec takserver netstat -an | grep 8089 | grep ESTABLISHED | wc -l

   # View TAK Server logs
   docker logs takserver --follow
   ```

---

## Summary of Critical Fixes

1. **clientAuth attribute** - Must be removed from CoreConfig.xml (TAK 5.5 doesn't accept "false")
2. **useAuth0 flag** - Required in server-connection.pref for username/password auth
3. **Unique usernames** - Each user must have different username to appear separately
4. **Database schema** - Auto-initializes on first run; recreate volume if corrupted
5. **Certificate passwords** - Default is `atakatak` for all generated certificates

---

## Version History

- **v1.2** - Added GUI troubleshooting and client package generation
- **v1.1** - Initial Docker guide
- **v1.0** - Base installation guide

