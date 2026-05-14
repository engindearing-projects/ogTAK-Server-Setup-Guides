# OpenLDAP Setup Guide for TAK Server

Complete guide for setting up OpenLDAP authentication with TAK Server using Docker.

## Overview

This guide shows how to set up OpenLDAP in Docker for TAK Server authentication, allowing centralized user management instead of managing individual certificates.

## Prerequisites

- TAK Server 5.5 or later installed and running (validated against 5.7-RELEASE-8)
- Docker and Docker Compose installed
- `ldap-utils` installed on the host if you plan to run the host-side
  `ldapsearch` troubleshooting commands: `sudo apt install ldap-utils`
- Ubuntu 24.04 LTS (or similar Linux distribution)

## Quick Start

### 1. Create OpenLDAP Docker Setup

```bash
# Create directory
mkdir -p ~/openldap-docker
cd ~/openldap-docker

# Create docker-compose.yml (see below)
```

### 2. Docker Compose Configuration

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  openldap:
    image: osixia/openldap:1.5.0
    container_name: openldap
    hostname: openldap
    restart: unless-stopped
    environment:
      LDAP_ORGANISATION: "TAK Organization"
      LDAP_DOMAIN: "tak.local"
      LDAP_ADMIN_PASSWORD: "admin123"
      LDAP_CONFIG_PASSWORD: "config123"
      LDAP_READONLY_USER: "false"
      LDAP_RFC2307BIS_SCHEMA: "false"
      LDAP_BACKEND: "mdb"
      LDAP_TLS: "true"
      LDAP_TLS_ENFORCE: "false"
    ports:
      - "389:389"
      - "636:636"
    volumes:
      - openldap-data:/var/lib/ldap
      - openldap-config:/etc/ldap/slapd.d
      - openldap-certs:/container/service/slapd/assets/certs/
    networks:
      - ldap-network

  phpldapadmin:
    image: osixia/phpldapadmin:0.9.0
    container_name: phpldapadmin
    hostname: phpldapadmin
    restart: unless-stopped
    environment:
      PHPLDAPADMIN_LDAP_HOSTS: "openldap"
      PHPLDAPADMIN_HTTPS: "false"
    ports:
      - "8080:80"
    depends_on:
      - openldap
    networks:
      - ldap-network

volumes:
  openldap-data:
    name: openldap-data
  openldap-config:
    name: openldap-config
  openldap-certs:
    name: openldap-certs

networks:
  ldap-network:
    name: ldap-network
    driver: bridge
```

### 3. Create Bootstrap Users

Create `bootstrap.ldif`:

```ldif
# Create Organizational Units
dn: ou=people,dc=tak,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=tak,dc=local
objectClass: organizationalUnit
ou: groups

# Create Users — defined before the groups so the group member
# references point at real entries. (LDAP doesn't enforce
# referential integrity, so out-of-order would still import, but a
# group whose only member is a non-existent DN is a footgun once
# TAK Server's group sync starts resolving members.)
dn: cn=john.doe,ou=people,dc=tak,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: john.doe
sn: Doe
givenName: John
uid: john.doe
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/john.doe
loginShell: /bin/bash
mail: john.doe@tak.local
userPassword: password123
description: TAK User - John Doe

dn: cn=jane.smith,ou=people,dc=tak,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: jane.smith
sn: Smith
givenName: Jane
uid: jane.smith
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/jane.smith
loginShell: /bin/bash
mail: jane.smith@tak.local
userPassword: password123
description: TAK Administrator - Jane Smith

# Create Groups — groupOfNames requires at least one member, so
# each group is seeded with a real user from ou=people above.
dn: cn=tak-users,ou=groups,dc=tak,dc=local
objectClass: groupOfNames
cn: tak-users
description: TAK Users Group
member: cn=john.doe,ou=people,dc=tak,dc=local

