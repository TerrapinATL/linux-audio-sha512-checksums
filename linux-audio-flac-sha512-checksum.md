### linux-audio-flac-sha512-checksum

**Version: v15** — Current version; supersedes v14. Step 1 terminal
output (header/footer) aligned to the moode-cleanup guide format,
`.mpdignore` files excluded from the stray-file audit, the stray
policy made explicit (list only — user decides what happens to strays),
and `Ignore.sha512sums.txt` accepted as a third generic manifest name
(written by 15d of the moOde cleanup guide inside each Ignore folder).
(Changes applied 2026-09-16 / 2026-09-17.)

Change log and version history are maintained separately:
[linux-audio-flac-sha512-checksum-changelog.md](linux-audio-flac-sha512-checksum-changelog.md)

---

01. Introduction

---

This document is a guide and automated script procedure for generating, verifying, and auditing cryptographic hashes across large audio libraries on Linux systems.

It is intended for users who require strict data preservation, routine auditing for silent data corruption (bit rot), and absolute verification.

Typically, these SHA-512 checksums are verified after data is copied to ensure the copy is a perfect match to the original.

Note: Always work off a backup library copy until a clean copy has been secured by these sha512 checksums.

---

02. Requirements

---

To successfully execute the scripts and workflows in this guide, your system must have the following command-line tools installed and available in your shell's PATH:

* sha512sum – Required for calculating and verifying the SHA-512 cryptographic hashes.

* Core Utilities – Standard GNU core utilities (find, sort, awk, grep, wc, basename, dirname, mktemp) commonly available in Linux environments like Linux Mint.

* Zenity / Nemo Actions (Optional) – For users integrating visual verification into the Nemo file manager via custom action scripts.

-- Software Preflight

Run the preflight diagnostic below BEFORE starting any step. It verifies that every tool the guide requires is installed, and fails loudly with install hints if anything is missing. It writes a diagnostic report to `~/.logs/linux-audio-flac-sha512-checksum/preflight.log` and does not modify any audio files.

--- Bash Script Preflight Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ---------------------------------------------------------------------------
# Software Preflight - Verify all required tools before any step runs
# ---------------------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-flac-sha512-checksum"
mkdir -p "$LOG_ROOT"
PREFLIGHT_LOG="$LOG_ROOT/preflight.log"

: > "$PREFLIGHT_LOG"

echo "================================================" | tee -a "$PREFLIGHT_LOG"
echo "Software Preflight - All Required Tools"         | tee -a "$PREFLIGHT_LOG"
echo "================================================" | tee -a "$PREFLIGHT_LOG"

missing=0
pass=0

check_cmd() {
    local tool="$1"
    if command -v "$tool" >/dev/null 2>&1; then
        printf "%-22s : OK\n" "$tool" >> "$PREFLIGHT_LOG"
        pass=$((pass + 1))
    else
        printf "%-22s : MISSING\n" "$tool" >> "$PREFLIGHT_LOG"
        missing=$((missing + 1))
    fi
}

for tool in sha512sum find sort awk grep wc basename dirname mktemp tee; do
    check_cmd "$tool"
done

echo "----------------------------------------" | tee -a "$PREFLIGHT_LOG"
echo "Pass: $pass   Missing: $missing"         | tee -a "$PREFLIGHT_LOG"

if [ "$missing" -eq 0 ]; then
    echo "RESULT: ALL SOFTWARE PRESENT - ready to run." | tee -a "$PREFLIGHT_LOG"
    echo "----------------------------------------" | tee -a "$PREFLIGHT_LOG"
    exit 0
else
    echo "RESULT: $missing item(s) missing. Install them and re-run preflight." | tee -a "$PREFLIGHT_LOG"
    echo "----------------------------------------" | tee -a "$PREFLIGHT_LOG"
    exit 1
