# Pertemuan 5 — Migration, Eloquent Model & CRUD

> **Sebelumnya:** `BookController` dan `CategoryController` sudah berisi logika nyata (validasi, redirect, flash message), tapi datanya masih array statis `private array $books` yang hidup dan mati bersama request — tidak pernah benar-benar tersimpan.
> **Pertemuan ini:** Array dummy digantikan tabel database sungguhan lewat Migration, dan diakses lewat Eloquent Model — sehingga data buku dan kategori yang ditambahkan hari ini masih ada besok.

---

## Capaian Pembelajaran

Setelah menyelesaikan pertemuan ini, mahasiswa mampu:
1. Memahami Migration sebagai version control untuk database schema
2. Membuat dan menjalankan migration untuk semua tabel sesuai desain database
3. Memahami Eloquent ORM, konsepnya, dan perbandingannya dengan query manual
4. Mengimplementasikan CRUD lengkap untuk `books` dan `categories` dengan data nyata

---

## Konsep: Mengapa Migration & ORM Ada?

Sampai Pertemuan 4, data buku dan kategori hidup sebagai array PHP yang dideklarasikan langsung di dalam Controller. Ini bekerja untuk keperluan belajar, tapi punya masalah fundamental: array itu **dibuat ulang dari nol setiap kali PHP menerima request baru**. Tambah satu buku lewat form, redirect ke halaman index, dan buku itu sudah hilang — karena proses PHP yang menangani request `POST /books` sudah selesai dan memori yang menyimpan array itu sudah dibuang, sementara proses baru yang menangani `GET /books` mulai dari array awal yang sama persis. Solusi paling dasar untuk masalah ini adalah database: sesuatu yang hidup di luar siklus hidup satu request, sehingga data yang ditulis oleh satu proses bisa dibaca oleh proses lain kapan saja. Tapi begitu database masuk ke gambaran, muncul dua masalah baru yang justru menjadi topik utama pertemuan ini.

Masalah pertama adalah **konsistensi struktur database antar developer dan antar environment**. Bayangkan sebuah tim berisi tiga mahasiswa mengerjakan project yang sama. Tanpa alat bantu, cara paling umum menyamakan struktur tabel adalah mengirim file `.sql` lewat chat, atau — lebih buruk — saling mengingatkan secara lisan "eh jangan lupa tabel `books` sekarang ada kolom `sampul` ya". Cara ini gagal dalam skala kecil sekalipun: satu orang lupa menjalankan `ALTER TABLE`, aplikasinya di laptop dia mendadak error `Unknown column 'sampul'`, dan butuh waktu untuk sadar penyebabnya cuma database yang tertinggal. Migration menyelesaikan ini dengan mengubah struktur schema jadi **kode yang di-commit ke Git**, persis seperti kode aplikasi lainnya. Setiap perubahan struktur — tabel baru, kolom baru, index baru — ditulis sebagai file PHP kecil yang bisa dijalankan (`up()`) atau dibatalkan (`down()`). Siapa pun yang `git pull` lalu menjalankan `php artisan migrate` otomatis punya struktur database yang identik, tanpa perlu koordinasi manual. Ini bukan konsep unik Laravel: Django punya `makemigrations`/`migrate`, Rails (Ruby) adalah framework yang mempopulerkan istilah "migration" untuk kasus ini, dan Express.js biasanya memakai library terpisah seperti Knex atau Sequelize untuk hal yang sama. Semua menyelesaikan masalah identik — schema database ikut ter-*version-control* seperti halnya kode.

Masalah kedua adalah **jarak konseptual antara tabel relasional dan objek dalam kode**. Database menyimpan data sebagai baris dan kolom; PHP (dan hampir semua bahasa pemrograman modern) bekerja dengan objek dan array asosiatif. Menjembatani dua dunia ini secara manual berarti menulis SQL mentah di mana-mana: `SELECT * FROM books WHERE id = ?`, memetakan hasilnya satu per satu ke variabel, lalu mengulang proses serupa untuk `INSERT`, `UPDATE`, `DELETE` — belum terhitung risiko *SQL Injection* kalau input user ditempel langsung ke string query tanpa parameter binding. **ORM (Object-Relational Mapping)** menyelesaikan ini dengan memetakan satu tabel ke satu class, dan satu baris data ke satu instance object dari class itu. Query yang tadinya SQL mentah berubah jadi pemanggilan method PHP biasa: `Book::find(1)`, `Book::create([...])`. Laravel memakai pendekatan ORM bernama **Active Record**, di mana object model itu sendiri tahu cara menyimpan dan mengambil dirinya dari database (`$book->save()`). Ini beda dengan pola **Data Mapper** yang dipakai misalnya oleh Doctrine (populer di ekosistem Symfony/PHP) atau Entity Framework (.NET), di mana object model murni menyimpan data sementara logika penyimpanan dipisah ke class "mapper" terpisah. Kedua pendekatan sama-sama valid; Active Record dipilih Laravel karena lebih ringkas untuk ditulis, meski Data Mapper dianggap lebih murni secara desain untuk aplikasi enterprise yang sangat besar.

