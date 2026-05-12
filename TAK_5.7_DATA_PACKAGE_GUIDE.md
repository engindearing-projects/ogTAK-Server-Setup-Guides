# TAK 5.7 Data Packages — Sim, Emulator, and Phone

Companion to `TAK_5.7_DOCKER_DELTAS.md`. Once the server is up on 8089
with `auth="x509"`, the question is how to get a client connected. The
**Mission Data Package** (`.zip`) is the one-tap path — drops the
connect string, the client cert, and the truststore in a single
import — and the official guides under-document it for the three
contexts a typical developer needs side-by-side: an iOS Simulator,
an Android emulator, and a real phone on the LAN.

This is the procedure that worked end-to-end on 2026-05-06 against
`takserver-docker-5.7-RELEASE-8` running on Colima on macOS, with
OmniTAK-iOS 2.18.0 (sim) and OmniTAK-Android 0.5.0 (emulator + phone)
as the clients.

Staging directory used: `~/Projects/tak57-test/`.

---

## 1. What's in a data package

A TAK data package is a regular `.zip` with two top-level entries:

```
tak57-mtls-<context>.zip
├── MANIFEST/
│   └── manifest.xml          ← package metadata, UID, contents list
└── certs/
    ├── server.pref           ← cot_streams + cert location pointers
    ├── <user>.p12            ← client cert (engie-test / -android / -phone)
    └── truststore-root.p12   ← server CA trust
```

TAK clients (ATAK, OmniTAK, WinTAK) all recognise this layout. Import
sets the `cot_streams` entry, attaches the client cert, trusts the
server's CA, and connects — no manual form-filling.

---

## 2. The `MANIFEST/manifest.xml` schema

```xml
<MissionPackageManifest version="2">
  <Configuration>
    <Parameter name="uid" value="884AE09D-9469-4048-9E4F-A4A40D65880C"/>
    <Parameter name="name" value="TAK_5.7_iOS_Sim"/>
    <Parameter name="onReceiveDelete" value="true"/>
  </Configuration>
  <Contents>
    <Content ignore="false" zipEntry="certs/server.pref"/>
    <Content ignore="false" zipEntry="certs/engie-test.p12"/>
    <Content ignore="false" zipEntry="certs/truststore-root.p12"/>
  </Contents>
</MissionPackageManifest>
```

| Field | Meaning |
|---|---|
| `uid` | A stable UUID for the package. **Must differ between contexts** — TAK keys imports by this UID and a second package with the same UID is silently treated as an update. Use `uuidgen` per context. |
| `name` | Display label in the TAK server-list. Use something that tells you-from-the-future which connection this is (`TAK_5.7_iOS_Sim`, `TAK_5.7_Android_Emu`, `TAK_5.7_Phone`). |
| `onReceiveDelete` | `true` = delete the package zip after a successful import. Recommended; otherwise the file lingers in the device's Downloads. |
| `<Content zipEntry="…">` | One per file. Path is relative to the zip root. |

---

## 3. The `certs/server.pref` schema

```xml
<?xml version="1.0" encoding="ASCII" standalone="yes"?>
<preferences>
  <preference version="1" name="cot_streams">
    <entry key="count" class="class java.lang.Integer">1</entry>
    <entry key="description0" class="class java.lang.String">TAK_5.7_iOS_Sim</entry>
    <entry key="enabled0" class="class java.lang.Boolean">true</entry>
    <entry key="connectString0" class="class java.lang.String">localhost:8089:ssl</entry>
  </preference>
  <preference version="1" name="com.atakmap.app_preferences">
    <entry key="displayServerConnectionWidget" class="class java.lang.Boolean">true</entry>
    <entry key="caLocation" class="class java.lang.String">cert/truststore-root.p12</entry>
    <entry key="caPassword" class="class java.lang.String">atakatak</entry>
    <entry key="clientPassword" class="class java.lang.String">atakatak</entry>
    <entry key="certificateLocation" class="class java.lang.String">cert/engie-test.p12</entry>
  </preference>
</preferences>
```

**Gotcha — `cert/` vs `certs/` path mismatch:** the zip has the certs
under `certs/` (plural) but the pref points to `cert/` (singular).
TAK clients normalise this on import, so the mismatch is intentional
and matches what `TakDataPackage` generator scripts emit. Don't "fix"
it — clients that follow ATAK convention expect the singular form.

---

## 4. The three contexts — what changes

The **only** field that differs between sim, emulator, and phone is
the `connectString0` host. Everything else (port, protocol, cert,
truststore, passwords) is identical.

| Context | `connectString0` | Why |
|---|---|---|
| **iOS Simulator** | `localhost:8089:ssl` | Sim shares the host's loopback. `127.0.0.1` works equivalently. |
| **Android Emulator** | `10.0.2.2:8089:ssl` | AVD-specific magic IP that maps to the host's loopback from inside the emulator. `localhost` from within the emulator would mean the emulator itself. |
| **Real phone on LAN** | `<host-lan-ip>:8089:ssl` | e.g. `192.168.1.80:8089:ssl`. Use `ifconfig` / System Settings → Network on the host. Phone and host must be on the same subnet (or the server reachable through a route). |

**Cert-name convention:** use a different client cert per context
(`engie-test.p12` / `engie-android.p12` / `engie-phone.p12`). TAK
server rejects two simultaneous connections from the same client UID,
so identical certs across contexts will silently disconnect whichever
peer connected second the moment the third connects. Different certs
= different UIDs = all three can be online simultaneously.

