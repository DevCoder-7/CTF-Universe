# W4-E1 — Bayangan Digital

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Jeopardy / Minigames — Stage 04 — Expert  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

**Evidence pattern:** LiME memory analysis + heap deobfuscation.

## Investigation / exploitation path

Case metadata identifies a suspicious `python3` process (PID 1337), a LiME memory dump, and an obfuscation key encoded in the case suffix (`3E`). The retained dump contains a recognizable heap marker and obfuscated `FLAGDATA`. Locate the process/heap region, extract the marked bytes, reverse the single-byte obfuscation, and validate the decoded result.

## Root cause / key observation

The case file itself supplies investigation context that turns a large memory image into a bounded heap-recovery problem.

## What this demonstrates

- evidence triage
- artifact correlation
- reconstruction instead of flag-string searching
- validation against the challenge format

## Portfolio note

Exact flags and sensitive recovered values are omitted from the public-ready portfolio.