dn: cn=tak-admins,ou=groups,dc=tak,dc=local
objectClass: groupOfNames
cn: tak-admins
description: TAK Administrators Group
member: cn=jane.smith,ou=people,dc=tak,dc=local
```

### 4. Start OpenLDAP

```bash
# Start containers
docker compose up -d

# Wait for startup (about 10 seconds)
sleep 10

# Import bootstrap data
docker cp bootstrap.ldif openldap:/tmp/bootstrap.ldif
docker exec openldap ldapadd -x -D "cn=admin,dc=tak,dc=local" -w admin123 -f /tmp/bootstrap.ldif

# Verify import
docker exec openldap ldapsearch -x -b "dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123 | grep "dn:"
```

### 5. Configure TAK Server

#### Via Web UI (Recommended)

1. Open TAK Server web UI: https://YOUR_SERVER_IP:8443
2. Navigate to **Settings → Security**
3. Scroll to **Authentication Configuration (LDAP)**
4. Fill in:
   - **URL**: `ldap://YOUR_SERVER_IP:389`
   - **User String**: `ou=people,dc=tak,dc=local`
   - **Service Account DN**: `cn=admin,dc=tak,dc=local`
   - **Service Account Password**: `admin123`
   - **Group Base RDN**: `ou=groups,dc=tak,dc=local`
   - **Group Prefix**: `cn=`
   - **Update Interval**: `600000`
5. Click **"Test Service Account"** - should show success
6. Click **"Save"**

TAK Server will automatically reload the configuration.

## Access Points

### phpLDAPadmin Web Interface
- **URL**: http://localhost:8080
- **Login DN**: `cn=admin,dc=tak,dc=local`
- **Password**: `admin123`

### TAK Server Web UI
- **URL**: https://YOUR_SERVER_IP:8443
- Use admin certificate (admin.p12) for access

## Test Users

| Username | Password | Email | DN |
|----------|----------|-------|-----|
| john.doe | password123 | john.doe@tak.local | cn=john.doe,ou=people,dc=tak,dc=local |
| jane.smith | password123 | jane.smith@tak.local | cn=jane.smith,ou=people,dc=tak,dc=local |

## Managing Users

### Add User via phpLDAPadmin (Easy)

1. Open http://localhost:8080
2. Login with admin credentials
3. Navigate to `ou=people,dc=tak,dc=local`
4. Click "Create new entry here"
5. Select template: "Generic: User Account"
6. Fill in user details:
   - Common Name (cn): username
   - Surname (sn): Last name
   - Given Name: First name
   - User ID (uid): username
   - Email: user@tak.local
   - Password: (set password)
7. Click "Create Object"

### Add User via Command Line

Create `newuser.ldif`:

```ldif
dn: cn=new.user,ou=people,dc=tak,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: new.user
sn: User
givenName: New
uid: new.user
uidNumber: 10003
gidNumber: 10003
homeDirectory: /home/new.user
loginShell: /bin/bash
mail: new.user@tak.local
userPassword: password123
description: New TAK User
```

Import:
```bash
docker exec -i openldap ldapadd -x -D "cn=admin,dc=tak,dc=local" -w admin123 < newuser.ldif
```

### List All Users

```bash
docker exec openldap ldapsearch -x -b "ou=people,dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123
```

### Test User Authentication

```bash
docker exec openldap ldapwhoami -x -D "cn=john.doe,ou=people,dc=tak,dc=local" -w password123
```

## TAK Client Configuration

### ATAK/WinTAK/iTAK Settings

LDAP replaces the **client** certificate (no more per-user `.p12`), but
the client still needs to **trust the server's TLS cert** on 8089.
Without the server CA in the client's truststore, ATAK will refuse the
connection with:

> Connection Error: Please re-import your truststore certificate

So every client needs `truststore-root.p12` regardless of LDAP. Two
ways to deliver it:

#### Option A — Manual truststore import (fastest for one device)

On the TAK server host, grab the server's CA:

```bash
# For the 5.7 docker bundle:
ls takserver-docker-5.7-RELEASE-*/tak/certs/files/truststore-root.p12

# Or pull it out of the running container:
docker cp takserver:/opt/tak/certs/files/truststore-root.p12 ./truststore-root.p12
```

Transfer `truststore-root.p12` to the phone (AirDrop, email, USB).
Default password: `atakatak`.

On ATAK:

1. Settings → **Network Preferences** → **Manage Server Connections** → tap **+**
2. **Server URL**: `YOUR_SERVER_IP:8089:ssl` (use the LAN IP — not `localhost`)
3. Tick **Use Authentication** → Username `john.doe` / Password `password123`
4. **Truststore** → Select File → pick the `truststore-root.p12` you copied → password `atakatak`
5. Leave **Client Certificate** empty (this is what makes it LDAP-mode rather than mTLS)
6. Save and toggle the connection on

If it still trips on trust, delete the saved connection and re-add —
ATAK caches a bad handshake.

#### Option B — One-click data package (preferred for fleet rollout)

Build a Mission Package zip that drops the truststore and a connection
pref onto the device in one import. On the TAK server host:

```bash
mkdir -p ldap-package/MANIFEST ldap-package/certs

# manifest
cat > ldap-package/MANIFEST/manifest.xml <<'EOF'
<MissionPackageManifest version="2">
  <Configuration>
    <Parameter name="uid" value="REPLACE-WITH-UUID"/>
    <Parameter name="name" value="TAK_LDAP"/>
    <Parameter name="onReceiveDelete" value="true"/>
  </Configuration>
  <Contents>
    <Content ignore="false" zipEntry="certs/server.pref"/>
    <Content ignore="false" zipEntry="certs/truststore-root.p12"/>
  </Contents>
</MissionPackageManifest>
EOF

sed -i "s|REPLACE-WITH-UUID|$(uuidgen)|" ldap-package/MANIFEST/manifest.xml

# connection prefs — replace YOUR.SERVER.IP with the LAN IP of the TAK host
cat > ldap-package/certs/server.pref <<'EOF'
<?xml version="1.0" encoding="ASCII" standalone="yes"?>
<preferences>
  <preference version="1" name="cot_streams">
    <entry key="count" class="class java.lang.Integer">1</entry>
    <entry key="description0" class="class java.lang.String">TAK_LDAP</entry>
    <entry key="enabled0" class="class java.lang.Boolean">true</entry>
    <entry key="connectString0" class="class java.lang.String">YOUR.SERVER.IP:8089:ssl</entry>
    <entry key="useAuth0" class="class java.lang.Boolean">true</entry>
  </preference>
  <preference version="1" name="com.atakmap.app_preferences">
    <entry key="displayServerConnectionWidget" class="class java.lang.Boolean">true</entry>
    <entry key="caLocation" class="class java.lang.String">cert/truststore-root.p12</entry>
    <entry key="caPassword" class="class java.lang.String">atakatak</entry>
  </preference>
</preferences>
EOF

cp takserver-docker-5.7-RELEASE-*/tak/certs/files/truststore-root.p12 ldap-package/certs/

cd ldap-package && zip -r ../tak-ldap.zip MANIFEST certs && cd ..
```

Notes on the package:

- The `cert/` vs `certs/` mismatch in `server.pref` is intentional —
  ATAK normalises the singular path on import. Don't "fix" it.
- No `certificateLocation` / `clientPassword` entries — that's what
  makes this an LDAP/password package rather than mTLS.
- `useAuth0=true` tells ATAK to prompt for username/password on first
  connect.

Drop `tak-ldap.zip` onto the phone (Downloads folder works). In ATAK →
**Import Manager** → **Local SD** → select the zip. It installs the
truststore and the server entry in one step, then prompts for
credentials when you enable the stream.

#### Connection settings (both options)

| Field | Value |
|---|---|
| Server Address | LAN IP of the TAK host (not `localhost`) |
| Port | `8089` |
| Protocol | TLS |
| Truststore | `truststore-root.p12` (password `atakatak`) |
| Client Cert | *(leave empty — LDAP mode)* |
| Username | `john.doe` |
| Password | `password123` |