---

## 5. Generating the certs

Done once, from inside the TAK Server container or the unpacked
`tak/certs/` tree:

```bash
export STATE=Washington
export CITY=Spokane
export ORGANIZATION=Engindearing
export ORGANIZATIONAL_UNIT=TAKServer

cd tak/certs
./makeRootCa.sh                   # interactive — choose a CA name
./makeCert.sh server takserver    # server cert; CN=takserver
./makeCert.sh client engie-test
./makeCert.sh client engie-android
./makeCert.sh client engie-phone
```

This produces `engie-{test,android,phone}.p12` (password `atakatak`
by default) and a matching `truststore-root.p12` for the CA chain.
Copy them to `~/Projects/tak57-test/<context>-package/certs/`.

---

## 6. Building each package

```bash
cd ~/Projects/tak57-test

# iOS Sim
mkdir -p ios-package/MANIFEST ios-package/certs
cat > ios-package/MANIFEST/manifest.xml <<'EOF'
<MissionPackageManifest version="2">
  <Configuration>
    <Parameter name="uid" value="$(uuidgen)"/>
    <Parameter name="name" value="TAK_5.7_iOS_Sim"/>
    <Parameter name="onReceiveDelete" value="true"/>
  </Configuration>
  <Contents>
    <Content ignore="false" zipEntry="certs/server.pref"/>
    <Content ignore="false" zipEntry="certs/engie-test.p12"/>
    <Content ignore="false" zipEntry="certs/truststore-root.p12"/>
  </Contents>
</MissionPackageManifest>
EOF
# write certs/server.pref with connectString0=localhost:8089:ssl
# (paste the server.pref template from §3, swap `name` + `description0`)
cp ../client-bundle/engie-test.p12       ios-package/certs/
cp ../client-bundle/truststore-root.p12  ios-package/certs/

cd ios-package
zip -r ../tak57-mtls-sim.zip MANIFEST certs
cd ..
```

Repeat for `android-package/` with `connectString0=10.0.2.2:8089:ssl`
and `engie-android.p12`, and `phone-package/` with
`connectString0=<host-lan-ip>:8089:ssl` and `engie-phone.p12`.

**Important — re-roll the UUID:** the heredoc above uses `$(uuidgen)`
but the shell won't expand it inside `'EOF'` quoting. Either drop the
single quotes (and escape `$`/backticks in the XML accordingly) or
run `sed -i '' "s|UID_PLACEHOLDER|$(uuidgen)|" MANIFEST/manifest.xml`
after writing the file with a placeholder.

---

## 7. Importing into the client

### iOS Simulator
1. From the host: drag `tak57-mtls-sim.zip` onto the sim window
   (drops it into the iOS Files app under On My iPhone → Downloads).
2. In Files: tap the zip → Share → "Copy to OmniTAK" (or
   "Open with…" → OmniTAK Mobile).
3. The import sheet shows the package contents; tap Import.
4. Settings → TAK Servers — the `TAK_5.7_iOS_Sim` entry should be
   green within a few seconds.

### Android Emulator
```bash
adb -s emulator-5554 push ~/Projects/tak57-test/tak57-mtls-emu.zip /sdcard/Download/
```
Then on the emulator: Files app → Downloads → tap the zip → Open
with OmniTAK → Import. Same green indicator within a few seconds.

### Real phone (Android or iOS)
- **AirDrop** the `.zip` to the iPhone (lands in Files → Downloads).
- Or **email** / Signal / `scp` to the device.
- Tap → Open with OmniTAK → Import.

---

## 8. Quick verification

```bash
# Inside the server's takserver container:
docker exec takserver tail -f /opt/tak/logs/takserver-messaging.log \
  | grep -E "Connected|Accepted|client cert"
```

A successful client connection logs:
```
Accepted SSL connection from <peer-ip>
client cert CN=engie-test, O=Engindearing
```

If you see `bad certificate` or `unable to find valid certification path`,
the truststore in the data package doesn't include the CA that signed
the server cert — re-export `truststore-root.p12` after `makeRootCa.sh`
and rebuild the package.

---

## 9. What this NOT replace

- **CSR enrollment over port 8446** — for clients that want to mint
  their own cert in-app rather than receive one pre-baked. Covered
  in `OMNITAK_TEST_GUIDE.md` §B and in OmniTAK's `tak://enroll` deep
  link. Use that path when distributing OmniTAK to people who
  shouldn't see your CA's private key bundle.
- **Federation between servers** — see `FEDERATION_SETUP.md`.
- **LDAP-backed auth instead of x509** — see `LDAP_SETUP_GUIDE.md`.
  Data packages still work the same way; only the server's
  `auth="…"` attribute changes.

---

## Tested with

- `takserver-docker-5.7-RELEASE-8` on Colima 0.9.x (Docker 29.4.2)
- macOS 25 (Tahoe), Apple Silicon
- OmniTAK-iOS 2.18.0 (build 26051104) on iPhone 17 Pro sim (iOS 26.2)
- OmniTAK-Android 0.5.0 (vc42) on `engie_emulator` AVD
- Bundle staging at `~/Projects/tak57-test/`
- All three contexts connected concurrently with no UID collisions
