# Open Recruitment NetSOS RISTEK 2026 — Web & Digital Forensics

**Glenn Josia Devano · PolarBear7**  
**Context:** NetSOS RISTEK Fasilkom UI Open Recruitment 2026  
**Portfolio scope:** Web Exploitation + Digital Forensics only

> This entry is a **technical recruitment/selection case study, not a competition placement**. I include it because the retained writeups show the same two areas I continue to focus on: web application security and digital forensics. My old repository also contained Binary Exploitation, Reverse Engineering, and Cryptography solutions from this recruitment set; those categories are intentionally omitted from this public portfolio.

## Why this case belongs in the portfolio

The two retained challenges cover very different forms of security reasoning:

| Challenge | Category | What it demonstrates |
|---|---|---|
| [Ledger of Lies](challenges/01-ledger-of-lies.md) | Web Exploitation | Source review across a polyglot microservice stack, trust-boundary analysis, vulnerability chaining, SSRF/CRLF-style request manipulation, JWT/HMAC design weaknesses, and remediation reasoning |
| [nyawit](challenges/02-nyawit.md) | Digital Forensics | Windows disk-image triage, registry analysis, browser-history reconstruction, timeline correlation, persistence identification, event/log review, PyInstaller malware extraction, and controlled data recovery |

## Technical highlights

### Web — cross-service trust failure

The web challenge was not a single-parameter bug. The useful path came from understanding how several services trusted one another:

```text
public health metadata
→ derivable application secret
→ weak JWT validation / shared-secret design
→ webhook request construction flaw
→ internal routing/header trust
→ protected audit service
```

The strongest lesson was that **individually moderate weaknesses can become critical when they cross service boundaries**. The portfolio version therefore emphasizes the attack graph, not the flag.

### Forensics — evidence correlation over single-artifact guessing

The forensic challenge began with an AccessData `.ad1` Windows image and a sequence of investigation questions. The retained notes walk through:

```text
host identity & registry
→ browser/download history
→ installer execution timeline
→ persistence via SSH authorized_keys
→ remote access & post-exploitation activity
→ ransomware location and execution
→ PyInstaller extraction
→ AES-CBC recovery of an encrypted victim file
```

This cleaned version improves the original notes by treating file timestamps as **corroborating evidence**, not infallible proof. Where a precise event time matters, log/process evidence should take priority and timestamps should be correlated across multiple artifacts.

## Tools represented

- Burp Suite / HTTP request analysis
- Python for exploit/proof automation
- Source-code review across Python, Go, JavaScript, and Rust
- AD1 extraction tooling
- `hivex` / Windows Registry analysis
- Impacket (`secretsdump`) in an offline CTF image
- SQLite browser-history analysis
- Windows event-log parsing
- PSReadLine history / filesystem metadata correlation
- PyInstaller extraction and Python bytecode inspection
- AES-CBC decryption for challenge data recovery

## Attribution & responsible-use note

The original files name **ultradiyow** as the challenge author for *Ledger of Lies* and **jay** as the challenge author for *nyawit*. Those names refer to the people who authored the challenges, not to authorship of this portfolio. This repository documents my retained solve/research notes for the authorized NetSOS recruitment environment.

Flags, NTLM hashes, SSH public keys, AES key/IV values, decrypted confidential content, event-only addresses, and other answer material are omitted from the public portfolio version. The source hashes in `evidence/SOURCE_MANIFEST.md` preserve traceability to the original local writeups.

## Interview-ready takeaway

This recruitment set is useful evidence because it predates the later competition results in this repository and shows a consistent technical direction: **understand the system first, reduce the attack/evidence surface, validate the hypothesis, and document the reasoning instead of only recording the final answer.**
