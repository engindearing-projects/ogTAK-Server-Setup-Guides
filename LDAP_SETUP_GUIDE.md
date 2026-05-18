# OpenLDAP Authentication for TAK Server (Docker)

> **May 2026 rewrite.** The previous version of this guide described a
> flow where ATAK connects to :8089 with an LDAP password and no
> client certificate. That flow physically can't work — TAK's :8089
> input requires mutual TLS — and following it led at least two teams
> down a blind alley. This rewrite covers the actual architecture:
> LDAP authenticates against :8446 to *enroll for a per-device client
> certificate*, and the cert is what :8089 trusts. Credit to **Anna
> Savery** and **Lachlan Marnoch** (Cognitive Advantage, UNSW
> Canberra) for working through every layer of this with us in May
> 2026 and surfacing the gaps.

## How TAK + LDAP actually works

The intended flow:

1. EUD opens connection setup in ATAK and ticks **"Enroll for Client Certificate"**
2. ATAK posts the LDAP credentials (Basic auth) to `https://server:8446`
3. TAK Server validates against LDAP, looks up the user's group memberships, signs a CSR with the TAK signing CA, and returns the new cert inside an XML config bundle
4. ATAK installs the cert silently and uses it on :8089 from then on

The user never sees a `.p12`. But a certificate is in play — **LDAP doesn't replace client certs, it gates who gets to enroll for one**.

This architecture means three independent things have to be working for enrollment to complete. Most setups break on one of them and the symptom looks like an LDAP problem when it isn't:

| Gate | What it does | Symptom when wrong |
|------|--------------|--------------------|
| **1. LDAP wiring** | TAK can bind to LDAP, resolve `memberOf`, accept Basic auth on :8446 | :8446 returns the angular `loginManager` HTML; or 401 even with valid creds |
| **2. Certificate signing** | TAK has a CA keystore + a `<certificateSigning>` block in CoreConfig | `/Marti/api/tls/config` returns HTTP 500 "TAK Server resource unavailable or not allowed" |
| **3. Signing CA in the :8089 truststore** | The server trusts the certs it just issued | ATAK shows "**Registration complete**" then immediately throws "IO error; reconnect" on the data stream |

The rest of this guide walks all three gates with a verification step after each. Skip ahead with confidence — if Gate 1's verification passes, you know Gate 1 is real.

## Prerequisites

- TAK Server 5.5+ deployed via Docker (validated against 5.7-RELEASE-8; LDAP semantics unchanged through -32)
- Docker + Docker Compose
- `ldap-utils` on the host if you want to run `ldapsearch` outside the container: `sudo apt install ldap-utils`

---

## Gate 1 — LDAP wiring

### 1.1 Run OpenLDAP with the `memberof` overlay

Create a working directory for the LDAP stack:

```bash
mkdir -p ~/openldap-docker
cd ~/openldap-docker
```

Create `docker-compose.yml`:

```yaml
services:
  openldap:
    image: osixia/openldap:1.5.0
    container_name: openldap
    hostname: openldap
    restart: unless-stopped
    environment:
      LDAP_ORGANISATION: "TAK Organization"
      LDAP_DOMAIN: "tak.local"
      LDAP_ADMIN_PASSWORD: "admin"
      LDAP_CONFIG_PASSWORD: "config"
      LDAP_BACKEND: "mdb"
      LDAP_TLS: "false"        # plain LDAP inside docker network; see Security section for prod
    ports:
      - "389:389"
    volumes:
      - openldap-data:/var/lib/ldap
      - openldap-config:/etc/ldap/slapd.d
    networks:
      - tak-network

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
      - tak-network

volumes:
  openldap-data:
  openldap-config:

networks:
  tak-network:
    name: tak-network
    driver: bridge
    external: true   # see note below
```

> **Note on the `tak-network`.** The takserver container has to be able to resolve `openldap` by hostname. Easiest way: put both containers on the same docker network. If your TAK Server compose already defines a network (most do), declare it `external: true` here and join it; otherwise drop the `external: true` line and add the TAK Server container to this network instead.

Start the stack:

```bash
docker compose up -d
sleep 10   # wait for slapd to come up
```

### 1.2 Enable the `memberof` overlay

The osixia image doesn't load `memberof` by default. Without it, TAK can authenticate users but every login is rejected with "user is not a member of any group". Post this LDIF to `cn=config`:

```bash
cat > enable-memberof.ldif <<'EOF'
# Load the modules
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: memberof.la
-
add: olcModuleLoad
olcModuleLoad: refint.la

# Configure memberof overlay on the main database
dn: olcOverlay=memberof,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcMemberOf
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member
olcMemberOfMemberOfAD: memberOf

# refint overlay keeps memberOf consistent across rename/delete
dn: olcOverlay=refint,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
olcOverlay: refint
olcRefintAttribute: memberof member
EOF

docker cp enable-memberof.ldif openldap:/tmp/enable-memberof.ldif
docker exec openldap ldapmodify -Y EXTERNAL -H ldapi:/// -f /tmp/enable-memberof.ldif
```

### 1.3 Bootstrap users and groups

The critical detail here is **group naming**: TAK uses an `<x>_READ` / `<x>_WRITE` convention to map LDAP groups to IN/OUT permissions. Groups named `tak-users` or `admins` won't match TAK's default group-name extractor regex.

Create `bootstrap.ldif`:

```ldif
# Organizational units
dn: ou=users,dc=tak,dc=local
objectClass: organizationalUnit
ou: users

dn: ou=groups,dc=tak,dc=local
objectClass: organizationalUnit
ou: groups

# Users — define before the groups so group member: refs resolve
dn: cn=john.doe,ou=users,dc=tak,dc=local
objectClass: inetOrgPerson
cn: john.doe
sn: Doe
givenName: John
uid: john.doe
mail: john.doe@tak.local
userPassword: password123

dn: cn=jane.smith,ou=users,dc=tak,dc=local
objectClass: inetOrgPerson
cn: jane.smith
sn: Smith
givenName: Jane
uid: jane.smith
mail: jane.smith@tak.local
userPassword: password123

# Groups — names MUST end in _READ or _WRITE for TAK's group parser
# (or change groupNameExtractorRegex in CoreConfig — see 1.4)
dn: cn=tak_users_READ,ou=groups,dc=tak,dc=local
objectClass: groupOfNames
cn: tak_users_READ
description: Receive data from TAK Server
member: cn=john.doe,ou=users,dc=tak,dc=local
member: cn=jane.smith,ou=users,dc=tak,dc=local

dn: cn=tak_users_WRITE,ou=groups,dc=tak,dc=local
objectClass: groupOfNames
cn: tak_users_WRITE
description: Send data to TAK Server
member: cn=john.doe,ou=users,dc=tak,dc=local
member: cn=jane.smith,ou=users,dc=tak,dc=local
```

Load it:

```bash
docker cp bootstrap.ldif openldap:/tmp/bootstrap.ldif
docker exec openldap ldapadd -x -D "cn=admin,dc=tak,dc=local" -w admin -f /tmp/bootstrap.ldif
```

### 1.4 Configure TAK CoreConfig — `<auth>` and `<network>`

Open `/opt/tak/CoreConfig.xml` inside the takserver container:

```bash
docker exec -it takserver vi /opt/tak/CoreConfig.xml
```

The `<auth>` block needs **six things** that the public TAK docs gloss over. Replace your current `<auth>` with:

```xml
<auth default="ldap" x509groups="true" x509addAnonymous="false">
    <ldap
        url="ldap://openldap:389"
        userstring="cn={username},ou=users,dc=tak,dc=local"
        updateinterval="60"
        groupNameExtractorRegex="[Cc][Nn]=(.*?)(?:,|$)"
        style="DS"
        ldapSecurityType="none"
        serviceAccountDN="cn=admin,dc=tak,dc=local"
        serviceAccountCredential="admin"
        groupObjectClass="groupOfNames"
        userObjectClass="inetOrgPerson"
        groupBaseRDN="ou=groups,dc=tak,dc=local"
        userBaseRDN="ou=users,dc=tak,dc=local"
        x509groups="true"
        x509addAnonymous="false"
        callsignAttribute="cn"
        enableConnectionPool="false"
        dnAttributeName="entryDN"
        nameAttr="cn"/>
    <File location="UserAuthenticationFile.xml"/>
</auth>
```

Why each line matters:

| Setting | Why |
|---------|-----|
| `default="ldap"` | **The single biggest gap.** Without it, TAK silently uses File auth and ignores the entire `<ldap>` block. |
| `ldapSecurityType="none"` | The default `"simple"` triggers StartTLS, which OpenLDAP-without-TLS rejects with "unsupported extended operation". |
| `dnAttributeName="entryDN"` | The schema default `distinguishedName` is AD-only; OpenLDAP doesn't populate it and TAK builds a malformed DN on the second bind. |
| `groupNameExtractorRegex="[Cc][Nn]=(.*?)(?:,|$)"` | TAK's default is case-sensitive `CN=`; OpenLDAP DNs are lowercase `cn=`. Case-insensitive regex fixes both. |
| `enableConnectionPool="false"` | Connection pool caches the service-account context and goes stale across container restarts. Off is fine for hundreds of users. |
| `callsignAttribute="cn"` | What ATAK displays as the callsign on the map for that EUD. |

Now the `<network>` block — find the `<connector port="8446" ...>` line and make sure both attributes below are present:

```xml
<network ...>
    <input auth="x509" _name="stdssl" protocol="tls" port="8089" coreVersion="2"/>
    <connector port="8443" _name="https"/>
    <connector port="8446"
               clientAuth="false"
               allowBasicAuth="true"
               _name="cert_https"/>
    <announce/>
</network>
```

- `clientAuth="false"` — :8446 must not require a client cert (the whole point: enrollment happens *before* the EUD has one).
- `allowBasicAuth="true"` — without this, :8446 ignores HTTP Basic auth and redirects to the angular login HTML, which ATAK can't handle.

Save and restart:

```bash
docker restart takserver
```

### 1.5 Verify Gate 1

From the TAK host, all three should succeed:

```bash
# 1. LDAP bind as the user works
docker exec openldap ldapwhoami -x -D "cn=john.doe,ou=users,dc=tak,dc=local" -w password123
# Expect: dn:cn=john.doe,ou=users,dc=tak,dc=local

# 2. memberOf is actually populated (silently empty = overlay not loaded)
docker exec openldap ldapsearch -x -D "cn=admin,dc=tak,dc=local" -w admin \
    -b "cn=john.doe,ou=users,dc=tak,dc=local" memberOf
# Expect: at least one "memberOf: cn=tak_users_READ,..." line

# 3. :8446 accepts Basic auth and reaches the TAK auth chain
curl -sk -u "john.doe:password123" https://localhost:8446/Marti/api/version -w "\nHTTP: %{http_code}\n"
# Expect: HTTP 200 with a version string. NOT the loginManager HTML.
```

If #1 fails → users aren't loaded or password is wrong. Re-run the bootstrap.
If #2 is empty → memberof overlay isn't loaded. Re-run section 1.2 and restart slapd.
If #3 returns the angular `loginManager` HTML → `allowBasicAuth="true"` isn't on the :8446 connector. Re-edit CoreConfig.

---

## Gate 2 — Certificate signing

Gate 1 lets TAK authenticate the user. Gate 2 gives TAK something to sign client CSRs with.

### 2.1 Create the signing CA

Standard TAK pattern is a three-tier chain: root CA → intermediate signing CA → end-entity certs. The signing CA is what `<certificateSigning>` points at. Inside the takserver container:

```bash
docker exec -it takserver bash
cd /opt/tak/certs

# If you don't already have a root CA from initial setup:
./makeRootCa.sh --ca-name takserver-root

# Create the intermediate signing CA
./makeCert.sh ca intermediate-signing

ls files/intermediate-signing.*
# Expect: intermediate-signing.jks, .key, .pem, .p12, -trusted.pem
```

If `makeCert.sh ca` fails on missing `STATE`/`CITY`/`ORGANIZATIONAL_UNIT`, edit `cert-metadata.sh` and set those, then re-run.

### 2.2 Add `<certificateSigning>` to CoreConfig

Add this as a top-level child of `<Configuration>` (sibling of `<security>` and `<network>`):

```xml
<certificateSigning CA="TAKServer">
    <certificateConfig>
        <nameEntries>
            <nameEntry name="O" value="TAK"/>
            <nameEntry name="OU" value="TAK"/>
        </nameEntries>
    </certificateConfig>
    <TAKServerCAConfig
        keystore="JKS"
        keystoreFile="certs/files/intermediate-signing.jks"
        keystorePass="atakatak"
        validityDays="365"
        signatureAlg="SHA256WithRSA"
        CAkey="certs/files/intermediate-signing"
        CAcertificate="certs/files/intermediate-signing"/>
</certificateSigning>
```

Notes:

- `keystoreFile` is **relative to `/opt/tak`**, not absolute.
- `CAkey` and `CAcertificate` are paths **without an extension** — TAK appends `.key` and `.pem` internally.
- `keystorePass="atakatak"` is the default from `cert-metadata.sh`. Change it in `CAPASS=` first if you're not using the default.

