# TAK Server 5.7 Docker — Deltas vs the 5.5 Guide

Captured live during a fresh install of `takserver-docker-5.7-RELEASE-8.zip` on
**macOS 25 (Tahoe) on Apple Silicon** with **Colima** as the Docker engine.

These are the things the existing `TAK_SERVER_DOCKER_GUIDE.md` (which targets
5.5 + Docker Desktop) gets wrong, doesn't cover, or could improve for 5.7.

Ship date of test: **2026-05-06**.

---

## 1. Package format change — `.zip`, not `.tar.gz`

**Guide says:**
```bash
tar -xzf takserver-docker-5.5-RELEASE-58.tar.gz
```

**5.7 reality:**
```bash
unzip takserver-docker-5.7-RELEASE-8.zip
```

The 5.7 Docker package from tak.gov ships as a regular ZIP. Layout inside
matches 5.5 (`docker/` + `tak/`).

**Action:** Update the "Extract and Setup" section to detect or document both
formats. Recommend documenting `.zip` as the current default with a note that
older releases used `.tar.gz`.

---

## 2. Container engine alternatives are non-trivial on macOS

**Guide says:**
> Install Docker Desktop from https://www.docker.com/products/docker-desktop/

**Reality on this machine:**
- No Docker Desktop installed.
- `colima` and `orbstack` symlinks present, neither working out-of-the-box.
- Path forward used: Colima.

**What had to happen to get Colima working:**

```bash
# 1. Install the docker CLI itself (Colima doesn't bundle it)
brew install docker

# 2. Install the docker compose v2 plugin AND link it where docker looks for plugins
brew install docker-compose
mkdir -p ~/.docker/cli-plugins
ln -sf "$(brew --prefix)/bin/docker-compose" ~/.docker/cli-plugins/docker-compose

# 3. Install buildx, same plugin treatment
brew install docker-buildx
ln -sf "$(brew --prefix)/opt/docker-buildx/bin/docker-buildx" ~/.docker/cli-plugins/docker-buildx

# 4. Strip leftover osxkeychain credential-helper config
#    (otherwise: "docker-credential-osxkeychain: executable file not found")
python3 -c "import json; c=json.load(open('$HOME/.docker/config.json')); c.pop('credsStore', None); c.pop('credHelpers', None); json.dump(c, open('$HOME/.docker/config.json','w'), indent=2)"

# 5. Boot a Colima VM with TAK-suitable resources
colima start --cpu 4 --memory 8 --disk 40
```

**Action:** Add a new "macOS without Docker Desktop" section right after the
"Install Docker" section in the guide. The current guide assumes Docker Desktop
is the only path on macOS — that's not true for many devs anymore (license
changes pushed people to Colima/OrbStack).

---

## 3. CoreConfig.xml is now shipped, not blank

**Guide says (5.5):**
> The official TAK Server Docker package may include a minimal or empty
> CoreConfig.xml. If your file is missing, empty, or doesn't contain the
> `<connection>` section, create it using this template:
>
> [167 lines of XML to copy-paste]

**5.7 reality:** The package ships a complete `CoreConfig.example.xml`
(167 lines) at `tak/CoreConfig.example.xml`. There is no `tak/CoreConfig.xml`
present until you copy it.

**Correct flow for 5.7:**
```bash
cp tak/CoreConfig.example.xml tak/CoreConfig.xml
sed -i '' 's|password=""|password="atakatak"|' tak/CoreConfig.xml
```

**Action:** Replace the giant XML template in the guide with a `cp + sed`
patch instruction. The existing template can stay as a fallback for users
whose `.example.xml` is somehow missing, but it shouldn't be the primary path.

---

## 4. `clientAuth="false"` is no longer broken in 5.7

**Guide's UPDATE 1 (5.5):**
> TAK Server 5.5 does NOT accept `clientAuth="false"` — it must be removed
> entirely or set to valid enum values.
>
> ```bash
> # AFTER (CORRECT):
> # <connector port="8443" _name="https"/>
> # <connector port="8446" _name="cert_https"/>
> ```

**5.7 reality:** The default `CoreConfig.example.xml` ships with:

```xml
<connector port="8446" clientAuth="false" _name="cert_https"/>
```

…and the API service starts cleanly with this attribute present. The bug
documented in UPDATE 1 of the 5.5 guide is **fixed upstream** in 5.7.

**Action:** Mark UPDATE 1 of the existing guide as "5.5 only — fixed in 5.7."
Don't ask 5.7 users to strip `clientAuth="false"`.

