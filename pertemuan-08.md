# Pertemuan 8 — Authentication & Middleware

> **Sebelumnya:** Pertemuan 7 menyelesaikan seluruh relasi antar Model dan CRUD `loans`, tapi siapa pun bisa mengakses `/books`, `/categories`, `/members`, maupun `/loans` tanpa identitas apa pun — `user_id` di setiap transaksi peminjaman bahkan masih dipilih manual lewat dropdown karena belum ada konsep "user yang sedang login".
> **Pertemuan ini:** Sistem login/logout berbasis session dibangun dari nol tanpa starter kit, seluruh route CRUD dilindungi middleware `auth`, middleware custom `CheckAdminRole` dibuat untuk membedakan hak akses admin dan petugas, dan `user_id` di form peminjaman akhirnya diisi otomatis dari akun yang login.

---

## Capaian Pembelajaran

Setelah menyelesaikan pertemuan ini, mahasiswa mampu:
1. Memahami perbedaan autentikasi dan otorisasi sebagai konsep universal
2. Mengimplementasikan login/logout berbasis session tanpa starter kit
3. Membuat Middleware custom dan mendaftarkannya
4. Melindungi semua route CRUD dan mengimplementasikan role check sederhana

---

## Konsep: Mengapa Autentikasi dan Otorisasi Ada?

Setiap aplikasi yang menyimpan data lebih dari satu jenis pengguna cepat atau lambat akan menghadapi dua pertanyaan yang terdengar mirip tapi sebenarnya sangat berbeda: "siapa kamu?" dan "kamu boleh melakukan apa?". Pertanyaan pertama adalah **autentikasi** (*authentication*) — proses membuktikan identitas seseorang, biasanya lewat kombinasi email/username dan password yang hanya diketahui pemiliknya. Pertanyaan kedua adalah **otorisasi** (*authorization*) — setelah identitas terbukti, sistem memutuskan data dan aksi apa saja yang boleh diakses identitas itu. Di project ini, autentikasi menjawab "apakah orang ini benar-benar Petugas Satu yang terdaftar di tabel `users`?", sementara otorisasi menjawab "kalau iya, apakah Petugas Satu boleh menghapus kategori buku, atau itu cuma hak admin?". Mencampur dua konsep ini adalah kesalahan pemula yang sangat umum — kode yang cuma mengecek "apakah user sudah login" lalu langsung mengizinkan segala aksi, padahal login saja tidak seharusnya otomatis berarti punya akses penuh.

Kenapa dua hal ini tidak bisa digabung jadi satu pengecekan saja? Karena keduanya beroperasi di lapisan yang berbeda dan berubah dengan kecepatan berbeda pula. Autentikasi biasanya diperiksa sekali per sesi (saat login) dan hasilnya "menempel" ke seluruh request berikutnya lewat mekanisme seperti session atau token — inilah yang dilakukan middleware `auth` di pertemuan ini: cukup memastikan *ada* identitas yang sah, tanpa peduli identitas itu boleh melakukan apa. Otorisasi sebaliknya harus dievaluasi ulang di setiap aksi spesifik, karena hak akses bisa berbeda-beda tergantung resource yang disentuh — pengguna yang sama bisa saja boleh mengedit data miliknya sendiri tapi tidak boleh mengedit data orang lain, atau boleh melihat data tapi tidak boleh menghapusnya. Middleware `CheckAdminRole` di pertemuan ini adalah pengecekan otorisasi paling sederhana yang mungkin: bukan berdasarkan kepemilikan data, tapi berdasarkan satu atribut tetap (`role`) yang melekat pada user. Sistem otorisasi yang lebih matang (seperti Policy dan Gate di Laravel) bisa mengevaluasi aturan yang jauh lebih rumit, tapi prinsip dasarnya tetap sama: autentikasi dulu, baru otorisasi, dan keduanya adalah pertanyaan yang berbeda.

