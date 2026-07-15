# Pertemuan 2 — Routing & Request Lifecycle

> **Sebelumnya:** Project Laravel 12 `app-perpustakaan` sudah terinstal, terhubung ke MySQL, dan sudah di-push ke GitHub dengan branch `main` dan `dev`.
> **Pertemuan ini:** Membangun seluruh definisi route untuk resource `books`, `categories`, `members`, dan `loans`, serta memahami perjalanan sebuah request dari browser hingga menjadi response.

---

## Capaian Pembelajaran

Setelah menyelesaikan pertemuan ini, mahasiswa mampu:
1. Memahami konsep routing dan mengapa dibutuhkan dibanding pendekatan PHP native
2. Membuat route dari closure sederhana hingga resource route
3. Memahami alur Request Lifecycle dari browser hingga response
4. Memahami konsep dan posisi Middleware dalam Request Lifecycle

---

## Konsep: Mengapa Routing Ada?

Bayangkan sebuah aplikasi PHP native tanpa framework. Setiap URL biasanya dipetakan langsung ke sebuah file fisik di server — `/produk.php`, `/produk-detail.php?id=5`, `/tambah-produk.php`. Semakin banyak fitur, semakin banyak file, dan semakin sulit mengontrol alur aplikasi. Sebagian developer mencoba mengakalinya dengan satu file `index.php` besar berisi tumpukan `if/elseif` atau `switch` yang membaca `$_GET['page']` atau `$_SERVER['REQUEST_URI']` untuk menentukan logika apa yang dijalankan. Pendekatan ini bekerja untuk aplikasi kecil, tapi cepat berubah menjadi *spaghetti code* ketika aplikasi tumbuh: sulit dibaca, sulit dicari, dan rawan bug karena tidak ada struktur yang konsisten.

Routing hadir sebagai solusi atas masalah ini. Alih-alih menerjemahkan URL menjadi file fisik atau logika kondisional yang bertebaran, framework menyediakan satu tempat terpusat (biasanya file konfigurasi khusus) di mana developer mendaftarkan setiap URL yang valid beserta kode yang harus dijalankan ketika URL tersebut diakses. Router bertugas mencocokkan URL yang diminta dengan daftar route yang terdaftar, lalu mengarahkan request ke handler yang sesuai. Ini memberi satu sumber kebenaran (*single source of truth*) tentang endpoint apa saja yang dimiliki aplikasi — sangat berguna ketika tim harus berkolaborasi atau ketika aplikasi perlu didokumentasikan.

Konsep ini bukan unik milik Laravel. Di Express.js (Node.js), routing didefinisikan dengan `app.get('/produk', handler)` atau `router.post('/produk', handler)`. Di Django (Python), routing disebut *URL dispatcher* yang memetakan pola URL ke *view function* di file `urls.py`. Di Spring Boot (Java), routing diwakili oleh anotasi `@GetMapping` atau `@PostMapping` di atas method controller. Semua framework backend modern pada dasarnya menyelesaikan masalah yang sama: bagaimana memetakan permintaan HTTP (method + URL) ke kode yang akan menanganinya, secara terstruktur dan mudah dilacak.

Router juga menjadi pintu masuk penting untuk menerapkan aturan lintas-request seperti middleware, grouping berdasarkan prefix, dan penamaan endpoint. Dengan mendaftarkan route secara eksplisit, framework bisa menampilkan seluruh peta aplikasi hanya dengan satu perintah (`php artisan route:list` di Laravel), sesuatu yang mustahil dilakukan secara rapi jika logika routing tersebar di banyak file `if/switch`. Semakin besar aplikasi, semakin terasa manfaat pendekatan terpusat ini — inilah alasan mengapa routing selalu menjadi salah satu fondasi pertama yang dipelajari saat belajar framework backend apa pun.

---

## Materi

### Routing di PHP Native vs Framework

