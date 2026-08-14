# W2-E1 — Jejak Memori

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Jeopardy / Minigames — Stage 02 — Intermediate  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

**Evidence pattern:** Memory fragment + process context + PID-derived obfuscation.

## Investigation / exploitation path

The supplied process list identifies a suspicious `svc_updater.exe` process with PID `4821` and notes that the process uses its own PID as an obfuscation key. The memory fragment contains structured markers (`CONFIG_BLOCK`, `PAYLOAD_START` / `PAYLOAD_END`). Isolate the marked payload, decode its representation, then reverse the PID-based obfuscation to recover the challenge data.

## Root cause / key observation

Process metadata is part of the deobfuscation key, so memory content and process-list evidence must be interpreted together.

## What this demonstrates

- evidence triage
- artifact correlation
- reconstruction instead of flag-string searching
- validation against the challenge format

## Portfolio note

Exact flags and sensitive recovered values are omitted from the public-ready portfolio.
