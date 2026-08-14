# Evidentia

**Event:** WRECK-IT 7.0  
**Stage:** Finals — Attack/Defense  
**Category:** Digital Forensics service theme (portfolio classification)  
**Evidence status:** Confirmed from retained A/D source review

## Summary

Evidentia was a forensic-triage dashboard with timeline and metadata functionality. The interesting security work came from reviewing how a forensic application processed analyst-controlled expressions and uploaded evidence.

## Investigation / exploitation path

1. Map the timeline endpoints and inspect `sandbox.py`, sanitization code, timeline handlers, and metadata processing.
2. Confirm that timeline parameters ultimately reach Python `eval()`.
3. Test the limits of the blacklist/sandbox and treat `eval()` itself as the root cause rather than chasing an endless list of blocked names.
4. Review the metadata endpoint and confirm that uploaded content can be interpreted as a filesystem path, creating an arbitrary-file-open primitive.
5. Patch by replacing expression evaluation with explicit operation mappings and by storing/processing uploads as server-owned files inside a constrained directory.
6. Re-test normal forensic functionality so the A/D checker remains healthy.

## Root cause / key observation

A security/forensics tool was itself unsafe because untrusted analyst input crossed into **code evaluation** and **filesystem path resolution**. Blacklists tried to contain dangerous primitives instead of removing them.

## What this demonstrates

- Python `eval()` risk analysis
- sandbox/blacklist review
- arbitrary file read/open reasoning
- secure evidence-upload handling
- defensive patch design and regression testing

## Defensive takeaway

Replace `eval()` with an explicit allowlist of supported operations. Never treat upload bytes as a path. Use server-generated filenames, realpath containment checks, symlink protections, size/type limits, and least-privileged parsers.
