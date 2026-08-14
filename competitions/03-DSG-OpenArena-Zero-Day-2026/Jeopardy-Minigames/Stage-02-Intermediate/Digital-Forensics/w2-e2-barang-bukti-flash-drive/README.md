# W2-E2 — Barang Bukti Flash Drive

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Jeopardy / Minigames — Stage 02 — Intermediate  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

**Evidence pattern:** FAT16 disk-image recovery.

## Investigation / exploitation path

The evidence is a bit-for-bit FAT16 flash-drive image that appears empty in normal viewing. The forensic path is to inspect filesystem structures and deleted directory entries/data rather than trust the mounted directory listing, then recover the deleted challenge file from the image.

## Root cause / key observation

“Empty” at the filesystem UI level does not mean the underlying sectors or deleted entries are empty.

## What this demonstrates

- evidence triage
- artifact correlation
- reconstruction instead of flag-string searching
- validation against the challenge format

## Portfolio note

Exact flags and sensitive recovered values are omitted from the public-ready portfolio.
