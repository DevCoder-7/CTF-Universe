# OpenArena Zero Day CTF 2026 — PT Digital Solusi Grup

**Result:** **Top 10 Honorable Mention**  
**Format represented in my local workspace:** staged Jeopardy/minigames + final round  
**Portfolio scope:** Web Exploitation + Digital Forensics

## Why I kept this event

This competition produced the broadest retained practice set in my 2026 workspace. Rather than publish dozens of shallow challenge notes, this portfolio condenses the event into the techniques I can actually evidence and explain.

An important limitation: the qualifier recap itself labels some old Web paths as **confirmed**, some as **high-confidence reconstruction**, and some as **inferred** because exact historical flags/steps were no longer present locally. I preserve that distinction here instead of presenting reconstruction as fact.

---

## Jeopardy / staged challenges — Web

### Confirmed: upload-validation failures → executable content

Two retained challenges (`W1-C3 Upload Dokumen` and `W2-C1 Portal Unggah Laporan`) preserve concrete payload artifacts. The pattern was not merely “upload PHP”:

- server-side extension filtering existed;
- `.htaccess` could alter Apache handler behavior;
- a `.jpg` could then be treated as PHP;
- alternate extensions/polyglot content were tested as validation bypasses.

This gave me a useful reusable checklist for upload surfaces: extension, MIME, magic bytes, storage path, web-server handler configuration, and whether uploaded files are ever interpreted rather than served as inert content.

### Confirmed: authenticated SSRF

`W3-C1 Portal Middleware Sentral` is retained in the event recap as an authenticated SSRF case. The important boundary is a “fetch/proxy” feature where an authenticated user can cause the server to access a URL. For a portfolio, the meaningful part is the validation model: canonicalize the destination, resolve DNS, reject internal address classes, and re-check redirects instead of relying on string filters.

### Confirmed: XXE → disclosure → admin pivot → SSTI

`W3-C3 Portal Solusi Terintegrasi` is the strongest qualifier Web chain preserved in the recap. XML import enabled external-entity abuse, which exposed source/secret material; that enabled an administrative pivot, followed by server-side template injection.

The chain is useful because it demonstrates **impact compounding**:

```text
parser weakness (XXE)
→ information disclosure
→ authentication/authorization pivot
→ SSTI
→ server-side execution / protected data access
```

A bug that looks like “only file disclosure” can become critical when exposed secrets unlock a second trust boundary.

---

## Jeopardy / staged challenges — Digital Forensics

The retained workspace contains multiple forensic challenge families, including image steganography, memory fragments, disk artifacts, intercepted traffic, and keyboard/session evidence. I am not claiming every forensic item in the full event list as an individually reconstructed solve; the strongest evidence-backed examples are below.

### Image / hidden-data triage

A Week 1 analyst note explicitly says a suspicious employee photo may contain hidden content. This is the kind of problem where I start with metadata, file structure, appended data, and pixel/steganography checks instead of guessing an encoder blindly.

### Memory-fragment analysis

A Week 2 incident note describes a memory fragment from a suspicious process, including acquisition noise and an internal marker. This reinforces a basic forensic habit: define the acquisition context and distinguish structured artifacts from noise before interpreting strings as evidence.

### Disk and traffic challenges

Later stages retained artifacts for “Warisan Disk Operator”, “Trafik Korporat Tersadap”, “Bayangan Digital”, and “Sesi Keyboard Tersembunyi”. I keep these at the portfolio-summary level because the exact historical execution path is not fully recoverable from the local recap alone.

---

## Final — Web exploitation chains

### `re-oa-M1: Gerbang Berlapis / KampusConnect`

This final challenge is well documented in my local writeup. The attack required several layers rather than one vulnerability:

1. enumerate public pages and discover GraphQL plus a public build identifier/service token;
2. inspect the GraphQL schema;
3. use a deprecated `adminAudit` field to obtain a sealed administrator token;
4. decode it using the public build identifier;
5. enter the admin surface with the recovered token;
6. abuse ZIP import to overwrite a JavaScript hook and obtain application-level RCE;
7. read application configuration and recover a local user credential;
8. pivot to SSH;
9. escalate privileges through a root-executed maintenance path;
10. verify root-level access.

For public release, endpoints, credentials, and flags are redacted. What matters is the reasoning: **deprecated API surface + weak secret protection + unsafe archive/import behavior + credential reuse + privileged automation** formed a complete compromise chain.

### `ArenaOps`

Another retained final writeup documents a Laravel application running with debug enabled. The analysis identified an exposed Ignition execution surface compatible with CVE-2021-3129, producing RCE as the web user. Application configuration then exposed database credentials that were reusable for a local account. The local account had constrained `sudoedit` rights on a maintenance file, and the installed sudo version was vulnerable to CVE-2023-22809, enabling root-file access.

This challenge was valuable because it connected:

- framework/version identification;
- public debug exposure;
- known-vulnerability validation;
- secret management failure;
- password reuse;
- local privilege escalation.

---

## Final — Digital Forensics

### `Insiden Lampiran`

The retained writeup starts with three artifacts: a DOCX, PCAP, and incident notes. Rather than opening the document, the analysis treats DOCX as OOXML and extracts it as a ZIP. Relationships reveal a linked OLE `htmlfile` object pointing to an external HTML template.

Network evidence then connects that document behavior to staged DNS/HTTP activity. The writeup describes an `ms-msdt:`-based execution chain with encoded PowerShell, key material derived from document properties and DNS beacons, and compressed/XORed data sent to an upload endpoint.

This is a strong portfolio example because the conclusion depends on **document structure + network capture + script decoding**, not any single artifact.

### Screenshot crop recovery

A second retained forensic writeup documents recovery from an image that had been visually cropped but not safely truncated at the file level. Bytes remaining after the logical PNG end still contained recoverable prior image data. The case demonstrates why “what the image viewer displays” and “what bytes the file still stores” are different forensic questions.

---

## What this event added to my skill set

- separating confirmed evidence from reconstruction;
- chaining several moderate Web weaknesses into full compromise;
- treating documents as containers and correlating them with PCAP evidence;
- analyzing file-format leftovers rather than trusting rendered appearance;
- writing technical notes that explain **root cause and remediation**, not only the flag.

See [`evidence/SOURCE_MANIFEST.md`](evidence/SOURCE_MANIFEST.md) for the retained source set.
