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

Ada juga alasan yang lebih praktis: memisahkan "cara data disajikan" dari "cara data ditampilkan". `DashboardController@index` di pertemuan ini tidak tahu dan tidak peduli bagaimana `Api\StatsController` menghitung `total_buku` — dia cuma tahu bahwa `GET /api/stats` akan mengembalikan JSON berbentuk `{"total_buku": ..., "total_anggota": ..., "peminjaman_aktif": ...}`. Kalau suatu saat logic penghitungan statistik berubah (misalnya butuh cache, atau query yang lebih rumit), `DashboardController` tidak perlu disentuh sama sekali selama bentuk responsnya tetap sama — inilah manfaat *separation of concerns* yang sama yang mendasari kenapa Model, View, dan Controller dipisah di MVC, diterapkan sekali lagi di level yang lebih besar: pemisahan antara *penyedia data* (API) dan *konsumen data* (Blade).

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

Faker secara default memakai locale `en_US` — nama, alamat, dan nama perusahaan yang dihasilkan berbahasa Inggris/Amerika, kurang pas untuk demo aplikasi perpustakaan kampus Indonesia. Laravel menyediakan konfigurasi `APP_FAKER_LOCALE` di `.env` persis untuk kasus ini:

```bash
# File: .env
APP_FAKER_LOCALE=id_ID
```

Begitu diubah, seluruh pemanggilan `fake()->name()`, `fake()->firstName()`, `fake()->address()` di semua Factory otomatis menghasilkan data bergaya Indonesia — tanpa mengubah satu baris kode Factory pun, karena locale dibaca dari config `faker_locale` secara global.

Tapi locale saja tidak cukup untuk semua field. Faker tidak punya generator "judul buku" atau "nama penerbit" — method seperti `fake()->words()` (dipakai untuk teks acak generik) tetap menghasilkan rangkaian kata Latin tanpa makna, apa pun locale-nya, karena tidak ada data referensi "judul buku sungguhan" yang bisa diacak Faker. Untuk field seperti ini, solusinya bukan mencari fitur Faker yang lebih canggih, melainkan **kurasi manual**: menulis sendiri daftar judul buku dan nama penerbit yang realistis, lalu memilih secara acak dari daftar itu pakai `fake()->randomElement()`:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
protected array $judulPerKategori = [
    'Novel' => ['Laskar Pelangi', 'Bumi Manusia', 'Negeri 5 Menara', /* ... */],
    'Teknologi' => ['Dasar Pemrograman dengan PHP', 'Algoritma dan Struktur Data', /* ... */],
    // ...
];

public function definition(): array
{
    $category = Category::inRandomOrder()->first();
    $pool = $this->judulPerKategori[$category?->nama_kategori] ?? $this->judulPerKategori['Novel'];

    return [
        'judul' => fake()->randomElement($pool),
        'category_id' => $category?->id,
        // ...
    ];
}
```

Prinsip yang sama dipakai untuk email anggota: `Member` di studi kasus ini adalah mahasiswa kampus yang sama dengan petugas, jadi masuk akal kalau emailnya memakai domain kampus `@pens.ac.id` juga — bukan domain generik hasil `fake()->safeEmail()`. Email dibentuk manual dari nama lewat `Str::slug()`, ditambah angka acak supaya tetap unik:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
$nama = fake()->firstName().' '.fake()->lastName();

'email' => Str::slug($nama, '.').fake()->unique()->numerify('###').'@pens.ac.id',
```

Pelajaran di balik ini: Faker mempercepat pembuatan data dalam jumlah banyak, tapi bukan pengganti keputusan desain data — kapan pakai locale bawaan, kapan kurasi manual, dan kapan menggabungkan keduanya (nama dari locale + domain email dari konteks aplikasi) tetap keputusan yang harus diambil sendiri sesuai kebutuhan studi kasus.

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

---

## Praktikum

> **Konteks:** Melanjutkan project `app-perpustakaan` dari Pertemuan 9.
> Pastikan branch aktif adalah `dev`.
> Di akhir praktikum ini kamu akan memiliki: `dashboard.blade.php` yang menampilkan statistik dari `GET /api/stats`, `loans/report.blade.php` yang menampilkan laporan dari `GET /api/loans`, serta Seeder & Factory lengkap untuk mengisi database dengan data dummy realistis.
>
> **Catatan soal dua server:** karena `php artisan serve` cuma memproses satu request per waktu (lihat bagian Konsep), praktikum ini butuh **dua terminal** berjalan bersamaan — satu di port biasa (`8000`, atau port lain yang biasa dipakai) untuk browser, satu lagi khusus dipanggil dari dalam aplikasi sendiri lewat `Http::get()`.

