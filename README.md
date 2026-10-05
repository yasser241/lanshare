# LanShare — share a folder over your local network (Android, single APK)

Pick a folder (or all storage volumes) on the phone, press **Start**, and anyone on the same Wi-Fi / hotspot can open
`http://<phone-ip>:<port>` in a browser to browse, download and (if you allow it) upload, create folders, rename and delete.
The web UI is **Persian / RTL by default** (English toggle), responsive, and fully embedded in the APK — no CDN, no internet.

The APK needs nothing else installed on the phone (no Java, Python, WebView component, etc.). Min Android 8.0 (API 26).

## What is different from the original (Windows-style) spec

| Spec item | Android equivalent |
|---|---|
| Run as administrator / UAC | Not a thing on Android. On first launch the app asks for the permissions it needs (All-files access, notifications). |
| Add a firewall rule for the port | Android has no per-app firewall; binding the port is enough. The app shows a hint (client/AP isolation on the router) and a *Wi-Fi settings* menu entry. |
| Drives (C:, D:…) | Storage volumes: internal storage + SD card/USB. Choose **One folder** or **All drives** (each volume appears as a top-level folder). |
| System tray | Foreground-service notification with a **Stop** button; sharing keeps running with the screen off (Wi-Fi + CPU locks). |
| Ports < 1024 | Not allowed on Android; the app accepts 1024–65535 and has a **Free port** button. |
| Settings JSON / portable mode | `settings.json` + `logs/lanshare.log` in the app's private storage (no "next to the EXE" on Android). |
| Double-click the .exe | Tap the .apk to install (allow "install unknown apps" once). |

## Project layout

```
app/src/main/
  AndroidManifest.xml
  assets/web/            index.html, app.css, app.js  (embedded web UI)
  java/com/lanshare/
    MainActivity.kt, MainViewModel.kt, LanShareApplication.kt
    data/    Settings.kt (JSON store), AppLog.kt, ShareState.kt
    net/     Net.kt (LAN IPs, free port, LAN-only check), IpFilter.kt (allow-list / CIDR)
    server/  HttpServer.kt (streaming HTTP/1.1), ShareRoot.kt (path safety), Auth.kt, ShareHandler.kt (routes)
    service/ ShareService.kt (foreground service, locks, notification)
    ui/      Screens.kt (Compose UI), Qr.kt
  res/       strings (English + Persian), icons, theme
.github/workflows/build.yml   builds the APK in GitHub Actions
```

## Build the APK

**Option A – Android Studio (easiest):** open this folder → wait for Gradle sync → *Build ▸ Build Bundle(s)/APK(s) ▸ Build APK(s)*.

**Option B – command line** (JDK 17 + Android SDK with platform 34; set `ANDROID_HOME`):
```
./gradlew assembleRelease        # Windows: gradlew.bat assembleRelease
```
Output: `app/build/outputs/apk/release/app-release.apk`

**Option C – no local tooling:** push this folder to a GitHub repo; the *build-apk* workflow produces the APK as a downloadable artifact (Actions ▸ latest run ▸ Artifacts ▸ LanShare-apk).

The release build is signed with the debug key so it installs directly. To publish it properly, create your own keystore and replace
`signingConfig` in `app/build.gradle.kts`.

## Use

1. Install the APK, open **LanShare**, grant *All files access* (and notifications on Android 13+).
2. **Share** tab → choose *One folder* (tap *Select folder*) or *All drives*, set the port, tap **Start sharing**.
3. The app lists the URLs (e.g. `http://192.168.1.23:8080`) with **Copy**, **Open in browser** and a **QR code**.
4. **Access** tab → read-only / read+upload / full management, optional password, optional IP allow-list
   (`192.168.1.20, 192.168.1.0/24`), max upload size. Changes apply on the next start.
5. **Log** tab shows connections, downloads, uploads, blocked requests and errors (also saved to a log file).

## Test

*Same phone:* Start sharing, tap **Open in browser** on the URL.
*Another device:* join the same Wi-Fi (or the phone's hotspot), open the URL or scan the QR code. Try download, multi-file upload, drag & drop (desktop browsers), pause Wi-Fi mid-upload (it resumes from the last chunk).
*Security checks* (replace IP/port; run from a PC on the LAN, mode = Full management only for the last one):
```
curl --path-as-is "http://IP:8080/dl?path=../../../../etc/hosts"      # -> 400
curl --path-as-is "http://IP:8080/dl?path=%2e%2e/%2e%2e/etc/hosts"    # -> 400
curl "http://IP:8080/api/list?path=/&x=1"                             # lists only the shared folder
curl -X PUT "http://IP:8080/api/upload?path=/&name=a.txt&total=1&offset=0" -d x   # -> 403 (no X-Requested-With) / 403 in read-only mode
```
*Port busy:* start sharing on a port already in use → a red message appears; tap **Free port**.
*Stop:* tap **Stop sharing** (or the notification's Stop) → the port closes (`curl` now refuses the connection).

## Security design

- **Path traversal:** web paths are split into segments; `..`, backslashes and control chars are rejected (400). Each resolved path is canonicalised (symlinks resolved) and must remain inside the shared folder/volume, otherwise 404. Symlinks pointing outside are hidden from listings and cannot be read, written or deleted through; recursive delete never follows links.
- **No execution:** downloads are always `application/octet-stream` + `attachment` + `nosniff`; there is no code path that runs commands or serves content inline. CSP forbids inline scripts.
- **LAN only:** the server accepts only loopback / private (RFC 1918) / link-local clients and refuses everything else, even if the phone is reachable from outside. Optional IP allow-list on top (a malformed list denies everyone).
- **Password:** PBKDF2-HMAC-SHA256 (120k iterations, random salt) — the plain password is never stored. Sessions use random 256-bit tokens in `HttpOnly; SameSite=Strict` cookies (8 h). 5 wrong attempts lock that IP for 60 s.
- **CSRF:** all state-changing requests need the `X-Requested-With: LanShare` header (cross-site pages cannot send it).
- **Large files:** everything streams (64 KB buffers). Uploads are 4 MB chunks written to `name.<size>-<mtime>.lspart` and renamed when complete, so an interrupted upload resumes where it stopped. A max upload size can be enforced; free space is checked.

## Known limitations (honest list)

- The server speaks plain **HTTP**, not HTTPS: on an untrusted Wi-Fi the password and files can be sniffed. Use it on networks you trust (or your own hotspot). Browsers would also warn about a self-signed certificate on LAN IPs.
- **Download progress** is the browser's own download manager (no custom progress bar); upload progress is shown in the page.
- Dropping a *folder* onto the page isn't supported (select files; create folders with the button).
- Two people uploading the same file name with identical size/mtime at the same moment can interfere with each other.
- A check-then-use race on symlinks (TOCTOU) is theoretically possible if another app swaps a link during a request.
- Android 11+ blocks `Android/data` and `Android/obb` of other apps even with All-files access.
- This code was written without access to an Android SDK in the authoring environment; it has not been compiled or run on a device. Expect to fix small compile errors on the first build.
