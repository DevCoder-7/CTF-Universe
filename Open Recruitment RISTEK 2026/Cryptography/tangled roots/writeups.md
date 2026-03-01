# CTF Writeup: Tangled Roots

**Kategori:** Cryptography  
**Flag:** `NETSOS{qu4dr4t1c_l4tt1c3_g0044TTT!}`

---

## Deskripsi Challenge

Challenge ini memberikan sebuah skrip Python (`chall.py`) dan file output (`output.txt`). Skrip tersebut meng-enkripsi flag menggunakan skema berikut:

- `p` — bilangan prima 1024-bit (digenerate sekali)
- `m` — flag yang dikonversi ke integer (`m < 2^300`)
- Untuk setiap dari 40 sampel, dibangkitkan:
  - `a`, `b` — koefisien acak 700-bit
  - `e` — error acak 64-bit
  - `h = (a * m² + b * m + e) mod p`

Output yang diberikan: `p` dan 40 tuple `(a, b, h)`.  
**Tujuan:** Pulihkan nilai `m` (flag).

---

## Tahap 1 — Analisis dan Linearisasi

Persamaan dasar setiap sampel:

```
h_i ≡ a_i * m² + b_i * m + e_i  (mod p)
```

Karena `m` adalah satu nilai yang sama di semua sampel, kita lakukan **linearisasi** dengan memperkenalkan dua variabel baru:

```
X = m²    (besar: X < 2^600)
Y = m     (besar: Y < 2^300)
```

Sehingga persamaan menjadi linear:

```
h_i ≡ a_i * X + b_i * Y + e_i  (mod p)
```

Atau dalam bentuk integer:

```
a_i * X + b_i * Y + e_i - h_i = k_i * p    (untuk suatu integer k_i)
```

Batas-batas variabel:

| Variabel | Batas |
|----------|-------|
| `X` | `< 2^600` |
| `Y` | `< 2^300` |
| `e_i` | `< 2^64` ← sangat kecil dibanding `p ≈ 2^1024` |
| `k_i` | `< 2^276` (estimasi dari ukuran relatif `a*X` terhadap `p`) |

**Kunci insight:** Error `e_i` sangat kecil (`2^64`) relatif terhadap `p` (`2^1024`). Ini membuka peluang serangan berbasis **lattice** (kisi).

---

## Tahap 2 — Konstruksi Lattice

Kita menyusun matriks basis lattice berukuran `(n+3) × (n+3)`, dengan `n = 20` sampel.

**Struktur matriks M:**

```
Kolom  :  e_0   e_1  ...  e_{n-1}  |  X*WX    Y*WY    1*WC
-----------+----------------------------------------------------------
Baris 0 :   p    0   ...    0      |   0       0       0       ← k_0
Baris 1 :   0    p   ...    0      |   0       0       0       ← k_1
    ...
Baris n : -a_0  -a_1  ... -a_{n-1} |  WX       0       0       ← X
Baris n+1: -b_0 -b_1  ... -b_{n-1} |   0      WY       0       ← Y
Baris n+2:  h_0  h_1  ...  h_{n-1} |   0       0      WC       ← konstanta
```

Dengan dimensi matriks = **23 × 23** (`n=20`).

**Faktor Scaling** (integer, menghindari floating point):

```
WX = 1
WY = 2^300   → sehingga Y * WY ~ 2^600, seimbang dengan X * WX
WC = 2^536   → sehingga 1 * WC ~ 2^536
```

> **Alasan scaling:** LLL bekerja optimal ketika semua komponen vektor target memiliki magnitude yang sebanding. Tanpa scaling, komponen Y (kecil) akan "tenggelam" dan tidak ditemukan LLL.

**Vektor target** yang diharapkan ada dalam lattice:

```
v = (e_0, e_1, ..., e_{n-1}, X, Y*2^300, 2^536)
```

Setiap komponen berorde `~2^600`, sehingga `v` adalah vektor yang "pendek" dalam lattice ini dan dapat ditemukan oleh algoritma LLL.

**Mengapa vektor ini ada dalam lattice?**  
Vektor `v` dapat dinyatakan sebagai kombinasi integer dari baris-baris `M`:

```
v = k_0*baris_0 + ... + k_{n-1}*baris_{n-1} + X*baris_n + Y*baris_{n+1} + 1*baris_{n+2}
```

Ini terbukti dari persamaan: `e_i = h_i - a_i*X - b_i*Y + k_i*p`

---

## Tahap 3 — LLL Reduction dan Ekstraksi Flag

Setelah matriks `M` dikonstruksi, kita jalankan algoritma **LLL** (Lenstra–Lenstra–Lovász):

```python
L = M.LLL()
```

LLL mereduksi basis lattice sehingga vektor-vektor terpendek muncul di baris-baris pertama matriks hasil. Karena `v` adalah vektor pendek alami dalam lattice ini, LLL berhasil menemukannya.

**Identifikasi Baris Solusi:**  
Baris yang mengandung solusi memiliki ciri khas:
- Kolom ke-`(n+2)` bernilai `±WC = ±2^536` (komponen konstanta)
- Komponen `e_i` di kolom `0..n-1` semua `< 2^64`

**Ekstraksi Flag:**  
Dari baris solusi yang ditemukan (baris `#0` setelah reduksi):

```
sign   = +1  (karena komponen WC positif)
Y_val  = sign * row[n+1] // WY    → ini adalah nilai m (flag sebagai integer)
X_val  = sign * row[n]   // WX    → ini adalah m²
```

**Verifikasi:**
- ✓ `X == Y²` — terbukti (konsistensi matematis)
- ✓ Semua `e_i < 2^64` — terbukti (error dalam batas)
- ✓ `e_0` check valid — terbukti (verifikasi kriptografi dengan sampel)
- ✓ `Y bits = 279` — masuk akal untuk flag ~35 karakter

**Konversi ke flag:**

```
int_to_bytes(Y_val) → b'NETSOS{qu4dr4t1c_l4tt1c3_g0044TTT!}'
```

---

## Script Solusi (SageMath)

```python
n   = 20
WX  = 1
WY  = 2^300
WC  = 2^536
dim = n + 3

M = Matrix(ZZ, dim, dim)

for i in range(n):
    M[i, i] = p                  # blok p diagonal

for i in range(n):
    ai, bi, hi = samples[i]
    M[n,   i] = -ai
    M[n+1, i] = -bi
    M[n+2, i] =  hi

M[n,   n  ] = WX
M[n+1, n+1] = WY
M[n+2, n+2] = WC

L = M.LLL()

# Scan baris hasil
for row in L:
    c = int(row[n + 2])
    if abs(c) != int(WC):
        continue
    sign  = 1 if c > 0 else -1
    Y_val = sign * int(row[n+1]) // int(WY)
    # konversi ke string → FLAG
```

---

## Ringkasan Serangan

| | |
|---|---|
| **Jenis serangan** | Lattice-based attack (LLL reduction) |
| **Tools** | SageMath (built-in LLL via `M.LLL()`) |
| **Dimensi lattice** | 23 × 23 |
| **Sampel dipakai** | 20 dari 40 sampel tersedia |
| **Kompleksitas** | O(n⁶ · log(max_entry)) — selesai dalam ~60 detik |

**Konsep kunci:**
1. **Linearisasi** — ubah persamaan kuadratik dalam `m` menjadi linear (`X=m²`, `Y=m`)
2. **Formulasi lattice** — susun persamaan modular sebagai lattice problem
3. **Scaling** — normalisasi besar komponen agar LLL efektif
4. **LLL reduction** — temukan vektor pendek yang mengandung `X` dan `Y`
5. **Ekstraksi** — decode `Y` kembali menjadi flag

**Kelemahan desain challenge:**
- Error `e_i` terlalu kecil (`2^64`) dibanding `p` (`2^1024`) → rasio `2^960`
- Persamaan kuadratik dalam `m` dapat dilinearisasi menjadi sistem LWE-like
- Banyak sampel tersedia (40) melebihi kebutuhan minimum

---

## Flag

```
NETSOS{qu4dr4t1c_l4tt1c3_g0044TTT!}
```
