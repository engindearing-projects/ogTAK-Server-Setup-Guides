# Verifying TAK Server with OmniTAK

End-to-end smoke test for a fresh TAK Server install (5.5, 5.6, or 5.7) using **OmniTAK-Android** and **OmniTAK-iOS** as the test clients.

OmniTAK is an open-source TAK client built by Engindearing — Kotlin/Compose on Android, Swift/SwiftUI on iOS. Both speak Cursor-on-Target over TLS to any TAK Server, support `.zip` data packages and CSR enrollment on port 8446, and are licensed Apache 2.0.

- Android: https://github.com/engindearing-projects/OmniTAK-Android
- iOS: https://github.com/engindearing-projects/OmniTAK-iOS

---

## What this guide covers

1. Generating a client cert on the server
2. Getting it onto a phone (three methods)
3. Wiring OmniTAK to the server
4. Round-trip test: drop a marker → see it in WebTAK
5. Optional: data package enrollment via 8446
6. Optional: building the OmniTAK clients yourself with the right ATAK 5.5/5.6 toolchain
7. Troubleshooting

---

## Prerequisites

- TAK Server 5.5 / 5.6 / 5.7 installed and running (see the matching `TAK_SERVER_5.X_COMPLETE_TUTORIAL.md`)
- Server reachable from your phone (same Wi-Fi or routable VPN)
- Admin cert imported into your browser to view WebTAK at `https://YOUR_SERVER_IP:8443`

---

## 1. Generate a client cert

On the server:

```bash
cd /opt/tak/certs
sudo ./makeCert.sh client engie-test
```

This produces `/opt/tak/certs/files/engie-test.p12` (and matching `.pem`/`.jks`) using the default password `atakatak`.

> Change the default cert password in `/opt/tak/certs/cert-metadata.sh` (`CAPASS=` and `PASS=`) before generating production certs. Do this **before** `makeRootCa.sh`.

---

## 2. Get the cert onto your phone

Pick one:

### A — AirDrop / direct file transfer (simplest)

```bash
# macOS dev box, server reachable via SSH
scp user@YOUR_SERVER_IP:/opt/tak/certs/files/engie-test.p12 ~/Desktop/
```

Then AirDrop to iPhone or `adb push` to Android:

```bash
adb push ~/Desktop/engie-test.p12 /sdcard/Download/
```

### B — Data package over the network

Build a TAK data package and drop it in OmniTAK:

```bash
# On the server, from /opt/tak/certs/files
zip engie-test-pack.zip engie-test.p12 ca.pem
```

Add a small `manifest.xml` if you want OmniTAK to auto-prefill the connection settings — minimum example:

```xml
<?xml version="1.0" standalone="yes"?>
<MissionPackageManifest version="2">
  <Configuration>
    <Parameter name="uid" value="engie-test-pack"/>
    <Parameter name="name" value="Engie Test Pack"/>
  </Configuration>
  <Contents>
    <Content zipEntry="engie-test.p12"/>
    <Content zipEntry="ca.pem"/>
    <Content zipEntry="server.pref"/>
  </Contents>
</MissionPackageManifest>
```

