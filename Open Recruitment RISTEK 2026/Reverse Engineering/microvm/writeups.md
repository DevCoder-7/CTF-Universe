# CTF Writeup — microvm

**Kategori:** Reverse Engineering  
**Author Soal:** kannrisha  
**Format Flag:** `NETSOS{...}`  
**File:** `vm`, `checker.bin`

> *"memang reverse engineer wajib ngerti per VM VM an"*

---

## 1. Pendahuluan

Challenge **microvm** merupakan soal reverse engineering yang melibatkan custom virtual machine (VM). Peserta diberikan dua file:

- `vm` — binary ELF 64-bit yang berperan sebagai interpreter/engine VM
- `checker.bin` — file bytecode yang dijalankan oleh VM tersebut

Objective challenge ini adalah memahami arsitektur VM, mendisassemble bytecode, lalu merekonstruksi logika pengecekan flag sehingga kita bisa menurunkan flag yang valid.

---

## 2. Analisis File

### 2.1 Identifikasi File

```bash
$ file vm checker.bin

vm:          ELF 64-bit LSB pie executable, x86-64, dynamically linked,
             with debug_info, not stripped
checker.bin: data
```

File `vm` adalah binary ELF 64-bit yang **not stripped** — artinya simbol debug masih ada, ini sangat mempermudah analisis. File `checker.bin` adalah data biner mentah yang merupakan bytecode untuk VM.

### 2.2 Eksplorasi Strings pada vm

```bash
$ strings vm

unknown opcode: 0x%02x at pc=%ld
cycle limit reached
:: Bytecode file not provided, fallback to debug mode
[1] Enter VM code
[2] Execute VM code
[3] Exit
MicroVM v1.0
```

Dari strings tersebut, kita mengetahui bahwa VM ini memiliki **debug mode** interaktif dan mendukung eksekusi dari file bytecode. Ada juga nama fungsi yang terlihat: `fetch8`, `fetch32`, `fetch64`, `vm_run`, `vm_syscall`, `update_flags`, `check_reg`, `check_ram`.

---

## 3. Arsitektur Virtual Machine

Dengan menganalisis fungsi `vm_run` melalui disassembly (`objdump -d vm`), kita dapat memetakan arsitektur lengkap VM ini.

### 3.1 Register & Memory

| Komponen | Ukuran | Keterangan |
|---|---|---|
| Register r0 – r7 | 8 × 64-bit | General purpose registers |
| RAM | 64 KB (0x10000 byte) | Memori utama VM |
| Stack Pointer (SP) | 64-bit | Mulai dari offset 0x800, tumbuh ke bawah |
| Program Counter (PC) | 64-bit | Offset ke dalam bytecode |
| Flags | 64-bit | Zero (bit 0), Negative (bit 1), Carry (bit 2) |
| Bytecode Buffer | Maks 8 KB | Buffer tempat `checker.bin` dimuat |

### 3.2 Instruction Set (Opcode Map)

Dari analisis `vm_run` yang merupakan dispatch table besar (switch-case per opcode):