### Langkah 1 — Menjalankan Server Kedua Khusus API Internal

Tambahkan variabel `INTERNAL_API_URL` di `.env`, menunjuk ke port yang berbeda dari server utama:

```bash
# File: .env
# Base URL instance kedua `php artisan serve` khusus panggilan API internal.
INTERNAL_API_URL=http://127.0.0.1:8011
```

Daftarkan sebagai config di `config/services.php` supaya bisa dipanggil lewat `config()` helper dari mana saja, bukan `env()` langsung di Controller (praktik yang tidak disarankan Laravel):

```php
// File: config/services.php
'internal_api' => [
    'base_url' => env('INTERNAL_API_URL', 'http://127.0.0.1:8011'),
],
```

Jalankan dua server di dua terminal terpisah:

```bash
# Terminal 1 — server utama, diakses browser
php artisan serve --port=8000

# Terminal 2 — server kedua, khusus dipanggil dari dalam aplikasi
php artisan serve --port=8011
```

### Langkah 2 — `DashboardController` Mengonsumsi `GET /api/stats`

```bash
php artisan make:controller DashboardController
```

```php
// File: app/Http/Controllers/DashboardController.php
use Illuminate\Http\Client\ConnectionException;
use Illuminate\Support\Facades\Http;

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
```

`try/catch` di sini menangkap dua skenario gagal sekaligus: response dengan status error (`successful()` bernilai `false`, misalnya API sendiri sedang bermasalah) **dan** kegagalan koneksi total (server kedua belum dinyalakan). Tanpa `catch (ConnectionException $e)`, skenario kedua akan lolos sebagai error 500 yang tidak tertangani — server kedua yang mati bukan cuma "response gagal", tapi request yang bahkan tidak pernah terkirim.

Route `/` yang sejak Pertemuan 8 hanya redirect sederhana ke `/books`, sekarang diarahkan ke Controller ini:

```php
// File: routes/web.php
Route::get('/', [DashboardController::class, 'index'])->name('dashboard');
```

`dashboard.blade.php` menampilkan tiga angka statistik dalam kartu sederhana:

```html
<!-- File: resources/views/dashboard.blade.php -->
@extends('layouts.app')

@section('content')
    <div class="stats-grid">
        <div class="stat-card">
            <div class="value">{{ $stats['total_buku'] }}</div>
            <div class="label">Total Buku</div>
        </div>
        {{-- ...kartu Total Anggota dan Peminjaman Aktif serupa... --}}
    </div>
@endsection
```

> 📸 *Screenshot: Halaman Dashboard menampilkan tiga kartu statistik (Total Buku, Total Anggota, Peminjaman Aktif) dengan angka sesuai data di database.*

### Langkah 3 — `LoanController@report` Mengonsumsi `GET /api/loans`

Route laporan didaftarkan **sebelum** `Route::resource('loans', ...)` — kalau didaftarkan sesudahnya, URL `/loans/report` akan tertangkap parameter `{loan}` milik route `loans.show` (Laravel mencocokkan route secara berurutan dari atas):

```php
// File: routes/web.php
Route::get('/loans/report', [LoanController::class, 'report'])->name('loans.report');
Route::resource('loans', LoanController::class);
```

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

`loans/report.blade.php` menampilkan data persis seperti `loans/index.blade.php`, tapi sumber datanya array hasil `json()`, bukan Collection Eloquent — karena itu diakses pakai notasi array (`$loan['member']['nama']`), bukan notasi objek (`$loan->member->nama`):

```html
<!-- File: resources/views/loans/report.blade.php -->
@foreach ($loans as $loan)
    <tr>
        <td>{{ $loan['member']['nama'] }}</td>
        <td>{{ $loan['petugas']['name'] }}</td>
        <td>
            @foreach ($loan['books'] as $book)
                {{ $book['judul'] }}@if (!$loop->last), @endif
            @endforeach
        </td>
        <td>{{ ucfirst($loan['status']) }}</td>
    </tr>
@endforeach
```