---

## 5. DB username is `martiuser`, not `postgres`

**Guide's docker-compose.yml example:**
```yaml
takserver:
  environment:
    DB_USERNAME: postgres
```

**5.7 reality:**
- `tak/CoreConfig.example.xml` line 73 declares `username="martiuser"`
- `tak/db-utils/takserver-setup-db.sh` creates the `martiuser` PostgreSQL role
  automatically on first run, with the password it reads out of `CoreConfig.xml`
- The TAK Server's API service connects as `martiuser`, not `postgres`

**Correct compose env:**
```yaml
takserver:
  environment:
    DB_USERNAME: martiuser
    DB_PASSWORD: ${POSTGRES_PASSWORD:-atakatak}
```

The `tak-database` container can still keep `POSTGRES_PASSWORD` for the
postgres-superuser fallback path, but the `takserver` service should connect
as `martiuser`.

**Action:** Update the example `docker-compose.yml` in the guide.

---

## 6. Container init scripts are now self-contained

5.7 ships two key scripts that the guide doesn't mention:

- `tak/configureInDocker.sh` — `Dockerfile.takserver`'s ENTRYPOINT. Launches
  all five Java services (config, messaging, api, retention, plugin-manager)
  inside one container. No systemd, no manual `start.sh`.
- `tak/db-utils/configureInDocker.sh` — `Dockerfile.takserver-db`'s ENTRYPOINT.
  Runs `pg_ctl initdb`, copies `pg_hba.conf`, runs `takserver-setup-db.sh` (which
  creates `martiuser` + `cot` DB), and `SchemaManager.jar upgrade`.

**Implication for the guide:** The "Build and Start Services" section can
shrink — there is no manual schema apply step needed for fresh installs. The
DB container's first-boot script handles it. `SchemaManager.jar upgrade` is
still relevant for in-place version upgrades (e.g., 5.6 → 5.7 with a preserved
DB volume).

**Action:** Add a one-paragraph "How startup works inside the containers" note
so users aren't surprised. Trim the manual schema-apply step out of the
fresh-install path.

---

## 7. Build is much faster than the guide suggests

**Guide says (5.5):**
> This step downloads PostgreSQL 15, Java 17, and builds the containers
> (5–10 minutes)

**5.7 reality on this hardware (Apple Silicon, Colima, fresh image cache):**

```
build start: 11:56:23
build end:   11:57:24
total:       ~61 seconds
```

Both images built in just over a minute. Your mileage will vary by host hardware
and network speed for the base image pulls, but the "5–10 minutes" warning is
outdated for modern Apple Silicon + Colima + cached images.

**Action:** Soften the warning. "First build pulls postgres:15.1 + temurin:17
base images (~600 MB combined). On modern hardware with a fast connection,
expect 1–3 minutes; older hardware or slow networks may take 5–10."

---

## 8. Default cert metadata env vars

The cert generation scripts in `tak/certs/` (`makeRootCa.sh`, `makeCert.sh`)
read `STATE`, `CITY`, `ORGANIZATION`, `ORGANIZATIONAL_UNIT` env vars. The 5.5
guide tells users to set them, but doesn't note that **a missing env var
silently falls back to the value baked into `cert-metadata.sh`**, which is
empty by default and produces certs with empty subject fields.

**Best practice:**
```bash
export STATE=Washington
export CITY=Spokane
export ORGANIZATION=Engindearing
export ORGANIZATIONAL_UNIT=TAKServer
cd tak/certs
./makeRootCa.sh   # prompts for CA name interactively
./makeCert.sh server takserver
./makeCert.sh client admin
./makeCert.sh client engie-test    # for OmniTAK
```

**Action:** Make this the canonical block in the "Generate Certificates"
section. Note that `makeRootCa.sh` is interactive (asks for the CA name).

---

## 9. New 5.7 features the guide should mention

Carrying over from the 5.7 Ubuntu tutorial, but worth flagging in the Docker
guide:

- **WebTAK 4.10.5** — performance overhaul; supports up to ~2800 simultaneous
  clients (TPC benchmark on c5.2xlarge / 8 vCPU)
- **takproto over UDP** — UDP inputs now accept binary takproto messages.
  Add `protocol="udp"` + `proto="takproto"` (verify attribute against your
  shipped XSD) for low-bandwidth radio gateways
- **Optional broadcast filter** — admins can stop specific users from
  broadcasting map items; configured in `CoreConfig.xml`
