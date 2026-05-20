[recon_recap_2005.md](https://github.com/user-attachments/files/28056696/recon_recap_2005.md)
# Live Deep Recon — Kemendikdasmen Bug Bounty 2026
**Tanggal:** 20 Mei 2026 | **IP:** 152.118.150.184 | **Deadline:** 22 Mei 2026

---

## Health Check Semua Target

| Target | URL | Status |
|--------|-----|--------|
| Pine | bb-pine.kemendikdasmen.go.id | 200 ✅ |
| Sandalwood | bb-sandalwood.kemendikdasmen.go.id | 200 ✅ |
| Ironwood | qc.data.kemendikdasmen.go.id/dasbor/ | 200 ✅ |
| Waru | bb-waru.kemendikdasmen.go.id/login | 200 ✅ |
| Walnut | bb-walnut.kemendikdasmen.go.id | 200 ✅ |

---

## WARU — Brute Force Validation ✅ CONFIRMED

**Verdict: CVSS 6.5 MEDIUM → Replace Walnut 6.1**

### Temuan
Livewire login (`POST /livewire/update`) tidak memiliki proteksi brute force.

### Initial State (setiap page load)
```
captchaVerified : True
showCaptcha     : False
recaptchaEnabled: False
captchaKey      : 1
```

### 5 Failed Login Attempts
| Attempt | captchaVerified | showCaptcha | captchaKey | Lockout | Captcha Muncul |
|---------|----------------|-------------|------------|---------|----------------|
| 1 | True | False | 1 | ❌ | ❌ |
| 2 | True | False | 1 | ❌ | ❌ |
| 3 | True | False | 1 | ❌ | ❌ |
| 4 | True | False | 1 | ❌ | ❌ |
| 5 | True | False | 1 | ❌ | ❌ |

### Root Cause
`captchaVerified=true` di-hardcode dalam Livewire snapshot saat page load pertama. Laravel throttle tidak aktif. Tidak ada lockout setelah N attempt gagal.

### CVSS Vector
`AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N` → **6.5 MEDIUM**

### CWE
CWE-307: Improper Restriction of Excessive Authentication Attempts

### Evidence File
`~/bug_bounty/kemen2026/evidence/waru_godmode.txt` (dari sesi 18 Mei)
`~/bug_bounty/kemen2026/evidence/live_deep/waru_live.txt`

---

## IRONWOOD AES — Key Exposure Validation ✅ KEY LIVE

**Verdict: 6.5 MEDIUM candidate — butuh authenticated session untuk upgrade**

### Temuan
AES key hardcoded dalam public JS bundle. HTTP interceptor Angular mengenkripsi semua request body sebelum dikirim ke backend menggunakan key ini.

### Key Details (Confirmed Live)
```
Bundle  : /dasbor/main-SXZ72ZJJ.js (2.29 MB, publicly accessible)
AES Key : Dd16c36E54F4a4E!@#b46E90a57fd8A
AES IV  : 7B1$7eb73!@#8d35
```

### JS Evidence
```javascript
// HTTP Interceptor (confirmed in bundle)
intercept(e,i){
  if(e.body instanceof FormData) return i.handle(e);
  { let n=this.encrypt(e.body||{}) ... }
}
```

### API Endpoint Found in Bundle
```
v1/verval-sp-service/auth/get-info
```

### Production Config (from bundle)
```
urlLogin: https://sso.data.kemdikbud.go.id/sys/login?appkey=9E788510-A7D6-467F-B491-3330C3A2DE51
```

### Status
- Key live di bundle publik ✅
- HTTP interceptor konfirmasi key dipakai enkripsi semua body ✅
- Backend base URL belum terresolve (SPA catch-all, butuh auth session)
- Untuk upgrade CVSS ke 7.5–9.0: login via SSO → intercept API response → decrypt → cek data ijazah/siswa

---

## PINE — Deep Enumeration 📌 No New Finding

**Verdict: Keep Pine 8.1 HIGH as-is**

### Yang Ditest
| Target | Hasil |
|--------|-------|
| `/f0t0/` directory listing | 403 Forbidden |
| `/f0t0/pasold.png` | 200 ✅ (file existing sudah ada di laporan) |
| `/f0t0/` enumeration (ktp, foto, ijazah, dll) | Semua 403/404 |
| `formwijaya_list.php` | 200 — isi: jadwal tahapan seleksi, BUKAN user PII |
| `ubahpasswd.php` (endpoint baru) | Redirect tanpa auth |
| `history.php`, `schedule.php`, dll | Redirect tanpa real user session |

### Kesimpulan
`formwijaya_list.php` = list jadwal program (Seleksi Administratif, Pengumuman Hasil, Pelaksanaan Ujian). Tidak ada nama/email/NIK user lain. Tidak ada bulk PII baru. Tidak ada distinct finding baru.

---

## SANDALWOOD — Deep Tab Enumeration 📌 No New Finding

**Verdict: Keep Sandalwood 9.1 CRITICAL as-is**

### Yang Ditest
Captcha bypass berhasil → akses `home.php`. Tab `?show=` dites: student, kelas, lapakhir, report, daily, final, nilai, score, ujian, hasil, penilaian, attendance, export, download, dll.

| Tab | Hasil |
|-----|-------|
| `?show=daily` | 200 — sama dengan home (bukan grade-specific tab) |
| `?show=final` | 200 — sama dengan home |
| `?show=report` | 200 — data jam/laporan user sendiri (self-data) |
| Email terdeteksi | `mentor_95237@gmail.com` — akun test dari captcha bypass (self-data) |
| Grade columns | Tidak ditemukan |
| NIK/PII user lain | Tidak ditemukan |

### Kesimpulan
`show=daily` dan `show=final` tidak membuka tab grade terpisah — render ulang halaman yang sama. Tidak ada exposure nilai/skor user lain. Tidak ada distinct finding baru.

---

## Portfolio Update

| # | App | CVSS | Status |
|---|-----|------|--------|
| 1 | Ulin | 9.3 CRITICAL | KEEP |
| 2 | Sandalwood | 9.1 CRITICAL | KEEP |
| 3 | Pine | 8.1 HIGH | KEEP |
| 4 | **Waru** | **6.5 MEDIUM** | **REPLACE Walnut 6.1** ← baru |
| 5 | Acacia | 5.3 MEDIUM | KEEP (Ironwood butuh API proof dulu) |

---

## Next Actions

1. **[URGENT] Tulis laporan Waru** — captchaVerified hardcoded true di Livewire snapshot, tidak ada rate limiting/lockout. Submit sebelum deadline 22 Mei.
2. **[OPTIONAL] Ironwood escalation** — login via SSO (`sso.data.kemdikbud.go.id`) → intercept API response pakai key `Dd16c36E54F4a4E!@#b46E90a57fd8A` → jika decrypt hasilkan data ijazah/siswa → upgrade 7.5–9.0 → replace Acacia 5.3.
3. **Pine & Sandalwood** — tidak ada aksi baru.

---

*Recon dilakukan Claude Code | NDA*
