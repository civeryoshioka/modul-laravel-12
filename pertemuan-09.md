# Pertemuan 9 — REST API Dasar

> **Sebelumnya:** Pertemuan 8 menyelesaikan autentikasi berbasis session — `AuthController`, middleware `auth`, dan `CheckAdminRole` melindungi seluruh halaman web, tapi semuanya masih berupa HTML yang cuma bisa dibaca browser.
> **Pertemuan ini:** Dedicated Controller API di `app/Http/Controllers/Api/` dibangun untuk `books`, `members`, `loans`, dan `stats` — semuanya mengembalikan JSON lewat API Resource, bukan `view()`, sehingga data aplikasi bisa dikonsumsi aplikasi lain (mobile, frontend terpisah, atau — seperti yang akan dilakukan di Pertemuan 10 — oleh aplikasi Laravel ini sendiri).

---

## Capaian Pembelajaran

Setelah menyelesaikan pertemuan ini, mahasiswa mampu:
1. Memahami konsep REST API dan perbedaan mendasarnya dengan aplikasi web biasa
2. Memahami JSON sebagai standar pertukaran data
3. Membuat API endpoint dengan dedicated Controller dan route `api`
4. Menggunakan API Resource untuk format response yang konsisten
5. Menguji API menggunakan Postman

---

## Konsep: Mengapa REST API Ada?

Sampai Pertemuan 8, aplikasi perpustakaan ini hanya punya satu jenis "klien": browser yang meminta halaman, lalu Laravel membalas dengan HTML lengkap siap tampil — ini disebut *server-rendered application*, di mana server bertanggung jawab penuh atas tampilan. Masalahnya muncul begitu ada kebutuhan klien lain yang ingin data yang sama tapi tidak butuh HTML sama sekali: aplikasi mobile kampus yang ingin menampilkan daftar buku dengan tampilan native Android/iOS, dashboard pihak ketiga yang ingin menarik statistik peminjaman, atau bahkan halaman Blade di aplikasi ini sendiri yang nanti (Pertemuan 10) ingin menampilkan data tanpa menulis ulang query Eloquent yang sama. HTML tidak berguna untuk klien-klien ini — mereka butuh data mentah dalam format yang bisa diproses program, bukan markup yang dirancang untuk ditampilkan visual oleh browser. Inilah masalah yang diselesaikan REST API: menyediakan satu sumber data yang sama, dalam format netral (JSON), yang bisa dikonsumsi klien apa pun tanpa peduli klien itu ditulis dalam bahasa atau platform apa.

REST (*Representational State Transfer*) bukan sekadar "route yang mengembalikan JSON" — dia adalah sekumpulan prinsip desain. Prinsip pertama, **statelessness**: setiap request API harus membawa semua informasi yang dibutuhkan server untuk memprosesnya, server tidak boleh mengandalkan "ingatan" dari request sebelumnya seperti session di aplikasi web. Prinsip kedua, **resource-based URL**: URL merepresentasikan *benda* (`/books`, `/members/5`), bukan *aksi* (`/getBooks`, `/deleteMember`) — aksi ditentukan lewat HTTP method, bukan lewat kata kerja di URL. Prinsip ketiga, **HTTP method sebagai aksi standar**: `GET` untuk membaca, `POST` untuk membuat, `PUT`/`PATCH` untuk memperbarui, `DELETE` untuk menghapus — klien mana pun yang paham HTTP otomatis paham konvensi ini tanpa perlu membaca dokumentasi kustom untuk setiap endpoint. Kombinasi tiga prinsip ini membuat API "bisa ditebak": begitu klien tahu `GET /api/books` mengembalikan daftar buku, dia bisa menebak `GET /api/books/1` mengembalikan satu buku dan `DELETE /api/books/1` menghapusnya, tanpa perlu dokumentasi tambahan.