Pada PHP native, kita sering melihat pola seperti berikut untuk menentukan halaman mana yang ditampilkan:

```php
// Pendekatan PHP native — tidak terpusat dan sulit dikembangkan
$page = $_GET['page'] ?? 'home';

if ($page === 'home') {
    include 'home.php';
} elseif ($page === 'produk') {
    include 'produk.php';
} elseif ($page === 'produk-detail') {
    include 'produk-detail.php';
} else {
    http_response_code(404);
    include '404.php';
}
```

Masalahnya: tidak ada validasi HTTP method (GET vs POST diperlakukan sama), tidak ada cara mudah menampilkan daftar seluruh endpoint, dan penamaan URL tercampur dengan logika bisnis. Laravel menggantikan pola ini dengan **Router** yang membaca definisi route dari file khusus dan mencocokkannya secara otomatis dengan URL yang diminta, lengkap dengan validasi HTTP method.

### `routes/web.php` vs `routes/api.php`

Laravel membedakan dua "pintu masuk" utama:

- **`routes/web.php`** — untuk route yang menghasilkan halaman HTML (Blade view), dilengkapi middleware bawaan seperti session dan proteksi CSRF.
- **`routes/api.php`** — untuk route yang menghasilkan JSON, tanpa session/CSRF, otomatis mendapat prefix `/api`, dan biasa dikonsumsi aplikasi mobile atau frontend terpisah.

Pertemuan ini berfokus pada `routes/web.php`. `routes/api.php` akan dibahas mendalam di Pertemuan 9.

### Route Closure dan Route Parameter

Bentuk paling sederhana sebuah route adalah closure — fungsi anonim yang langsung dieksekusi:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
Route::get('/tentang', function () {
    return 'Halaman Tentang Perpustakaan';
});
```

Route juga bisa menerima **parameter** dari URL. Parameter wajib ditulis dengan kurung kurawal `{}`, sedangkan parameter opsional ditambahkan tanda tanya `?`:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
Route::get('/books/{id}', function ($id) {
    return "Detail buku dengan id: {$id}";
});

Route::get('/books/{id}/{slug?}', function ($id, $slug = null) {
    return "id: {$id}, slug: " . ($slug ?? '-');
});
```

### HTTP Methods dan Maknanya

Setiap request HTTP membawa sebuah *method* yang menyatakan maksud dari request tersebut:

| Method | Makna | Contoh Penggunaan |
|---|---|---|
| GET | Mengambil data / menampilkan halaman | Menampilkan daftar buku |
| POST | Mengirim data baru | Menyimpan buku baru |
| PUT/PATCH | Memperbarui data yang sudah ada | Mengubah data buku |
| DELETE | Menghapus data | Menghapus buku |

Browser secara native hanya mendukung GET dan POST melalui `<form>`. Laravel menyiasatinya dengan *method spoofing* menggunakan `@method('PUT')` di Blade, yang menyisipkan input tersembunyi `_method` agar Laravel tahu maksud sebenarnya dari request tersebut.

### Route Naming dan `route()` Helper

Setiap route bisa diberi nama unik dengan `->name()`. Nama ini dipakai di kode (Controller, Blade) untuk menghasilkan URL tanpa harus menuliskan path secara manual — sehingga jika path berubah, kode lain tidak perlu diubah:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
Route::get('/books', [BookController::class, 'index'])->name('books.index');

// Dipakai di Blade atau Controller:
route('books.index'); // menghasilkan: /books
```

### Route Grouping dengan Prefix

Ketika beberapa route memiliki awalan URL yang sama, kita bisa mengelompokkannya dengan `Route::prefix()` agar tidak perlu menuliskan awalan berulang-ulang:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
Route::prefix('admin')->group(function () {
    Route::get('/dashboard', function () {
        return 'Admin Dashboard';
    });
});
// URL yang dihasilkan: /admin/dashboard
```

### Resource Route: 7 Route Otomatis

