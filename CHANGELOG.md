# Changelog

All notable changes to Trust User Certs are documented here.

## v2.6 — 2026-06-11

### WebUI
- **Fixed log viewer showing only 1 line** — `ksu.exec()` in SukiSU-Ultra truncates `stdout` after the first newline; log output is now piped through `base64 -w0` and decoded with `TextDecoder('utf-8')` on the JS side, preserving all lines
- **Fixed broken characters in log** (`âãœ` artifacts) — `atob()` decodes as Latin-1; replaced with `TextDecoder('utf-8')` over a `Uint8Array` for correct UTF-8 handling (emoji, dashes, etc.)
- **Fixed log box height** — `max-height` is ignored in KSU WebView when a parent has `overflow: hidden`; changed to a fixed `height: 280px` with `-webkit-overflow-scrolling: touch`
- **Fixed scroll-to-bottom not working** — replaced single `requestAnimationFrame` with a double `rAF` to guarantee layout is complete before scrolling in Android WebView
- **Action buttons now show loading state** — spinner ring + color glow appear while an operation is running; label changes to `syncing…` / `injecting…` / `resetting…` and restores on completion via `finally` block (button can no longer get stuck in loading state)
- **Fixed `writeCfg` silently failing** — `grep -v` returns exit code 1 when no lines match, aborting the pipeline before `mv`; added `|| true` to handle this correctly
- **Fixed `doAction inject` using `ps -P`** — non-standard flag on AOSP; replaced with direct `/proc` traversal, consistent with the same fix in `service.sh`
- **Fixed dead variable in `toggleFilter`** — removed unused `const btn` declaration
- **Header subtitle corrected** — was showing `SukiSU Ultra` only; now shows `Magisk · KSU · SukiSU`
- **Increased log tail** from 120 to 200 lines
- **Codebase translated to English** — all comments and UI strings were in Bulgarian; now fully in English for public readability

### service.sh
- **Removed unreliable `ps -o pid= -P`** — non-standard on AOSP, can return wrong results on some ROMs; replaced with `ps awk` as primary method and `/proc` traversal as fallback
- **Fixed `[NS:?]` always showing unknown namespace** — `inject_into_pid` now reads `readlink /proc/$pid/ns/mnt` of the target PID instead of the root shell's own namespace
- **Inject action delegates to `service.sh --force-inject`** — instead of running a slow `/proc` loop with many `exec()` calls from JS; JS loop retained as fallback
- **Codebase translated to English**

### post-fs-data.sh
- **Added 24h auto-reset for `boot_fail_count`** — if the fail counter file is older than 24 hours it is automatically reset, preventing a module from staying permanently disabled after an accidental crash
- **Fixed `mkdir -p -m 777`** (SC2174) — `-m` only applies to the deepest directory when used with `-p`; split into `mkdir -p` + `chmod 777`
- **Added `shellcheck disable` directives** for Magisk-specific false positives (`SC2034` for `SKIPUNZIP`, `SC1090` for dynamic config source)
- **Codebase translated to English**

### customize.sh
- **Fixed version mismatch** — `ui_print` and generated config header were still showing `v2.4` instead of `v2.5`
- **Codebase translated to English**

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
