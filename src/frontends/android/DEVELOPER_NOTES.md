# Developer Notes — strongSwan Android (16KB + Log/Traffic customizations)

Branch: `build-16KB-hide-logs-expose-tunnel-trafic`

| | |
|---|---|
| **Date** | 2026-08-25 |
| **strongSwan version** | 6.0.7 |
| **Base upstream commit** | `2265fcd1b` — *stroke: Avoid underflow when reporting number of SAs* (2026-08-24) |

This note records the customizations made to the stock strongSwan Android app on top
of upstream `master`, the build toolchain used, and how to reproduce the build.

---

## Changelog

### 1. 16 KB page-size support
- **`app/build.gradle`** — bumped `ndkVersion` from `27.3.13750724` to
  **`28.2.13676358`**. NDK r28 emits 16 KB-aligned load segments by default, so
  64-bit native libraries (`arm64-v8a`, `x86_64`) are built with `LOAD` alignment
  `0x4000` (16 384). 32-bit ABIs remain 4 KB (`0x1000`), which is correct.
- `compileSdk`/`targetSdk` were already `36`; `app/src/main/jni/Application.mk`
  already sets `APP_SUPPORT_FLEXIBLE_PAGE_SIZES := true`.
- **OpenSSL** rebuilt as static `libcrypto.a` from **OpenSSL 3.5.1** for all four
  ABIs (via `openssl/build.sh`) into `app/src/main/jni/openssl/`.

Verified: all 16 64-bit `.so` files report `LOAD align = 0x4000`; `zipalign -c -P 16`
on the APK passes; app runs on a 16 KB-page emulator (`getconf PAGE_SIZE` = 16384)
with every strongSwan native library loaded and no crashes.

### 2. Hide tunnel logs (no charon output to Android logcat)
Both native log sinks that write to the logcat `"charon"` tag were silenced:
- **`app/src/main/jni/libandroidbridge/charonservice.c`** — `dbg_android()`:
  commented out the `__android_log_print(...)` call. This suppresses the early
  `00[...]` library/daemon bootstrap messages.
- **`src/libcharon/plugins/android_log/android_log_logger.c`** — `log_()`:
  commented out both `__android_log_print(...)` calls (loop control preserved).
  This suppresses the actual IKE/tunnel operational logs
  (`07[IKE]`, `13[NET]`, `09[ENC]`, peer IPs, traffic selectors, NAT-T, rekeying).

> Note: silencing only `charonservice.c` (as an initial attempt did) hides just the
> `00[...]` bootstrap lines — the bulk of tunnel logs flow through the `android-log`
> plugin, hence the second change is required for full suppression.

### 3. Expose IPsec tunnel traffic counters to Java (JNI bridge)
- **`app/src/main/jni/libandroidbridge/charonservice.c`** — new JNI method
  `getTrafficStatistics()`. It enumerates active IKE_SAs via
  `charon->controller->create_ike_sa_enumerator(FALSE)`, walks each `child_sa`,
  and aggregates `child_sa->get_usestats()` for inbound (received) and outbound
  (transmitted) SAs. Returns `long[]{rxBytes, txBytes, rxPackets, txPackets}`.
  `wait=FALSE` avoids blocking the calling thread; the enumerator locks the
  IKE_SA manager only until it is destroyed (immediately after the loop).
- **`app/src/main/java/org/strongswan/android/logic/CharonVpnService.java`**:
  - `public native long[] getTrafficStatistics();`
  - `public static class TrafficStats` — immutable holder with
    `getRxBytes()/getTxBytes()/getRxPackets()/getTxPackets()`.
  - `public TrafficStats getTunnelTraffic()` — typed convenience wrapper
    (returns all-zeros when no tunnel is up or on error).

Counters are cumulative per SA (as tracked by userland `libipsec`) and reset when
the tunnel is re-established. `inbound=TRUE` → received/decrypted;
`inbound=FALSE` → transmitted/encrypted.

Verified: exported symbol
`Java_org_strongswan_android_logic_CharonVpnService_getTrafficStatistics`
present in `libandroidbridge.so`.

---

## Files changed (vs. upstream `master`)

| File | Change |
|------|--------|
| `app/build.gradle` | NDK → 28.2.13676358 |
| `app/src/main/jni/libandroidbridge/charonservice.c` | hide bootstrap logs + `getTrafficStatistics` JNI method |
| `src/libcharon/plugins/android_log/android_log_logger.c` | hide IKE/tunnel logs |
| `app/src/main/java/.../logic/CharonVpnService.java` | traffic-stats native method + `TrafficStats` API |

---

## Build toolchain

| Component | Version |
|-----------|---------|
| Android NDK | 28.2.13676358 |
| compileSdk / targetSdk | 36 |
| minSdk | 21 |
| OpenSSL (static libcrypto) | 3.5.1 |
| Android Gradle Plugin | 8.13.0 |
| Gradle | 8.13 |

## Reproducing the build

```sh
# 1. Prepare strongSwan sources (generates required files, then cleans host build)
./autogen.sh && ./configure && make && make distclean

# 2. Build static OpenSSL libcrypto for all ABIs into app/src/main/jni/openssl/
NO_DOCKER=1 \
  ANDROID_NDK_ROOT=/path/to/Android/Sdk/ndk/28.2.13676358 \
  OPENSSL_SRC=/path/to/openssl-3.5.1 \
  src/frontends/android/openssl/build.sh

# 3. Build the APK (from src/frontends/android/)
./gradlew assembleDebug      # or assembleRelease (produces app-release-unsigned.apk)
```

## Verifying 16 KB alignment

```sh
# ELF LOAD alignment must be 0x4000 for 64-bit ABIs
llvm-readelf -l lib/arm64-v8a/libcharon.so | awk '/LOAD/{print $NF; exit}'   # -> 0x4000

# APK-level page alignment
zipalign -c -v -P 16 4 app-release-unsigned.apk   # -> Verification successful
```