fi
```

--- Bash Script Preflight End ---

-- Note on File Naming Conventions

To prioritize strict data preservation and structural clarity, this workflow utilizes a specific tiered naming convention for its manifests: `ARTIST.sha512sums.txt` for top-level artist directories and `ALBUM.sha512sums.txt` for individual album folders. Maintaining this exact convention is critical for the automated auditing scripts and Nemo action configurations to function correctly.

-- Library Structure

```
Parent/
├── Artist1/
│   ├── artist-level files
│   ├── Album1/
│   │   ├── album files
│   │   └── ...
│   └── Album2/
└── Artist2/
```

---

03. Design Philosophy

---

This guide is built on four core data-management principles:

* Decentralized Verification: Hash files are stored locally within the directory of the files they protect, ensuring that data and its verification record travel together during transfers.
* Relative Pathing: Scripts operate within subshells to generate strictly relative paths in the hash files, preventing verification failures if the library is moved to a different drive or mount point.
* Traceable Execution: All operations generate clean, timestamped logs and separate success/failure lists, ensuring total transparency and auditability across massive data sets.
* Built-In Permission Guards: Every script includes an automated permission check and interactive chown prompt to catch and resolve root-locked files or mounts (common when formatting drives on desktop systems for Raspberry Pi environments) before processing begins.

---

04. Workflow Overview

---

The hashing process follows a comprehensive six-step operational pipeline designed to audit, establish, confirm, and maintain data integrity:

* Step 1: Stray File Audit and Identification — Scan and isolate unrecognized files prior to hashing
* Step 2: Create SHA-512 Checksums for Album Folders
* Step 3: Verification of Album Folders
* Step 4: Create SHA-512 Checksums for Artist Folders
* Step 5: Verification of Artist Folders
* Step 6: Auditing the Library — Scan the file system to identify rogue folders or incorrect checksum naming

All scripts direct their tracking data to `$HOME/.logs/linux-audio-flac-sha512-checksum` regardless of which library folder you are working in. See the Log File Reference in Section 11 for details.

---

05. Step 1 – Stray File Audit and Identification

---

-- Purpose

This step scans the Parent folder, all Artist folders, and all Album folders to identify and report any stray, unrecognized, or misplaced files anywhere within the library hierarchy.

Before generating checksum manifests, it is critical to ensure that no unexpected or extraneous files are included in the cryptographic hash baseline. This step flags any files that are not authorized audio files, valid album artwork, or existing checksum manifests.

-- Logging

This step writes `step1-run.log` (full transcript), `step1-oks.log` (clean audit result), and `step1-fails.log` (stray/unrecognized files) to `$HOME/.logs/linux-audio-flac-sha512-checksum`.

**Stray policy:** strays are listed only — the audit never moves or deletes anything. Deciding what happens to each stray (rename, relocate, or delete) is always a user decision.

--- Bash Script Step 1 Start ---
```bash

#!/usr/bin/env bash

#Step 1 - Stray File Audit and Identification

LOG_ROOT="$HOME/.logs/linux-audio-flac-sha512-checksum"

# AUTOPURGE: remove all logs from any previous run of this workflow.
# This runs at the START of the workflow, so the previous run's logs remain
# on disk for review until the next run replaces them.
rm -rf "$LOG_ROOT"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/step1-run.log"
OKS_LOG="$LOG_ROOT/step1-oks.log"
FAILS_LOG="$LOG_ROOT/step1-fails.log"

: > "$RUN_LOG"; : > "$OKS_LOG"; : > "$FAILS_LOG"

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R $USER:$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by $USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------

{
    echo "========== Step 1: Stray File Audit and Identification =========="
    echo "Root: $PWD"
    echo "Started: $(date)"
    echo

    find "$PWD" -type f \
        ! -iname "*.flac" \
        ! -iname "*.mp3" \
        ! -iname "*.m4a" \
        ! -iname "*.wav" \
        ! -iname "*.ogg" \
        ! -iname "*.opus" \
        ! -iname "*.alac" \
        ! -iname "*.jpg" \
        ! -iname "*.jpeg" \
        ! -iname "*.png" \
        ! -name "ARTIST.sha512sums.txt" \
        ! -name "ALBUM.sha512sums.txt" \
        ! -name "Ignore.sha512sums.txt" \
        ! -iname ".mpdignore" \
        -print | tee "$FAILS_LOG"

    count=$(wc -l < "$FAILS_LOG" 2>/dev/null || echo 0)

    if [ "$count" -gt 0 ]; then
        echo "WARNING: Found $count stray/unrecognized file(s). Review '$FAILS_LOG' before generating checksums."
        echo "Decision on each stray is left to the user - nothing is moved or deleted automatically."
        printf "%s\n" "$(basename "$PWD") : $count stray file(s) found" > "$OKS_LOG"
    else
        echo "OK: No unauthorized stray files detected. Library is clean for checksum generation."
        printf "%s\n" "$(basename "$PWD") : clean" > "$OKS_LOG"
    fi

    echo
    echo "----------------------------------------"
    echo "Library scanned  Stray files: $count"
    echo "----------------------------------------"
    echo "Step 1 – Stray File Audit and Identification"
    echo "----------------------------------------"
} | tee "$RUN_LOG"

