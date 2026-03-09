# Writeup: know the trick

**Event:** ARA7 Qualification  
**Category:** Cryptography  
**Solved by:** demtcsre (Ahmad Rizki Daffaa)  
**Flag:** `ARA7{C0N9R4T5_y0u_F1ND_m3_lcmethod_fl4gggg}`

---

## Deskripsi Challenge

Connect ke server mendapatkan info berikut:

```
I have a sequence generator. Can you predict the next state?

PARAMETERS:
p = 115792089237316195423570985008687907853269984665640564039457584007913129639917
a = 100720434724302814904053626510495455391082506492327747842141262032751684401854
b = 514631507721405306298073637848375664226723355710112857507800679889911926255
n = 40

OBSERVATIONS:
h0 = x0 >> n = 968852466726078703819955219989690981844639388292366248354398...
h1 = x1 >> n = 449668252294223273482129166825505907550445430057509954353406...
h2 = x2 >> n = 147594155351770872376683351341621974313247776495462865465626...

OBJECTIVE:
Find x3 (the full 256-bit value)
Submit x3 as a decimal integer.
```

---

## Analisis

Ini adalah **Truncated Linear Congruential Generator (LCG)** problem.

Diberikan parameter `p`, `α`, `β` dan high bits (`h_i`) dari tiga state berurutan `x_0, x_1, x_2`. Tujuannya adalah memulihkan hidden lower `n = 40` bits (`l_0`) dari state pertama untuk merekonstruksi sequence dan menemukan `x_3`.

Relasi antar state:
```
x_{i+1} = α * x_i + β  (mod p)
```

Substitusi `x_i = 2^n * h_i + l_i`:
```
2^n * h_1 + l_1 = α(2^n * h_0 + l_0) + β  (mod p)
```

Ini memungkinkan kita untuk mengekspresikan `l_1` (dan selanjutnya `l_2`) sebagai fungsi linear dari `l_0` modulo `p`.

### The Attack Method: Lattice Basis Reduction (LLL)

Karena `n = 40` relatif kecil dibanding 256-bit modulus, masalah ini dapat dimodelkan sebagai **Hidden Number Problem (HNP)** dan diselesaikan menggunakan **Lattice Basis Reduction (LLL)**.

Kita dapat mengkonstruksi lattice basis di mana target vector yang mengandung `(l_0, l_2, 2^n)` adalah shortest vector, yang dapat ditemukan LLL.

---

## Script Solver

```python
from fractions import Fraction

# LLL Implementation
def dot(v, u): return sum(vi * ui for vi, ui in zip(v, u))
def sub(v, u): return [vi - ui for vi, ui in zip(v, u)]
def mul(scalar, v): return [scalar * vi for vi in v]

def gram_schmidt(basis):
    ortho = []
    mu = [[Fraction(0) for _ in range(len(basis))] for _ in range(len(basis))]
    for i in range(len(basis)):
        b_i = basis[i]
        for j in range(i):
            mu[i][j] = Fraction(dot(basis[i], ortho[j]), dot(ortho[j], ortho[j]))
            b_i = sub(b_i, mul(mu[i][j], ortho[j]))
        ortho.append(b_i)
    mu[i][i] = Fraction(1)
    return ortho, mu

def lll(basis, delta=Fraction(3, 4)):
    n = len(basis)
    ortho, mu = gram_schmidt(basis)
    k = 1
    while k < n:
        for j in range(k - 1, -1, -1):
            if abs(mu[k][j]) > Fraction(1, 2):
                q = int(round(mu[k][j]))
                basis[k] = sub(basis[k], mul(q, basis[j]))
                ortho, mu = gram_schmidt(basis)
        if dot(ortho[k], ortho[k]) >= (delta - mu[k][k-1]**2) * dot(ortho[k-1], ortho[k-1]):
            k += 1
        else:
            basis[k], basis[k-1] = basis[k-1], basis[k]
            ortho, mu = gram_schmidt(basis)
            k = max(k - 1, 1)
    return basis

# Challenge Parameters
p = 115792089237316195423570985008687907853269984665640564039457584007913129639917
a = 100720434724302814904053626510495455391082506492327747842141262032751684401854
b = 514631507721405306298073637848375664226723355710112857507800679889911926255
n_bits = 40
scale = 2**n_bits

h0 = 96885246672607870381995521998969098184463938829236624835439804720
h1 = 44966825229422327348212916682550590755044543005750995435340658608
h2 = 14759415535177087237668335134162197431324777649546286546562649086

# Solver
print("Solving for x3...")

K1 = (a * (h0 << n_bits) + b - (h1 << n_bits)) % p
K2 = (a * (h1 << n_bits) + b - (h2 << n_bits)) % p
A_const = (a * a) % p
K_const = (a * K1 + K2) % p

basis = [
    [1, A_const, 0],
    [0, p, 0],
    [0, K_const, scale]
]

reduced = lll(basis)

x3 = None
for row in reduced:
    if abs(row[2]) == scale:
        l0 = abs(row[0])
        print(f"Recovered l0: {l0}")
        x0 = (h0 << n_bits) + l0
        x1 = (a * x0 + b) % p
        x2 = (a * x1 + b) % p
        x3 = (a * x2 + b) % p
        break

if x3 is not None:
    print("-" * 30)
    print(f"x3 = {x3}")
    print("-" * 30)
```

**Output:**
```
Solving for x3...
Recovered l0: 39212098888
------------------------------
x3 = 56954351784517349849389922314885909780783728077285047640119127900717674790677
------------------------------
```

---

## Dekripsi Flag

Setelah mendapatkan `x3` yang benar (`[+] CORRECT!`), server memberikan encrypted flag:

```
IV:         9a6fae360627140a80bd9358d90a13fe
Ciphertext: f9032029d2820d06be0f09cf6ae9f5f5ea462d9d03b821fd37e4b1eb75c6c85dbb0b677fda19160d3d0621
Key:        Linear Congruential Method
```

Script dekripsi menggunakan `SHA256(bytes(x3))` sebagai key AES-CTR:

```python
import binascii, hashlib
from Crypto.Cipher import AES
from Crypto.Util import Counter

x3 = 56954351784517349849389922314885909780783728077285047640119127900717674790677
iv_hex = "9a6fae360627140a80bd9358d90a13fe"
ct_hex = "f9032029d2820d06be0f09cf6ae9f5f5ea462d9d03b821fd37e4b1eb75c6c85dbb0b677fda19160d3d0621"

key_bytes = hashlib.sha256(int(x3).to_bytes(32, 'big')).digest()
iv_int = int(iv_hex, 16)

ctr = Counter.new(128, initial_value=iv_int)
cipher = AES.new(key_bytes, AES.MODE_CTR, counter=ctr)
pt = cipher.decrypt(binascii.unhexlify(ct_hex))
print(pt.decode('utf-8'))
```

---

## Flag

```
ARA7{C0N9R4T5_y0u_F1ND_m3_lcmethod_fl4gggg}
```
