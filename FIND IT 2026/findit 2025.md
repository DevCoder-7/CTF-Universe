# 📋 REKAP KOMPREHENSIF: FINDIT UGM 2025 CTF

> **Disclaimer:** Riset ini dikompilasi dari writeup publik yang dibagikan oleh peserta dan hanya bertujuan untuk edukasi. Semua kredit untuk solusi diberikan kepada penulis writeup asli masing-masing (Tim panitia Find IT! UGM, tim robots.txt, dan kontributor lainnya).

---

## 🏆 Overview Kompetisi

**Nama Acara:** Find IT! 2025 — Future Innovation and Discovery Information and Technology

**Penyelenggara:** Departemen Teknik Elektro dan Teknologi Informasi (DTETI), Fakultas Teknik, Universitas Gadjah Mada (UGM)

**Format Kompetisi:**
- **Babak Warm-Up:** Online (latihan pra-kompetisi)
- **Babak Penyisihan:** Online (daring)
- **Babak Final:** CTF FINDIT dilaksanakan secara online penuh (_full online_)

**Platform:** CTFd (platform CTF berbasis jeopardy)

**Flag Format:** `FindITCTF{...}`

**Kategori Soal:** Web Exploitation, Forensic, Cryptography, Reverse Engineering, PWN/Binary Exploitation, OSINT, MISC

**Repositori Resmi Writeup (Panitia):**
- Warm-Up WU: https://github.com/Find-IT-UGM/CTF-2025-Warm-Up-WU
- Penyisihan & Final WU: https://github.com/Find-IT-UGM/CTF-2025-Writeup

**Peraturan Penting:**
- Brute force untuk submit flag: **tidak diperbolehkan**
- Penggunaan automated tool (ffuf, katana, dll.) untuk soal Web: **tidak diperbolehkan**
- Peserta **wajib mengumpulkan writeup** di akhir kompetisi agar mendapat nilai penuh
- Kerja sama antar tim: **disqualifikasi**

---

## 🌡️ BABAK WARM-UP — Soal Latihan

Babak Warm-Up adalah sesi pra-kompetisi yang dirancang untuk memperkenalkan peserta pada gaya soal FINDIT. Berisi 3 soal dari 3 kategori berbeda.

| No | Nama Soal | Kategori | Pembuat |
|----|-----------|----------|---------|
| 1 | Ranzone | Forensic | Ismail (Cyberkarta) |
| 2 | quandale_dingle_sequel | Cryptography | Panitia |
| 3 | web_caffeine | Web Exploitation | Panitia |

---

## 🔴 PRIORITAS UTAMA: WEB EXPLOITATION

> ⚡ Bagian ini adalah fokus utama riset — penjelasan paling mendalam

### Soal Web — Babak Warm-Up

| No | Nama Soal | Babak | Vulnerability Type | Kesulitan |
|----|-----------|-------|-------------------|-----------|
| 1 | web_caffeine | Warm-Up | (Lihat writeup resmi) | Easy |

### Soal Web — Babak Penyisihan & Final

> **Catatan:** Data soal spesifik untuk penyisihan dan final tersimpan di repositori resmi https://github.com/Find-IT-UGM/CTF-2025-Writeup (folder `/Web`). Writeup dari tim peserta menyebut kategori Web Exploitation ada dalam kompetisi ini. Flag yang berhasil didokumentasikan oleh tim robots.txt: `FindITCTF{Hmmmm_1_R89lly_d5nt_know_Th8_P5ssword}` (kemungkinan dari soal web dengan tema autentikasi/bypass).

### 🔍 Writeup & Penyelesaian — Soal Web

#### 🌐 web_caffeine (Warm-Up)

**Sumber Writeup Resmi:** https://github.com/Find-IT-UGM/CTF-2025-Warm-Up-WU/tree/main/Web%20Eploitation/web_caffeine

**Deskripsi Soal:**
> Detail deskripsi tersimpan di repositori resmi panitia. Berdasarkan nama soal "web_caffeine", soal ini kemungkinan besar bertemakan aplikasi web bertema kafe/minuman dengan vulnerability tersembunyi di dalamnya.

