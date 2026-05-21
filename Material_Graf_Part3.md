# Graf

**Matematika Diskret 2 — Fakultas Ilmu Komputer Universitas Indonesia**

*Slide acknowledgment: Gatot Wahyudi, Adila A. Krisnadhi, Kurniawati Azizah*

---

## Agenda

- Isomorfisme
- Bipartite dan Matching
- Lintasan Terpendek

---

# 1. Isomorfisme

## Definisi

> **Dua graf sederhana** $G_1 = (V_1, E_1)$ dan $G_2 = (V_2, E_2)$ dikatakan **isomorfik** jika terdapat fungsi bijektif $f$ dari $V_1$ ke $V_2$ sehingga dua vertex $v_1$ dan $v_2$ bertetangga di $G_1$ jika dan hanya jika vertex $f(v_1)$ dan $f(v_2)$ bertetangga di $G_2$.
>
> Jika tidak ada isomorfisme antara $G_1$ dan $G_2$, maka $G_1$ dan $G_2$ dikatakan **nonisomorfik**.

**Sifat:** jika $G_1 = (V_1, E_1)$ dan $G_2 = (V_2, E_2)$ isomorfik, maka:

- $|V_1| = |V_2|$
- $|E_1| = |E_2|$
- Untuk setiap $n \in \mathbb{N}$:

  $$
  |\{u \in V_1 \mid \deg(u) = n\}| = |\{v \in V_2 \mid \deg(v) = n\}|
  $$

> ⚠️ Konversnya belum tentu berlaku.

Untuk mengecek apakah suatu fungsi dari $V_1$ ke $V_2$ adalah isomorfisme, kita dapat menggunakan **matriks ketetanggaan**. Matriks ketetanggaan kedua graf harus serupa.

---

## Menentukan Isomorfisme

- Untuk mengetahui dua buah graf sederhana bersifat **isomorfik**, perlu dicari **fungsi bijektif** yang memetakan $V_1$ ke $V_2$ sedemikian hingga ketetanggaan di kedua graf tetap terjaga.
- Jika kedua graf memiliki $n$ jumlah simpul, maka terdapat $n!$ kemungkinan fungsi bijektif yang dapat dibangun.
- Untuk menentukan dua graf **nonisomorfik**, kita dapat memeriksa adanya pelanggaran terhadap **invarian graf**.

### Invarian Graf

**Invarian graf** adalah properti pada graf yang menjaga sifat isomorfisme graf tersebut.

Contoh invarian graf:

- Jumlah simpul
- Jumlah sisi
- Jumlah loop
- Jumlah simpul dengan derajat tertentu

---

## Contoh Penentuan Isomorfisme

### Soal

Apakah dua graf $G = (V, E)$ dan $H = (W, F)$ isomorf apabila:

- $V = \{v_1, v_2, v_3, v_4, v_5\}$
- $E = \{(v_1,v_2), (v_2,v_3), (v_3,v_4), (v_4,v_1), (v_1,v_5)\}$
- $W = \{w_1, w_2, w_3, w_4, w_5\}$
- $F = \{(w_1,w_3), (w_1,w_4), (w_2,w_5), (w_4,w_2), (w_5,w_1)\}$

### Jawab

#### Langkah 1 — Periksa $|V| = |W|$ dan $|E| = |F|$

- $|V| = 5$, $|W| = 5$ ✓
- $|E| = 5$, $|F| = 5$ ✓

Karena keduanya terpenuhi, lanjutkan ke pemeriksaan lebih lanjut.

#### Langkah 2 — Tentukan derajat masing-masing vertex di $V$ dan $W$

| Vertex pada $G$ | Derajat | Vertex pada $H$ | Derajat |
|---|---:|---|---:|
| $v_1$ | 3 | $w_1$ | 3 |
| $v_2$ | 2 | $w_2$ | 2 |
| $v_3$ | 2 | $w_3$ | 1 |
| $v_4$ | 2 | $w_4$ | 2 |
| $v_5$ | 1 | $w_5$ | 2 |

Urutan derajat:

- $G: \{3, 2, 2, 2, 1\}$
- $H: \{3, 2, 2, 2, 1\}$ ✓

