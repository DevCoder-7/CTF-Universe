# W1-E2 — Lalu Lintas Jaringan

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Jeopardy / Minigames — Stage 01 — Beginner  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

**Evidence pattern:** PCAP HTTP triage + Base64 exfil value.

## Investigation / exploitation path

Strings/HTTP reconstruction from the retained PCAP shows mostly normal traffic plus a suspicious form POST containing a Base64-encoded `secret_token`. Decode the anomalous value and correlate it with the surrounding C2-like application flow.

## Root cause / key observation

The challenge rewards reducing network noise and inspecting application parameters rather than searching the capture for a literal flag string.

## What this demonstrates

- evidence triage
- artifact correlation
- reconstruction instead of flag-string searching
- validation against the challenge format

## Portfolio note

Exact flags and sensitive recovered values are omitted from the public-ready portfolio.
