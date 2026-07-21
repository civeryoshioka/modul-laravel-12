# Pertemuan 10 — Blade + Konsumsi API & Project Clinic

> **Sebelumnya:** Pertemuan 9 membangun REST API lengkap (`books`, `members`, `loans`, `stats`) yang mengembalikan JSON lewat API Resource — tapi endpoint-endpoint itu baru diuji dari luar aplikasi, lewat Postman.
> **Pertemuan ini:** Aplikasi Laravel yang sama menjadi *klien* dari API-nya sendiri — `DashboardController` dan `LoanController@report` memakai Laravel HTTP Client untuk memanggil `GET /api/stats` dan `GET /api/loans`, lalu menampilkan hasilnya di halaman Blade. Project juga dilengkapi Seeder & Factory supaya database punya data dummy realistis untuk demo, menutup rangkaian praktikum sebelum UAS.

---

## Capaian Pembelajaran

Setelah menyelesaikan pertemuan ini, mahasiswa mampu:
1. Memahami konsep mengonsumsi API dari dalam aplikasi sendiri
2. Menggunakan Laravel HTTP Client untuk memanggil API endpoint
3. Menampilkan data dari respons API ke halaman Blade
4. Membuat data dummy realistis dengan Seeder dan Factory

---

## Konsep: Mengapa Consume API Ada?

Pertemuan 9 menutup dengan satu kalimat penting: REST API menyediakan "satu sumber data yang sama, dalam format netral, yang bisa dikonsumsi klien apa pun tanpa peduli klien itu ditulis dalam bahasa atau platform apa". Klien itu tidak harus berupa aplikasi terpisah seperti mobile app atau frontend JavaScript — aplikasi Laravel yang sama, dari sisi Blade-nya, juga bisa berperan sebagai klien dari API miliknya sendiri. Ini terdengar aneh di awal ("kenapa aplikasi manggil dirinya sendiri lewat HTTP, padahal bisa langsung query Eloquent?"), tapi justru di situlah nilainya sebagai latihan: begitu sebuah halaman Blade terbukti bisa mengonsumsi API internal dengan benar, halaman yang sama itu — tanpa perubahan logic tampilan sama sekali — siap dialihkan untuk mengonsumsi API dari server lain di masa depan (microservice, layanan pihak ketiga, atau versi mobile-first dari aplikasi yang sama). Pola konsumsi API tidak berubah, yang berubah cuma URL dan mungkin autentikasinya.

Ada juga alasan yang lebih praktis: memisahkan "cara data disajikan" dari "cara data ditampilkan". `DashboardController@index` di pertemuan ini tidak tahu dan tidak peduli bagaimana `Api\StatsController` menghitung `total_buku`. dia cuma tahu bahwa `GET /api/stats` akan mengembalikan JSON berbentuk `{"total_buku": ..., "total_anggota": ..., "peminjaman_aktif": ...}`. Kalau suatu saat logic penghitungan statistik berubah (misalnya butuh cache, atau query yang lebih rumit), `DashboardController` tidak perlu disentuh sama sekali selama bentuk responsnya tetap sama — inilah manfaat *separation of concerns* yang sama yang mendasari kenapa Model, View, dan Controller dipisah di MVC, diterapkan sekali lagi di level yang lebih besar: pemisahan antara *penyedia data* (API) dan *konsumen data* (Blade).

Pola ini juga framework-agnostic dan sangat umum di industri, sering disebut *BFF (Backend for Frontend)* atau *service-to-service call*. Aplikasi Next.js sering punya API Route internal (`/api/...`) yang dipanggil dari halaman React di aplikasi yang sama lewat `fetch()`, alih-alih React langsung query database. Aplikasi Django dengan Django REST Framework kerap punya template HTML yang memanggil endpoint API-nya sendiri lewat `requests.get()` di view function, terutama saat data yang sama juga perlu diekspos ke klien eksternal. Bahkan di arsitektur microservice skala besar, layanan "penyaji halaman" (*gateway* atau *BFF service*) hampir selalu memanggil layanan lain lewat HTTP/gRPC, bukan mengakses database layanan lain secara langsung — prinsip yang sama, cuma skalanya lebih besar. Memahami pola ini di skala kecil (satu aplikasi Laravel memanggil dirinya sendiri) memberi fondasi untuk memahami arsitektur yang jauh lebih besar nanti.

Namun ada satu kejutan teknis penting yang justru menjadi pelajaran tersendiri di pertemuan ini: server pengembangan `php artisan serve` (PHP built-in web server) secara default hanya memproses **satu request pada satu waktu**. Kalau sebuah request yang sedang diproses server itu sendiri membuat request HTTP baru ke server yang sama, request baru itu tidak akan pernah bisa diterima — server masih sibuk memproses request pertama, padahal request pertama itu sedang menunggu balasan dari request kedua. Ini disebut *deadlock*: dua pihak saling menunggu, dan tidak ada yang bisa maju. Skenario ini bukan bug di kode aplikasi, melainkan karakteristik dari server pengembangan yang dipakai — di produksi sungguhan (Nginx + PHP-FPM, atau Laravel Octane), banyak request bisa diproses bersamaan sehingga masalah ini tidak muncul. Memahami *kenapa* deadlock ini terjadi, dan bagaimana cara pragmatis mengatasinya di lingkungan pengembangan lokal, adalah bagian dari materi pertemuan ini — bukan sekadar "tempelkan kode ini supaya jalan".

---

## Materi

### Laravel HTTP Client