#### Langkah 3 — Cari fungsi bijektif

Coba fungsi bijektif berikut:

| Vertex | Pemetaan |
|---|---|
| $f(v_1)$ | $w_1$ |
| $f(v_2)$ | $w_4$ |
| $f(v_3)$ | $w_2$ |
| $f(v_4)$ | $w_5$ |
| $f(v_5)$ | $w_3$ |

Verifikasi keterpeliharaan keterikatan:

| Sisi di $E$ | Hasil pemetaan | Ada di $F$? |
|---|---|---:|
| $(v_1, v_2) \in E$ | $(f(v_1), f(v_2)) = (w_1, w_4)$ | ✓ |
| $(v_2, v_3) \in E$ | $(f(v_2), f(v_3)) = (w_4, w_2)$ | ✓ |
| $(v_3, v_4) \in E$ | $(f(v_3), f(v_4)) = (w_2, w_5)$ | ✓ |
| $(v_4, v_1) \in E$ | $(f(v_4), f(v_1)) = (w_5, w_1)$ | ✓ |
| $(v_1, v_5) \in E$ | $(f(v_1), f(v_5)) = (w_1, w_3)$ | ✓ |

**Kesimpulan:** $G$ dan $H$ isomorf. ✓

---

## Latihan — Tentukan Apakah Pasangan Graf Berikut Isomorfik

### Latihan 1

Graf 1, di kiri, memiliki simpul $\{a, b, c, d\}$ dengan sisi membentuk pola silang seperti dua busur berlawanan. Graf 2, di kanan, memiliki simpul $\{1, 2, 3, 4\}$ dengan sisi membentuk pola silang serupa.

```text
Graf 1:           Graf 2:
  a   b             1   4
  |\ /|             |\ /|
  | X |             | X |
  |/ \|             |/ \|
  d   c             2   3
```

### Latihan 2

```text
Graf G (kiri):             Graf H (kanan):
  a ──────── b               s ──────── t
  |\ f      |                |   w    x |
  | \   \   |                |    \  /  |
  h  \   \  |                |  z  \/   |
  |   \   \ |                |     /\   |
  d ──────── c               v ──────── u
```

Graf berisi 8 simpul masing-masing:

- $G: \{a, b, c, d, e, f, g, h\}$
- $H: \{s, t, u, v, w, x, y, z\}$

### Latihan 3

- Graf $G$: 6 simpul, 7 sisi, struktur seperti $X$ dengan sisi vertikal tambahan.
- Graf $H$: 6 simpul, 7 sisi, struktur $Z$ dengan segitiga di bawah.

### Latihan 4

- Graf $G$: 5 simpul membentuk segi lima tidak beraturan dengan 2 diagonal.
- Graf $H$: 5 simpul membentuk struktur segitiga dengan 2 sisi tambahan.

### Latihan 5

- Graf $G$: 6 simpul, terdapat diagonal silang membentuk $K$ dengan sekitar 9 sisi.
- Graf $H$: 6 simpul, persegi dengan diagonal dan simpul atas dengan sekitar 9 sisi.

### Latihan 6

- Graf $G$: 6 simpul, segi lima dan 1 simpul tengah dengan beberapa diagonal.
- Graf $H$: 6 simpul, persegi dan 2 simpul dalam dengan beberapa diagonal.

---

## Contoh Pemanfaatan Isomorfisme

### Dalam Bidang Bioinformatics

- Graf molecular dapat memodelkan senyawa kimia, dengan **atom** sebagai vertex dan **ikatan kimia antaratom** sebagai sisi.
- Ketika suatu senyawa kimia baru dapat disintesa, senyawa tersebut dapat dibandingkan dengan basis data senyawa yang sudah pernah ada.

### Dalam Bidang Electronics

- Sirkuit elektronik dapat dimodelkan dengan graf, dengan **komponen elektronik** sebagai vertex dan **hubungan antarkomponen** sebagai sisi.
- Isomorfisme dapat digunakan untuk menentukan apakah sirkuit yang dibuat sesuai dengan model awal.
- Isomorfisme juga dapat digunakan untuk menentukan apakah produk sirkuit perusahaan lain melanggar paten.

---

