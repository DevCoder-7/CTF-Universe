# Writeup: that-simple

**Event:** ARA7 Qualification  
**Category:** Cryptography  
**Tags:** DSA, Nonce Reuse, Hash Collision  
**Flag:** (diperoleh dari private key yang di-recover)

---

## 1. Analisis Kode (Reconnaissance)

Diberikan script server `chall.py` yang mengimplementasikan sistem tanda tangan digital mirip DSA dengan beberapa modifikasi kustom.

### A. Fungsi Hash Kustom (H)

```python
def H(msg):
    state = bytearray([(i * 13 + 37) % 256 for i in range(32)])
    # ... looping update state ...
    return bytes_to_long(state)
```

- **Observasi:** Fungsi hash ini deterministik dan linear (hanya operasi XOR dan pergeseran bit).
- **Kelemahan:** Karena kita bisa mengontrol input `msg`, kita bisa memanipulasi `state` agar menghasilkan nilai output apa pun yang kita mau (**Preimage Attack**).

### B. Update Nonce (k)

Dalam DSA standar, nilai `k` (nonce) harus acak untuk setiap tanda tangan. Di soal ini, `k` diperbarui menggunakan rumus:

```
k_new = (k_old^5 XOR h) + h^(k_old)  (mod q)
```

- **Observasi:** Rumus ini bergantung pada `h` (hasil hash pesan).

### C. Pembuatan Tanda Tangan

```
r = g^k  (mod p)  (mod q)
s = (h + x * r) * k^-1  (mod q)
```

Di mana `x` adalah private key (flag).

---

## 2. Celah Keamanan

### Celah 1: Zero-Hash Attack

Karena fungsi `H(msg)` memproses pesan byte demi byte, kita bisa melakukan inverse calculation byte per byte untuk membuat pesan `m` sedemikian rupa sehingga `H(m) = 0`.

### Celah 2: Nonce Relation Attack

Jika `H(m) = 0` (atau `h = 0`), perhatikan rumus update nonce:

```
k_new = (k_old^5 XOR 0) + 0^(k_old)  (mod q)
```

Karena `0^k = 0` (untuk `k ≠ 0`), maka:

```
k_new = k_old^5  (mod q)
```

Ini adalah **celah fatal** — kita tahu hubungan pasti antara nonce pertama dan kedua.

---

## 3. Strategi Penyerangan

1. **Crafting Payload** — Buat pesan khusus yang menghasilkan Hash = 0.
2. **Request Signature 1** — Kirim pesan ke server. Server memakai `k_1`. Kita dapat `(r_1, s_1)`.
3. **Request Signature 2** — Kirim pesan yang sama lagi. Server update nonce menjadi `k_2 = k_1^5`. Kita dapat `(r_2, s_2)`.
4. **Recover Private Key (x)** — Gunakan aljabar untuk mencari `k_1`, lalu hitung `x`.

---

## 4. Matematika di Balik Layar

Dengan `h = 0`, persamaan DSA menjadi:

```
s * k = x * r  (mod q)
x = s * k * r^-1  (mod q)
```

Kita punya dua sesi:
```
x = s_1 * k_1 * r_1^-1  (mod q)
x = s_2 * k_2 * r_2^-1  (mod q)
```

Menggabungkan dan substitusi `k_2 = k_1^5`:

```
s_1 * k_1 * r_1^-1 = s_2 * k_1^5 * r_2^-1  (mod q)
k_1^4 = s_1 * s_2^-1 * r_1^-1 * r_2  (mod q)
k_1 = (RHS)^(1/4)  (mod q)
```

Nilai RHS bisa kita hitung dari output server. Akar pangkat 4 dicari dengan dua kali **Tonelli-Shanks** (akar kuadrat).

Setelah `k_1` ditemukan:
```
x = s_1 * k_1 * r_1^-1  (mod q)
```

Convert `x` ke bytes → **FLAG**.

---

## 5. Script Solver

