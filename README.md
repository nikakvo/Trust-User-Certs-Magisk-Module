# Trust User Certs

A Magisk / KernelSU / KernelSU Next module that automatically promotes user-installed CA certificates to the system trust store, making them trusted by all apps without any app-side configuration.

This eliminates the need to add `network_security_config` to an application's manifest — useful for SSL/TLS traffic inspection, penetration testing, and corporate certificate deployments.

## Features

- Works on Android 7 through Android 16
- Supports both certificate store locations:
  - Legacy: `/system/etc/security/cacerts`
  - Modern (conscrypt/APEX): `/apex/com.android.conscrypt/cacerts`
- Supports multiple Android users
- Injects into all running zygote namespaces at boot and re-injects after zygote restarts
- Dynamic SELinux context — resilient to future Android changes
- Clean uninstall without reboot via `uninstall.sh`
- Compatible with Magisk, KernelSU, and KernelSU Next

## How It Works

Android 14+ (and earlier Android versions with Google Play Security Updates) moved the system CA certificate store from `/system/etc/security/cacerts` into an APEX module at `/apex/com.android.conscrypt/cacerts`. Each app process inherits this path from its zygote namespace, which makes it impossible to simply replace the files.

This module handles both scenarios:

**Legacy path (Android 7–13 without conscrypt updates):**
`post-fs-data.sh` collects all user certificates from `/data/misc/user/*/cacerts-added/` and merges them with the system certificates into the module's overlay directory. Magisk/KernelSU then automatically bind-mounts this directory over `/system/etc/security/cacerts`.

**Conscrypt/APEX path (Android 14+ or updated earlier versions):**
`service.sh` additionally mounts a `tmpfs` over `/system/etc/security/cacerts`, populates it with the merged certificates, then uses `nsenter` to inject a `--rbind` mount of that directory into every running zygote process namespace (both `zygote` and `zygote64`). A background monitor loop watches for zygote restarts and re-applies the injection automatically.

## Requirements

- Rooted device with Magisk v20.0+, KernelSU, or KernelSU Next
- Android 7 or higher

## Installation

1. Install a CA certificate as a **user certificate** via Android Settings → Security → Certificates (exact path varies by device and Android version)
2. Flash this module via your root manager (Magisk / KernelSU)
3. Reboot

The certificate will now be trusted system-wide.

## Removing a Certificate

1. Remove the certificate from the user store via Android Settings
2. Reboot

## Verifying It Works

After reboot, confirm the certificate count matches in both stores:

```sh
su -c ls /system/etc/security/cacerts/ | wc -l
su -c ls /apex/com.android.conscrypt/cacerts/ | wc -l
```

Both numbers should be equal and higher than the default system count. You can also inspect the module log:

```sh
su -c cat /data/adb/modules/trustusercerts/log.txt
```

## Troubleshooting

**Certificate not trusted after reboot:**
Check the log for any `FAILED` injection entries. A small number of `FAILED` lines is normal (processes that died between detection and injection). If all injections fail, there may be a SELinux policy conflict on your device.

**Module log not found:**
Verify the module ID with `su -c ls /data/adb/modules/` and adjust the path accordingly.

## Credits

Originally based on [AlwaysTrustUserCerts](https://github.com/NVISOsecurity/AlwaysTrustUserCerts) by Jeroen Beckers (NVISO).
Rewritten and maintained by [Tears Burn](https://github.com/nikakvo).

## License

MIT
