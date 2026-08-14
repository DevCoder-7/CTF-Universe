# CTF Universe — Web Exploitation & Digital Forensics Portfolio

**Glenn Josia Devano · `PolarBear7`**  
Information Systems, Universitas Indonesia  
[GitHub](https://github.com/DevCoder-7) · [LinkedIn](https://linkedin.com/in/glennjosiadevano) · [Portfolio](https://portofolio-website-sand.vercel.app/)

> A curated record of my 2026 CTF work, organized as **Event → Stage → Category → Challenge** so a recruiter, interviewer, or security practitioner can quickly see what I worked on, how I approached it, and what technical evidence supports the write-up.

## At a glance

My current cybersecurity focus is **Web Exploitation / Application Security** and **Digital Forensics**. I use CTFs as controlled environments to practice attack-surface analysis, exploit validation, evidence reconstruction, root-cause analysis, and—when the format allows it—defensive patching and re-testing.

This repository intentionally does **not** try to archive every challenge my teams solved. It includes only work that is relevant to my Web/Forensics portfolio and that I can support from retained notes, source, artifacts, or competition write-ups. Categories handled mainly by teammates are excluded from my personal portfolio claim.

## Selected 2026 results represented here

| Event | Result / context | Coverage in this repository |
|---|---|---|
| [POLRI CTF 2026](competitions/01-POLRI-CTF-2026/README.md) | **1st Place** | Jeopardy Qualifier + Attack/Defense Final |
| [WRECK-IT 7.0](competitions/02-WRECKIT-7.0-2026/README.md) | **Finalist** | Jeopardy Qualifier + Attack/Defense Final |
| [OpenArena Zero Day CTF 2026 — DSG](competitions/03-DSG-OpenArena-Zero-Day-2026/README.md) | **Top 10 Honorable Mention** | Staged Jeopardy/Minigames + Final |
| [Hack The Box GCSB Business CTF 2026](competitions/04-HTB-GCSB-Business-CTF-2026/README.md) | Technical competition portfolio | Web + Forensics challenge work |
| [Open Recruitment NetSOS RISTEK 2026](assessments/01-Open-Recruitment-NetSOS-RISTEK-2026/README.md) | Recruitment technical assessment | Web + Digital Forensics |

The first three result labels above match the achievement wording used in my current CV. I do not add a placement claim for HTB where I do not have one documented in the supplied career material.

## How to navigate this repository

The directory layout follows the competition itself first, then the stage and category:

```text
CTF-Universe/
├── README.md
├── SECURITY.md
├── docs/
│   ├── METHODOLOGY.md
│   └── PORTFOLIO_SCOPE.md
│
├── competitions/
│   ├── 01-POLRI-CTF-2026/
│   │   ├── Quals/
│   │   │   ├── Web-Exploitation/<challenge>/README.md
│   │   │   └── Digital-Forensics/<challenge>/README.md
│   │   └── Finals/
│   │       └── Web-Exploitation/<service>/README.md
│   │
│   ├── 02-WRECKIT-7.0-2026/
│   │   ├── Quals/
│   │   │   ├── Web-Exploitation/
│   │   │   └── Digital-Forensics/
│   │   └── Finals/
│   │       ├── Web-Exploitation/
│   │       └── Digital-Forensics/
│   │
│   ├── 03-DSG-OpenArena-Zero-Day-2026/
│   │   ├── Jeopardy-Minigames/
│   │   │   └── Stage-XX/<category>/<challenge>/README.md
│   │   └── Finals/
│   │       ├── Web-Exploitation/
│   │       └── Digital-Forensics/
│   │
│   └── 04-HTB-GCSB-Business-CTF-2026/
│       ├── Web-Exploitation/
│       └── Digital-Forensics/
│
└── assessments/
    └── 01-Open-Recruitment-NetSOS-RISTEK-2026/
        ├── Web-Exploitation/
        └── Digital-Forensics/
```

### Why challenge-level folders?

Each challenge now has its own `README.md`. That keeps the technical narrative attached to the **actual challenge name and category**, instead of mixing unrelated techniques inside one long event write-up. Event-level READMEs are indexes and context; challenge-level READMEs are the technical evidence.

## Featured technical write-ups

If you only have a few minutes, these are representative of the two areas I currently emphasize:

- **Web — POLRI Quals:** [`wasm-mirage`](competitions/01-POLRI-CTF-2026/Quals/Web-Exploitation/wasm-mirage/README.md) — case-sensitive WAF routing mismatch → SQL injection.
- **Forensics — POLRI Quals:** [`Berkas Perkara`](competitions/01-POLRI-CTF-2026/Quals/Digital-Forensics/berkas-perkara/README.md) — spectrogram clue → LSB extraction → AES-CBC recovery.
- **Web — WRECK-IT Quals:** [`Enterprise`](competitions/02-WRECKIT-7.0-2026/Quals/Web-Exploitation/enterprise/README.md) — DOM XSS / legacy AngularJS behavior in an admin-bot flow.
- **Forensics — WRECK-IT Quals:** [`Wulung-2604: Sinkron Terakhir`](competitions/02-WRECKIT-7.0-2026/Quals/Digital-Forensics/wulung-2604-sinkron-terakhir/README.md) — OVA/VMDK triage, service/config correlation, cryptographic recovery.
- **Web — DSG Final:** [`Gerbang Berlapis`](competitions/03-DSG-OpenArena-Zero-Day-2026/Finals/Web-Exploitation/gerbang-berlapis/README.md) — GraphQL disclosure → admin pivot → ZIP Slip → RCE → privilege escalation.
- **Forensics — RISTEK:** [`nyawit`](assessments/01-Open-Recruitment-NetSOS-RISTEK-2026/Digital-Forensics/nyawit/README.md) — Windows incident reconstruction and offline ransomware recovery.

## Technical themes demonstrated

**Web / AppSec:** attack-surface mapping, WAF/proxy trust mismatches, SSRF, SQL injection, DOM XSS, JWT weaknesses, SSTI, XXE, unsafe archive extraction, file-upload validation, forwarded-header trust, GraphQL/API review, and Attack/Defense patching.

**Digital Forensics:** PCAP triage, DNS/HTTP/TLS correlation, image steganography, spectrogram analysis, memory-fragment analysis, Windows Registry/browser artifacts, FAT/ext filesystem recovery, VM/disk triage, OOXML analysis, HID reconstruction, and static malware/package analysis.

## Evidence, attribution, and public-release policy

- **Scope:** only Web Exploitation and Digital Forensics work is used for my personal portfolio narrative.
- **Team events:** a team result is credited as a team result; this repository does not imply I personally solved every challenge in an event.
- **Evidence status:** when an old solve path is reconstructed rather than fully preserved, the challenge page says so explicitly.
- **Redaction:** flags, event credentials, live callback URLs, API keys, cookies, and unnecessary target addresses are removed from the public-ready version.
- **Large artifacts:** disk images, VM images, memory dumps, PCAP bundles, malware, and dependency trees are not redistributed. Their presence and provenance are recorded in event `evidence/` manifests when available.
- **Environment:** all exploitation described here occurred in authorized CTF or recruitment-challenge environments.

For the reasoning standard used across the repository, see [`docs/METHODOLOGY.md`](docs/METHODOLOGY.md) and [`docs/PORTFOLIO_SCOPE.md`](docs/PORTFOLIO_SCOPE.md).
