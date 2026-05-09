# 🔍 FindIT CTF 2026 — DIGITAL FORENSICS CHEATSHEET

> **Event:** FindIT CTF 2026 Final · 9 Mei 2026 · 10:00–17:00 WIB
> **Format:** Jeopardy · 7 jam · Face cam aktif
> **Aturan:** No external LLM saat lomba. Cheatsheet ini referensi mandiri offline.

---

## 📑 DAFTAR ISI

1. [Metodologi Umum Forensic CTF](#1-metodologi-umum)
2. [Sub-Kategori Forensic Lengkap](#2-sub-kategori-forensic)
   - 2A. Steganography
   - 2B. Disk Image & File System
   - 2C. Memory Forensics
   - 2D. Network Forensics (PCAP)
   - 2E. Forensic Image Format (AD1, E01, RAW)
   - 2F. File Carving & Magic Bytes
3. [Encoding & Decoding Cheatsheet](#3-encoding--decoding)
4. [Metadata & ExifTool](#4-metadata--exiftool)
5. [Script Python Siap Pakai](#5-python-scripts)
6. [Tools Installation](#6-tools-installation)
7. [Skenario Prediksi Final](#7-skenario-prediksi)
8. [⚡ Quick Reference (Cetak ini!)](#8-quick-reference)

---

## 1. METODOLOGI UMUM

> **Aturan emas:** *Jangan pernah percaya ekstensi file.* Selalu cek magic bytes dulu.

### Alur kerja standar (selalu jalankan urutan ini)

```bash
TARGET=challenge.bin   # ganti sesuai nama file

# ── STEP 1: Initial triage (apa file ini sebenarnya?) ──────────────
file $TARGET                    # identifikasi via magic bytes
xxd $TARGET | head -5           # lihat header bytes mentah
xxd $TARGET | tail -5           # cek ada trailer mencurigakan
ls -la $TARGET                  # ukuran file (clue: mismatch dgn ekspektasi?)
md5sum $TARGET                  # hash untuk catatan

# ── STEP 2: Metadata extraction ────────────────────────────────────
exiftool $TARGET                # all metadata
exiftool -v3 $TARGET            # verbose, struktur internal

# ── STEP 3: String hunting ─────────────────────────────────────────
strings -n 6 $TARGET | grep -iE "FindIT|flag|CTF|password|http|secret"
strings -el $TARGET | grep -iE "FindIT|flag"   # UTF-16LE (Windows wide)
strings -eb $TARGET | grep -iE "FindIT|flag"   # UTF-16BE
strings -a $TARGET | wc -l                      # total strings count

# ── STEP 4: Embedded file detection ────────────────────────────────
binwalk $TARGET                 # scan signatures
binwalk -e $TARGET              # auto extract
binwalk --dd='.*' $TARGET       # extract everything (incl. unknown)
foremost -i $TARGET -o /tmp/out # carve known formats

# ── STEP 5: Category-specific (lihat bagian 2) ─────────────────────
# Tergantung file type, masuk ke sub-kategori yang sesuai

# ── STEP 6: Flag extraction ────────────────────────────────────────
grep -aoP "FindITCTF\{[^}]+\}" $TARGET
strings -a $TARGET | grep -oP "FindITCTF\{[^}]+\}"
```

### Contoh output yang harus diperhatikan

| Output `file` | Artinya | Action |
|---|---|---|
| `data` | Tidak dikenali — kemungkinan terenkripsi/encoded | XOR brute, magic bytes scan |
| `Zip archive data` di file `.png` | Polyglot / append | `binwalk -e` |
| `ASCII text` tapi extension binary | Terbalik — coba decode base64/hex | base64 -d |
| `Linux rev 1.0 ext4 filesystem data` | Disk image | `debugfs` / mount |
| `MS Windows minidump` | Memory dump | `volatility` / minidump parser |

---

## 2. SUB-KATEGORI FORENSIC

### 2A — STEGANOGRAPHY 🎯 *(prioritas tinggi — hint panitia)*

#### Image Stego (PNG, JPG, BMP, PPM, GIF)

**Tools:** `steghide`, `zsteg`, `stegsolve`, `exiftool`, `binwalk`, `pngcheck`, `stegseek`, `outguess`

**Command sequence wajib (jalankan SEMUANYA — jangan skip):**

```bash
IMG=flag.png

# 1. Identifikasi & metadata
file $IMG
exiftool $IMG                                   # cek Comment, Description, UserComment
pngcheck -v $IMG                                # PNG only — cek chunk anomali

# 2. Strings & embedded
strings $IMG | grep -iE "FindIT|flag|CTF"
binwalk $IMG                                    # scan
binwalk -e $IMG                                 # extract appended data
foremost $IMG -o /tmp/foremost_out

# 3. zsteg (PNG/BMP — paling powerful untuk LSB)
zsteg $IMG                                      # default scan
zsteg -a $IMG                                   # ALL channels & bit orders
zsteg -E b1,r,lsb,xy $IMG > extract.bin         # extract LSB red
zsteg -E b1,bgr,lsb,xy $IMG > extract.bin       # LSB BGR (common)
zsteg --all $IMG | grep -E "text|FindIT|flag"

# 4. steghide (JPEG paling sering, juga BMP/WAV)
steghide info $IMG                              # cek embedded
steghide extract -sf $IMG -p ""                 # tanpa password
steghide extract -sf $IMG -p "password"         # dgn password
stegseek $IMG /usr/share/wordlists/rockyou.txt  # brute force pwd

# 5. outguess (jarang tapi bisa)
outguess -r $IMG output.txt

# 6. StegSolve (GUI — manual analysis)
java -jar stegsolve.jar
# → Analyze → Data Extract: cycle bit planes, RGB
# → File Format: cek struktur PNG chunks
# → Image Combiner: XOR antara dua gambar

# 7. Manual LSB extract (Python — kalau zsteg fail)
python3 << 'EOF'
from PIL import Image
img = Image.open('flag.png')
pixels = img.load()
bits = ""
for y in range(img.height):
    for x in range(img.width):
        r, g, b = pixels[x, y][:3]
        bits += str(r & 1)            # ganti ke g & 1, b & 1 sesuai channel
chars = [chr(int(bits[i:i+8], 2)) for i in range(0, len(bits)-7, 8)]
text = ''.join(chars)
import re
for m in re.finditer(r'FindIT[A-Z]*\{[^}]+\}', text):
    print(m.group(0))
EOF
```

**Teknik lanjutan & gotcha:**
- **Histogram analysis** — kalau LSB stego, histogram pixel value nampak "even-odd flat"
- **Alpha channel** — banyak yang lupa cek RGBA: `zsteg -E a1,a,lsb,xy` 
- **Append data** — file ZIP/text setelah IEND chunk PNG (binwalk pasti detect)
- **PPM/PGM raw** — sering carving target karena tidak ada kompresi
- **JPEG quality drop** — kalau di-resave, LSB hilang. **JANGAN edit gambar!**

#### Audio Stego (WAV, MP3, FLAC)

**Tools:** Audacity, Sonic Visualizer, `sox`, `mp3stego`, `stegano`

```bash
AUD=audio.wav

# 1. Standar
file $AUD
exiftool $AUD                                   # cek metadata
strings $AUD | grep -iE "FindIT|flag"

# 2. Spectrogram (PALING SERING di CTF!)
sox $AUD -n spectrogram -x 1500 -y 500 -z 120 -o spec.png
# Buka spec.png → cari teks tersembunyi di frekuensi tertentu

# 3. Kalau spectrogram standar tidak menampilkan, coba parameter berbeda
sox $AUD -n spectrogram -m -x 3000 -y 1000 -o spec_hi.png   # high-res monochrome
sox $AUD -n spectrogram -X 500 -o spec_zoom.png             # zoom in time

# 4. Audacity: Effect → Plot Spectrum / Spectrogram view
#    Zoom ke frekuensi 0-8kHz, banyak teks tersembunyi di range ini

# 5. LSB audio (manual Python)
python3 << 'EOF'
import wave
with wave.open('audio.wav', 'rb') as w:
    frames = w.readframes(w.getnframes())
bits = ""
for byte in frames:
    bits += str(byte & 1)
chars = ''.join(chr(int(bits[i:i+8], 2)) for i in range(0, len(bits)-7, 8))
import re
for m in re.finditer(r'FindIT[A-Z]*\{[^}]+\}', chars):
    print(m.group(0))
EOF

# 6. mp3stego (MP3 spesifik)
mp3stego decode -X audio.mp3                    # menghasilkan audio.mp3.txt

# 7. DTMF / morse di audio?
# Audacity: zoom waveform, dengar manual; atau pakai DTMF decoder online
```

**🎯 Catatan dari "I wanna go home" (FindIT 2026 Quals):**
> Soal AD1 mengandung WAV → spectrogram menampilkan teks → ada hidden RIFF file ber-ekstensi `.dat`. **Selalu carving file di dalam file** dengan binwalk + cek extension misnomer.

#### Stego di format lain

```bash
# PDF stego
pdfinfo file.pdf
pdf-parser.py file.pdf
qpdf --qdf --object-streams=disable file.pdf out.pdf  # uncompress streams
python3 -c "
from stegano import lsb
print(lsb.reveal('image.png'))
"

# ZIP stego (data di komentar / file extra)
unzip -l file.zip                               # list isi
zipdetails file.zip                             # struktur detail
unzip -z file.zip                               # tampilkan archive comment

# Office docs (docx/xlsx = ZIP)
unzip -d unpacked/ document.docx
ls unpacked/word/media/                         # gambar embedded
cat unpacked/docProps/core.xml                  # metadata
```

---

### 2B — DISK IMAGE & FILE SYSTEM

**Tools utama:** `debugfs`, `mmls`, `fls`, `icat`, `foremost`, `photorec`, `testdisk`, `mount`

#### Mount-based approach

```bash
IMG=disk.img

# 1. Identifikasi struktur partisi
file $IMG
fdisk -l $IMG
mmls $IMG                                       # sleuthkit partition table

# 2. Mount langsung (partisi tunggal / ext4)
sudo mkdir -p /mnt/img
sudo mount -o loop,ro $IMG /mnt/img
ls -la /mnt/img

# 3. Mount partisi spesifik (offset dari mmls)
# Misal start sector 2048, sector size 512 → offset = 2048*512 = 1048576
sudo mount -o loop,ro,offset=1048576 $IMG /mnt/img

# 4. Browse forensic
ls -la /mnt/img/home/user/
find /mnt/img -name "*.txt" -exec cat {} \;
find /mnt/img -name ".*" -type f                # hidden files
cat /mnt/img/home/user/.bash_history
cat /mnt/img/etc/shadow                         # password hashes
cat /mnt/img/etc/passwd

# 5. Unmount setelah selesai
sudo umount /mnt/img
```

#### debugfs approach (ext2/3/4 — TANPA SUDO!)

```bash
# debugfs sangat berguna kalau tidak ada akses root
debugfs disk.img -R "ls -l /"
debugfs disk.img -R "ls -la /home"
debugfs disk.img -R "ls -la /root"
debugfs disk.img -R "ls -la /home/user"

# Dump file dari image
debugfs disk.img -R "dump /home/user/secret.txt /tmp/secret.txt"
debugfs disk.img -R "dump /etc/shadow /tmp/shadow"
debugfs disk.img -R "dump /home/user/.bash_history /tmp/bash_hist"

# 🎯 LIHAT FILE YANG DIHAPUS!
debugfs disk.img -R "lsdel"                     # list deleted inodes
# Outputnya: Inode  Owner  Mode  Size  Blocks  Time deleted  Name
debugfs disk.img -R "dump <12345> /tmp/recovered.bin"   # dump by inode

# Cat (read tanpa dump)
debugfs disk.img -R "cat /home/user/notes.txt"

# Lihat semua history command interactif
debugfs disk.img
debugfs:  ls /home/user
debugfs:  stat <12>                             # info inode
debugfs:  quit
```

#### Sleuthkit approach

```bash
# fls — list files (termasuk yang dihapus)
fls -r -d disk.img                              # deleted only
fls -r disk.img                                 # all recursive
fls -r -p disk.img | grep -i flag

# icat — read by inode (bekerja untuk deleted file!)
icat disk.img 12345 > recovered.txt

# tsk_recover — recover semua deleted
tsk_recover disk.img /tmp/recovered/
tsk_recover -e disk.img /tmp/all/               # extract semua, bukan hanya deleted
```

#### Raw carving (kalau filesystem corrupt)

```bash
# Cari magic bytes di raw image
xxd disk.img | grep "504b 0304" | head          # ZIP
xxd disk.img | grep "8950 4e47"                 # PNG
xxd disk.img | grep "ffd8 ffe0\|ffd8 ffe1"      # JPEG

# Setelah dapat offset (misal di line dgn alamat 0x980800):
OFFSET=$((0x980800))
dd if=disk.img of=/tmp/carved.zip bs=1 skip=$OFFSET count=1048576
# Trim ekor sampah:
file /tmp/carved.zip                            # cek apakah valid

# Foremost (otomatis)
foremost -i disk.img -o /tmp/foremost/
ls /tmp/foremost/                               # /jpg /pdf /zip /txt dst

# Photorec (interaktif, lebih powerful)
photorec disk.img
```

#### Git repository di dalam image (sering jadi clue!)

```bash
REPO=/tmp/extracted_repo

# Setelah dump folder .git ke /tmp/extracted_repo/.git
git -C $REPO log --all --oneline                # semua commit
git -C $REPO log --all --pretty=fuller          # info komit detail
git -C $REPO show <commit-hash>                 # diff komit
git -C $REPO diff HEAD~1 HEAD                   # diff vs sebelumnya
git -C $REPO stash list                         # stash tersimpan
git -C $REPO reflog                             # history HEAD movements
git -C $REPO checkout <commit-hash> -- file.txt # restore file dari commit

# 🎯 Cari file yang di-delete dalam commit
git -C $REPO log --diff-filter=D --summary      # commits yang menghapus file
git -C $REPO log --all --full-history -- "deleted_file.txt"
```

**🎯 Catatan dari "debris" (FindIT 2026 Quals):**
> ext4 image → `debugfs` → ada ZIP terhapus → **raw bytes carving** dengan grep magic bytes → ZIP berisi gambar steghide → flag dibungkus base64 multi-layer. **Pola: disk → carving → stego → encoding chain.**

---

### 2C — MEMORY FORENSICS

#### Strings-first approach (jangan langsung loncat ke Volatility!)

```bash
DUMP=memory.dmp

# Quick wins — sering dapat flag tanpa Volatility
strings -n 8 $DUMP | grep -oP "FindITCTF\{[^}]+\}"
strings -el $DUMP | grep -oP "FindITCTF\{[^}]+\}"     # UTF-16LE
strings -eb $DUMP | grep -oP "FindITCTF\{[^}]+\}"     # UTF-16BE

# Cari konteks sensitif
strings $DUMP | grep -iE "password|token|secret|api_key" | head
strings -el $DUMP | grep -iE "password|api" | head     # wide string version

# Cari URL / IP exfiltration
strings $DUMP | grep -oE "https?://[^ '\"]+" | sort -u
strings $DUMP | grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" | sort -u
```

#### Volatility 3 (Windows / Linux modern)

```bash
VOL="python3 vol.py -f memory.dmp"

# 1. Identifikasi OS
$VOL banners.Banners                            # cari kernel banner Linux
$VOL windows.info                               # info Windows version

# 2. Process listing
$VOL windows.pslist
$VOL windows.pstree                             # parent-child tree
$VOL windows.cmdline                            # command-line per proses
$VOL windows.psscan                             # scan termasuk hidden/exited

# 3. File hunting
$VOL windows.filescan | grep -iE "flag|secret|note|desktop"
$VOL windows.filescan > all_files.txt
# Setelah dapat virtual address (kolom Offset):
$VOL windows.dumpfiles --virtaddr 0xfa800123abcd
$VOL windows.dumpfiles --pid 1234

# 4. Network
$VOL windows.netscan                            # active connections + listening
$VOL windows.netstat

# 5. Malware hunting
$VOL windows.malfind                            # hidden/injected code
$VOL windows.ldrmodules                         # DLL anomalies
$VOL windows.svcscan                            # services

# 6. Registry (Windows)
$VOL windows.registry.hivelist
$VOL windows.registry.printkey \
    --key "Software\Microsoft\Windows\CurrentVersion\Run"
$VOL windows.registry.userassist                # programs run by user

# 7. Linux specific
$VOL linux.pslist
$VOL linux.bash                                 # shell history!
$VOL linux.psaux                                # process detail
$VOL linux.lsof                                 # open files

# 8. Dump process memory
$VOL windows.memmap --pid 1234 --dump
$VOL windows.dumpfiles --pid 1234
```

#### Volatility 2 (untuk Win XP/7 lama)

```bash
VOL2="python2 vol.py -f memory.dmp"

# Identifikasi profile dulu (WAJIB)
$VOL2 imageinfo
# Output suggest profiles, contoh: Win7SP1x64

PROF=Win7SP1x64
$VOL2 --profile=$PROF pslist
$VOL2 --profile=$PROF cmdline
$VOL2 --profile=$PROF filescan | grep -i flag
$VOL2 --profile=$PROF dumpfiles -Q 0x000000003e1d40b0 -D /tmp/dump/
$VOL2 --profile=$PROF memdump -p 1234 -D /tmp/
$VOL2 --profile=$PROF clipboard
$VOL2 --profile=$PROF hashdump                  # ekstrak NTLM hash
```

#### Minidump analysis (Windows crash / stealer)

```python
#!/usr/bin/env python3
"""
Universal minidump scanner — flag, credentials, wide strings
"""
from minidump.minidumpfile import MinidumpFile
import re, sys

mf = MinidumpFile.parse(sys.argv[1])
reader = mf.get_reader()

# Patterns to hunt
patterns_bytes = [
    rb"FindITCTF\{[^\}]+\}",
    rb"FindIT\{[^\}]+\}",
    rb"flag\{[^\}]+\}",
]
patterns_text = [
    r"FindITCTF\{[^\}]+\}",
    r"FindIT\{[^\}]+\}",
    r"https?://[^\s'\"]+",
    r"password['\":]?\s*[='\"]?([A-Za-z0-9_!@#$%]{6,})",
]

print(f"[*] Total memory segments: {len(reader.memory_segments)}")
hits = set()

for seg in reader.memory_segments:
    try:
        data = reader.read(seg.start_virtual_address, seg.size)
    except Exception:
        continue

    for pat in patterns_bytes:
        for m in re.finditer(pat, data):
            hits.add(m.group(0).decode(errors="replace"))

    # UTF-16LE wide strings (Windows)
    try:
        wide = data.decode("utf-16le", errors="ignore")
        for pat in patterns_text:
            for m in re.finditer(pat, wide):
                hits.add(m.group(0))
    except Exception:
        pass

    # ASCII fallback
    try:
        ascii_text = data.decode("latin-1", errors="ignore")
        for pat in patterns_text:
            for m in re.finditer(pat, ascii_text):
                hits.add(m.group(0))
    except Exception:
        pass

print(f"\n[+] Hits ({len(hits)}):")
for h in sorted(hits):
    print(f"  {h}")

# List loaded modules
print(f"\n[*] Loaded modules:")
for m in mf.modules.modules[:20]:
    print(f"  0x{m.baseaddress:016x}  {m.name}")
```

**🎯 Catatan dari "weird stealer" (FindIT 2026 Quals):**
> Minidump dari Go binary → scan memory segments → ada **AEAD-encrypted payload** + **key di memory**. Pola: cari **patern key length umum** (16/24/32 bytes), lalu **nonce 12 bytes** untuk AES-GCM, dekripsi manual.

---

### 2D — NETWORK FORENSICS (PCAP)

**Tools:** Wireshark, `tshark`, NetworkMiner, `scapy`, `ngrep`

#### Quick triage

```bash
PCAP=capture.pcap

# 1. Statistik umum
capinfos $PCAP                                  # info file
tshark -r $PCAP -z io,stat,0                   # stats global
tshark -r $PCAP -z conv,tcp                    # TCP conversation list
tshark -r $PCAP -z hosts                       # daftar host

# 2. Protocol breakdown
tshark -r $PCAP -z io,phs                      # protocol hierarchy

# 3. Top talkers
tshark -r $PCAP -q -z endpoints,ip
```

#### Filter & ekstraksi

```bash
# Daftar URL HTTP
tshark -r $PCAP -Y "http.request" \
    -T fields -e ip.src -e http.host -e http.request.uri

# HTTP request + response code
tshark -r $PCAP -Y "http" -T fields \
    -e http.request.method -e http.host -e http.request.uri \
    -e http.response.code

# Follow TCP stream tertentu (text)
tshark -r $PCAP -q -z follow,tcp,ascii,5        # stream index 5

# Export semua HTTP objects
mkdir -p /tmp/http_obj
tshark -r $PCAP --export-objects http,/tmp/http_obj/
ls /tmp/http_obj/

# Export semua SMB / FTP / TFTP
tshark -r $PCAP --export-objects smb,/tmp/smb/
tshark -r $PCAP --export-objects ftp-data,/tmp/ftp/

# Cari credential plaintext
tshark -r $PCAP -Y "ftp.request.command == \"USER\" || ftp.request.command == \"PASS\"" \
    -T fields -e ftp.request.command -e ftp.request.arg
tshark -r $PCAP -Y "telnet" -T fields -e telnet.data
tshark -r $PCAP -Y "http.authorization" -T fields -e http.authorization

# DNS exfil detection (subdomain panjang & banyak query)
tshark -r $PCAP -Y "dns" -T fields -e dns.qry.name | sort | uniq -c | sort -rn | head -20

# Extract raw payload dari stream
tshark -r $PCAP -Y "tcp.stream eq 3" -T fields -e tcp.payload | \
    tr -d '\n,:' | xxd -r -p > /tmp/stream3.bin

# ICMP exfil
tshark -r $PCAP -Y "icmp" -T fields -e data.data | \
    tr -d '\n,:' | xxd -r -p
```

#### Wireshark display filters (untuk GUI)

```
http.request.method == "POST"
tcp contains "FindITCTF"
frame contains "flag"
tcp.flags.syn == 1 && tcp.flags.ack == 0       (SYN scan)
http.user_agent contains "curl"
ip.addr == 192.168.1.1
tcp.port == 4444                                (sering jadi reverse shell)
dns.qry.name matches ".*\.attacker\..*"
```

#### scapy approach (scripting custom)

```python
from scapy.all import rdpcap, TCP, Raw, DNS

packets = rdpcap("capture.pcap")
for pkt in packets:
    if pkt.haslayer(Raw):
        payload = pkt[Raw].load
        if b"FindIT" in payload:
            print(f"[{pkt[TCP].sport}→{pkt[TCP].dport}] {payload[:100]}")
    if pkt.haslayer(DNS) and pkt[DNS].qd:
        name = pkt[DNS].qd.qname.decode()
        if len(name) > 50:                      # subdomain panjang = exfil suspect
            print(f"[DNS exfil?] {name}")
```

**Common gotcha PCAP CTF:**
- File transfer terpotong jadi banyak TCP segment → gunakan **Follow TCP Stream** di Wireshark, save raw
- HTTP gzip/chunked → Wireshark auto-decode, tapi `tshark` butuh parameter tambahan
- **TLS** → kalau ada `(pre)-master-secret.log`, set di Wireshark Edit→Preferences→TLS untuk decrypt
- **USB capture** → field `usb.capdata`, decode HID keystroke (lihat USB HID code table)

---

### 2E — FORENSIC IMAGE FORMAT

#### AD1 (AccessData FTK)

```bash
AD1=usb_dump.ad1

# AD1 = proprietary, tidak bisa mount langsung tanpa FTK Imager
# WORKAROUND: binwalk untuk carving
binwalk -e $AD1 -C /tmp/ad1_extract/
ls -laR /tmp/ad1_extract/

# Verifikasi semua file dicarve dgn benar
for f in $(find /tmp/ad1_extract -type f); do
    echo "=== $f ==="
    file "$f"
done

# Kalau punya FTK Imager (Windows): File → Add Evidence Item → Image File → AD1
# Mac/Linux: tidak ada free tool resmi, hanya binwalk yang work

# Alternatif: ad1xtract (Python community tool)
pip install ad1xtract
ad1xtract $AD1 -o /tmp/ad1_out/
```

#### E01 (EnCase / Expert Witness)

```bash
E01=evidence.E01

# Info image
ewfinfo $E01                                    # metadata, MD5, jumlah segment

# Mount E01 → menjadi "ewf1" raw image
sudo mkdir -p /mnt/ewf
sudo ewfmount $E01 /mnt/ewf
ls /mnt/ewf                                     # ada file ewf1

# Lalu mount ewf1 sebagai biasa
sudo mount -o loop,ro /mnt/ewf/ewf1 /mnt/img
ls /mnt/img

# Atau langsung pakai sleuthkit
mmls /mnt/ewf/ewf1
fls -r /mnt/ewf/ewf1
```

#### Raw / DD image

```bash
RAW=image.dd

file $RAW
fdisk -l $RAW                                   # list partisi
mmls $RAW                                       # alternative

# Mount dengan offset (kalau ada partisi)
sudo mount -o loop,ro,offset=$((2048*512)) $RAW /mnt/img

# Atau langsung debugfs/sleuthkit untuk single partition
debugfs $RAW -R "ls -l /"
fls -r $RAW
```

---

### 2F — FILE CARVING & MAGIC BYTES

#### Tabel magic bytes WAJIB HAFAL

| Format | Header (Hex) | Footer (Hex) | Catatan |
|---|---|---|---|
| **JPEG** | `FF D8 FF E0` / `FF D8 FF E1` | `FF D9` | E0=JFIF, E1=EXIF |
| **PNG** | `89 50 4E 47 0D 0A 1A 0A` | `49 45 4E 44 AE 42 60 82` | IHDR start, IEND end |
| **GIF87a** | `47 49 46 38 37 61` | `00 3B` | |
| **GIF89a** | `47 49 46 38 39 61` | `00 3B` | |
| **PDF** | `25 50 44 46 2D` | `25 25 45 4F 46` | "%PDF-" / "%%EOF" |
| **ZIP** | `50 4B 03 04` | `50 4B 05 06` | Local + EOCD |
| **RAR** | `52 61 72 21 1A 07 00` | — | "Rar!\x1A\x07\x00" |
| **GZIP** | `1F 8B 08` | — | |
| **BZIP2** | `42 5A 68` | — | "BZh" |
| **7ZIP** | `37 7A BC AF 27 1C` | — | |
| **ELF** | `7F 45 4C 46` | — | "\x7FELF" |
| **PE/EXE** | `4D 5A` | — | "MZ" |
| **MACHO 64** | `CF FA ED FE` | — | |
| **WAV** | `52 49 46 46` ... `57 41 56 45` | — | RIFF...WAVE |
| **AVI** | `52 49 46 46` ... `41 56 49 20` | — | RIFF...AVI |
| **MP3 (ID3)** | `49 44 33` | — | "ID3" |
| **MP3 (raw)** | `FF FB` / `FF F3` / `FF F2` | — | sync byte |
| **FLAC** | `66 4C 61 43` | — | "fLaC" |
| **OGG** | `4F 67 67 53` | — | "OggS" |
| **SQLite** | `53 51 4C 69 74 65 20 66 6F 72 6D 61 74 20 33 00` | — | "SQLite format 3\0" |
| **TAR** | `75 73 74 61 72` (offset 257) | — | "ustar" |
| **DOCX/PPTX/XLSX** | `50 4B 03 04` | — | (ZIP-based) |
| **DOC/XLS lama** | `D0 CF 11 E0 A1 B1 1A E1` | — | OLE compound |
| **ISO** | `43 44 30 30 31` (offset 32769) | — | "CD001" |
| **VHD** | `63 6F 6E 65 63 74 69 78` | — | "conectix" |
| **REG** | `52 65 67 66` | — | "Regf" |

#### Manual carving workflow

```bash
INPUT=blob.bin

# Cari semua occurrence magic bytes
xxd $INPUT | grep "504b 0304"                   # ZIP — banyak hasil
xxd $INPUT | grep "ffd8 ffe0\|ffd8 ffe1"        # JPEG
xxd $INPUT | grep "8950 4e47"                   # PNG

# Konversi alamat dari hex (left column xxd) → decimal
# Misalnya line: 00098000: 504b 0304 0a00 ...
# → offset hex 0x98000 = decimal 622592

OFFSET=622592
SIZE=10000000   # estimate generous (lebih baik kebanyakan dari kekurangan)
dd if=$INPUT of=/tmp/carved.zip bs=1 skip=$OFFSET count=$SIZE 2>/dev/null
file /tmp/carved.zip                            # validasi

# Trim ekor sampah (untuk ZIP — cari EOCD signature 50 4B 05 06)
python3 << 'EOF'
data = open('/tmp/carved.zip','rb').read()
end = data.rfind(b'\x50\x4b\x05\x06')
if end > 0:
    # EOCD = 22 bytes minimum + comment
    eocd_size = 22 + int.from_bytes(data[end+20:end+22], 'little')
    open('/tmp/carved_clean.zip','wb').write(data[:end+eocd_size])
    print(f"[+] Trimmed to {end+eocd_size} bytes")
EOF
unzip -l /tmp/carved_clean.zip
```

#### Python carver (multi-format dalam satu file)

```python
#!/usr/bin/env python3
"""Universal magic-bytes carver"""
import sys, os

SIGNATURES = {
    'jpeg': (b'\xff\xd8\xff', b'\xff\xd9', '.jpg'),
    'png':  (b'\x89PNG\r\n\x1a\n', b'IEND\xaeB`\x82', '.png'),
    'pdf':  (b'%PDF-', b'%%EOF', '.pdf'),
    'zip':  (b'PK\x03\x04', None, '.zip'),         # variable end
    'gz':   (b'\x1f\x8b\x08', None, '.gz'),
    'elf':  (b'\x7fELF', None, '.elf'),
    'mz':   (b'MZ', None, '.exe'),
}

data = open(sys.argv[1], 'rb').read()
os.makedirs('/tmp/carved', exist_ok=True)

for name, (magic, end, ext) in SIGNATURES.items():
    pos = 0
    idx = 0
    while True:
        i = data.find(magic, pos)
        if i < 0: break
        if end:
            j = data.find(end, i + len(magic))
            if j > 0:
                blob = data[i:j+len(end)]
            else:
                blob = data[i:i+1024*1024]      # 1MB fallback
        else:
            blob = data[i:i+1024*1024]          # 1MB chunk
        out = f'/tmp/carved/{name}_{idx}{ext}'
        open(out, 'wb').write(blob)
        print(f"[+] {name} @ {i:#x} → {out} ({len(blob)} bytes)")
        idx += 1
        pos = i + len(magic)
```

---

## 3. ENCODING & DECODING

### Tabel cepat

| Encoding | Cara kenali | Decode CLI | Decode Python |
|---|---|---|---|
| **Base64** | A-Z, a-z, 0-9, +/, padding `=` | `base64 -d` | `base64.b64decode(s)` |
| **Base32** | A-Z, 2-7, padding `=` | `base32 -d` | `base64.b32decode(s)` |
| **Base58** | tidak ada 0,O,I,l | — | `base58.b58decode(s)` |
| **Base85/85** | wider charset | — | `base64.b85decode(s)` |
| **Hex** | hanya 0-9 a-f, panjang genap | `xxd -r -p` | `bytes.fromhex(s)` |
| **URL-encode** | banyak `%XX` | `urldecode` (web) | `urllib.parse.unquote(s)` |
| **HTML entity** | `&amp;`, `&#65;`, `&#x41;` | — | `html.unescape(s)` |
| **ROT13** | English text terbalik | `tr 'A-Za-z' 'N-ZA-Mn-za-m'` | `codecs.decode(s,'rot13')` |
| **Morse** | `.- -... -.-.` | — | `dcode.fr` / lib morse |
| **Binary** | hanya 0 dan 1 | — | `int(s,2).to_bytes(...)` |

### Command examples

```bash
# Base64 (text)
echo "RmluZElUQ1RGe2hlbGxvfQ==" | base64 -d
# binary safe:
echo "..." | base64 -d > out.bin

# Base32
echo "JBSWY3DPEB3W64TMMQ======" | base32 -d
# Output: Hello world

# Base85 (Adobe atau Z85)
python3 -c "import base64; print(base64.b85decode('Xk~0{Zy'))"

# Hex
echo "46696e644954" | xxd -r -p          # → FindIT
echo "Hello" | xxd -p                     # → 48656c6c6f

# ROT13 / ROT-N
echo "Uryyb" | tr 'A-Za-z' 'N-ZA-Mn-za-m'           # → Hello
# ROT-N (caesar) brute force:
for i in {1..25}; do
    echo "Shift $i: $(echo 'Uryyb' | tr "a-z" $(echo 'a-z' | tr 'a-z' "$(echo a-z | tr 'a-z' "$(printf '%s\n' a-z | sed 's/.*/&/')")"))"
done
# Easier — Python:
python3 -c "
s='Uryyb'
for n in range(26):
    print(n, ''.join(chr((ord(c)-97+n)%26+97) if c.islower() else (chr((ord(c)-65+n)%26+65) if c.isupper() else c) for c in s))
"

# URL decode
python3 -c "from urllib.parse import unquote; print(unquote('%46%6C%61%67'))"

# Binary string → ASCII
python3 -c "
b='01000110 01101100 01100001 01100111'
print(''.join(chr(int(x,2)) for x in b.split()))
"

# Integer → bytes (big-endian)
python3 -c "
n = 26852480951005303
print(n.to_bytes((n.bit_length()+7)//8, 'big'))
"

# Multiple integers comma-separated
python3 -c "
nums = '1601464371,1819045152,1734110208'
import struct
for n in nums.split(','):
    print(struct.pack('>I', int(n)))
"

# XOR brute force (single byte key)
python3 << 'EOF'
data = bytes.fromhex("3a232a3a262e26333a232f2a2e23")
for k in range(256):
    out = bytes(b ^ k for b in data)
    if all(32 <= c < 127 for c in out):
        try:
            t = out.decode()
            if any(w in t.lower() for w in ['flag','findit','ctf']):
                print(f"key={k:#x}: {t}")
        except: pass
EOF
```

### CyberChef Magic chains yang sering work

1. **`Magic`** (depth=5) — auto-detect & coba semua
2. **`From Base64` → `From Hex` → `Gunzip`** — common nested
3. **`From Hex` → `XOR Brute Force`** — short data
4. **`From Morse Code` → `ROT13`** — old-school chain
5. **`From Base64` → `From Base64` → `Gunzip`** — double base64
6. **`From Base85` → `From Base64`** — disguised data

---

## 4. METADATA & EXIFTOOL

```bash
# Extract semua metadata
exiftool image.jpg                              # human readable
exiftool -v3 image.jpg                          # verbose, struktur internal
exiftool -json image.jpg                        # JSON output
exiftool -G image.jpg                           # tampilkan group

# Field yang sering menyimpan flag
exiftool -Comment -UserComment -Description \
    -XPComment -ImageDescription -Software image.jpg

# GPS coordinates (kalau ada)
exiftool -GPS* image.jpg
exiftool -GPSLatitude -GPSLongitude -c "%.6f" image.jpg

# Recursive untuk semua file di direktori
exiftool -r /tmp/images/

# Comparison metadata sebelum/sesudah modifikasi
exiftool original.jpg > before.txt
exiftool modified.jpg > after.txt
diff before.txt after.txt

# Cari file dengan field tertentu
exiftool -if '$Comment' -p '$FileName: $Comment' /tmp/images/

# Field tersembunyi yang sering dilewatkan
exiftool -ee image.jpg                          # extract embedded data
exiftool -b -ThumbnailImage image.jpg > thumb.jpg   # ekstrak thumbnail
                                                # 🎯 thumbnail sering tidak ikut di-edit!
```

**🎯 Trik thumbnail JPEG:** Banyak gambar punya **thumbnail tersimpan di EXIF** yang menampilkan versi ASLI (sebelum di-crop / di-blur). Jangan lupa cek!

```bash
exiftool -b -ThumbnailImage suspect.jpg > thumb.jpg
xdg-open thumb.jpg
```

---

## 5. PYTHON SCRIPTS SIAP PAKAI

### A. Universal Flag Hunter

```python
#!/usr/bin/env python3
"""Multi-encoding flag hunter — jalankan pada apapun"""
import re, sys, base64

if len(sys.argv) < 2:
    print("Usage: flaghunter.py <file>")
    sys.exit(1)

with open(sys.argv[1], 'rb') as f:
    data = f.read()

PATTERNS = [
    rb"FindITCTF\{[^\}]+\}",
    rb"FindIT\{[^\}]+\}",
    rb"flag\{[^\}]+\}",
    rb"FLAG\{[^\}]+\}",
    rb"CTF\{[^\}]+\}",
]

print(f"[*] File size: {len(data)} bytes")

# Layer 1: raw search
for pat in PATTERNS:
    for m in re.finditer(pat, data, re.IGNORECASE):
        print(f"[+] RAW @ 0x{m.start():x}: {m.group(0).decode(errors='replace')}")

# Layer 2: UTF-16LE wide string
try:
    wide = data.decode("utf-16le", errors="ignore")
    for pat in PATTERNS:
        pat_str = pat.decode()
        for m in re.finditer(pat_str, wide, re.IGNORECASE):
            print(f"[+] WIDE: {m.group(0)}")
except Exception:
    pass

# Layer 3: try base64-decoded chunks (hunting for nested flag)
ascii_text = data.decode('latin-1', errors='ignore')
b64_pattern = re.compile(r'[A-Za-z0-9+/]{20,}={0,2}')
for m in b64_pattern.finditer(ascii_text):
    try:
        decoded = base64.b64decode(m.group(0) + '===')
        for pat in PATTERNS:
            for fm in re.finditer(pat, decoded, re.IGNORECASE):
                print(f"[+] B64 nested: {fm.group(0).decode(errors='replace')}")
    except Exception:
        pass

# Layer 4: hex-encoded chunks
hex_pattern = re.compile(rb'([0-9a-fA-F]{40,})')
for m in hex_pattern.finditer(data):
    try:
        decoded = bytes.fromhex(m.group(1).decode())
        for pat in PATTERNS:
            for fm in re.finditer(pat, decoded, re.IGNORECASE):
                print(f"[+] HEX nested: {fm.group(0).decode(errors='replace')}")
    except Exception:
        pass

print("[*] Done.")
```

### B. XOR Brute Force (single & multi-byte)

```python
#!/usr/bin/env python3
"""XOR brute force — single byte then short multi-byte"""
import sys, itertools

data = open(sys.argv[1], 'rb').read()
TARGET = b"FindIT"

# Single byte
print("=== Single byte ===")
for k in range(256):
    out = bytes(b ^ k for b in data)
    if TARGET in out:
        print(f"[+] key=0x{k:02x}: {out[:200]}")

# Multi-byte XOR cycling (sampai length 4)
print("\n=== Multi-byte (length 2-4) ===")
def xor_cycle(d, key):
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(d))

# Length 2-4: brute force kalau panjang data tidak besar
if len(data) < 1_000_000:
    for L in range(2, 5):
        # Cari first occurrence "Find" yg di-encrypt — assume plaintext starts with "FindIT"
        for offset in range(min(len(data), 1024)):
            chunk = data[offset:offset+6]
            if len(chunk) < 6: break
            key = bytes(c ^ p for c, p in zip(chunk, b"FindIT"))[:L]
            test = xor_cycle(data[offset:offset+200], key * (200 // L + 1))
            if b"FindITCTF{" in test:
                print(f"[+] L={L} key={key.hex()} offset={offset}: {test[:200]}")
                break
```

### C. ZIP / PDF Password Brute Force

```python
#!/usr/bin/env python3
"""ZIP password brute force pakai wordlist + common CTF passwords"""
import zipfile, sys

CTF_COMMON = [
    "", "password", "admin", "flag", "secret", "ctf",
    "FindIT", "findit", "findit2026", "FindITCTF",
    "infected", "malware", "stego", "hidden",
    "12345", "123456", "password123", "qwerty",
    "letmein", "welcome",
]

def try_zip(zfile, passwords):
    with zipfile.ZipFile(zfile) as zf:
        for pw in passwords:
            try:
                zf.extractall(path="/tmp/zip_out", pwd=pw.encode())
                return pw
            except (RuntimeError, zipfile.BadZipFile):
                continue
    return None

zfile = sys.argv[1]

# 1. Common passwords first
pw = try_zip(zfile, CTF_COMMON)
if pw is not None:
    print(f"[+] Common: '{pw}'")
    sys.exit(0)

# 2. Wordlist
with open("/usr/share/wordlists/rockyou.txt", encoding="latin-1") as f:
    wl = (line.strip() for line in f)
    pw = try_zip(zfile, wl)
    if pw is not None:
        print(f"[+] Rockyou: '{pw}'")
```

### D. Steghide Brute Force

```python
#!/usr/bin/env python3
"""Brute force steghide password"""
import subprocess, sys

CTF_PWS = [
    "", "password", "steghide", "secret", "flag",
    "FindIT", "findit", "findit2026", "FindITCTF",
    "ctf", "stego", "hidden", "key",
]

img = sys.argv[1]
for pw in CTF_PWS:
    result = subprocess.run(
        ["steghide", "extract", "-sf", img, "-p", pw, "-f"],
        capture_output=True, text=True
    )
    if result.returncode == 0:
        print(f"[+] Password: '{pw}'")
        print(result.stdout)
        sys.exit(0)

# Fallback ke stegseek (lebih cepat)
print("[*] Fallback to stegseek...")
subprocess.run(["stegseek", img, "/usr/share/wordlists/rockyou.txt"])
```

### E. Layered Decoding (Base64 → Hex → ASCII)

```python
#!/usr/bin/env python3
"""Auto-decode chains layered encoding"""
import base64, re, sys

def try_b64(s):
    try: return base64.b64decode(s + "===").decode()
    except: return None

def try_hex(s):
    try: return bytes.fromhex(s).decode()
    except: return None

def try_b32(s):
    try: return base64.b32decode(s + "=" * (8 - len(s) % 8)).decode()
    except: return None

def is_flag(s):
    return bool(re.search(r"FindIT[A-Z]*\{", s or ""))

s = sys.argv[1]
print(f"[0] Input: {s[:100]}")

# Coba semua kombinasi sampai depth 5
queue = [(s, 0, [])]
while queue:
    cur, depth, path = queue.pop(0)
    if depth > 5: continue
    if is_flag(cur):
        flag = re.search(r"FindIT[A-Z]*\{[^}]+\}", cur).group(0)
        print(f"[+] FLAG: {flag}")
        print(f"    Path: {' → '.join(path)}")
        sys.exit(0)
    for name, fn in [("b64", try_b64), ("hex", try_hex), ("b32", try_b32)]:
        nxt = fn(cur)
        if nxt and nxt != cur and len(nxt) > 5:
            queue.append((nxt, depth+1, path + [name]))

print("[-] Tidak ditemukan flag — coba manual")
```

### F. Spectrogram Generator (audio → image cepat)

```python
#!/usr/bin/env python3
"""Generate spectrogram dgn matplotlib (alternative ke sox)"""
import sys
import numpy as np
import matplotlib.pyplot as plt
from scipy.io import wavfile

rate, data = wavfile.read(sys.argv[1])
if data.ndim > 1:
    data = data[:, 0]                           # ambil channel 0

plt.figure(figsize=(20, 8))
plt.specgram(data, NFFT=2048, Fs=rate, noverlap=1024, cmap="viridis")
plt.ylabel("Frequency [Hz]")
plt.xlabel("Time [s]")
plt.savefig("spectrogram.png", dpi=150, bbox_inches="tight")
print("[+] Saved spectrogram.png")
```

---

## 6. TOOLS INSTALLATION

### Quick install (Kali / Ubuntu)

```bash
# Update
sudo apt update

# Forensics core
sudo apt install -y \
    exiftool binwalk foremost steghide pngcheck \
    sleuthkit autopsy testdisk \
    ewf-tools afflib-tools \
    sox libsox-fmt-all audacity \
    wireshark tshark tcpdump \
    qpdf poppler-utils \
    ent xxd \
    ruby ruby-dev

# zsteg (Ruby gem — wajib untuk PNG LSB)
sudo gem install zsteg

# stegseek (steghide brute force — jauh lebih cepat dari hashcat)
wget https://github.com/RickdeJager/stegseek/releases/latest/download/stegseek_0.6-1.deb
sudo apt install -y ./stegseek_0.6-1.deb

# Python forensic packages
pip3 install --user \
    minidump volatility3 \
    pycryptodome \
    Pillow numpy scipy matplotlib \
    stegano scapy \
    requests cryptography \
    python-magic

# Volatility 3 (clone fresh kalau pip versi tua)
cd ~ && git clone https://github.com/volatilityfoundation/volatility3.git
cd volatility3 && pip3 install -r requirements.txt
python3 vol.py --help

# Volatility 2 (untuk Windows XP/7 lama — pakai Python 2)
git clone https://github.com/volatilityfoundation/volatility.git
cd volatility && python2 vol.py --info

# StegSolve (GUI — Java)
wget http://www.caesum.com/handbook/Stegsolve.jar -O ~/Stegsolve.jar
java -jar ~/Stegsolve.jar &

# Wordlist (kalau belum ada rockyou)
sudo gunzip /usr/share/wordlists/rockyou.txt.gz 2>/dev/null
```

### Verifikasi tools (jalankan dulu sebelum lomba!)

```bash
echo "=== Forensic tools check ==="
for cmd in file exiftool binwalk foremost steghide stegseek zsteg \
           pngcheck debugfs mmls fls icat fdisk ewfinfo ewfmount \
           sox tshark wireshark python3 strings xxd; do
    if command -v $cmd >/dev/null 2>&1; then
        echo "  ✓ $cmd"
    else
        echo "  ✗ $cmd MISSING"
    fi
done

echo "=== Python modules ==="
python3 -c "import minidump, Crypto, PIL, scapy.all, stegano; print('✓ all OK')"
```

---

## 7. SKENARIO PREDIKSI FINAL

### Skenario A — Disk Image + Stego Hybrid (HIGH probability)

**Signal:** disk.img / disk.dd / .ad1 ber-size besar (>50MB)

**Playbook:**
```bash
# 1. Identifikasi
file disk.img && mmls disk.img

# 2. Browse filesystem
debugfs disk.img -R "ls -la /home"
sudo mount -o loop,ro disk.img /mnt/img

# 3. Cari file mencurigakan (gambar, audio, archive)
find /mnt/img -type f \( -name "*.png" -o -name "*.jpg" -o -name "*.wav" -o -name "*.zip" -o -name "*.dat" \)

# 4. Cek bash history untuk clue tools/passwords
cat /mnt/img/home/*/.bash_history

# 5. Kalau ada deleted file
debugfs disk.img -R "lsdel"
# Atau raw carving:
foremost -i disk.img -o /tmp/carved/

# 6. Stego analysis pada file yang ditemukan
zsteg -a /mnt/img/path/to/image.png
steghide info /mnt/img/path/to/image.jpg
sox /mnt/img/path/to/audio.wav -n spectrogram -o spec.png
```

### Skenario B — Memory Dump + Malware

**Signal:** .dmp / .raw / .vmem 200MB+, hint "stealer"/"ransomware"

**Playbook:**
```bash
# 1. Quick strings
strings -n 8 mem.dmp | grep -oP "FindIT(CTF)?\{[^}]+\}"
strings -el mem.dmp | grep -oP "FindIT(CTF)?\{[^}]+\}"

# 2. Volatility processes
python3 vol.py -f mem.dmp windows.pslist
python3 vol.py -f mem.dmp windows.cmdline | grep -iE "powershell|encoded|http"

# 3. Network indicators
python3 vol.py -f mem.dmp windows.netscan
strings mem.dmp | grep -oE "https?://[^ '\"]+" | sort -u

# 4. Suspicious process → dump memory
python3 vol.py -f mem.dmp windows.memmap --pid <PID> --dump

# 5. Kalau dump = minidump
python3 minidump_scan.py mem.dmp     # script C section 5

# 6. Cari Crypto key (32 bytes high entropy block)
python3 -c "
data = open('mem.dmp','rb').read()
import math
def entropy(b):
    if not b: return 0
    p = [b.count(c)/len(b) for c in set(b)]
    return -sum(x*math.log2(x) for x in p if x>0)
for i in range(0, len(data)-32, 1024):
    chunk = data[i:i+32]
    if entropy(chunk) > 7.5:
        print(f'{i:#x}: {chunk.hex()}')
"
```

### Skenario C — PCAP + Encoded Exfil

**Signal:** .pcap / .pcapng dengan banyak HTTP / DNS query

**Playbook:**
```bash
# 1. Overview
capinfos capture.pcap
tshark -r capture.pcap -z io,phs

# 2. HTTP objects export
mkdir -p /tmp/http_obj
tshark -r capture.pcap --export-objects http,/tmp/http_obj/
ls /tmp/http_obj/

# 3. Search interesting strings di payload
tshark -r capture.pcap -Y "tcp" -T fields -e tcp.payload | \
    grep -aoE "FindIT(CTF)?\{[^}]+\}"

# 4. DNS exfil — ekstrak subdomain panjang
tshark -r capture.pcap -Y "dns.qry.name" -T fields -e dns.qry.name | \
    awk -F. 'length($1) > 20 {print $1}' | head

# 5. Chunked exfil → reassemble
tshark -r capture.pcap -Y "dns" -T fields -e dns.qry.name | \
    grep '\.attacker\.com$' | \
    awk -F. '{print $1}' | tr -d '\n' | base64 -d
```

### Skenario D — Pure Stego (flag.png)

**Signal:** Hanya 1 image diberikan (sesuai hint panitia: "*ak steagno*")

**Playbook (kombinasi semua sebagai shotgun):**
```bash
IMG=flag.png

# Quick triage
file $IMG && exiftool $IMG && pngcheck -v $IMG

# Strings
strings -n 5 $IMG | grep -iE "findit|flag|password"

# Append data
binwalk $IMG && binwalk -e $IMG

# LSB stego (PNG)
zsteg -a $IMG > zsteg.txt && cat zsteg.txt | grep -iE "text|FindIT"

# JPEG → steghide
steghide info $IMG
stegseek $IMG /usr/share/wordlists/rockyou.txt

# Visual (StegSolve)
java -jar ~/Stegsolve.jar    # buka file, cycle bit planes

# Manual LSB extract
python3 -c "
from PIL import Image
img = Image.open('$IMG').convert('RGB')
bits = ''
for y in range(img.height):
    for x in range(img.width):
        r,g,b = img.getpixel((x,y))
        bits += str(r&1) + str(g&1) + str(b&1)
chars = [chr(int(bits[i:i+8],2)) for i in range(0,len(bits)-7,8)]
text = ''.join(chars)
import re
for m in re.finditer(r'FindIT[A-Z]*\{[^}]+\}', text): print(m.group())
"

# Alpha channel (kalau RGBA)
zsteg -E a1,a,lsb,xy $IMG

# Kalau gagal semua → cek thumbnail EXIF
exiftool -b -ThumbnailImage $IMG > thumb.jpg
xdg-open thumb.jpg
```

### Skenario E — Multi-layer Encoding

**Signal:** File text/note dengan blob aneh, atau output volatility yang berisi base64

**Playbook:**
```bash
# 1. Identifikasi encoding via charset
echo "BLOB" | head -c 100
# Hanya A-Z2-7=  → Base32
# A-Za-z0-9+/=  → Base64
# A-F0-9        → Hex
# Charset wider → Base85/Base91

# 2. Coba decode chain otomatis (script E section 5)
python3 layered_decoder.py "BLOB_STRING"

# 3. CyberChef Magic (browser):
#    https://gchq.github.io/CyberChef/
#    Paste → Magic → Intensive: 5

# 4. Kalau ada XOR layer di tengah
python3 xor_brute.py blob.bin
```

---

## 8. ⚡ QUICK REFERENCE

> **Cetak halaman ini! Tempel di samping monitor saat lomba.**

### One-liner triage (jalankan SETIAP file challenge baru)

```bash
F=challenge_file
file $F; ls -la $F; md5sum $F
exiftool $F | head -30
strings -n 6 $F | grep -iE "FindIT|flag|password" | head
strings -el $F | grep -iE "FindIT|flag" | head
binwalk $F
```

### Magic bytes paling penting

```
JPEG: FF D8 FF        ZIP:  50 4B 03 04
PNG:  89 50 4E 47     PDF:  25 50 44 46
GIF:  47 49 46 38     ELF:  7F 45 4C 46
WAV:  52 49 46 46     PE:   4D 5A
GZ:   1F 8B 08        7Z:   37 7A BC AF
```

### Cheat command per format

| File | Tool urutan |
|---|---|
| **PNG** | `pngcheck -v` → `zsteg -a` → `binwalk -e` → StegSolve |
| **JPG** | `exiftool` → `steghide info` → `stegseek` → `binwalk` |
| **WAV** | `sox … spectrogram` → Audacity → strings |
| **disk.img** | `mmls` → `debugfs ls` → `lsdel` → `foremost` |
| **AD1** | `binwalk -e` → cek `/tmp/_ad1.extracted` |
| **E01** | `ewfmount` → `mount /ewf1` |
| **PCAP** | `capinfos` → `tshark -z io,phs` → `--export-objects` |
| **DMP** | `strings -el` → `volatility3 pslist` → `minidump.py` |

### Common CTF passwords (steghide / ZIP)

```
""              password        flag            secret
admin           steghide        FindIT          findit2026
ctf             stego           hidden          findit
12345           qwerty          letmein         infected
```

### Common flag locations

| Tipe | Lokasi sering |
|---|---|
| Image | EXIF Comment, LSB pixel, append after IEND, thumbnail |
| Audio | Spectrogram, ID3 Comment, LSB, file append |
| Disk | `/home/*/note.txt`, bash_history, deleted files, `.git` |
| Memory | strings raw, process cmdline, clipboard, registry |
| PCAP | HTTP body, DNS subdomain (concat), TCP payload |
| ZIP | comment, hidden file, encrypted file content |

### Regex flag yang dipakai berulang

```python
FLAG_REGEX = [
    rb"FindITCTF\{[^\}]+\}",
    rb"FindIT\{[^\}]+\}",
    rb"flag\{[^\}]+\}",
]
```

### Strategi waktu (7 jam)

```
10:00–10:30   Baca SEMUA soal forensic, urutkan by point/difficulty
10:30–12:00   Quick wins: stego sederhana, strings, exiftool flag
12:00–14:00   Medium: disk image, PCAP analysis, encoding chain
14:00–15:30   Hard: memory dump, multi-step chain
15:30–16:00   ⚠️ SUBMIT semua flag SEBELUM scoreboard freeze (16:00)
16:00–17:00   Sapu bersih + screenshot untuk writeup
```

### Mental model — Kalau stuck > 30 menit

1. ✅ Sudah cek `file` + `exiftool` + `strings` + `binwalk`?
2. ✅ Sudah cek hidden/deleted file (kalau disk)?
3. ✅ Sudah cek metadata thumbnail (kalau image)?
4. ✅ Sudah coba decode multi-layer (kalau text aneh)?
5. ✅ Sudah cek alpha channel & semua bit plane (kalau image)?
6. ✅ Sudah brute force password dengan CTF common list?
7. **Kalau semua sudah → SWITCH KE SOAL LAIN.** Kembali nanti dengan fresh mind.

---

## 📚 REFERENSI EXTERNAL (boleh diakses saat lomba)

- **HackTricks Forensics:** book.hacktricks.xyz/forensics/basic-forensic-methodology
- **CyberChef:** gchq.github.io/CyberChef
- **PNG chunk reference:** w3.org/TR/PNG/#11Chunks
- **Volatility cheatsheet:** github.com/volatilityfoundation/volatility/wiki
- **dcode.fr:** untuk caesar / morse / classical cipher
- **CrackStation:** cek hash kalau dapat MD5/SHA

---

*Disusun untuk persiapan FindIT CTF 2026 Final · Materi belajar mandiri · Tidak digunakan saat jam kompetisi (face cam aktif). Semua teknik publicly documented dan standar di bidang digital forensics.*