Cara membuktikan identitas (autentikasi) sendiri punya dua pendekatan besar yang dipakai luas di industri: **session-based** dan **token-based**. Session-based authentication — yang dipakai di pertemuan ini lewat `Auth` facade — bekerja dengan cara server menyimpan status "user X sedang login" di penyimpanan sisi server (file, database, atau Redis), lalu mengirim satu ID acak ke browser lewat cookie. Setiap request berikutnya, browser otomatis mengirim balik cookie itu, dan server mencocokkannya dengan data session yang tersimpan untuk tahu siapa yang sedang mengakses. Pendekatan ini cocok untuk aplikasi web tradisional seperti sistem perpustakaan ini, karena browser sudah otomatis menangani pengiriman cookie tanpa kode tambahan di sisi klien. Token-based authentication (paling umum lewat JWT — JSON Web Token) bekerja berbeda: server tidak menyimpan status apa pun, melainkan memberi klien sebuah token terenkripsi berisi identitas dan masa berlaku, dan klien wajib menyertakan token itu secara eksplisit di setiap request (biasanya lewat header `Authorization`). Token-based lebih cocok untuk API yang dikonsumsi aplikasi mobile atau frontend terpisah yang tidak berbagi cookie dengan server — inilah kenapa REST API di Pertemuan 9 nanti secara konseptual lebih dekat ke pendekatan token, meski project ini tidak mengimplementasikan token auth secara penuh untuk API-nya.

Middleware, komponen yang jadi rumah bagi pengecekan `auth` dan `CheckAdminRole` di pertemuan ini, sebenarnya bukan konsep yang lahir bersama Laravel — dia adalah pola arsitektur *request pipeline* yang dipakai luas di berbagai framework. Express.js (Node.js) punya `app.use(middleware)` yang bekerja persis dengan filosofi yang sama: fungsi yang berjalan di antara request masuk dan handler akhir, bisa memeriksa lalu meneruskan (`next()`) atau menghentikan request lebih awal. Django (Python) punya `MIDDLEWARE` di `settings.py` yang membungkus setiap request lewat serangkaian lapisan pemroses. Spring Boot (Java) punya `Filter` dan `Interceptor` yang menjalankan peran serupa di ekosistem Java. Kesamaan pola ini bukan kebetulan — middleware menyelesaikan masalah universal: banyak jenis pengecekan (autentikasi, otorisasi, logging, rate limiting, CORS) perlu dijalankan **sebelum** logika bisnis utama, dan tanpa middleware, setiap Controller harus menulis ulang pengecekan yang sama berkali-kali. Dengan middleware, pengecekan itu ditulis sekali dan ditempelkan ke route mana pun yang membutuhkannya lewat satu baris konfigurasi, persis seperti `Route::middleware(['auth', 'admin'])` yang dipakai di praktikum ini.

---

## Materi

### `Auth` Facade

Laravel menyediakan `Auth` facade sebagai pintu masuk utama untuk seluruh operasi autentikasi berbasis session, tanpa perlu menulis manual logika pengecekan password atau pengelolaan session:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
use Illuminate\Support\Facades\Auth;

Auth::attempt(['email' => $email, 'password' => $password]); // true/false
Auth::user();     // instance User yang sedang login, atau null
Auth::id();       // ID user yang sedang login, atau null
Auth::check();    // true kalau ada user yang login
Auth::logout();   // menghapus status login
```

`Auth::attempt()` adalah method paling penting: dia menerima array kredensial, mencari baris `users` dengan email yang cocok, lalu membandingkan password yang dikirim dengan hash password tersimpan lewat algoritma bcrypt — semua ini terjadi otomatis di balik satu pemanggilan method, termasuk perbandingan hash yang aman terhadap *timing attack*. Kalau cocok, Laravel otomatis membuat session baru dan mengembalikan `true`.

### CSRF Protection

Setiap form `POST`/`PUT`/`DELETE` di Laravel wajib menyertakan `@csrf`, dan ini bukan sekadar formalitas:

```blade
{{-- Contoh ilustrasi konsep — bukan langkah praktikum --}}
<form action="{{ route('login') }}" method="POST">
    @csrf
    {{-- ... --}}
