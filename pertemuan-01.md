# Pertemuan 1 — Framework, MVC & Setup Proyek

> **Sebelumnya:** Belum ada — ini adalah pertemuan pertama.
> **Pertemuan ini:** Memahami konsep framework dan arsitektur MVC, lalu menginstal Laravel 12 dan menyiapkan repository GitHub untuk project `app-perpustakaan`.

---

## Capaian Pembelajaran

Setelah menyelesaikan pertemuan ini, mahasiswa mampu:
1. Menjelaskan mengapa framework digunakan dan masalah apa yang diselesaikannya
2. Memahami pola arsitektur MVC dan tanggung jawab tiap komponennya
3. Menginstal Laravel 12 via Composer dan menjalankan server development
4. Memahami struktur direktori Laravel 12 dan fungsi tiap foldernya
5. Membuat repository GitHub, branch `dev`, dan melakukan push pertama

---

## Konsep: Mengapa Framework Ada?

Bayangkan kamu diminta membangun sebuah rumah. Kamu bisa saja menuang beton sendiri, merangkai kabel listrik dari nol, dan merancang sistem pipa air dari awal — tapi hampir tidak ada orang yang melakukan itu. Kita menggunakan bahan bangunan yang sudah distandarkan, tukang yang sudah tahu pola kerja umum, dan cetakan (blueprint) yang sudah terbukti aman. Framework dalam pemrograman web bekerja dengan logika yang sama: ia adalah kerangka kerja yang sudah menyediakan solusi untuk masalah-masalah yang **selalu muncul** di hampir semua aplikasi web — routing URL ke kode yang tepat, koneksi ke database, validasi input, manajemen sesi login, hingga pengiriman response HTML atau JSON. Tanpa framework, setiap developer akan menulis ulang solusi untuk masalah yang sama, dengan kualitas dan tingkat keamanan yang berbeda-beda.

Pada masa awal PHP (sering disebut era "spaghetti code"), satu file `.php` bisa berisi campuran query SQL, logika bisnis, dan HTML sekaligus dalam satu halaman. Ini terlihat cepat pada awalnya, tapi begitu aplikasi bertambah besar — puluhan halaman, puluhan developer — kode menjadi sangat sulit ditelusuri. Mengubah tampilan bisa merusak logika, mengubah logika bisa merusak query database, dan tidak ada satu pun yang bisa dikerjakan secara paralel oleh tim tanpa saling tabrakan. Framework lahir sebagai jawaban atas masalah ini: dengan memaksakan **struktur dan aturan main yang konsisten**, kode menjadi lebih mudah dipahami oleh siapa pun yang membaca — termasuk dirimu sendiri enam bulan kemudian.

Penting untuk membedakan framework dari library. Library adalah alat yang kamu panggil sesuai kebutuhan — kamu yang mengendalikan alur program, dan library hanya dipanggil ketika dibutuhkan (contohnya: sebuah library untuk memformat tanggal). Framework justru sebaliknya: framework yang mengendalikan alur utama aplikasi, dan kode yang kamu tulis "dipanggil" oleh framework pada titik-titik tertentu. Konsep ini disebut **Inversion of Control** — kendali dibalik, dari kode kamu ke framework. Inilah sebabnya belajar framework terasa berbeda dari belajar library biasa: kamu harus memahami "aturan main" framework tersebut, bukan sekadar memanggil fungsinya.

Setiap bahasa pemrograman punya framework populer dengan filosofi serupa meski sintaksnya berbeda. Laravel (PHP) yang akan kita pakai sepanjang modul ini punya saudara di ekosistem lain: **Express.js** di Node.js (lebih minimalis, banyak hal harus ditambahkan manual), **Django** di Python (mirip Laravel — "batteries included", banyak fitur bawaan), dan **Spring Boot** di Java (populer di lingkungan enterprise). Semua framework ini menyelesaikan masalah yang sama — routing, akses database, validasi, autentikasi — hanya dengan sintaks dan konvensi yang berbeda. Artinya, begitu kamu memahami *konsep* di balik Laravel, kamu akan jauh lebih cepat beradaptasi ketika suatu saat harus bekerja dengan framework lain. Inilah alasan modul ini selalu menekankan "mengapa", bukan hanya "bagaimana".