Pola CRUD (Create, Read, Update, Delete) sangat umum sehingga Laravel menyediakan `Route::resource()` untuk mendaftarkan **7 route sekaligus** dengan satu baris kode:

| HTTP Method | URI | Nama Route | Fungsi |
|---|---|---|---|
| GET | `/books` | `books.index` | Menampilkan daftar buku |
| GET | `/books/create` | `books.create` | Menampilkan form tambah buku |
| POST | `/books` | `books.store` | Menyimpan buku baru |
| GET | `/books/{book}` | `books.show` | Menampilkan detail satu buku |
| GET | `/books/{book}/edit` | `books.edit` | Menampilkan form edit buku |
| PUT/PATCH | `/books/{book}` | `books.update` | Memperbarui data buku |
| DELETE | `/books/{book}` | `books.destroy` | Menghapus buku |

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
Route::resource('books', BookController::class);
```

Satu baris ini setara dengan menulis 7 route manual — inilah kenapa resource route sangat efisien untuk aplikasi CRUD seperti sistem perpustakaan ini.

### Resource Route Parsial: `->only()` dan `->except()`

`Route::resource()` bersifat *all-or-nothing* — begitu dipanggil, ketujuh route otomatis terdaftar sekaligus. Padahal dalam praktiknya, tidak semua Controller butuh ketujuh method itu. Contoh nyata di aplikasi perpustakaan ini:

- Halaman **kategori** mungkin tidak perlu route `show` (detail kategori tersendiri), karena informasi kategori cukup ditampilkan langsung di daftar buku.
- Sebuah endpoint yang sifatnya *read-only* mungkin hanya butuh `index` dan `show`, tanpa `create`, `store`, `edit`, `update`, maupun `destroy`.

Untuk kasus seperti ini, Laravel menyediakan dua method tambahan yang dirantai (*chained*) setelah `Route::resource()`:

- **`->except([...])`** — daftarkan semua method resource, **kecuali** yang disebutkan.
- **`->only([...])`** — daftarkan **hanya** method yang disebutkan, sisanya tidak dibuat sama sekali.

```php
// Contoh ilustrasi konsep — bukan langkah praktikum

// Semua route categories terdaftar, KECUALI show
// (categories.index, categories.create, categories.store,
//  categories.edit, categories.update, categories.destroy)
Route::resource('categories', CategoryController::class)->except(['show']);

// HANYA index dan show yang terdaftar (route lain tidak ada sama sekali)
Route::resource('categories', CategoryController::class)->only(['index', 'show']);
```

Poin pentingnya: method yang tidak disertakan **benar-benar tidak terdaftar** di router — bukan sekadar disembunyikan. Ini berbeda dari sekadar mengosongkan isi method di Controller: dengan `->only()`/`->except()`, route-nya sendiri yang dihilangkan dari daftar `route:list`, sehingga tidak butuh method Controller yang bersangkutan sama sekali.

Response yang dihasilkan saat URI yang di-exclude tetap diakses tergantung apakah URI-nya masih dipakai oleh route lain:

- Kalau `->except(['show'])` seperti contoh `categories` di atas, URI `/categories/{category}` **masih terdaftar** untuk method `PUT`/`PATCH` (`update`) dan `DELETE` (`destroy`). Jadi mengakses `GET /categories/5` menghasilkan **`405 Method Not Allowed`** — Laravel mengenali URI-nya, tapi method GET tidak diizinkan di situ.
- Kalau seluruh route untuk satu URI dikecualikan (misalnya `->only(['index', 'create', 'store'])`, yang otomatis membuang `show`, `edit`, `update`, `destroy` sekaligus), maka `GET /categories/5` menghasilkan **`404 Not Found`**, karena URI itu memang sudah tidak dikenali router sama sekali.

`->only()` dan `->except()` cocok dipakai selama kamu **masih mengikuti konvensi URI dan nama resource** (`categories.index`, `categories.store`, dst) tapi tidak semua method dibutuhkan.

### Mendaftarkan Route Manual Satu per Satu

Untuk kasus yang lebih ekstrem — di mana kamu butuh kontrol penuh atas URI, nama route, atau ingin menerapkan middleware berbeda di tiap route — kamu bisa melewati `Route::resource()` sepenuhnya dan mendaftarkan route satu per satu, persis seperti route closure biasa yang sudah dipelajari sebelumnya:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
Route::get('/categories', [CategoryController::class, 'index'])->name('categories.index');
Route::get('/categories/create', [CategoryController::class, 'create'])->name('categories.create');
Route::post('/categories', [CategoryController::class, 'store'])->name('categories.store');
Route::get('/categories/{category}/edit', [CategoryController::class, 'edit'])->name('categories.edit');
Route::put('/categories/{category}', [CategoryController::class, 'update'])->name('categories.update');
Route::delete('/categories/{category}', [CategoryController::class, 'destroy'])->name('categories.destroy');
```

