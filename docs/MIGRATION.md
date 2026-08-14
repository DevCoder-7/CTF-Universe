# Migration from the Previous CTF-Universe Archive

The previous archive (`CTF-Universe.zip`, SHA-256 `2fdd6f1f426742e3496ed66b3b7bc95de38228d7d2bfe4ea4ffa5fada44f5a2e`) was useful as a personal notebook but not yet recruiter-oriented. It contained the repository's `.git` directory, early PicoCTF material, ARA CTF, FINDIT preparation, Open Recruitment RISTEK, Pekris, inconsistent naming, challenge dumps, and writeups across categories that were not always my own primary contribution.

## Cleanup decision

This portfolio release intentionally does **not** copy that archive forward wholesale.

The new version:

- removes embedded `.git` internals from the distributed ZIP;
- limits the public story to four selected 2026 competitions plus one NetSOS RISTEK recruitment technical-assessment case study;
- limits personal technical claims to Web Exploitation and Digital Forensics;
- promotes results and competition format to the top-level index;
- replaces flag-centric notes with attack path, evidence, root cause, remediation, and lessons;
- removes raw flags, credentials, temporary endpoints, API keys, callbacks, and bulky evidence;
- keeps source hashes and artifact inventories so the work remains traceable;
- adds a `.gitignore` designed to prevent accidental recommit of multi-GB forensic images and secrets.

The old archive should be kept privately if historical notes are still useful, but the cleaned ZIP is the version intended for GitHub/portfolio presentation.

## Open Recruitment RISTEK exception

The first cleaned release omitted the Open Recruitment RISTEK material entirely. This revision restores **only the Web Exploitation and Digital Forensics portions** because they align with the portfolio's personal-scope rule and provide useful evidence of the technical path that preceded later NetSOS project work. Binary Exploitation, Reverse Engineering, and Cryptography recruitment notes remain excluded.
