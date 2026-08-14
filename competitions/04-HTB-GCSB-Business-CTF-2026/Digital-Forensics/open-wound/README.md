# Open Wound

**Event:** Hack The Box GCSB Business CTF 2026  
**Stage:** Jeopardy  
**Category:** Digital Forensics  
**Evidence status:** Artifact-tree evidence retained; large source images omitted

## Summary

IIS incident investigation across disk + web logs + PCAP + binary/module analysis

## Investigation / exploitation path

The retained workspace includes an AD1 image, IIS logs/configuration history, `traffic.pcap`, HTTP reconstruction output, Ghidra/decompile artifacts, Registry/deleted/slack-space recovery directories, and investigation scripts. The case therefore centers on correlating IIS server artifacts with network behavior and a malicious module/component rather than treating each evidence source independently.

## Root cause / key observation

The public portfolio preserves the **investigation plan and retained artifact scope**, not a fabricated full replay of evidence images that were too large to upload.

## What this demonstrates

- large-evidence triage
- filesystem/host artifact reduction
- network/server correlation
- evidence-discipline under incomplete portability

## Portfolio note

Raw forensic images are intentionally absent. See the event evidence manifest/artifact inventory.
