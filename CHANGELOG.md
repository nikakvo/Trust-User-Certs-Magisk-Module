# Changelog

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