Konsep ORM juga bukan monopoli Laravel. Django (Python) punya ORM bawaan yang filosofinya nyaris identik dengan Eloquent — `Book.objects.get(id=1)` di Django setara dengan `Book::find(1)` di Eloquent. Node.js punya beberapa pilihan populer seperti Prisma atau Sequelize. Spring Boot (Java) memakai Hibernate/JPA, yang secara historis justru menjadi salah satu inspirasi awal konsep ORM modern. Yang membedakan tiap implementasi biasanya soal seberapa "ajaib" ORM itu bekerja di belakang layar, dan seberapa mudah developer bisa "keluar" dari ORM dan menulis query manual kalau kebutuhannya sudah terlalu kompleks untuk direpresentasikan lewat method-method bawaan. Laravel sengaja dirancang supaya Eloquent tidak menjadi kotak tertutup: di baliknya ada **Query Builder** yang bisa dipanggil kapan saja untuk kasus yang lebih rumit, dan di balik Query Builder itu masih ada PDO — lapisan akses database PHP paling dasar. Jadi belajar Eloquent bukan belajar sesuatu yang eksklusif dan terpisah, melainkan belajar lapisan kenyamanan paling atas dari tumpukan yang bisa ditelusuri sampai ke SQL mentah kapan pun dibutuhkan.

Ada satu prinsip keamanan penting yang datang gratis begitu pindah dari array dummy ke Eloquent: **mass assignment protection**. Kalau seorang developer ceroboh menulis `Book::create($request->all())` tanpa berpikir panjang, dan form HTML-nya (atau permintaan API yang dipalsukan) menyertakan field tersembunyi seperti `role=admin` atau `id=999`, tanpa perlindungan apa pun database bisa saja menerima nilai itu mentah-mentah — celah yang dikenal sebagai *mass assignment vulnerability*. Eloquent menutup celah ini lewat properti `$fillable`: daftar putih (*whitelist*) kolom yang boleh diisi lewat `create()` atau `update()` massal. Kolom apa pun di luar daftar itu otomatis diabaikan Eloquent, bukan error, hanya diam-diam tidak disimpan. Alternatifnya adalah `$guarded` — daftar hitam (*blacklist*) kolom yang **tidak** boleh diisi massal, dengan semua kolom lain otomatis diizinkan. Pertemuan ini konsisten memakai `$fillable` karena sifatnya "aman secara default": kolom baru yang lupa didaftarkan otomatis tertolak, bukan otomatis diterima.

---

## Materi

### Migration: Perintah Dasar

Empat perintah artisan ini yang paling sering dipakai sepanjang siklus hidup sebuah migration:

```bash
php artisan make:migration create_books_table   # membuat file migration baru
php artisan migrate                             # menjalankan semua migration yang belum jalan
php artisan migrate:rollback                    # membatalkan batch migration terakhir
php artisan migrate:status                      # melihat migration mana yang sudah/belum jalan
```

`migrate:rollback` bekerja dengan memanggil method `down()` di file migration — makanya method itu penting ditulis dengan benar (biasanya `Schema::dropIfExists()`), bukan sekadar dibiarkan kosong. Ada juga `php artisan migrate:fresh`, yang **menghapus semua tabel** lalu menjalankan ulang seluruh migration dari nol — sangat berguna saat development ketika struktur tabel masih sering berubah dan data di dalamnya belum penting untuk dipertahankan, tapi **jangan pernah dijalankan di database production** karena sifatnya destruktif total.

### Urutan Migration: Parent Harus Ada Duluan

