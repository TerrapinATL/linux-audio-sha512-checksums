### linux-audio-flac-sha512-checksum — Change Log

All version changes are appended to this file, newest last, one `## vX Change Log` section per version.

**Update rule:** before writing to the version-less main guide file, the current content must first be saved as a versioned copy (e.g. `linux-audio-flac-sha512-checksum-v14.md`) so every published version stays retrievable.

**Current version: v14** — Current version; supersedes v13.

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
