# W1-E1 — Foto Mencurigakan

**Event:** OpenArena Zero Day CTF 2026 — DSG  
**Stage:** Jeopardy / Minigames — Stage 01 — Beginner  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

**Evidence pattern:** JPEG metadata + steganography.

## Investigation / exploitation path

The JPEG contains a comment with a password hint. After reading metadata, use the recovered password with a standard steganography extraction workflow to recover the embedded `hidden.txt`, then validate the recovered CTF output.

## Root cause / key observation

The useful sequence is **metadata first, extraction second**. A simple comment that looks incidental can be the keying material for hidden content.

## What this demonstrates

- evidence triage
- artifact correlation
- reconstruction instead of flag-string searching
- validation against the challenge format

## Portfolio note

Exact flags and sensitive recovered values are omitted from the public-ready portfolio.