</form>
```

`@csrf` mencetak satu input tersembunyi berisi token acak yang unik per session. Laravel menolak request `POST`/`PUT`/`DELETE` apa pun yang tidak menyertakan token yang cocok dengan token di session — ini mencegah **Cross-Site Request Forgery**: skenario di mana situs jahat mencoba mengirim form tersembunyi ke aplikasi kita memanfaatkan cookie session korban yang sedang login, tanpa sepengetahuan korban. Karena token CSRF hanya diketahui halaman yang benar-benar dimuat dari aplikasi kita, situs luar tidak bisa menebaknya.

### Middleware Bawaan: `auth`, `guest`, `throttle`

Laravel sudah menyediakan beberapa middleware siap pakai untuk kebutuhan umum:

| Middleware | Fungsi |
|---|---|
| `auth` | Menolak request kalau belum login, redirect ke route `login` |
| `guest` | Kebalikannya — menolak request kalau *sudah* login (dipakai di halaman login supaya user yang sudah login tidak bisa mengakses form login lagi) |
| `throttle` | Membatasi jumlah request dalam rentang waktu tertentu, mencegah brute-force |

Praktikum pertemuan ini memakai `auth` untuk melindungi seluruh route CRUD.

### Membuat Middleware Custom

Middleware custom dibuat lewat Artisan dan berisi satu method `handle()` yang menerima `$request` dan closure `$next`:

```php
// Contoh ilustrasi konsep — struktur dasar middleware
public function handle(Request $request, Closure $next): Response
{
    if (/* kondisi ditolak */) {
        abort(403);
    }

    return $next($request);
}
```

Memanggil `$next($request)` berarti "lanjutkan ke tujuan berikutnya di pipeline" (middleware lain, atau akhirnya Controller). Tidak memanggilnya sama sekali — seperti `abort(403)` di atas — berarti request dihentikan di situ juga, Controller tidak akan pernah dieksekusi.

### Mendaftarkan Middleware Alias

Middleware custom perlu didaftarkan dengan sebuah nama pendek (*alias*) sebelum bisa dipakai di route, dilakukan di `bootstrap/app.php` pada Laravel 12 (bukan `Kernel.php` seperti versi Laravel sebelumnya):

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'admin' => \App\Http\Middleware\CheckAdminRole::class,
    ]);
})
```

Setelah alias `admin` terdaftar, dia bisa dipakai di route mana pun cukup lewat string `'admin'`, tanpa perlu menuliskan nama class lengkapnya berulang-ulang.

### `@auth` dan `@guest` di Blade

Blade menyediakan directive khusus untuk menampilkan konten berbeda tergantung status login, tanpa perlu menulis `if (Auth::check())` manual di setiap view:

```blade
{{-- Contoh ilustrasi konsep — bukan langkah praktikum --}}
@auth
    <p>Halo, {{ auth()->user()->name }}</p>
@endauth

@guest
    <a href="{{ route('login') }}">Login</a>
@endguest
```

---

## Praktikum

> **Konteks:** Melanjutkan project `app-perpustakaan` dari Pertemuan 7.
> Pastikan branch aktif adalah `dev`.
> Di akhir praktikum ini kamu akan memiliki: sistem login/logout yang berfungsi penuh, seluruh route CRUD terlindungi middleware `auth`, middleware `CheckAdminRole` yang membatasi menu Kategori khusus admin, navbar yang menampilkan identitas user login, dan form peminjaman yang mengisi petugas secara otomatis dari akun yang login.

### Langkah 1 — Membuat `UserSeeder`

`UserSeeder` dibuat untuk mengisi 1 admin dan 2 petugas dengan password ter-hash, memakai `updateOrCreate()` (bukan `create()`) supaya seeder aman dijalankan berkali-kali tanpa membuat baris duplikat:

```bash
php artisan make:seeder UserSeeder
```