Tabel yang punya foreign key hanya bisa dibuat **setelah** tabel yang dirujuknya sudah ada. `books` merujuk `categories` lewat `category_id`, jadi migration `categories` harus dieksekusi lebih dulu. Laravel menentukan urutan eksekusi migration murni dari **nama file**, yang diawali timestamp otomatis saat file dibuat (`2026_07_16_012106_create_categories_table.php`). Kalau dua file dibuat dalam detik yang sama persis — misalnya lewat dua perintah `make:migration` yang dijalankan berurutan sangat cepat — timestamp keduanya bisa identik, dan Laravel jatuh ke urutan alfabetis sebagai *tie-breaker*. Ini persis yang terjadi saat migration `create_loans_table` dan `create_members_table` dibuat di praktikum ini: keduanya kebetulan mendapat timestamp yang sama, dan karena "loans" < "members" secara alfabet, Laravel mencoba membuat tabel `loans` (yang butuh `member_id` merujuk ke `members`) **sebelum** tabel `members` sendiri ada — migration gagal dengan error *"Foreign key constraint is incorrectly formed"*. Solusinya sederhana: ganti nama file supaya timestamp `members` lebih kecil dari `loans`. Ini bukan kesalahan langka; siapa pun yang membuat beberapa migration dengan foreign key saling terkait dalam waktu berdekatan berpotensi mengalami hal yang sama, jadi selalu periksa urutan file di `database/migrations/` setelah membuat beberapa migration sekaligus.

### Tipe Kolom Blueprint

`Schema::create()` menerima closure berisi objek `Blueprint`, yang menyediakan method untuk tiap tipe kolom:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
Schema::create('books', function (Blueprint $table) {
    $table->id();                                     // BIGINT UNSIGNED, primary key, auto increment
    $table->string('judul', 200);                     // VARCHAR(200)
    $table->text('deskripsi')->nullable();            // TEXT, boleh NULL
    $table->integer('stok')->default(1);              // INT, nilai default 1
    $table->year('tahun_terbit');                     // YEAR
    $table->enum('status', ['aktif', 'nonaktif']);    // ENUM dengan pilihan terbatas
    $table->foreignId('category_id')->constrained();  // BIGINT UNSIGNED + foreign key otomatis
    $table->timestamps();                             // created_at & updated_at otomatis
});
```

`foreignId('category_id')->constrained()` adalah bentuk singkat Laravel untuk tiga hal sekaligus: membuat kolom `category_id` bertipe `BIGINT UNSIGNED`, menambahkan foreign key constraint, dan menebak otomatis tabel yang dirujuk dari nama kolomnya (`category_id` → tabel `categories`). Kalau nama kolom tidak mengikuti konvensi ini, tabel rujukan bisa ditentukan manual lewat `->constrained('nama_tabel')`.

### Eloquent Model: Konvensi Nama

Eloquent menebak nama tabel dari nama Model **tanpa perlu dikonfigurasi manual**, selama konvensi penamaan diikuti dengan benar:

| Model (singular, PascalCase) | Tabel (plural, snake_case) |
|---|---|
| `Book` | `books` |
| `Category` | `categories` |
| `LoanItem` | `loan_items` |

Kalau suatu saat nama tabel tidak bisa mengikuti konvensi ini (misalnya karena migrasi dari database lama), Eloquent tetap bisa dipakai dengan mendefinisikan properti `protected $table = 'nama_tabel_custom';` di dalam class Model.

### `$fillable`: Kolom yang Boleh Diisi Massal

```php
// File: app/Models/Book.php
class Book extends Model
{
    protected $fillable = [
        'judul', 'penulis', 'penerbit', 'tahun_terbit',
        'isbn', 'stok', 'category_id', 'sampul',
    ];
}
```

Dengan `$fillable` terisi seperti ini, `Book::create($request->validated())` hanya akan menyimpan kolom yang namanya ada di daftar itu — meskipun `$request->validated()` kebetulan berisi key lain, key itu diam-diam diabaikan, bukan disimpan atau memicu error.

### CRUD Dasar Eloquent

Lima method ini menutupi hampir seluruh kebutuhan query di pertemuan ini:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
Book::all();                          // ambil semua baris, kembalikan Collection
Book::find(1);                        // ambil satu baris berdasar primary key, null kalau tidak ada
Book::findOrFail(1);                  // sama seperti find(), tapi lempar 404 otomatis kalau tidak ada
Book::where('stok', '>', 0)->get();   // query dengan kondisi, tetap kembalikan Collection
Book::create(['judul' => 'Contoh']);  // INSERT baru, kembalikan instance Model yang baru dibuat
$book->update(['stok' => 10]);        // UPDATE baris ini saja
$book->delete();                      // DELETE baris ini saja
```

