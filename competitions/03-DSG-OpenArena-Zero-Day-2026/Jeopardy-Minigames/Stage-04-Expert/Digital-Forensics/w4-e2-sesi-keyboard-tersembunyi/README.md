# W4-E2 — Sesi Keyboard Tersembunyi

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Jeopardy / Minigames — Stage 04 — Expert  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

**Evidence pattern:** USB HID reconstruction + editing semantics + AES-128 recovery.

## Investigation / exploitation path

The retained PCAP is accompanied by a USB HID usage table and an encrypted note. Reconstruct keyboard reports, correctly handle Shift and Backspace rather than naively mapping keycodes, recover the typed key material, then use it to decrypt the retained AES-128 challenge note.

## Root cause / key observation

HID reconstruction is a stateful interpretation problem: modifier keys and editing keys change the final text, so packet-to-character mapping alone is insufficient.

## What this demonstrates

- evidence triage
- artifact correlation
- reconstruction instead of flag-string searching
- validation against the challenge format

## Portfolio note

Exact flags and sensitive recovered values are omitted from the public-ready portfolio.
