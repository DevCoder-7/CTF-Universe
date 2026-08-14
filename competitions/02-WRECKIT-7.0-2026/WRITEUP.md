# WRECK-IT 7.0 — Finalist

**Team:** Pengen coba AnD (final working context also used “CobaAnd”)  
**Members in qualifier writeup:** Ahmad Rizki Daffaa, Farel Boston Corinthians Nadeak, Glenn Josia Devano  
**Result:** **Finalist**  
**Stages:** Jeopardy qualifier → Attack/Defense final  
**Portfolio scope:** Web Exploitation + Digital Forensics + defensive A/D analysis

## Qualifier — Web Exploitation

### `Enterprise` — sanitizer configuration + legacy AngularJS → DOM XSS

The application rendered a user-controlled `note` after passing it through DOMPurify. At first glance that sounds safe, but the custom sanitizer profile explicitly allowed AngularJS attributes such as `ng-focus`, `ng-blur`, `ng-click`, and `autofocus`.

The admin/reviewer browser also loaded legacy AngularJS and exposed a policy object containing a sensitive ticket into Angular scope. That combination created the vulnerability: HTML remained “sanitized” in the ordinary XSS sense, but Angular directives were still executable program logic.

My solve chain was:

1. inspect `main.js` and `server.js` rather than treating the page as a pure black box;
2. identify the dangerous allowlisted directive attributes;
3. confirm the sensitive `enterprisePolicy.policy_ticket` value entered Angular scope;
4. craft an autofocus/`ng-focus` expression that redirects to a controlled callback with the policy value;
5. submit the generated URL to the Reviewer Queue;
6. verify the reviewer accepted the submission and observe the callback.

**Key lesson:** a sanitizer can be correct for HTML and still be unsafe for the framework that interprets the sanitized output. Security depends on the complete rendering pipeline.

### `Tailgate` — authentication-flow information exposure

This challenge was much simpler but still worth keeping as a contrast. After registering a normal account, the login flow exposed the protected value directly at the login endpoint. The useful skill here was not building a complex exploit; it was noticing that the application violated its own trust boundary and **not overcomplicating the solve**.

---

## Qualifier — Digital Forensics

### `Wulung-2604: Sinkron Terakhir` — VM image triage and Linux persistence chain

The original forensic attachment was too large to include in this portfolio, so the repository keeps an evidence tree and the analysis steps instead.

The supplied archive contained an OVA and checksum file. I verified/extracted the VM, obtained the VMDK, and used targeted filename enumeration before attempting a full filesystem review. Keywords around sync/cache/package/token/registry/preload quickly reduced the evidence set to a small group of high-value paths:

```text
/etc/.cache/netcache.conf
/etc/systemd/system/systemd-netcache.service
/usr/local/sbin/netcache-syncd
/etc/ld.so.preload
/home/forensic/projects/telemetry-agent/package-registry.log
/var/spool/.cache/final_flag.README
/var/spool/.cache/final_flag.enc.b64
```

The systemd service showed that `systemd-netcache.service` launched `/usr/local/sbin/netcache-syncd`. From there the investigation could connect service persistence, configuration (`node_id`), package-registry activity, loader behavior, and the encrypted staged artifact.

Instead of extracting the entire multi-gigabyte disk repeatedly, I used 7-Zip to pull only the relevant paths into an `analysis/extracted/` directory. That is the forensic practice I want this writeup to show: **reduce the evidence set without losing provenance**.

The exact raw VM image is intentionally not redistributed here; the retained directory tree is in [`evidence/wulung-artifact-tree.txt`](evidence/wulung-artifact-tree.txt).

---

## Final — Attack/Defense

The final working context shows a source-assisted A/D workflow: audit each service, identify root cause, write a minimal patch, preserve checker compatibility, test, deploy through Git, and automate attack actions that were already validated.

For this portfolio I focus on two services aligned with my Web/Forensics interests.

### `Kurir` — webhook/fetch service and SSRF hardening

The service acted as a URL fetcher/webhook processor. The earlier guard relied on literal host blacklists, which are fragile against alternate IP representations, IPv6, DNS behavior, userinfo, redirects, and internal address classes.

The defensive design in the retained notes moved toward:

- strict URL parsing;
- allowlisting `http`/`https`;
- rejecting credentials in URLs;
- canonical hostname/IP handling;
- blocking loopback, private, link-local, multicast, unspecified, and reserved ranges;
- resolving and validating **all** DNS answers;
- validating redirects again;
- limiting input size and using strict Base64/JSON parsing for the render path.

This is stronger portfolio evidence than a single SSRF payload because it shows the remediation model and the bypass classes a robust fix must consider.

### `Evidentia` — unsafe Python evaluation + file handling

The service exposed timeline operations where user input reached Python `eval()`. A blacklist attempted to make that safe, but the fundamental root cause remained: attacker-controlled text was still being interpreted as Python code.

The correct defensive direction was to delete dynamic evaluation and map a small allowlist of permitted operations to server-side functions. A second issue in metadata processing allowed user-controlled content to influence a file path, creating arbitrary-file-open behavior even when the downstream EXIF parser did not always reveal the file contents cleanly.

The final notes therefore emphasize two principles:

1. **remove the code-execution primitive**, do not grow the blacklist;
2. confine file handling to a dedicated upload directory using server-generated names, `realpath` checks, symlink rejection, size/type validation, and least privilege.

### A/D lesson

The final reinforced a workflow I now use in security projects:

```text
find root cause
→ reproduce safely
→ patch minimally
→ syntax/functional test
→ security regression test
→ health/checker test
→ deploy
→ monitor / rollback if needed
```

That is much closer to AppSec and security engineering work than a one-shot CTF flag hunt.

---

## Evidence retained

See [`evidence/SOURCE_MANIFEST.md`](evidence/SOURCE_MANIFEST.md) and [`evidence/wulung-artifact-tree.txt`](evidence/wulung-artifact-tree.txt). Large forensic images and event secrets are intentionally omitted.