Konsep ini sepenuhnya framework-agnostic — REST API bukan fitur eksklusif Laravel. Express.js (Node.js) membangun REST API dengan `app.get('/api/books', handler)` yang polanya nyaris identik dengan `Route::get('/books', ...)` di Laravel. Django REST Framework menyediakan `Serializer` yang perannya sama persis dengan API Resource di Laravel: mengubah objek model jadi struktur data yang siap dikirim sebagai JSON, sambil menyembunyikan field yang tidak seharusnya bocor ke klien (seperti password). Spring Boot (Java) punya `@RestController` yang otomatis mengubah return value method jadi JSON, mirip cara Laravel otomatis meng-JSON-kan return value API Resource. Kesamaan pola ini terjadi karena masalah yang diselesaikan sama di semua bahasa: bagaimana caranya server bahasa apa pun bisa "bicara" dengan klien bahasa apa pun. Jawabannya selalu sama — sepakati satu format data netral. JSON menang sebagai standar de facto (mengalahkan XML yang lebih verbose) karena strukturnya ringan, mudah dibaca manusia, dan hampir semua bahasa pemrograman modern punya parser JSON bawaan tanpa perlu library tambahan.

HTTP Status Code adalah bagian tak terpisahkan dari REST API yang sering diremehkan pemula: kode ini bukan detail teknis, melainkan bagian dari "bahasa" yang dipahami universal oleh klien API. `200 OK` berarti sukses membaca/memperbarui, `201 Created` secara spesifik berarti sukses *membuat* resource baru (bukan cuma `200` generik), `400 Bad Request` berarti request klien cacat secara struktural, `401 Unauthorized` berarti klien belum terbukti identitasnya, `403 Forbidden` berarti identitas klien diketahui tapi tidak punya izin, `404 Not Found` berarti resource yang diminta tidak ada, `422 Unprocessable Entity` secara spesifik berarti struktur request benar tapi datanya gagal validasi (field kosong, format salah, dsb — inilah yang otomatis dikembalikan Laravel saat validasi gagal di route API), dan `500 Internal Server Error` berarti server sendiri yang bermasalah, bukan salah klien. Klien API yang baik (termasuk Postman yang dipakai menguji di pertemuan ini) selalu memeriksa status code terlebih dahulu sebelum membaca isi body — status code memberi tahu *kategori* hasil dalam satu angka, tanpa perlu mem-parsing pesan teks yang bisa berubah-ubah.

---

## Materi

### `routes/api.php` dan Prefix `/api`

Laravel memisahkan route web dan route API ke dua file berbeda karena keduanya punya kebutuhan middleware yang berbeda — route web butuh session dan proteksi CSRF, route API pada dasarnya stateless dan tidak selalu berjalan di browser. Laravel 12 tidak lagi membuat `routes/api.php` secara otomatis di project baru (berbeda dari versi sebelumnya) — file ini didaftarkan manual di `bootstrap/app.php`:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
```

Setiap route yang didaftarkan di `routes/api.php` otomatis mendapat prefix `/api` di depannya — jadi `Route::get('/books', ...)` di file ini sebenarnya bisa diakses lewat `/api/books`, tanpa perlu menulis prefix itu berulang-ulang di setiap route.

### API Controller: Tidak `return view()`, tapi `return` Data

Controller API strukturnya mirip Controller web (menerima `Request`, memproses lewat Model), tapi cara meresponsnya berbeda total:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
class BookController extends Controller
{
    public function index()
    {
        return Book::all(); // Laravel otomatis meng-encode Collection jadi JSON
    }
}
```

`return view('books.index', ...)` di Controller web berarti "render template ini jadi HTML". `return Book::all()` di Controller API berarti "encode koleksi ini jadi JSON" — Laravel secara otomatis mendeteksi kalau return value sebuah Controller adalah Model, Collection, atau array, lalu meng-*encode*-nya jadi response JSON dengan header `Content-Type: application/json`, tanpa perlu memanggil `json_encode()` manual.

### API Resource: Kenapa Tidak Langsung `return Book::all()`?

