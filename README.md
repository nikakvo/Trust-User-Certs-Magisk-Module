# Trust User Certs

A SUkiSU Ultra / KernelSU / KernelSU Next module that automatically promotes user-installed CA certificates to the system trust store, making them trusted by all apps — without any app-side configuration.

Eliminates the need to patch `network_security_config` in app manifests. Useful for SSL/TLS traffic interception, penetration testing, and security research.

---

## Features

- Android 7 through Android 16 (arm64)
- Supports both certificate store paths:
  - Legacy: `/system/etc/security/cacerts`
  - Modern APEX/conscrypt: `/apex/com.android.conscrypt/cacerts`
- **Live sync** — add or remove certificates without rebooting (inotifyd watcher)
- Injects into all zygote namespaces at boot, re-injects after zygote restarts
- Multi-user support — merges certs from all Android user profiles
- Dynamic SELinux context — resilient to future Android label changes
- **Config file** — control behavior without touching scripts
- **Structured logging** — PID + namespace ID in every log line
- **Fail-safe bootloop protection** — auto-disables after 3 consecutive failures
- Bundled `inotifyd` binary (arm64) — no dependency on system binary
- Clean live uninstall via `uninstall.sh` — no reboot required

---

## How It Works

Android 14+ moved the system CA store from `/system/etc/security/cacerts` into a read-only APEX module at `/apex/com.android.conscrypt/cacerts`. Each app process inherits this path through an isolated zygote mount namespace — it cannot be overridden from outside.

**Legacy path (Android 7–13):**
`post-fs-data.sh` merges system certs with all user-installed certs into the module overlay directory. The root manager auto-mounts it over `/system/etc/security/cacerts` on boot.

**APEX/conscrypt path (Android 14+):**
`service.sh` mounts a `tmpfs` over `/system/etc/security/cacerts`, populates it with the merged certs, then uses `nsenter` to bind-mount that directory into every running zygote process namespace (`zygote` and `zygote64`). A background monitor loop watches for zygote restarts and re-injects automatically.

**Live sync (v2.3):**
An `inotifyd` watcher monitors `/data/misc/user/*/cacerts-added/`. When a cert is installed or removed via Android Settings, the module re-syncs the tmpfs store and re-injects into all namespaces within ~1–2 seconds — no reboot needed. Falls back to 30-second polling if `inotifyd` is unavailable.

---

## Requirements

- SUkiSU Ultra, KernelSU, or KernelSU Next
- Android 7 or higher
- arm64 device

---

## Installation

1. Flash the module ZIP via your root manager
2. Reboot
3. Install your CA certificate via **Android Settings → Security → Trusted credentials → Install**

On Android 14+ with APEX path, the cert is picked up by the live sync watcher — no reboot needed after step 3.

---

## Manual Cert Push

You can push a certificate directly without going through Android Settings:

```sh
# Compute the Android hash name
HASH=$(openssl x509 -inform PEM -subject_hash_old -in yourcert.pem | head -1)

# Place it in the manual cert directory
cp yourcert.pem /data/local/tmp/cert/${HASH}.0
```

The directory `/data/local/tmp/cert/` is created automatically on first boot with `chmod 777` — writable from Termux without root. Certs placed here are merged into the trust store on every boot and live sync cycle.

---

## Config

Edit `/data/adb/trust-user-certs/config` to control module behavior. Created automatically on first flash.

```sh
LOG_LEVEL=1       # 0=off  1=normal  2=verbose
AUTO_REPAIR=1     # re-inject after zygote restarts
LIVE_SYNC=1       # inotifyd watcher (0 = polling fallback)
INJECT_DELAY=0    # seconds to wait before each nsenter call
MULTI_USER=1      # merge certs from all Android users (0 = uid 0 only)
```

---

## Logs

```sh
# Structured logs (v2.3)
su -c cat /data/adb/trust-user-certs/logs/boot.log
su -c cat /data/adb/trust-user-certs/logs/service.log

# Legacy log (root manager webUI)
su -c cat /data/adb/modules/trustusercerts/log.txt
```

Each line in the structured logs includes timestamp, level (`INFO`/`WARN`/`ERROR`/`DEBUG`), PID, and mount namespace ID. Logs auto-rotate — `boot.log` keeps the last 200 lines, `service.log` keeps the last 500 lines.

---

## Verifying It Works

```sh
# Cert counts must match
su -c ls /system/etc/security/cacerts/ | wc -l
su -c ls /apex/com.android.conscrypt/cacerts/ | wc -l

# Confirm fail-safe counter is 0 (healthy)
su -c cat /data/adb/trust-user-certs/boot_fail_count
```

---

## Removing a Certificate

Remove via **Android Settings → Security → Trusted credentials → User**. On APEX path devices (Android 14+) the live sync watcher picks up the removal automatically. On legacy path a reboot is required.

---

## Troubleshooting

**Certificate not trusted after install:**
Check `service.log` for injection results. A few `FAILED` lines are normal — processes that died between detection and injection are handled by the monitor loop on the next 5-second cycle.

**Module auto-disabled:**
The fail-safe triggered after 3 consecutive boot failures. Check `boot.log` for the cause, fix it, then re-enable the module via your root manager. Reset the counter manually if needed:
```sh
su -c "echo 0 > /data/adb/trust-user-certs/boot_fail_count"
```

**ERROR: nsenter not found:**
Your ROM does not include `nsenter`. This only affects APEX path injection (Android 14+). The legacy path works without it.

**ERROR: tmpfs mount failed:**
Rare — usually a kernel restriction or SELinux denial. Check `dmesg` for details.

---

## Credits

Originally based on [AlwaysTrustUserCerts](https://github.com/NVISOsecurity/AlwaysTrustUserCerts) by Jeroen Beckers (NVISO).  
Rewritten and maintained by [Tears Burn](https://github.com/nikakvo).

---

## License

MIT
