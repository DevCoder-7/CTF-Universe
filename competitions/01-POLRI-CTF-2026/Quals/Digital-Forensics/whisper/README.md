# whisper

**Event:** POLRI CTF 2026  
**Stage:** Quals — Jeopardy  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

The evidence was a small PCAP from a suspected workstation data leak. Traffic looked ordinary at first—DNS, ICMP, and HTTP—but the useful evidence was distributed across protocols rather than contained in one obviously malicious stream.

## Investigation / exploitation path

1. Use Wireshark protocol hierarchy and targeted filters to reduce the capture.
2. Identify an unusual DNS TXT exchange and correlate it with the surrounding HTTP/session activity.
3. Reconstruct the staged data handoff across DNS/ICMP/HTTP instead of treating each protocol independently.
4. Recover key material from the application-layer session context.
5. Decode/decrypt the transferred content and validate the recovered document structure.

## Root cause / key observation

The exfiltration path intentionally blended with normal-looking traffic and split information across multiple protocols. Correlation and ordering were more important than any single packet signature.

## What this demonstrates

- PCAP triage
- DNS/ICMP/HTTP filtering
- cross-protocol timeline correlation
- session/key-material extraction
- content reconstruction and validation