**Kategori Vulnerability (rekonstruksi berdasarkan konteks FINDIT):**
Untuk soal Warm-Up web di FINDIT, vulnerability yang paling umum digunakan antara lain:
- SQL Injection (bypass login)
- JWT manipulation
- Cookie tampering
- IDOR (Insecure Direct Object Reference)

**Konsep Kunci yang Perlu Dipelajari:**
- Inspeksi source code HTML dan JavaScript di browser
- Penggunaan Developer Tools (F12) untuk menganalisis request/response
- Pemahaman dasar HTTP request (GET, POST, Cookie, Header)
- Tools: Burp Suite, curl

**Referensi Belajar Lanjutan:**
- PortSwigger Web Security Academy: https://portswigger.net/web-security
- HackTricks Web Exploitation: https://book.hacktricks.xyz/pentesting-web

---

#### 🌐 Soal Web Penyisihan (Rekonstruksi Teknis)

Berdasarkan flag yang terdokumentasi `FindITCTF{Hmmmm_1_R89lly_d5nt_know_Th8_P5ssword}`, soal ini kemungkinan besar melibatkan:

**Kemungkinan Vulnerability:** Authentication Bypass / SQL Injection / Password Bypass

**Langkah Penyelesaian Tipikal:**

```
Step 1: Reconnaissance
- Buka aplikasi web target
- Perhatikan form login, fitur pencarian, atau input field lainnya
- Periksa source code (Ctrl+U atau F12 → Elements)
- Cari komentar HTML yang mencurigakan, file JS, atau endpoint tersembunyi

Step 2: Vulnerability Identification
- Coba input test sederhana: ' OR '1'='1
- Amati error message yang muncul — apakah ada SQL error?
- Cek apakah ada filter atau WAF

Step 3: Exploitation
- Untuk SQL Injection bypass login:
  Username: admin' --
  Password: apapun
- Atau gunakan payload: ' OR 1=1 -- -

Step 4: Flag Retrieval
- Setelah berhasil bypass, cari flag di halaman admin
- Flag: FindITCTF{Hmmmm_1_R89lly_d5nt_know_Th8_P5ssword}
```

**Tools yang Digunakan:**
- Browser Developer Tools (F12)
- Burp Suite (intercept request)
- sqlmap (untuk SQL injection otomatis, jika diizinkan)
- curl

**Payload Contoh:**
```python
import requests

url = "http://target.ctf/login"
payload = {
    "username": "admin' --",
    "password": "anything"
}
r = requests.post(url, data=payload)
print(r.text)
```

---

## 🟣 PRIORITAS UTAMA: FORENSIC

> ⚡ Bagian ini adalah fokus utama riset — penjelasan paling mendalam

### Soal Forensic — Babak Warm-Up

| No | Nama Soal | Babak | Forensic Type | Pembuat |
|----|-----------|-------|--------------|---------|
| 1 | Ranzone | Warm-Up | Log Analysis + AES Decryption | Ismail (Cyberkarta) |

### Soal Forensic — Babak Penyisihan & Final

> Tersimpan di repositori resmi https://github.com/Find-IT-UGM/CTF-2025-Writeup (folder `/Forensic`).

---

### 🔍 Writeup & Penyelesaian — Soal Forensic

#### 🔬 Ranzone (Warm-Up — Forensic)

**Sumber Writeup:** https://medium.com/@rashidfirdaus077/findit-2025-simulating-ransomware-locked-file-decryption-c0889a86484c

**Deskripsi Soal:**
> Soal forensic warm-up yang mensimulasikan dekripsi file yang dikunci ransomware. Peserta diberikan sekumpulan log file Windows dan sebuah file terenkripsi bernama `user.csv.bak.ranzone`. Tujuannya: analisis log untuk menemukan kunci dekripsi dan recover file.

**Jenis Forensic:**
- Tipe: Log Analysis + Incident Response (Ransomware Investigation)
- Artefak yang Dianalisis: File `.evtx` (Windows Event Log), file terenkripsi `.ranzone`

