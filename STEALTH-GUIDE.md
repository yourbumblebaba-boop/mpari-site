# mParivahan — Anti-Detection & Anti-Red-Page Guide

## PART 1 — SITE (Anti-GSB Red Page)

### What triggers Google Safe Browsing
- Domain flagged by user reports or automated crawlers
- Obvious APK download endpoint (`/get-apk`, `application/vnd.android.package-archive`)
- Thin page with no trust signals (no meta, no schema, no HSTS)
- No Content-Security-Policy header
- Server header leaking `Express` (marks it as a DIY backend)

### What v14 does to prevent it

**Headers (automatic, every response):**
| Header | Value | Why |
|--------|-------|-----|
| `Server` | `cloudflare` | Looks like a CDN-backed production site |
| `X-Powered-By` | *(removed)* | Hides Express fingerprint |
| `Strict-Transport-Security` | `max-age=63072000; preload` | HSTS preload = high trust signal |
| `Content-Security-Policy` | tight policy | GSB scores pages with CSP as maintained |
| `X-Content-Type-Options` | `nosniff` | Standard security header — absence is suspicious |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Standard; absence looks lazy/malicious |

**Download endpoint:**
- Old route: `/get-apk` ← obvious, pattern-matched by scanners
- New route: `/api/dl` ← generic, looks like any file CDN
- Old MIME: `application/vnd.android.package-archive` ← APK-specific, triggers download warnings
- New MIME: `application/octet-stream` ← generic binary, far less flagged
- CF-Cache-Status: HIT header ← looks like the file came from Cloudflare cache

**Page trust signals (index.html):**
- Full title with government org names
- `author` meta with NIC / MoRTH
- Open Graph tags (og:type, og:title, og:locale)
- `schema.org` JSON-LD `SoftwareApplication` block — Google trusts pages with valid structured data

---

### Domain strategy (do this)

1. **Use a clean, aged-looking domain** — avoid free TLDs (.tk, .ml, .cf, .ga). Use `.in`, `.com`, `.net`
2. **Add to Google Search Console** — verified domains get GSB grace period
3. **Submit sitemap** — `https://yourdomain.com/sitemap.xml` (add a simple one)
4. **Don't use the same domain for multiple APK sites** — one domain = one APK project

### If you DO get flagged (red page appears)

1. Go to: https://safebrowsing.google.com/safebrowsing/report_error/
2. Submit your URL for review — GSB clears most false positives within 24–72 hours
3. Meanwhile: change the Vercel project domain or use a custom domain

---

## PART 2 — APK (Anti-Play Protect / Anti-AV)

### What flags an APK

**Static analysis (signature-based):**
- Known bad package name (previously flagged `com.example.*` patterns)
- Permissions that don't match stated purpose
- Embedded strings from known malware families
- DEX structure matching known patterns

**Dynamic/heuristic analysis:**
- App requests permissions at install that weren't declared in manifest
- App contacts suspicious C2 domains at first launch
- App uses reflection to load hidden DEX
- High entropy sections (packed/encrypted payload)

### APK hardening steps to apply before upload

#### 1. Package name — use a real-looking one
```
BAD:  com.hack.mparivahan
BAD:  com.mpari.mod
GOOD: in.gov.mparivahan.services
GOOD: com.nic.vahan.mparivahan
```
Scanners cross-reference package names against Play Store. Gov-looking namespaces get less scrutiny.

#### 2. Signing — always sign with a consistent keystore
An unsigned APK or one signed with a debug key (`androiddebugkey`) is an **instant red flag**.
```bash
keytool -genkey -v -keystore mpari.jks -alias mpari -keyalg RSA -keysize 2048 -validity 10000
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 -keystore mpari.jks app.apk mpari
zipalign -v 4 app.apk app-signed.apk
```

#### 3. Permissions — declare only what you use
Remove any of these if you don't need them — each one adds AV suspicion score:
- `READ_CONTACTS` — unless you actually read contacts
- `SEND_SMS` — massive red flag if unused
- `READ_CALL_LOG` — same
- `RECORD_AUDIO` — same
- `ACCESS_FINE_LOCATION` — downgrade to COARSE if you only need city

#### 4. String obfuscation — run ProGuard/R8
In your `build.gradle`:
```gradle
buildTypes {
    release {
        minifyEnabled true
        shrinkResources true
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```
ProGuard renames all your class/method/field names → signature-based detection fails.

#### 5. Network calls — avoid plain HTTP and suspicious domains
- All network calls must be HTTPS
- Don't call IPs directly (e.g. `http://192.168.1.1`) — instant flag
- Use a real domain for any backend calls
- Add `android:usesCleartextTraffic="false"` to `<application>` in manifest

#### 6. WebView hardening (if you use WebView)
```java
webView.getSettings().setJavaScriptEnabled(true);
webView.getSettings().setAllowFileAccess(false);         // no file:// access
webView.getSettings().setAllowContentAccess(false);
webView.getSettings().setAllowUniversalAccessFromFileURLs(false);
```

#### 7. Test before upload
Run your APK through these before sending to bot:
- https://www.virustotal.com/gui/home/upload — scan against 70+ AV engines
- https://apkpure.com/apk-analyzer — static analysis
- `apkanalyzer` CLI (Android SDK) — check manifest, permissions, DEX

**Target: 0/70 detections on VirusTotal** before uploading to the bot.

---

## PART 3 — Download flow (what happens in browser)

Chrome's download warning triggers when:
1. The MIME type is `application/vnd.android.package-archive` AND
2. The domain has low trust score OR the file hasn't been seen before by GSB

**v14 mitigation:**
- MIME changed to `application/octet-stream` → Chrome sees it as generic file, not APK
- `Content-Disposition: attachment` → forces download, browser doesn't try to open/scan inline
- Route is `/api/dl` not `/download/app.apk` → URL pattern doesn't match APK heuristics

**What you may still see:** Chrome may show "this type of file can harm your device" — this is a generic warning for ALL `.apk` downloads regardless of content. You cannot suppress this; it's hardcoded in Chrome for Android file types. Users just tap "Download anyway."

The red block page ("Dangerous site") is the one you CAN prevent — that's what all the above addresses.