`findOrFail()` dipakai konsisten di praktikum ini untuk method `show`, `edit`, `update`, dan `destroy` — dibanding `find()` biasa yang mengembalikan `null` diam-diam kalau ID tidak ditemukan (dan berpotensi memicu error samar di baris berikutnya saat `null` itu dipakai seakan-akan objek Model), `findOrFail()` langsung menghentikan request dengan halaman 404 standar Laravel, jauh lebih mudah dilacak penyebabnya.

### Pagination

Mengganti `Book::all()` dengan `Book::paginate(10)` sudah cukup untuk mendapatkan 10 data per halaman lengkap dengan metadata halaman (halaman saat ini, total halaman, dst) — Eloquent otomatis membaca parameter `?page=` dari URL:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
public function index()
{
    $books = Book::paginate(10);

    return view('books.index', compact('books'));
}
```

Karena `LengthAwarePaginator` yang dikembalikan `paginate()` tetap bisa di-loop persis seperti Collection biasa (`@foreach`/`@forelse` tidak perlu berubah sama sekali), yang perlu ditambahkan di Blade hanyalah satu baris di akhir tabel:

```blade
{{-- Contoh ilustrasi konsep --}}
{{ $books->links() }}
```

`links()` merender navigasi halaman (nomor halaman, tombol sebelumnya/berikutnya) memakai style Tailwind CSS secara default — tetap tampil sebagai HTML biasa meskipun project ini tidak memakai Tailwind, hanya saja tanpa styling visual khusus.

### Eloquent vs Query Builder: Kapan Pakai yang Mana?

Eloquent (`Book::where(...)`) sebenarnya lapisan tipis di atas **Query Builder** (`DB::table('books')->where(...)`) — keduanya menghasilkan SQL yang mirip, bedanya Eloquent mengembalikan instance Model (punya akses ke `$fillable`, dan nantinya di Pertemuan 7, relationship), sedangkan Query Builder mengembalikan `stdClass` polos yang lebih ringan tapi tidak punya fitur tambahan itu. Aturan praktis paling umum: pakai Eloquent selama masih berurusan dengan satu baris yang punya identitas jelas (butuh disimpan, diupdate, dihapus, atau punya relasi ke tabel lain) — yang mencakup hampir seluruh kebutuhan CRUD di project ini. Pindah ke Query Builder murni biasanya baru masuk akal untuk query agregat berat yang lintas banyak tabel dan tidak butuh representasi sebagai objek, misalnya laporan statistik gabungan — bukan kebutuhan pertemuan ini, tapi baik untuk diketahui batasannya sejak awal.

---

## Praktikum

> **Konteks:** Melanjutkan project `app-perpustakaan` dari Pertemuan 4.
> Pastikan branch aktif adalah `dev`.
> Di akhir praktikum ini kamu akan memiliki: enam tabel di database (`users` dengan kolom `role` tambahan, `categories`, `books`, `members`, `loans`, `loan_items`), lima Model Eloquent dengan `$fillable` lengkap, serta CRUD `books` dan `categories` yang sepenuhnya memakai data nyata dari database (bukan array lagi).
> Kalau butuh lihat lagi desain lengkap keenam tabel beserta relasinya sebelum mulai, buka **[Studi Kasus & Desain Database](./studi-kasus-database.md)**.

### Langkah 1 — Menambahkan Kolom `role` ke Migration `users`

Tabel `users` bawaan Laravel belum punya kolom `role` yang dibutuhkan desain database project ini. Karena migration ini belum pernah dijalankan di database manapun, kolom itu ditambahkan langsung ke file migration bawaan, bukan lewat migration baru terpisah.

```php
// File: database/migrations/0001_01_01_000000_create_users_table.php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->enum('role', ['admin', 'petugas'])->default('petugas');
    $table->rememberToken();
    $table->timestamps();
});
```

> Kalau migration ini **sudah pernah** dijalankan sebelumnya (ada baris di tabel `migrations`), menambahkan kolom harus lewat migration baru (`php artisan make:migration add_role_to_users_table`) yang memakai `Schema::table()` alih-alih `Schema::create()` — mengedit migration lama yang sudah jalan tidak akan mengubah database yang sudah ada.

### Langkah 2 — Membuat Migration Lima Tabel Lainnya

Setiap tabel dibuat lewat `make:migration`, dengan urutan pembuatan mengikuti urutan dependency: `categories` dan `books` (yang merujuknya) duluan, baru `members`, `loans`, dan `loan_items`.

```bash
php artisan make:migration create_categories_table
php artisan make:migration create_books_table
php artisan make:migration create_members_table
php artisan make:migration create_loans_table
php artisan make:migration create_loan_items_table
```

Isi masing-masing file sesuai desain database:

```php
// File: database/migrations/xxxx_create_categories_table.php
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('nama_kategori', 100);
    $table->text('deskripsi')->nullable();
    $table->timestamps();
});
```

```php
// File: database/migrations/xxxx_create_books_table.php
Schema::create('books', function (Blueprint $table) {
    $table->id();
    $table->string('judul', 200);
    $table->string('penulis', 100);
    $table->string('penerbit', 100);
    $table->year('tahun_terbit');
    $table->string('isbn', 20)->unique()->nullable();
    $table->integer('stok')->default(1);
    $table->foreignId('category_id')->constrained('categories');
    $table->string('sampul')->nullable();
    $table->timestamps();
});
```

```php
// File: database/migrations/xxxx_create_members_table.php
Schema::create('members', function (Blueprint $table) {
    $table->id();
    $table->string('nama', 100);
    $table->string('nim', 20)->unique();
    $table->string('email', 100)->unique();
    $table->string('nomor_telepon', 15);
    $table->text('alamat');
    $table->enum('status', ['aktif', 'nonaktif'])->default('aktif');
    $table->timestamps();
});
```

```php
// File: database/migrations/xxxx_create_loans_table.php
Schema::create('loans', function (Blueprint $table) {
    $table->id();
    $table->foreignId('member_id')->constrained('members');
    $table->foreignId('user_id')->constrained('users');
    $table->date('tanggal_pinjam');
    $table->date('tanggal_kembali');
    $table->date('tanggal_dikembalikan')->nullable();
    $table->enum('status', ['dipinjam', 'dikembalikan', 'terlambat'])->default('dipinjam');
    $table->timestamps();
});
```

```php
// File: database/migrations/xxxx_create_loan_items_table.php
Schema::create('loan_items', function (Blueprint $table) {
    $table->id();
    $table->foreignId('loan_id')->constrained('loans');
    $table->foreignId('book_id')->constrained('books');
    $table->timestamps();
});
```

> ⚠️ **Periksa urutan file** di `database/migrations/` sebelum lanjut. Kalau `create_members_table` dan `create_loans_table` (atau pasangan parent-child lainnya) kebetulan punya timestamp identik di nama filenya, ganti salah satu nama file supaya parent (`members`) berada lebih dulu secara alfabetis/timestamp dari child (`loans`) — lihat penjelasan lengkapnya di bagian Materi.

Jalankan migration, lalu verifikasi semua tabel terbuat tanpa error:

```bash
php artisan migrate
```

> 📸 *Screenshot: output terminal menampilkan enam baris migration dengan status `DONE`, tanpa ada yang `FAIL`.*

### Langkah 3 — Membuat Model Eloquent

```bash
php artisan make:model Category
php artisan make:model Book
php artisan make:model Member
php artisan make:model Loan
php artisan make:model LoanItem
```

Isi `$fillable` di setiap Model sesuai kolom yang boleh diisi lewat form (di luar `id`, `created_at`, `updated_at` yang dikelola otomatis):

```php
// File: app/Models/Category.php
class Category extends Model
{
    protected $fillable = ['nama_kategori', 'deskripsi'];
}
```

```php
// File: app/Models/Book.php
class Book extends Model
{
    protected $fillable = [
        'judul', 'penulis', 'penerbit', 'tahun_terbit',
        'isbn', 'stok', 'category_id', 'sampul',
    ];
}
```

Method dan relasi (`hasMany`, `belongsTo`) sengaja **belum ditambahkan** ke Model manapun di pertemuan ini — itu topik utama Pertemuan 7, setelah CRUD dasar benar-benar dikuasai dulu.

### Langkah 4 — Mengganti Data Dummy `CategoryController` dengan Eloquent

Seluruh array `private array $categories` dihapus. Setiap method sekarang memanggil Model `Category` langsung, dan `edit`/`update`/`destroy` yang sebelumnya cuma stub string kini diisi logika sungguhan:

```php
// File: app/Http/Controllers/CategoryController.php
use App\Models\Category;

