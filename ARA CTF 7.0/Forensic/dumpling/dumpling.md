# Writeup: Dumpling

**Event:** ARA7 Qualification  
**Category:** Digital Forensics / Memory Forensics  
**Difficulty:** Medium  
**Solved by:** PolarBear7 (Glenn Josia Devano)   
**Flag:** `ARA7{f4b2619c33cea30e3159e7397fb1d6a7}`

---

## Objective

Mendapatkan NTLM Hash dari user **"Hannn"** dari sebuah file memory dump.

---

## 1. Reconnaissance & Extraction

### Tantangan

Peserta diberikan file arsip bernama `mem.zip`. Saat mencoba ekstraksi menggunakan `unzip` standar, muncul error:

```bash
unzip mem.zip
# Output:
# skipping: mem.DMP  unsupported compression method 99
```

### Analisis

Error `compression method 99` mengindikasikan bahwa file ZIP dienkripsi menggunakan **AES (Advanced Encryption Standard)**, yang tidak didukung oleh `unzip` versi lama.

### Solusi

Ekstraksi dilakukan menggunakan tools yang mendukung AES-Zip (seperti The Unarchiver, Keka, atau 7-Zip) dengan password: `rawrrr`.

**Hasil:** Didapatkan file `mem.DMP`.

---

## 2. Analisis Awal dengan Volatility (Jalan Buntu)

Awalnya diasumsikan `mem.DMP` adalah Full Memory Dump. Volatility 3 digunakan untuk analisis:

```bash
vol -f mem.DMP windows.info
vol -f mem.DMP windows.hashdump
```

```
Unsatisfied requirement plugins.Info.kernel.layer_name
Unsatisfied requirement plugins.Info.kernel.symbol_table_name
...
No suitable kernels found during pdbscan
```

### Diagnosa Kegagalan

Volatility gagal karena `mem.DMP` **bukan** Full Memory Dump:

- **Full Memory Dump** — Berisi kernel OS, driver, dan semua proses. Volatility membutuhkan Kernel untuk memetakan memori.
- **Minidump** — Berisi memori dari satu proses spesifik saja saat terjadi crash atau diambil secara manual.

Pengecekan menggunakan `file` mengonfirmasi hal ini:

```bash
file mem.DMP
# Output: mem.DMP: Mini DuMP crash report, 17 streams...
```

---

## 3. Pivot Strategi: LSASS Minidump

Karena target adalah **Hash Password** dan filenya adalah Minidump, asumsi terkuat adalah ini merupakan dump dari proses `lsass.exe` (Local Security Authority Subsystem Service) — yang bertanggung jawab menyimpan kredensial pengguna aktif (NTLM Hash, Kerberos ticket, dll).

**Tool yang digunakan:** `Pypykatz` (implementasi Mimikatz dalam Python).

---

## 4. Eksekusi & Eksploitasi

### Langkah 1 — Instalasi Pypykatz

```bash
python3 -m venv ctf_tools
source ctf_tools/bin/activate
pip install pypykatz
```

### Langkah 2 — Parsing Minidump

```bash
pypykatz lsa minidump mem.DMP
```

### Langkah 3 — Analisis Output

Filter output terhadap username target `"Hannn"`. Ditemukan bagian MSV (Authentication Package):

```
== LogonSession ==
authentication_id 44225954 (2a2d5a2)
username          Hannn
domainname        VulnWin
...
 == MSV ==
 Username: Hannn
 Domain:   VulnWin
 LM: NA
 NT: f4b2619c33cea30e3159e7397fb1d6a7
```

Nilai `NT` di atas adalah **NTLM Hash** dari password user Hannn.

---

## 5. Flag

```
ARA7{f4b2619c33cea30e3159e7397fb1d6a7}
```

---