```

--- Bash Script Step 1 End ---

-- Review Results

View the generated reports by running:

--- Bash Script Results 1 Start ---
```bash

cat "$LOG_ROOT/step1-run.log"
cat "$LOG_ROOT/step1-oks.log"
cat "$LOG_ROOT/step1-fails.log"

```
--- Bash Script Results 1 End ---

---

06. Step 2 – Create SHA-512 Checksums for Album Folders

---

-- Purpose

This step establishes the cryptographic fingerprint for individual files located inside nested Album folders. It creates the standard `ALBUM.sha512sums.txt` file.

-- Logging

This step writes `step2-run.log` (full transcript), `step2-oks.log` (successfully checksummed albums), `step2-fails.log` (albums that failed), and `step2-errors.log` (detailed utility error output) to `$HOME/.logs/linux-audio-flac-sha512-checksum`.

--- Bash Script Step 2 Start ---
```bash

#!/usr/bin/env bash

#Step 2 - Create SHA-512 Checksums for Album Folders

LOG_ROOT="$HOME/.logs/linux-audio-flac-sha512-checksum"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/step2-run.log"
OKS_LOG="$LOG_ROOT/step2-oks.log"
FAILS_LOG="$LOG_ROOT/step2-fails.log"
ERRORS_LOG="$LOG_ROOT/step2-errors.log"

: > "$RUN_LOG"; : > "$OKS_LOG"; : > "$FAILS_LOG"; : > "$ERRORS_LOG"

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R $USER:$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by $USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------
# Detect folder level by probing actual directory structure below $PWD
# (relative to $PWD, not absolute path depth -- works regardless of mount point)
if find "$PWD" -mindepth 2 -maxdepth 2 -type d -print -quit 2>/dev/null | grep -q .; then
    min=2; max=2   # Parent folder: Parent/Artist/Album
elif find "$PWD" -mindepth 1 -maxdepth 1 -type d -print -quit 2>/dev/null | grep -q .; then
    min=1; max=1   # Artist folder: Artist/Album
else
    min=0; max=0   # Album folder itself
fi

