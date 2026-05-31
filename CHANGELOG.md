# Changelog

## v2.4

### Fixed

- **Fail-safe bootloop protection now works correctly.**
  Previously `mark_boot_success` was called at the end of `post-fs-data.sh`, resetting the counter on every boot regardless of whether the injection succeeded — meaning the counter could never reach 3. The reset is now done in `service.sh` only after a successful `inject_zygote64()`, i.e. a real healthy boot.

- **Mount validation is now accurate.**
  `has_mount()` previously used `grep` to search for the source path in `mountinfo`, which never matched — the source path does not appear as plain text in that file. Replaced with `awk '$5==p'` which checks field 5 (the mountpoint) directly, as per the `mountinfo` format spec.

- **tmpfs double-mount on service restart no longer crashes.**
  If `service.sh` was restarted for any reason (manual trigger, live-sync bug, Magisk service restart), the unconditional `mount -t tmpfs` would fail. Now guarded with a `/proc/mounts` check — mounts only if not already mounted.

- **`compute_cert_hash()` no longer returns a false hash on empty cert store.**
  `md5sum` on empty input returns `d41d8cd98f00b204e9800998ecf8427e`, not an empty string, so the previous `[ -z ]` guard was dead code. The function now checks whether `find` produced any output before passing to `md5sum`, and returns the literal string `"empty"` when no certs are present.

- **`compute_cert_hash()` is now safe for filenames containing spaces.**
  The previous `xargs stat` implementation would break on cert filenames with spaces (rare but valid). Replaced with `-exec stat {} +` which passes filenames directly to `stat` without shell word splitting.

- **Status reporting no longer counts non-cert files.**
  All `ls | wc -l` calls replaced with `find -name "*.0" | wc -l` throughout `service.sh` and `post-fs-data.sh`.

- **`local` keyword removed from loops and subshells.**
  Some KernelSU environments handle `local` unexpectedly outside strict function scope. All variables promoted to plain assignments declared at function entry.

### Changed

- **32-bit zygote injection removed.**
  `zygote` (32-bit) is no longer injected or monitored. On 64-bit only devices it runs as a compatibility layer, restarts frequently, and was the sole cause of the constant `WARN: zygote has 1 children, waiting` noise in logs. All injection and monitoring is now `zygote64`-only.

- **Live sync upgraded from `inotifyd` to `inotifywait`.**
  `inotifywait` is now the primary live sync backend. It blocks on events directly in the loop — no external handler script needed. Falls back to `inotifyd`, then 30s polling. A bundled `inotifywait` arm64 binary is included so no Termux or system package is required.

- **Checksum cache — unnecessary re-injections eliminated.**
  A hash of the cert store state (filename + size + mtime via `-exec stat {} +`) is saved after every sync. On `inotifywait` events, `inotifyd` triggers, and polling cycles, a full `sync_certs_to_tmpfs` + `inject_zygote64` is only performed if the hash has actually changed.

- **Zygote PID cache in monitor loop.**
  `monitor_zygote64()` tracks the last known `zygote64` PID. If the PID is unchanged on a 5-second cycle, only a quick mount check is performed. A full re-injection is triggered only on a new PID — an actual restart.

- **`find -exec rm` instead of `rm -rf dir/*` for cert cleanup.**
  Overlay and tmpfs directories are cleared with `find -maxdepth 1 -name "*.0" -exec rm -f {} +`, touching only cert files and leaving any other directory content intact.

- **`uninstall.sh` updated.**
  Now unmounts only `zygote64` namespaces (matching injection scope). Cleans up runtime files on uninstall: `cert_store.hash`, `cert_sync_handler.sh`, `moddir`, `boot_fail_count`.


## v2.3
- **Live sync**: inotifyd watcher on all user cert directories — add/remove certs without reboot
- **Polling fallback**: 30s polling when inotifyd unavailable (graceful degradation)
- **Structured logging**: separate boot.log and service.log under `/data/adb/trust-user-certs/logs/`
  - Each line includes timestamp, level, PID, and mount namespace ID
  - Log rotation: keeps last 200 lines (boot.log) / 500 lines (service.log) from previous session