Pendekatan ini lebih verbose (lebih banyak baris kode), tapi memberi fleksibilitas penuh. Cocok dipakai ketika:

- URI tidak mengikuti pola resource standar, misalnya `/categories/daftar` alih-alih `/categories`
- Setiap route butuh middleware yang berbeda-beda (bukan middleware seragam untuk semua method)
- Controller tersebut memang hanya punya 1–2 endpoint saja, sehingga memakai `Route::resource()` justru berlebihan

**Kapan pakai yang mana?** Sebagai patokan: gunakan `Route::resource()` penuh untuk Controller CRUD standar (seperti `books` dan `loans` di project ini), gunakan `->only()`/`->except()` ketika sebagian method resource memang tidak relevan (seperti contoh `categories` tanpa `show` di atas), dan baru turun ke route manual satu per satu kalau kebutuhannya benar-benar di luar pola resource.

### `php artisan route:list`

Perintah ini menampilkan seluruh route yang terdaftar di aplikasi beserta method, URI, nama, dan controller/action yang menanganinya. Perintah ini sangat penting untuk memverifikasi bahwa semua route sudah terdaftar dengan benar sebelum lanjut ke tahap berikutnya.

### Request Lifecycle

Setiap kali browser mengirim request ke aplikasi Laravel, request tersebut melewati serangkaian tahapan berikut sebelum akhirnya menghasilkan response:

```
Browser
   │  (mengirim HTTP request)
   ▼
public/index.php          ← Entry point tunggal seluruh aplikasi
   │
   ▼
Kernel (HTTP)              ← Bootstrap aplikasi, load service provider
   │
   ▼
Middleware                 ← Filter request (auth, CSRF, dsb) sebelum sampai ke Router
   │
   ▼
Router                     ← Mencocokkan URL + method dengan route yang terdaftar
   │
   ▼
Controller / Closure       ← Menjalankan logika bisnis
   │
   ▼
Response                   ← View, JSON, redirect, dsb
   │
   ▼
Browser                    ← Menampilkan hasil ke pengguna
```

Semua request — tanpa terkecuali — masuk melalui **satu file**: `public/index.php`. File inilah yang menghidupkan (bootstrap) seluruh framework, lalu menyerahkan proses ke Kernel HTTP untuk diteruskan melalui pipeline middleware, router, hingga akhirnya sampai ke Controller yang sesuai.

### Middleware: Posisi dan Contoh Use Case

Middleware adalah lapisan yang **memeriksa atau memodifikasi request sebelum sampai ke Controller**, atau memodifikasi response sebelum dikirim ke browser. Posisinya persis di antara Router dan Controller pada diagram di atas. Contoh use case:

- `auth` — memastikan pengguna sudah login sebelum mengakses halaman tertentu
- `guest` — memastikan pengguna belum login (misalnya untuk halaman login itu sendiri)
- CSRF protection — memvalidasi token keamanan pada setiap request POST/PUT/DELETE dari form

