# FLAC Library SHA-512 Verification

A collection of Bash scripts for generating, verifying, and auditing cryptographic hashes across music libraries on Linux.

## Overview

This repository contains automated procedures designed for strict data preservation and bit-rot auditing. It establishes a baseline of SHA-512 checksums to ensure library integrity, using a decentralized approach where manifests travel with the data they protect.

## Included Tools & Workflows

* **Stray File Audit:** Identifies unrecognized files before hashing to ensure a clean baseline.
* **Checksum Generation:** Creates `ARTIST.sha512sums.txt` and `ALBUM.sha512sums.txt` using relative pathing.
* **Integrity Verification:** Validates file integrity against existing manifests in strict mode.
* **Library Audit:** Scans for missing manifests or non-compliant checksum filenames.
* **Permission Check:** Automated checks and guidance to ensure your user has the correct read/write ownership and access to the library directory before executing mass batch operations.

## Prerequisites

Ensure the following command-line utilities are installed and available in your shell's PATH:

* `sha512sum`
* GNU Core Utilities (`find`, `sort`, `awk`, `grep`, `wc`, `basename`, `dirname`)

**Directory Permissions:** Before running any batch processing scripts, verify that your current user has full read and write permissions to the target library directory to prevent "Permission Denied" failures during execution.

## Recommended Workflow

A four part series to clean, verify, and lockdown securely the integrity of an audio file library. 

1. linux-audio-moode-prep: https://github.com/TerrapinATL/linux-audio-moode-prep

2. linux-audio-sha512-checksums: https://github.com/TerrapinATL/linux-audio-sha512-checksums

3. linux-os-nemo-sha512-shortcut:https://github.com/TerrapinATL/linux-os-nemo-sha512-shortcut

4. linux-audio-folder-recertification: https://github.com/TerrapinATL/linux-audio-folder-recertification

## Disclaimer

This file was created as a mix of AI generated content, user input, and user editing. It was a cooperative effort between Claude, Gemini, ChatGPT, and user.

## IMPORTANT

Your Original Library should be treated as immutable.

You should only work on a COPY of your Original Library when processing these scripts. The workflow is designed around creating a validated secondary copy, testing the results, and only then promoting that copy to become a replacement.

Before promotion, files should be cleaned, verified with flac -t, and protected with two layers of SHA-512 checksums.

NOTE: Step 5 – Verify Artist Folders contains the required verification logic for ARTIST.sha512sums.txt. ARTIST.sha512sums.txt stores hashes of album contents, not individual files. It must be verified using the Artist verification routine, which recreates each album hash and compares it. Do not use sha512sum -c on Artist manifests. Replacing this logic with standard checksum verification will cause Artist-level verification to fail because ARTIST.sha512sums.txt is designed as the upper tier of the two-tier verification system.

The purpose is to ensure you have a verifiable library that can be copied, backed up, and restored repeatedly while still matching the validated cleaned copy.