- **Config file**: `/data/adb/trust-user-certs/config` — control LOG_LEVEL, LIVE_SYNC, AUTO_REPAIR, INJECT_DELAY, MULTI_USER
- **Fail-safe bootloop protection**: tracks consecutive failed boots, auto-disables module after 3 failures
- **Bundled inotifyd** (arm64): module ships own binary as fallback if system lacks it
- **customize.sh**: creates config directory and default config on first install
- **INJECT_DELAY** config option: adds delay before each nsenter call for devices where processes die mid-inject


## v2.2

### Added
- **Manual certificate push via `/data/local/tmp/cert/`**
  Certificates placed in this directory are automatically merged into the
  system trust store on every boot — no Android Settings required. The
  directory is created on first boot with `chmod 777`, making it writable
  from Termux without root. Files must follow the Android cert naming
  convention: `<subject_hash_old>.0`.

- **Automatic conflict cleanup for intermediate CA certificates**
  Known conflicting intermediate CA certificates are now detected and
  removed from the merged store during both the early-boot phase
  (`post-fs-data.sh`) and the APEX injection phase (`service.sh`).
  Removal is done by filename hash pattern and by certificate content
  string, so renamed copies are also caught. This prevents broken SSL
  interception caused by certificate chain resolution through an
  unintended issuer.

## v2.1

* **Fixed:** `find_nsenter()` — replaced `command -v nsenter` with physical path probing (`/system/bin`, `/usr/bin`, `/bin`). Previous approach returned a bare name on some ROMs causing `[ -x ]` check to fail even when nsenter existed
* **Fixed:** `ls | head` anti-pattern in `copy_selinux_context()` replaced with `for` loop — safer with special characters in filenames
* **Fixed:** missing `mkdir -p` in fallback cert copy path in `service.sh`
* **Improved:** collision logging in `post-fs-data.sh` — when a user cert replaces a system cert with the same filename, it is now logged explicitly

## v2.0

* **Fixed critical bug:** `monitor_zygote` was checking undefined `$pid` instead of `$zp` when testing whether a bind-mount was already present — causing injection to be skipped or applied incorrectly
* **Fixed cert priority:** system certificates are now copied before user certificates in `post-fs-data.sh`, so user certs correctly override system certs on filename collision instead of the reverse
* **Improved child process detection:** three-strategy fallback for finding zygote children — modern `ps -P`, legacy `ps | awk`, and direct `/proc` scan. The `/proc` method is the most reliable and works on all Android versions including Android 16
* **Dynamic SELinux context:** label is now copied from an existing APEX cert file at runtime instead of being hardcoded, making the module resilient to future Google renames
* **Boot timeout:** `service.sh` now aborts cleanly if `sys.boot_completed` is never set (max 180s), preventing an infinite spin on broken boot states
* **Improved cert pipeline:** `service.sh` now uses the pre-merged cert directory built by `post-fs-data.sh` (system + user certs combined) instead of copying only APEX certs and losing user certs in the process
* **Added `uninstall.sh`:** cleanly removes all active bind-mounts from zygote namespaces and tmpfs overlays without requiring a reboot
* **Glob safety:** added `[ -f "$cert" ]` guard to prevent errors when `cacerts-added` directories are empty
* **Logging improvements:** all operations now log success/failure individually, with warnings for prolonged missing zygote and boot timeout

## v1.3

* Fixed bug on A14+

## v1.2

* Added automatic update support

## v1.1

* Fixed permission issue for non-conscrypt
* Fixed removal of certs for non-conscrypt
* Renamed repo

## v1.0

* Add support for mainline/conscrypt certificates
* Add support for multiple users
* Add support for KernelSU

## v0.4.1

* Supports Android 10
* Updated module to be compatible with latest Magisk module template (v20.4+)

## v0.3

* Module now removes all user-installed certificates from the system store before copying them over, so that removed user certificates no longer persist in the system store

## v0.2

* Fixed directory creation bug
* Updated module to be compatible with latest Magisk module template (v15+)

## v0.1

* Initial release
