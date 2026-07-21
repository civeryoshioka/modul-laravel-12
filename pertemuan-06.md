# Pertemuan 6 — UTS

> **Sebelumnya:** Pertemuan 1–5 membangun `app-perpustakaan` dari nol — setup project & MVC, routing resource, Controller & validasi, Blade master layout, sampai migration & CRUD `books`/`categories` dengan data nyata dari database.
> **Pertemuan ini:** Bukan pertemuan praktikum baru. Evaluasi tengah semester atas seluruh capaian Pertemuan 1–5, dalam bentuk laporan tertulis dan review repository GitHub — tidak ada kode baru yang dikerjakan di sesi ini.

---

## Format Ujian

| Item | Detail |
|---|---|
| Bentuk | Laporan tertulis (PDF) + link GitHub repository |
| Sifat | Individu |
| Cakupan | Materi Pertemuan 1 s.d. 5 |
| Yang dinilai | Repository (kode nyata) dan laporan (dokumentasi + pemahaman konsep) |

UTS ini **tidak menambah fitur baru** ke `app-perpustakaan`. Yang dinilai adalah kondisi project di akhir Pertemuan 5, ditambah Tugas mandiri CRUD `members` kalau sudah sempat dikerjakan (lihat catatan di bagian [Sebelum Mengumpulkan](#sebelum-mengumpulkan)).

---

## Cakupan Materi

Mahasiswa dianggap sudah menguasai kelima topik berikut sebelum mengerjakan laporan:

1. **Pertemuan 1** — Framework vs PHP native, arsitektur MVC, struktur direktori Laravel 12
2. **Pertemuan 2** — Routing, Resource Route, Request Lifecycle, posisi Middleware
3. **Pertemuan 3** — Controller, class `Request`, validasi input, Form Request
4. **Pertemuan 4** — Blade templating, layout inheritance (`@extends`/`@section`/`@yield`), partial view
5. **Pertemuan 5** — Migration sebagai version control schema, Eloquent ORM, CRUD dengan data nyata, mass assignment protection (`$fillable`)

---

## Syarat Repository

Sebelum menulis laporan, pastikan repository GitHub memenuhi semua syarat berikut. Ini adalah syarat **minimum kelulusan** — repository yang tidak memenuhi salah satu poin di bawah akan mengurangi nilai signifikan di aspek Repository dan CRUD (lihat rubrik).

- [ ] Branch `dev` ada dan aktif sebagai branch pengembangan (branch `main` boleh tertinggal, tidak masalah untuk UTS)
- [ ] Minimal 5 commit rapi di branch `dev`, mengikuti checkpoint P-1 s.d. P-5 (`[P-1] ...` s.d. `[P-5] ...`)
- [ ] CRUD `books` berfungsi penuh: create, read (index + show), update, delete — dengan data yang benar-benar tersimpan di database (bukan array/dummy)
- [ ] CRUD `categories` berfungsi penuh: create, read, update, delete — dengan data nyata
- [ ] Halaman `index` (daftar) `books` dan `categories` memakai master layout (`@extends('layouts.app')`) — ini yang jadi praktikum wajib Pertemuan 4, bukan Tugas mandiri
- [ ] Tidak ada error PHP (halaman *whoops*, 500, atau error validasi yang tidak tertangkap) saat mengakses seluruh halaman `books` dan `categories`

Cara cepat memeriksa daftar commit sebelum mengumpulkan:

```bash
git log --oneline dev
```

---

## Struktur Laporan

Laporan dikumpulkan dalam format PDF, dengan urutan bagian sebagai berikut:

### 1. Halaman Sampul
Nama, NIM, kelas, judul "UTS Pemrograman Framework — Sistem Perpustakaan Digital Kampus".

### 2. Link Repository
Cantumkan link repository GitHub dan pastikan branch `dev` bisa diakses (repository publik, atau *invite* dosen sebagai kolaborator kalau privat).

### 3. Dokumentasi Fitur (Screenshot)
Untuk **setiap** operasi CRUD `books` dan `categories`, sertakan screenshot dengan urutan: tampilan sebelum aksi → form/aksi yang dilakukan → hasil setelah aksi (termasuk flash message sukses kalau ada). Total minimal:
- 4 screenshot untuk `books` (index dengan pagination, create, edit, hasil delete)
- 4 screenshot untuk `categories` (index dengan pagination, create, edit, hasil delete)
- 1 screenshot pesan error validasi (submit form kosong)

### 4. Penjelasan Konsep
Jawab kelima pertanyaan di bagian [Pertanyaan Konsep](#pertanyaan-konsep-wajib-dijawab) dengan kalimat sendiri, masing-masing minimal 3–5 kalimat.

### 5. Kendala & Solusi
Ceritakan minimal satu kendala teknis nyata yang ditemui selama Pertemuan 1–5 (error, bug, kebingungan konsep) dan bagaimana cara menyelesaikannya. Boleh mengambil dari pengalaman sendiri, tidak harus sama dengan contoh di modul.

---

## Pertanyaan Konsep (Wajib Dijawab)

Pertanyaan ini menguji pemahaman *mengapa*, bukan sekadar *bagaimana* — sesuai prinsip modul ini. Jawaban yang hanya mendefinisikan istilah tanpa mengaitkan ke alasan/masalah yang diselesaikan akan dinilai kurang lengkap.

1. Kalau aplikasi ini dibangun tanpa framework (PHP native murni), masalah apa yang akan muncul seiring project ini bertambah besar? Kaitkan jawabanmu dengan konsep MVC.
2. Jelaskan urutan lengkap yang terjadi ketika seorang petugas mengklik tombol "Simpan" di form tambah buku, mulai dari browser mengirim request sampai halaman index buku tampil kembali dengan data baru. Sebutkan komponen apa saja yang dilalui.
3. Mengapa validasi input tetap diperlukan di sisi server (Controller/Form Request) meskipun form HTML sudah punya atribut `required` di sisi browser?
4. Apa masalah yang diselesaikan oleh Migration dibanding mengelola struktur tabel secara manual lewat phpMyAdmin/query manual? Jelaskan dalam konteks kerja tim.
5. Apa itu *mass assignment vulnerability*, dan bagaimana `$fillable` di Eloquent mencegahnya?

---

## Sebelum Mengumpulkan

Tugas mandiri CRUD `members` dari Pertemuan 5 **tidak wajib** untuk syarat kelulusan repository UTS di atas — syarat minimum hanya mensyaratkan `books` dan `categories`. Tapi kalau sudah sempat dikerjakan, sertakan juga screenshot-nya di laporan sebagai nilai tambah (lihat rubrik, poin bonus di aspek Repository).

---

## Kriteria Penilaian

| Aspek | Bobot |
|---|---|
| Repository GitHub (branch dev, commit history rapi) | 25% |
| CRUD Books berfungsi penuh | 25% |
| CRUD Categories berfungsi penuh | 20% |
| Tampilan Blade menggunakan master layout | 15% |
| Laporan (dokumentasi, screenshot, penjelasan konsep) | 15% |

### Rincian Aspek 1 — Repository GitHub (25%)

| Indikator | Poin |
|---|---|
| Branch `dev` ada dan jadi branch aktif pengembangan | 8 |
| Minimal 5 commit rapi, pesan commit mengikuti format `[P-X] deskripsi` dan sesuai isi perubahan aslinya | 12 |
| Riwayat commit mencerminkan progres bertahap (bukan satu commit besar "upload semua") | 5 |
| **Bonus** — CRUD `members` (Tugas P5) sudah dikerjakan dan ter-commit | +3 (maks. tetap 25) |

### Rincian Aspek 2 — CRUD Books (25%)

| Indikator | Poin |
|---|---|
| Create — data tersimpan nyata di database, redirect + flash message sukses | 6 |
| Read — index tampil dengan pagination, show menampilkan detail benar | 6 |
| Update — form edit ter-*prefill* data lama, perubahan tersimpan benar | 6 |
| Delete — data terhapus dari database dan hilang dari index | 4 |
| Validasi — pesan error tampil saat submit data tidak valid/kosong | 3 |

### Rincian Aspek 3 — CRUD Categories (20%)

| Indikator | Poin |
|---|---|
| Create — data tersimpan nyata di database, redirect + flash message sukses | 5 |
| Read — index tampil dengan pagination | 5 |
| Update — form edit ter-*prefill* data lama, perubahan tersimpan benar | 5 |
| Delete — data terhapus dari database dan hilang dari index | 3 |
| Validasi — pesan error tampil saat submit data tidak valid/kosong | 2 |

### Rincian Aspek 4 — Master Layout (15%)

Yang jadi praktikum wajib Pertemuan 4 hanya refactor `index.blade.php` kedua resource ke master layout — refactor `create.blade.php` sengaja dijadikan Tugas mandiri P4 (opsional per mahasiswa), dan `edit.blade.php`/`show.blade.php` **tidak pernah diminta** direfactor sama sekali di modul manapun (P4 atau P5). Karena itu rincian poin di bawah membedakan yang wajib dari yang bernilai tambah.

| Indikator | Poin |
|---|---|
| `books/index.blade.php` memakai `@extends('layouts.app')` (wajib) | 4 |
| `categories/index.blade.php` memakai `@extends('layouts.app')` (wajib) | 4 |
| Navbar konsisten tampil di halaman `index` kedua resource, termasuk *active state* menu yang benar | 3 |
| Flash message pakai partial (`partials/alert.blade.php`), bukan ditulis ulang per halaman | 2 |
| **Nilai tambah** — `books/create.blade.php` dan/atau `categories/create.blade.php` sudah direfactor ke master layout (Tugas mandiri P4) | +1 per file, maks. +2 |
| **Nilai tambah** — `edit.blade.php`/`show.blade.php` salah satu resource sudah direfactor ke master layout meski tidak pernah diminta | +1 per file, maks. +2 (maks. total tetap 15) |

### Rincian Aspek 5 — Laporan (15%)

| Indikator | Poin |
|---|---|
| Screenshot lengkap sesuai daftar minimum di [Struktur Laporan](#3-dokumentasi-fitur-screenshot) | 5 |
| Jawaban 5 pertanyaan konsep — benar dan menjelaskan *alasan*, bukan sekadar definisi istilah | 7 |
| Bagian Kendala & Solusi diisi dengan pengalaman nyata, bukan generik/kosong | 3 |

### Skala Konversi Akhir

Nilai akhir UTS = jumlah poin yang diperoleh di kelima aspek (total maksimum 100, dengan bonus di Aspek 1 bisa membuat total sedikit melebihi 100 sebelum dibatasi ke 100).

| Total Poin | Predikat |
|---|---|
| 85 – 100 | A |
| 70 – 84 | B |
| 55 – 69 | C |
| 40 – 54 | D |
| < 40 | E |

---

## Catatan untuk Dosen Penilai

- Kalau repository tidak bisa diakses sama sekali (link mati, private tanpa akses) saat batas waktu pengumpulan, Aspek 1–4 (85% dari total) dinilai 0 — laporan tetap dinilai berdasarkan isi tertulisnya di Aspek 5, kecuali laporan juga tidak dikumpulkan.
- Kolom kategori di `books/index.blade.php` dan `books/show.blade.php` yang menampilkan `category_id` mentah (bukan nama kategori) **bukan kekurangan** untuk UTS ini — relasi Eloquent baru topik Pertemuan 7, jangan dikurangi nilainya di Aspek 2.
- CRUD `members` memang belum wajib di titik ini (baru Tugas mandiri P5) — jangan dinilai sebagai kekurangan di luar poin bonus Aspek 1.
- `create.blade.php`, `edit.blade.php`, dan `show.blade.php` yang **masih HTML mandiri** (bukan `@extends('layouts.app')`) **bukan kekurangan** di Aspek 4 — hanya `index.blade.php` kedua resource yang jadi praktikum wajib Pertemuan 4. Refactor `create.blade.php` sengaja jadi Tugas mandiri (boleh belum dikerjakan), dan `edit.blade.php`/`show.blade.php` memang tidak pernah diminta direfactor di modul manapun. Jangan disamakan dengan halaman yang belum pakai master layout karena tidak dikerjakan — periksa dulu apakah itu memang cakupan wajib sebelum mengurangi nilai.

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-05.md) | [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-07.md)*
