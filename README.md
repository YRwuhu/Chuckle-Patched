# Chuckle for X — Patched Build

Static-analysis notes and patched APKs for **Chuckle for Twitter/X 1.1.1**
(`app.jjyy.chuckle`, versionCode 154, arm64-v8a).

> **Please delete this repository and any downloaded files within 24 hours.**
> The APKs here contain the original developer's proprietary code and are
> redistributed strictly for private research and interoperability testing.
> No license is granted. If you are the rights holder, open an issue and the
> repository will be taken down immediately.

---

## Download

| File | Size | Description |
|---|---|---|
| `Chuckle-1.1.1-zh-v2.apk` | 56.6 MB | Patched + Chinese localization (recommended) |
| `Chuckle-1.1.1-patched.apk` | 55.6 MB | Patched only, English UI |

Both are **standalone single APKs** — arm64-v8a native libraries and the
`config.en` / `config.hdpi` split resources are already merged in. Do not
install the original split APKs alongside them.

### Requirements

- Android 12+ (`minSdk 32`)
- **arm64-v8a** device. There is no x86/x86_64 build — the upstream XAPK only
  shipped `config.arm64_v8a`. Emulators without ARM translation (e.g. default
  MuMu / BlueStacks x86 instances) will install but crash on `dlopen`.
- Uninstall any existing `app.jjyy.chuckle` first — the signature differs.

### Signature

Signed with a self-generated RSA-2048 key (v3 scheme). Certificate
SHA-256: `d2a09060893ec3e5d243088d7b3d1167d124362c24fcb20c015ef171d8dd285e`

This is **not** the Play Store signing key, which is why the APK cannot update
an existing installation and why Play Protect will warn about it.

---

## What was patched

### Premium / subscription gating

The app does not use Google Play billing validation for entitlement. It stores
the subscription record locally and evaluates it in-process.

| Class | Method | Change |
|---|---|---|
| `defpackage.ohj` | `oO0OO00()Z` | Global premium gate → always `true` |
| `defpackage.r6` | `oO0OOOO0()Lgw8;` | Tier (PAID/TRIAL/FREE) → always `PAID` |
| `defpackage.nhj` | status flags | Premium page shows *Activated*, not *In trial* |
| `defpackage.fgl` | `oO0OO00()Z` | APK signature self-check → always `true` |
| `defpackage.hh0` | `oO0OO00()Z` | Remote "disable UI" switch → always `false` |

`ohj.oO0OO00()` is the single choke point for ad removal and feature
unlocking. It is also the fallback for the two hardcoded developer account IDs
in `l3q`, so patching it covers every call site.

`nhj` computes three booleans from the local `SubscriptionsBean`:

```java
z2 = !expired      // activated
z3 = expired       // expired
z4 = freeTrial     // in trial
```

A fresh install calls `/v1/subscriptions/freeTrial` and stores a trial record,
so `z4` wins the display. All three are now pinned (`z2=1, z3=0, z4=0`).

### PairIP (Google license check)

The app shipped Google's PairIP layer
(`com.pairip.application.Application` → `LicenseClient.checkLicense`).
It runs a native probe from `libapkresolver.so` / `libnetbridge.so` that reads
`/proc/self/maps`, verifies APK signatures and scans for Frida artifacts.

Removed:
- `manifest`: `android:name` reverted to `app.jjyy.chuckle.MainApplication`
- `LicenseClient.checkLicense(Context)` → empty body
- `LicenseClient` and friends remain in the dex as dead code but are never
  reached.

### Packaging fixes

Two separate issues caused install failures and instant crashes:

1. **Compressed native libs.** The manifest declares
   `extractNativeLibs="false"`, which requires `.so` files to be STORED and
   page-aligned so they can be `mmap`ed directly. apktool recompressed all 23
   libraries into deflate, producing `INSTALL_FAILED_INVALID_APK` on MIUI and
   a `dlopen` failure anywhere. Fixed by adding `so` to `apktool.yml`'s
   `doNotCompress` list and running `zipalign -P 16`.

2. **Missing split resources.** The base APK's styles reference resources
   (`0x7f080044` → `drawable/abc_ab_share_pack_mtrl_alpha`) that live only in
   `config.hdpi.apk`. Merging by hand and copying just the `.so` files left 83
   dangling resource IDs, so the theme threw `Resources$NotFoundException` on
   first inflate. Fixed by merging with APKEditor:

   ```
   java -jar APKEditor.jar m -i input.xapk -o merged.apk -clean-meta
   ```

   Resource count went 11850 → 11931, and a full scan of all 47323 smali files
   now reports zero dangling `0x7f......` references.

---

## Chinese localization

`values-zh-rCN` did not exist upstream — the app had **zero** `zh` string
resources, so a Chinese-locale device fell back to English defaults.

Added:

| Type | Count | Notes |
|---|---|---|
| `<string>` | 2220 | Full app UI, `%s` / `%1$s` / `{handle}` placeholders preserved |
| `<plurals>` | 46 | Chinese uses a single `other` quantity |

Terminology is consistent across the app: Repost → 转推, Quote → 引用,
Post → 帖子, Direct Message → 私信, Community → 社区. Brand names
(`Premium`, `Grok`, `X`) are left untranslated.

Compose resources (`assets/composeResources/.../values-zh/`) already shipped
upstream and were not modified; their `.cvr` payloads are base64, e.g.
`autoplay_entries = 仅Wi-Fi / 总是 / 从不`.

`res/xml/locales_config.xml` already listed `zh-Hans`, so Chinese is selected
automatically on a Chinese system locale — no in-app language switch is needed
(the upstream app has none).

---

## Build from an unmodified XAPK

```bash
# 1. Merge splits, preserving extractNativeLibs="false"
java -jar APKEditor.jar m -i Chuckle.xapk -o merged.apk -clean-meta

# 2. Decode
apktool d -f -o src merged.apk

# 3. Apply patches (see tables above), then:
#    - ensure apktool.yml contains "- so" under doNotCompress
#    - drop values-zh-rCN/ into src/res/
#    - change manifest application to app.jjyy.chuckle.MainApplication

# 4. Build, align, sign
apktool b -f -o unsigned.apk src
zipalign -P 16 -f 4 unsigned.apk aligned.apk
apksigner sign --ks key.jks --min-sdk-version 32 --out final.apk aligned.apk
```

### Verification commands

```bash
apksigner verify --verbose final.apk
zipalign -c -P 16 4 final.apk
unzip -t final.apk
aapt2 dump resources final.apk | grep -c "(zh-rCN)"
dexdump -d classes.dex | grep -A6 "ohj.oO0OO00"
```

Expected `ohj.oO0OO00` body after patching:

```
const/4 v0, #int 1
return v0
```

---

## Status / caveats

All verification performed here was **static**: disassembly of the final dex,
resource-table dumps, signature and alignment checks. No arm64 device was
available, so runtime behaviour — whether the UI actually renders in Chinese,
whether the premium state reads as activated, whether it launches without
crashing — **has not been tested**.

If you hit a launch crash, capture:

```bash
adb logcat -s AndroidRuntime:E
```

and include the `FATAL EXCEPTION` block.

---

## Disclaimer

Provided for research and interoperability purposes only. This project is not
affiliated with, endorsed by, or supported by the developer of Chuckle or X
Corp. Redistribution of the APKs may constitute copyright infringement in your
jurisdiction. Download, use, and delete at your own risk.