```php
// File: database/seeders/UserSeeder.php
use App\Models\User;
use Illuminate\Support\Facades\Hash;

public function run(): void
{
    User::updateOrCreate(
        ['email' => 'admin@pens.ac.id'],
        [
            'name' => 'Admin Perpustakaan',
            'password' => Hash::make('password'),
            'role' => 'admin',
        ]
    );

    User::updateOrCreate(
        ['email' => 'petugas1@pens.ac.id'],
        [
            'name' => 'Petugas Satu',
            'password' => Hash::make('password'),
            'role' => 'petugas',
        ]
    );

    User::updateOrCreate(
        ['email' => 'petugas2@pens.ac.id'],
        [
            'name' => 'Petugas Dua',
            'password' => Hash::make('password'),
            'role' => 'petugas',
        ]
    );
}
```

`use App\Models\User;` wajib ditambahkan di baris paling atas file (di bawah `namespace Database\Seeders;` yang sudah dibuat otomatis oleh Artisan) — tanpa baris ini, `User::updateOrCreate(...)` akan dicari PHP di namespace `Database\Seeders\User` yang tidak ada, dan seeder gagal jalan dengan error `Class "Database\Seeders\User" not found`. `Hash::make()` wajib dipakai — password tidak pernah disimpan sebagai teks biasa di database. `updateOrCreate()` mencari baris berdasarkan argumen pertama (`email`), dan kalau ketemu, memperbarui kolom di argumen kedua alih-alih membuat baris baru — ini penting karena project ini sudah punya data user manual dari pengujian Pertemuan 7 (`admin@test.local`) yang tidak boleh terduplikasi.

`DatabaseSeeder` dipanggil untuk mengorkestrasi `UserSeeder`:

```php
// File: database/seeders/DatabaseSeeder.php
public function run(): void
{
    $this->call([
        UserSeeder::class,
    ]);
}
```

```bash
php artisan db:seed --class=UserSeeder
```

### Langkah 2 — Membuat `AuthController`

`AuthController` dibuat manual (tanpa starter kit seperti Breeze/Jetstream) supaya mahasiswa memahami persis apa yang terjadi di balik proses login. Karena controller ini tidak berbentuk resource (tidak ada `index`/`store`/dst standar), dibuat sebagai controller polos:

```bash
php artisan make:controller AuthController
```

Perintah di atas menghasilkan class kosong, tapi **sudah otomatis menyertakan** `use Illuminate\Http\Request;` di bagian atas file — beda dari middleware/seeder yang stub-nya benar-benar kosong. Cek dulu isi file yang baru dibuat sebelum menambah apa pun, supaya baris itu tidak dituliskan dobel (menulis `use Illuminate\Http\Request;` dua kali di file yang sama membuat PHP gagal dengan error `Cannot use Illuminate\Http\Request as Request because the name is already in use`). Yang perlu ditambahkan secara manual hanya `use Illuminate\Support\Facades\Auth;` (satu baris saja, di bawah `use Illuminate\Http\Request;` yang sudah ada), lalu isi ketiga method di dalam class:

```php
// File: app/Http/Controllers/AuthController.php
// use Illuminate\Http\Request; ← baris ini SUDAH ADA dari hasil generate, jangan ditulis ulang
use Illuminate\Support\Facades\Auth; // ← baris ini yang perlu ditambahkan manual

public function showLogin()
{
    return view('auth.login');
}

public function login(Request $request)
{
    $credentials = $request->validate([
        'email' => 'required|email',
        'password' => 'required',
    ], [
        'email.required' => 'Email wajib diisi.',
        'email.email' => 'Format email tidak valid.',
        'password.required' => 'Password wajib diisi.',
    ]);

    if (! Auth::attempt($credentials, $request->boolean('remember'))) {
        return back()->withErrors([
            'email' => 'Email atau password salah.',
        ])->onlyInput('email');
    }

    $request->session()->regenerate();

    return redirect()->intended(route('books.index'))
        ->with('success', 'Login berhasil, selamat datang ' . Auth::user()->name . '.');
}

public function logout(Request $request)
{
    Auth::logout();

    $request->session()->invalidate();
    $request->session()->regenerateToken();

    return redirect()->route('login')
        ->with('success', 'Logout berhasil.');
}
```

