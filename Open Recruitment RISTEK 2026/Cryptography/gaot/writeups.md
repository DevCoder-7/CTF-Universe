# Writeup: Goat 🐐

**Event:** NETSOS CTF  
**Category:** Cryptography  
**Difficulty:** Hard  
**Flag:** `NETSOS{tipsenn_nokiaa_kanrisshhaaa_fahruLLL_sushiikannnn_simbolll_ma_gaooooottt_ma_kambingg!!!!}`

---

## Overview

Challenge **Goat** merupakan tantangan kriptografi multi-tahap yang menggabungkan tiga kelemahan kriptografi berbeda secara berantai:

1. **Truncated LCG** — Memulihkan seed dari 10 output yang dipotong (hanya 33 bit teratas dari state 67-bit).
2. **AGCD (Approximate GCD)** — Memulihkan bilangan prima rahasia `secret_xor` dari 5 sampel berstruktur `q·p + r`.
3. **Linear AES** — Memulihkan flag menggunakan serangan known-plaintext memanfaatkan modifikasi AES yang menghilangkan `SubBytes`.

File yang diberikan:
```
chall.py   — logika server utama
aes.py     — implementasi AES (yang dimodifikasi)
xor.py     — utilitas XOR
```

---

## Analisis Source Code

### `chall.py`

```python
# Seed LCG 67-bit
seed = secrets.randbits(67)
lcg  = LCG(seed)
leak = [lcg.next() >> 34 for _ in range(10)]   # hanya 33 bit teratas

# enc(plain) = [q*p + r, ...]  dimana q < p, r < 2^50
def enc(plain):
    return [secrets.randbelow(plain)*plain + secrets.randbelow(2**BITS) for _ in range(5)]

# Flag dienkripsi dengan AES (tanpa SubBytes) + XOR dengan secret_xor
flag = xor(flag, long_to_bytes(secret_xor))
enc_flag = AES(key).encrypt_block(...)

# Oracle diberikan:
print("wut is this??? ", enc(secret_xor))
print("Here is one of our goats: ", c.encrypt_block(b"tipsen my gaot!!").hex())
print("enc_flag:", enc_flag.hex())
```

### `aes.py` (bagian krusial)

```python
def encrypt_block(self, plaintext):
    plain_state = bytes2matrix(plaintext)
    add_round_key(plain_state, self._key_matrices[0])
    for i in range(1, self.n_rounds):
        shift_rows(plain_state)
        mix_columns(plain_state)          # ← TIDAK ada sub_bytes()!
        add_round_key(plain_state, self._key_matrices[i])
    shift_rows(plain_state)
    add_round_key(plain_state, self._key_matrices[-1])
    return matrix2bytes(plain_state)
```

> **Kunci utama:** `sub_bytes` dihilangkan dari `encrypt_block`, menjadikan cipher ini sebuah **fungsi affine** atas GF(2).

---

## Tahap 1: Truncated LCG — Memulihkan `seed`

### Parameter LCG

| Parameter | Nilai |
|-----------|-------|
| `m` | `2⁶⁷` |
| `a` | `6364136223846793005` |
| `c` | `1442695040888963407` |
| Output | `state >> 34` (33 bit teratas dari 67-bit state) |

### Analisis

Server memberikan 10 nilai `leak[i] = stateᵢ >> 34`. Diketahui hanya 33 bit teratas setiap state; 34 bit bawah merupakan *unknown noise*.

Dengan notasi:

```
t[i] = leak[i] << 34     ← bagian tinggi yang diketahui
e[i] = state[i] & (2^34 - 1)   ← noise 34-bit yang tidak diketahui
```

Dari relasi LCG: `e[i+1] ≡ a·e[i] + d[i] (mod m)`

Masalah ini merupakan **Truncated LCG Problem**, diselesaikan dengan **Simultaneous Diophantine Approximation via LLL**.

### Konstruksi Lattice (n×n)