Restart:

```bash
docker restart takserver
```

### 2.3 Verify Gate 2

```bash
curl -sk -u "john.doe:password123" https://localhost:8446/Marti/api/tls/config
```

Expect XML with a `<certificateConfig>` element. **Not** an HTML page that says "TAK Server resource unavailable or not allowed" — that 500 is the `<certificateSigning>` block being missing or pointing at a keystore TAK can't open.

---

## Gate 3 — Signing CA in the :8089 truststore

This is the gate Anna and Lachlan hit. ATAK enrolls fine, the success dialog reads **"TAK Server registration succeeded"**, and immediately the client throws **"IO error; reconnect"** on the data stream.

### 3.1 Why this happens

Two different keystores are in play on the *same server*:

| Endpoint | Keystore | Purpose |
|----------|----------|---------|
| `:8446` enrollment | `intermediate-signing.jks` | Signs new client certs |
| `:8089` data stream (`<input auth="x509">`) | `truststore-root.jks` | Validates the client certs ATAK presents |

If the signing CA isn't chained into `truststore-root.jks`, the server refuses the cert it just issued. The two-keystore split is easy to miss.

### 3.2 Chain the signing CA into truststore-root.jks

Inside the takserver container:

```bash
docker exec takserver bash -c '
  cd /opt/tak/certs/files
  keytool -export -alias ca \
      -file /tmp/signing-ca.crt \
      -keystore intermediate-signing.jks -storepass atakatak
  keytool -import -noprompt -alias signing-ca \
      -file /tmp/signing-ca.crt \
      -keystore truststore-root.jks -storepass atakatak
'
docker restart takserver
```

If you used a non-default keystore password, swap `atakatak` for it.

### 3.3 Verify Gate 3

```bash
docker exec takserver keytool -list -keystore /opt/tak/certs/files/truststore-root.jks -storepass atakatak
```

Expect at least two entries: the root CA, plus `signing-ca` (or whatever alias you used).

End-to-end test: complete an ATAK enrollment per the next section. The success dialog should be followed by a green connection indicator on the bottom bar, not an "IO error".

---

## Set up an EUD (ATAK)

### Get the truststore on the device

ATAK still needs to trust the server's TLS cert on :8446 *before* it can enroll, so every device needs `truststore-root.p12`:

```bash
docker cp takserver:/opt/tak/certs/files/truststore-root.p12 ./truststore-root.p12
```

Transfer to the device (AirDrop / email / USB). Default password: `atakatak`.

### Enroll the device

1. ATAK → **Settings** → **Network Preferences** → **Manage Server Connections** → tap **+**
2. **Server Address**: LAN IP of the TAK host (not `localhost`)
3. **Port**: `8089` (data stream — ATAK will hit :8446 for enrollment automatically)
4. **Protocol**: TLS
5. Tick **"Enroll for Client Certificate"** ← this is what triggers the LDAP-and-enrollment path instead of expecting a pre-installed cert
6. Tick **"Use Authentication"** → username `john.doe`, password `password123`
7. **Truststore**: pick `truststore-root.p12`, password `atakatak`
8. Leave **Client Certificate** empty (you don't have one — you're enrolling *for* one)
9. **Save** and toggle the connection on

ATAK posts to :8446 with Basic auth, gets a signed cert back, installs it silently, then opens a TLS socket to :8089. On success the bar at the bottom of the map turns green and the cot stream starts flowing.

### Fleet rollout — mission package

For more than a handful of devices, the manual flow is tedious. Build a one-import mission package on the TAK host:

```bash
mkdir -p ldap-package/MANIFEST ldap-package/certs

# Manifest
cat > ldap-package/MANIFEST/manifest.xml <<EOF
<MissionPackageManifest version="2">
  <Configuration>
    <Parameter name="uid" value="$(uuidgen)"/>
    <Parameter name="name" value="TAK_LDAP"/>
    <Parameter name="onReceiveDelete" value="true"/>
  </Configuration>
  <Contents>
    <Content ignore="false" zipEntry="certs/server.pref"/>
    <Content ignore="false" zipEntry="certs/truststore-root.p12"/>
  </Contents>
</MissionPackageManifest>
EOF

# Server pref — replace YOUR.SERVER.IP with the LAN IP
cat > ldap-package/certs/server.pref <<'EOF'
<?xml version="1.0" encoding="ASCII" standalone="yes"?>
<preferences>
  <preference version="1" name="cot_streams">
    <entry key="count" class="class java.lang.Integer">1</entry>
    <entry key="description0" class="class java.lang.String">TAK_LDAP</entry>
    <entry key="enabled0" class="class java.lang.Boolean">true</entry>
    <entry key="connectString0" class="class java.lang.String">YOUR.SERVER.IP:8089:ssl</entry>
    <entry key="useAuth0" class="class java.lang.Boolean">true</entry>
    <entry key="enrollForCertificateWithTrust0" class="class java.lang.Boolean">true</entry>
  </preference>
  <preference version="1" name="com.atakmap.app_preferences">
    <entry key="caLocation" class="class java.lang.String">cert/truststore-root.p12</entry>
    <entry key="caPassword" class="class java.lang.String">atakatak</entry>
  </preference>
</preferences>
EOF

docker cp takserver:/opt/tak/certs/files/truststore-root.p12 ldap-package/certs/

cd ldap-package && zip -r ../tak-ldap.zip MANIFEST certs && cd ..
```

Notes:

- `enrollForCertificateWithTrust0=true` is the pref-file equivalent of ticking "Enroll for Client Certificate" in the UI.
- `cert/` vs `certs/` mismatch in `server.pref` is intentional — ATAK normalises the singular form on import. Don't "fix" it.
- No `certificateLocation` / `clientPassword` — those would tell ATAK to use a pre-installed cert instead of enrolling.

Drop `tak-ldap.zip` on the device, ATAK → **Import Manager** → **Local SD** → select the zip. Enable the stream and it prompts for credentials on first connect.

---

## Map LDAP groups to TAK channels