Meski `return Book::all()` "berhasil", ini punya masalah serius: seluruh kolom Model ikut terekspos apa adanya, termasuk kolom yang seharusnya tidak pernah bocor ke klien luar (bayangkan kalau ini terjadi pada Model `User` — kolom `password` yang ter-hash pun ikut terkirim). API Resource menyelesaikan masalah ini dengan menjadi lapisan transformasi eksplisit antara Model dan response JSON:

```bash
php artisan make:resource BookResource
```

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
class BookResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'judul' => $this->judul,
            'category' => [
                'id' => $this->category?->id,
                'nama_kategori' => $this->category?->nama_kategori,
            ],
        ];
    }
}
```

Method `toArray()` adalah tempat memutuskan field mana yang boleh dikirim, dan bagaimana bentuknya — termasuk mengubah relasi Eloquent (`$this->category`) jadi struktur nested yang rapi, alih-alih membiarkan klien harus tahu cara meng-query relasi sendiri. Karena semua endpoint yang mengembalikan Book memakai `BookResource` yang sama, format response-nya **konsisten** — klien yang sudah paham struktur JSON dari `GET /api/books` otomatis paham struktur `GET /api/books/1`, tanpa kejutan.

### API Resource Collection dan Pagination

Memanggil `BookResource::collection($books)` alih-alih `new BookResource($book)` otomatis membungkus banyak item, dan kalau `$books` berasal dari `paginate()`, Laravel otomatis menyertakan metadata halaman:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
public function index()
{
    $books = Book::with('category')->paginate(10);

    return BookResource::collection($books); // otomatis membungkus data + links + meta
}
```

Response yang dihasilkan berbentuk `{"data": [...], "links": {...}, "meta": {...}}` — `data` berisi array item Resource, `links` berisi URL halaman pertama/terakhir/sebelumnya/berikutnya, dan `meta` berisi info seperti `current_page`, `total`, dan `per_page`. Klien (aplikasi mobile, frontend JS, atau halaman laporan Pertemuan 10 nanti) tidak perlu menghitung sendiri ada berapa halaman — semua informasi navigasi sudah tersedia di response.

---

## Praktikum

> **Konteks:** Melanjutkan project `app-perpustakaan` dari Pertemuan 8.
> Pastikan branch aktif adalah `dev`.
> Di akhir praktikum ini kamu akan memiliki: endpoint REST API lengkap untuk `books`, `members`, `loans`, dan `stats`, semuanya memakai API Resource untuk format response konsisten, dan sudah diuji berfungsi lewat Postman.
>
> **Catatan soal proteksi API:** middleware `auth` yang dibangun Pertemuan 8 berbasis session browser, dan tidak otomatis berlaku untuk `routes/api.php` — Laravel 12 tidak lagi menyertakan `routes/api.php` secara default (beda dari versi sebelumnya), file ini harus didaftarkan manual dulu. Autentikasi API yang sesungguhnya (lewat token, misalnya Laravel Sanctum) adalah topik tersendiri di luar cakupan pertemuan ini, jadi seluruh endpoint di bawah ini **sengaja dibiarkan publik** untuk sekarang — fokus pertemuan ini murni pada struktur REST API dan format response-nya.

### Langkah 1 — Mendaftarkan Routing API

`routes/api.php` belum ada sama sekali di project ini (Laravel 12 tidak membuatnya otomatis), jadi didaftarkan dulu di `bootstrap/app.php`:

```php
// File: bootstrap/app.php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    api: __DIR__.'/../routes/api.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

File `routes/api.php` lalu dibuat berisi seluruh endpoint sesuai struktur di bagian G `master-outline.md`:

```php
// File: routes/api.php
use App\Http\Controllers\Api\BookController;
use App\Http\Controllers\Api\LoanController;
use App\Http\Controllers\Api\MemberController;
use App\Http\Controllers\Api\StatsController;
use Illuminate\Support\Facades\Route;