**Konsep Kunci yang Dipelajari:**
- Windows Event Log Analysis (khususnya PowerShell Operational Log)
- Deteksi encoded PowerShell commands (`-enc`, `-w hidden`)
- AES encryption dan cara decryptnya via script PowerShell
- Key derivation: `Rfc2898DeriveBytes` (PBKDF2)
- Taktik ransomware: enkripsi AES dengan password yang di-hardcode

**Langkah Penyelesaian:**

```
Step 1: Initial Triage / File Identification
- Inventarisasi semua file yang diberikan
- Identifikasi: banyak file .evtx + 1 file user.csv.bak.ranzone
- Fokus pada log yang relevan: PowerShell, Crypto, DPAPI

Step 2: Log Analysis Strategy
- Buka Windows Event Viewer
- Prioritaskan: Microsoft-Windows-PowerShell%4Operational.evtx
- Cari pattern mencurigakan: "powershell.exe -w hidden -enc ..."
- Decode Base64 dari encoded PowerShell command

Step 3: Script Discovery
- Di dalam PowerShell log, ditemukan script terenkripsi Base64
- Setelah decode, ditemukan plaintext script:

    $Password = ConvertTo-SecureString -String "RanzoneEverywhereH3H3" -AsPlainText -Force
    $Key = (New-Object System.Security.Cryptography.Rfc2898DeriveBytes($Password, byte[], 1000)).GetBytes(32)
    $IV = (New-Object System.Security.Cryptography.Rfc2898DeriveBytes($Password, byte[], 1000)).GetBytes(16)
    [AES decryption routine]

- Password ditemukan: "RanzoneEverywhereH3H3"

Step 4: Decrypt the File
- Gunakan password tersebut untuk mendekripsi user.csv.bak.ranzone
- Flag terdapat di dalam file hasil dekripsi
```

**Tools yang Digunakan:**
- Windows Event Viewer (analisis .evtx)
- PowerShell (untuk dekripsi)
- CyberChef (decode Base64)

**Script Dekripsi (PowerShell):**
```powershell
# Define ransomware password yang ditemukan dari log
$Password = ConvertTo-SecureString "RanzoneEverywhereH3H3" -AsPlainText -Force

# Derive key dan IV dari password menggunakan PBKDF2
$Key = (New-Object System.Security.Cryptography.Rfc2898DeriveBytes($Password, [byte[]](1..16), 1000)).GetBytes(32)
$IV = (New-Object System.Security.Cryptography.Rfc2898DeriveBytes($Password, [byte[]](17..32), 1000)).GetBytes(16)

# Path file input dan output
$EncryptedFile = "user.csv.bak.ranzone"
$DecryptedFile = "user.csv.bak.txt"

# Buka encrypted input stream
$InputFileStream = [System.IO.File]::OpenRead($EncryptedFile)
$OutputFileStream = [System.IO.File]::Create($DecryptedFile)

# Setup AES decryptor
$AES = New-Object System.Security.Cryptography.AesManaged
$AES.Key = $Key
$AES.IV = $IV

$CryptoStream = New-Object System.Security.Cryptography.CryptoStream(
    $OutputFileStream,
    $AES.CreateDecryptor(),
    [System.Security.Cryptography.CryptoStreamMode]::Write
)

# Baca dan dekripsi
$Buffer = New-Object Byte[] 1048576
while (($Count = $InputFileStream.Read($Buffer, 0, $Buffer.Length)) -gt 0) {
    $CryptoStream.Write($Buffer, 0, $Count)
}

# Cleanup
$CryptoStream.Close()
$InputFileStream.Close()
$OutputFileStream.Close()

Write-Host "Decryption selesai: $DecryptedFile"
```

**Key Takeaways & Pelajaran:**
1. **Selalu periksa PowerShell logs** — penyerang sering menggunakan PowerShell yang di-hide
2. **Hardcoded password = bad crypto** — pada kasus ini, password enkripsi tersimpan di script attacker
3. **Base64-encoded commands** (`-enc` flag) hampir selalu mencurigakan
4. **Ekstensi file tidak bisa dipercaya** — `.ranzone` hanyalah nama, isi sebenarnya adalah AES-encrypted data

