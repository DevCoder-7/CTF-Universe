[laporan_waru_draft.md](https://github.com/user-attachments/files/28056692/laporan_waru_draft.md)
# Laporan Pengujian atau Hasil Pencarian Celah Keamanan oleh Peserta Bug Bounty 2026
## WARU — Brute Force Login Tidak Terlindungi (CAPTCHA Auto-Verified + No Rate Limiting)

---

## No. 1 — Nama Target

**Nama Aplikasi:** Aplikasi LATVO (Manajemen Pelatihan dan Diklat)
**Kode Target Bug Bounty:** Waru
**URL:** https://bb-waru.kemendikdasmen.go.id/
**Pengelola:** Kementerian Pendidikan Dasar dan Menengah (Kemendikdasmen)
**Fungsi Bisnis:** Platform manajemen pelatihan/diklat berbasis web yang dibangun menggunakan framework Laravel dengan komponen frontend Livewire. Sistem ini digunakan dalam lingkungan Kemendikdasmen untuk pengelolaan kegiatan pelatihan, termasuk autentikasi peserta dan pengelola diklat.

---

## No. 2 — Nama Kerentanan

**Brute Force Login Tidak Terlindungi akibat CAPTCHA yang Terverifikasi Otomatis di Sisi Server (_captchaVerified_ Pre-set _True_) dan Tidak Adanya _Rate Limiting_ / _Account Lockout_ pada Endpoint Autentikasi Livewire**

(_Missing Brute Force Protection due to Server-Side CAPTCHA Auto-Verification and Absence of Rate Limiting on Livewire Authentication Endpoint_)

---

## No. 3 — Jenis Kerentanan

**OWASP Top 10 2021:** A07:2021 (_Identification and Authentication Failures_)

**CWE:**
- **CWE-307:** _Improper Restriction of Excessive Authentication Attempts_ — sistem tidak membatasi jumlah percobaan login yang gagal dari satu sumber, memungkinkan penyerang melakukan percobaan kredensial secara tidak terbatas.
- **CWE-287:** _Improper Authentication_ — mekanisme verifikasi CAPTCHA tidak berfungsi sebagaimana mestinya karena nilai `captchaVerified` telah di-set `true` oleh server sebelum pengguna melakukan interaksi apapun.

**Tingkat Risiko:** MEDIUM
**CVSS v3.1 Base Score: 6.5 MEDIUM**
**Vector String:** `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N`

---

## No. 4 — Deskripsi Kerentanan

### Ringkasan

Aplikasi LATVO mengimplementasikan CAPTCHA sebagai mekanisme perlindungan pada halaman login. Namun, terdapat dua kelemahan fundamental yang saling menguatkan: **(1)** nilai `captchaVerified` di dalam _Livewire snapshot_ sudah di-set `true` oleh server sejak halaman pertama kali dimuat, sebelum pengguna mengisi atau memvalidasi CAPTCHA apapun; dan **(2)** tidak ada mekanisme _rate limiting_, _account lockout_, atau throttling yang aktif pada endpoint `POST /livewire/update` yang digunakan sebagai jalur autentikasi. Kombinasi keduanya menjadikan CAPTCHA tidak berfungsi sebagai lapisan pertahanan, dan membuka pintu bagi serangan brute force tanpa hambatan teknis apapun.

### Penyebab Utama Teknis

Saat halaman login diakses (`GET /login`), server mengirimkan _Livewire component snapshot_ yang tertanam dalam HTML response sebagai atribut `wire:snapshot`. Snapshot ini berisi state awal komponen login, di mana **nilai `captchaVerified` sudah bernilai `true`**, `showCaptcha` bernilai `false`, dan `recaptchaEnabled` bernilai `false`:

```json
{
  "data": {
    "captchaVerified": true,
    "showCaptcha": false,
    "recaptchaEnabled": false,
    "captchaKey": 1
  }
}
```

Karena `captchaVerified` sudah `true` dari awal, logika server-side yang memeriksa status CAPTCHA menganggap verifikasi telah selesai dilakukan, tanpa pengguna pernah menyelesaikan tantangan CAPTCHA nyata. Akibatnya, endpoint `POST /livewire/update` menerima setiap permintaan login tanpa validasi CAPTCHA yang sesungguhnya.

Selain itu, setelah 5 kali percobaan login dengan kredensial yang salah, **tidak ada satupun respons yang mengindikasikan lockout, throttling, atau pembatasan akses**. Nilai `captchaKey` tetap `1`, `captchaVerified` tetap `true`, dan `showCaptcha` tetap `false` pada setiap respons. Laravel _rate limiting middleware_ (yang seharusnya memicu `Too Many Attempts` / HTTP 429) tidak aktif pada route ini.

### Mekanisme Serangan

Penyerang dapat melakukan brute force credential stuffing secara langsung ke endpoint `POST /livewire/update` tanpa perlu:
- Menyelesaikan atau memvalidasi CAPTCHA
- Merotasi sesi / cookie
- Melewati mekanisme lockout apapun

Cukup dengan satu sesi cookie yang valid (diperoleh dari `GET /login`), penyerang dapat mengirimkan ribuan permintaan login dengan kombinasi email dan password yang berbeda secara berurutan, tanpa hambatan teknis apapun dari sisi server.

### Konteks Aplikasi

LATVO adalah sistem diklat/pelatihan resmi dalam ekosistem Kemendikdasmen. Akun yang berhasil dikompromikan melalui brute force dapat memberikan akses ke data pelatihan, informasi peserta program, dan fitur-fitur administratif platform yang seharusnya hanya dapat diakses oleh pengguna terdaftar yang terverifikasi.

---

## No. 5 — Lokasi / URL

**Endpoint Rentan:**

- `https://bb-waru.kemendikdasmen.go.id/login` **(GET):** Halaman login utama; mengirimkan _Livewire snapshot_ dengan `captchaVerified: true` yang sudah di-set sejak awal dalam HTML response, sebelum ada interaksi pengguna.
- `https://bb-waru.kemendikdasmen.go.id/livewire/update` **(POST):** Endpoint proses autentikasi yang bersifat _vulnerable_. Menerima permintaan login berulang tanpa mekanisme pembatasan percobaan (_rate limiting_) atau penguncian akun (_account lockout_).
- `https://bb-waru.kemendikdasmen.go.id/dashboard` **(GET):** Halaman _dashboard_ setelah login berhasil; redirect target setelah autentikasi sukses (HTTP 302).

**Parameter Rentan:**

- `form.email` (dalam _Livewire update payload_) — nilai email yang diuji
- `form.password` (dalam _Livewire update payload_) — nilai password yang diuji
- `captchaVerified` (dalam _Livewire snapshot_) — nilai ini sudah `true` sejak server mengirim halaman login, tidak perlu dimanipulasi oleh penyerang

---

## No. 6 — IP Address Source (IP Address Peserta)

Pengujian dilakukan dari sesi berikut:
- **152.118.150.184** (sesi live validation, 20 Mei 2026)

---

## No. 7 — Dampak

**1. Serangan Brute Force Tanpa Hambatan Teknis**

Penyerang dapat melakukan percobaan kredensial (_credential stuffing_ / _password spraying_) secara tidak terbatas terhadap seluruh akun yang terdaftar di sistem LATVO. Tidak ada mekanisme apapun — baik CAPTCHA, _rate limiting_, _lockout_, maupun throttling — yang membatasi jumlah percobaan. Satu sesi cookie dari `GET /login` sudah cukup untuk menjalankan ribuan percobaan login secara berurutan.

**2. Kompromi Akun Pengguna Terdaftar**

Apabila serangan brute force berhasil mengidentifikasi kredensial yang valid, penyerang memperoleh akses penuh ke akun pengguna tersebut dalam sistem LATVO, termasuk data pelatihan, riwayat diklat, dan fitur-fitur yang hanya tersedia bagi pengguna terautentikasi.

**3. CAPTCHA Sebagai Kontrol Keamanan yang Tidak Efektif**

Keberadaan CAPTCHA memberikan persepsi keamanan (_false sense of security_) kepada administrator sistem, padahal secara teknis CAPTCHA tidak pernah aktif karena `captchaVerified` selalu sudah `true` dari server. Hal ini mengakibatkan _single point of failure_ pada lapisan autentikasi: satu-satunya pertahanan yang ada ternyata tidak berfungsi.

**4. Potensi _Credential Stuffing_ Skala Besar**

Dalam konteks kebocoran data yang sering terjadi, penyerang dapat menggunakan daftar credential yang bocor dari sumber lain (_credential stuffing_) untuk mencoba login ke sistem LATVO. Tanpa pembatasan percobaan, seluruh daftar tersebut dapat diuji secara otomatis tanpa risiko terdeteksi atau diblokir.

**5. Implikasi Regulatoris**

Kegagalan melindungi akun pengguna dari serangan brute force berpotensi melanggar kewajiban pengelola sistem pemerintah dalam melindungi data pribadi pengguna sebagaimana diatur dalam **UU No. 27 Tahun 2022 tentang Perlindungan Data Pribadi (PDP)**.

---

## No. 8 — Langkah Penetrasi dan Tangkapan Layar (_Screenshots_)

### Langkah 1: Identifikasi — Akses Halaman Login dan Ekstraksi _Livewire Snapshot_

Akses `https://bb-waru.kemendikdasmen.go.id/login` via browser. Halaman login LATVO berhasil dimuat dengan HTTP 200. Kemudian lakukan _View Page Source_ (`Ctrl+U`) dan cari atribut `wire:snapshot` pada elemen Livewire. Di dalam JSON snapshot yang dikirimkan server, ditemukan bahwa **`captchaVerified` sudah bernilai `true`**, `showCaptcha` bernilai `false`, dan `recaptchaEnabled` bernilai `false` — padahal pengguna belum melakukan interaksi apapun dengan CAPTCHA.

[SS-1: Browser menampilkan halaman login LATVO, URL bar menunjukkan https://bb-waru.kemendikdasmen.go.id/login]

[SS-2: View Page Source (Ctrl+U), tampilkan bagian wire:snapshot dengan nilai captchaVerified: true, showCaptcha: false, recaptchaEnabled: false yang sudah di-set sejak server response pertama]

---

### Langkah 2: Eksekusi — 5 Percobaan Login Gagal Berturut-turut

Dengan menggunakan sesi cookie dari `GET /login`, kirimkan 5 permintaan `POST /livewire/update` berurutan menggunakan kredensial yang salah. Setiap permintaan menggunakan format _Livewire update payload_ dengan method `login`. Pantau respons JSON dari setiap percobaan untuk melihat apakah ada perubahan pada nilai `captchaVerified`, `showCaptcha`, `captchaKey`, atau apakah ada respons _lockout_ / HTTP 429.

Perintah yang digunakan:

```bash
# Mengekstrak snapshot dan XSRF token dari halaman login
curl -sk "https://bb-waru.kemendikdasmen.go.id/login" \
  -c /tmp/waru_c.txt -o /tmp/waru_l.html

# Mengirim 5 percobaan login dengan kredensial salah
# (menggunakan Python script untuk otomasi payload Livewire)
python3 waru_bruteforce_test.py
```

**Hasil 5 Percobaan (Output Terminal):**

```
Attempt 1: captchaVerified=True, showCaptcha=False, captchaKey=1, lockout=False, captcha=False
Attempt 2: captchaVerified=True, showCaptcha=False, captchaKey=1, lockout=False, captcha=False
Attempt 3: captchaVerified=True, showCaptcha=False, captchaKey=1, lockout=False, captcha=False
Attempt 4: captchaVerified=True, showCaptcha=False, captchaKey=1, lockout=False, captcha=False
Attempt 5: captchaVerified=True, showCaptcha=False, captchaKey=1, lockout=False, captcha=False
```

[SS-3: Terminal menampilkan output 5 percobaan login — semua menunjukkan captchaVerified=True, captchaKey=1, lockout=False tanpa perubahan, tidak ada HTTP 429 atau pesan lockout]

---

### Langkah 3: Verifikasi — Inspeksi Raw Request dan Response POST /livewire/update

Tampilkan raw HTTP request dan response dari percobaan login ke-5 untuk membuktikan tidak ada mekanisme proteksi apapun yang aktif. Response JSON dari server tidak mengandung pesan error _rate limiting_, tidak ada header `Retry-After`, dan struktur JSON menunjukkan `captchaVerified` masih `true` serta `captchaKey` masih `1`.

[SS-4: Burp Suite / terminal menampilkan raw POST request ke /livewire/update dengan payload Livewire (form.email, form.password, method: login)]

[SS-5: Raw JSON response dari percobaan ke-5 — tampilkan captchaVerified: true, showCaptcha: false, captchaKey: 1, tidak ada indikator lockout atau rate limit di response body maupun header]

---

### Langkah 4: Verifikasi Absensi HTTP 429 / Laravel Throttle

Sebagai konfirmasi tambahan, bandingkan response HTTP status code setelah 5+ percobaan. Sistem yang terlindungi seharusnya mengembalikan **HTTP 429 Too Many Requests** disertai header `Retry-After`. Namun pada LATVO, seluruh percobaan ke-1 hingga ke-5 (dan seterusnya) tetap mengembalikan **HTTP 200** tanpa perubahan apapun pada behavior server.

[SS-6: Perbandingan HTTP status code dari percobaan ke-1 hingga ke-5 (semua HTTP 200), tidak ada HTTP 429 muncul]

---

## No. 9 — Rekomendasi

**1. Perbaiki Logika _captchaVerified_ — Jangan Pre-set True di Server (PRIORITAS UTAMA)**

Nilai `captchaVerified` dalam _Livewire component_ harus diinisialisasi sebagai `false` dan hanya di-set `true` setelah pengguna berhasil menyelesaikan verifikasi CAPTCHA yang valid di sisi server (bukan di sisi client). Logika verifikasi CAPTCHA harus divalidasi server-side, bukan hanya berdasarkan nilai state Livewire yang dapat dikirimkan kembali oleh client.

```php
// Contoh perbaikan: inisialisasi state yang benar
public bool $captchaVerified = false; // Bukan true

// Validasi CAPTCHA di server sebelum memproses login
public function login()
{
    if (!$this->captchaVerified) {
        $this->addError('captcha', 'Verifikasi CAPTCHA diperlukan.');
        return;
    }
    // ... lanjut proses login
}
```

**2. Aktifkan Laravel Rate Limiting pada Route Login**

Terapkan `throttle` middleware atau `RateLimiter` bawaan Laravel pada route autentikasi. Batasi percobaan login maksimal **5 kali per IP per menit**, dengan respons HTTP 429 dan header `Retry-After` yang tepat setelah batas tercapai.

```php
// routes/web.php atau RouteServiceProvider
Route::middleware(['throttle:5,1'])->group(function () {
    Route::post('/livewire/update', ...);
});

// Atau menggunakan RateLimiter di AuthServiceProvider
RateLimiter::for('login', function (Request $request) {
    return Limit::perMinute(5)->by($request->ip());
});
```

**3. Implementasi _Account Lockout_ Sementara**

Setelah N kali percobaan gagal berturut-turut dari email/IP yang sama, kunci akun sementara selama 15–30 menit. Simpan counter percobaan di Redis/cache dengan TTL. Kirimkan notifikasi email ke pemilik akun jika terdeteksi percobaan login berulang yang gagal.

**4. Pisahkan Validasi CAPTCHA dari _State_ Livewire**

Untuk CAPTCHA yang benar-benar fungsional, gunakan implementasi server-side yang tidak bergantung pada nilai _state_ Livewire yang dikirim client:
- **Google reCAPTCHA v3** dengan validasi token di backend via Google API
- **CAPTCHA berbasis gambar server-side** (library: `gregwar/captcha` untuk Laravel) dengan nilai disimpan di session server, bukan dalam payload yang bisa dimanipulasi

**5. _Security Audit_ pada Seluruh Komponen Livewire**

Audit semua komponen Livewire lainnya dalam aplikasi LATVO untuk memastikan tidak ada nilai _state_ keamanan lain (`isAdmin`, `isVerified`, `hasPermission`, dll.) yang juga di-preset atau dapat dimanipulasi melalui mekanisme yang sama.

---

## No. 10 — Referensi

- **OWASP Top 10 2021: A07:2021 (Identification and Authentication Failures):** https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/
- **OWASP Web Security Testing Guide (Testing for Brute Force):** https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/04-Authentication_Testing/03-Testing_for_Weak_Lock_Out_Mechanism
- **CWE-307: Improper Restriction of Excessive Authentication Attempts:** https://cwe.mitre.org/data/definitions/307.html
- **CWE-287: Improper Authentication:** https://cwe.mitre.org/data/definitions/287.html
- **OWASP Authentication Cheat Sheet (Account Lockout):** https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- **Laravel Rate Limiting Documentation:** https://laravel.com/docs/11.x/routing#rate-limiting
- **Livewire Security Considerations:** https://livewire.laravel.com/docs/security
- **UU No. 27 Tahun 2022 tentang Perlindungan Data Pribadi (PDP):** Relevan terkait kewajiban pengamanan akun yang menyimpan data pribadi pengguna dari akses tidak sah.
- **CVSSv3.1 Calculator:** https://www.first.org/cvss/calculator/3.1