Middleware akan dibahas lebih mendalam di Pertemuan 8, namun penting untuk memahami *posisinya* dalam Request Lifecycle sejak sekarang.

---

## Praktikum

> **Konteks:** Melanjutkan project `app-perpustakaan` dari pertemuan sebelumnya.
> Pastikan branch aktif adalah `dev`.
> Di akhir praktikum ini kamu akan memiliki: route resource lengkap untuk `books`, `members`, `loans` (7 route masing-masing) dan `categories` (6 route, tanpa `show`), masing-masing terhubung ke Controller kosong yang mengembalikan string dummy, dan terverifikasi lewat `php artisan route:list`.

### Langkah 1 — Membuat Resource Controller Kosong

Gunakan flag `--resource` agar Artisan otomatis membuatkan kerangka 7 method CRUD di dalam Controller.

```bash
php artisan make:controller BookController --resource
php artisan make:controller CategoryController --resource
php artisan make:controller MemberController --resource
php artisan make:controller LoanController --resource
```

Perintah ini menghasilkan 4 file Controller baru di `app/Http/Controllers/`, masing-masing sudah berisi 7 method kosong (`index`, `create`, `store`, `show`, `edit`, `update`, `destroy`) sesuai konvensi Resource Controller.

### Langkah 2 — Mengisi Method dengan Return String Dummy

Karena Model dan View belum dibuat (baru Pertemuan 4–5), setiap method untuk saat ini cukup mengembalikan string dummy yang menunjukkan method mana yang berhasil dipanggil. Ini berguna untuk memverifikasi bahwa routing sudah benar sebelum logika sesungguhnya ditambahkan.

```php
// File: app/Http/Controllers/BookController.php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class BookController extends Controller
{
    public function index()
    {
        return 'BookController@index';
    }

    public function create()
    {
        return 'BookController@create';
    }

    public function store(Request $request)
    {
        return 'BookController@store';
    }

    public function show(string $id)
    {
        return "BookController@show, id: {$id}";
    }

    public function edit(string $id)
    {
        return "BookController@edit, id: {$id}";
    }

    public function update(Request $request, string $id)
    {
        return "BookController@update, id: {$id}";
    }

    public function destroy(string $id)
    {
        return "BookController@destroy, id: {$id}";
    }
}
```

Lakukan pola yang sama untuk `CategoryController`, `MemberController`, dan `LoanController` — cukup ganti nama class dan string dummy sesuai nama Controller masing-masing.

Khusus `CategoryController`, **hapus method `show()`**. Merujuk ke struktur view yang sudah disepakati sejak awal modul, folder `categories/` hanya berisi `index`, `create`, `edit` — tidak ada halaman detail kategori tersendiri, karena info kategori cukup ditampilkan langsung di daftar buku. Karena method-nya memang tidak akan pernah dipakai, method tersebut dihapus dari Controller, bukan dibiarkan kosong tak terpakai:

```php
// File: app/Http/Controllers/CategoryController.php
// Method show() TIDAK dibuat — categories tidak punya halaman detail tersendiri
```

Khusus `LoanController`, tambahkan satu method tambahan `kembalikan()` di luar 7 method resource, karena fitur pengembalian buku bukan bagian dari CRUD standar:

```php
// File: app/Http/Controllers/LoanController.php
public function kembalikan(string $id)
{
    return "LoanController@kembalikan, id: {$id}";
}
```

### Langkah 3 — Mendaftarkan Resource Route

Buka `routes/web.php` dan daftarkan keempat resource menggunakan `Route::resource()`. Khusus `categories`, tambahkan `->except(['show'])` karena method `show()` sudah dihapus dari Controller-nya di Langkah 2 — kalau route `categories.show` tetap didaftarkan padahal method-nya tidak ada, Laravel akan melempar error `Method does not exist` saat route itu diakses. Tambahkan juga satu route kustom `PUT /loans/{id}/kembalikan` di luar pola resource untuk menangani aksi pengembalian buku.