Salah satu prinsip arsitektur yang diadopsi hampir semua framework web modern (termasuk keempat framework di atas) adalah **MVC — Model, View, Controller**. MVC adalah pola pemisahan tanggung jawab kode ke dalam tiga peran berbeda:

- **Model** — bertanggung jawab atas data dan aturan bisnis terkait data tersebut. Model adalah satu-satunya bagian yang "tahu" tentang struktur tabel `books`, `members`, atau `loans`, dan bagaimana relasi antar data itu bekerja.
- **View** — bertanggung jawab murni atas tampilan yang dilihat pengguna (HTML). View tidak boleh berisi logika bisnis rumit, hanya menampilkan data yang sudah disiapkan.
- **Controller** — bertanggung jawab menerima request dari pengguna, memanggil Model untuk mengambil/mengubah data, lalu mengirim data tersebut ke View untuk ditampilkan. Controller adalah "penghubung" antara Model dan View.

Kenapa pemisahan ini penting? Pertama, **separation of concerns** — setiap bagian kode punya satu tanggung jawab yang jelas, sehingga lebih mudah dipahami dan ditelusuri saat terjadi bug. Kedua, **maintainability** — ketika desain tampilan berubah total, kamu cukup mengubah View tanpa menyentuh logika data di Model. Ketiga, **kolaborasi tim** — seorang frontend developer bisa fokus mengerjakan View, sementara backend developer fokus mengerjakan Model dan Controller, tanpa saling mengganggu pekerjaan satu sama lain. Pola MVC ini bukan konsep eksklusif Laravel — Django memakai istilah MTV (Model-Template-View) yang pada dasarnya sama, dan Spring Boot MVC secara eksplisit menggunakan nama yang sama. Begitu kamu paham MVC di Laravel, kamu akan langsung mengenali pola yang sama di framework lain.

---

## Studi Kasus: Sistem Perpustakaan Digital Kampus

Sepanjang modul ini, seluruh konsep di atas — MVC, routing, validasi, dan seterusnya — akan langsung dipraktikkan lewat satu project yang sama dari pertemuan ke pertemuan: **Sistem Perpustakaan Digital Kampus**, aplikasi yang dikelola petugas/admin untuk mengelola buku, anggota, dan transaksi peminjaman.

Project ini tidak dibangun sekaligus. Pertemuan ini baru menyiapkan kerangka kosongnya (instalasi Laravel, koneksi database), dan setiap pertemuan berikutnya menambah satu-dua bagian sampai jadi aplikasi utuh di UAS. Supaya kamu tidak "tersesat" di tengah jalan — misalnya bingung kenapa tabel `loan_items` ada padahal belum pernah dijelaskan — baca dulu **[Studi Kasus & Desain Database](./studi-kasus-database.md)** yang berisi gambaran lengkap aktor, fitur akhir, desain tabel, dan relasinya. Halaman itu jadi referensi yang bisa dibuka ulang kapan pun sepanjang semester, terutama nanti di Pertemuan 5 saat tabel-tabelnya benar-benar dibuat.

---

## Materi

### Evolusi Pengembangan Web: Spaghetti Code → Framework

Sebelum framework populer, developer PHP biasa menulis satu file untuk satu halaman, mencampur query SQL, logika, dan HTML dalam satu tempat. Kode seperti ini disebut *spaghetti code* karena alur eksekusinya sulit diikuti — mirip mie yang saling melilit. Masalah muncul ketika aplikasi berkembang: perubahan kecil di satu tempat berisiko merusak bagian lain yang tidak terduga, dan developer baru butuh waktu lama untuk memahami kode yang ada. Framework hadir untuk memaksa struktur yang konsisten sehingga masalah ini bisa diminimalkan sejak awal.