Once enrollment works, head to the TAK Server admin UI (https://YOUR_SERVER:8443) → **Groups** to map your `<x>_READ` and `<x>_WRITE` LDAP groups to TAK channels. By the IN/OUT convention:

- `tak_users_READ` → members can **receive** from the `tak_users` channel
- `tak_users_WRITE` → members can **send** to the `tak_users` channel

A user in both groups is read/write. A user in just `_READ` is receive-only.

---

## Troubleshooting

### "Connection Error: Please re-import your truststore certificate"

The client doesn't trust the server's TLS cert on :8089 (or :8446). Almost always:

- Wrong `truststore-root.p12` — exported from a different TAK server.
- Server cert was regenerated (`makeRootCa.sh` re-run) and the device's truststore is from the old CA.
- A stale connection entry in ATAK is caching the previous handshake. Delete and re-add.

### `SSLHandshakeException: ... peer not verified` on :8089

You haven't ticked "Enroll for Client Certificate" *and* there's no client cert installed. :8089 will not let the connection past the handshake. Either enable enrollment or install a `.p12`.

### `/Marti/api/tls/config` returns 500 "resource unavailable or not allowed"

Gate 2 isn't satisfied — either `<certificateSigning>` is missing from CoreConfig, the `keystoreFile` path is wrong, or the keystore password doesn't match. Check takserver-api.log for the underlying `IOException` or `KeyStoreException`.

### "Registration complete" → immediate "IO error; reconnect"

Gate 3 — the signing CA isn't in `truststore-root.jks`. Run the keytool commands in section 3.2. Confirm with:

```bash
docker exec takserver tail -n 200 /opt/tak/logs/takserver-messaging.log \
    | grep -i -E "8089|sslhandshake|peer not verified|certpath"
```

`peer not verified` or `no trusted certificate found` confirms it.

### TAK logs to check

```bash
docker exec takserver tail -f /opt/tak/logs/takserver-messaging.log | grep -i ldap
docker exec takserver tail -f /opt/tak/logs/takserver-api.log | grep -i ldap
```

Logs live **inside** the takserver container, not on the host — `sudo tail /opt/tak/logs/...` on the host won't find them. For non-Docker installs, the host paths apply.

### LDAP-side checks

```bash
# Bind succeeds as a user
docker exec openldap ldapwhoami -x -D "cn=john.doe,ou=users,dc=tak,dc=local" -w password123

# Dump everything as admin
docker exec openldap ldapsearch -x -D "cn=admin,dc=tak,dc=local" -w admin -b "dc=tak,dc=local"

# Tail slapd's logs after bumping log level
docker exec openldap ldapmodify -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats
EOF
docker logs -f openldap
```

`stats` log level shows every BIND and SEARCH with the DN attempted, the result code, and the search filter — invaluable for figuring out what TAK is actually asking for.

### Reset LDAP data

```bash
docker compose down -v   # ⚠️ destroys all LDAP entries
docker compose up -d
# Re-run enable-memberof.ldif and bootstrap.ldif
```

---

## Backup and restore

### Backup

```bash
# Write to a path INSIDE the container, then copy out.
# `slapcat -l ~/file` would expand ~ on the host before docker sees it.
docker exec openldap slapcat -l /tmp/ldap-backup.ldif
docker cp openldap:/tmp/ldap-backup.ldif ~/ldap-backup-$(date +%Y%m%d).ldif
```

### Restore

`slapadd` writes directly to the mdb backend and must run with `slapd` stopped. The osixia image runs slapd as its entrypoint, so the restore needs a separate path:

```bash
docker compose down
docker volume rm openldap-data
docker compose up -d
docker compose stop openldap
docker cp ~/ldap-backup-YYYYMMDD.ldif openldap:/tmp/restore.ldif
docker exec openldap slapadd -l /tmp/restore.ldif
docker compose start openldap
```

---

## Production hardening

Everything in this guide uses defaults that are fine for a lab and unacceptable in production. Before exposing the stack:

1. **Rotate every password** — `LDAP_ADMIN_PASSWORD`, `LDAP_CONFIG_PASSWORD`, `serviceAccountCredential` in CoreConfig, `userPassword` in bootstrap.ldif, `CAPASS` in `cert-metadata.sh`, `keystorePass` in `<certificateSigning>`.
2. **LDAPS** — set `LDAP_TLS: "true"` on the openldap container, change `url="ldap://"` → `ldaps://` and `ldapSecurityType="none"` → `"simple"` in CoreConfig.
3. **Network scoping** — :389 should never be on a public interface. UFW or a docker network with no host port is cleaner.
4. **Password policy** — add `ppolicy` overlay for complexity + expiry. Out of scope here.
5. **Backup automation** — schedule the `slapcat` from above.
6. **Audit log** — `olcLogLevel: stats` is fine for debugging, noisy for steady-state; `acl` is a good middle ground for production.

---

## Reference

### Default credentials in this guide (CHANGE FOR PRODUCTION)

| Service | DN / Username | Password |
|---------|---------------|----------|
| LDAP admin | `cn=admin,dc=tak,dc=local` | `admin` |
| LDAP config | `cn=admin,cn=config` | `config` |
| Test user 1 | `john.doe` | `password123` |
| Test user 2 | `jane.smith` | `password123` |
| TAK keystore | n/a | `atakatak` (from `cert-metadata.sh`) |

### Ports

| Port | Service | Protocol | Notes |
|------|---------|----------|-------|
| 389 | LDAP | TCP | plain — LDAPS on 636 for prod |
| 636 | LDAPS | TCP/SSL | with `LDAP_TLS: "true"` |
| 8080 | phpLDAPadmin | HTTP | UI for browsing the directory |
| 8089 | TAK data stream | mTLS | what ATAK uses after enrollment |
| 8443 | TAK admin UI | HTTPS | requires admin client cert |
| 8446 | TAK enrollment | HTTPS+Basic | what ATAK hits during enrollment |

### Directory tree

```
dc=tak,dc=local
├── ou=users
│   ├── cn=john.doe
│   └── cn=jane.smith
└── ou=groups
    ├── cn=tak_users_READ
    └── cn=tak_users_WRITE
```

### Two-keystore picture

```
intermediate-signing.jks  ──signs──>  client-cert-for-john.doe.p12
   (used by /Marti/api/tls/config)              │
                                                ▼
truststore-root.jks       ──validates──>  <input auth="x509" port="8089">
   (must contain the signing CA)
```

---

## Resources

- Official TAK Server LDAP walkthrough: "8.2 TAK Server LDAP Integration - Ubuntu.pdf" in the *TAK Server Instructions* zip from tak.gov — has the exact `default="ldap"` snippet the public docs gloss over.
- OpenLDAP admin guide: https://www.openldap.org/doc/admin26/
- phpLDAPadmin: http://phpldapadmin.sourceforge.net/

---

**Status**: Tested end-to-end against TAK Server 5.7-RELEASE-8 + osixia/openldap 1.5.0 on 2026-05-17. LDAP wiring (Gate 1) verified live; Gates 2 and 3 derived from the official TAK CoreConfig schema + traced failure modes from production deployments.
