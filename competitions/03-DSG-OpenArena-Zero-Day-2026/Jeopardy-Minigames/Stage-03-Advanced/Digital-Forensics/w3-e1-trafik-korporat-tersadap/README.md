# W3-E1 — Trafik Korporat Tersadap

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Jeopardy / Minigames — Stage 03 — Advanced  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

**Evidence pattern:** TLS traffic decryption using retained key log.

## Investigation / exploitation path

The challenge provides an encrypted traffic capture plus `sslkeys.txt`. Load the key log into Wireshark/TLS preferences (or an equivalent offline parser), decrypt the relevant TLS session, then inspect the recovered application traffic for the exfiltrated data.

## Root cause / key observation

Encrypted PCAP analysis becomes straightforward when endpoint key material is legitimately available; the skill is correlating the key log with the correct session and then rebuilding the application flow.

## What this demonstrates

- evidence triage
- artifact correlation
- reconstruction instead of flag-string searching
- validation against the challenge format

## Portfolio note

Exact flags and sensitive recovered values are omitted from the public-ready portfolio.