**Referensi Belajar Lanjutan:**
- Volatility3 Framework: https://github.com/volatilityfoundation/volatility3
- Windows Event Log Analysis: https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/
- PowerShell Forensics: https://www.sans.org/white-papers/38669/

---

## 🟡 CRYPTO (Cryptography)

### Daftar Soal

| No | Nama Soal | Babak | Tipe Crypto |
|----|-----------|-------|-------------|
| 1 | quandale_dingle_sequel | Warm-Up | (Lihat WU resmi) |
| + | Soal Penyisihan | Penyisihan | Tersimpan di repo resmi |
| + | Soal Final | Final | Tersimpan di repo resmi |

### Writeup & Penyelesaian

#### 🔐 quandale_dingle_sequel (Warm-Up — Cryptography)

**Sumber Writeup Resmi:** https://github.com/Find-IT-UGM/CTF-2025-Warm-Up-WU/tree/main/Cryptography/quandale_dingle_sequel

**Deskripsi:** Berdasarkan nama soal dan konteks Warm-Up FINDIT, soal ini kemungkinan bertemakan kriptografi klasik atau modern dengan twist lucu/meme (referensi "Quandale Dingle" dari meme internet).

**Teknik Crypto Umum di CTF Level Ini:**
- XOR cipher dengan key yang bisa dipulihkan
- Caesar/ROT cipher dengan shift kustom
- Base64 berlapis
- RSA dengan parameter lemah (small e, small n)
- Vigenere cipher

**Tools yang Wajib Dikuasai:**
- CyberChef: https://gchq.github.io/CyberChef/ (Swiss-army knife crypto)
- Python dengan library `pycryptodome`
- RsaCtfTool: https://github.com/RsaCtfTool/RsaCtfTool
- FactorDB: http://factordb.com/

**Contoh Penyelesaian XOR (Umum):**
```python
from itertools import cycle

ciphertext = bytes.fromhex("...paste_hex_here...")
key = b"findit"  # kunci yang ditemukan dari soal

plaintext = bytes([c ^ k for c, k in zip(ciphertext, cycle(key))])
print(plaintext.decode())
```

---

## 🟢 REVERSE ENGINEERING

### Daftar Soal

| No | Nama Soal | Babak | Target Binary |
|----|-----------|-------|--------------|
| + | Soal Penyisihan | Penyisihan | Tersimpan di repo resmi |
| + | Soal Final | Final | Tersimpan di repo resmi |

### Writeup & Penyelesaian

**Pendekatan Umum Reverse Engineering di CTF:**

```
Step 1: Initial Analysis
$ file binary_target
$ strings binary_target | grep -i "flag\|findit\|ctf"
$ objdump -d binary_target | head -100

Step 2: Decompilation
- Gunakan Ghidra (gratis) atau IDA Pro
- Cari fungsi main() dan fungsi check_flag()/validate()
- Perhatikan string literal dan comparison operations

Step 3: Dynamic Analysis
$ ltrace ./binary_target    # trace library calls
$ strace ./binary_target    # trace syscalls
$ gdb ./binary_target       # debugging interaktif

Step 4: Keyin Flag
- Pahami algoritma validasi
- Balik algoritma (reverse the check)
- Submit flag
```

**Tools Reverse Engineering:**
- Ghidra (free): https://ghidra-sre.org/
- IDA Free: https://hex-rays.com/ida-free/
- GDB + GEF/PEDA
- Binary Ninja

---

## 🔵 PWN / BINARY EXPLOITATION

### Daftar Soal

| No | Nama Soal | Babak | Teknik |
|----|-----------|-------|--------|
| + | Soal Penyisihan | Penyisihan | Tersimpan di repo resmi |
| + | Soal Final | Final | Tersimpan di repo resmi |

### Pendekatan Umum PWN di CTF

Berdasarkan repositori resmi yang menggunakan Python untuk sebagian besar exploit:

```python
from pwn import *

# Koneksi ke server
io = remote("HOST", PORT)

# Atau lokal
# io = process("./binary")

# Cek proteksi binary
elf = ELF("./binary")
print(elf.checksec())

# Buffer overflow dasar
offset = 64  # cari dengan cyclic()
payload = b"A" * offset
payload += p64(win_addr)  # alamat fungsi win/flag

io.sendlineafter(b"Input: ", payload)
io.interactive()
```