public function index()
{
    $categories = Category::paginate(10);

    return view('categories.index', compact('categories'));
}

public function store(StoreCategoryRequest $request)
{
    $validated = $request->validated();

    Category::create($validated);

    return redirect()->route('categories.index')
        ->with('success', "Kategori \"{$validated['nama_kategori']}\" berhasil ditambahkan.");
}

public function edit(string $id)
{
    $category = Category::findOrFail($id);

    return view('categories.edit', compact('category'));
}

public function update(Request $request, string $id)
{
    $category = Category::findOrFail($id);

    $validated = $request->validate([
        'nama_kategori' => 'required|string|max:100',
        'deskripsi' => 'nullable|string',
    ]);

    $category->update($validated);

    return redirect()->route('categories.index')
        ->with('success', "Kategori \"{$validated['nama_kategori']}\" berhasil diperbarui.");
}

public function destroy(string $id)
{
    $category = Category::findOrFail($id);
    $category->delete();

    return redirect()->route('categories.index')
        ->with('success', 'Kategori berhasil dihapus.');
}
```

Karena Eloquent Model mengimplementasikan `ArrayAccess`, sintaks `$category['nama_kategori']` yang sudah dipakai di `categories/index.blade.php` dan `categories/create.blade.php` sejak Pertemuan 3-4 **tetap berfungsi tanpa perlu diubah** — meski sekarang `$category` adalah instance Model, bukan array asosiatif lagi. Yang perlu ditambahkan hanya `categories/edit.blade.php` (belum pernah dibuat sebelumnya karena `edit()` masih stub) dan satu baris `{{ $categories->links() }}` di `categories/index.blade.php` untuk menampilkan navigasi pagination.

```blade
{{-- File: resources/views/categories/edit.blade.php --}}
<form action="{{ route('categories.update', $category['id']) }}" method="POST">
    @csrf
    @method('PUT')

    <label for="nama_kategori">Nama Kategori</label>
    <input type="text" name="nama_kategori" id="nama_kategori" value="{{ old('nama_kategori', $category['nama_kategori']) }}">
    @error('nama_kategori')
        <div class="error">{{ $message }}</div>
    @enderror

    <label for="deskripsi">Deskripsi (opsional)</label>
    <textarea name="deskripsi" id="deskripsi" rows="4">{{ old('deskripsi', $category['deskripsi']) }}</textarea>

    <button type="submit" class="btn">Perbarui</button>