# 2. Bipartite dan Matching

## Graf Bipartite

### Definisi

> **Sebuah graf** $G = (V, E)$ disebut **graf bipartite** jika dan hanya jika $V$ dapat dipartisi menjadi dua himpunan tidak kosong, $V_1$ dan $V_2$, yang saling lepas sedemikian sehingga untuk setiap sisi $e \in E$, salah satu *endpoint* $e$ ada di dalam $V_1$ dan *endpoint* lainnya ada di dalam $V_2$.

### Contoh

Graf $G = (V, E)$ dengan:

- $V = \{a, b, c, d, e, f, g\}$
- $E = \{(a,c), (a,e), (a,f), (a,g), (b,c), (b,e), (b,g), (d,c), (d,e), (d,f)\}$

Partisi:

- $U = \{a, b, d\}$
- $W = \{c, e, f, g\}$

Gambar ulang dalam bentuk bipartite:

```text
  a    d    b
  |\ / |\ /|
  | X  | X |
  |/ \ |/ \|
  g    e    c    f
```

Setiap sisi menghubungkan simpul di $U$ dengan simpul di $W$.

---

## Graf Bipartite Lengkap

### Definisi

> **Sebuah graf** $G = (V, E)$ disebut **graf bipartite lengkap**, $K_{m,n}$, apabila $V$ merupakan gabungan dari dua subset, yaitu:
>
> - $U$ dengan $m \ne 0$ unsur
> - $W$ dengan $n \ne 0$ unsur
>
> yang saling lepas, sedemikian sehingga sisi $(u, w) \in E \Longleftrightarrow u \in U$ dan $w \in W$.

### Contoh $K_{2,3}$

```text
a       b
 /|\     /|\
/ | \   / | \
c  e  d
```

Setiap simpul pada $\{a,b\}$ terhubung ke semua simpul pada $\{c,e,d\}$.

### Contoh $K_{3,3}$

```text
f       a       b
  |\ \  / |\ \  /|
  |  \/   |  \/  |
  |  /\   |  /\  |
  | /  \  | /  \ |
  c       e       d
```

Setiap simpul di $\{f,a,b\}$ terhubung ke semua simpul di $\{c,e,d\}$.

---

## Menentukan Bipartite

### Teorema

> **Sebuah graf sederhana adalah bipartite jika dan hanya jika** setiap simpulnya bisa diberi satu dari dua warna berbeda sedemikian sehingga tidak ada simpul yang saling bertetangga yang memiliki warna yang sama.

Dengan kata lain:

> Graf adalah bipartite $\Longleftrightarrow$ graf dapat di-2-warnai / tidak mengandung siklus ganjil.

---

## Matching

### Definisi

> Diketahui $G = (V, E)$ adalah sebuah graf tidak berarah. Sebuah **matching** adalah himpunan sisi $M \subseteq E$ sedemikian sehingga tidak ada dua sisi di $M$ yang bertumpuan pada vertex yang sama.

### Jenis-Jenis Matching

| Jenis | Definisi |
|---|---|
| **Maximal** | $M$ bukan *proper subset* dari matching lainnya pada $G$. |
| **Maximum** | Kardinalitas $M$ lebih besar atau sama dengan kardinalitas matching lainnya pada $G$. |
| **Perfect** | Semua vertex pada $G$ memiliki pasangan. |
| **Complete** dari $V_1$ ke $V_2$ | Jika $G$ adalah graf bipartite dengan $V_1$ dan $V_2$ sebagai partisinya, setiap vertex di $V_1$ memiliki pasangan di $V_2$. |

### Contoh Matching

Graf memiliki simpul $\{a,b,c,d,e,f,g,h\}$ dengan edges membentuk dua baris:

```text
  f ──── g ──── h
  |  \  / \  / |
  |   \/   \/  |
  |   /\   /\  |
  |  /  \ /  \ |
  d ──── (mid) ── e
  |              |
  a ──── b ──── c
```

**Perfect dan maximum matching berukuran 4:**

$$
\{(f,d), (g,b), (h,e), (a,c)\}
$$

Semua vertex berpasangan.

**Maximal matching berukuran 3:**

$$
\{(d,a), (h,e), (f,g)\}
$$

