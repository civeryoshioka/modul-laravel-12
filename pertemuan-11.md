# Pertemuan 11 — UAS

> **Sebelumnya:** Pertemuan 1–10 membangun `app-perpustakaan` dari nol sampai lengkap — MVC & setup project, routing, controller & validasi, Blade & master layout, migration & CRUD nyata, UTS, Eloquent relationships, autentikasi & middleware, REST API, sampai konsumsi API di Dashboard/Laporan plus Seeder & Factory.
> **Pertemuan ini:** Bukan pertemuan praktikum baru. Evaluasi akhir semester dalam bentuk **demo aplikasi live** dan tanya jawab konseptual — tidak ada kode baru yang dikerjakan di sesi ini.

---

## Format Ujian

| Item | Detail |
|---|---|
| Bentuk | Demo aplikasi live (browser + Postman) + tanya jawab lisan |
| Sifat | Individu atau kelompok (mengikuti pembagian kelas) |
| Durasi | ±15 menit per mahasiswa/kelompok |
| Cakupan | Seluruh materi Pertemuan 1 s.d. 10 |
| Yang dinilai | Fungsionalitas aplikasi (live), kualitas kode, repository GitHub, dan pemahaman konsep (tanya jawab) |

UAS ini **tidak menambah fitur baru** ke `app-perpustakaan`. Yang didemokan adalah kondisi project di akhir Pertemuan 10, ditambah Tugas mandiri dari pertemuan-pertemuan sebelumnya kalau sudah sempat dikerjakan (lihat [Sebelum Demo](#sebelum-demo)).

---

## Cakupan Materi

Mahasiswa dianggap sudah menguasai seluruh topik berikut sebelum demo:

1. **Pertemuan 1** — Framework vs PHP native, arsitektur MVC, struktur direktori Laravel 12
2. **Pertemuan 2** — Routing, Resource Route, Request Lifecycle, posisi Middleware
3. **Pertemuan 3** — Controller, class `Request`, validasi input, Form Request
4. **Pertemuan 4** — Blade templating, layout inheritance, partial view
5. **Pertemuan 5** — Migration sebagai version control schema, Eloquent ORM, CRUD data nyata
6. **Pertemuan 7** — Eloquent Relationships (`hasMany`/`belongsTo`), N+1 Query Problem, eager loading
7. **Pertemuan 8** — Autentikasi session-based, Middleware custom, role check
8. **Pertemuan 9** — REST API, JSON, API Resource, HTTP Status Code, Postman
9. **Pertemuan 10** — Laravel HTTP Client (consume API internal), Seeder & Factory

Selain kesembilan topik di atas, dasar **Git & GitHub** (lihat [materi tambahan](./tambahan/git-github-dasar.md)) juga dianggap sudah dikuasai — bukan topik pertemuan tersendiri, tapi dipakai sejak Pertemuan 1 dan ikut dinilai lewat Aspek 5 (Repository GitHub) di rubrik.

---

## Sebelum Demo

Cek dulu poin-poin berikut sebelum jadwal demo, supaya sesi tidak terpotong oleh kendala teknis yang sebenarnya bisa dicegah:

- [ ] **Dua server harus jalan bersamaan** — server utama (trafik browser) dan server kedua untuk `INTERNAL_API_URL`/`services.internal_api.base_url`. Kalau hanya satu yang jalan, halaman **Dashboard** dan **Laporan Peminjaman** akan tampil kosong/gagal (bukan berarti fitur tidak dikerjakan — lihat [Konsep: Deadlock Server](./pertemuan-10.md) kalau lupa alasannya).
- [ ] `php artisan migrate:fresh --seed` sudah dijalankan sebelum demo, supaya data yang ditampilkan konsisten (bukan sisa data uji coba manual yang berantakan).
- [ ] Branch `dev` berisi seluruh checkpoint `[P-1]` s.d. `[P-10]` (Tugas P10: merge `dev` ke `main`).
- [ ] `DEMO.md` (skenario demo tertulis) dan `README.md` (instruksi menjalankan project, termasuk kebutuhan 2 server) sudah dibuat/diupdate — ini bagian dari Tugas P10.
- [ ] Kredensial login (admin dan petugas) sudah disiapkan dan diketahui hafal, tidak perlu buka `.env`/database saat demo berlangsung.

**Catatan soal Tugas mandiri lintas pertemuan:** beberapa fitur berikut sifatnya Tugas mandiri (bukan wajib di praktikum inti) — kalau belum sempat dikerjakan, **bukan berarti gagal UAS**, tapi tidak dapat poin tambahan di rubrik terkait:
- Fitur "kembalikan buku" versi **web** (Tugas P7) — kalau belum dikerjakan, method `LoanController@kembalikan` masih stub. Mahasiswa **boleh** mendemokan pengembalian buku lewat endpoint API (`PUT /api/loans/{id}/kembalikan` di Postman, sudah berfungsi penuh sejak P9) sebagai gantinya di skenario demo langkah 8.
- Badge status peminjaman berwarna (Tugas P7)
- Fitur search nama anggota (Tugas P5)
- Halaman profil petugas + ganti password (Tugas P8)
- Dokumentasi tabel endpoint API (Tugas P9)

---

## Skenario Demo

Ikuti urutan berikut saat demo di depan dosen penguji. Tiap langkah diamati langsung (bukan dijelaskan lewat slide) — siapkan browser dan Postman terbuka sebelum mulai.

### 1. Login sebagai petugas/admin
Buka halaman `/login`, masuk dengan salah satu akun (admin atau petugas). Tunjukkan redirect ke Dashboard setelah login berhasil, dan nama + role user tampil di navbar.

### 2. Tampilkan dashboard statistik
Tunjukkan 3 kartu statistik (Total Buku, Total Anggota, Peminjaman Aktif) di halaman Dashboard, dan jelaskan singkat bahwa angka ini diambil lewat `Http::get()` ke `GET /api/stats`, bukan query langsung ke database.

### 3. Tambah kategori baru
Buka `/categories`, tambah 1 kategori baru lewat form, tunjukkan data tersimpan dan muncul di index dengan flash message sukses.

### 4. Tambah buku dengan kategori
Buka `/books`, tambah 1 buku baru dengan memilih kategori dari dropdown (termasuk kategori yang baru dibuat di langkah 3), tunjukkan data tersimpan dengan nama kategori tampil benar (bukan `category_id` mentah).

### 5. Tambah anggota baru
Buka `/members`, tambah 1 anggota baru, tunjukkan data tersimpan dan validasi (misal `nim`/`email` unik) berfungsi kalau diuji dengan data duplikat.

### 6. Buat transaksi peminjaman
Buka `/loans/create`, buat 1 transaksi peminjaman baru memilih anggota dan buku (termasuk anggota/buku yang baru ditambahkan). Tunjukkan `user_id` petugas terisi otomatis dari sesi login (bukan input manual).

### 7. Lihat daftar peminjaman aktif
Buka `/loans`, tunjukkan transaksi baru tampil dengan status `dipinjam`, lengkap dengan nama anggota, petugas, dan daftar buku (hasil eager loading relasi, bukan ID mentah).

### 8. Proses pengembalian buku
Proses pengembalian transaksi yang baru dibuat di langkah 6 — lewat halaman web kalau Tugas P7 sudah dikerjakan, atau lewat `PUT /api/loans/{id}/kembalikan` di Postman kalau belum (lihat [Sebelum Demo](#sebelum-demo)). Tunjukkan status berubah jadi `dikembalikan`.

### 9. Tampilkan halaman laporan
Buka `/loans/report`, tunjukkan seluruh transaksi (termasuk yang baru saja diproses) tampil lengkap, dan jelaskan singkat bahwa data ini diambil dari `GET /api/loans`, bukan query Eloquent langsung dari Controller web.

### 10. Demo API di Postman
Jalankan langsung di Postman:
- `GET /api/books` — tunjukkan response JSON terstruktur (format `BookResource`), termasuk pagination.
- `GET /api/stats` — tunjukkan response `total_buku`, `total_anggota`, `peminjaman_aktif` sesuai data terkini.

### 11. Tanya jawab konseptual
Dosen memilih beberapa pertanyaan dari [Daftar Pertanyaan Tanya Jawab](#daftar-pertanyaan-tanya-jawab) sesuai sisa waktu.

---

## Daftar Pertanyaan Tanya Jawab

Dosen memilih 3–5 pertanyaan sesuai sisa waktu per mahasiswa/kelompok. Penilaian fokus pada pemahaman *mengapa*, bukan hafalan istilah — jawaban yang hanya mendefinisikan tanpa mengaitkan ke alasan/masalah yang diselesaikan dinilai kurang lengkap.

1. Kalau `app-perpustakaan` dibangun tanpa framework, masalah apa yang muncul seiring project bertambah besar? Kaitkan dengan MVC.
2. Jelaskan urutan lengkap request-response saat petugas submit form tambah buku, mulai dari browser sampai halaman index tampil lagi dengan data baru. Sebutkan posisi Middleware dalam alur ini.
3. Mengapa validasi input tetap wajib di sisi server meskipun form HTML sudah punya `required`?
4. Apa masalah yang diselesaikan Migration dibanding mengelola struktur tabel manual, dalam konteks kerja tim?
5. Jelaskan N+1 Query Problem memakai contoh nyata dari relasi `Loan` → `Member`/`Book` di project ini, dan bagaimana eager loading menyelesaikannya.
6. Apa beda relasi yang didefinisikan di Model Eloquent (`hasMany`/`belongsTo`) dengan foreign key di migration? Kenapa keduanya tetap dibutuhkan?
7. Jelaskan beda autentikasi dan otorisasi. Di mana masing-masing diterapkan pada project ini (`AuthController` vs `CheckAdminRole`)?
8. Kenapa REST API di project ini mengembalikan JSON, bukan HTML? Siapa saja konsumen yang diuntungkan pola ini selain browser?
9. Apa fungsi API Resource (`BookResource`, dst) dibanding langsung `return response()->json($book)`?
10. Jelaskan alasan teknis kenapa Dashboard dan Laporan Peminjaman butuh 2 server berbeda saat development di `php artisan serve`, dan apakah masalah ini akan muncul juga di server produksi sungguhan.
11. Apa fungsi Seeder dan Factory, dan kenapa keduanya dipisah (bukan digabung jadi satu file)?
12. Kalau harus menambah 1 tabel baru ke aplikasi ini (misalnya `publishers`), sebutkan urutan lengkap file yang perlu dibuat/diubah dari migration sampai tampil di Blade.
13. Kenapa project ini memakai 2 branch (`dev` dan `main`) alih-alih commit langsung ke satu branch saja? Jelaskan juga kenapa commit checkpoint bertahap per pertemuan (`[P-1]`, `[P-2]`, dst) lebih baik dibanding satu commit besar di akhir semester.

---

## Kriteria Penilaian

| Aspek | Bobot |
|---|---|
| Fungsionalitas CRUD (books, categories, members, loans) | 30% |
| Autentikasi dan middleware berfungsi | 20% |
| REST API endpoint berfungsi dan response terstruktur | 20% |
| Kualitas kode (konvensi, konsistensi) | 15% |
| Repository GitHub (commit history, branch) | 15% |

### Rincian Aspek 1 — Fungsionalitas CRUD (30%)

| Indikator | Poin |
|---|---|
| CRUD `books` (create, read index+show, update, delete) dengan data nyata dan kategori tampil benar | 8 |
| CRUD `categories` (create, read, update, delete) dengan data nyata | 6 |
| CRUD `members` (create, read, update, delete) dengan data nyata dan validasi unik (`nim`/`email`) | 8 |
| CRUD `loans` (create dengan relasi anggota/buku, read index dengan relasi lengkap) + pengembalian buku berfungsi (web atau API) | 8 |

### Rincian Aspek 2 — Autentikasi & Middleware (20%)

| Indikator | Poin |
|---|---|
| Login/logout berfungsi, validasi kredensial salah menampilkan pesan error | 6 |
| Middleware `auth` melindungi seluruh route CRUD (akses tanpa login redirect ke `/login`) | 6 |
| `CheckAdminRole` berfungsi (route khusus admin ditolak untuk petugas biasa) | 5 |
| Info nama dan role user login tampil benar di navbar | 3 |

### Rincian Aspek 3 — REST API (20%)

| Indikator | Poin |
|---|---|
| Endpoint books (GET all, GET by id, POST, PUT, DELETE) berfungsi penuh | 8 |
| Response konsisten pakai API Resource (`BookResource`, `MemberResource`, `LoanResource`) | 5 |
| Demo langsung di Postman (`GET /api/books`, `GET /api/stats`) berhasil live saat ujian, bukan sekadar screenshot lama | 4 |
| HTTP Status Code sesuai konteks (200 sukses, 201 created, 404 tidak ditemukan, 422 validasi gagal) | 3 |

### Rincian Aspek 4 — Kualitas Kode (15%)

| Indikator | Poin |
|---|---|
| Konvensi penamaan konsisten (Model, Controller, route name, folder view sesuai `master-outline.md` bagian G) | 5 |
| Logika tidak menumpuk di Controller (validasi lewat Form Request, tidak ada query kompleks di route closure) | 4 |
| Eager loading dipakai pada relasi yang ditampilkan berulang (tidak ada N+1 Query Problem baru) | 3 |
| Blade konsisten pakai master layout dan partial (`partials/navbar.blade.php`, `partials/alert.blade.php`), tidak ada HTML berdiri sendiri yang seharusnya sudah direfactor | 3 |

### Rincian Aspek 5 — Repository GitHub (15%)

| Indikator | Poin |
|---|---|
| Branch `dev` aktif dengan commit checkpoint rapi `[P-1]` s.d. `[P-10]`, pesan sesuai isi perubahan aslinya | 5 |
| Branch `main` sudah menerima merge dari `dev` (Tugas P10) — bukan `main` yang masih tertinggal di `[P-1]` | 4 |
| `README.md` diupdate berisi instruksi menjalankan project, termasuk kebutuhan 2 server untuk Dashboard/Laporan | 3 |
| `DEMO.md` berisi skenario demo tertulis, cukup jelas untuk diikuti orang lain tanpa penjelasan tambahan | 3 |

### Skala Konversi Akhir

Nilai akhir UAS = jumlah poin yang diperoleh di kelima aspek (total maksimum 100).

| Total Poin | Predikat |
|---|---|
| 85 – 100 | A |
| 70 – 84 | B |
| 55 – 69 | C |
| 40 – 54 | D |
| < 40 | E |

Nilai akhir mata kuliah mengikuti kebijakan pembobotan UTS/UAS/tugas harian masing-masing dosen pengampu — rubrik ini hanya mengatur skor komponen UAS.

---

## Catatan untuk Dosen Penilai

- **Wajib cek 2 server jalan sebelum menilai Dashboard/Laporan.** Kalau mahasiswa hanya menjalankan 1 server dan kedua halaman ini tampil kosong/error, ini bukan otomatis kekurangan Aspek 1 — minta mahasiswa menjalankan server kedua (`INTERNAL_API_URL`) dulu sebelum menyimpulkan fitur tidak berfungsi. Kalau setelah 2 server jalan tetap gagal, baru dinilai sebagai kekurangan nyata.
- **Fitur "kembalikan buku" versi web adalah Tugas mandiri (P7), bukan wajib.** Kalau method `LoanController@kembalikan` masih stub dan mahasiswa mendemokan lewat endpoint API sebagai gantinya, tetap beri poin penuh di indikator terkait Aspek 1 selama endpoint API-nya terbukti berfungsi live.
- **Badge status berwarna, search anggota, halaman profil/ganti password, dan dokumentasi tabel API semuanya Tugas mandiri** dari pertemuan-pertemuan sebelumnya (P5/P7/P8/P9) — tidak masuk indikator wajib rubrik ini. Kalau sudah dikerjakan, boleh jadi pertimbangan nilai plus di Aspek 4 (Kualitas Kode) secara kualitatif, tapi bukan pengurang nilai kalau belum sempat dikerjakan.
- **REST API di project ini sengaja tanpa proteksi token/session** (keputusan cakupan P9, sesuai `master-outline.md`) — jangan mengurangi nilai Aspek 3 karena endpoint API bisa diakses tanpa login. Ini bukan kelalaian keamanan yang perlu dinilai, karena topik proteksi API (misalnya Sanctum) memang di luar cakupan modul ini.
- Kalau repository tidak bisa diakses sama sekali (link mati, private tanpa akses) saat jadwal demo, Aspek 3 dan 5 (35% dari total) dinilai 0 — demo tetap dijalankan dari kode lokal mahasiswa untuk menilai Aspek 1, 2, dan 4.

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-10.md) | [Daftar Isi](./README.md)*