`login()` dan `logout()` sama-sama menerima parameter `Request $request`, makanya `use Illuminate\Http\Request;` wajib ada — tapi karena Artisan sudah menyediakannya otomatis di stub, tugas mahasiswa di sini cuma memastikan baris itu **tidak dihapus**, bukan menambahkannya lagi. `$request->session()->regenerate()` setelah login berhasil mengganti ID session lama dengan yang baru — mencegah **session fixation attack**, skenario di mana penyerang sudah tahu ID session korban *sebelum* korban login, lalu memanfaatkannya begitu korban berhasil login. Pola yang sama berlaku terbalik saat logout: `invalidate()` menghapus seluruh data session, dan `regenerateToken()` mengganti token CSRF supaya form yang mungkin masih terbuka di tab lain tidak bisa dipakai lagi. `redirect()->intended()` mengembalikan user ke halaman yang tadinya ingin diakses sebelum diarahkan ke login (kalau ada), atau ke `/books` sebagai default.

### Langkah 3 — Membuat Halaman Login

`resources/views/auth/login.blade.php` dibuat sebagai halaman mandiri, file baru penuh (tidak memakai `layouts/app.blade.php`, karena navbar aplikasi hanya relevan untuk user yang sudah login — halaman ini butuh `<!DOCTYPE html>`, `<head>`, dan style sendiri):

```blade
{{-- File: resources/views/auth/login.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Login — Perpustakaan Digital Kampus</title>
    <style>
        body { font-family: sans-serif; margin: 0; background: #f1f5f9; display: flex; align-items: center; justify-content: center; min-height: 100vh; }
        .login-box { background: #fff; padding: 32px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,.1); width: 100%; max-width: 360px; }
        .login-box h1 { font-size: 20px; margin-top: 0; text-align: center; }
        label { display: block; margin-top: 12px; font-weight: bold; }
        input { width: 100%; padding: 8px; margin-top: 4px; box-sizing: border-box; }
        .error { color: #b91c1c; font-size: 14px; margin-top: 4px; }
        .alert-success { background: #d1fae5; color: #065f46; padding: 10px 14px; border-radius: 4px; margin-bottom: 16px; }
        .checkbox-row { display: flex; align-items: center; gap: 6px; margin-top: 12px; }
        .checkbox-row label { display: inline; margin-top: 0; font-weight: normal; }
        .btn { margin-top: 20px; width: 100%; padding: 10px; background: #2563eb; color: #fff; border: none; border-radius: 4px; cursor: pointer; font-size: 15px; }
    </style>
</head>
<body>
    <div class="login-box">
        <h1>📚 Login Perpustakaan</h1>

        @if (session('success'))
            <div class="alert-success">{{ session('success') }}</div>
        @endif

        <form action="{{ route('login') }}" method="POST">
            @csrf

            <label for="email">Email</label>
            <input type="email" name="email" id="email" value="{{ old('email') }}" autofocus>
            @error('email')
                <div class="error">{{ $message }}</div>
            @enderror

            <label for="password">Password</label>
            <input type="password" name="password" id="password">
            @error('password')
                <div class="error">{{ $message }}</div>
            @enderror

            <div class="checkbox-row">
                <input type="checkbox" name="remember" id="remember">
                <label for="remember">Ingat saya</label>
            </div>

            <button type="submit" class="btn">Login</button>
        </form>
    </div>
</body>
</html>
```

Blok `@if (session('success'))` di atas form penting jangan sampai kelewat — `AuthController@logout` mengirim flash message `'Logout berhasil.'`, dan halaman ini satu-satunya tempat pesan itu bisa tampil (karena setelah logout, user diarahkan balik ke `/login`, bukan ke halaman ber-layout `app.blade.php`).

> 📸 *Screenshot: halaman `/login` menampilkan form email, password, checkbox "Ingat saya", dan tombol Login.*

### Langkah 4 — Menambahkan Route Login/Logout dan Melindungi Route CRUD