</form>
```

> 📸 *Screenshot: halaman `/categories` menampilkan data kategori yang benar-benar tersimpan di database, lengkap dengan navigasi pagination di bagian bawah tabel.*

### Langkah 5 — Mengganti Data Dummy `BookController` dengan Eloquent

Pola yang sama diterapkan ke `BookController`. Dropdown kategori di form create/edit sekarang diisi dari `Category::all()`, bukan array kategori dummy:

```php
// File: app/Http/Controllers/BookController.php
use App\Models\Book;
use App\Models\Category;

public function index()
{
    $books = Book::paginate(10);

    return view('books.index', compact('books'));
}

public function create()
{
    $categories = Category::all();

    return view('books.create', compact('categories'));
}

public function store(StoreBookRequest $request)
{
    $validated = $request->validated();

    Book::create($validated);

    return redirect()->route('books.index')
        ->with('success', "Buku \"{$validated['judul']}\" berhasil ditambahkan.");
}

public function show(string $id)
{
    $book = Book::findOrFail($id);

    return view('books.show', compact('book'));
}

public function edit(string $id)
{
    $book = Book::findOrFail($id);
    $categories = Category::all();

    return view('books.edit', compact('book', 'categories'));
}

public function update(Request $request, string $id)
{
    $book = Book::findOrFail($id);

    $validated = $request->validate([
        'judul' => 'required|string|max:200',
        'penulis' => 'required|string|max:100',
        'penerbit' => 'required|string|max:100',
        'tahun_terbit' => 'required|integer|min:1900|max:'.date('Y'),
        'isbn' => 'nullable|string|max:20',
        'stok' => 'required|integer|min:0',
        'category_id' => 'required|integer|exists:categories,id',
    ]);

    $book->update($validated);

    return redirect()->route('books.index')
        ->with('success', "Buku \"{$validated['judul']}\" berhasil diperbarui.");
}