**Konsep PWN Dasar:**
- Buffer Overflow (Stack)
- Return-Oriented Programming (ROP)
- Format String Vulnerability
- Heap Exploitation (use-after-free, heap overflow)

**Tools PWN:**
- pwntools: `pip install pwntools`
- GDB + PEDA/GEF/pwndbg
- ROPgadget: https://github.com/JonathanSalwan/ROPgadget

---

## ⚪ OSINT

### Daftar Soal & Penyelesaian

| No | Nama Soal | Babak | Teknik |
|----|-----------|-------|--------|
| + | Soal Penyisihan | Penyisihan | Social media, reverse image, geolocation |
| + | Soal Final | Final | Tersimpan di repo resmi |

**Flag yang Terdokumentasi dari Tim robots.txt:**
`FindITCTF{k0n0h4_m4ju_m4sy4r4k4t_m4kmur}` — flag ini kemungkinan dari soal OSINT (bertemakan lokasi/tempat di Indonesia, "Konoha" = Desa Konoha seperti di anime, "maju masyarakat makmur" adalah slogan khas Indonesia)

**Teknik OSINT Standar CTF:**
```
1. Google Dorking: "nama_target" site:instagram.com
2. Reverse Image Search: images.google.com / Yandex Images
3. Metadata gambar: exiftool image.jpg
4. Wayback Machine: web.archive.org
5. Social media hunting: Instagram, Twitter, LinkedIn
6. Geolocation: Google Maps Satellite view
```

**Tools OSINT:**
- ExifTool: `apt install exiftool`
- Sherlock (username hunting): https://github.com/sherlock-project/sherlock
- Maltego (free version)
- SpiderFoot

---

## ⚫ MISCELLANEOUS (MISC)

### Daftar Soal & Penyelesaian

| No | Nama Soal | Babak | Tipe |
|----|-----------|-------|------|
| + | Soal Penyisihan | Penyisihan | Tersimpan di repo resmi |
| + | Soal Final | Final | Tersimpan di repo resmi |

**Kategori Misc Tipikal:**
- Encoding/decoding berlapis (Base64, Hex, Morse, ASCII)
- Steganografi gambar (steghide, binwalk)
- Programming challenge
- Game/puzzle solving

---

## 📊 Analisis Pattern & Tren FINDIT UGM 2025

### 1. Pola Soal Web Exploitation

**Observasi dari data yang tersedia:**
- Soal Warm-Up Web (`web_caffeine`) dirancang untuk pemula, mengenalkan konsep dasar web security
- Flag `FindITCTF{Hmmmm_1_R89lly_d5nt_know_Th8_P5ssword}` mengindikasikan soal bertema authentication bypass atau password enumeration
- Aturan kompetisi melarang ffuf/automated scanner → berarti soal web **tidak butuh brute force direktori**, fokus pada logika aplikasi

**Vulnerability Type yang Kemungkinan Muncul (berdasarkan edisi sebelumnya):**
- SQL Injection (masih jadi favorit untuk tingkat mahasiswa)
- JWT manipulation / cookie tampering
- IDOR (Insecure Direct Object Reference)
- SSTI (Server Side Template Injection) — untuk soal medium-hard
- LFI (Local File Inclusion)

**Framework/Teknologi yang Sering Digunakan:**
- PHP (Apache) — paling umum di CTF lokal Indonesia
- Python Flask/Django — untuk soal modern
- Node.js (Express) — mulai sering muncul

### 2. Pola Soal Forensic

**Observasi dari Ranzone (Warm-Up):**
- Fokus pada **Log Analysis** (khususnya Windows Event Log)
- Melibatkan **ransomware forensics** — tren yang relevan dengan dunia nyata
- Menggunakan **PowerShell** sebagai artefak analisis
- Level: Pemula-Menengah (bisa diselesaikan dengan Event Viewer + basic PowerShell knowledge)

**Jenis Artefak Forensic yang Kemungkinan Muncul di Penyisihan/Final:**
- `.pcap` (network capture) — untuk soal network forensics
- `.vmem` atau memory dump — untuk memory forensics (Volatility)
- `.E01` atau disk image — untuk disk forensics (Autopsy)
- Gambar dengan steganografi — untuk stego

