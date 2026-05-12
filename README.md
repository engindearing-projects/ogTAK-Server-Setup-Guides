# ogTAK Server Setup Guides

Practical, tested install guides for **TAK Server 5.5, 5.6, and 5.7** on Ubuntu 24.04 LTS — including federation, LDAP, Docker, and an end-to-end smoke test using OmniTAK.

> Latest TAK Server upstream tags (TAK-Product-Center/Server, April 2026):
> - **5.7-RELEASE-14** — newest, includes WebTAK 4.10.5 and major messaging perf wins
> - **5.6-RELEASE-35** — current stable
> - **5.5-RELEASE-82** — long-term stable; 5.5-RELEASE-58 (the original target of these guides) is superseded

Pick the version that matches the `.deb` you downloaded from [tak.gov](https://tak.gov).

---

## Which guide should I use?

| You want to... | Read this |
|---|---|
| Install the newest TAK Server | [TAK_SERVER_5.7_COMPLETE_TUTORIAL.md](TAK_SERVER_5.7_COMPLETE_TUTORIAL.md) |
| Install current stable | [TAK_SERVER_5.6_COMPLETE_TUTORIAL.md](TAK_SERVER_5.6_COMPLETE_TUTORIAL.md) |
| Install 5.5 (legacy) | [TAK_SERVER_5.5_COMPLETE_TUTORIAL.md](TAK_SERVER_5.5_COMPLETE_TUTORIAL.md) |
| Run TAK Server in Docker | [TAK_SERVER_DOCKER_GUIDE.md](TAK_SERVER_DOCKER_GUIDE.md) |
| 5.7-on-Docker deltas vs the 5.5 guide (Colima, `.zip` package, `martiuser`, etc.) | [TAK_5.7_DOCKER_DELTAS.md](TAK_5.7_DOCKER_DELTAS.md) |
| Build a one-tap `.zip` data package for sim / emulator / phone | [TAK_5.7_DATA_PACKAGE_GUIDE.md](TAK_5.7_DATA_PACKAGE_GUIDE.md) |
| Federate to OpenTAKServer / another TAK Server | [FEDERATION_SETUP.md](FEDERATION_SETUP.md) |
| Wire up OpenLDAP / Active Directory auth | [LDAP_SETUP_GUIDE.md](LDAP_SETUP_GUIDE.md) |
| Smoke-test the install with a real client | [OMNITAK_TEST_GUIDE.md](OMNITAK_TEST_GUIDE.md) |

---

## The same gotchas apply across 5.5 / 5.6 / 5.7

These bite on every fresh Ubuntu 24.04 install. None of them have been fixed upstream because they're really Ubuntu/packaging quirks, not TAK Server bugs:

### 1. PostgreSQL MUST be version 15

- TAK Server 5.5/5.6/5.7 all pin to **PostgreSQL 15** (verified in source — `requires('postgresql-15')`, `requires('postgresql-15-postgis-3')`)
- Ubuntu 24.04's default `apt install postgresql` installs **16**, which breaks TAK Server with cryptic DB errors
- Install 15 explicitly from the PGDG APT repo (every guide above shows the steps)

### 2. Keep OpenJDK 17 installed alongside Temurin

- The TAK Server `.deb` declares `requires('openjdk-17-jdk')` — don't remove it
- Run Temurin 17 as the runtime AND keep OpenJDK 17 to satisfy the package dependency
- Removing OpenJDK 17 will fail your `dpkg -i`

### 3. Federation Hub is a separate `.deb`

- `takserver-fed-hub_<version>_all.deb` — separate download from tak.gov
- The fed-hub scripts need PATH fixes for Ubuntu 24.04 (full paths to `/usr/bin/awk` and `/usr/bin/java`)
- See [FEDERATION_SETUP.md](FEDERATION_SETUP.md)

### 4. Re-apply launcher-script PATH fixes after every `dpkg -i`

- `dpkg` overwrites `/opt/tak/setenv.sh`, `takserver-messaging.sh`, etc.
- Each tutorial's "Fix Startup Scripts" section is the same fix — re-run it on upgrade

---

## Version-specific notes

### What's new in 5.7
- **WebTAK 4.10.5** — performance overhaul, scales to ~2800 clients on c5.2xlarge
- **takproto over UDP** inputs (previously TCP/TLS only) — useful for Meshtastic/LoRa gateways
- **Optional broadcast filter** to prevent specific users from broadcasting map items
- **Websocket compression off by default** for performance — re-enable in `CoreConfig.xml` if you need it for low-bandwidth clients

### What changed in 5.6 (vs 5.5)
- Spring Security bumped to 6.5.9 — re-test custom OAuth/SSO config
- Apache Commons Codec 1.15 → 1.17.1
- Spring LDAP 3.1.6 → 3.2.0 — re-test custom LDAP/AD bind
- `commons-beanutils` no longer pulled in by core
- OS deps unchanged (Java 17, PostgreSQL 15)

### 5.5 baseline
- See [TAK_SERVER_5.5_COMPLETE_TUTORIAL.md](TAK_SERVER_5.5_COMPLETE_TUTORIAL.md)

---

## Tested environments

- **OS**: Ubuntu 24.04.3 LTS (Noble Numbat)
- **Java**: Eclipse Temurin 17.0.16 + OpenJDK 17 (both required)
- **PostgreSQL**: 15.x with PostGIS 3
- **TAK Server**: 5.5-RELEASE-82, 5.6-RELEASE-35, 5.7-RELEASE-14
- **Federation Hub**: matching `<version>-RELEASE-<n>` to the server
- **Clients**: OmniTAK-Android (Kotlin/Compose), OmniTAK-iOS (Swift/SwiftUI), ATAK 5.5+, WinTAK

---

## Port reference (all versions)

### TAK Server core

| Port | Protocol | Purpose |
|------|----------|---------|
| 5432 | TCP | PostgreSQL 15 |
| 8089 | TCP/TLS | Client connections (ATAK / WinTAK / OmniTAK) |
| 8443 | TCP/HTTPS | Web admin / WebTAK (client cert required) |
| 8444 | TCP/HTTPS | Federation HTTPS |
| 8446 | TCP/HTTPS | Certificate enrollment |
| 6969 | UDP | Multicast SA discovery |

### Federation Hub

| Port | Protocol | Purpose |
|------|----------|---------|
| 9000 | TCP/TLS | Federation V1 (legacy) |
| 9001 | TCP/TLS | Federation V2 server |
| 9100 | HTTPS | Federation Hub web UI |
| 9101 | TCP/TLS | Fed Hub V1 broker |
| 9102 | TCP/TLS | Fed Hub V2 broker |

---

## Quick start

1. Download the TAK Server `.deb` from [tak.gov](https://tak.gov) (registration required)
2. Read the gotchas above
3. Follow the matching `TAK_SERVER_5.X_COMPLETE_TUTORIAL.md` top-to-bottom
4. Verify with [OMNITAK_TEST_GUIDE.md](OMNITAK_TEST_GUIDE.md)
5. Optionally add federation, LDAP, or Docker

Time budget: 1–2 hours for a clean install on a fresh VM, plus 15 minutes for the OmniTAK smoke test.

---

## Security notes

- Default cert password is `atakatak` — change `cert-metadata.sh` (`CAPASS=` and `PASS=`) **before** running `makeRootCa.sh`
- Database password `takserver123` in the tutorials is a placeholder — replace it before exposing the server
- Never commit certificates or private keys
- Review firewall rules before exposing the server to the internet
- For production, consider running behind a reverse proxy and fronting it with proper CA-issued certs (the self-signed CA is fine for internal use)

---

## Package downloads

All TAK Server packages must be downloaded from [tak.gov](https://tak.gov) (free registration). This repo cannot redistribute them.

You'll typically need:

| Filename pattern | Purpose |
|---|---|
| `takserver_5.X-RELEASE<n>_all.deb` | Main server |
| `takserver-fed-hub_5.X-RELEASE<n>_all.deb` | Federation Hub (optional) |
| `takserver-docker-5.X-RELEASE-<n>.tar.gz` | Docker bundle (optional) |

---

## Contributing

Issues, corrections, and improvements welcome. Please ensure:
- No sensitive data (certificates, keys, passwords)
- Commands tested on Ubuntu 24.04 (or the OS you're documenting)
- Steps are concrete and reproducible

---

## Disclaimer

This is an unofficial community guide. For official TAK Server documentation and support:
- https://tak.gov
- https://tak.gov/community
- https://github.com/TAK-Product-Center/Server

TAK Server is government software. All packages must be obtained from official sources.

---

## Version history

- **v2.0** — Added 5.6 + 5.7 install guides, OmniTAK end-to-end smoke test, refreshed README
- **v1.2** — Added Docker installation guide
- **v1.1** — Added Federation Hub setup with detailed gotchas
- **v1.0** — Initial 5.5 install guide for Ubuntu 24.04