- **Websocket compression default flipped to OFF** — re-enable in
  `<websocket compressionEnabled="true">` if your CDN/proxy expects compressed
  frames

**Action:** Cross-reference the 5.7 Ubuntu tutorial's "What's new" section
from the Docker guide so users know what they're getting.

---

## 10. Verified working ports + smoke tests

After the stack is up, here's the canonical health check used during this run:

```bash
# Confirm both containers running
docker compose ps

# Confirm all 5 Java microservices started
docker exec takserver ps -ef | grep -E "java" | grep -v grep
# Should show: config, messaging, api, retention, takserver-pm

# Confirm listening ports inside container
docker exec takserver netstat -tlnp | grep -E "8089|8443|8446"

# From host, confirm 8443 demands a client cert (good signal)
curl -k -m 3 https://localhost:8443/
# Expected: "bad certificate" / SSL alert  ← server is up, just wants admin cert

# Confirm 8446 is alive (cert enrollment endpoint)
curl -k -m 3 -o /dev/null -w "%{http_code}\n" https://localhost:8446/
# Expected: 403 (endpoint up; rejects unauthenticated GET)
```

**Action:** Add this as a "Quick health check" section to the guide; it's
faster and more reliable than chasing log strings.

---

## 11. Volume name change

When migrating from a 5.5 docker-compose to 5.7, note the **named volume**
gets a new name based on the project directory. The 5.5 update doc references
`takserver-docker_tak-db-data` for volume removal during a schema reset; on
5.7 with the directory `takserver-docker-5.7-RELEASE-8/`, the actual name is
`takserver-docker-57-release-8_db-data`.

**Always look it up dynamically:**
```bash
docker volume ls | grep tak
```

**Action:** Update the troubleshooting "Database Schema Not Initialized"
section to use a `docker volume ls` lookup rather than a hardcoded name.

---

## OmniTAK-Android side-load procedure (worked first try)

The cert bundle is at `~/Projects/tak57-test/client-bundle/`:

```
ca.pem
engie-test.p12         (password: atakatak)
truststore-root.p12    (password: atakatak)
```

Push to a running Android emulator:

```bash
adb push ~/Projects/tak57-test/client-bundle/engie-test.p12 /sdcard/Download/
adb push ~/Projects/tak57-test/client-bundle/ca.pem /sdcard/Download/
```

Then in OmniTAK-Android:
1. **Settings → TAK Servers → +**
2. Address: `10.0.2.2` (emulator → host loopback) or `192.168.1.80` (host LAN)
3. Port: `8089`
4. Protocol: SSL/TLS
5. Client cert: `/sdcard/Download/engie-test.p12`
6. Cert password: `atakatak`
7. Save → connect → indicator should turn green

**Note:** If using `192.168.1.80` (host LAN IP), the server's cert is signed
to CN=`takserver`, not to the IP. OmniTAK will need its hostname-verification
relaxed or the cert regenerated to include the IP as a SAN. For emulator
testing, `10.0.2.2` is more reliable.

---

## Summary — patch list for `TAK_SERVER_DOCKER_GUIDE.md`

1. ✏️ Replace `tar -xzf` with `unzip` for 5.7+
2. ✏️ Add "macOS without Docker Desktop (Colima)" section
3. ✏️ Replace XML-template-construction with `cp CoreConfig.example.xml + sed`
4. ✏️ Mark UPDATE 1 (`clientAuth="false"`) as 5.5-only; fixed in 5.7
5. ✏️ Change `DB_USERNAME: postgres` → `DB_USERNAME: martiuser` in compose
6. ✏️ Add "How startup works inside the containers" paragraph
7. ✏️ Trim "5–10 minutes" build estimate
8. ✏️ Make the cert env-var export block the canonical pattern
9. ✏️ Cross-link the 5.7 "What's new" section (WebTAK 4.10.5, takproto-UDP, etc.)
10. ✏️ Add the "Quick health check" smoke test
11. ✏️ Update volume-name references to use `docker volume ls` lookup
12. ➕ Add the OmniTAK-Android side-load section

---

**Tested with:**
- `takserver-docker-5.7-RELEASE-8.zip`
- macOS 25 (Tahoe) on Apple Silicon
- Colima 0.9.x with Docker 29.4.2
- Bundle: ~/Projects/tak57-test/takserver-docker-5.7-RELEASE-8/
- Bring-up time: ~3 minutes total (extract + cert gen + build + boot)
