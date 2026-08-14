# nyawit — Digital Forensics Deep Dive

**Event:** Open Recruitment NetSOS RISTEK 2026  
**Challenge author:** jay  
**Category:** Digital Forensics  
**Challenge format:** Windows `.ad1` disk image + 18 investigation questions  
**Public status:** Solved; flag and sensitive answer material redacted

> The original challenge explicitly warned that the evidence contained live malware. Analysis therefore belongs in an isolated forensic/sandbox environment; the malware sample itself is intentionally not included in this repository.

## Case summary

The scenario began with a user who searched for a cracked Microsoft Word installer. After executing what appeared to be an installer, the host was later found with inaccessible confidential data.

The investigation required reconstructing the incident from a Windows disk image rather than simply locating a single hidden flag. The retained work covered four phases:

```text
host baseline
→ initial access & execution
→ persistence / remote access
→ ransomware analysis & recovery
```

## 1. Evidence preparation and host baseline

The evidence was supplied as an AccessData `.ad1` image. Because AD1 is not a normal mountable filesystem image, the working notes used dedicated AD1 extraction tooling before examining the contained Windows artifacts.

### Registry triage

The first questions were answered from offline Registry hives:

- `SYSTEM` → computer name, time zone, network configuration;
- `SOFTWARE` → Windows product/version information;
- `SAM` + `SYSTEM` → offline credential-hash extraction for the CTF question.

Representative commands from the retained workflow:

```bash
hivexget /tmp/SYSTEM \
  '\ControlSet001\Control\ComputerName\ComputerName' \
  'ComputerName'

hivexget /tmp/SYSTEM \
  '\ControlSet001\Control\TimeZoneInformation' \
  'TimeZoneKeyName'

hivexget /tmp/SOFTWARE \
  '\Microsoft\Windows NT\CurrentVersion' \
  'ProductName'

impacket-secretsdump -sam /tmp/SAM -system /tmp/SYSTEM LOCAL
```

The credential hash recovered for the challenge is not reproduced publicly.

### Why this matters

This phase established the machine context before timeline analysis. In an incident investigation, timezone and host identity are foundational: later timestamps are easy to misinterpret if the system's local-time configuration is unknown.

## 2. Browser history and initial-access reconstruction

The retained notes then moved to Microsoft Edge/Chromium history. Chromium stores browsing activity in SQLite databases, including URL visits and download records.

The investigation used the browser history to establish:

- when the suspicious site was first visited;
- what file was downloaded;
- the source host/port used by the challenge attacker;
- the local destination of the download.

Representative query shape:

```sql
SELECT target_path, tab_url
FROM downloads;
```

Chromium/WebKit timestamps were converted from microseconds since the 1601 epoch into Unix/local time before being placed on the incident timeline.

### Timeline-quality improvement over the original notes

The original working writeup sometimes treated a single filesystem timestamp as an exact execution boundary. For a recruiter-facing forensic report, that is too strong. Filesystem `atime`, `mtime`, and `ctime` can be affected by OS behavior, extraction tools, mount options, and subsequent access.

The stronger method is:

```text
filesystem timestamp
+ browser/download record
+ process/event-log evidence
+ script history / other host artifacts
= corroborated timeline
```

Accordingly, this cleaned writeup treats file metadata as **corroborating evidence**, not as an infallible timestamp oracle.

## 3. Installer execution and persistence

The malicious installer path contained a PowerShell script. The retained notes identified that the script planted an attacker-controlled SSH public key into the user's `authorized_keys` file.

That behavior maps to:

```text
MITRE ATT&CK T1098.004 — Account Manipulation: SSH Authorized Keys
```

The actual public key is redacted from this portfolio.

### Investigation approach

Useful artifacts included:

- the PowerShell script itself;
- the user's `.ssh/authorized_keys`;
- Windows event/process records;
- PowerShell/PSReadLine history;
- filesystem timestamps around the installer directory.

This phase turned the case from “malicious download” into a persistence story: the attacker did not just execute code once; the system was modified to support later remote access.

## 4. Remote access and post-exploitation activity

The working notes correlated successful remote authentication with later command activity and file transfer. PSReadLine/console history was used to recover commands executed after access, while process/event artifacts were used to place those actions on the timeline.

The challenge's transfer mechanism was identified as `scp`, consistent with the SSH-based persistence already found.

For public release, the exact user/account details, source IPs, keys, and answer strings are intentionally omitted.

## 5. Ransomware discovery

A suspicious executable mimicking a Windows system-process name was located under a temporary Windows directory. The retained notes linked its execution to the point at which encrypted files began appearing.

The important forensic question was not merely “where is the binary?” but:

1. Is it part of the incident timeline?
2. How was it transferred?
3. What execution evidence exists?
4. How does it transform victim data?
5. Can the encrypted challenge file be recovered safely without executing the malware?

## 6. Static extraction of a PyInstaller-packed payload

The ransomware binary was packaged with PyInstaller. Rather than execute the sample, the analysis extracted its embedded Python bytecode and inspected constants from the resulting `.pyc`.

Representative workflow:

```bash
python3 pyinstxtractor.py <redacted-malware>.exe
```

Then, in a controlled analysis directory:

```python
import marshal

with open('<extracted-module>.pyc', 'rb') as f:
    f.read(16)  # Python .pyc header for the challenge artifact
    code = marshal.load(f)

for value in code.co_consts:
    if isinstance(value, bytes):
        # inspect candidate cryptographic constants
        print(len(value), repr(value[:8]))
```

The retained challenge solution recovered an AES key and IV from the bytecode. Those values are redacted here.

## 7. Controlled decryption / data recovery

The malware logic used AES-CBC. After recovering the key material statically, the encrypted challenge file could be decrypted without running the ransomware:

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b'<redacted>'
iv = b'<redacted>'

ciphertext = open('<victim-file>.enc', 'rb').read()
plaintext = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext), 16)
print(plaintext.decode(errors='replace'))
```

The recovered confidential content and final flag are omitted from this public portfolio.

## Incident narrative

The evidence supports the following high-level chain in the CTF scenario:

```text
user visits suspicious download source
→ downloads fake Office installer
→ PowerShell payload executes
→ SSH authorized key is planted for persistence
→ attacker returns via remote access
→ commands are executed and ransomware is transferred
→ ransomware runs from a temporary directory
→ victim data is encrypted
→ static malware extraction reveals encryption material
→ encrypted challenge file is recovered offline
```

## Forensic lessons

### 1. Establish timezone before building a timeline

A technically correct timestamp converted using the wrong timezone is still a wrong incident timeline.

### 2. Correlate artifacts

Browser history, Registry data, filesystem metadata, event logs, console history, and malware internals each answer different questions. Confidence comes from correlation.

### 3. Do not execute evidence unnecessarily

The malware key material was recoverable statically from a PyInstaller package. Running the ransomware was unnecessary and would have increased risk.

### 4. Separate observation from inference

A timestamp or process record should be described according to what it actually proves. This portfolio version deliberately avoids presenting circumstantial metadata as certainty.

## Skills demonstrated

- forensic image handling;
- offline Windows Registry analysis;
- browser SQLite analysis;
- credential-artifact extraction in a CTF image;
- timeline reconstruction;
- persistence identification;
- Windows event and command-history analysis;
- malware packaging analysis;
- Python bytecode inspection;
- AES-CBC data recovery;
- evidence-based incident narration.