Route::get('/books', [BookController::class, 'index']);
Route::get('/books/{id}', [BookController::class, 'show']);
Route::post('/books', [BookController::class, 'store']);
Route::put('/books/{id}', [BookController::class, 'update']);
Route::delete('/books/{id}', [BookController::class, 'destroy']);

Route::get('/members', [MemberController::class, 'index']);
Route::get('/members/{id}', [MemberController::class, 'show']);

Route::get('/loans', [LoanController::class, 'index']);
Route::post('/loans', [LoanController::class, 'store']);
Route::put('/loans/{id}/kembalikan', [LoanController::class, 'kembalikan']);

Route::get('/stats', [StatsController::class, 'index']);
```

Jalankan `php artisan route:list --path=api` untuk memastikan seluruh 11 route terbaca dengan prefix `/api`.

### Langkah 2 — Membuat Api Controller dan API Resource

Empat Controller API dibuat di folder terpisah `Api/` (bukan bercampur dengan Controller web) sesuai konvensi di `CLAUDE.md`, begitu juga tiga API Resource:

```bash
php artisan make:controller Api/BookController
php artisan make:controller Api/MemberController
php artisan make:controller Api/LoanController
php artisan make:controller Api/StatsController
php artisan make:resource BookResource
php artisan make:resource MemberResource
php artisan make:resource LoanResource
```

### Langkah 3 — Mengisi `BookResource`, `MemberResource`, `LoanResource`

Ketiga Resource mendefinisikan field yang boleh dikirim ke klien. `LoanResource` sekaligus mengubah relasi `member`, `user` (ditampilkan sebagai `petugas`), dan `loanItems.book` (ditampilkan sebagai `books`) jadi struktur nested yang rapi:

```php
// File: app/Http/Resources/BookResource.php
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'judul' => $this->judul,
        'penulis' => $this->penulis,
        'penerbit' => $this->penerbit,
        'tahun_terbit' => $this->tahun_terbit,
        'isbn' => $this->isbn,
        'stok' => $this->stok,
        'sampul' => $this->sampul,
        'category' => [
            'id' => $this->category?->id,
            'nama_kategori' => $this->category?->nama_kategori,
        ],
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

```php
// File: app/Http/Resources/LoanResource.php
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'member' => [
            'id' => $this->member?->id,
            'nama' => $this->member?->nama,
        ],
        'petugas' => [
            'id' => $this->user?->id,
            'name' => $this->user?->name,
        ],
        'tanggal_pinjam' => $this->tanggal_pinjam,
        'tanggal_kembali' => $this->tanggal_kembali,
        'tanggal_dikembalikan' => $this->tanggal_dikembalikan,
        'status' => $this->status,
        'books' => $this->loanItems->map(fn ($item) => [
            'id' => $item->book?->id,
            'judul' => $item->book?->judul,
        ]),
    ];
}
```

`MemberResource` mengikuti pola yang sama, cukup mengembalikan kolom-kolom `members` apa adanya karena tidak ada relasi yang perlu ditransformasi untuk kebutuhan endpoint `GET /api/members`.

### Langkah 4 — Endpoint Books: Filter dan Pagination

`Api\BookController@index` mendukung filter opsional lewat query string `?judul=` (pencarian sebagian, pakai `LIKE`) dan `?category_id=` (pencocokan persis), plus pagination:

```php
// File: app/Http/Controllers/Api/BookController.php
public function index(Request $request)
{
    $query = Book::with('category');

    if ($request->filled('judul')) {
        $query->where('judul', 'like', '%'.$request->query('judul').'%');
    }

    if ($request->filled('category_id')) {
        $query->where('category_id', $request->query('category_id'));
    }

    $books = $query->paginate(10);

    return BookResource::collection($books);
}
```

`store()` memakai ulang `StoreBookRequest` yang sudah dibuat di Pertemuan 3 untuk validasi — aturan validasi buku sama saja baik lewat form web maupun lewat API, jadi tidak perlu ditulis dua kali:

```php
// File: app/Http/Controllers/Api/BookController.php
public function store(StoreBookRequest $request)
{
    $book = Book::create($request->validated());

    return (new BookResource($book->load('category')))
        ->response()
        ->setStatusCode(201);
}
```

`->setStatusCode(201)` secara eksplisit menandai response sebagai "resource berhasil dibuat" — status code default Laravel untuk return value biasa adalah `200`, tapi konvensi REST mengharapkan `201` khusus untuk operasi create. Method `show`, `update`, dan `destroy` melengkapi CRUD penuh untuk `books` sesuai daftar endpoint yang diminta.

> 📸 *Screenshot: Postman memanggil `GET /api/books?judul=laskar` dan hanya mengembalikan buku yang judulnya cocok.*

### Langkah 5 — Endpoint Members: Read-Only dengan Pagination

Sesuai daftar endpoint di `master-outline.md`, `members` hanya butuh `GET` (list dan detail) — tidak ada create/update/delete lewat API di pertemuan ini:

```php
// File: app/Http/Controllers/Api/MemberController.php
public function index()
{
    $members = Member::paginate(10);

    return MemberResource::collection($members);
}

public function show(string $id)
{
    $member = Member::findOrFail($id);

    return new MemberResource($member);
}
```

### Langkah 6 — Endpoint Loans: Store dan Kembalikan

`Api\LoanController@store` strukturnya mirip `LoanController@store` di web (satu transaksi bisa mencakup banyak buku lewat `book_ids[]`), tapi dengan satu perbedaan penting: karena API ini publik dan tidak berjalan di atas session `auth()`, `user_id` (petugas pencatat) wajib dikirim eksplisit di body request, tidak bisa diisi otomatis dari `auth()->id()` seperti di web:

```php
// File: app/Http/Controllers/Api/LoanController.php
public function store(Request $request)
{
    $validated = $request->validate([
        'member_id' => 'required|integer|exists:members,id',
        'user_id' => 'required|integer|exists:users,id',
        'tanggal_pinjam' => 'required|date',
        'tanggal_kembali' => 'required|date|after_or_equal:tanggal_pinjam',
        'book_ids' => 'required|array|min:1',
        'book_ids.*' => 'integer|exists:books,id',
    ]);

    $loan = Loan::create([
        'member_id' => $validated['member_id'],
        'user_id' => $validated['user_id'],
        'tanggal_pinjam' => $validated['tanggal_pinjam'],
        'tanggal_kembali' => $validated['tanggal_kembali'],
    ]);

    foreach ($validated['book_ids'] as $bookId) {
        $loan->loanItems()->create(['book_id' => $bookId]);
    }

    $loan->refresh()->load(['member', 'user', 'loanItems.book']);

    return (new LoanResource($loan))->response()->setStatusCode(201);
}
```

`$loan->refresh()` sengaja dipanggil sebelum di-load ke Resource — `Loan::create()` mengembalikan objek di memori yang **belum tahu** nilai default kolom `status` (`'dipinjam'`) yang diisi otomatis oleh database saat insert, jadi tanpa `refresh()`, response pertama setelah create akan menampilkan `status: null` walau data di database sebenarnya sudah benar. `refresh()` mengambil ulang baris yang baru dibuat dari database supaya seluruh atribut (termasuk yang diisi default oleh database) ikut terbawa ke response.

Endpoint `kembalikan` — yang di versi web (`LoanController@kembalikan`) masih sengaja dibiarkan stub sebagai Tugas mandiri Pertemuan 7 — di sisi API justru diimplementasikan penuh karena secara eksplisit diminta di daftar endpoint pertemuan ini:

```php
// File: app/Http/Controllers/Api/LoanController.php
public function kembalikan(string $id)
{
    $loan = Loan::with(['member', 'user', 'loanItems.book'])->findOrFail($id);

    $loan->update([
        'status' => 'dikembalikan',
        'tanggal_dikembalikan' => now()->toDateString(),
    ]);

    return new LoanResource($loan);
}
```

