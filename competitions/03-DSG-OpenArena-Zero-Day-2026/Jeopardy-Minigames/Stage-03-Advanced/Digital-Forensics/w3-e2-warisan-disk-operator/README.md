# W3-E2 — Warisan Disk Operator

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Jeopardy / Minigames — Stage 03 — Advanced  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

**Evidence pattern:** ext4 deleted-inode recovery.

## Investigation / exploitation path

Case notes state that a sensitive file existed under `/home/operator/` and was deleted shortly before acquisition. Analyze the ext4 image with deletion/timeline context, identify the deleted inode/data blocks, recover the file content, and validate the recovered CTF value.

## Root cause / key observation

The timeline narrows the search, while inode-level recovery avoids relying on current directory listings after deletion.

## What this demonstrates

- evidence triage
- artifact correlation
- reconstruction instead of flag-string searching
- validation against the challenge format

## Portfolio note

Exact flags and sensitive recovered values are omitted from the public-ready portfolio.