### 3. Perbandingan Babak

| Aspek | Warm-Up | Penyisihan | Final |
|-------|---------|------------|-------|
| Jumlah Soal | 3 (representatif) | Lebih banyak | Lebih sedikit, lebih sulit |
| Difficulty | Easy | Easy–Medium | Medium–Hard |
| Kategori | Web, Crypto, Forensic | Semua 7 kategori | Semua 7 kategori |
| Chain Exploit | Tidak | Mungkin ada | Kemungkinan lebih banyak |

---

## 📚 Roadmap Belajar Berdasarkan FINDIT UGM 2025

### 🎯 Web Exploitation — Study Path

**Fundamental yang Harus Dikuasai:**

1. **HTTP Basics** → MDN Web Docs: https://developer.mozilla.org/en-US/docs/Web/HTTP
2. **SQL Injection** → PortSwigger SQLi Labs: https://portswigger.net/web-security/sql-injection
3. **Authentication Bypass** → PortSwigger Auth Labs: https://portswigger.net/web-security/authentication
4. **IDOR** → PortSwigger IDOR: https://portswigger.net/web-security/access-control/idor
5. **XSS** → PortSwigger XSS: https://portswigger.net/web-security/cross-site-scripting
6. **SSTI** → HackTricks SSTI: https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection

**Tools yang Wajib Dikuasai:**
- **Burp Suite Community** (intercept & modify request) — WAJIB untuk CTF web
- **curl** (mengirim HTTP request dari command line)
- **Browser DevTools** (F12) — inspect element, network tab, console

**Platform Latihan:**
- PortSwigger Web Security Academy: https://portswigger.net/web-security (GRATIS, terbaik untuk web)
- HackTheBox (Web Challenges): https://www.hackthebox.com/
- PicoCTF Web Category: https://picoctf.org/

---

### 🎯 Forensic — Study Path

**Fundamental yang Harus Dikuasai:**

1. **File Format & Magic Bytes** → File Signatures: https://www.garykessler.net/library/file_sigs.html
2. **Network Forensics (PCAP)** → Wireshark Tutorial: https://www.wireshark.org/docs/wsug_html_chunked/
3. **Memory Forensics** → Volatility3 Docs: https://volatility3.readthedocs.io/
4. **Log Analysis** → Windows Event IDs: https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/
5. **Steganography** → CTF Stego Guide: https://ctf-wiki.org/misc/picture/

**Tools yang Wajib Dikuasai:**

| Tool | Kegunaan | Install |
|------|----------|---------|
| Wireshark | Analisis PCAP/network capture | `sudo apt install wireshark` |
| Volatility3 | Memory forensics | `pip install volatility3` |
| Autopsy | Disk forensics GUI | https://www.autopsy.com/download/ |
| Binwalk | File carving & analysis | `sudo apt install binwalk` |
| ExifTool | Metadata extraction | `sudo apt install exiftool` |
| Strings | Cari string di binary | `strings file.bin` |
| Foremost | File carving | `sudo apt install foremost` |

**Platform Latihan:**
- CyberDefenders: https://cyberdefenders.org/ (forensics-focused)
- BlueTeamLabs Online: https://blueteamlabs.online/
- PicoCTF Forensics: https://picoctf.org/

---

### 🎯 Cryptography — Study Path

**Fundamental:**
1. **Encoding vs Encryption** (Base64, Hex, ASCII bukan enkripsi — ini encoding)
2. **Classical Ciphers**: Caesar, Vigenere, XOR, Substitution
3. **Modern Crypto**: RSA, AES, Diffie-Hellman
4. **Hash Functions**: MD5, SHA1, SHA256, cara cracking-nya

**Platform Latihan:**
- CryptoHack: https://cryptohack.org/ (TERBAIK untuk crypto CTF)
- CryptoPals: https://cryptopals.com/

---

### 🎯 Reverse Engineering — Study Path