Pagination di halaman ini juga dibangun manual dari `meta.current_page` dan `meta.last_page` hasil API — bukan `{{ $loans->links() }}` seperti di halaman Eloquent biasa, karena `$loans` di sini array biasa, bukan objek `LengthAwarePaginator`.

> 📸 *Screenshot: Halaman Laporan Peminjaman menampilkan tabel transaksi dari `GET /api/loans`, lengkap dengan navigasi halaman.*

### Langkah 4 — Seeder & Factory

`Book` dan `Member` perlu trait `HasFactory` supaya method `factory()` tersedia — kalau lupa, Laravel melempar `BadMethodCallException: Call to undefined method`:

```php
// File: app/Models/Book.php
use Illuminate\Database\Eloquent\Factories\HasFactory;

class Book extends Model
{
    use HasFactory;
    // ...
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

`CategorySeeder` mengisi 5 kategori tetap, `BookSeeder` dan `MemberSeeder` masing-masing memanggil Factory-nya untuk membuat 20 buku dan 15 anggota dummy — keduanya pakai kurasi judul/penerbit dan email `@pens.ac.id` seperti dijelaskan di bagian Materi "Data Dummy yang Terasa Nyata". `LoanSeeder` sedikit berbeda — ditulis manual (bukan lewat Factory) karena butuh kombinasi tanggal dan status yang saling konsisten (`tanggal_dikembalikan` hanya terisi kalau `status` `'dikembalikan'`), plus insert manual ke `loan_items` lewat relasi:

```php
// File: database/seeders/LoanSeeder.php
public function run(): void
{
    $memberIds = Member::pluck('id');
    $userIds = User::pluck('id');
    $bookIds = Book::pluck('id');

    for ($i = 0; $i < 10; $i++) {
        $tanggalPinjam = now()->subDays(fake()->numberBetween(1, 30));
        $status = fake()->randomElement(['dipinjam', 'dikembalikan', 'dikembalikan', 'terlambat']);

        $loan = Loan::create([
            'member_id' => $memberIds->random(),
            'user_id' => $userIds->random(),
            'tanggal_pinjam' => $tanggalPinjam->toDateString(),
            'tanggal_kembali' => $tanggalPinjam->copy()->addDays(7)->toDateString(),
            'tanggal_dikembalikan' => $status === 'dikembalikan'
                ? $tanggalPinjam->copy()->addDays(fake()->numberBetween(4, 7))->toDateString()
                : null,
            'status' => $status,
        ]);

        $bookIds->random(fake()->numberBetween(1, 3))
            ->each(fn ($bookId) => $loan->loanItems()->create(['book_id' => $bookId]));
    }
}
```

Orkestrasi urutan seeding didaftarkan di `DatabaseSeeder`:

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

### Langkah 5 — Ujicoba

1. Jalankan `php artisan migrate:fresh --seed` — pastikan seluruh 5 Seeder selesai tanpa error.
2. Nyalakan **dua** server (`--port=8000` dan `--port=8011`, lihat Langkah 1).
3. Login di `http://127.0.0.1:8000/login`, pastikan redirect setelah login sekarang menuju Dashboard (bukan lagi `/books`).
4. Buka `/` — pastikan tiga kartu statistik terisi angka sesuai data hasil seeding (20 buku, 15 anggota, dan jumlah peminjaman aktif sesuai `LoanSeeder`), dan nama petugas yang login (misalnya "Bambang Sutrisno") tampil di navbar.
5. Klik "Lihat Laporan Peminjaman" atau buka `/loans/report` — pastikan tabel 10 transaksi tampil lengkap dengan nama anggota, petugas, daftar buku (semuanya nama Indonesia, sesuai kurasi `BookFactory`/`MemberFactory`), dan status.
6. Coba matikan server kedua (port `8011`) sementara server utama tetap jalan, lalu refresh Dashboard dan Laporan Peminjaman — pastikan **kedua** halaman tetap tampil (statistik angka `0`, tabel laporan kosong) alih-alih error 500, membuktikan `try/catch ConnectionException` di `DashboardController` dan `LoanController@report` bekerja menangkap kegagalan koneksi total, bukan cuma respons berstatus gagal.

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