public function destroy(string $id)
{
    $book = Book::findOrFail($id);
    $book->delete();

    return redirect()->route('books.index')
        ->with('success', 'Buku berhasil dihapus.');
}
```

Aturan `exists:categories,id` ditambahkan di validasi `category_id` (baik di `StoreBookRequest` maupun validasi inline `update()`) supaya form tidak bisa menyimpan buku dengan `category_id` yang sebenarnya tidak ada di tabel `categories` — kegagalan yang kalau tidak divalidasi di awal baru akan ketahuan lewat error database yang membingungkan.

Karena Model Book belum punya relasi `category()` (baru dipelajari Pertemuan 7), kolom kategori di `books/index.blade.php` dan `books/show.blade.php` untuk sementara menampilkan **`category_id` mentah**, bukan nama kategorinya:

```blade
{{-- File: resources/views/books/index.blade.php (potongan) --}}
<th>ID Kategori</th>
...
<td>{{ $book['category_id'] }}</td>
```

> 📸 *Screenshot: halaman `/books` menampilkan data buku nyata dari database, kolom "ID Kategori" berisi angka (bukan nama), dan navigasi pagination di bawah tabel.*

### Langkah 6 — Ujicoba

Jalankan server, lalu uji siklus CRUD penuh untuk kedua resource:

```bash
php artisan serve
```

1. Buka `/categories/create`, tambahkan kategori baru — pastikan redirect ke `/categories` dengan flash message sukses, dan data itu **masih ada** setelah halaman di-refresh manual (bukti data benar-benar tersimpan, bukan sekadar tampil dari hasil redirect).
2. Edit kategori yang baru dibuat lewat `/categories/{id}/edit`, ubah nama atau deskripsinya, submit, dan pastikan perubahan tersimpan.
3. Buka `/books/create`, pastikan dropdown "Kategori" berisi kategori yang benar-benar ada di database (bukan lagi tiga kategori dummy hardcoded). Tambahkan satu buku.
4. Buka detail buku (`/books/{id}`) dan halaman edit (`/books/{id}/edit`) — pastikan data yang ditampilkan/di-prefill sesuai dengan yang baru disimpan, termasuk kategori yang otomatis terpilih (`selected`) di dropdown.
5. Hapus buku itu lewat tombol "Hapus" di halaman index, pastikan baris itu hilang dan flash message tampil.
6. Coba submit form `/books/create` dalam keadaan kosong — pastikan semua pesan error validasi tetap tampil seperti sebelumnya (validasi tidak berubah, hanya sumber datanya).

---

## Checkpoint GitHub

```bash
git add .
git commit -m "[P-5] migrasi semua tabel dan crud books categories selesai"
git push origin dev
```

---

## Tugas

Lengkapi CRUD `members` mengikuti pola persis yang baru dikerjakan untuk `categories` di praktikum ini:

1. Buat `resources/views/members/create.blade.php`, `edit.blade.php`, dan `show.blade.php`, mengikuti struktur form yang sama seperti `categories/create.blade.php`/`edit.blade.php` (sesuaikan field: `nama`, `nim`, `email`, `nomor_telepon`, `alamat`, `status`).
2. Buat `StoreMemberRequest` (`php artisan make:request StoreMemberRequest`) dengan validasi: `nama` wajib, `nim` wajib dan unik, `email` wajib format email dan unik, `nomor_telepon` wajib, `alamat` wajib, `status` wajib salah satu dari `aktif`/`nonaktif`.
3. Isi seluruh method `MemberController` (`create`, `store`, `show`, `edit`, `update`, `destroy`) memakai `Member::create()`, `findOrFail()`, `update()`, `delete()` — ganti juga `index()` dari array dummy ke `Member::paginate(10)`.
4. Tambahkan fitur **search nama anggota** di halaman `/members`: input teks di atas tabel yang mengirim `GET` dengan parameter `?search=`, lalu di `MemberController@index` tambahkan `Member::when(request('search'), fn ($query, $search) => $query->where('nama', 'like', "%{$search}%"))->paginate(10)`. Pastikan hasil pencarian tetap mempertahankan parameter `search` saat berpindah halaman pagination (`{{ $members->appends(request()->query())->links() }}`).

**Yang dikumpulkan:**
- Link commit GitHub (branch `dev`) yang berisi hasil tugas
- Screenshot `/members` menampilkan data nyata dari database beserta hasil pencarian nama anggota

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-04.md) | [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-06.md)*
