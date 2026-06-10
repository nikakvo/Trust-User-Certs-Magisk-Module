# Changelog

All notable changes to Trust User Certs are documented here.

---

## v2.5

### Added
- **Zygote64-only injection** — 32-bit zygote injection removed. Eliminates unnecessary re-injection noise on 64-bit-only devices where the 32-bit zygote is merely a compatibility shim.
- **Real fail-safe bootloop protection** — boot fail counter increments in `post-fs-data.sh` and resets only after a confirmed successful inject in `service.sh`. Module auto-disables after 3 consecutive failures.
- **Checksum cache** (`cert_store.hash`) — cert store is hashed by filename + size + mtime. Live sync and polling skip re-injection if the store hasn't actually changed.
- **Bundled arm64 `inotifywait` binary** — no Termux dependency. Automatically installed to `/system/bin/inotifywait` via Magisk overlay.
- **Live sync inotify hierarchy**: bundled `inotifywait` → `inotifyd` fallback → 30-second polling fallback. Auto-detected at runtime.
- **Improved mount check** — uses `/proc/<pid>/mountinfo` field 5 (mountpoint) for reliable namespace mount detection, independent of source text.
- **`nsenter` dependency check** — exits with a clear error if `nsenter` is not available instead of silently failing.
- **Zygote PID cache** — inject only triggers on actual PID change (zygote restart), not on every monitor loop iteration.
- **AdGuard conflict detection** — removes AdGuard Personal Intermediate CA by hash (`47ec1af8.*`) and by content pattern to prevent SSL conflicts.
- **Manual cert drop directory** — place renamed `.0` certs in `/data/local/tmp/cert/` for injection without Android's cert installer.
- **Multi-user cert merging** — configurable via `MULTI_USER` flag.
- **`INJECT_DELAY` config option** — per-namespace injection delay for ROMs with slow zygote startup.
- **Verbose logging** — `LOG_LEVEL=2` logs namespace ID, PID, and per-cert operations.
- **`uninstall.sh`** — live unmount from all zygote64 namespaces without requiring reboot.

### Changed
- `post-fs-data.sh` and `service.sh` use separate log files (`boot.log` / `service.log`) instead of a single shared file.
- Log rotation: previous session preserved (last 200 / 500 lines) at the top of each new log.
- Config file preserved across reinstalls — existing config is never overwritten.
- Module description in `module.prop` updated at runtime to reflect current working status.

### Fixed
- Race condition where fail counter could reset before `service.sh` confirmed successful injection.
- False "cert store changed" triggers from empty `find` output producing a fixed md5sum hash.
- Mount idempotency — tmpfs is only mounted if not already present, preventing errors on `service.sh` restart.

---
