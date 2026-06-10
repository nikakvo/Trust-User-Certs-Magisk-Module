# Trust User Certs

**Promotes user-installed CA certificates to the system trust store.**  
Supports Android 7–16 · SukiSU · KernelSU · KernelSU Next · Magisk

---

## What it does

Android keeps user-installed certificates (e.g. from Burp Suite, mitmproxy, Charles Proxy, or corporate CAs) separate from the system trust store. Most apps — and all apps targeting API 24+ — ignore user certs entirely.

This module solves that by merging your user certificates into the system store at boot, making them trusted system-wide, including in apps that pin to the system store.

Two injection paths are handled automatically:

- **Legacy path** (`/system/etc/security/cacerts`) — Android 7–13, handled via Magisk overlay
- **APEX/Conscrypt path** (`/apex/com.android.conscrypt/cacerts`) — Android 14+, handled via tmpfs + namespace injection into zygote64

---

## Features

- Works on Android 7 through 16
- Supports SukiSU, KernelSU, KernelSU Next and Magisk
- **Live sync** — re-injects certs without reboot when you add/remove a user cert
  - Uses bundled `inotifywait` (arm64) → `inotifyd` fallback → 30s polling fallback
- **Zygote monitor** — automatically re-injects if zygote64 restarts (e.g. Xiaomi.eu ROM)
- **Bootloop protection** — disables itself after 3 consecutive failed boots
- **Multi-user support** — merges certs from all Android users (configurable)
- **Manual cert drop** — place `.0` certs in `/data/local/tmp/cert/` for injection without using Android's cert installer
- **AdGuard conflict detection** — automatically removes conflicting AdGuard Personal Intermediate certs
- **Checksum cache** — skips unnecessary re-syncs when cert store hasn't changed
- Status shown in module description (Magisk/KSU app)

---

## Requirements

| Requirement | Value |
|---|---|
| Android | 7.0+ (API 24+) |
| Root manager | Magisk 20.4+ / KernelSU 0.9.0+ / KernelSU Next |
| Architecture | arm64-v8a |
| Min KernelSU | 11684 |

---

## Installation

1. Download the latest `.zip` from [Releases](../../releases/latest)
2. Flash via SukiSU / KernelSU / KernelSU Next / Magisk
3. Reboot

No additional setup required. The module auto-detects your Android version and uses the correct injection method.

---

## Configuration

Config file is created at first install:

```
/data/adb/trust-user-certs/config
```

| Option | Default | Description |
|---|---|---|
| `LOG_LEVEL` | `1` | `0` = off, `1` = normal, `2` = verbose |
| `AUTO_REPAIR` | `1` | Re-inject if zygote64 restarts |
| `LIVE_SYNC` | `1` | Re-inject when cert store changes (no reboot needed) |
| `INJECT_DELAY` | `0` | Seconds to wait before each namespace injection |
| `MULTI_USER` | `1` | `1` = merge certs from all Android users, `0` = primary user only |

Edit the file and reboot to apply changes.

---

## Manual cert installation

To inject a cert without using Android's certificate installer:

1. Rename your cert to its OpenSSL subject hash: `openssl x509 -subject_hash_old -in cert.pem | head -1` → e.g. `1a2b3c4d.0`
2. Copy the renamed cert to `/data/local/tmp/cert/`
3. Reboot (or wait for live sync)

---

## Logs

```
/data/adb/trust-user-certs/logs/boot.log     # post-fs-data output
/data/adb/trust-user-certs/logs/service.log  # service / live sync output
```

---

## Uninstall

Uninstall via SukiSU / KernelSU app. The `uninstall.sh` script automatically unmounts all active bind-mounts from zygote64 namespaces — no reboot required to clean up.

---

## Compatibility notes

- **Zygote64-only injection** — 32-bit zygote is skipped intentionally. On 64-bit-only devices the 32-bit zygote is a compatibility shim that restarts frequently; injecting into it causes unnecessary noise.
- **AdGuard** — if AdGuard Personal Intermediate CA is detected in the cert store, it is removed automatically to prevent conflicts.
- **Xiaomi.eu / custom ROMs** — zygote64 restarts on these ROMs are handled by the zygote monitor.

---

## Credits

Built and maintained by [Tears Burn](https://github.com/nikakvo).

Inspired by the original [MagiskTrustUserCerts](https://github.com/NVISOsecurity/MagiskTrustUserCerts) concept by NVISO Security.