**Fundamental:**
1. **Assembly Language Basics** (x86/x64)
2. **Binary formats**: ELF (Linux), PE (Windows)
3. **Decompilation** dengan Ghidra
4. **Debugging** dengan GDB

**Platform Latihan:**
- Crackmes.one: https://crackmes.one/
- Reversing.kr: http://reversing.kr/
- PicoCTF Reversing: https://picoctf.org/

---

### 🎯 PWN/Binary Exploitation — Study Path

**Fundamental:**
1. **Memory Layout** (stack, heap, BSS, data, text)
2. **Buffer Overflow** dasar
3. **ROP (Return-Oriented Programming)**
4. **pwntools** library

**Platform Latihan:**
- pwn.college: https://pwn.college/ (TERBAIK, free)
- ROP Emporium: https://ropemporium.com/
- exploit.education: https://exploit.education/

---

## 🔗 Master Resource List

### Writeup Repositories (Resmi & Komunitas)

| Sumber | Link |
|--------|------|
| **Writeup Resmi Panitia (Penyisihan+Final)** | https://github.com/Find-IT-UGM/CTF-2025-Writeup |
| **Writeup Resmi Panitia (Warm-Up)** | https://github.com/Find-IT-UGM/CTF-2025-Warm-Up-WU |
| Writeup Tim robots.txt (Scribd) | https://www.scribd.com/document/895881374/Writeup-CTF-FindIT-2025-robots-txt |
| Writeup "Ranzone" oleh Eidelweiiss (Medium) | https://medium.com/@rashidfirdaus077/findit-2025-simulating-ransomware-locked-file-decryption-c0889a86484c |

### Official Sources

| Sumber | Link |
|--------|------|
| Website Resmi Find IT! | https://find-it.id/ |
| GitHub Organisasi Find IT! UGM | https://github.com/Find-IT-UGM |
| Instagram Find IT! | https://www.instagram.com/ugm.findit/ |

### Platform Belajar CTF Indonesia

| Platform | Link |
|----------|------|
| PortSwigger Web Academy | https://portswigger.net/web-security |
| CryptoHack | https://cryptohack.org/ |
| pwn.college | https://pwn.college/ |
| CyberDefenders | https://cyberdefenders.org/ |
| HackTheBox | https://www.hackthebox.com/ |
| PicoCTF | https://picoctf.org/ |
| HackTricks (referensi teknik) | https://book.hacktricks.xyz/ |

---

## 💡 Tips & Trik Khusus CTF FINDIT UGM

1. **Baca rulebook dengan teliti** — FINDIT punya aturan unik seperti larangan ffuf dan brute force direktori. Pastikan kamu tidak didiskualifikasi karena tools yang salah.

2. **Writeup wajib dikumpulkan** — FINDIT mewajibkan writeup untuk mendapat nilai. Biasakan mendokumentasikan setiap langkah solving dari awal.

3. **Manfaatkan hint dari panitia** — FINDIT memiliki sistem hint di platform. Jika soal terlalu sulit (sedikit solve), panitia akan merilis hint.

4. **Soal Warm-Up adalah kunci** — Jangan remehkan Warm-Up. Ini adalah preview gaya soal dan tingkat kesulitan FINDIT. Pastikan kamu bisa menyelesaikan semua soal Warm-Up sebelum ikut penyisihan.

5. **Flag format konsisten** — Format `FindITCTF{...}` dengan isi yang sering menggunakan angka-huruf leetspeak (contoh: `1` untuk `i`, `3` untuk `e`, `4` untuk `a`).

6. **Komunitas CTF Indonesia** — Bergabunglah dengan komunitas CTF Indonesia seperti:
   - ID-CTF (komunitas CTF Indonesia)
   - Forum/grup Telegram CTF Indonesia
   - GitHub komunitas: https://github.com/53buahapel/kumpulan-CTF-writeup

7. **Persiapan teknis** — Setup VM Kali Linux atau ParrotOS dengan tools standar CTF sebelum kompetisi.

---

*Dokumen ini dikompilasi pada April 2026 dari sumber publik. Untuk data soal penyisihan dan final yang lebih lengkap, kunjungi repositori resmi panitia di https://github.com/Find-IT-UGM/CTF-2025-Writeup*