| Opcode | Instruksi | Format | Deskripsi |
|---|---|---|---|
| `0x01` | MOV | reg, imm64 | Load konstanta 64-bit ke register |
| `0x29` | MOVI | reg, imm32 | Load konstanta 32-bit ke register |
| `0x02` | MOV | dst, src | Copy antar register |
| `0x03` | ADD | dst, src | dst = dst + src |
| `0x04` | SUB | dst, src | dst = dst − src |
| `0x05` | MUL | dst, src | dst = dst × src |
| `0x06` | DIV | dst, src | dst = dst / src (signed) |
| `0x07` | MOD | dst, src | dst = dst % src |
| `0x08` | AND | dst, src | dst = dst & src |
| `0x09` | OR | dst, src | dst = dst \| src |
| `0x0A` | XOR | dst, src | dst = dst ^ src |
| `0x0B` | NOT | reg | reg = ~reg |
| `0x0C` | SHL | dst, src | dst = dst << (src & 0x3F) |
| `0x0D` | SHR | dst, src | dst = dst >> (src & 0x3F) |
| `0x0E` | CMP | dst, src | Hitung dst − src, update flags |
| `0x0F` | JMP | addr32 | Loncat ke alamat |
| `0x10` | JZ | addr32 | Loncat jika Zero flag set |
| `0x11` | JNZ | addr32 | Loncat jika Zero flag tidak set |
| `0x12` | JNS | addr32 | Loncat jika Negative flag tidak set |
| `0x13` | JS | addr32 | Loncat jika Negative flag set |
| `0x16` | PUSH | reg | Simpan register ke stack |
| `0x17` | POP | reg | Ambil nilai dari stack ke register |
| `0x1E` | LOAD8 | dst, [src] | Baca 1 byte dari RAM[src] |
| `0x1F` | STORE8 | [dst], src | Tulis 1 byte ke RAM[dst] |
| `0x1D` | STORE16 | [dst], src | Tulis 2 byte ke RAM[dst] |
| `0x18` | LOAD64 | dst, [src] | Baca 8 byte dari RAM[src] |
| `0x19` | STORE64 | [dst], src | Tulis 8 byte ke RAM[dst] |
| `0x20` | SYSCALL | — | System call (read/write/exit) |
| `0x21` | ADDI | dst, imm8 | dst += imm (signed 8-bit) |
| `0x23` | INC | reg | reg += 1 |
| `0x24` | DEC | reg | reg −= 1 |
| `0x25` | CALL | addr32 | Panggil fungsi, push return addr |
| `0x26` | RET | — | Return dari fungsi |
| `0x27` | XCHG | dst, src | Tukar nilai dua register |
| `0x2A` | NEG | reg | reg = −reg |
| `0x2B` | TEST | dst, src | AND tanpa simpan hasil, update flags |
| `0xFF` | HALT | — | Hentikan VM |

### 3.3 Syscall Interface

Opcode `0x20` (SYSCALL) memanggil fungsi `vm_syscall`. Register `r0` menentukan nomor syscall:

```
Syscall 0 (read):
  r0=0, r1=fd, r2=buffer_addr(RAM), r3=size
  → membaca dari stdin ke RAM

Syscall 1 (write):
  r0=1, r1=fd, r2=buffer_addr(RAM), r3=size
  → menulis dari RAM ke stdout

Syscall 2 (exit):
  r0=2 → menghentikan VM
```

---

## 4. Analisis Bytecode checker.bin

Untuk memahami bytecode, kita tulis disassembler Python sederhana berdasarkan instruction set yang sudah dipetakan.

### 4.1 Script Disassembler

```python
import struct

with open('checker.bin', 'rb') as f:
    code = f.read()

pos = 0

def fetch8():
    global pos
    v = code[pos]; pos += 1; return v

def fetch32():
    global pos
    v = struct.unpack_from('<I', code, pos)[0]; pos += 4; return v

def fetch64():
    global pos
    v = struct.unpack_from('<Q', code, pos)[0]; pos += 8; return v

while pos < len(code):
    start = pos
    op = fetch8()
    if op == 0x01:
        reg = fetch8(); imm = fetch64()
        print(f"{start:06x}: MOV r{reg}, {imm}")
    elif op == 0x29:
        reg = fetch8(); imm = fetch32()
        print(f"{start:06x}: MOVI r{reg}, {imm}")
    elif op == 0x20:
        print(f"{start:06x}: SYSCALL")
    # ... (dan seterusnya untuk setiap opcode)
```

### 4.2 Fase 1 — Setup String di RAM

Bagian awal bytecode (offset `0x000` – `0x1A3`) berisi serangkaian instruksi `MOVI` + `STORE8` yang menulis string ke RAM:

