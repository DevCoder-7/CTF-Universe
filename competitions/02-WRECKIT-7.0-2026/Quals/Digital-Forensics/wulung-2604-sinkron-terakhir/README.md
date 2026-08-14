# Wulung-2604: Sinkron Terakhir

**Event:** WRECK-IT 7.0  
**Stage:** Quals — Jeopardy  
**Category:** Digital Forensics  
**Evidence status:** Confirmed

## Summary

A Linux build VM was seized after suspicious package-synchronization activity. The original OVA/VMDK was too large for this public portfolio, so the write-up records the evidence path and the small extracted artifacts instead of redistributing the disk image.

## Investigation / exploitation path

1. Verify the supplied archive/checksum and extract the OVA.
2. Identify the embedded `Wulung-2604-disk001.vmdk` and enumerate it for terms related to sync/cache/package/token/registry/preload.
3. Extract a minimal evidence set instead of mounting/copying the entire disk: the netcache configuration, package-registry log, systemd unit, sync binary path, preload configuration, and encrypted final artifact metadata.
4. Read `systemd-netcache.service` to identify `/usr/local/sbin/netcache-syncd`.
5. Recover the `node_id` from the netcache configuration.
6. Recover the package-publish token hash from the registry log.
7. Read the recovery instructions showing AES-256-CBC + PBKDF2 and the passphrase format `<node_id>:<token_hash>`.
8. Reconstruct the passphrase and decrypt the retained encrypted artifact with OpenSSL.
9. Validate the recovered CTF output without publishing it here.

## Root cause / key observation

The answer was not “search the disk for `flag`.” The key material was split across **service configuration + application log + encrypted spool artifact**, and the systemd unit explained which binary/service connected those artifacts.

## What this demonstrates

- OVA/VMDK triage
- selective artifact extraction
- Linux systemd/service analysis
- configuration/log correlation
- cryptographic recovery from forensic evidence
- evidence minimization for large disk cases