Tidak dapat ditambah sisi lain tanpa konflik.

**Maximal matching berukuran 2:**

$$
\{(g,h), (d,a)\}
$$

Matching ini maximal, tetapi bukan maximum.

> ⚠️ Setiap matching **maximum** pasti **maximal**, tetapi tidak sebaliknya.

---

# 3. Lintasan Terpendek

## Contoh Aplikasi

Kasus sistem penerbangan:

- Setiap **kota** merupakan *vertex* dari graf.
- **Jalur penerbangan** antarkota merupakan *edge* dari graf.

Pertanyaan yang dapat dijawab:

- Berapa **waktu minimal** yang diperlukan untuk mengunjungi kota tertentu?
- Berapa **ongkos minimal** yang diperlukan untuk mengunjungi kota tertentu?

Untuk menerapkan teori graf pada permasalahan seperti ini, dibutuhkan pengertian tentang **graf berbobot**.

---

## Contoh Graf Berbobot: Jaringan Penerbangan

### Berdasarkan Waktu Terbang (*Flight Times*)

```text
San Francisco ──4:05──────────── Chicago ──2:10──── Boston
       \──2:55─ Chicago    Chicago ──1:50── New York ──0:50── Boston
       ──2:20── Denver ────────────────────────────────────
       ──1:15── Los Angeles                          ──1:55── New York
                Los Angeles ──3:50── (mid) ──1:40── Atlanta
                              Denver ──2:10── (mid)
                                      Atlanta ──1:30── Miami
                                      Miami ──2:45── New York
```

### Berdasarkan Tarif (*Fares*)

```text
San Francisco ──$129─ Chicago ──$79─ Boston
              ──$99── Chicago ──$59── New York ──$39── Boston
              ──$89── Denver
              ──$39── Los Angeles ──$89── (mid)
              Denver ──$69── (mid) ──$99── Atlanta
              Los Angeles ──$129── (mid) ──$79── Atlanta
                              Atlanta ──$69── Miami
                              Miami ──$69── New York
```

---

## Definisi Graf Berbobot

> **Sebuah graf** $G = (V, E)$ disebut **graf berbobot** (*weighted graph*) apabila terdapat fungsi bobot bernilai real $W$ pada himpunan $E$:
>
> $$
> W: E \to \mathbb{R}
> $$
>
> - Nilai $W(e)$ disebut **bobot** untuk sisi $e$, untuk setiap $e \in E$.
> - Graf berbobot dinyatakan sebagai $G = (V, E, W)$.

### Contoh Graf Berbobot

| Aplikasi | $V$ | $E$ | $W$ |
|---|---|---|---|
| Jaringan penerbangan | Himpunan kota | Himpunan rute penerbangan antarkota | Ongkos penerbangan tiap rute |
| Jaringan komputer | Himpunan komputer | Himpunan jalur kabel langsung antar dua komputer | Jarak / ongkos / waktu |

---

## Makna Lintasan Terpendek

Permasalahan lintasan terpendek dapat berarti salah satu dari menentukan:

- **Jalur termurah**
- **Jalur terdekat**
- **Jalur tercepat**

Makna dari lintasan terpendek bergantung pada **makna fungsi bobot** yang terdapat pada graf yang dibentuk.

---

## Algoritma Dijkstra

### Deskripsi

Algoritma Dijkstra merupakan salah satu cara yang dapat digunakan untuk menyelesaikan permasalahan menemukan **lintasan terpendek** pada suatu graf berbobot.

### Inti Algoritma Dijkstra

Misalkan ingin dicari lintasan terpendek $P(a, z)$ dari vertex $a \in V$ ke vertex $z \in V$:

1. Cari dahulu lintasan terpendek $P(a, b)$ dari vertex $a$ ke suatu vertex $b$ di $V$.
2. Selanjutnya, cari lintasan terpendek $P(a, c)$ dari vertex $a$ ke suatu vertex $c$ di $V$.
3. Proses dilanjutkan terus-menerus dan berhenti ketika lintasan terpendek $P(a, z)$ diperoleh.

### Pseudocode Algoritma Dijkstra