```asm
; Menulis "Enter flag: " ke RAM[0x100]
000000: MOVI r0, 69       ; 'E'
000006: MOVI r1, 256      ; addr = 0x100
00000c: STORE8 [r1], r0
00000f: MOVI r0, 110      ; 'n'
000015: MOVI r1, 257
00001b: STORE8 [r1], r0
         ... (dan seterusnya)

; Menulis "Correct!\n" ke RAM[0x120]
0000b4: MOVI r0, 67       ; 'C'
0000ba: MOVI r1, 288      ; addr = 0x120
0000c0: STORE8 [r1], r0
         ...

; Menulis "Wrong!\n" ke RAM[0x140]
00013b: MOVI r0, 87       ; 'W'
000141: MOVI r1, 320      ; addr = 0x140
000147: STORE8 [r1], r0
```

### 4.3 Fase 2 — Loading Tabel Expected Values

Setelah setup string, bytecode memuat **43 nilai 16-bit** ke `RAM[0x200 .. 0x256]`. Nilai-nilai ini adalah target pengecekan:

```asm
; Tabel expected values di RAM[0x200]
0001a4: MOVI r0, 5414   ; expected[0]
0001aa: MOVI r1, 512    ; RAM addr = 0x200
0001b0: STORE16 [r1], r0

0001b3: MOVI r0, 4792   ; expected[1]
0001b9: MOVI r1, 514    ; RAM addr = 0x202
0001bf: STORE16 [r1], r0

         ... (43 entri total)
```

Tabel lengkap 43 nilai:

```python
expected = [
    5414, 4792, 5766, 5756, 5487, 5754, 8449, 7703, 7826, 3478,
    8046, 8047, 8369, 6582, 7889, 8043, 3636, 7575, 6854, 3639,
    7822, 6849, 6573, 8185, 7513, 3167, 3164, 3165, 6567, 7579,
    3278, 8236, 6619, 7793, 8273, 7653, 6623, 7176, 7938, 6620,
    4074, 4637, 8699
]
```

### 4.4 Fase 3 — Input & Validasi Flag

```asm
; Print "Enter flag: "
000429: MOVI r0, 1    ; syscall write
00042f: MOVI r1, 1    ; fd = stdout
000435: MOVI r2, 256  ; buffer = RAM[0x100]
00043b: MOVI r3, 12   ; size = 12
000441: SYSCALL

; Read input ke RAM[0x00]
000442: MOVI r0, 0    ; syscall read
000448: MOVI r1, 0    ; fd = stdin
00044e: MOVI r2, 0    ; buffer = RAM[0x00]
000454: MOVI r3, 256  ; size = 256
00045a: SYSCALL

; r5 = bytes_read - 1 (strip newline)
00045b: MOV  r5, r0
00045e: SUBI r5, 1

; Cek panjang == 43
000472: MOV  r0, r5
000475: MOVI r7, 43
00047b: CMP  r0, r7
00047e: JNZ  1272        ; salah panjang -> Wrong!

; === Loop pengecekan (i = 0 .. 42) ===
000483: MOVI r4, 0        ; i = 0

; LOOP_START:
000489: LOAD8  r0, [r4]   ; r0 = flag[i]
00048c: MOVI   r6, 255
000492: AND    r0, r6     ; r0 &= 0xFF
000495: MOVI   r2, 69
00049b: MUL    r0, r2     ; r0 *= 69
00049e: MOV    r2, r4
0004a1: ADDI   r2, 32     ; r2 = i + 32
0004a4: XOR    r0, r2     ; r0 ^= (i + 32)
0004a7: MOV    r2, r4
0004aa: MOVI   r3, 2
0004b0: MUL    r2, r3     ; r2 = i * 2
0004b3: MOVI   r3, 512    ; r3 = 0x200
0004b9: ADD    r2, r3     ; r2 = 0x200 + i*2
0004bc: LOAD16 r5, [r2]   ; r5 = expected[i]
0004bf: MOVI   r6, 65535
0004c5: AND    r5, r6     ; r5 &= 0xFFFF
0004c8: CMP    r0, r5     ; bandingkan hasil vs expected
0004cb: JNZ    1272       ; tidak sama -> Wrong!
0004d0: INC    r4         ; i++
0004d2: CMP    r4, r7     ; i < 43?
0004d5: JS     1161       ; loop kembali

; Semua benar -> Correct!
0004da: MOVI r2, 288
0004ec: MOVI r3, 9
0004f2: SYSCALL
```