Laravel menyediakan `Http` facade sebagai pembungkus [Guzzle](https://docs.guzzlephp.org/) yang jauh lebih ringkas untuk membuat request HTTP keluar dari kode PHP:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
use Illuminate\Support\Facades\Http;

$response = Http::get('https://api.contoh.com/data');
$response = Http::post('https://api.contoh.com/data', ['nama' => 'Budi']);
```

`Http::get()` dan `Http::post()` mengembalikan objek `Illuminate\Http\Client\Response`, bukan string mentah — objek ini punya method siap pakai untuk memeriksa dan mem-parsing hasilnya, jauh lebih nyaman dibanding fungsi bawaan PHP seperti `curl_exec()` atau `file_get_contents()` yang butuh banyak kode boilerplate untuk hal yang sama.

### Parsing JSON Response

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
$response = Http::get('https://api.contoh.com/stats');

if ($response->successful()) {
    $data = $response->json(); // array PHP hasil decode JSON
    echo $data['total_buku'];
}
```

`$response->successful()` mengembalikan `true` kalau status code response ada di rentang `2xx` — cara idiomatis untuk memastikan request benar-benar berhasil sebelum mencoba membaca isinya. Tapi `successful()` cuma menangani kasus "server membalas, tapi dengan status gagal" (misalnya `404` atau `500`) — ada kasus lain yang lebih parah: server tujuan **sama sekali tidak bisa dihubungi** (mati, port salah, firewall). Dalam kasus itu, `Http::get()` tidak mengembalikan `Response` sama sekali, melainkan melempar `Illuminate\Http\Client\ConnectionException`. Kalau exception ini tidak ditangkap, kode tidak akan pernah sampai ke baris `$response->successful()` — request-nya sendiri gagal total sebelum ada response untuk diperiksa, dan halaman akan menampilkan error 500 ke pengguna. Kode yang baik menyediakan nilai *fallback* untuk **kedua** skenario ini — status gagal maupun gagal connect sama sekali — bukan cuma salah satunya:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
try {
    $response = Http::get('https://api.contoh.com/stats');

    $data = $response->successful() ? $response->json() : ['total_buku' => 0];
} catch (\Illuminate\Http\Client\ConnectionException $e) {
    $data = ['total_buku' => 0]; // server API sama sekali tidak bisa dihubungi
}
```

### Kenapa Consume API Internal Bisa Deadlock — dan Solusinya

Seperti dibahas di bagian Konsep, `php artisan serve` memproses satu request per waktu. Kalau `DashboardController` memanggil `Http::get(url('/api/stats'))` — yaitu, memanggil balik ke *port yang sama* dengan server yang sedang memproses request itu — server tidak akan pernah bisa menerima request kedua itu, karena dia masih "terkunci" memproses request pertama yang justru sedang menunggu balasan dari request kedua. Hasilnya: halaman menggantung (hang) tanpa henti sampai akhirnya `Http::get()` timeout dengan error koneksi.

Solusi pragmatis untuk lingkungan pengembangan lokal adalah menjalankan **dua instance** `php artisan serve` di port berbeda — satu untuk melayani trafik browser seperti biasa, satu lagi khusus menerima panggilan API dari dalam aplikasi sendiri:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
// config/services.php
'internal_api' => [
    'base_url' => env('INTERNAL_API_URL', 'http://127.0.0.1:8011'),
],
```

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
$response = Http::get(config('services.internal_api.base_url').'/api/stats');
```

Karena kedua instance server menjalankan kode aplikasi dan terhubung ke database yang sama persis, request ke port kedua tetap membaca data yang sama — bedanya cuma proses PHP yang menanganinya berbeda, sehingga tidak saling mengunci. Di server produksi sungguhan (bukan `php artisan serve`), masalah ini pada dasarnya tidak muncul karena web server produksi (Nginx, Apache) memang dirancang menangani banyak request bersamaan.

### Seeder: Data Dummy yang Konsisten

Sepanjang pertemuan sebelumnya, data uji coba dibuat manual lewat `tinker` satu-per-satu — cara ini tidak bisa diulang secara konsisten, dan merepotkan setiap kali database perlu di-reset. Seeder menyelesaikan masalah ini: kelas PHP yang isinya instruksi "isi tabel ini dengan data berikut", dijalankan lewat `php artisan db:seed` atau otomatis lewat `migrate:fresh --seed`.

```bash
php artisan make:seeder CategorySeeder
```

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
public function run(): void
{
    Category::updateOrCreate(
        ['nama_kategori' => 'Novel'],
        ['deskripsi' => 'Buku fiksi panjang...']
    );
}
```

`updateOrCreate()` dipakai alih-alih `create()` supaya Seeder aman dijalankan berkali-kali tanpa membuat baris duplikat — pola yang sama yang sudah dipakai `UserSeeder` sejak Pertemuan 8.

### Factory: Data Dummy dalam Jumlah Banyak

Seeder cocok untuk data yang jumlahnya sedikit dan nilainya spesifik (5 kategori tetap, misalnya). Tapi untuk kebutuhan seperti "20 buku dummy" atau "15 anggota dummy", menuliskan nilainya satu-satu tidak masuk akal — di sinilah Factory dipakai, memanfaatkan library [Faker](https://fakerphp.github.io/) untuk menghasilkan data acak yang realistis:

```bash
php artisan make:factory BookFactory --model=Book
```

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
public function definition(): array
{
    return [
        'judul' => ucfirst(fake()->words(3, true)),
        'penulis' => fake()->name(),
        'tahun_terbit' => fake()->numberBetween(1990, (int) date('Y')),
        'category_id' => Category::inRandomOrder()->value('id'),
    ];
}
```

`Book::factory()->count(20)->create()` lalu memanggil `definition()` dua puluh kali dan langsung menyimpan hasilnya ke database. Perhatikan `category_id` diambil lewat `Category::inRandomOrder()->value('id')` — artinya Seeder yang memakai `BookFactory` **wajib** dijalankan setelah `CategorySeeder`, supaya ada baris `categories` yang bisa dipilih. Urutan ini bukan detail sepele; salah urutan berarti `category_id` yang dihasilkan `null` atau error foreign key.

### Data Dummy yang Terasa Nyata: Kurasi vs Faker Default

Faker secara default memakai locale `en_US` — nama, alamat, dan nama perusahaan yang dihasilkan berbahasa Inggris/Amerika, kurang pas untuk demo aplikasi perpustakaan kampus Indonesia. Laravel menyediakan konfigurasi `APP_FAKER_LOCALE` di `.env` persis untuk kasus ini. Begitu diubah ke `id_ID`, seluruh pemanggilan `fake()->name()`, `fake()->firstName()`, `fake()->address()` di semua Factory otomatis menghasilkan data bergaya Indonesia — tanpa mengubah satu baris kode Factory pun, karena locale dibaca dari config `faker_locale` secara global.

Tapi locale saja tidak cukup untuk semua field. Faker tidak punya generator "judul buku" atau "nama penerbit" — method seperti `fake()->words()` (dipakai untuk teks acak generik) tetap menghasilkan rangkaian kata Latin tanpa makna, apa pun locale-nya, karena tidak ada data referensi "judul buku sungguhan" yang bisa diacak Faker. Untuk field seperti ini, solusinya bukan mencari fitur Faker yang lebih canggih, melainkan **kurasi manual**: menulis sendiri daftar judul buku dan nama penerbit yang realistis (idealnya dikelompokkan per kategori, supaya judul yang keluar konsisten dengan kategorinya), lalu memilih secara acak dari daftar itu pakai `fake()->randomElement()`. Prinsip yang sama dipakai untuk email anggota: `Member` di studi kasus ini adalah mahasiswa kampus yang sama dengan petugas, jadi masuk akal kalau emailnya memakai domain kampus `@pens.ac.id` juga — dibentuk manual dari nama lewat `Str::slug()`, ditambah angka acak supaya tetap unik, bukan domain generik hasil `fake()->safeEmail()`.

Pelajaran di balik ini: Faker mempercepat pembuatan data dalam jumlah banyak, tapi bukan pengganti keputusan desain data — kapan pakai locale bawaan, kapan kurasi manual, dan kapan menggabungkan keduanya (nama dari locale + domain email dari konteks aplikasi) tetap keputusan yang harus diambil sendiri sesuai kebutuhan studi kasus. Kode kurasi lengkapnya ada di Langkah 6 Praktikum.

### `DatabaseSeeder`: Orkestrasi Urutan

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
public function run(): void
{
    $this->call([
        UserSeeder::class,
        CategorySeeder::class,
        BookSeeder::class,
        MemberSeeder::class,
        LoanSeeder::class,
    ]);
}
```

`DatabaseSeeder::run()` adalah satu-satunya tempat urutan seeding didefinisikan secara eksplisit. Urutannya mengikuti ketergantungan foreign key: `Category` harus ada sebelum `Book` (butuh `category_id`), dan `Member` + `User` harus ada sebelum `Loan` (butuh `member_id` dan `user_id`). `php artisan migrate:fresh --seed` menjalankan seluruh migration dari nol lalu memanggil `DatabaseSeeder`, cara paling umum me-reset database ke kondisi demo yang bersih dan konsisten.

### Kenapa Pagination Bawaan Laravel Bisa Tampil Rusak

`{{ $books->links() }}` yang sudah dipakai sejak Pertemuan 5 sebenarnya menyembunyikan asumsi penting: view pagination default Laravel (namanya `tailwind`) dirender pakai ikon panah SVG dan class CSS Tailwind (`sm:inline-flex`, `rounded-md`, dsb) untuk mengatur ukuran, warna, dan tata letaknya. Selama jumlah data masih sedikit (≤ 10 baris, jadi cuma 1 halaman), `hasPages()` bernilai `false` dan `links()` tidak merender apa-apa — masalahnya tersembunyi begitu saja. Begitu Pertemuan 10 menambah data lewat Seeder (20 buku, 15 anggota — otomatis jadi 2 halaman), pagination akhirnya benar-benar dirender, dan baru ketahuan project ini **tidak pernah memuat Tailwind CSS** sama sekali (layout `app.blade.php` sejak Pertemuan 4 cuma pakai CSS custom polos di dalam tag `<style>`). Tanpa Tailwind, ikon SVG tetap muncul tapi ukurannya tidak dibatasi apa-apa — tampil raksasa dan merusak layout halaman.

Ini bukan bug di kode CRUD yang sudah dibuat sejak Pertemuan 5-7 — kodenya benar, cuma asumsinya (Tailwind tersedia) tidak sesuai kondisi project. Solusinya bukan memasang Tailwind CSS (perubahan besar di luar cakupan modul ini), melainkan mengganti *view* pagination dengan versi yang cuma pakai teks/link biasa, konsisten dengan gaya visual project yang sudah ada. Laravel mendukung ini lewat `Paginator::defaultView()` — didaftarkan sekali di `AppServiceProvider`, otomatis berlaku ke **semua** pemanggilan `->links()` di seluruh project tanpa perlu mengubah satu pun Blade file CRUD yang sudah ada.

---

## Praktikum

> **Konteks:** Melanjutkan project `app-perpustakaan` dari Pertemuan 9.
> Pastikan branch aktif adalah `dev`.
> Di akhir praktikum ini kamu akan memiliki: `dashboard.blade.php` yang menampilkan statistik dari `GET /api/stats`, `loans/report.blade.php` yang menampilkan laporan dari `GET /api/loans`, Seeder & Factory lengkap dengan data bergaya Indonesia, dan pagination yang tampil rapi di seluruh halaman.
>
> **Catatan soal dua server:** karena `php artisan serve` cuma memproses satu request per waktu (lihat bagian Konsep), praktikum ini butuh **dua terminal** berjalan bersamaan — satu di port biasa (`8000`, atau port lain yang biasa dipakai) untuk browser, satu lagi khusus dipanggil dari dalam aplikasi sendiri lewat `Http::get()`. Servernya baru benar-benar dibutuhkan mulai Langkah 8 (Ujicoba), tapi tidak masalah kalau mau dinyalakan dari awal.

### Langkah 1 — Konfigurasi Server Kedua Khusus API Internal

Tambahkan variabel `INTERNAL_API_URL` di `.env`, menunjuk ke port yang berbeda dari server utama:

```bash
# File: .env
# (tambahkan setelah baris APP_URL)

# Base URL instance kedua `php artisan serve` khusus panggilan API internal
# (Http::get() dari DashboardController/LoanController@report ke /api/...).
# Wajib port berbeda dari server utama, lihat config/services.php.
INTERNAL_API_URL=http://127.0.0.1:8011
```

Karena `.env` tidak ikut ter-commit ke Git, tambahkan baris yang sama juga ke `.env.example` supaya rekan satu tim tahu variabel ini wajib ada.

Daftarkan sebagai config di `config/services.php` supaya bisa dipanggil lewat `config()` helper dari mana saja, bukan `env()` langsung di Controller (praktik yang tidak disarankan Laravel). Tambahkan array berikut sebelum tanda kurung siku penutup `];` di paling akhir file:

```php
// File: config/services.php
'internal_api' => [
    'base_url' => env('INTERNAL_API_URL', 'http://127.0.0.1:8011'),
],
```

### Langkah 2 — `DashboardController` Mengonsumsi `GET /api/stats`

```bash
php artisan make:controller DashboardController
```

Isi `index()`-nya, dibungkus `try/catch` supaya tidak error 500 kalau server kedua belum menyala (lihat penjelasan `ConnectionException` di bagian Materi):

```php
// File: app/Http/Controllers/DashboardController.php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Client\ConnectionException;
use Illuminate\Support\Facades\Http;

class DashboardController extends Controller
{
    public function index()
    {
        $stats = [
            'total_buku' => 0,
            'total_anggota' => 0,
            'peminjaman_aktif' => 0,
        ];

        try {
            $response = Http::get(config('services.internal_api.base_url').'/api/stats');

            if ($response->successful()) {
                $stats = $response->json();
            }
        } catch (ConnectionException $e) {
            // Server internal API (port 8011) tidak bisa dihubungi — statistik tetap tampil nol.
        }

        return view('dashboard', compact('stats'));
    }
}
```

`try/catch` di sini menangkap dua skenario gagal sekaligus: response dengan status error (`successful()` bernilai `false`) **dan** kegagalan koneksi total (server kedua belum dinyalakan). Tanpa `catch (ConnectionException $e)`, skenario kedua akan lolos sebagai error 500 yang tidak tertangani.

Route `/` yang sejak Pertemuan 8 hanya redirect sederhana ke `/books`, sekarang **diganti** (bukan ditambah — hapus closure lamanya) supaya mengarah ke Controller ini. Perhatikan juga baris `use` untuk `DashboardController` wajib ditambahkan di bagian atas file, kalau lupa Laravel akan melempar error `Class "DashboardController" not found`:

```php
// File: routes/web.php
<?php

use App\Http\Controllers\AuthController;
use App\Http\Controllers\BookController;
use App\Http\Controllers\CategoryController;
use App\Http\Controllers\DashboardController;
use App\Http\Controllers\LoanController;
use App\Http\Controllers\MemberController;
use Illuminate\Support\Facades\Route;

// Public
Route::get('/login', [AuthController::class, 'showLogin'])->name('login');
Route::post('/login', [AuthController::class, 'login']);
Route::post('/logout', [AuthController::class, 'logout'])->name('logout');

// Protected
Route::middleware(['auth'])->group(function () {
    Route::get('/', [DashboardController::class, 'index'])->name('dashboard');

    Route::resource('books', BookController::class);
    Route::resource('members', MemberController::class);
    Route::resource('loans', LoanController::class);
    Route::put('/loans/{id}/kembalikan', [LoanController::class, 'kembalikan'])
        ->name('loans.kembalikan');

    Route::middleware(['admin'])->group(function () {
        Route::resource('categories', CategoryController::class)->except(['show']);
    });
});
```

> Baris `Route::get('/loans/report', ...)` sengaja belum ditambahkan di sini — itu bagian Langkah 3, supaya urutannya jelas kenapa dia harus ditaruh sebelum `Route::resource('loans', ...)`.

`dashboard.blade.php` menampilkan tiga angka statistik dalam kartu sederhana, plus link ke halaman laporan (dibuat Langkah 3):

```html
<!-- File: resources/views/dashboard.blade.php -->
@extends('layouts.app')

@section('title', 'Dashboard')

@section('content')
    <h1>Dashboard</h1>
    <p>Ringkasan statistik perpustakaan, diambil langsung dari <code>GET /api/stats</code>.</p>

    <div class="stats-grid">
        <div class="stat-card">
            <div class="value">{{ $stats['total_buku'] }}</div>
            <div class="label">Total Buku</div>
        </div>
        <div class="stat-card">
            <div class="value">{{ $stats['total_anggota'] }}</div>
            <div class="label">Total Anggota</div>
        </div>
        <div class="stat-card">
            <div class="value">{{ $stats['peminjaman_aktif'] }}</div>
            <div class="label">Peminjaman Aktif</div>
        </div>
    </div>

    <p style="margin-top: 24px;"><a href="{{ route('loans.report') }}" class="btn">Lihat Laporan Peminjaman</a></p>
@endsection
```

Kartu statistik di atas pakai class `.stats-grid`/`.stat-card` yang belum ada di CSS. Tambahkan ke blok `<style>` di layout master, setelah baris `.alert-success`:

```css
/* File: resources/views/layouts/app.blade.php (di dalam tag <style>) */
.stats-grid { display: flex; gap: 20px; flex-wrap: wrap; margin-top: 20px; }
.stat-card { flex: 1; min-width: 160px; background: #f8fafc; border: 1px solid #e5e7eb; border-radius: 8px; padding: 20px; text-align: center; }
.stat-card .value { font-size: 32px; font-weight: bold; color: #1e3a8a; }
.stat-card .label { margin-top: 6px; color: #6b7280; font-size: 14px; }
```

> 📸 *Screenshot: Halaman Dashboard menampilkan tiga kartu statistik (Total Buku, Total Anggota, Peminjaman Aktif) dengan angka sesuai data di database.*

### Langkah 3 — `LoanController@report` Mengonsumsi `GET /api/loans`

Route laporan didaftarkan **sebelum** `Route::resource('loans', ...)` di `routes/web.php` — kalau didaftarkan sesudahnya, URL `/loans/report` akan tertangkap parameter `{loan}` milik route `loans.show` (Laravel mencocokkan route secara berurutan dari atas):

```php
// File: routes/web.php (sisipkan tepat sebelum baris Route::resource('loans', ...))
Route::get('/loans/report', [LoanController::class, 'report'])->name('loans.report');
Route::resource('loans', LoanController::class);
```

`LoanController.php` sudah ada sejak Pertemuan 7 (bukan file baru), jadi dua `use` berikut wajib ditambahkan manual di bagian atas file — kalau lupa, Laravel melempar error `Class "Http" not found` / `Class "ConnectionException" not found`:

```php
// File: app/Http/Controllers/LoanController.php (tambahkan di antara use yang sudah ada)
use Illuminate\Http\Client\ConnectionException;
use Illuminate\Support\Facades\Http;
```

Lalu tambahkan method `report()` (boleh diletakkan setelah method `index()`):

```php
// File: app/Http/Controllers/LoanController.php
public function report(Request $request)
{
    $body = ['data' => [], 'meta' => null];

    try {
        $response = Http::get(config('services.internal_api.base_url').'/api/loans', [
            'page' => $request->query('page', 1),
        ]);

        if ($response->successful()) {
            $body = $response->json();
        }
    } catch (ConnectionException $e) {
        // Server internal API (port 8011) tidak bisa dihubungi — tabel laporan tampil kosong.
    }

    return view('loans.report', [
        'loans' => $body['data'] ?? [],
        'meta' => $body['meta'] ?? null,
    ]);
}
```

`loans/report.blade.php` menampilkan data persis seperti `loans/index.blade.php`, tapi sumber datanya array hasil `json()`, bukan Collection Eloquent — karena itu diakses pakai notasi array (`$loan['member']['nama']`), bukan notasi objek (`$loan->member->nama`). Pagination-nya juga dibangun manual dari `meta.current_page`/`meta.last_page` hasil API, bukan `{{ $loans->links() }}`, karena `$loans` di sini array biasa bukan objek `LengthAwarePaginator`:

```html
<!-- File: resources/views/loans/report.blade.php -->
@extends('layouts.app')

@section('title', 'Laporan Peminjaman')

@section('content')
    <h1>Laporan Peminjaman</h1>
    <p>Data diambil langsung dari <code>GET /api/loans</code>, bukan query database langsung.</p>

    <table>
        <thead>
            <tr>
                <th>ID</th>
                <th>Anggota</th>
                <th>Petugas</th>
                <th>Buku</th>
                <th>Tgl Pinjam</th>
                <th>Tgl Kembali</th>
                <th>Status</th>
            </tr>
        </thead>
        <tbody>
            @forelse ($loans as $loan)
                <tr>
                    <td>{{ $loan['id'] }}</td>
                    <td>{{ $loan['member']['nama'] }}</td>
                    <td>{{ $loan['petugas']['name'] }}</td>
                    <td>
                        @foreach ($loan['books'] as $book)
                            {{ $book['judul'] }}@if (!$loop->last), @endif
                        @endforeach
                    </td>
                    <td>{{ $loan['tanggal_pinjam'] }}</td>
                    <td>{{ $loan['tanggal_kembali'] }}</td>
                    <td>{{ ucfirst($loan['status']) }}</td>
                </tr>
            @empty
                <tr>
                    <td colspan="7">Belum ada data peminjaman.</td>
                </tr>
            @endforelse
        </tbody>
    </table>

    @if ($meta && $meta['last_page'] > 1)
        <p>
            @if ($meta['current_page'] > 1)
                <a href="{{ route('loans.report', ['page' => $meta['current_page'] - 1]) }}">&laquo; Sebelumnya</a>
            @endif
            Halaman {{ $meta['current_page'] }} dari {{ $meta['last_page'] }}
            @if ($meta['current_page'] < $meta['last_page'])
                <a href="{{ route('loans.report', ['page' => $meta['current_page'] + 1]) }}">Selanjutnya &raquo;</a>
            @endif
        </p>
    @endif
@endsection
```

> 📸 *Screenshot: Halaman Laporan Peminjaman menampilkan tabel transaksi dari `GET /api/loans`, lengkap dengan navigasi halaman.*

### Langkah 4 — Redirect Login Diarahkan ke Dashboard

Sejak Pertemuan 8, `AuthController@login` mengarahkan pengguna ke `route('books.index')` setelah login berhasil — masuk akal saat itu karena Dashboard belum ada. Sekarang `/` adalah halaman Dashboard sungguhan, jadi redirect-nya perlu diperbarui supaya landing page setelah login konsisten:

```php
// File: app/Http/Controllers/AuthController.php (method login(), ganti baris return-nya)
return redirect()->intended(route('dashboard'))
    ->with('success', 'Login berhasil, selamat datang ' . Auth::user()->name . '.');
```

### Langkah 5 — Tautan Dashboard dan Laporan di Navbar

Tambahkan dua item navigasi baru di `partials/navbar.blade.php`, satu ke Dashboard (paling depan) dan satu ke Laporan (paling belakang), supaya kedua halaman baru bisa diakses tanpa mengetik URL manual:

```html
<!-- File: resources/views/partials/navbar.blade.php -->
<nav>
    <div class="brand">📚 Perpustakaan Digital Kampus</div>
    @auth
        <ul>
            <li><a href="{{ route('dashboard') }}" class="{{ request()->routeIs('dashboard') ? 'active' : '' }}">Dashboard</a></li>
            <li><a href="{{ route('books.index') }}" class="{{ request()->routeIs('books.*') ? 'active' : '' }}">Buku</a></li>
            @if (auth()->user()->role === 'admin')
                <li><a href="{{ route('categories.index') }}" class="{{ request()->routeIs('categories.*') ? 'active' : '' }}">Kategori</a></li>
            @endif
            <li><a href="{{ route('members.index') }}" class="{{ request()->routeIs('members.*') ? 'active' : '' }}">Anggota</a></li>
            <li><a href="{{ route('loans.index') }}" class="{{ request()->routeIs('loans.index') || request()->routeIs('loans.create') || request()->routeIs('loans.show') || request()->routeIs('loans.edit') ? 'active' : '' }}">Peminjaman</a></li>
            <li><a href="{{ route('loans.report') }}" class="{{ request()->routeIs('loans.report') ? 'active' : '' }}">Laporan</a></li>
        </ul>
        <div class="navbar-user">
            <span>{{ auth()->user()->name }} ({{ ucfirst(auth()->user()->role) }})</span>
            <form action="{{ route('logout') }}" method="POST" class="inline">
                @csrf
                <button type="submit" class="btn-logout">Logout</button>
            </form>
        </div>
    @endauth
    @guest
        <a href="{{ route('login') }}" class="{{ request()->routeIs('login') ? 'active' : '' }}">Login</a>
    @endguest
</nav>
```

Perhatikan class `active` untuk link "Peminjaman" sekarang dicek lewat beberapa `routeIs()` sekaligus (`loans.index`, `loans.create`, `loans.show`, `loans.edit`) — kalau tetap pakai `loans.*` seperti sebelumnya, link "Peminjaman" akan ikut aktif saat membuka `/loans/report`, padahal seharusnya link "Laporan" yang aktif di halaman itu.

### Langkah 6 — Seeder & Factory dengan Data Bergaya Indonesia

Ubah locale Faker ke Indonesia dulu, supaya semua Factory otomatis menghasilkan nama/alamat Indonesia tanpa perlu diatur satu-satu:

```bash
# File: .env dan .env.example
# Cari baris APP_FAKER_LOCALE=en_US, ganti jadi:
APP_FAKER_LOCALE=id_ID
```

`Book` dan `Member` butuh trait `HasFactory` supaya method `factory()` tersedia — kalau lupa salah satu, Laravel melempar `BadMethodCallException: Call to undefined method`:

```php
// File: app/Models/Book.php
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Book extends Model
{
    use HasFactory;

    // ...$fillable dan relationship yang sudah ada, tidak berubah
}
```

```php
// File: app/Models/Member.php
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Member extends Model
{
    use HasFactory;

    // ...$fillable dan relationship yang sudah ada, tidak berubah
}
```

Buat seluruh Seeder dan Factory yang dibutuhkan:

```bash
php artisan make:seeder CategorySeeder
php artisan make:seeder BookSeeder
php artisan make:factory BookFactory --model=Book
php artisan make:seeder MemberSeeder
php artisan make:factory MemberFactory --model=Member
php artisan make:seeder LoanSeeder
```

Isi `CategorySeeder` dengan 5 kategori tetap:

```php
// File: database/seeders/CategorySeeder.php
<?php

namespace Database\Seeders;

use App\Models\Category;
use Illuminate\Database\Seeder;

class CategorySeeder extends Seeder
{
    public function run(): void
    {
        $categories = [
            ['nama_kategori' => 'Novel', 'deskripsi' => 'Buku fiksi panjang dengan alur cerita dan karakter yang kompleks.'],
            ['nama_kategori' => 'Komik', 'deskripsi' => 'Buku bergambar dengan narasi cerita yang divisualisasikan.'],
            ['nama_kategori' => 'Sains', 'deskripsi' => 'Buku seputar ilmu pengetahuan alam dan penelitian ilmiah.'],
            ['nama_kategori' => 'Teknologi', 'deskripsi' => 'Buku seputar pemrograman, komputer, dan perkembangan teknologi.'],
            ['nama_kategori' => 'Sejarah', 'deskripsi' => 'Buku seputar peristiwa dan tokoh sejarah.'],
        ];

        foreach ($categories as $category) {
            Category::updateOrCreate(
                ['nama_kategori' => $category['nama_kategori']],
                $category
            );
        }
    }
}
```

Isi `BookFactory` — judul dikurasi manual per kategori (bukan `fake()->words()` yang menghasilkan Lorem gibberish tanpa makna), penerbit dikurasi dari daftar penerbit Indonesia asli (bukan `fake()->company()` yang menghasilkan nama badan usaha generik):

```php
// File: database/factories/BookFactory.php
<?php

namespace Database\Factories;

use App\Models\Category;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<\App\Models\Book>
 */
class BookFactory extends Factory
{
    protected array $judulPerKategori = [
        'Novel' => [
            'Laskar Pelangi', 'Bumi Manusia', 'Negeri 5 Menara', 'Ronggeng Dukuh Paruk',
            'Ayat-Ayat Cinta', 'Pulang', 'Cantik Itu Luka', 'Sang Pemimpi',
            'Perahu Kertas', 'Gadis Kretek', 'Amba', 'Saman',
        ],
        'Komik' => [
            'Si Juki: Kuliah Kerja Nyata', 'Garudayana', 'Wonder Wayang', 'Tuti and Friends',
            'Grojogan Sewu', 'Petualangan Panji', 'Nusantaranger', 'Kabayan City',
        ],
        'Sains' => [
            'Pengantar Fisika Dasar', 'Kimia Organik untuk Pemula', 'Biologi Sel dan Molekuler',
            'Astronomi: Menjelajah Alam Semesta', 'Ekologi dan Lingkungan Hidup',
            'Genetika Modern', 'Dasar-Dasar Termodinamika',
        ],
        'Teknologi' => [
            'Dasar Pemrograman dengan PHP', 'Algoritma dan Struktur Data', 'Jaringan Komputer Modern',
            'Kecerdasan Buatan untuk Pemula', 'Pengantar Basis Data', 'Rekayasa Perangkat Lunak',
            'Keamanan Siber Praktis',
        ],
        'Sejarah' => [
            'Sejarah Indonesia Modern', 'Proklamasi dan Perjuangan Bangsa', 'Jejak Kerajaan Nusantara',
            'Revolusi Indonesia 1945-1949', 'Perang Diponegoro', 'Sejarah Perdagangan Rempah',
            'Jalur Sutra dan Nusantara',
        ],
    ];

    protected array $penerbit = [
        'Gramedia Pustaka Utama', 'Mizan', 'Bentang Pustaka', 'Kepustakaan Populer Gramedia',
        'Balai Pustaka', 'Erlangga', 'Andi Publisher', 'Informatika Bandung',
        'Elex Media Komputindo', 'Rajawali Pers', 'Republika Penerbit', 'Kompas Media Nusantara',
    ];

    public function definition(): array
    {
        $category = Category::inRandomOrder()->first();
        $pool = $this->judulPerKategori[$category?->nama_kategori] ?? $this->judulPerKategori['Novel'];

        return [
            'judul' => fake()->randomElement($pool),
            'penulis' => fake()->name(),
            'penerbit' => fake()->randomElement($this->penerbit),
            'tahun_terbit' => fake()->numberBetween(1990, (int) date('Y')),
            'isbn' => fake()->unique()->isbn13(),
            'stok' => fake()->numberBetween(1, 10),
            'category_id' => $category?->id,
        ];
    }
}
```

```php
// File: database/seeders/BookSeeder.php
<?php

namespace Database\Seeders;

use App\Models\Book;
use Illuminate\Database\Seeder;

class BookSeeder extends Seeder
{
    public function run(): void
    {
        Book::factory()->count(20)->create();
    }
}
```

Isi `MemberFactory` — nama dari `firstName()`+`lastName()` (bukan `name()`, supaya tidak ikut kebawa gelar seperti "S.IP"/"Dr." yang kadang muncul dari Faker `id_ID`), email dibentuk manual dari nama + domain kampus:

```php
// File: database/factories/MemberFactory.php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

/**
 * @extends Factory<\App\Models\Member>
 */
class MemberFactory extends Factory
{
    public function definition(): array
    {
        $nama = fake()->firstName().' '.fake()->lastName();

        return [
            'nama' => $nama,
            'nim' => fake()->unique()->numerify('##########'),
            'email' => Str::slug($nama, '.').fake()->unique()->numerify('###').'@pens.ac.id',
            'nomor_telepon' => fake()->numerify('08##########'),
            'alamat' => fake()->address(),
            'status' => fake()->randomElement(['aktif', 'aktif', 'aktif', 'nonaktif']),
        ];
    }
}
```

```php
// File: database/seeders/MemberSeeder.php
<?php

namespace Database\Seeders;

use App\Models\Member;
use Illuminate\Database\Seeder;

class MemberSeeder extends Seeder
{
    public function run(): void
    {
        Member::factory()->count(15)->create();
    }
}
```

`LoanSeeder` ditulis manual (bukan lewat Factory) karena butuh kombinasi tanggal dan status yang saling konsisten (`tanggal_dikembalikan` hanya terisi kalau `status` `'dikembalikan'`), plus insert manual ke `loan_items` lewat relasi:

```php
// File: database/seeders/LoanSeeder.php
<?php

namespace Database\Seeders;

use App\Models\Book;
use App\Models\Loan;
use App\Models\Member;
use App\Models\User;
use Illuminate\Database\Seeder;

class LoanSeeder extends Seeder
{
    public function run(): void
    {
        $memberIds = Member::pluck('id');
        $userIds = User::pluck('id');
        $bookIds = Book::pluck('id');

        for ($i = 0; $i < 10; $i++) {
            $tanggalPinjam = now()->subDays(fake()->numberBetween(1, 30));
            $tanggalKembali = $tanggalPinjam->copy()->addDays(7);
            $status = fake()->randomElement(['dipinjam', 'dikembalikan', 'dikembalikan', 'terlambat']);

            $loan = Loan::create([
                'member_id' => $memberIds->random(),
                'user_id' => $userIds->random(),
                'tanggal_pinjam' => $tanggalPinjam->toDateString(),
                'tanggal_kembali' => $tanggalKembali->toDateString(),
                'tanggal_dikembalikan' => $status === 'dikembalikan'
                    ? $tanggalKembali->copy()->subDays(fake()->numberBetween(0, 3))->toDateString()
                    : null,
                'status' => $status,
            ]);

            $bookIds->random(fake()->numberBetween(1, min(3, $bookIds->count())))
                ->each(fn ($bookId) => $loan->loanItems()->create(['book_id' => $bookId]));
        }
    }
}
```

Terakhir, update nama 3 user hasil seeding di `UserSeeder` (dibuat Pertemuan 8) — sebelumnya masih label generik ("Admin Perpustakaan", "Petugas Satu", "Petugas Dua"), diganti jadi nama orang supaya terasa seperti data sungguhan. Email dan role tetap sama persis, cuma field `name` yang berubah:

```php
// File: database/seeders/UserSeeder.php (ganti nilai 'name' di tiga updateOrCreate())
User::updateOrCreate(
    ['email' => 'admin@pens.ac.id'],
    ['name' => 'Bambang Sutrisno', 'password' => Hash::make('password'), 'role' => 'admin']
);

User::updateOrCreate(
    ['email' => 'petugas1@pens.ac.id'],
    ['name' => 'Siti Rahmawati', 'password' => Hash::make('password'), 'role' => 'petugas']
);

User::updateOrCreate(
    ['email' => 'petugas2@pens.ac.id'],
    ['name' => 'Ahmad Fauzi', 'password' => Hash::make('password'), 'role' => 'petugas']
);
```

Orkestrasi urutan seeding didaftarkan di `DatabaseSeeder` — urutan ini wajib, salah urutan berarti Factory mencoba mengambil relasi dari tabel yang masih kosong:

```php
// File: database/seeders/DatabaseSeeder.php
public function run(): void
{
    $this->call([
        UserSeeder::class,
        CategorySeeder::class,
        BookSeeder::class,
        MemberSeeder::class,
        LoanSeeder::class,
    ]);
}
```

### Langkah 7 — Memperbaiki Tampilan Pagination

Buat view pagination custom (lihat penjelasan lengkap di bagian Materi "Kenapa Pagination Bawaan Laravel Bisa Tampil Rusak"):

```bash
mkdir -p resources/views/vendor/pagination
```

```html
<!-- File: resources/views/vendor/pagination/custom.blade.php -->
@if ($paginator->hasPages())
    <nav class="pagination">
        @if ($paginator->onFirstPage())
            <span class="pagination-disabled">&laquo; Sebelumnya</span>
        @else
            <a href="{{ $paginator->previousPageUrl() }}">&laquo; Sebelumnya</a>
        @endif

        @foreach ($paginator->getUrlRange(1, $paginator->lastPage()) as $page => $url)
            @if ($page == $paginator->currentPage())
                <span class="pagination-current">{{ $page }}</span>
            @else
                <a href="{{ $url }}">{{ $page }}</a>
            @endif
        @endforeach

        @if ($paginator->hasMorePages())
            <a href="{{ $paginator->nextPageUrl() }}">Berikutnya &raquo;</a>
        @else
            <span class="pagination-disabled">Berikutnya &raquo;</span>
        @endif
    </nav>
@endif
```

Daftarkan sebagai view pagination default lewat `Paginator::defaultView()` di `AppServiceProvider::boot()` — begitu didaftarkan di sini, **semua** pemanggilan `->links()` di seluruh project (books, members, categories, loans index) otomatis pakai view ini, tidak perlu mengubah Blade file CRUD satu-satu:

```php
// File: app/Providers/AppServiceProvider.php
<?php

namespace App\Providers;

use Illuminate\Pagination\Paginator;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        //
    }

    public function boot(): void
    {
        Paginator::defaultView('vendor.pagination.custom');
    }
}
```

Tambahkan CSS untuk class `.pagination` di layout master, langsung setelah CSS `.stat-card` yang ditambahkan di Langkah 2:

```css
/* File: resources/views/layouts/app.blade.php (di dalam tag <style>) */
.pagination { margin-top: 16px; display: flex; gap: 12px; align-items: center; flex-wrap: wrap; }
.pagination a { color: #2563eb; text-decoration: none; }
.pagination a:hover { text-decoration: underline; }
.pagination .pagination-current { font-weight: bold; color: #1e3a8a; }
.pagination .pagination-disabled { color: #9ca3af; }
```

> 📸 *Screenshot: Halaman `/books` menampilkan pagination berupa teks/link bersih (`« Sebelumnya`, nomor halaman, `Berikutnya »`) — bukan lagi ikon SVG raksasa tak berstyle.*

### Langkah 8 — Ujicoba

1. Jalankan `php artisan migrate:fresh --seed` — pastikan seluruh 5 Seeder selesai tanpa error.
2. Nyalakan **dua** server (`--port=8000` dan `--port=8011`, lihat Langkah 1).
3. Login di `http://127.0.0.1:8000/login` (kredensial dari `UserSeeder`, misalnya `admin@pens.ac.id` / `password`), pastikan redirect setelah login sekarang menuju Dashboard (bukan lagi `/books`), dan nama "Bambang Sutrisno" tampil di navbar.
4. Buka `/` — pastikan tiga kartu statistik terisi angka sesuai data hasil seeding (20 buku, 15 anggota, dan jumlah peminjaman aktif sesuai `LoanSeeder`).
5. Buka `/books` dan `/members` — pastikan judul buku, penulis, penerbit, nama anggota, dan email semuanya bergaya Indonesia (bukan Lorem/nama Barat), dan pagination di baris terakhir tabel tampil rapi berupa teks/link, bisa diklik pindah halaman.
6. Klik "Lihat Laporan Peminjaman" atau buka `/loans/report` — pastikan tabel 10 transaksi tampil lengkap dengan nama anggota, petugas, daftar buku, dan status.
7. Coba matikan server kedua (port `8011`) sementara server utama tetap jalan, lalu refresh Dashboard dan Laporan Peminjaman — pastikan **kedua** halaman tetap tampil (statistik angka `0`, tabel laporan kosong) alih-alih error 500, membuktikan `try/catch ConnectionException` bekerja.

> 📸 *Screenshot: Dua terminal berjalan berdampingan menampilkan `php artisan serve --port=8000` dan `php artisan serve --port=8011` masing-masing "Server running".*

---

## Checkpoint GitHub

```bash
git add .
git commit -m "[P-10] dashboard laporan seeder factory dan finalisasi project"
git push origin dev
```

---

## Tugas

Project sudah lengkap secara fungsional — tugas pertemuan ini murni finalisasi menjelang UAS:

1. **Merge branch `dev` ke `main`** — pastikan `main` mencerminkan kondisi project yang stabil dan siap didemokan.
2. **Buat skenario demo tertulis** (file terpisah, misalnya `DEMO.md`) berisi urutan langkah demo sesuai skenario UAS di `master-outline.md` bagian I (Pertemuan 11): login, tampilkan dashboard, tambah kategori, tambah buku, tambah anggota, buat peminjaman, lihat daftar peminjaman aktif, proses pengembalian, tampilkan laporan, demo API di Postman.
3. **Update `README.md`** di root repository `app-perpustakaan` — cara clone, install dependency (`composer install`), setup `.env` (termasuk `INTERNAL_API_URL`), migrate & seed database, serta cara menjalankan **dua** server (`php artisan serve --port=8000` dan `--port=8011`) yang dibutuhkan supaya Dashboard dan Laporan berfungsi.

**Yang dikumpulkan:**
- Link commit GitHub (branch `main`, hasil merge dari `dev`) yang berisi hasil tugas
- File `DEMO.md` berisi skenario demo tertulis
- `README.md` yang sudah diperbarui

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-09.md) | [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-11.md)*
