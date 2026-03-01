# Writeup — Harmony (Cryptography)

**Event:** NETSOS Open Recruitment  
**Author:** fele  
**Category:** Cryptography  
**Flag:** `NETSOS{Tw0_LFSRs_4re_st1ll_l1ne4r_d0nt_b3_f00led}`

---

## Deskripsi

> The bits dance in harmony. Can you hear the rhythm?

Challenge ini menyediakan dua file:
- `task.py` — script enkripsi beserta ciphertext-nya
- `utils.py` — implementasi generator keystream berbasis dua buah LFSR

---

## Analisis Source Code

### `utils.py`

```python
class LFSR:
    def __init__(self, state, nbits, mask):
        self.nbits = nbits
        self.state = state & ((1 << self.nbits) - 1)
        self.mask = mask

    def clock(self, steps=1):
        for _ in range(steps):
            res = self.state & self.mask
            bit = sum([(res >> i) & 1 for i in range(self.nbits)]) & 1
            self.state = ((self.state << 1) ^ bit) & ((1 << self.nbits) - 1)
        return bit


class Gen:
    def __init__(self, seed):
        self.nbits = 128
        s1 = (seed >> 64) & ((1 << 64) - 1)
        s2 = seed & ((1 << 64) - 1)
        self.lfsr1 = LFSR(s1, 64, 0xb4bcd35c738901cf)
        self.lfsr2 = LFSR(s2, 64, 0xe4d972fb67531acd)

    def __next__(self):
        out = 0
        for _ in range(8):
            b1 = self.lfsr1.clock(1337)
            b2 = self.lfsr2.clock(2023)
            out = (out << 1) ^ (b1 ^ b2)
        return out
```

`Gen` menggunakan dua **64-bit LFSR** dengan mask/tap berbeda:
- `lfsr1` diklock **1337 langkah** per bit output
- `lfsr2` diklock **2023 langkah** per bit output
- Setiap output byte dihasilkan dari **8 bit**, masing-masing bit = XOR dari satu bit lfsr1 dan satu bit lfsr2

### `task.py`

```python
seed = int(os.urandom(16).hex(), 16)
gen = utils.Gen(seed)
msg = b'iniFlagnya: ' + flag
enc = bytes(m ^ next(gen) for m in msg).hex()
print(enc)

# 2ac6d421a1c75eb35f0dce8592755ef2bafa6ee74cfb83f89604575652d05782861807532c39ceec03ebdee490761e47c3b3fb7b56a1782b2f90f033d8
```

Pesan yang dienkripsi adalah `b'iniFlagnya: ' + flag`, di-XOR byte per byte dengan output `Gen`.

---

## Kelemahan: Linearitas LFSR

Meskipun menggunakan **dua** LFSR, sistem ini tetap **linear** atas GF(2). Operasi XOR dari dua LFSR menghasilkan kombinasi linear dari state awal keduanya — sehingga seluruh keystream dapat direpresentasikan sebagai perkalian matriks atas GF(2).

### Known Plaintext Attack

Kita mengetahui bahwa pesan dimulai dengan `b'iniFlagnya: NETSOS{'` (19 byte = 152 bit) dari prefix tetap dan format flag yang diketahui. Dengan XOR ciphertext dan plaintext yang diketahui, kita mendapat **152 bit keystream**.

---

## Solusi

### Langkah 1 — Representasi LFSR sebagai Matriks GF(2)

Satu langkah clock LFSR dapat direpresentasikan sebagai perkalian matriks **M ∈ GF(2)^{64×64}**:

```
new_state[0]   = XOR(old_state[i]) untuk i di mana mask[i] = 1   (feedback bit)
new_state[i]   = old_state[i-1]    untuk i = 1..63               (shift)
```

Sehingga **clock(N steps) = M^N** (perpangkatan matriks atas GF(2)).

### Langkah 2 — Bit Output sebagai Fungsi Linear

Bit output dari `clock(steps)` adalah **bit terakhir** yang dihasilkan, yaitu:

```
output_bit = M[0] · (M^(steps-1) · s0)
```

di mana `s0` adalah state awal LFSR.

Untuk bit ke-k dari keystream:
- LFSR1 telah di-clock `1337 × k` langkah dari state awal
- LFSR2 telah di-clock `2023 × k` langkah dari state awal

Masing-masing bit output = fungsi linear dari `s1` dan `s2`.

### Langkah 3 — Sistem Persamaan Linear GF(2)

Gabungkan vektor baris untuk setiap bit keystream yang diketahui:

```
[ out_row1 @ P1^k  |  out_row2 @ P2^k ] · [s1; s2] = ks_bit[k]
```

Ini membentuk sistem **152 persamaan** dengan **128 variabel** (64 bit state s1 + 64 bit state s2), dan diselesaikan dengan **eliminasi Gauss atas GF(2)**.