```
B = | 1   a   a²  …  a^(n-1) |  mod m
    | 0   m   0   …  0       |
    | 0   0   m   …  0       |
    | …                      |
```

Target CVP: cari `u ∈ L` terdekat ke `target[i] = 2^33 - p[i]` (particular solution digeser ke pusat `[0, 2^34)`).

### Kannan Embedding

Tambahkan baris `(target | W)` ke matriks (n+1)×(n+1), jalankan LLL:

```
M_aug = [ B    | 0 ]
        [target| W ]
```

LLL menemukan vektor pendek dengan komponen terakhir `±W`, yang secara langsung memberikan nilai `e[i]`, sehingga `seed` dapat dihitung mundur.

### Script (SageMath)

```python
def build_lattice(n, a, m, b):
    M = Matrix(ZZ, n, n)
    M[0, 0] = ZZ(2)**b
    for j in range(1, n):
        M[0, j] = pow(a, j, m)
    for j in range(1, n):
        M[j, j] = m
    return M

# Kannan embedding + LLL → dapatkan e[0] → hitung seed
x0   = t[0] + e_vec[0]
seed = int((x0 - c) * pow(a, -1, m) % m)
```

### Hasil

```
Leaks: [5619970225, 6425014657, 3241181396, ...]
→ Seed: 72150071613510701447   ✓ (diterima server)
```

---

## Tahap 2: AGCD — Memulihkan `secret_xor`

### Struktur Data

```python
def enc(plain):
    return [secrets.randbelow(plain)*plain + secrets.randbelow(2**BITS) for _ in range(5)]
```

Setiap elemen: `v[i] = q[i] · p + r[i]` di mana:

| Variabel | Ukuran |
|----------|--------|
| `p` (secret_xor) | 128-bit prima |
| `q[i]` | acak ∈ `[0, p)` ≈ 2¹²⁸ |
| `r[i]` (noise) | acak ∈ `[0, 2⁵⁰)` |
| `v[i]` | ≈ 2²⁵⁶ |

Ini adalah **Approximate GCD Problem (AGCD)**: noise 50-bit terlalu besar untuk GCD biasa.

### Analisis Bit (Kunci Keberhasilan)

Agar LLL berhasil, semua komponen vektor target harus **seimbang**:

```
Target short vector: (q₀·2^k,  q₀·r₁ - q₁·r₀,  …)
                      ↑ 2^(128+k)    ↑ ≈ 2^(128+50) = 2^178
```

Keseimbangan terjadi saat `k = BITS = 50`, menghasilkan semua komponen ≈ `2^178`.

### Konstruksi Lattice (Simultaneous Diophantine Approximation)

```
     col₀    col₁   col₂   col₃   col₄
[ 2^50,  v[1], v[2], v[3], v[4] ]  ← Row 0
[    0,  v[0],   0,    0,    0  ]  ← Row 1
[    0,    0,  v[0],   0,    0  ]  ← Row 2
[    0,    0,    0,  v[0],   0  ]  ← Row 3
[    0,    0,    0,    0,  v[0] ]  ← Row 4
```

Vektor `(q₀, -q₁, -q₂, -q₃, -q₄)` dikali matriks ini menghasilkan vektor pendek yang diinginkan.

### Recovery

```python
# Dari short vector:  short[0] = q₀ · 2^50
q0  = short[0] // (2**50)
p   = v[0] // q0          # ≈ (q₀·p) / q₀
r0  = v[0] - q0 * p
# Verifikasi: semua v[i] % p < 2^50
```

### Hasil

```
[+] RECOVERED secret_xor:
    p = 233793777343845975642582614361461386217
    bit_length = 128,  is_prime = True   ✓
```

---

## Tahap 3: Linear AES — Memulihkan Flag

### Mengapa AES Menjadi Linear?

`encrypt_block` di server **tidak memanggil `sub_bytes`**. Karena `ShiftRows` dan `MixColumns` adalah operasi **linear** atas GF(2), seluruh cipher menjadi fungsi **affine**:

```
AES(X) = L(X) ⊕ c
```

di mana:
- `L` = transformasi linear (ShiftRows + MixColumns, 10 putaran), **tidak bergantung key**
- `c` = konstanta affine = `AES(0)`, bergantung hanya pada round keys

### Sifat Affine yang Dieksploitasi

Karena `L` tidak bergantung key, dengan key = `0x00 × 16`:

```
AES₀(X) = L(X)        ← zero-key AES = pure linear map
AES₀⁻¹(Y) = L⁻¹(Y)   ← invertible karena AES tanpa SubBytes masih bijektif
```

### Langkah-Langkah Serangan

**Langkah 1** — Hitung `L(P₁)` menggunakan zero-key AES:
```python
zero_aes = LinearAES(b'\x00' * 16)
L_P1 = zero_aes.encrypt_block(b"tipsen my gaot!!")
```

**Langkah 2** — Ekstrak konstanta affine `c` dari oracle:
```
AES(P₁) = L(P₁) ⊕ c = C₁
∴  c = C₁ ⊕ L(P₁)
```

**Langkah 3** — Invert setiap blok ciphertext flag:
```
C_flag[i] = AES(flag_i ⊕ p_b)
           = L(flag_i ⊕ p_b) ⊕ c

AES⁻¹(Y) = L⁻¹(Y ⊕ c) = zero_aes.decrypt_block(Y ⊕ c)

flag_i = L⁻¹(C_flag[i] ⊕ c) ⊕ p_b
       = zero_aes.decrypt_block(C_flag[i] ⊕ c) ⊕ p_b
```

### Implementasi

```python
# Derive affine constant
c_const = xor_b(ORACLE_CIPHER, zero_aes.encrypt_block(ORACLE_PLAIN))

# Recover each block
for blk in flag_blocks:
    L_inv = zero_aes.decrypt_block(xor_b(blk, c_const))
    flag_i = xor_b(L_inv, p_bytes)
```

### Hasil

```
Block 0: NETSOS{tipsenn_n
Block 1: okiaa_kanrisshha
Block 2: aa_fahruLLL_sush
Block 3: iikannnn_simboll
Block 4: l_ma_gaooooottt_
Block 5: ma_kambingg!!!!}
Block 6: [PKCS#7 padding]

FLAG: NETSOS{tipsenn_nokiaa_kanrisshhaaa_fahruLLL_sushiikannnn_simbolll_ma_gaooooottt_ma_kambingg!!!!}
```

---

## Alur Serangan Lengkap

```
Server
  │
  ├─ leak (10 nilai LCG >> 34)
  │       │
  │       └─ [LLL / Kannan Embedding]
  │               │
  │               └──► seed ──► submit ke server ──► unlock tahap berikutnya
  │
  ├─ enc(secret_xor)  →  [v₀, v₁, v₂, v₃, v₄]
  │       │
  │       └─ [SDA / LLL, k=50]
  │               │
  │               └──► secret_xor (p) = 233793777343845975642582614361461386217
  │
  ├─ oracle: encrypt_block("tipsen my gaot!!")  →  C₁
  │       │
  │       └─ L(P₁) = AES₀(P₁) ;  c = C₁ ⊕ L(P₁)
  │
  └─ enc_flag
          │
          └─ flag_i = AES₀⁻¹(C_flag[i] ⊕ c) ⊕ p_bytes
                  │
                  └──► 🎉 FLAG
```

---

## Tools & Libraries

| Tool | Kegunaan |
|------|----------|
| SageMath | LLL lattice reduction (Tahap 1 & 2) |
| Python 3 | Implementasi serangan AES linear (Tahap 3) |

---

## Flag

```
NETSOS{tipsenn_nokiaa_kanrisshhaaa_fahruLLL_sushiikannnn_simbolll_ma_gaooooottt_ma_kambingg!!!!}
```
