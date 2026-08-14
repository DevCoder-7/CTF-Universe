# CTF Universe — Web Exploitation & Digital Forensics Portfolio

**Glenn Josia Devano · PolarBear7**  
Information Systems — Universitas Indonesia  
[GitHub](https://github.com/DevCoder-7) · [LinkedIn](https://linkedin.com/in/glennjosiadevano) · [Portfolio](https://portofolio-website-sand.vercel.app/)

This repository is a curated cybersecurity portfolio built from authorized 2026 CTF competition and recruitment-challenge material I retained. It is intentionally narrower than my old CTF archive: **only Web Exploitation and Digital Forensics work is presented here**, because those are the areas I want this repository to represent as my own technical contribution.

## Why this repository exists

A long folder of flags is not very useful to a recruiter or interviewer. This version is organized around evidence of how I work:

- identifying an attack surface and forming hypotheses;
- validating a vulnerability rather than stopping at a guess;
- writing or adapting small automation scripts when repetition matters;
- reconstructing forensic evidence from disk, network, document, and host artifacts;
- in Attack/Defense finals, thinking about **both exploitation and remediation**;
- documenting what is confirmed, what is inferred, and what is deliberately omitted.

## Selected 2026 competition & recruitment case studies

| Event / case study | Result / context | Format covered here | Portfolio focus |
|---|---|---|---|
| [POLRI CTF 2026](competitions/01-POLRI-CTF-2026/WRITEUP.md) | **1st Place** | Online Jeopardy Qualifier + Offline Attack/Defense Final | Web, Forensics, A/D security workflow |
| [WRECK-IT 7.0](competitions/02-WRECKIT-7.0-2026/WRITEUP.md) | **Finalist** | Jeopardy Qualifier + Attack/Defense Final | Web, Forensics, patching & regression thinking |
| [OpenArena Zero Day CTF 2026 — DSG](competitions/03-DSG-OpenArena-Zero-Day-2026/WRITEUP.md) | **Top 10 Honorable Mention** | Staged Jeopardy / Minigames + Final | Web exploitation chains, incident-style forensics |
| [Hack The Box GCSB Business CTF 2026](competitions/04-HTB-GCSB-Business-CTF-2026/WRITEUP.md) | Technical competition portfolio | Jeopardy | Web + disk/host forensics evidence retained locally |
| [Open Recruitment NetSOS RISTEK 2026](competitions/05-Open-Recruitment-NetSOS-RISTEK-2026/WRITEUP.md) | **Recruitment technical assessment** — not a competition placement | Challenge set | Microservice web exploit chain + Windows incident forensics |

## Repository structure

```text
CTF-Universe-Portfolio-2026/
├── README.md
├── SECURITY.md
├── docs/
│   ├── METHODOLOGY.md
│   ├── MIGRATION.md
│   └── PORTFOLIO_SCOPE.md
└── competitions/
    ├── 01-POLRI-CTF-2026/
    │   ├── WRITEUP.md
    │   └── evidence/
    ├── 02-WRECKIT-7.0-2026/
    │   ├── WRITEUP.md
    │   └── evidence/
    ├── 03-DSG-OpenArena-Zero-Day-2026/
    │   ├── WRITEUP.md
    │   └── evidence/
    ├── 04-HTB-GCSB-Business-CTF-2026/
    │   ├── WRITEUP.md
    │   └── evidence/
    └── 05-Open-Recruitment-NetSOS-RISTEK-2026/
        ├── WRITEUP.md
        ├── challenges/
        │   ├── 01-ledger-of-lies.md
        │   └── 02-nyawit.md
        └── evidence/
```

There are **four competition writeups plus one recruitment technical-assessment writeup**. The RISTEK entry also contains two challenge deep dives because the source material is detailed enough to support them. The `evidence/` files are supporting inventories/checksums, not separate achievement claims.

## Public-release hygiene

Flags, competition credentials, live callback URLs, API keys, cookies, and event-only target addresses are intentionally redacted. Large disk images, VM images, PCAPs, and bundled application dependencies are also omitted. Their presence is recorded through local artifact inventories and source-archive hashes where available.

> All techniques documented here were performed in authorized CTF environments. Nothing in this repository is an invitation to target third-party systems.