mapfile -d '' dirs < <(find "$PWD" -mindepth $min -maxdepth $max -type d -print0 | LC_ALL=C sort -z)
total=${#dirs[@]}
i=0

for d in "${dirs[@]}"; do
    i=$((i+1))

    if [ $min -eq 0 ]; then
        album=$(basename "$d"); artist=$(basename "$(dirname "$d")"); label="$artist-$album"
    elif [ $min -eq 1 ]; then
        album=$(basename "$d"); artist=$(basename "$PWD"); label="$artist-$album"
    else
        album=$(basename "$d"); artist=$(basename "$(dirname "$d")"); label="$artist-$album"
    fi
    err=$(mktemp "$LOG_ROOT/step2-temp.XXXXXX")
    (
        cd "$d" || exit 1
        shopt -s nullglob
        files=(*)
        shopt -u nullglob
        target_files=()
        for f in "${files[@]}"; do
            if [[ -f "$f" && "$f" != "ARTIST.sha512sums.txt" && "$f" != "ALBUM.sha512sums.txt" ]]; then
                target_files+=("$f")
            fi
        done
        if [ ${#target_files[@]} -gt 0 ]; then
            sha512sum "${target_files[@]}" > "ALBUM.sha512sums.txt" 2>"$err"
            exit $?
        else
            exit 0
        fi
    )
    rc=$?
    if [ $rc -ne 0 ]; then
        echo "FAIL [$i/$total] $label"
        echo "$label" >> "$FAILS_LOG"
        sed "s/^/[$i\/$total] ERROR: $label :: /" "$err" >> "$ERRORS_LOG"
    else
        echo "OK   [$i/$total] $label"
        echo "$label" >> "$OKS_LOG"
    fi
    rm -f "$err"
done | tee "$RUN_LOG"

echo "SUMMARY: $total album folder(s) processed." | tee -a "$RUN_LOG"

```

--- Bash Script Step 2 End ---

-- Review Results

View the generated reports by running:

--- Bash Script Results 2 Start ---
```bash

cat "$LOG_ROOT/step2-run.log"
cat "$LOG_ROOT/step2-oks.log"
cat "$LOG_ROOT/step2-fails.log"
cat "$LOG_ROOT/step2-errors.log"

```
--- Bash Script Results 2 End ---

---

07. Step 3 – Verification of Album Folders

---

-- Purpose

This step validates the integrity of the individual album folders, confirming that no audio files have suffered bit rot or silent corruption.

-- Logging

This step writes `step3-run.log` (full transcript), `step3-oks.log` (verified albums), `step3-fails.log` (albums that failed), and `step3-errors.log` (checksum mismatch detail) to `$HOME/.logs/linux-audio-flac-sha512-checksum`.

--- Bash Script Step 3 Start ---
```bash

#!/usr/bin/env bash

#Step 3 - Verification of Album Folders

LOG_ROOT="$HOME/.logs/linux-audio-flac-sha512-checksum"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/step3-run.log"
OKS_LOG="$LOG_ROOT/step3-oks.log"
FAILS_LOG="$LOG_ROOT/step3-fails.log"
ERRORS_LOG="$LOG_ROOT/step3-errors.log"

: > "$RUN_LOG"; : > "$OKS_LOG"; : > "$FAILS_LOG"; : > "$ERRORS_LOG"

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R $USER:$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by $USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------

mapfile -d '' manifests < <(find "$PWD" -type f -name "ALBUM.sha512sums.txt" -print0 | LC_ALL=C sort -z)

total=${#manifests[@]}
i=0

if [ "$total" -eq 0 ]; then
    echo "ALERT: No ALBUM.sha512sums.txt files found under $PWD."
    echo "Nothing to verify -- check that you are in the correct directory." | tee -a "$RUN_LOG"
    exit 1
fi

for m in "${manifests[@]}"; do
    i=$((i+1))
    dir_path=$(dirname "$m")
    album_name=$(basename "$dir_path")
    artist_name=$(basename "$(dirname "$dir_path")")
    label="$artist_name-$album_name"

    err=$(mktemp "$LOG_ROOT/step3-temp.XXXXXX")
    (
        cd "$dir_path" || exit 1
        sha512sum -c --quiet --strict "ALBUM.sha512sums.txt" 2>&1
    ) > "$err"
    rc=$?

    if [ $rc -ne 0 ]; then
        echo "FAIL [$i/$total] $label"
        echo "$label" >> "$FAILS_LOG"
        sed "s/^/[$i\/$total] ERROR: $label :: /" "$err" >> "$ERRORS_LOG"
    else
        echo "OK   [$i/$total] $label"
        echo "$label" >> "$OKS_LOG"
    fi
    rm -f "$err"
done | tee "$RUN_LOG"

echo "SUMMARY: $total album manifest(s) verified." | tee -a "$RUN_LOG"

```

--- Bash Script Step 3 End ---

-- Review Results

View the generated reports by running:

--- Bash Script Results 3 Start ---
```bash

cat "$LOG_ROOT/step3-run.log"
cat "$LOG_ROOT/step3-oks.log"
cat "$LOG_ROOT/step3-fails.log"
cat "$LOG_ROOT/step3-errors.log"

```
--- Bash Script Results 3 End ---

---

08. Step 4 – Create SHA-512 Checksums for Artist Folders

---

-- Purpose

This step establishes the baseline cryptographic fingerprint for files located directly within the parent Artist directories. It creates the standard `ARTIST.sha512sums.txt` manifest.

This step may be run from the Parent folder (processing every artist) or from within a single artist folder. It computes a single aggregate hash per album folder (excluding the `ALBUM.sha512sums.txt` file) and records it in the artist manifest.

-- Logging

This step writes `step4-run.log` (full transcript), `step4-oks.log` (successful artists), `step4-fails.log` (failed artists), and `step4-errors.log` (hash failures) to `$HOME/.logs/linux-audio-flac-sha512-checksum`.

--- Bash Script Step 4 Start ---
```bash

#!/usr/bin/env bash

#Step 4 - Create SHA-512 Checksums for Artist Folders

LOG_ROOT="$HOME/.logs/linux-audio-flac-sha512-checksum"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/step4-run.log"
OKS_LOG="$LOG_ROOT/step4-oks.log"
FAILS_LOG="$LOG_ROOT/step4-fails.log"
ERRORS_LOG="$LOG_ROOT/step4-errors.log"

: > "$RUN_LOG"; : > "$OKS_LOG"; : > "$FAILS_LOG"; : > "$ERRORS_LOG"
err=$(mktemp "$LOG_ROOT/step4-temp.XXXXXX")

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R $USER:$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by $USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------

generate_artist_checksums() {
    local artist_dir="$1"
    local artist_name
    artist_name=$(basename "$artist_dir")

    (
        cd "$artist_dir" || exit 1
        > ARTIST.sha512sums.txt

        find . -mindepth 1 -maxdepth 1 -type d -print0 |
        LC_ALL=C sort -z |
        while IFS= read -r -d '' album; do
            name=$(basename "$album")
            hash=$(cd "$album" 2>/dev/null && find . -type f ! -name ALBUM.sha512sums.txt -print0 | LC_ALL=C sort -z | xargs -0 sha512sum | sha512sum | cut -d" " -f1)

            if [ -n "$hash" ]; then
                printf "%s  %s\n" "$hash" "$name" >> ARTIST.sha512sums.txt
            else
                echo "ERROR: Failed to hash album '$name' under artist '$artist_name'" >> "$err"
            fi
        done
    )
}

# Main execution loop piped to tee
{
    processed=0
    if [ -z "$(find . -mindepth 2 -maxdepth 2 -type d)" ]; then
        # Running inside a single artist folder
        artist_name=$(basename "$PWD")
        generate_artist_checksums "$PWD"
        echo "OK [1/1] $artist_name"
        echo "$artist_name" >> "$OKS_LOG"
        processed=1
    else
        # Running at the root of the music library with multiple artist folders
        mapfile -d '' artists < <(find . -mindepth 1 -maxdepth 1 -type d -print0 | LC_ALL=C sort -z)
        total=${#artists[@]}

        if [ "$total" -eq 0 ]; then
            echo "ALERT: No artist subdirectories found under $PWD."
            exit 1
        fi

        i=0
        for artist in "${artists[@]}"; do
            i=$((i + 1))
            artist_name=$(basename "$artist")

            generate_artist_checksums "$artist"
            echo "OK [$i/$total] $artist_name"
            echo "$artist_name" >> "$OKS_LOG"
        done
        processed=$total
    fi

    if [ -s "$err" ]; then
        sed "s/^/[ERROR] /" "$err" >> "$ERRORS_LOG"
    fi
    echo "SUMMARY: $processed artist folder(s) checksummed."
} | tee "$RUN_LOG"

rm -f "$err"

```

--- Bash Script Step 4 End ---

-- Review Results

View the generated reports by running:

--- Bash Script Results 4 Start ---
```bash

cat "$LOG_ROOT/step4-run.log"
cat "$LOG_ROOT/step4-oks.log"
cat "$LOG_ROOT/step4-fails.log"
cat "$LOG_ROOT/step4-errors.log"

```
--- Bash Script Results 4 End ---

---

09. Step 5 – Verification of Artist Folders

---

-- Purpose

This step validates the integrity of the top-level artist hashes against the current state of the files to detect missing or corrupted data.

This step may be run from the Parent folder (verifying every artist) or from within a single artist folder.

-- Logging

This step writes `step5-run.log` (full transcript), `step5-oks.log` (matched albums), `step5-fails.log` (missing albums and mismatches), and `step5-errors.log` (detailed mismatch information) to `$HOME/.logs/linux-audio-flac-sha512-checksum`.

--- Bash Script Step 5 Start ---
```bash

#!/usr/bin/env bash

set -o pipefail

#Step 5 - Verification of Artist Folders

LOG_ROOT="$HOME/.logs/linux-audio-flac-sha512-checksum"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/step5-run.log"
OKS_LOG="$LOG_ROOT/step5-oks.log"
FAILS_LOG="$LOG_ROOT/step5-fails.log"
ERRORS_LOG="$LOG_ROOT/step5-errors.log"

: > "$RUN_LOG"; : > "$OKS_LOG"; : > "$FAILS_LOG"; : > "$ERRORS_LOG"

# --- Permission check ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Cannot write to:"
    echo "$PWD"
    echo ""
    echo "Fix ownership with:"
    echo "sudo chown -R $(id -un):$(id -gn) \"$PWD\""
    echo "=========================================================="
    exit 1
fi

# Verify one artist folder
verify_artist() {
    local artist_dir="$1"
    local artist

    artist=$(basename "$artist_dir")

    echo
    echo "=== $artist ==="

    local checksum_file="$artist_dir/ARTIST.sha512sums.txt"

    if [ ! -f "$checksum_file" ]; then
        echo "MISSING ARTIST.sha512sums.txt in $artist"
        echo "MISSING: $artist_dir/ARTIST.sha512sums.txt" >> "$ERRORS_LOG"
        echo "$artist" >> "$FAILS_LOG"
        return 1
    fi

    local total_albums
    total_albums=$(grep -cve '^[[:space:]]*$' "$checksum_file")

    local count=0

    while IFS= read -r line; do

        # Ignore blank lines
        [ -z "$line" ] && continue

        local stored_hash
        local album

        # Split hash from album name
        stored_hash="${line%% *}"
        album="${line#*  }"

        count=$((count + 1))

        if [ ! -d "$artist_dir/$album" ]; then
            echo "MISSING ALBUM: $artist - $album"
            echo "MISSING ALBUM: $artist - $album" >> "$ERRORS_LOG"
            echo "$artist - $album" >> "$FAILS_LOG"
            continue
        fi

        local actual_hash

        actual_hash=$(
            cd "$artist_dir/$album" || exit 1

            find . \
                -type f \
                ! -name "ALBUM.sha512sums.txt" \
                -print0 |
            LC_ALL=C sort -z |
            xargs -0 sha512sum |
            sha512sum |
            awk '{print $1}'
        )

        if [ "$stored_hash" = "$actual_hash" ]; then
            echo "OK       [$count/$total_albums] $artist - $album"
            echo "$artist - $album" >> "$OKS_LOG"
        else
            echo "MISMATCH [$count/$total_albums] $artist - $album"
            echo "$artist - $album" >> "$FAILS_LOG"

            {
                echo "MISMATCH: $artist - $album"
                echo "Expected: $stored_hash"
                echo "Got:      $actual_hash"
                echo
            } >> "$ERRORS_LOG"
        fi

    done < "$checksum_file"
}

# Main execution

{
    echo "Starting Step 5 verification"
    echo "Location: $PWD"
    echo

    # If already inside an artist folder
    if [ -f "$PWD/ARTIST.sha512sums.txt" ]; then

        verify_artist "$PWD"

    else

        found_artist=false

        for d in ./*/; do
            [ -d "$d" ] || continue

            if [ -f "$d/ARTIST.sha512sums.txt" ]; then
                found_artist=true
                verify_artist "$d"
            fi
        done

        if [ "$found_artist" = false ]; then
            echo "ERROR: No artist folders with ARTIST.sha512sums.txt found."
            exit 1
        fi

    fi

    echo
    echo "Verification complete."

    oks=$(wc -l < "$OKS_LOG" 2>/dev/null || echo 0)
    fails=$(wc -l < "$FAILS_LOG" 2>/dev/null || echo 0)
    echo "SUMMARY: $oks album(s) verified OK, $fails album(s) failed."

    if [ -s "$ERRORS_LOG" ]; then
        echo
        echo "Errors were found."
        echo "See: $ERRORS_LOG"
        exit 1
    fi

} | tee "$RUN_LOG"

```

--- Bash Script Step 5 End ---

-- Review Results

View the generated reports by running:

--- Bash Script Results 5 Start ---
```bash

cat "$LOG_ROOT/step5-run.log"
cat "$LOG_ROOT/step5-oks.log"
cat "$LOG_ROOT/step5-fails.log"
cat "$LOG_ROOT/step5-errors.log"

```
--- Bash Script Results 5 End ---

---

10. Step 6 – Auditing the Library

---

-- Purpose

This step audits the structural integrity of the library itself. It scans for any directories that are missing their required manifest files, or files that violate the strict naming convention, ensuring nothing escapes the verification safety net.

-- Logging

This step writes `step6-run.log` (full transcript), `step6-rogue-names.log` (non-compliant checksum filenames), and `step6-missing-manifests.log` (directories with files but no manifest) to `$HOME/.logs/linux-audio-flac-sha512-checksum`.

--- Bash Script Step 6 Start ---
```bash

#!/usr/bin/env bash

#Step 6 - Auditing the Library

LOG_ROOT="$HOME/.logs/linux-audio-flac-sha512-checksum"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/step6-run.log"
ROGUE_LOG="$LOG_ROOT/step6-rogue-names.log"
MISSING_LOG="$LOG_ROOT/step6-missing-manifests.log"

: > "$RUN_LOG"; : > "$ROGUE_LOG"; : > "$MISSING_LOG"

# --- UNIVERSAL PERMISSION & ROOT-LOCK CHECK ---
if [ ! -w "$PWD" ]; then
    echo ""
    echo "=========================================================="
    echo "CRITICAL ERROR: Write/operational permission denied on $PWD."
    echo "This often happens if a drive was formatted on a desktop system"
    echo "and the root filesystem is locked to 'root'."
    echo "=========================================================="
    echo "Off-script solution:"
    echo "Fix mount point or drive ownership by running:"
    echo "  sudo chown -R $USER:$USER \"$PWD\""
    echo ""
    exit 1
fi

if ! find "$PWD" ! -user "$USER" -print -quit 2>/dev/null | grep -q .; then
    : # All good, owned by current user
else
    echo "Notice: Some files/folders are not owned by $USER."
    read -rp "Fix permissions now using sudo chown? (y/N): " fix_choice
    if [[ "$fix_choice" =~ ^[Yy]$ ]]; then
        sudo chown -R "$USER:$USER" "$PWD" || { echo "chown failed. Check sudo privileges."; exit 1; }
    else
        echo "Aborting due to permission mismatch."; exit 1
    fi
fi
# ---------------------------------------------

echo "Auditing library for rogue files and missing manifests..." | tee "$RUN_LOG"

find "$PWD" -type f -iname "*.sha512*" \
    ! -name "ARTIST.sha512sums.txt" \
    ! -name "ALBUM.sha512sums.txt" \
    ! -name "Ignore.sha512sums.txt" \
    -print | tee -a "$ROGUE_LOG"

mapfile -d '' dirs < <(find "$PWD" -type d -print0 | LC_ALL=C sort -z)

for d in "${dirs[@]}"; do
    if [[ ! -f "$d/ARTIST.sha512sums.txt" && ! -f "$d/ALBUM.sha512sums.txt" && ! -f "$d/Ignore.sha512sums.txt" ]]; then
        shopt -s nullglob
        files=("$d"/*)
        shopt -u nullglob

        has_files=0
        for f in "${files[@]}"; do
            if [[ -f "$f" ]]; then
                has_files=1
                break
            fi
        done

        if [ $has_files -eq 1 ]; then
            echo "WARNING: Directory contains files but no manifest: $d" | tee -a "$MISSING_LOG"
        fi
    fi
done

echo "Audit complete. Check the step6 logs for detailed anomalies." | tee -a "$RUN_LOG"

rogue_count=$(wc -l < "$ROGUE_LOG" 2>/dev/null || echo 0)
missing_count=$(wc -l < "$MISSING_LOG" 2>/dev/null || echo 0)
echo "SUMMARY: $rogue_count rogue checksum filename(s), $missing_count directories missing manifests." | tee -a "$RUN_LOG"

```

--- Bash Script Step 6 End ---

-- Review Results

View the generated reports by running:

--- Bash Script Results 6 Start ---
```bash

cat "$LOG_ROOT/step6-run.log"
cat "$LOG_ROOT/step6-rogue-names.log"
cat "$LOG_ROOT/step6-missing-manifests.log"

```
--- Bash Script Results 6 End ---

---

11. Troubleshooting, Log File Reference & Disclaimer

---

-- Common Issues and Fixes

1. Checksum Mismatch ("computed checksum did NOT match")

If an audit reveals a mismatch, the file has been altered since the hash was generated. Do not regenerate the hash to "fix" the error, as you will be validating corrupted data. Delete the corrupted file and replace it from a known-good backup.

2. Missing File Errors ("FAILED open or read")

This occurs if a file tracked in the manifest has been renamed or deleted. You must delete the existing manifest in that directory and re-run the Generation scripts to establish a new baseline.

3. Permission Denied Errors

If tools like sha512sum fail to write data to a directory, it usually indicates a file ownership or permission issue. This is common when copying files from different filesystems, running tools in a virtual machine, or moving data from external drives. The built-in permission guard prompts for an interactive sudo chown, or you can run it manually:

--- Bash Script Start ---
```bash

sudo chown -R $USER:$USER /path/to/your/music/library
chmod -R u+rw /path/to/your/music/library

```
--- Bash Script End ---

-- Log File Reference & Understanding Your Log Files

Throughout this workflow, all scripts direct their tracking data to a dedicated log directory created in your home directory ($HOME/.logs/linux-audio-flac-sha512-checksum), regardless of which library folder you're working in. This ensures your music directories remain completely free of random text files and gives you a single, centralized place to review the results of your mass operations across every run.

Every log file lives directly in that one directory and is named for the step that produced it, using the pattern stepNN-logname.log (e.g. step1-run.log, step2-fails.log, step3-errors.log). The naming convention is consistent across all processing steps:

  1. stepNN-run.log
    The complete, raw output of the script. It lists every directory sequentially as it is processed, showing the OK or FAIL status for each one.

  2. stepNN-oks.log
    A filtered list containing only the albums (or artists) that were processed successfully.

  3. stepNN-fails.log
    A filtered list of albums (or artists) that encountered an issue. This is your primary "to-do" list for manual troubleshooting. If this file is empty, the step was a 100% success.

  4. stepNN-errors.log
    Contains the detailed standard error (stderr) output from the specific command-line utilities (like sha512sum). When an album shows up in the fails.log, you can check this error file to see exactly why it failed (e.g., "file not found," "computed checksum did NOT match," "permission denied").

Additionally, Step 6 writes two specialized logs:

  5. step6-rogue-names.log
    Lists any `.sha512*` files found that do not use the accepted generic manifest names: `ARTIST.sha512sums.txt`, `ALBUM.sha512sums.txt`, or `Ignore.sha512sums.txt` (the self-contained manifest that 15d of the moOde cleanup guide writes inside each Ignore folder).

  6. step6-missing-manifests.log
    Lists directories that contain files but are missing both required manifest files.

-- General Cleanup

Once you have reviewed the final logs, verified that your master library is fully processed, and completed your rsync transfer to the external backup drive, you can safely delete the entire $HOME/.logs/linux-audio-flac-sha512-checksum directory. It is completely independent of the audio files and is no longer needed once the project is finished.

\---------------------------------------------------------------------------------------

-- Disclaimer

This guide was developed through iterative collaborative effort between ChatGPT, Claude, Gemini, Mistral and the user. I cannot thank OpenCode project enough. I was about to give up on the other four (well, actually I did) when I came across OpenCode. I run a 10+ year old laptop yet OpenCode ran perfectly well, offloading the heaving lifting to an offsite server.

https://opencode.ai/
