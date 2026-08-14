# Stay Hydrated

**Event:** Hack The Box GCSB Business CTF 2026  
**Stage:** Jeopardy  
**Category:** Digital Forensics  
**Evidence status:** Artifact-tree evidence retained; large source images omitted

## Summary

Windows Data Deduplication / ransomware recovery

## Investigation / exploitation path

The retained workspace includes a Windows VHDX, an E01 evidence image reference, password/recovery candidate scripts, and recovery logs. The documented challenge objective involves reconstructing Data Deduplication data after ransomware, recovering keylogger-derived credentials, opening a password database, and using the recovered project key to restore source/package data from the Dedup ChunkStore.

## Root cause / key observation

The public portfolio preserves the **investigation plan and retained artifact scope**, not a fabricated full replay of evidence images that were too large to upload.

## What this demonstrates

- large-evidence triage
- filesystem/host artifact reduction
- network/server correlation
- evidence-discipline under incomplete portability

## Portfolio note

Raw forensic images are intentionally absent. See the event evidence manifest/artifact inventory.
