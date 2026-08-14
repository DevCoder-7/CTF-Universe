# Hack The Box — GCSB Business CTF 2026 (Project Nightfall)

**Format:** Jeopardy / multi-category Business CTF  
**Portfolio scope:** Web Exploitation + Digital Forensics  
**Achievement note:** I do **not** state a placement here because my current CV supplied for this portfolio does not contain an HTB placement claim. This section is evidence of technical work, not a fabricated ranking.

## Web Exploitation — retained solve artifacts

My local Web archive contains source plus solve material for four challenges. I focus on the vulnerability chains rather than reproducing flags.

### `trust-fall` — proxy trust → admin impersonation → unsandboxed formula execution

The retained writeup shows Grist behind Nginx configured to trust `X-Forwarded-User`. Because the edge proxy did not strip attacker-supplied versions of that header, the application accepted a forged administrator identity.

With install-admin access, a formula column could be added to a Grist document. The deployment used an unsandboxed Python formula mode, turning spreadsheet-formula functionality into OS-level Python execution.

Chain:

```text
client-controlled trusted header
→ admin impersonation
→ privileged document modification
→ unsandboxed Python formula
→ server-side file/code access
```

The key lesson is a proxy trust-boundary rule I now look for explicitly: forwarded identity headers are only safe when the edge component **removes any client copy and writes its own authoritative value**.

### `sarym_control` — route/middleware mismatch → admin configuration → query-string injection

The retained exploit shows that an admin settings route could be reached before the intended authorization middleware took effect. That allowed registration to be enabled with an administrator default role. A second flaw built an internal query string without percent-encoding user-controlled values, so an embedded `&command=...` became a new parameter downstream.

The internal Python service selected the first parsed `command` value and executed it through a shell path when it was not on the whitelist. This is a compact example of **cross-service parser disagreement**: the outer JSON object looked harmless, but its value changed meaning after being serialized into another protocol.

### `gridwatch` — SAML wrapping → authenticated SSRF → internal automation service

The retained local solver corresponds to a multi-stage chain: an authentication bypass in SAML handling provides privileged access; an authenticated relay/fetch feature becomes SSRF; and the SSRF reaches an internal Node-RED automation service without adequate administrative authentication. The practical lesson is that identity, SSRF, and internal-service trust often compose into a larger failure than any one bug suggests.

### `portalistic` — browser interaction to server compromise

This was the most complex retained Web artifact. The solve context/scripts show a chain involving supplier registration and admin review, bypassing CSRF assumptions, an XS-Leak-style audit oracle to recover a verification value, verified-account access, upload path traversal/arbitrary file write, middleware/route bypass behavior, and a Next.js restart leading to server-side code execution.

For a portfolio, the useful signal is the ability to keep state across a long chain and validate each boundary transition before moving to the next.

---

## Digital Forensics — large artifacts omitted, investigation evidence retained

The original forensic evidence was too large to upload here. The artifact tree supplied with this portfolio still gives useful evidence of the investigation scope.

### `Trust and Betrayal`

The retained tree contains a Windows forensic export with NTFS metadata, Registry/user artifacts, Event Logs, Recent/Jump List data, PowerShell paths, and a developer project named `VeldoriaPanel` containing `node_modules`, including Axios. This aligns with the challenge’s supply-chain investigation theme.

The portfolio value is the host-artifact triage process: identify the developer workspace, package ecosystem, execution/persistence traces, and timeline evidence rather than searching the disk for a literal flag string.

### `Open Wound`

The `hard/` working directory is clearly an IIS incident investigation workspace. Retained material includes:

- `disk.ad1` plus AD1 extraction/index scripts;
- IIS logs and IIS configuration history;
- `traffic.pcap` and HTTP reconstruction output;
- Ghidra project/decompile artifacts;
- `investigate_open_wound.py`;
- reconstructed/decrypted HTTP data;
- Registry and deleted/slack-space recovery directories.

This supports a workflow that combines disk forensics, IIS web-server artifacts, network traffic, and binary/module reverse analysis to understand a malicious IIS component and its communications.

### `Stay Hydrated`

The `insane/forensics_stay_hydrated/` workspace retains a `C.vhdx`, `D.E01`, password/recovery candidate scripts, and multiple recovery logs/`.rec` files. The official challenge theme requires Windows Data Deduplication recovery after ransomware, reconstructing keylogger-derived credentials, opening KeePass, and using the recovered project key to restore packaged source from the Dedup Chunkstore.

Because the raw images are not present in this portfolio package, I document the retained artifact evidence and investigation objective without pretending to reproduce the entire recovery from scratch here.

---

## Why this HTB section matters

The Web set pushed me toward longer exploitation chains with multiple trust boundaries. The Forensics set pushed in the opposite direction: large evidence, careful reduction, and correlation across filesystem, server, network, and application artifacts.

Together they reinforce the two areas I want CTF-Universe to communicate: **Web exploitation with root-cause reasoning** and **Digital Forensics with evidence discipline**.

See [`evidence/SOURCE_MANIFEST.md`](evidence/SOURCE_MANIFEST.md) and [`evidence/forensics-retained-artifacts.txt`](evidence/forensics-retained-artifacts.txt).
