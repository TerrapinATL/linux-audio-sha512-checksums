### linux-audio-flac-sha512-checksum — Change Log

All version changes are appended to this file, newest last, one `## vX Change Log` section per version.

**Update rule:** before writing to the version-less main guide file, the current content must first be saved as a versioned copy (e.g. `linux-audio-flac-sha512-checksum-v14.md`) so every published version stays retrievable.

**Suite convention (auto-purge):** every guide/repo with error logging must purge its log directory at the START of the workflow (first step), so the previous run's logs remain reviewable until the next run replaces them. This applies to all current and future repositories.

**Current version: v15** — supersedes v14. Step 1 terminal output aligned
to the moode-cleanup guide format. See the v15 entry below.

Main guide: [linux-audio-flac-sha512-checksum.md](linux-audio-flac-sha512-checksum.md)

---

## v14 Change Log (2026-08-29)

Summary of changes from v13 (no per-version change log was recorded for v13
and earlier; those versions are archived locally only):

* **Software Preflight (new)** — a diagnostic script (Section 02) that runs
  before any step and verifies every required tool (`sha512sum` plus core
  utilities), failing loudly with install hints if anything is missing. It
  writes `~/.logs/linux-audio-flac-sha512-checksum/preflight.log` and never
  touches audio files.
* **Per-step Logging sections (new)** — every step now documents its
  five-file log set (`stepNN-run/oks/fails/errors/summary`) written under
  the centralized `~/.logs/linux-audio-flac-sha512-checksum/` directory.
* **Section 11 expanded** — "Troubleshooting & Reference Information" became
  "Troubleshooting, Log File Reference & Disclaimer," adding:
  * Permission Denied Errors — `chown`/`chmod` fix for files copied across
    filesystems or drives.
  * Log File Reference — what each log file contains and how to use
    fails.log as the troubleshooting punch list.
  * General Cleanup — when it is safe to delete the entire log directory.
  * Disclaimer.
* **Requirements** — `mktemp` added to the core-utilities list.
* **Backup note** — the Introduction now carries the standing reminder to
  work off a backup library copy until a clean copy has been secured by
  these checksums.
* **Title standardized** — the old "FLAC Library SHA-512 Checksum &
  Verification Guide" heading was replaced with the suite's file-name-style
  title and version line.
* **Auto-purge added (2026-09-14)** — Step 1 now purges the entire log
  directory at the start of the workflow (suite auto-purge convention), so
  the previous run's logs remain reviewable until the next run replaces
  them.

---

## v15 Change Log (2026-09-16)

* **Step 1 header/footer aligned to the moode-cleanup guide format.**
  The Stray File Audit now opens with the standard banner block —
  `========== Step 1: Stray File Audit and Identification ==========`,
  followed by `Root:` and `Started:` lines — and closes with the standard
  footer (`----` dividers, `Library scanned  Stray files: N`, then the
  step-title banner as the strictly-final output). The plain
  "started.../completed.../SUMMARY:" lines were replaced by the banner
  block; log files (`step1-run.log`, `step1-oks.log`, `step1-fails.log`)
  keep the same content and names.
* **`.mpdignore` files excluded from the stray-file audit.** These are
  intentional MPD/moOde library-exclusion markers (the user's Ignore-
  folder convention), not strays; they must not hold up checksum
  generation. The manifest-name logic elsewhere is unchanged, and no
  behavioral changes to the scan, other exclusions, permission checks,
  or auto-purge.
* **`Ignore.sha512sums.txt` accepted as a third generic manifest name.**
  15d of the moOde cleanup guide writes this self-contained manifest
  inside each Ignore folder (integrity test + per-file SHA-512 for
  library files hidden from moOde via `.mpdignore`). The Step 1 stray
  audit, the Step 6 rogue-name check, and the Step 6 missing-manifest
  check all accept it. (Note: the artist-level aggregate hash already
  fingerprints Ignore content; 15d adds per-file granularity and decode
  testing.)
* **Stray policy made explicit; no automation of stray handling.** Step 1
  lists strays and nothing else — it never moves or deletes files. The
  decision on each stray (rename, relocate, delete) is always the
  user's. (An optional auto-quarantine was prototyped during this
  revision and removed at the library owner's direction: strays are
  listed, not acted upon.) Motivation: during the 2026-09-16 walkthrough,
  stray files left in the library root (`step2c3.sh`,
  `album_checklist.md`, legacy `sha256sums.txt`, `.flac.not`,
  extensionless FLAC files) required manual review and cleanup; the
  audit's job is to surface them, not to act on them.
* Also folded in: the minor documentation corrections that had been
  applied to the working copy after v14 was pushed (changelog link in
  the header, auto-purge comment in Step 1, standardized divider).