```text
procedure Dijkstra(G: weighted connected simple graph,
                   with all weights positive)
{G has vertices a = v₀, v₁, ..., vₙ = z and lengths w(vᵢ, vⱼ),
 where w(vᵢ, vⱼ) = ∞ if {vᵢ, vⱼ} is not an edge in G}

for i := 1 to n
    L(vᵢ) := ∞
L(a) := 0
S := ∅
{the labels are now initialized so that the label of a is 0 and all
 other labels are ∞, and S is the empty set}

while z ∉ S
    u := a vertex not in S with L(u) minimal
    S := S ∪ {u}
    for all vertices v not in S
        if L(u) + w(u,v) < L(v) then L(v) := L(u) + w(u,v)
        {this adds a vertex to S with minimal label and updates the
         labels of vertices not in S}

return L(z)
{L(z) = length of a shortest path from a to z}
```

### Ilustrasi Langkah-Langkah Dijkstra

Graf contoh memiliki simpul $a, b, c, d, e, z$:

```text
b ∞        d ∞
       / \        / \
      4   5      6   2
     /     \    /     \
  0 a ──1── (m) ──2──  z ∞
     \     /    \     /
      2   10     3   (sisi lainnya)
       \ /        \ /
        c ∞        e ∞
```

**Bobot sisi:**

- $a-b = 4$, $a-(mid) = 1$, $a-c = 2$
- $b-d = 5$, $(mid)-d = 6$, $(mid)-e = 10$ atau lainnya
- $c-e = 3$, $d-z = 2$, $e-z$ mengikuti graf

**Iterasi algoritma:**

| Langkah | $S$ | $L(a)$ | $L(b)$ | $L(c)$ | $L(d)$ | $L(e)$ | $L(z)$ |
|---|---|---:|---|---|---|---|---|
| (a) Inisialisasi | $\{a\}$ | 0 | ∞ | ∞ | ∞ | ∞ | ∞ |
| (b) Proses $a$ | $\{a\}$ | 0 | 4(a) | 2(a) | ∞ | ∞ | ∞ |
| (c) Proses $c$ | $\{a,c\}$ | 0 | 3(a,c) | 2(a) | 10(a,c) | 12(a,c) | ∞ |
| (d) Proses $b$ | $\{a,c,b\}$ | 0 | 3(a,c) | 2(a) | 8(a,c,b) | 12(a,c) | ∞ |
| (e) Proses $d$ | $\{a,c,b,d\}$ | 0 | 3(a,c) | 2(a) | 8(a,c,b) | 10(a,c,b,d) | 14(a,c,b,d) |
| (f) Proses $e$ | $\{a,c,b,d,e\}$ | 0 | 3(a,c) | 2(a) | 8(a,c,b) | 10(a,c,b,d) | 13(a,c,b,d,e) |
| (g) Proses $z$ | $\{a,c,b,d,e,z\}$ | 0 | 3(a,c) | 2(a) | 8(a,c,b) | 10(a,c,b,d) | **13(a,c,b,d,e)** |

**Lintasan terpendek dari $a$ ke $z$ adalah 13**, melalui jalur:

$$
a \to c \to b \to d \to e \to z
$$

---

## Latihan — Lintasan Terpendek

Carilah lintasan terpendek pada graf berbobot berikut ini.

```text
b ────5──── d
          / \          / \
         2   \        /   2
        /     \      /     \
       a        2──1        z
        \      /      \     /
         3    /        \   4
          \  /          \ /
           c ────5──── e
```

**Daftar sisi dan bobot:**

| Sisi | Bobot |
|---|---:|
| $a-b$ | 2 |
| $a-c$ | 3 |
| $b-d$ | 5 |
| $b-e$ | 2 |
| $d-e$ | 1 |
| $d-z$ | 2 |
| $e-z$ | 4 |
| $c-e$ | 5 |

Gunakan Algoritma Dijkstra untuk menentukan lintasan terpendek dari $a$ ke $z$.

---

*Selamat belajar…*

---

> **Sumber:** Slide Matematika Diskret 2, Fakultas Ilmu Komputer Universitas Indonesia  
> *Slide acknowledgment: Gatot Wahyudi, Adila A. Krisnadhi, Kurniawati Azizah*