### Langkah 4 — Dekripsi

Setelah state awal `s1` dan `s2` ditemukan, rekonstruksi generator dan XOR dengan ciphertext untuk mendapat plaintext.

---

## Exploit Script

```python
import numpy as np
import sys
sys.path.insert(0, '.')
import utils

nbits = 64
mask1 = 0xb4bcd35c738901cf
mask2 = 0xe4d972fb67531acd

def get_transition_matrix(mask, nbits=64):
    M = np.zeros((nbits, nbits), dtype=np.uint8)
    for i in range(1, nbits):
        M[i][i-1] = 1
    for j in range(nbits):
        if (mask >> j) & 1:
            M[0][j] = 1
    return M

def mat_mul_gf2(A, B):
    return (A @ B) % 2

def mat_pow_gf2(M, n, nbits=64):
    result = np.eye(nbits, dtype=np.uint8)
    base = M.copy()
    while n > 0:
        if n & 1:
            result = mat_mul_gf2(result, base)
        base = mat_mul_gf2(base, base)
        n >>= 1
    return result

def gf2_solve(A, b):
    n_eq, n_var = A.shape
    Ab = np.hstack([A, b.reshape(-1,1)]).astype(np.uint8)
    pivot_cols = []
    row = 0
    for col in range(n_var):
        found = -1
        for r in range(row, n_eq):
            if Ab[r, col]:
                found = r
                break
        if found == -1:
            continue
        Ab[[row, found]] = Ab[[found, row]]
        for r in range(n_eq):
            if r != row and Ab[r, col]:
                Ab[r] = (Ab[r] + Ab[row]) % 2
        pivot_cols.append((row, col))
        row += 1
    sol = np.zeros(n_var, dtype=np.uint8)
    for (r, c) in pivot_cols:
        sol[c] = Ab[r, n_var]
    return sol

enc = bytes.fromhex(
    '2ac6d421a1c75eb35f0dce8592755ef2bafa6ee74cfb83f89604575652d0578'
    '2861807532c39ceec03ebdee490761e47c3b3fb7b56a1782b2f90f033d8'
)
known = b'iniFlagnya: NETSOS{'
keystream_known = bytes(e ^ p for e, p in zip(enc, known))
ks_bits = []
for byte in keystream_known:
    for i in range(7, -1, -1):
        ks_bits.append((byte >> i) & 1)

M1 = get_transition_matrix(mask1)
M2 = get_transition_matrix(mask2)
P1 = mat_pow_gf2(M1, 1337)
P2 = mat_pow_gf2(M2, 2023)

out_row1 = M1[0] @ mat_pow_gf2(M1, 1336) % 2
out_row2 = M2[0] @ mat_pow_gf2(M2, 2022) % 2

A_rows, b_vals = [], []
P1_pow = np.eye(nbits, dtype=np.uint8)
P2_pow = np.eye(nbits, dtype=np.uint8)
for k in range(len(ks_bits)):
    row = np.concatenate([out_row1 @ P1_pow % 2, out_row2 @ P2_pow % 2])
    A_rows.append(row)
    b_vals.append(ks_bits[k])
    P1_pow = mat_mul_gf2(P1_pow, P1)
    P2_pow = mat_mul_gf2(P2_pow, P2)

A = np.array(A_rows, dtype=np.uint8)
b = np.array(b_vals, dtype=np.uint8)
sol = gf2_solve(A, b)

s1_bits = sol[:64]
s2_bits = sol[64:]
s1 = sum(int(s1_bits[i]) << i for i in range(64))
s2 = sum(int(s2_bits[i]) << i for i in range(64))

seed = (s1 << 64) | s2
gen = utils.Gen(seed)
decrypted = bytes(e ^ next(gen) for e in enc)
print(decrypted)
# b'iniFlagnya: NETSOS{Tw0_LFSRs_4re_st1ll_l1ne4r_d0nt_b3_f00led}'
```

---

## Output

```
b'iniFlagnya: NETSOS{Tw0_LFSRs_4re_st1ll_l1ne4r_d0nt_b3_f00led}'
```

---

## Flag

```
NETSOS{Tw0_LFSRs_4re_st1ll_l1ne4r_d0nt_b3_f00led}
```

---

## Kesimpulan

Challenge ini mengajarkan bahwa **menggabungkan dua LFSR dengan XOR tidak menambah keamanan** secara kriptografis, karena XOR dari dua fungsi linear tetaplah linear. Dengan known-plaintext attack dan aljabar linear atas GF(2), state awal kedua LFSR dapat dipulihkan hanya dari 152 bit keystream, memungkinkan dekripsi penuh pesan.

> *"Two LFSRs are still linear — don't be fooled."*
