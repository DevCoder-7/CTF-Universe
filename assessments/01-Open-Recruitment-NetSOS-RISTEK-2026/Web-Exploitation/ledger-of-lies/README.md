# Ledger of Lies — Web Exploitation Deep Dive

**Event:** Open Recruitment NetSOS RISTEK 2026  
**Challenge author:** ultradiyow  
**Category:** Web Exploitation  
**Public status:** Solved; flag redacted

## Challenge model

The supplied source represented an internal accounting platform built from several services:

| Component | Technology | Security-relevant role |
|---|---|---|
| `nginx` | Reverse proxy | Public ingress |
| `gateway` | Go | API routing and authentication boundary |
| `auth` | Flask / Python | Registration, login, JWT issuance, health metadata |
| `ledger` | Express / Node.js | Ledger CRUD and outbound webhook behavior |
| `audit` | Axum / Rust | Protected internal audit endpoint / flag storage |
| `redis` | Redis | Application data |

The target data lived behind the internal audit service and was guarded by an internal token plus an HMAC-based reason header. The intended solve required understanding how these controls interacted rather than attacking the audit service directly.

## Attack-surface review

I approached the source as a trust-boundary problem:

1. **What information can the public side disclose?**
2. **Which component verifies authentication, and does it actually validate signatures?**
3. **Which services can make server-side requests?**
4. **Which headers cause the gateway to trust or route a request differently?**
5. **Are secrets independent, random, and scoped to a single purpose?**

That review exposed four weaknesses that could be chained.

## Finding 1 — Predictable secret derived from startup time

The authentication service built its secret from process startup time:

```python
STARTUP_TIME = int(time.time())
SECRET_KEY = f"ledger_secret_{STARTUP_TIME}"
```

A public health endpoint returned both server time and uptime. That meant the startup epoch could be reconstructed approximately as:

```text
startup_time ≈ server_time - uptime_seconds
```

The security issue is not the health endpoint by itself. The issue is using a **low-entropy, externally inferable value as key material**.

### Root cause

- deterministic key generation;
- security secret derived from operational metadata;
- health response exposed the inputs needed to reconstruct it.

### Remediation

Generate secrets with a cryptographically secure RNG and provide them through a secret-management mechanism. Operational timing data should never be part of authentication key derivation.

## Finding 2 — JWT claims parsed without signature verification

The Go gateway used an unverified parsing path for JWT claims. In the retained source notes, the relevant logic used `ParseUnverified`.

That creates a dangerous trust gap: the gateway can treat attacker-controlled claims as authenticated application state even when the token signature is invalid.

### Root cause

The code conflated **parsing a token** with **verifying a token**.

### Remediation

- verify the signature using an explicit expected algorithm;
- reject unsigned/invalid tokens;
- validate issuer, audience, expiry, and other required claims;
- centralize authentication so downstream services do not invent their own trust semantics.

## Finding 3 — Webhook SSRF plus CRLF/header injection

The ledger service accepted a `notify_url`, restricted the hostname to an allowlist, then manually assembled an HTTP request using a decoded path and a raw TCP socket.

The dangerous pattern was conceptually:

```javascript
const parsed = new URL(notifyUrl);
const path = decodeURIComponent(parsed.pathname + parsed.search);

const req = [
  `GET ${path} HTTP/1.1`,
  `Host: ${hostname}`,
  `X-Webhook-Entry: ${entryId}`,
  `Connection: close`,
  '', ''
].join('\r\n');

socket.write(req);
```

Because the decoded path was inserted directly into the HTTP request line, encoded CR/LF characters could become real header separators. The hostname allowlist therefore did not prevent manipulation of the request sent to an allowed internal host.

### Security impact

This turned a constrained server-side request feature into a primitive capable of influencing internal request headers and routing behavior.

### Root cause

- server-side request capability to internal hosts;
- decoded attacker input inserted into a raw protocol message;
- no rejection of CR/LF after decoding;
- reliance on host allowlisting as the primary SSRF control.

### Remediation

- use a mature HTTP client rather than hand-building raw requests;
- reject control characters after canonicalization/decoding;
- enforce destination policy at the resolved-address/network layer;
- do not allow user-controlled values to become internal routing/authentication headers;
- strip and overwrite internal-only headers at trust boundaries.

## Finding 4 — Secret reuse across independent security controls

The audit HMAC key was written from the same application secret used by the auth service. Once that secret became derivable, a second supposedly independent control failed as well.

This is a classic **blast-radius problem**: one key compromise should not collapse multiple trust mechanisms.

### Remediation

Use independent, random secrets with distinct rotation and access policies for:

- JWT signing/verification;
- audit HMAC validation;
- internal service authentication.

## Exploit chain

The retained solve path can be summarized without publishing the event flag or live payload:

```text
1. Query public health metadata
2. Reconstruct startup time and derive the weak application secret
3. Obtain/construct an application JWT path
4. Compute the audit HMAC value using the reused secret
5. Submit a ledger webhook pointed at an allowed internal host
6. Encode CR/LF in the path so the raw request gains internal-only headers
7. Reach the gateway's audit route from a trusted internal source
8. Let the gateway add its own internal token
9. Satisfy the audit service's remaining HMAC check
10. Confirm protected data access in the CTF environment
```

A redacted proof-of-concept shape is retained here only to document the automation logic:

```python
# Pseudocode / redacted portfolio form
health = GET('/api/auth/health').json()
startup = floor(health['server_time'] - health['uptime_seconds'])
secret = derive_from(startup)

ts = current_timestamp()
audit_reason = HMAC_SHA256(secret, ts)

notify_url = build_internal_webhook_url(
    path='/api/audit/internal/flag',
    injected_headers={
        'X-Forwarded-Service': 'audit',
        'X-Audit-Reason': f'{ts}:<redacted-hmac>'
    }
)

POST('/api/ledger/entries', notify_url=notify_url)
```

## What made this challenge valuable

The most useful part was not the CRLF payload. It was recognizing how **four design mistakes in four different trust decisions** combined into one path:

```text
weak secret generation
+ authentication verification failure
+ unsafe server-side request construction
+ secret reuse / internal-header trust
= compromise of a protected internal service
```

That is much closer to real application-security work than treating vulnerabilities as isolated checklist items.

## Skills demonstrated

- polyglot source review;
- API trust-boundary mapping;
- JWT security reasoning;
- SSRF analysis;
- HTTP request-splitting / CRLF reasoning;
- HMAC/key-management analysis;
- exploit-chain construction;
- root-cause remediation design.