`routes/web.php` disusun ulang total mengikuti struktur di bagian G `master-outline.md`: route login/logout tetap publik, sementara seluruh route CRUD dibungkus middleware `auth`. **Ganti seluruh isi `routes/web.php` dengan kode di bawah ini** (perhatikan baris `use` di paling atas — `AuthController` baru, wajib ditambahkan, kalau tidak PHP tidak akan kenal class itu dan seluruh halaman ikut error karena file ini dimuat di setiap request):

```php
// File: routes/web.php
use App\Http\Controllers\AuthController;
use App\Http\Controllers\BookController;
use App\Http\Controllers\CategoryController;
use App\Http\Controllers\LoanController;
use App\Http\Controllers\MemberController;
use Illuminate\Support\Facades\Route;

// Public
Route::get('/login', [AuthController::class, 'showLogin'])->name('login');
Route::post('/login', [AuthController::class, 'login']);
Route::post('/logout', [AuthController::class, 'logout'])->name('logout');

// Protected
Route::middleware(['auth'])->group(function () {
    Route::get('/', function () {
        return redirect()->route('books.index');
    });

    Route::resource('books', BookController::class);
    Route::resource('members', MemberController::class);
    Route::resource('loans', LoanController::class);
    Route::put('/loans/{id}/kembalikan', [LoanController::class, 'kembalikan'])
        ->name('loans.kembalikan');
});
```

Perhatikan: baris `Route::resource('categories', CategoryController::class)->except(['show']);` yang sejak Pertemuan 5 berdiri sendiri di luar group mana pun **sengaja dihilangkan dulu** dari potongan di atas — bukan lupa. Kalau baris lama itu masih tersisa di file kamu setelah paste kode di atas, **hapus manual**, karena akan ditambahkan lagi dalam bentuk yang sudah diproteksi di Langkah 5. Kalau baris lama dibiarkan nyangkut di luar group, route lama yang tidak terproteksi itu tetap aktif dan bisa "menang" saat dicocokkan Laravel — akibatnya middleware admin di Langkah 5 terlihat tidak berfungsi padahal sebenarnya cuma route lama yang belum dihapus.

Route `/` untuk sementara hanya redirect ke `/books` — halaman dashboard dengan statistik sungguhan baru dibangun di Pertemuan 10 setelah REST API tersedia, jadi belum ada `DashboardController` di titik ini.

> 📸 *Screenshot: mencoba membuka `/books` dalam kondisi belum login otomatis redirect ke `/login`.*

### Langkah 5 — Membuat Middleware `CheckAdminRole`

```bash
php artisan make:middleware CheckAdminRole
```

Isi `handle()` memeriksa kolom `role` milik user yang sedang login:

```php
// File: app/Http/Middleware/CheckAdminRole.php
public function handle(Request $request, Closure $next): Response
{
    if ($request->user()?->role !== 'admin') {
        abort(403, 'Halaman ini hanya bisa diakses oleh admin.');
    }

    return $next($request);
}
```

Middleware ini didaftarkan dengan alias `admin` di `bootstrap/app.php`:

```php
// File: bootstrap/app.php
use App\Http\Middleware\CheckAdminRole;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'admin' => CheckAdminRole::class,
    ]);
})
```

Berdasarkan deskripsi aktor di bagian D `master-outline.md` — admin "mengelola seluruh data" sementara petugas fokus ke "operasional harian (peminjaman, pengembalian)" — kelola Kategori dipilih sebagai contoh route yang dibatasi khusus admin, karena sifatnya administratif/konfigurasi, bukan pekerjaan harian petugas. Tambahkan blok ini **di dalam** group `auth` yang sudah dibuat di Langkah 4 (ditulis persis sebelum tanda kurung tutup `});` milik group `auth`, bukan di luarnya):

```php
// File: routes/web.php — masih di dalam Route::middleware(['auth'])->group(function () { ... });
Route::middleware(['admin'])->group(function () {
    Route::resource('categories', CategoryController::class)->except(['show']);
});
```