## Docker Commands

```bash
# Start OpenLDAP
cd ~/openldap-docker
docker compose up -d

# Stop OpenLDAP
docker compose down

# View logs
docker compose logs -f openldap

# Restart after changes
docker compose restart openldap

# Check status
docker compose ps
```

## Troubleshooting

### LDAP Connection Test

```bash
# Test basic connection
ldapsearch -x -H ldap://localhost:389 -b "dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123

# Test user authentication
docker exec openldap ldapwhoami -x -D "cn=john.doe,ou=people,dc=tak,dc=local" -w password123
```

### ATAK error: "Please re-import your truststore certificate"

The client doesn't trust the server's TLS cert on 8089. LDAP only
replaces the **client** certificate — every device still needs
`truststore-root.p12` (the server CA) installed. See the
[TAK Client Configuration](#tak-clientwintakitak-settings) section
for the two ways to deliver it. Common causes once a truststore is
installed:

- Wrong truststore — exported from a different TAK server / CA than
  the one currently running. Re-export from
  `tak/certs/files/truststore-root.p12` on the live server.
- Server cert was regenerated (`makeRootCa.sh` re-run) and the
  truststore on the device is now from the old CA. Re-export and
  re-import.
- A stale connection entry in ATAK is caching the previous handshake.
  Delete the server entry in **Manage Server Connections** and re-add.

### TAK Server LDAP Issues

1. **Check TAK Server logs:** In the Docker TAK Server deployment the
   logs live *inside* the `takserver` container, not on the host —
   `sudo tail /opt/tak/logs/...` on the host won't find them. Use:
   ```bash
   docker exec takserver tail -f /opt/tak/logs/takserver-messaging.log | grep -i ldap
   docker exec takserver tail -f /opt/tak/logs/takserver-api.log | grep -i ldap
   ```
   (For a bare-metal / non-Docker TAK Server install, the host paths
   `sudo tail -f /opt/tak/logs/...` apply instead.)

2. **Verify LDAP is running:**
   ```bash
   docker compose ps
   ```

3. **Test from TAK Server host:** requires `ldap-utils` on the host
   (`sudo apt install ldap-utils`). If you'd rather not install it,
   run the same query inside the container with
   `docker exec openldap ldapsearch ...`.
   ```bash
   ldapsearch -x -H ldap://localhost:389 -b "dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123
   ```

### Reset LDAP Data

```bash
# WARNING: This deletes all LDAP data!
docker compose down -v
docker compose up -d
# Re-import bootstrap.ldif
```

## Backup and Restore

### Backup LDAP Data

```bash
# Backup all entries.
# NOTE: write slapcat's output to a path INSIDE the container, then
# copy it out. `slapcat -l ~/file` would expand `~` on the *host*
# shell before docker sees it, producing a host path the container
# can't write to.
docker exec openldap slapcat -l /tmp/ldap-backup.ldif
docker cp openldap:/tmp/ldap-backup.ldif ~/ldap-backup-$(date +%Y%m%d).ldif

# Backup specific OU — here the `>` redirect runs on the host shell,
# so `~` resolves correctly to your host home directory.
docker exec openldap ldapsearch -x -b "ou=people,dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123 > ~/ldap-users-backup-$(date +%Y%m%d).ldif
```

### Restore LDAP Data

`slapadd` writes directly to the mdb backend files and must run with
**slapd stopped** — running it against a live directory can corrupt
the database. The osixia image runs slapd as its entrypoint, so the
restore is done via a one-shot container with the entrypoint
overridden:

```bash
# Stop OpenLDAP and clear the old data volume
docker compose down
docker volume rm openldap-data

# Recreate the (empty) data volume and copy the backup into a
# one-shot container that does NOT start slapd
docker compose up -d
docker compose stop openldap
docker cp ~/ldap-backup-YYYYMMDD.ldif openldap:/tmp/restore.ldif

# Run slapadd with slapd stopped (entrypoint bypassed)
docker exec openldap slapadd -l /tmp/restore.ldif

# Bring slapd back up
docker compose start openldap
```

If `slapadd` complains the database is non-empty, the volume wasn't
cleared — re-run `docker volume rm openldap-data` after
`docker compose down`.

## Security Considerations

**This is a TEST/DEVELOPMENT setup!**

For production:

1. **Change default passwords:**
   - Admin password in docker-compose.yml
   - User passwords from `password123`

2. **Use LDAPS (port 636):**
   - Configure SSL certificates
   - Update TAK Server URL to `ldaps://`

3. **Restrict network access:**
   ```bash
   # Firewall rules
   sudo ufw allow from 192.168.0.0/24 to any port 389
   sudo ufw deny 389
   ```

4. **Strong passwords:**
   - Enforce password complexity
   - Set password expiration policies

5. **Regular backups:**
   - Automate LDAP backups
   - Test restore procedures

## Advanced Configuration

### Add User to Group

```ldif
dn: cn=tak-admins,ou=groups,dc=tak,dc=local
changetype: modify
add: member
member: cn=john.doe,ou=people,dc=tak,dc=local
```

Apply:
```bash
docker exec -i openldap ldapmodify -x -D "cn=admin,dc=tak,dc=local" -w admin123 < add-to-group.ldif
```

### Change User Password

Via phpLDAPadmin:
1. Navigate to user
2. Click "Password" link
3. Enter new password
4. Click "Update Password"

Via command line:
```bash
docker exec openldap ldappasswd -x -D "cn=admin,dc=tak,dc=local" -w admin123 -s newpassword123 "cn=john.doe,ou=people,dc=tak,dc=local"
```

## Integration with TAK Server Groups

In TAK Server web UI:

1. Go to **Settings → Groups**
2. Map LDAP groups to TAK permissions:
   - `tak-admins` → Full admin access
   - `tak-users` → Standard user access
3. Users in LDAP groups automatically get corresponding TAK permissions

## Reference

### LDAP Structure

```
dc=tak,dc=local
├── ou=people (users)
│   ├── cn=john.doe
│   ├── cn=jane.smith
│   └── (other users)
└── ou=groups
    ├── cn=tak-users
    └── cn=tak-admins
```

### Default Credentials

| Service | Username/DN | Password |
|---------|-------------|----------|
| LDAP Admin | cn=admin,dc=tak,dc=local | admin123 |
| LDAP Config | cn=admin,cn=config | config123 |
| Test User 1 | john.doe | password123 |
| Test User 2 | jane.smith | password123 |

### Ports

| Port | Service | Protocol |
|------|---------|----------|
| 389 | LDAP | TCP |
| 636 | LDAPS | TCP/SSL |
| 8080 | phpLDAPadmin | HTTP |
| 8089 | TAK Server | TLS |
| 8443 | TAK Server Web UI | HTTPS |

## Next Steps

1. ✅ Change default passwords
2. ✅ Add real users via phpLDAPadmin
3. ✅ Test authentication from TAK client
4. ✅ Configure group-based permissions
5. ✅ Set up regular backups
6. ✅ Consider LDAPS for production

## Additional Resources

- OpenLDAP Documentation: https://www.openldap.org/doc/
- phpLDAPadmin: http://phpldapadmin.sourceforge.net/
- TAK Server Documentation: https://tak.gov

## Support

For issues:
- Check logs: `docker compose logs openldap`
- Verify connectivity: Test LDAP connection from TAK Server host
- Review TAK Server logs for LDAP errors

---

**Status**: Development/Testing Setup
**Security Level**: Low (change passwords for production!)
**Tested On**: Ubuntu 24.04 LTS with TAK Server 5.5-RELEASE-58; LDAP
commands re-validated against TAK Server 5.7-RELEASE-8 + osixia/openldap
1.5.0 on 2026-05-14
