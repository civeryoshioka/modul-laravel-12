# Glosarium — Daftar Istilah

> Referensi cepat untuk istilah-istilah yang muncul sepanjang Pertemuan 1–11. Diurutkan mengikuti urutan kemunculan pertama di modul, bukan alfabetis, supaya bisa dibaca berurutan sebagai ringkasan seluruh semester. Setiap istilah ditautkan ke pertemuan tempat ia pertama dijelaskan lebih dalam.

---

| Istilah | Penjelasan Singkat | Pertama Dibahas |
|---|---|---|
| **Framework** | Kerangka kerja siap pakai yang menyediakan struktur dan aturan baku, supaya developer tidak menulis ulang hal-hal umum (routing, koneksi database, dst) dari nol di setiap project. | [P1](../pertemuan-01.md) |
| **MVC (Model-View-Controller)** | Pola arsitektur yang memisahkan tanggung jawab kode jadi 3 bagian: Model (data & aturan bisnis), View (tampilan), Controller (penghubung keduanya). | [P1](../pertemuan-01.md) |
| **Routing** | Mekanisme yang memetakan URL + HTTP Method ke kode yang harus dijalankan (Controller atau closure). | [P2](../pertemuan-02.md) |
| **Resource Route** | Satu baris route (`Route::resource`) yang otomatis mendaftarkan 7 route CRUD standar (index, create, store, show, edit, update, destroy). | [P2](../pertemuan-02.md) |
| **Request Lifecycle** | Urutan lengkap yang dilalui satu request HTTP, dari diterima server sampai response dikirim balik ke browser (index.php → Kernel → Middleware → Router → Controller → Response). | [P2](../pertemuan-02.md) |
| **Middleware** | Lapisan kode yang "menyaring" request sebelum sampai ke Controller (atau response sebelum sampai ke browser) — misalnya mengecek apakah user sudah login. | [P2](../pertemuan-02.md) |
| **Form Request** | Class khusus untuk menampung aturan validasi input, supaya Controller tidak penuh kode validasi. | [P3](../pertemuan-03.md) |
| **Validasi** | Proses memastikan data yang dikirim user sesuai aturan (format, wajib diisi, unik, dst) sebelum disimpan. | [P3](../pertemuan-03.md) |
| **Template Engine** | Sistem yang memungkinkan HTML dicampur dengan logika tampilan sederhana (perulangan, kondisi) tanpa menulis PHP mentah berulang-ulang. Blade adalah template engine bawaan Laravel. | [P4](../pertemuan-04.md) |
| **Layout Inheritance** | Pola "satu layout induk, banyak halaman anak" (`@extends`/`@section`/`@yield` di Blade), supaya elemen berulang (navbar, footer) tidak ditulis ulang di tiap halaman. | [P4](../pertemuan-04.md) |
| **Partial View** | Potongan tampilan kecil yang bisa dipakai ulang di banyak halaman lewat `@include` (contoh: `partials/navbar.blade.php`). | [P4](../pertemuan-04.md) |
| **Migration** | Berkas kode yang mendefinisikan struktur tabel database, berfungsi sebagai "version control" untuk skema database supaya konsisten antar developer/environment. | [P5](../pertemuan-05.md) |
| **Eloquent ORM** | Lapisan yang menerjemahkan baris tabel database jadi objek PHP (dan sebaliknya), sehingga query bisa ditulis lewat method PHP (`Book::find(1)`) alih-alih SQL mentah. | [P5](../pertemuan-05.md) |
| **Mass Assignment / `$fillable`** | Mekanisme keamanan Eloquent yang membatasi kolom mana saja yang boleh diisi lewat `create()`/`update()` sekaligus, mencegah user menyuntik kolom yang tidak seharusnya bisa diubah (misal `role`). | [P5](../pertemuan-05.md) |
| **Relationship (`hasMany`/`belongsTo`)** | Definisi relasi antar tabel di level Model Eloquent, supaya data terkait bisa diakses lewat properti objek (`$book->category`) tanpa menulis JOIN manual. | [P7](../pertemuan-07.md) |
| **N+1 Query Problem** | Bug performa yang muncul saat sebuah loop memicu 1 query tambahan per barisnya, sehingga total query membengkak seiring jumlah data. Lihat [pendalaman lengkap](./n-plus-1-query-problem.md). | [P7](../pertemuan-07.md) |
| **Eager Loading (`with()`)** | Solusi N+1 Query Problem — mengambil data relasi di awal lewat satu query tambahan (`WHERE id IN (...)`), bukan satu-satu di dalam loop. | [P7](../pertemuan-07.md) |
| **Autentikasi (Authentication)** | Proses membuktikan identitas pengguna — "kamu ini siapa?" — biasanya lewat email/password. | [P8](../pertemuan-08.md) |
| **Otorisasi (Authorization)** | Proses menentukan hak akses setelah identitas terbukti — "kamu boleh melakukan apa?". Berbeda dari autentikasi meski sering tertukar. | [P8](../pertemuan-08.md) |
| **Session-based Authentication** | Pendekatan autentikasi di mana server menyimpan status login, lalu mengirim ID sesi ke browser lewat cookie — dipakai project ini untuk sisi web. | [P8](../pertemuan-08.md) |
| **Token-based Authentication** | Pendekatan autentikasi di mana klien menyimpan token (misalnya JWT) dan mengirimkannya eksplisit di setiap request lewat header — umum dipakai API untuk mobile/frontend terpisah. | [P8](../pertemuan-08.md) |
| **REST API** | Gaya arsitektur API berbasis HTTP yang memperlakukan data sebagai "resource" beracuan URL, dengan HTTP Method sebagai aksinya (GET=baca, POST=buat, dst), dan biasanya bertukar data lewat JSON. | [P9](../pertemuan-09.md) |
| **JSON** | Format teks ringan untuk pertukaran data terstruktur antar sistem (menggantikan HTML sebagai response saat berkomunikasi dengan aplikasi lain, bukan browser manusia). | [P9](../pertemuan-09.md) |
| **HTTP Status Code** | Kode angka 3 digit di response yang menandakan hasil request (200 sukses, 201 berhasil dibuat, 404 tidak ditemukan, 422 validasi gagal, 500 error server). | [P9](../pertemuan-09.md) |
| **API Resource** | Class Laravel untuk mengatur format response JSON secara konsisten (memilih field, transformasi data) tanpa mengembalikan seluruh kolom mentah dari Model. | [P9](../pertemuan-09.md) |
| **Laravel HTTP Client** | Wrapper Laravel (`Http::get()`, `Http::post()`) untuk memanggil API dari dalam kode PHP sendiri — dipakai project ini supaya Controller web bisa mengonsumsi REST API-nya sendiri. | [P10](../pertemuan-10.md) |
| **Seeder** | Kode yang mengisi database dengan data awal/contoh secara terprogram dan berulang (`php artisan db:seed`), alih-alih diisi manual satu-satu. | [P10](../pertemuan-10.md) |
| **Factory** | Kode yang mendefinisikan cara menghasilkan data dummy realistis (lewat Faker) untuk 1 Model, biasanya dipakai bersama Seeder atau testing. | [P10](../pertemuan-10.md) |
| **Git / Version Control** | Sistem pencatatan riwayat perubahan kode secara terstruktur (commit, branch, merge). Lihat [penjelasan dasar lengkap](./git-github-dasar.md). | [P1](../pertemuan-01.md) |

---

*Referensi lain: [Daftar Isi Modul](../README.md)*