Route group ini tetap berada **di dalam** group `auth` di Langkah 4, jadi request harus lolos dua lapis pengecekan: login dulu (`auth`), baru role admin (`admin`). Kalau ditulis di luar group `auth` (misalnya diletakkan setelah `});` penutup group `auth`), route ini tetap terproteksi admin, tapi tidak lagi mewajibkan login lebih dulu — hasilnya bisa tidak konsisten dengan menu navbar yang mengasumsikan user sudah login.

> 📸 *Screenshot: user dengan role petugas yang mencoba membuka `/categories` mendapat halaman 403 Forbidden.*

### Langkah 6 — Menampilkan Identitas User di Navbar

`partials/navbar.blade.php` diubah untuk menyembunyikan seluruh menu dari user yang belum login, menampilkan menu Kategori hanya untuk admin, dan menambahkan nama, role, serta tombol logout. Atribut `class="{{ request()->routeIs(...) }}"` di setiap link (fitur *active state* dari Pertemuan 4) **tetap dipertahankan**, bukan dihapus:

```blade
{{-- File: resources/views/partials/navbar.blade.php --}}
<nav>
    <div class="brand">📚 Perpustakaan Digital Kampus</div>
    @auth
        <ul>
            <li><a href="{{ route('books.index') }}" class="{{ request()->routeIs('books.*') ? 'active' : '' }}">Buku</a></li>
            @if (auth()->user()->role === 'admin')
                <li><a href="{{ route('categories.index') }}" class="{{ request()->routeIs('categories.*') ? 'active' : '' }}">Kategori</a></li>
            @endif
            <li><a href="{{ route('members.index') }}" class="{{ request()->routeIs('members.*') ? 'active' : '' }}">Anggota</a></li>
            <li><a href="{{ route('loans.index') }}" class="{{ request()->routeIs('loans.*') ? 'active' : '' }}">Peminjaman</a></li>
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

Logout wajib berupa `<form method="POST">`, bukan link `<a>` biasa — route `/logout` didaftarkan sebagai `POST` (bukan `GET`) supaya logout tidak bisa dipicu tidak sengaja lewat prefetch browser atau crawler yang mengikuti semua link `GET` di halaman.

Elemen `.navbar-user` dan `.btn-logout` di atas belum punya style — tanpa ditambahkan dulu, tombol Logout akan tampil sebagai tombol putih polos bawaan browser, tidak menyatu dengan warna navbar. Tambahkan 3 baris berikut ke dalam `<style>` yang sudah ada di `layouts/app.blade.php`, persis di bawah baris `nav ul li a.active { ... }`:

```css
/* File: resources/views/layouts/app.blade.php — di dalam <style>, setelah aturan nav ul li a.active */
nav .navbar-user { display: flex; align-items: center; gap: 12px; color: #cbd5e1; font-size: 14px; }
nav .btn-logout { background: none; border: 1px solid #cbd5e1; color: #cbd5e1; padding: 4px 10px; border-radius: 4px; cursor: pointer; font-size: 14px; }
nav .btn-logout:hover { background: #1e40af; color: #fff; }
```

> 📸 *Screenshot: navbar menampilkan nama dan role user login (contoh: "Admin Perpustakaan (Admin)") beserta tombol Logout, dengan menu Kategori hanya muncul untuk role admin.*

### Langkah 7 — Menyederhanakan `LoanController@store`

Dropdown "Petugas" yang tadinya dipilih manual di `loans/create.blade.php` (workaround sementara dari Pertemuan 7) sekarang diganti `auth()->id()`, karena identitas user yang login sudah tersedia. Ini menyentuh dua method di `LoanController`: `create()` tidak lagi perlu mengambil daftar user, dan `store()` tidak lagi menerima `user_id` dari input form.

`create()` disederhanakan — hapus baris `$users = User::all();` dan hapus `use App\Models\User;` dari bagian atas file (sudah tidak dipakai method mana pun lagi):

```php
// File: app/Http/Controllers/LoanController.php
public function create()
{
    $members = Member::all();
    $books = Book::all();

    return view('loans.create', compact('members', 'books'));
}
```

`store()` tidak lagi memvalidasi `user_id` dari input, langsung diisi dari `auth()->id()`:

```php
// File: app/Http/Controllers/LoanController.php
public function store(Request $request)
{
    $validated = $request->validate([
        'member_id' => 'required|integer|exists:members,id',
        'tanggal_pinjam' => 'required|date',
        'tanggal_kembali' => 'required|date|after_or_equal:tanggal_pinjam',
        'book_ids' => 'required|array|min:1',
        'book_ids.*' => 'integer|exists:books,id',
    ]);

    $loan = Loan::create([
        'member_id' => $validated['member_id'],
        'user_id' => auth()->id(),
        'tanggal_pinjam' => $validated['tanggal_pinjam'],
        'tanggal_kembali' => $validated['tanggal_kembali'],
    ]);

    // ...
}
```

Validasi `user_id` dihapus karena kolom itu tidak lagi datang dari input form sama sekali — nilainya selalu dijamin valid karena `auth()->id()` hanya bisa berisi ID user yang benar-benar sedang login (dijamin middleware `auth` di route). Dropdown Petugas di `loans/create.blade.php` juga dihapus, diganti teks informatif:

```blade
{{-- File: resources/views/loans/create.blade.php --}}
<p><em>Petugas pencatat: {{ auth()->user()->name }} (otomatis dari akun yang login).</em></p>
```

> 📸 *Screenshot: form `/loans/create` tidak lagi punya dropdown Petugas, digantikan teks nama user yang sedang login.*

### Langkah 8 — Ujicoba

1. Buka `/books` dalam kondisi belum login — pastikan otomatis redirect ke `/login`.
2. Login dengan `admin@pens.ac.id` / `password` — pastikan redirect ke `/books` dengan flash message sukses, navbar menampilkan "Admin Perpustakaan (Admin)", dan menu Kategori muncul.
3. Buka `/categories` sebagai admin — pastikan bisa diakses normal.
4. Logout, lalu login dengan `petugas1@pens.ac.id` / `password` — pastikan menu Kategori **tidak** muncul di navbar.
5. Coba akses `/categories` langsung lewat URL sebagai petugas — pastikan mendapat halaman 403 Forbidden.
6. Buka `/loans/create` sebagai petugas — pastikan tidak ada dropdown Petugas, hanya teks nama user yang login.
7. Submit form peminjaman baru — pastikan di `/loans` kolom "Petugas" pada baris baru terisi otomatis dengan nama user yang tadi login, tanpa perlu memilih apa pun.
8. Login dengan password salah — pastikan muncul pesan error "Email atau password salah." tanpa membocorkan apakah emailnya valid atau tidak.
9. Klik Logout — pastikan kembali ke `/login`, dan mencoba membuka `/books` lagi sesudahnya kembali redirect ke login.

---

## Checkpoint GitHub

```bash
git add .
git commit -m "[P-8] implementasi autentikasi login logout dan middleware"
git push origin dev
```

---

## Tugas

Lengkapi dua fitur berikut di atas sistem autentikasi yang sudah berfungsi:

1. **Halaman profil petugas** — buat route `GET /profil` (terproteksi `auth`) dan `ProfileController@show` yang menampilkan nama, email, dan role user yang sedang login lewat `auth()->user()`. Tambahkan link "Profil" di navbar.
2. **Fitur ganti password** — tambahkan form di halaman profil dengan tiga input: password lama, password baru, dan konfirmasi password baru. Validasi lengkap: password lama harus cocok dengan yang tersimpan (gunakan `Hash::check()`), password baru wajib minimal 8 karakter dan harus sama dengan konfirmasinya (`confirmed` rule). Simpan password baru dengan `Hash::make()`, jangan pernah simpan sebagai teks biasa.

**Yang dikumpulkan:**
- Link commit GitHub (branch `dev`) yang berisi hasil tugas
- Screenshot halaman profil dan proses ganti password yang berhasil (disertai pesan sukses)

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-07.md) | [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-09.md)*
