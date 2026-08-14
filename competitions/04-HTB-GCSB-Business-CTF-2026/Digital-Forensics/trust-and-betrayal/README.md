# Trust and Betrayal

**Event:** Hack The Box GCSB Business CTF 2026  
**Stage:** Jeopardy  
**Category:** Digital Forensics  
**Evidence status:** Artifact-tree evidence retained; large source images omitted

## Summary

Windows developer-workstation / software-supply-chain investigation

## Investigation / exploitation path

The retained artifact tree contains NTFS metadata, Registry/user artifacts, Event Logs, Recent/Jump List data, PowerShell paths, Windows Search databases, and a developer project (`VeldoriaPanel`) with a Node.js dependency tree. The investigation scope is to identify the developer workspace, package/dependency activity, execution/persistence traces, and timeline evidence associated with the suspected supply-chain compromise.

## Root cause / key observation

The public portfolio preserves the **investigation plan and retained artifact scope**, not a fabricated full replay of evidence images that were too large to upload.

## What this demonstrates

- large-evidence triage
- filesystem/host artifact reduction
- network/server correlation
- evidence-discipline under incomplete portability

## Portfolio note

Raw forensic images are intentionally absent. See the event evidence manifest/artifact inventory.