> 📸 *Screenshot: Postman memanggil `PUT /api/loans/3/kembalikan` dan response menampilkan `status: "dikembalikan"` beserta `tanggal_dikembalikan` yang terisi tanggal hari ini.*

### Langkah 7 — Endpoint Stats

`Api\StatsController@index` tidak memakai API Resource karena responsnya bukan representasi satu jenis Model, melainkan ringkasan angka — cukup `response()->json()` langsung:

```php
// File: app/Http/Controllers/Api/StatsController.php
public function index()
{
    return response()->json([
        'total_buku' => Book::count(),
        'total_anggota' => Member::count(),
        'peminjaman_aktif' => Loan::where('status', 'dipinjam')->count(),
    ]);
}
```

Endpoint ini yang nanti dikonsumsi `dashboard.blade.php` di Pertemuan 10 untuk menampilkan kartu statistik.

### Langkah 8 — Ujicoba di Postman

1. Buat collection baru di Postman untuk `app-perpustakaan`, base URL `http://127.0.0.1:8000/api` (atau port yang dipakai `php artisan serve`).
2. Set header `Accept: application/json` di setiap request — tanpa header ini, sebagian klien tidak secara eksplisit meminta JSON dan bisa memengaruhi bagaimana Laravel merender error.
3. `GET /books` — pastikan response berbentuk `{"data": [...], "links": {...}, "meta": {...}}` dengan field kategori ter-nested.
4. `GET /books?judul=laskar` dan `GET /books?category_id=2` — pastikan hasil terfilter sesuai query.
5. `POST /books` dengan body JSON lengkap — pastikan status `201` dan buku baru langsung muncul di `GET /books`.
6. `POST /books` dengan body kosong — pastikan status `422` dan `errors` berisi pesan validasi per field.
7. `PUT /books/{id}` dan `DELETE /books/{id}` — pastikan perubahan/hapus berhasil.
8. `GET /books/999` (id yang tidak ada) — pastikan status `404`.
9. `GET /members` dan `GET /members/{id}` — pastikan pagination dan detail anggota tampil benar.
10. `GET /loans` — pastikan setiap item menampilkan `member`, `petugas`, dan daftar `books` dalam transaksi.
11. `POST /loans` dengan `member_id`, `user_id`, `tanggal_pinjam`, `tanggal_kembali`, dan `book_ids` — pastikan status `201` dan `status` transaksi baru terbaca `"dipinjam"` (bukan `null`).
12. `PUT /loans/{id}/kembalikan` — pastikan `status` berubah jadi `"dikembalikan"` dan `tanggal_dikembalikan` terisi.
13. `GET /stats` — pastikan tiga angka statistik sesuai jumlah data aktual di database.

> 📸 *Screenshot: Postman collection "app-perpustakaan" berisi seluruh request di atas, masing-masing menampilkan response body dan status code yang berhasil.*

---

## Checkpoint GitHub

```bash
git add .
git commit -m "[P-9] implementasi rest api books members loans dan stats"
git push origin dev
```

---

## Tugas

Dokumentasikan seluruh endpoint API yang sudah dibuat dalam bentuk tabel markdown, mencakup:

| Method | URL | Body (jika ada) | Contoh Response |
|---|---|---|---|
| GET | `/api/books` | - | `{...}` |
| ... | ... | ... | ... |

Tabel harus mencakup **semua 11 endpoint** yang dibangun di pertemuan ini (`books`, `members`, `loans`, `stats`), masing-masing disertai contoh response sungguhan (hasil copy-paste dari Postman, bukan dikarang manual) dan status code yang dikembalikan untuk skenario sukses maupun gagal (misalnya `422` untuk `POST /books` dengan data tidak valid).

**Yang dikumpulkan:**
- Link commit GitHub (branch `dev`) yang berisi hasil tugas
- File dokumentasi endpoint (boleh `API.md` terpisah di root repository, atau ditambahkan ke `README.md`)

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-08.md) | [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-10.md)*