### Framework vs Library

| | Library | Framework |
|---|---|---|
| Siapa mengendalikan alur | Kode kamu | Framework |
| Kapan dipanggil | Kamu panggil saat butuh | Framework memanggil kode kamu (Inversion of Control) |
| Contoh | Carbon (format tanggal), Guzzle (HTTP client) | Laravel, Django, Express.js, Spring Boot |

### Framework Populer di Berbagai Bahasa

| Bahasa | Framework | Karakteristik |
|---|---|---|
| PHP | Laravel | "Batteries included" — banyak fitur bawaan (auth, ORM, queue, dll) |
| JavaScript (Node.js) | Express.js | Minimalis, sebagian besar fitur ditambahkan lewat package terpisah |
| Python | Django | Mirip Laravel — banyak fitur bawaan, konvensi kuat |
| Java | Spring Boot | Umum di lingkungan enterprise, berbasis dependency injection |

### Arsitektur MVC: Tanggung Jawab Tiap Komponen

```
Browser ──request──▶ Controller ──ambil/ubah data──▶ Model ──▶ Database
                          │
                          └──kirim data──▶ View ──HTML──▶ Browser
```

- **Model** (`app/Models/`) — representasi tabel database beserta relasi dan aturan bisnisnya
- **View** (`resources/views/`) — file `.blade.php` yang berisi HTML dan menerima data untuk ditampilkan
- **Controller** (`app/Http/Controllers/`) — menerima request, memproses lewat Model, mengembalikan View atau response lain

### Kenapa MVC: Separation of Concerns, Maintainability, Kolaborasi Tim

Tanpa MVC, satu file bisa menangani query database, validasi, dan HTML sekaligus — sulit dites, sulit dipelihara, sulit dikerjakan berdua. Dengan MVC, setiap file punya satu tanggung jawab spesifik. Ini juga membuat kode lebih mudah diuji (testable) karena Model bisa diuji terpisah dari tampilan.

### Struktur Direktori Laravel 12

| Folder/File | Fungsi |
|---|---|
| `app/Http/Controllers/` | Tempat semua Controller |
| `app/Models/` | Tempat semua Model Eloquent |
| `resources/views/` | Tempat semua file Blade (View) |
| `routes/web.php` | Definisi route untuk halaman web (HTML) |
| `routes/api.php` | Definisi route untuk endpoint API (JSON) |
| `database/migrations/` | Definisi struktur tabel database |
| `database/seeders/` | Data awal/dummy untuk database |
| `public/` | Folder yang diakses langsung browser (entry point `index.php`) |
| `config/` | File konfigurasi aplikasi (database, session, dll) |
| `.env` | Konfigurasi environment (database, app key, mode debug) |
| `artisan` | CLI tool bawaan Laravel untuk generate kode dan menjalankan perintah |

### File `.env`: Konfigurasi Database, App Key, Environment

File `.env` menyimpan konfigurasi yang berbeda-beda tiap environment (local, staging, production) — seperti kredensial database — sehingga tidak perlu (dan tidak boleh) ditulis langsung di kode dan di-commit ke Git. File ini ada di `.gitignore` secara default agar kredensial tidak pernah bocor ke repository publik.

### Perintah Artisan Dasar

| Perintah | Fungsi |
|---|---|
| `php artisan serve` | Menjalankan development server |
| `php artisan list` | Menampilkan semua perintah artisan yang tersedia |
| `php artisan make:controller` | Membuat file Controller baru |
| `php artisan make:model` | Membuat file Model baru |
| `php artisan make:migration` | Membuat file migration baru |

### Git Workflow: Init, Remote Add, Branch Dev, Push ke GitHub

Sepanjang modul ini kita memakai dua branch utama: `main` untuk kode yang sudah stabil di tiap checkpoint, dan `dev` untuk pengembangan aktif. Semua pekerjaan harian dilakukan di branch `dev`.

