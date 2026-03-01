# Writeup: nyawit

**Event:** NETSOS CTF  
**Category:** Forensics  
**Points:** 387 pts  
**Author:** jay  
**Flag:** `NETSOS{c0ngrAtS_y0U_hAv3_unC0v3r3d_r3g1s7rY_p3rs1sT3nc3_4nD_r3m07e_sh3ll_a1l_1n_0n3}`

---

## Deskripsi

> The client reported that they initially attempted to search for a cracked version of Microsoft Word because the official license was too expensive. They then executed a file they believed to be a legitimate installer. However, the following morning, they discovered that their confidential file could no longer be accessed.

Diberikan sebuah disk image format `.ad1` (AccessData format) dan NC server dengan 18 pertanyaan.

> ⚠️ **WARNING:** This challenge contains live malware. Strictly perform all analysis within an isolated sandbox environment and **DO NOT execute any files on your host machine.**

Karena `.ad1` adalah format proprietary AccessData, file ini tidak bisa di-mount dengan perintah `mount` standar. Diperlukan tools khusus dari repositori [AD1-tools](https://github.com/al3ks1s/AD1-tools.git). Setelah cloning repo dan persiapan selesai, baru bisa dilakukan analisis terhadap file `.ad1` yang diberikan.

---

## PART 1

### Question 1/18 — Computer Name (Hostname)

Windows menyimpan konfigurasi nama komputer di dalam registry hive `SYSTEM`, tepatnya pada path:

```
HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Control\ComputerName\ComputerName
```

Kita bisa menggunakan `hivexget`, yaitu command-line tool yang memungkinkan pembacaan Windows Registry Hive secara langsung dari Linux tanpa perlu menjalankan Windows.

```bash
for hive in SYSTEM SOFTWARE SAM SECURITY; do
    sudo cp "/home/$USER/gunyit_mount/C:\\:NONAME [NTFS]/[root]/Windows/System32/config/$hive" /tmp/$hive
done
sudo chmod 644 /tmp/SYSTEM /tmp/SOFTWARE /tmp/SAM /tmp/SECURITY
hivexget /tmp/SYSTEM '\ControlSet001\Control\ComputerName\ComputerName' 'ComputerName'
```

**Answer Q1:** `SAWIT67`

---

### Question 2/18 — Configured Time Zone

Windows menyimpan nama timezone di registry pada path:

```
HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Control\TimeZoneInformation\TimeZoneKeyName
```

```bash
hivexget /tmp/SYSTEM '\ControlSet001\Control\TimeZoneInformation' 'TimeZoneKeyName'
```

**Answer Q2:** `SE Asia Standard Time`

---

### Question 3/18 — Dynamically Assigned IPv4 Address

Windows menyimpan konfigurasi jaringan (termasuk IP address yang pernah digunakan) di registry `SYSTEM` pada path `TCPIP\Parameters\Interfaces`. Bisa juga ditemukan di log DHCP atau System Event Log.

```bash
sudo grep -aobP "1\x009\x002\x00.\x001\x006\x008\x00.\x005\x006\x00.\x001\x000\x002" /tmp/SYSTEM

# Atau

sudo strings -e l /tmp/SYSTEM | grep -E '192\.168\.'
```

**Answer Q3:** `192.168.56.102`

---

### Question 4/18 — Exact Operating System

Registry hive `SOFTWARE` menyimpan semua konfigurasi aplikasi dan sistem operasi yang terinstall, termasuk versi Windows. Versi OS ada di path:

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
```

```bash
hivexget /tmp/SOFTWARE '\Microsoft\Windows NT\CurrentVersion' 'ProductName'
```

**Answer Q4:** `Windows 10 Home`

---

### Question 5/18 — NTLM Hash of Non-System User

NTLM (NT LAN Manager) Hash adalah hasil hashing MD4 dari password Windows. Windows tidak menyimpan password dalam bentuk plain text, melainkan dalam bentuk hash di registry hive `SAM` (Security Account Manager).

Windows 10/11 versi terbaru menggunakan enkripsi AES tambahan (SAM encryption). Oleh karena itu, diperlukan tools seperti `impacket-secretsdump` yang bekerja dengan cara:
1. Membaca BootKey dari registry `SYSTEM` (System Key / SysKey)
2. Menggunakan BootKey untuk mendekripsi data terenkripsi di `SAM`
3. Mengekstrak hash NTLM yang sesungguhnya

```bash
sudo impacket-secretsdump -sam /tmp/SAM -system /tmp/SYSTEM LOCAL
```

**Answer Q5:** `6d6033b6b48902ee605fe5bba436f8dc`

---

## PART 2

### Question 6/18 — First Visit to jaysenlestari.github.io (System Local Time)

Microsoft Edge (dan semua browser modern berbasis Chromium) menyimpan riwayat browsing dalam format database SQLite. File bernama `History` (tanpa ekstensi) di folder profil browser adalah database SQLite yang berisi tabel-tabel relasional:

- `urls` — daftar URL yang pernah dikunjungi beserta jumlah kunjungan
- `visits` — setiap kunjungan individual dengan timestamp presisi

Browser berbasis Chromium menggunakan **WebKit Epoch** untuk menghitung waktu dalam mikrosekon sejak 1 Januari 1601 (bukan 1970 seperti Unix). Rumus konversinya:

```
Unix Timestamp = (WebKit Timestamp / 1,000,000) - 11,644,473,600
```

Angka `11,644,473,600` adalah selisih detik antara tahun 1601 dan 1970. Setelah dikonversi ke Unix timestamp, ditambahkan argumen `localtime` agar SQLite menyesuaikan ke timezone sistem (UTC+7).

**Answer Q6:** `2026-02-10 20:44:41`

---

### Question 7/18 — Malicious File Name, IP, and Port

Database History Edge/Chrome memiliki tabel `downloads` yang menyimpan informasi lengkap tentang setiap file yang pernah diunduh, termasuk:
- `target_path` — lokasi file disimpan di komputer korban
- `tab_url` — URL halaman tempat download dimulai
- `referrer` — URL sumber/referer

```bash
sqlite3 /tmp/edge_history "SELECT target_path, tab_url FROM downloads;"
```

Dari query ditemukan bahwa korban mengunduh file dari server penyerang (`192.168.56.1` = host di jaringan VirtualBox) yang menjalankan web server sederhana di port `8000`. Nama file mengandung typo `Intsaller` yang disengaja sebagai social engineering.

**Answer Q7:** `MicrosoftWordIntsaller.zip_192.168.56.1_8000`

---

### Question 8/18 — When Did the Victim Finish Extracting the Installer? (System Local Time)

Tool `stat` menampilkan metadata lengkap sebuah file/folder dari filesystem, termasuk tiga jenis timestamp: Access (`atime`), Modify (`mtime`), dan Change (`ctime`).

Ketika Windows mengekstrak file ZIP, ia membuat folder baru dengan nama yang sama. Pada sistem NTFS, timestamp **Modified** pada folder diperbarui tepat saat proses ekstraksi selesai.

```bash
sudo stat "/home/Devan_Linux/gunyit_mount/C:\:NONAME [NTFS]/[root]/Users/n37su/Downloads/MicrosoftWordIntsaller"
```

```
Modify: 2026-02-10 16:34:58 UTC → 23:34:58 WIB
```

**Answer Q8:** `2026-02-10 23:34:58`

---

### Question 9/18 — When Did the Victim Execute the Malicious Script for the First Time? (System Local Time)

File `MicrosoftWordIntsaller.ps1` memiliki **Access Time** yang diperbarui saat interpreter PowerShell pertama kali membuka dan membaca file tersebut untuk dieksekusi. Windows juga merekam eksekusi PowerShell di Security Event Log dengan Event ID `4104` (Script Block Logging) atau Event ID `400` (Engine Lifecycle).

```bash
sudo stat "/home/Devan_Linux/gunyit_mount/C:\:NONAME [NTFS]/[root]/Users/n37su/Downloads/MicrosoftWordIntsaller/MicrosoftWordIntsaller.ps1"

sudo cp "/home/Devan_Linux/gunyit_mount/C:\:NONAME [NTFS]/[root]/Windows/System32/winevt/Logs/Security.evtx" /tmp/
sudo python3 -c "
import Evtx.Evtx as evtx
import Evtx.Views as v
with evtx.Evtx('/tmp/Security.evtx') as log:
    for r in log.records():
        print(r.xml())
" > /tmp/security_plain.txt

grep -B 40 "MicrosoftWordIntsaller.ps1" /tmp/security_plain.txt | grep "TimeCreated"
```

Timeline eksekusi:
- **`:22` — Process Start:** Kernel Windows mulai membuat proses `powershell.exe`. Script `.ps1` belum dibaca.
- **`:26` — File Access:** PowerShell mulai membuka dan membaca isi file script.
- **`:30` — Script Running:** PowerShell selesai kompilasi internal dan mulai mengeksekusi perintah pertama dalam script.

**Answer Q9:** `2026-02-10 23:35:30`

---

## PART 3

### Question 10/18 — SSH Public Key Added to authorized_keys

File `authorized_keys` di direktori `~/.ssh/` adalah mekanisme autentikasi SSH berbasis kunci publik. Jika penyerang menambahkan public key miliknya ke file ini, ia bisa login ke sistem korban via SSH tanpa password kapan saja — dikategorikan sebagai **MITRE ATT&CK T1098.004**.

Script `MicrosoftWordIntsaller.ps1` berisi variable `$AttackerPubKey` yang menyimpan public key penyerang, lalu menuliskannya ke file `authorized_keys`.

```bash
sudo cat "/home/Devan_Linux/gunyit_mount/C:\:NONAME [NTFS]/[root]/Users/n37su/.ssh/authorized_keys"
```

**Answer Q10:** `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIvI7knMQfEaINUOcl/7YemBV3W8/obgj8ebIf6hmpH9 jaysen@LAPTOP-ULIU5NSO`

---

### Question 11/18 — MITRE ATT&CK Technique ID

MITRE ATT&CK adalah framework yang mendokumentasikan taktik dan teknik yang digunakan penyerang nyata dalam serangan siber:

- `T1098` = **Account Manipulation** (kategori induk: penyerang memanipulasi akun untuk mempertahankan akses)
- `.004` = Sub-teknik **SSH Authorized Keys** (spesifik: modifikasi file `authorized_keys`)

**Answer Q11:** `T1098.004`

---

### Question 12/18 — When Did the Attacker First Successfully Authenticate via SSH? (System Local Time)

Windows mencatat semua aktivitas autentikasi di Security Event Log. Saat SSH berhasil login, Windows mencatat event:
- **Event ID 4624:** Successful Account Logon
- **Logon Type 10:** RemoteInteractive (untuk SSH/RDP)

```bash
sudo cp "/home/Devan_Linux/gunyit_mount/C:\:NONAME [NTFS]/[root]/Windows/System32/winevt/Logs/Security.evtx" /tmp/Security_Analisis.evtx

sudo python3 -c "
import Evtx.Evtx as evtx
with evtx.Evtx('/tmp/Security_Analisis.evtx') as log:
    for r in log.records():
        print(r.xml())
" > /tmp/security_q12_full.txt

grep -B 30 "LogonType" /tmp/security_plain.txt | grep -A 1 "10" -B 30 | grep -E "TimeCreated|IpAddress" | grep "16:55:17" -A 1
```

**Answer Q12:** `2026-02-10 23:55:17`

---

### Question 13/18 — First Command Executed by the Attacker

`ConsoleHost_history.txt` adalah file yang secara otomatis dibuat oleh PowerShell untuk merekam riwayat semua perintah yang pernah diketik user — setara dengan `~/.bash_history` di Linux.

```
Path: C:\Users\n37su\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Dalam skenario post-exploitation, `whoami` adalah command paling umum pertama yang dijalankan penyerang untuk memverifikasi identitas user aktif dan level privilege.

```bash
grep -A 500 "16:55:17" /tmp/security_plain.txt | grep "CommandLine" | grep -vE "conhost|sshd.exe\" -z|cmd.exe\"$" | head -n 20
```

**Answer Q13:** `whoami`

---

### Question 14/18 — Tool Used to Transfer the Ransomware Binary

`scp` (Secure Copy Protocol) adalah tool transfer file yang berjalan di atas protokol SSH. Karena penyerang sudah memiliki akses SSH ke sistem korban via `authorized_keys` yang ditanam sebelumnya, menggunakan `scp` adalah pilihan natural — tidak perlu autentikasi tambahan dan komunikasinya terenkripsi.

```bash
grep -A 1 "CommandLine" /tmp/security_plain.txt | grep -B 1 " -R"
```

**Answer Q14:** `scp`

---

## PART 4

### Question 15/18 — Location of the Ransomware Binary

Direktori `Temp` adalah lokasi favorit malware karena user biasa memiliki write permission ke sana tanpa perlu privilege admin. Nama `svch0st.exe` menyerupai `svchost.exe` (proses Windows sah) untuk mengelabui user.

```bash
sudo find /home/$USER/gunyit_mount/ -name '*svch0st*' -ls
```

**Answer Q15:** `C:\Windows\Temp\svch0st.exe`

---

### Question 16/18 — When Did the Ransomware Begin Encrypting Files? (UTC)

File `.enc` (encrypted) adalah bukti langsung aktivitas ransomware. Security Event Log mencatat proses creation (**Event ID 4688**) saat `svch0st.exe` dieksekusi.

```bash
grep -B 20 "svch0st.exe" /tmp/security_plain.txt | grep "TimeCreated"
```

> **Note:** Waktu di Event Log menggunakan UTC, sehingga tidak perlu ditambahkan 7 jam.

**Answer Q16:** `2026-02-10 16:56:29`

---

### Question 17/18 — AES Encryption Key and IV Hardcoded in Ransomware

`svch0st.exe` dikemas menggunakan **PyInstaller** yang meng-embed bytecode Python (`.pyc`) yang dikompresi di dalam binary. Key dan IV tidak tersimpan sebagai string ASCII biasa, melainkan sebagai Python byte literal (`b'...'`) di dalam bytecode Python format `marshal`.

Proses recovery:

```bash
# 1. Salin binary malware ke folder kerja
sudo cp "/path/Windows/Temp/svch0st.exe" ~/ctf/nyawit/
sudo chmod 644 ~/ctf/nyawit/svch0st.exe

# 2. Setup virtual environment
cd ~/ctf/nyawit/
python3 -m venv venv && source venv/bin/activate

# 3. Download pyinstxtractor
wget https://raw.githubusercontent.com/extremecoders-re/pyinstxtractor/master/pyinstxtractor.py

# 4. Ekstrak isi binary PyInstaller
python3 pyinstxtractor.py svch0st.exe
# → Menghasilkan folder: svch0st.exe_extracted/

# 5. Baca semua konstanta dari bytecode Python
python3 -c "
import marshal
f = open('svch0st.exe_extracted/svch0st.pyc', 'rb')
f.read(16)  # Skip 16-byte header (magic bytes)
code = marshal.load(f)
for c in code.co_consts:
    if isinstance(c, bytes) and len(c) in [16, 32]:
        print(repr(c))
"
```

Output:
```
b'J01nNe7sO5kar3NA'   ← KEY (16 bytes = AES-128)
b'fAnSd3n9ant1psEn'   ← IV  (16 bytes = AES-CBC IV)
```

**Answer Q17:** `J01nNe7sO5kar3NA_fAnSd3n9ant1psEn`

---

### Question 18/18 — Content of the Decrypted File

Dari analisis bytecode di Q17, malware menggunakan `Crypto.Cipher.AES` dengan mode **CBC** (Cipher Block Chaining). Proses dekripsi AES-CBC:
1. Baca file `.enc` sebagai bytes (ciphertext)
2. Inisialisasi cipher AES dengan Key dan IV dari Q17
3. Dekripsi → hasilkan plaintext dengan padding
4. Hapus PKCS7 padding dengan `unpad` → plaintext bersih

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b'J01nNe7sO5kar3NA'   # Key dari Q17
iv  = b'fAnSd3n9ant1psEn'   # IV dari Q17

with open('wILzSz9PfH.enc', 'rb') as f:
    ciphertext = f.read()

cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = unpad(cipher.decrypt(ciphertext), 16)
print(plaintext.decode('utf-8'))
```

Output:
```
Bank Pin : 197876
```

**Answer Q18:** `Bank Pin : 197876`

---

## Flag

```
NETSOS{c0ngrAtS_y0U_hAv3_unC0v3r3d_r3g1s7rY_p3rs1sT3nc3_4nD_r3m07e_sh3ll_a1l_1n_0n3}
```