```python
import socket
import sys

def bytes_to_long(b): return int.from_bytes(b, 'big')
def long_to_bytes(n): return n.to_bytes((n.bit_length() + 7) // 8, 'big')
def egcd(a, b):
    if a == 0: return (b, 0, 1)
    else: g, y, x = egcd(b % a, a); return (g, x - (b // a) * y, y)
def modinv(a, m):
    g, x, y = egcd(a, m)
    if g != 1: raise Exception('Modular inverse does not exist')
    return x % m
def legendre_symbol(a, p):
    ls = pow(a, (p - 1) // 2, p)
    return -1 if ls == p - 1 else ls
def tonelli_shanks(n, p):
    if legendre_symbol(n, p) != 1: return None
    if n == 0: return 0
    if p == 2: return p
    if p % 4 == 3: return pow(n, (p + 1) // 4, p)
    s = p - 1; e = 0
    while s % 2 == 0: s //= 2; e += 1
    z = 2
    while legendre_symbol(z, p) != -1: z += 1
    c = pow(z, s, p); x = pow(n, (s + 1) // 2, p); t = pow(n, s, p); m = e
    while True:
        if t == 1: return x
        i = 0; temp = t
        while temp != 1 and i < m: temp = pow(temp, 2, p); i += 1
        if i == m: return None
        b = pow(c, 1 << (m - i - 1), p)
        m = i; c = (b * b) % p; t = (t * c) % p; x = (x * b) % p

def simulate_H_step(state, acc, byte_msg, i):
    acc = ((acc << 5) | (acc >> 3)) & 0xFF
    acc ^= byte_msg
    first = state[(i - 1) % 32]
    end = state[(i - 7) % 32]
    new_state_val = state[i] ^ acc ^ first ^ end
    return new_state_val, acc

def craft_zero_hash_msg():
    state = [(i * 13 + 37) % 256 for i in range(32)]
    acc = 0x55
    crafted_msg = []
    for i in range(32):
        target_acc = state[i] ^ state[(i-1)%32] ^ state[(i-7)%32]
        acc_rotated = ((acc << 5) | (acc >> 3)) & 0xFF
        byte_needed = target_acc ^ acc_rotated
        crafted_msg.append(byte_needed)
        state[i], acc = simulate_H_step(state, acc, byte_needed, i)
    return bytes(crafted_msg)

def solve():
    HOST = 'chall-ctf.ara-its.id'
    PORT = 6969
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((HOST, PORT))

    def recv_until(char_list):
        buf = b''
        while True:
            chunk = s.recv(1)
            if not chunk: break
            buf += chunk
            for c in char_list:
                if c in buf: return buf
        return buf

    payload = craft_zero_hash_msg()

    # Signature 1
    recv_until([b':', b'?'])
    s.sendall(b'y\n')
    recv_until([b':'])
    s.sendall(payload.hex().encode() + b'\n')
    resp1 = recv_until([b'\n']).decode().strip()
    r1, s1, h1, q = eval(resp1[resp1.find('('):resp1.find(')')+1])

    # Signature 2
    recv_until([b':', b'?'])
    s.sendall(b'y\n')
    recv_until([b':'])
    s.sendall(payload.hex().encode() + b'\n')
    resp2 = recv_until([b'\n']).decode().strip()
    r2, s2, h2, _ = eval(resp2[resp2.find('('):resp2.find(')')+1])

    # Recover k_1 and x
    inv_s2 = modinv(s2, q)
    inv_r1 = modinv(r1, q)
    val = (s1 * inv_s2 * inv_r1 * r2) % q

    roots_lvl1 = [tonelli_shanks(val, q), q - tonelli_shanks(val, q)]
    k_candidates = []
    for r in roots_lvl1:
        if r:
            res = tonelli_shanks(r, q)
            if res: k_candidates.extend([res, q-res])

    for k in k_candidates:
        x = (s1 * k * inv_r1) % q
        try:
            print(f"Flag: {long_to_bytes(x).decode()}")
            break
        except: pass

    s.close()

if __name__ == "__main__":
    solve()
```