---

## Praktikum

> **Konteks:** Membuat project `app-perpustakaan` dari nol.
> Pastikan branch aktif adalah `dev` di akhir praktikum ini.
> Di akhir praktikum ini kamu akan memiliki: project Laravel 12 yang berjalan di `http://127.0.0.1:8000`, terhubung ke database `db_perpustakaan`, dan sudah berada di repository GitHub dengan branch `main` dan `dev`.

### Langkah 1 — Instal Laravel 12 via Composer

Composer adalah dependency manager PHP — alat yang mengunduh Laravel beserta seluruh library pendukungnya. Jalankan perintah berikut di folder `modul-framework/`:

```bash
composer create-project laravel/laravel app-perpustakaan
cd app-perpustakaan
```

Perintah ini membuat project Laravel 12 baru lengkap dengan struktur folder standar (lihat tabel struktur direktori di atas).

### Langkah 2 — Jalankan Development Server

Pastikan project bisa berjalan sebelum melangkah lebih jauh.

```bash
php artisan serve
```

Buka `http://127.0.0.1:8000` di browser — akan tampil halaman selamat datang bawaan Laravel.

> 📸 *Screenshot: Halaman welcome default Laravel 12 di browser, menampilkan logo Laravel dan link dokumentasi.*

### Langkah 3 — Konfigurasi Database di `.env`

Buka file `.env` di root project dan sesuaikan bagian koneksi database:

```env
# File: .env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=db_perpustakaan
DB_USERNAME=root
DB_PASSWORD=
```

Buat database kosong bernama `db_perpustakaan` di MySQL/phpMyAdmin — belum ada tabel apa pun di pertemuan ini, koneksi database baru akan benar-benar dipakai mulai Pertemuan 5 (Migration).

### Langkah 4 — Inisialisasi Git dan Hubungkan ke GitHub

Buat dua repository kosong di GitHub terlebih dahulu (misalnya `app-perpustakaan`), lalu hubungkan project lokal:

```bash
git init
git remote add origin https://github.com/[USERNAME]/app-perpustakaan.git
git add .
git commit -m "[P-1] inisialisasi project laravel 12"
git push -u origin main
```

### Langkah 5 — Buat Branch `dev`

Seluruh pengembangan mulai pertemuan berikutnya dikerjakan di branch `dev`, sementara `main` hanya diperbarui di checkpoint-checkpoint tertentu.

```bash
git checkout -b dev
git push -u origin dev
```

### Langkah 6 — Ujicoba

Pastikan branch aktif adalah `dev`:

```bash
git branch
```

Jalankan ulang server dan pastikan halaman welcome masih tampil tanpa error:

```bash
php artisan serve
```

Buka `http://127.0.0.1:8000` — jika halaman welcome Laravel tampil tanpa error PHP, project siap dilanjutkan ke Pertemuan 2.

---

## Checkpoint GitHub

```bash
git add .
git commit -m "[P-1] inisialisasi project laravel 12"
git push origin dev
```

---

## Tugas

Setelah menyelesaikan praktikum, kerjakan secara mandiri:
1. Pastikan repository `app-perpustakaan` di GitHub memiliki branch `main` dan `dev`, keduanya sudah ter-push
2. Update `README.md` di root project dengan deskripsi singkat aplikasi (nama aplikasi, tujuan, cara menjalankan project secara lokal)
3. Tuliskan dalam 2-3 kalimat (boleh sebagai komentar di README atau file terpisah): apa perbedaan Model, View, dan Controller menurut pemahamanmu sendiri

**Yang dikumpulkan:**
- Link repository GitHub `app-perpustakaan` (branch `dev`) yang berisi hasil setup
- Screenshot halaman welcome Laravel yang berhasil berjalan di `http://127.0.0.1:8000`

---

*Navigasi: [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-02.md)*
