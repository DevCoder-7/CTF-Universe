# POLRI CTF 2026 — 1st Place

**Team:** CSUI — “cape lomba belum pernah menang”  
**My handle:** PolarBear7  
**Result:** **1st Place**  
**Stages:** Online Jeopardy qualifier → Offline Attack/Defense grand final  
**Portfolio scope:** Web Exploitation + Digital Forensics + A/D workflow

## Why this is a portfolio milestone

POLRI CTF is the clearest competition result in this repository because the final required a different skill set from a normal Jeopardy CTF. Qualifying required solving discrete challenges; the final required us to understand running services quickly, attack other teams, defend our own services, keep them available, and make decisions under round pressure.

I am only documenting the Web/Forensics material I can support from retained evidence. Other categories from the team writeup are intentionally outside this portfolio scope.

---

## Qualifier — Web Exploitation

### 1. `pointers` — preview service → normalization/traversal bypass

The application accepted a URL, fetched it server-side, and generated a preview. My first hypothesis was SSRF because the server was acting as a network client on behalf of the user. Straightforward private-network payloads were blocked, so the useful question became: **where does validation happen, and does the same normalized value reach the internal component?**

The retained solver shows a bypass built around alternate Unicode full-width representations for `.` and `/`, followed by traversal into an internal snapshot/file path. I converted this into an enumeration helper instead of testing payloads manually one by one.

What this challenge demonstrates:

- recognizing SSRF-like trust boundaries from application behavior;
- testing parser/normalization inconsistencies rather than assuming a blacklist is authoritative;
- moving from one working bypass to repeatable enumeration with Python;
- validating the final path with a deterministic script rather than a browser-only proof.

**Portfolio takeaway:** canonicalization bugs are often more important than the obvious initial vulnerability class. A validator and a downstream component can interpret the same string differently.

### 2. `wasm-mirage` — case-sensitive WAF route matching → SQL injection

This was a black-box product-search API protected by an Envoy-facing WAF. I started by collecting a clean baseline and identifying the proxy/backend behavior. Character probes showed that common SQLi tokens were blocked on the expected `/api/products` route.

The critical observation was that WAF enforcement was **case-sensitive on the route**, while the application still accepted a differently cased route. Requests to `/api/Products` bypassed the filter. From there I:

1. confirmed SQL injection;
2. determined the query shape/column count;
3. identified PostgreSQL from `version()`;
4. enumerated the relevant table/columns;
5. used a `UNION SELECT` to retrieve the protected value.

What matters to me in this solve is not the final payload—it is the mismatch between **edge security policy** and **backend routing semantics**.

---

## Qualifier — Digital Forensics

### 3. `whisper` — PCAP triage and cross-protocol reconstruction

The evidence was a small PCAP from a suspected workstation leak. I used Wireshark protocol hierarchy and targeted filters to separate ordinary traffic from the suspicious sequence. The retained analysis identifies DNS, ICMP, and HTTP as the useful protocols, including a suspicious DNS TXT exchange.

The solve ultimately required correlating data across protocols: an HTTP session value contributed key material and the exfiltrated content was recovered by decoding the staged traffic and applying RC4. This is a good example of why I prefer timeline/correlation thinking over looking for a single “magic packet.”

Skills demonstrated:

- protocol hierarchy triage;
- DNS/ICMP/HTTP filtering;
- extracting and correlating application-layer values;
- reconstructing a multi-step exfiltration path;
- validating decrypted output against expected structure.

### 4. `Berkas Perkara` — multimodal evidence, spectrogram + LSB + AES

The challenge bundle contained three evidence files. I first checked file types and SHA-256 values, then used structural inspection (`binwalk`) to rule out simple appended payloads. PNG metadata and file size suggested pixel-level steganography instead.

A WAV artifact produced a spectrogram containing the text `SAKSI-99`. That value became the password/keying input for the next stage. I then extracted the red-channel LSB stream from the PNG, parsed the embedded length/payload, separated the IV from ciphertext, derived an AES key using SHA-256 of the recovered password, and decrypted the payload using AES-CBC.

The important part of this chain was preserving the order of evidence:

```text
WAV → spectrogram clue
    ↓
password/key material
    ↓
PNG → LSB payload → IV + ciphertext
    ↓
key derivation → AES-CBC decrypt → recovered evidence
```

This challenge is representative of the forensic work I enjoy: multiple artifact types where one source explains how to interpret another.

---

## Grand Final — Attack/Defense

The final moved from “solve a challenge once” to a live operational loop. Each team ran vulnerable services while simultaneously attacking opponents and keeping its own services healthy.

### My portfolio-relevant working pattern

The retained final workspace contains Web-oriented recon/exploit notes, PCAP captures, automated scripts, and defensive patch records. Representative examples include:

- **JUDOL777:** API/JavaScript recon, auth/session behavior review, IDOR testing, race-condition probing, and API-key exposure analysis.
- **PINJOL:** profile/input testing, SSTI-style payload exploration, JWT/role hypotheses, file-read probes, and bot/browser interaction testing.
- **leakss:** a working RSA parity-oracle attack was documented for one service; although cryptographic in root cause, I keep it out of my personal category claim and only use the record as evidence of the A/D operating workflow.
- Defensive notes record the need to patch the root cause while preserving checker behavior, then re-test before deployment.

The strongest lesson from A/D was that a “secure” patch that breaks the service is still a losing patch. Exploitation, availability, rollback, and regression testing all matter at once.

### What I would explain in an interview

- how I decide whether a failed payload invalidates the vulnerability hypothesis or only the payload;
- why normalization mismatches are dangerous at reverse-proxy/WAF boundaries;
- how I reduce PCAP noise before deep analysis;
- how I turn a one-off exploit into reliable round automation;
- how I would patch the same root cause without breaking an A/D checker.

---

## Evidence retained

See [`evidence/SOURCE_MANIFEST.md`](evidence/SOURCE_MANIFEST.md). Raw flags, event endpoints, credentials, and large challenge files are deliberately excluded from this public-ready version.