Where `server.pref` is a TAK preferences file pointing at your server. iOS OmniTAK imports `.zip` packages via Files / AirDrop / iCloud Drive (see the [OmniTAK-iOS README](https://github.com/engindearing-projects/OmniTAK-iOS#getting-started)).

### C — CSR enrollment over port 8446 (no manual cert transfer)

If you set up CSR enrollment in `CoreConfig.xml`, the OmniTAK clients can request their own certificate from the server using a username/password. Both Android and iOS clients support this — point them at `https://YOUR_SERVER_IP:8446` in **Settings → TAK Servers → Enroll**.

This is the cleanest production path. Method A is fine for a smoke test.

---

## 3. Wire OmniTAK to the server

### OmniTAK-Android

1. Install the debug APK: `cd ~/Projects/OmniTAK-Android && ./gradlew installDebug`
2. **Settings → TAK Servers → +**
   - **Address**: `YOUR_SERVER_IP`
   - **Port**: `8089`
   - **Protocol**: `SSL/TLS`
   - **Client cert**: pick `engie-test.p12` from `/sdcard/Download/`
   - **Cert password**: `atakatak`
3. Save and connect — the connection indicator should turn green

### OmniTAK-iOS

1. Open `OmniTAKMobile.xcodeproj` in Xcode
2. Set your Team and Bundle Identifier in Signing & Capabilities
3. Build and run on a physical device or simulator
4. **Settings → TAK Servers → +** (or import the `.zip` data package via Files → Share → OmniTAK)
   - Same fields as above

---

## 4. Round-trip test

Once connected:

1. Long-press the map in OmniTAK → drop a marker (any CoT type)
2. In a browser already authenticated with the admin cert, open `https://YOUR_SERVER_IP:8443`
3. Navigate to **Marti → COT** (or open the WebTAK situation map)
4. The marker should appear within 1–2 seconds with the correct icon and timestamp

If the marker round-trips, the install is healthy: TLS works, the cert is trusted, the messaging service is forwarding CoT, and WebTAK can read the COT store.

### Bonus checks

- **Bidirectional**: drop a marker in WebTAK, confirm it shows up in OmniTAK
- **Multi-client**: install on a second phone, confirm both see each other
- **5.7 only**: if you opened a takproto-over-UDP input, send a CoT XML packet via `nc -u` and confirm it lands in the COT log (`/opt/tak/logs/takserver-messaging-*.log`)

---

## 5. Building OmniTAK with the right toolchain

OmniTAK currently builds against TAK Server 5.5-era client libs. If you're testing the parity story across server versions, here are the canonical ATAK plugin/build-chain specs from tak.gov so you can match the toolchain:

### ATAK 5.5 build chain

| Item | Version |
|------|---------|
| `minSdkVersion` | 21 |
| `compileSdkVersion` | 35 |
| `targetSdkVersion` | 35 |
| Gradle | 8.13 |
| Android Gradle Plugin | 8.9.0 |
| Java (build env) | Temurin JDK 17 |
| NDK | 25.1.8937393 |
| Kotlin (transitive) | 2.1.10 |
| `androidx.core` | 1.15.0 |
| `androidx.fragment` | 1.8.7 |
| `androidx.exifinterface` | 1.4.1 |
| `androidx.localbroadcastmanager` | 1.1.0 |
| `androidx.lifecycle:lifecycle-process` | 2.9.0 |
| `org.greenrobot:eventbus` | 3.2.0 |
| `com.squareup.okhttp3:okhttp` | 4.11.0 |
| `org.mockito:mockito-core` | 5.15.2 |

### ATAK 5.6 build chain

| Item | Version |
|------|---------|
| `minSdkVersion` | 21 |
| `compileSdk` | 36 |
| `targetSdkVersion` | 36 |
| Gradle | 8.14.3 |
| Android Gradle Plugin | 8.13.0 |
| Java (build env) | Temurin JDK 17 |
| NDK | 27.3.13750724 |
| Kotlin (transitive) | 2.2.0 |
| `androidx.core` | 1.17.0 |
| `androidx.fragment` | 1.8.9 |
| `androidx.exifinterface` | 1.4.1 |
| `androidx.localbroadcastmanager` | 1.1.0 |
| `androidx.lifecycle:lifecycle-process` | 2.9.4 |
| `org.greenrobot:eventbus` | 3.2.0 |
| `com.squareup.okhttp3:okhttp` | 4.11.0 |
| `org.mockito:mockito-core` | 5.20.0 |

### Source/target compatibility

Source/target is still `1.8` for ATAK plugin compatibility, but the build environment must be JDK 17 (https://adoptium.net).

### Common gotchas (5.5+)

- **`Context.registerReceiver`** — Android 14+ requires the `RECEIVER_EXPORTED` / `RECEIVER_NOT_EXPORTED` flag. Wrap with `Build.VERSION.SDK_INT >= TIRAMISU` check, or use ATAK's `AtakBroadcast` helper.
- **Package visibility** — `QUERY_ALL_PACKAGES` is removed; declare a faux query intent to allow ATAK to discover your component.
- **AndroidX duplicates** — exclude AndroidX deps that core ATAK already provides (see `configurations.implementation { exclude ... }` block in the ATAK plugin template).
- **Native libs** — set `android:extractNativeLibs="true"` if `minSdkVersion >= 26`.
- **`gradle.properties`** — set `android.bundle.enableUncompressedNativeLibs=false` for AAB native libs.
- **AAB → APK transparent signing** — drop `app/src/main/res/xml/com_android_vending_archive_opt_out.xml` with `<optOut />`.

### Current OmniTAK-Android settings

As of this writing, OmniTAK-Android targets:

```
compileSdk = 35
minSdk = 26
targetSdk = 35
JavaVersion = 17 (source/target)
kotlin jvmTarget = 17
Gradle wrapper = 8.11.1
```

To bring it to ATAK 5.6 parity, bump `compileSdk`/`targetSdk` to 36 in `app/build.gradle.kts` and `distributionUrl` in `gradle/wrapper/gradle-wrapper.properties` to `gradle-8.14.3-bin.zip`. OmniTAK isn't a plugin — it's a standalone client — so the AndroidX exclusion list doesn't apply; you can use the latest stable libs without coordinating with ATAK core.

### iOS

OmniTAK-iOS targets iOS 17.0 / Swift 5.9 per the README, though the current Xcode project file shows `IPHONEOS_DEPLOYMENT_TARGET = 15.0`. There's no equivalent ATAK 5.5/5.6 iOS toolchain matrix — iTAK is closed-source and ships through the App Store, so you can build iOS OmniTAK with whatever Xcode you have on hand. Xcode 15.4+ matches the README requirement.

---

## 6. Troubleshooting

### "Connection refused" / "Cannot connect"

```bash
# On the server
sudo systemctl status takserver-messaging
sudo netstat -plnt | grep 8089
sudo ufw status
```

If `8089` isn't listening, check `/opt/tak/logs/takserver-messaging-console.log` for the bind error. If it's listening but UFW is blocking, `sudo ufw allow 8089/tcp`.

### "Certificate is not trusted"

The TAK Server's CA is self-signed by default. OmniTAK should accept the cert chain bundled in the `.p12`, but if you see a trust error:

- Make sure you copied the full `.p12` (not just `.pem`) — the PKCS12 includes the chain
- Re-run `makeCert.sh client <name>` to regenerate fresh
- On Android, "Network security config" can block self-signed CAs in release builds — debug builds work; for release, add a `network_security_config.xml` that trusts user CAs

### "Marker doesn't appear in WebTAK"

```bash
tail -f /opt/tak/logs/takserver-messaging-console.log
```

If you see the inbound CoT logged but it's not in WebTAK:
- Hard-refresh the browser (Ctrl+Shift+R)
- On 5.7, websocket compression default flipped — confirm no proxy is forcing a stale assumption
- Check the TAK Server **Marti → Inputs** view for the right input bound to port 8089

### "OmniTAK keeps disconnecting"

- Battery optimization on Android: exclude OmniTAK from "Optimize battery"
- iOS: background socket lifetime is short — backgrounding the app for >30s drops the connection by design. Re-foreground to reconnect.
- Server-side: check `/opt/tak/logs/takserver-messaging-console.log` for client-side disconnect reasons (idle timeout, cert mismatch, etc.)

### Multicast discovery doesn't find the server

```bash
# On the server
sudo netstat -unlp | grep 6969
sudo ufw allow 6969/udp
```

Multicast across subnets requires IGMP-aware switches. If your phone and server aren't on the same broadcast domain, manually configure the server in OmniTAK instead of relying on auto-discovery.

---

## What "verified" means

After this guide, you've confirmed:

- TLS is correctly configured on port 8089
- Self-signed CA chain is reachable to clients
- CoT messages flow client → server → other clients
- WebTAK renders incoming CoT
- (5.7) WebTAK 4.10.5 loads and stays responsive

That's enough to declare the install good for development and small-team operational use. For production, also run the federation, LDAP, and load tests called out in the other guides.
