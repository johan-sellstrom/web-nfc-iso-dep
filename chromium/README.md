# Chromium prototype patches

Prototype of `NFCIsoDepReader` for Chromium on Android, behind the `WebNFCIsoDep` runtime flag.
These are prototype patches for discussion, not yet a reviewable CL series.

| Patch | What it does |
|---|---|
| `0001` | Mojo interface (`NFCIsoDepTag`, `WatchIsoDep`), Android device service (`EmrtdPolicy`, `IsoDepSession`, `NfcImpl`), Blink API (`NFCIsoDepReader`, `NFCIsoDepTag`, `NFCIsoDepReadingEvent`), runtime feature `WebNFCIsoDep` |
| `0002` | Build fixes |
| `0003` | Allow `GET RESPONSE` (response chaining); 30 s ISO-DEP timeout for slow chips |
| `0005` | Passport-friendly reader mode while only ISO-DEP readers are active: NFC-A/B, `FLAG_READER_SKIP_NDEF_CHECK`, 1 s presence-check delay |

There is no `0004` in this directory: it was a demo-only change that turned the feature on by
default.

## Build

Chrome for Android builds on an x86_64 Linux host (about 100 GB of disk).

1. Check out Chromium for Android ([instructions](https://chromium.googlesource.com/chromium/src/+/main/docs/android_build_instructions.md))
   at the revision the patches were made against:

   ```sh
   cd chromium/src
   git checkout c404659e2c7ac8060b760a58c18d9f0119d48758
   gclient sync -D
   ```

2. Apply the patches:

   ```sh
   git am /path/to/this/repo/chromium/*.patch
   ```

3. Configure and build:

   ```sh
   gn gen out/Android --args='target_os="android" target_cpu="arm64" is_debug=false is_component_build=false symbol_level=0'
   autoninja -C out/Android chrome_public_apk
   ```

4. Install `out/Android/apks/ChromePublic.apk`, then enable *Experimental Web Platform
   features* in `chrome://flags` and relaunch.

## Where the policy lives

The card classification and the command filter run in the browser process, in
`services/device/nfc/android/java/src/org/chromium/device/nfc/EmrtdPolicy.java`. The renderer
cannot bypass them.

## Not yet done

Web platform tests, Android unit tests for `EmrtdPolicy`, a Finch kill switch, use counters,
and a document-specific permission prompt. These are planned before an Intent to Prototype.