```php
// File: routes/web.php
<?php

use App\Http\Controllers\BookController;
use App\Http\Controllers\CategoryController;
use App\Http\Controllers\LoanController;
use App\Http\Controllers\MemberController;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return view('welcome');
});

Route::resource('books', BookController::class);
Route::resource('categories', CategoryController::class)->except(['show']);
Route::resource('members', MemberController::class);
Route::resource('loans', LoanController::class);
Route::put('/loans/{id}/kembalikan', [LoanController::class, 'kembalikan'])
    ->name('loans.kembalikan');
```

Perhatikan bahwa route kustom `loans.kembalikan` didaftarkan **setelah** `Route::resource('loans', ...)`. Urutan ini tidak menimbulkan konflik karena URI-nya (`/loans/{id}/kembalikan`) berbeda dari seluruh URI resource `loans`.

### Langkah 4 — Verifikasi dengan `route:list`

Jalankan perintah berikut untuk memastikan seluruh route terdaftar dengan benar:

```bash
php artisan route:list
```

Pastikan output menampilkan 7 baris untuk `books`, `members`, dan `loans`, tapi **hanya 6 baris untuk `categories`** (tanpa `categories.show`, karena di-`except()`), ditambah satu baris tambahan `loans.kembalikan` — total 28 route custom aplikasi (di luar route bawaan framework seperti `storage/{path}` dan `up`).

> 📸 *Screenshot: output terminal `php artisan route:list` menampilkan seluruh route resource dan `loans.kembalikan`*

### Langkah 5 — Ujicoba di Browser

Jalankan development server:

```bash
php artisan serve
```

Buka beberapa URL berikut di browser untuk memastikan setiap route terhubung ke Controller yang benar:

- `http://127.0.0.1:8000/books` → menampilkan teks `BookController@index`
- `http://127.0.0.1:8000/categories` → menampilkan teks `CategoryController@index`
- `http://127.0.0.1:8000/members` → menampilkan teks `MemberController@index`
- `http://127.0.0.1:8000/loans` → menampilkan teks `LoanController@index`
- `http://127.0.0.1:8000/books/3` → menampilkan teks `BookController@show, id: 3`
- `http://127.0.0.1:8000/categories/3` → menampilkan halaman **405 Method Not Allowed**, karena `categories.show` (GET) memang tidak pernah didaftarkan di router, sementara URI yang sama masih dipakai oleh `categories.update` dan `categories.destroy`

> 📸 *Screenshot: browser menampilkan teks `BookController@index` saat mengakses `/books`, dan halaman 405 Method Not Allowed saat mengakses `/categories/3`*

> **Catatan:** `SESSION_DRIVER` di `.env` project ini sudah diset ke `file` (bukan `database`), sehingga tidak butuh migration untuk bisa menjalankan `php artisan serve` dan mengakses seluruh route di atas. Tabel database sesungguhnya baru akan dibuat di Pertemuan 5.

---

## Checkpoint GitHub

```bash
git add .
git commit -m "[P-2] setup routing resource semua tabel"
git push origin dev
```

---

## Tugas

Berdasarkan materi Route Grouping yang sudah dipelajari, buatlah satu route group baru dengan prefix `/admin` di `routes/web.php` yang berisi minimal satu route closure sederhana (misalnya `/admin/info` yang mengembalikan string informasi). Setelah itu, jalankan kembali `php artisan route:list` dan simpan hasilnya (screenshot atau salin teksnya) sebagai bukti route baru sudah terdaftar dengan prefix yang benar.

**Yang dikumpulkan:**
- Link commit GitHub (branch `dev`) yang berisi hasil tugas
- Dokumentasi (screenshot atau teks) output `php artisan route:list` yang menunjukkan route group `/admin`

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-01.md) | [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-03.md)*
