# Writeup: AES Stream Oracle Attack

**Event:** ARA7 Qualification  
**Category:** Cryptography  
**Solved by:** farelboston (Farel Boston Corinthians)
**Flag:** `ARA7{a9055fb26edf78ccd6368858b118ff29de264480b211d2812ff111c49d7080aa}`

---

## Deskripsi Challenge

Challenge menyediakan service dengan 2 fitur utama:

**1. Test Encryption**
- User dapat memasukkan IV (16 byte) dan Msg (16 byte)
- Server mengenkripsi dengan fungsi:

```python
def stream_encrypt(iv, p, key):
    cipher = AES.new(key, AES.MODE_ECB)
    keystream = cipher.encrypt(iv)
    return xor(p, keystream)
```

**2. Get Flag**
- Flag dipadding PKCS#7
- IV random 16 byte
- Enkripsi dilakukan per blok dengan mode chaining:

```python
for each block:
    ct_block = stream_encrypt(iv, block, KEY)
    iv = ct_block
```

---

## Analisis Kerentanan

Fungsi enkripsi:
```
CT = P XOR AES_K(IV)
```

Pada menu Get Flag, IV diperbarui seperti ini:
```
IV_next = CT_current
```

Sehingga skemanya menjadi:
```
CT_i = P_i XOR AES_K(CT_{i-1})
```

Ini bukan CBC, tetapi lebih mirip **CFB mode custom**.

### Titik Lemah: Oracle Vulnerability

Menu 1 memperbolehkan kita memilih IV dan Plaintext bebas. Jika kita memilih:

```
Msg = 00...00  (16 byte nol)
```

Maka:
```
CT = 0 XOR AES_K(IV) = AES_K(IV)
```

**Server adalah AES encryption oracle** — kita bisa mendapatkan output AES terhadap input apa pun.

---

## Strategi Attack

Diketahui:
```
CT_i = P_i XOR AES_K(CT_{i-1})
```

Untuk mendapatkan plaintext:
```
P_i = CT_i XOR AES_K(CT_{i-1})
```

Kita bisa menghitung `AES_K(CT_{i-1})` dengan cara:
- Gunakan menu 1
- Set `IV = CT_{i-1}`
- Set `Msg = 00...00`

---

## Langkah Eksploitasi

**Step 1 — Ambil Encrypted Flag**
```
Gunakan menu 2 → Server mengembalikan IV dan Ciphertext
```

**Step 2 — Bagi Ciphertext per 16 Byte**
```
C0 = IV
C1 | C2 | C3 | ...
```

**Step 3 — Untuk setiap blok:**
1. Kirim ke menu 1: `IV = previous_block`, `Msg = 00...00`
2. Dapatkan: `Keystream = AES_K(previous_block)`
3. Hitung: `Plaintext = Ciphertext XOR Keystream`
4. Update: `previous_block = current_block`

**Step 4 — Remove Padding**
Hapus PKCS#7 padding setelah semua blok terdekripsi.

---

## Mengapa Attack Ini Berhasil?

- AES digunakan dalam ECB (tidak aman untuk stream mode)
- Keystream dapat di-query langsung
- Tidak ada autentikasi
- Tidak ada pembatasan query

---

## Flag

```
ARA7{a9055fb26edf78ccd6368858b118ff29de264480b211d2812ff111c49d7080aa}
```

---

## Kesimpulan

Kerentanan terjadi karena AES-ECB digunakan sebagai primitive dan XOR stream construction yang salah desain, memberikan attacker akses ke `AES_K(x)` untuk input arbitrer.

> **"Don't design your own crypto."**

Mode kriptografi yang benar seperti AES-CBC, AES-GCM, atau AES-CTR tidak akan vulnerable seperti ini jika diimplementasikan dengan benar.