---

## 5. Reverse Engineering — Menurunkan Flag

### 5.1 Memahami Algoritma Check

Dari analisis bytecode, persamaan pengecekan untuk setiap karakter flag adalah:

```
(flag[i] & 0xFF) * 69 XOR (i + 32) == expected[i]
```

Karena XOR bersifat invertible, kita bisa balik persamaannya. Untuk setiap indeks `i` (0 sampai 42), kita cari karakter `c` dalam rentang ASCII printable (32–126) yang memenuhi:

```
(c * 69) ^ (i + 32) == expected[i]
```

### 5.2 Script Solusi

```python
expected = [
    5414, 4792, 5766, 5756, 5487, 5754, 8449, 7703, 7826, 3478,
    8046, 8047, 8369, 6582, 7889, 8043, 3636, 7575, 6854, 3639,
    7822, 6849, 6573, 8185, 7513, 3167, 3164, 3165, 6567, 7579,
    3278, 8236, 6619, 7793, 8273, 7653, 6623, 7176, 7938, 6620,
    4074, 4637, 8699
]

flag = []
for i in range(43):
    for c in range(32, 127):  # ASCII printable
        if (c * 69) ^ (i + 32) == expected[i]:
            flag.append(chr(c))
            break

print('Flag:', ''.join(flag))
```

### 5.3 Eksekusi & Verifikasi

```bash
$ python3 solve.py
Flag: NETSOS{pr3tty_st4nd4rd_vm..._n0w_pwn_it_:D}

$ ./vm checker.bin
Enter flag: NETSOS{pr3tty_st4nd4rd_vm..._n0w_pwn_it_:D}
Correct!
```

---

## 6. Flag

```
NETSOS{pr3tty_st4nd4rd_vm..._n0w_pwn_it_:D}
```

---

## 7. Ringkasan Langkah Penyelesaian

1. **Identifikasi file** — `vm` adalah ELF interpreter, `checker.bin` adalah bytecode VM.
2. **Reverse engineering VM** — Analisis `vm_run` dan `vm_syscall` via `objdump` untuk memetakan seluruh instruction set (~35+ opcode).
3. **Tulis disassembler** — Script Python untuk membaca `checker.bin` dan mentranslasikan setiap byte sesuai opcode map.
4. **Analisis bytecode** — Identifikasi 3 fase: setup string di RAM, loading tabel 43 nilai 16-bit, loop validasi karakter.
5. **Reverse persamaan check** — `(flag[i] & 0xFF) * 69 ^ (i + 32) == expected[i]`, bruteforce karakter printable per index.
6. **Dapatkan flag** — `NETSOS{pr3tty_st4nd4rd_vm..._n0w_pwn_it_:D}`

---

## 8. Tips & Pembelajaran

- Ketika menghadapi custom VM, selalu mulai dengan **memetakan instruction set secara lengkap** sebelum mencoba memahami bytecode.
- File yang **not stripped** sangat membantu — nama fungsi seperti `fetch8`, `fetch32`, `vm_run`, `vm_syscall` langsung terlihat di disassembly.
- Perhatikan pola **MOVI + STORE berulang** di awal bytecode — biasanya itu setup data/string/tabel.
- Algoritma check VM sering sederhana; yang penting adalah memahami **encoding instruksi** dengan benar.
- Selalu **verifikasi solusi** dengan menjalankan binary asli untuk memastikan flag diterima.
