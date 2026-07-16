# Pertemuan 7 — Eloquent Relationships

> **Sebelumnya:** UTS (Pertemuan 6) menandai `books` dan `categories` sudah CRUD penuh dengan data nyata, tapi keduanya masih berdiri sendiri-sendiri — kolom `category_id` di tabel buku tampil sebagai angka mentah, bukan nama kategorinya.
> **Pertemuan ini:** Setiap Model dihubungkan lewat method relasi (`hasMany`/`belongsTo`), N+1 Query Problem didemonstrasikan lalu diselesaikan dengan eager loading, dan CRUD `loans` dibangun dari nol karena satu-satunya tabel yang benar-benar butuh menyatukan tiga tabel sekaligus (anggota, petugas, buku) dalam satu transaksi.

---

## Capaian Pembelajaran

Setelah menyelesaikan pertemuan ini, mahasiswa mampu:
1. Memahami mengapa relasi perlu didefinisikan di level Model, tidak hanya di database
2. Mengimplementasikan `hasMany()` dan `belongsTo()` di Eloquent
3. Memahami N+1 Query Problem dan mengatasinya dengan eager loading
4. Membangun CRUD Loans yang melibatkan relasi ke beberapa tabel sekaligus

---

## Konsep: Mengapa Relasi Perlu Didefinisikan di Model?

Sejak Pertemuan 5, foreign key `category_id` di tabel `books` sudah ada di database — MySQL sendiri sudah tahu bahwa setiap baris buku "merujuk" ke satu baris kategori, dan bahkan sudah menolak insert yang menyebut `category_id` tidak valid berkat *constraint* itu. Pertanyaan wajarnya: kalau database sudah tahu relasinya, kenapa masih perlu menulis method `category()` di Model `Book`? Jawabannya adalah karena foreign key di database dan relationship di Eloquent menyelesaikan dua masalah yang berbeda. Foreign key adalah aturan **integritas data** — dia memastikan tidak ada buku yang menunjuk ke kategori yang tidak ada, murni soal konsistensi angka di kolom `category_id`. Tapi begitu data itu perlu dipakai di kode PHP — misalnya menampilkan "Fiksi" alih-alih angka `3` di halaman daftar buku — sesuatu harus menerjemahkan angka itu jadi baris data kategori yang sesungguhnya, dan itu bukan tanggung jawab database, melainkan tanggung jawab lapisan aplikasi. Relationship di Eloquent adalah lapisan penerjemah itu: sekali didefinisikan lewat `$book->category()`, seluruh kompleksitas "cari baris di tabel `categories` yang `id`-nya sama dengan `category_id` milik buku ini" disembunyikan di balik satu baris kode `$book->category`.

Konsep ini juga berlaku sebaliknya, dan justru di situlah kekuatan sebenarnya terlihat. Foreign key di tabel `books` cuma mendefinisikan relasi dari sisi buku ke kategori (`belongsTo`) — tidak ada kolom apa pun di tabel `categories` yang menyimpan "daftar buku miliknya", karena itu memang bukan cara kerja database relasional. Tabel `categories` tidak perlu tahu apa-apa tentang buku; justru buku yang menyimpan referensi ke kategori. Tapi dari sisi kebutuhan aplikasi, sangat wajar butuh pertanyaan sebaliknya: "buku apa saja yang ada di kategori Fiksi?" Inilah relasi `hasMany()` — Eloquent membalik arah pertanyaan itu jadi query `SELECT * FROM books WHERE category_id = ?` di belakang layar, tapi ditulis di kode sesederhana `$category->books`. Satu foreign key di database bisa menghasilkan dua relasi berbeda di Model tergantung arah mana yang ditanyakan: `belongsTo` dari sisi anak (banyak → satu), `hasMany` dari sisi induk (satu → banyak). Inilah kenapa relationship "hidup" di Model, bukan di database — karena database cuma tahu satu arah hubungan (lewat foreign key), sementara aplikasi butuh menanyakan hubungan itu dari kedua arah.

Prinsip ini sama sekali bukan eksklusif milik Laravel. Setiap ORM modern yang bekerja di atas database relasional menghadapi masalah identik dan menyelesaikannya dengan cara yang secara konseptual mirip. Django (Python) punya `ForeignKey` yang otomatis menghasilkan relasi terbalik lewat `_set` (misalnya `category.book_set.all()`), meski sejak versi modern lebih umum dipakai lewat `related_name` kustom. Sequelize (Node.js) punya `hasMany()`/`belongsTo()` yang penamaannya nyaris identik dengan Eloquent karena memang terinspirasi dari pola yang sama. Hibernate/JPA (Java, dipakai Spring Boot) punya anotasi `@OneToMany` dan `@ManyToOne` yang menjalankan peran serupa lewat cara yang lebih verbose. Yang membedakan tiap implementasi biasanya cuma soal seberapa banyak *boilerplate* yang harus ditulis manual — tapi konsep dasarnya universal: **relasi di level objek/Model selalu lebih kaya daripada yang bisa direpresentasikan murni lewat foreign key di database**, karena foreign key hanya satu arah sementara kebutuhan aplikasi hampir selalu dua arah.

Ada satu jenis relasi lagi yang tidak dipakai langsung di studi kasus perpustakaan ini tapi penting diketahui keberadaannya: **Many-to-Many**. Relasi One-to-Many yang dipakai di seluruh pertemuan ini (`categories`↔`books`, `members`↔`loans`, dst) cukup dengan satu foreign key di tabel "anak". Tapi bayangkan kasus lain — misalnya satu buku bisa ditulis banyak penulis, dan satu penulis bisa menulis banyak buku. Tidak ada cara menaruh satu foreign key di salah satu tabel untuk merepresentasikan ini, karena kedua sisi sama-sama "banyak". Solusinya butuh tabel ketiga khusus (*pivot table*) yang isinya cuma pasangan `book_id` dan `author_id`. Menariknya, tabel `loan_items` di studi kasus ini sebenarnya **berbentuk seperti pivot table** — dia menjembatani `loans` dan `books` — tapi sengaja dibuat sebagai Model penuh (`LoanItem` dengan `belongsTo` ke dua arah) alih-alih pivot table murni Many-to-Many, karena setiap baris `loan_items` punya identitasnya sendiri yang berarti (baris ke berapa dari transaksi peminjaman yang mana, buku yang mana) — bukan sekadar penghubung tanpa makna tambahan. Ini adalah pola umum: kalau tabel penghubung punya data atau makna tersendiri di luar sekadar menghubungkan dua ID, dia lebih baik dimodelkan sebagai Model penuh dengan dua `belongsTo`, bukan relasi `belongsToMany()` bawaan Eloquent.

---

## Materi

### `hasMany()` dan `belongsTo()`

Kedua method ini selalu dipasang berpasangan — satu sisi `belongsTo` selalu punya lawan `hasMany` di Model yang dirujuknya:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum

// Model: Category — "satu kategori punya banyak buku"
public function books(): HasMany
{
    return $this->hasMany(Book::class);
}

// Model: Book — "satu buku dimiliki satu kategori"
public function category(): BelongsTo
{
    return $this->belongsTo(Category::class);
}
```

Nama method (`books`, `category`) bebas ditentukan, tapi konvensinya penting: sisi `hasMany` dinamai jamak (karena hasilnya banyak baris), sisi `belongsTo` dinamai tunggal (karena hasilnya satu baris). Eloquent menebak foreign key dan primary key otomatis dari nama Model — `belongsTo(Category::class)` di dalam `Book` otomatis mencari kolom `category_id` di tabel `books`, mengikuti pola `{nama_model_singular}_id`. Kalau nama kolom tidak mengikuti pola ini, foreign key bisa ditentukan manual lewat argumen kedua: `belongsTo(Category::class, 'nama_kolom_custom')`.

### Mengakses Data Relasi

Method relasi yang sudah didefinisikan dipanggil **tanpa tanda kurung** saat diakses sebagai data (bukan dipanggil sebagai method biasa):

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
$book = Book::find(1);
echo $book->category->nama_kategori;   // "Fiksi" — bukan angka category_id

$category = Category::find(1);
foreach ($category->books as $book) {  // Collection semua buku di kategori ini
    echo $book->judul;
}
```

Ini sebenarnya "keajaiban" PHP magic method (`__get()`) yang dipakai Eloquent: `$book->category` (tanpa kurung) memicu Eloquent memanggil method `category()`, mengeksekusi query-nya, lalu **menyimpan hasilnya** supaya panggilan `$book->category` berikutnya di request yang sama tidak query ulang — perilaku inilah yang jadi akar N+1 Query Problem di bagian selanjutnya. Karena Eloquent Model juga mengimplementasikan `ArrayAccess` (seperti sudah dibahas di Pertemuan 5 untuk mengakses kolom biasa), sintaks `$book['category']['nama_kategori']` juga berfungsi persis sama seperti `$book->category->nama_kategori` — project ini tetap konsisten memakai sintaks array di seluruh view mengikuti konvensi yang sudah berjalan sejak Pertemuan 3.

### N+1 Query Problem

Bayangkan halaman `/books` menampilkan 20 buku, dan setiap baris perlu menampilkan nama kategorinya:

```php
// Contoh ilustrasi konsep — JANGAN ditiru, ini yang bermasalah
$books = Book::all();               // 1 query: ambil 20 buku
foreach ($books as $book) {
    echo $book->category->nama_kategori;  // 1 query BARU untuk SETIAP buku
}
```

Baris `$book->category` di dalam loop terlihat tidak berbahaya, tapi setiap kali dieksekusi dia memicu query baru ke tabel `categories` — karena Eloquent tidak tahu di awal bahwa relasi ini akan diakses berkali-kali dalam sebuah loop, dia hanya tahu mengambil data relasi *saat diminta* (disebut **lazy loading**). Hasilnya: 1 query untuk mengambil 20 buku, ditambah 20 query terpisah untuk 20 kategorinya masing-masing — total **21 query** untuk data yang sebenarnya bisa diambil dengan 2 query saja. Inilah **N+1 Query Problem**: 1 query awal, ditambah N query tambahan (N = jumlah baris hasil query pertama). Untuk 20 buku ini terlihat sepele, tapi bayangkan halaman laporan yang menampilkan 5.000 transaksi peminjaman, masing-masing menampilkan nama anggota, nama petugas, dan judul buku — itu berpotensi menjadi belasan ribu query terpisah untuk satu kali membuka satu halaman. Di production, ini salah satu penyebab paling umum aplikasi jadi lambat tanpa sebab yang terlihat jelas di kode — kodenya "terlihat benar", cuma tidak efisien.

N+1 Query Problem bukan masalah unik Eloquent atau Laravel — dia melekat pada *lazy loading*, pola yang dipakai hampir semua ORM secara default demi kenyamanan menulis kode. Django punya masalah identik dan solusi bernama `select_related()`/`prefetch_related()`. Hibernate (Java) menyebutnya `FetchType.EAGER` vs `FetchType.LAZY`. Sequelize (Node.js) punya opsi `include` di query-nya. Semua menyelesaikan masalah yang sama persis: memberi tahu ORM di awal query bahwa data relasi juga akan dibutuhkan, supaya dia bisa mengambilnya sekaligus lewat query tambahan yang jumlahnya tetap (bukan berkembang seiring jumlah baris).

> 📖 *Ingin penjelasan lebih dalam soal N+1 (analogi, studi kasus terpisah, tabel perbandingan jumlah query, cara mendeteksinya di project nyata)? Baca **[N+1 Query Problem — Pendalaman](./tambahan/n-plus-1-query-problem.md)**.*

### Eager Loading: `with()`

Solusinya di Eloquent adalah method `with()`, dipanggil di query awal untuk memberi tahu Eloquent relasi apa saja yang akan dipakai:

```php
// Contoh ilustrasi konsep — solusi yang benar
$books = Book::with('category')->get();   // tetap cuma 2 query total
foreach ($books as $book) {
    echo $book->category->nama_kategori;  // TIDAK ada query baru di sini
}
```

`with('category')` membuat Eloquent mengeksekusi 2 query saja, berapa pun jumlah baris buku: 1 query `SELECT * FROM books`, lalu 1 query kedua `SELECT * FROM categories WHERE id IN (...)` yang sudah menyertakan **semua** `category_id` yang muncul di hasil query pertama sekaligus. Eloquent kemudian mencocokkan hasil kedua query itu di memori PHP sebelum loop dimulai, sehingga `$book->category` di dalam `foreach` tidak pernah memicu query baru — datanya sudah "dipasang" di masing-masing objek `$book` sejak sebelum loop berjalan. Inilah kenapa disebut **eager loading**: relasi diambil "duluan, tanpa diminta", berlawanan dengan lazy loading yang menunggu sampai benar-benar diakses.

### Multiple & Nested Eager Loading

`with()` menerima array untuk memuat beberapa relasi sekaligus, dan notasi titik (`.`) untuk relasi yang bersarang (relasi dari relasi):

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
$loans = Loan::with(['member', 'user', 'loanItems.book'])->get();
```

Baris ini mengambil satu transaksi peminjaman lengkap dengan anggotanya (`member`), petugas yang mencatatnya (`user`), dan seluruh item buku beserta detail bukunya masing-masing (`loanItems.book` — relasi `book` yang ada *di dalam* setiap `loanItem`) — semua dalam jumlah query yang tetap kecil dan tidak bertambah seiring banyaknya baris `loans`, persis pola yang dipakai `LoanController@index` di praktikum ini.

### Kondisi pada Eager Loading

`with()` juga bisa dibatasi dengan closure kalau hanya baris relasi tertentu yang dibutuhkan, tanpa mengambil semuanya:

```php
// Contoh ilustrasi konsep — bukan langkah praktikum
$categories = Category::with(['books' => function ($query) {
    $query->where('stok', '>', 0);
}])->get();
```

Ini berguna kalau relasi yang dimuat berpotensi sangat banyak dan hanya sebagian yang relevan ditampilkan — tidak dipakai langsung di praktikum pertemuan ini, tapi baik diketahui batasannya sejak awal.

---

## Praktikum

> **Konteks:** Melanjutkan project `app-perpustakaan` dari Pertemuan 6 (UTS).
> Pastikan branch aktif adalah `dev`.
> Di akhir praktikum ini kamu akan memiliki: relasi lengkap di semua Model, halaman buku yang menampilkan nama kategori (bukan ID), N+1 Query Problem yang sudah didemonstrasikan dan diselesaikan, serta CRUD `loans` yang sepenuhnya berfungsi dengan data anggota, petugas, dan buku yang saling terhubung.
> Kalau butuh lihat lagi definisi lengkap seluruh relasi antar tabel, buka **[Studi Kasus & Desain Database](./studi-kasus-database.md)**.

### Langkah 0 — Menyelesaikan Tugas Mandiri Pertemuan 5 yang Tertunda

CRUD `members` (Tugas mandiri Pertemuan 5) ternyata jadi prasyarat wajib pertemuan ini — form tambah peminjaman butuh dropdown anggota nyata, dan salah satu output praktikum ini adalah halaman detail anggota yang menampilkan riwayat peminjamannya. Kalau kamu belum sempat menyelesaikan tugas itu, kerjakan dulu bagian ini sebelum lanjut ke relasi.

`MemberController` diubah dari array dummy ke Eloquent penuh, mengikuti pola persis `CategoryController` di Pertemuan 5:

```php
// File: app/Http/Controllers/MemberController.php
use App\Http\Requests\StoreMemberRequest;
use App\Models\Member;

public function index()
{
    $members = Member::paginate(10);

    return view('members.index', compact('members'));
}

public function store(StoreMemberRequest $request)
{
    $validated = $request->validated();

    Member::create($validated);

    return redirect()->route('members.index')
        ->with('success', "Anggota \"{$validated['nama']}\" berhasil ditambahkan.");
}
```

`StoreMemberRequest` dibuat lewat `php artisan make:request StoreMemberRequest` dengan validasi `nama` wajib, `nim` wajib dan unik, `email` wajib format email dan unik, `nomor_telepon` wajib, `alamat` wajib, `status` wajib salah satu dari `aktif`/`nonaktif`. Method `edit`, `update`, dan `destroy` mengikuti pola `findOrFail()` yang sama seperti `CategoryController`, dan tiga view baru (`create.blade.php`, `edit.blade.php`, `show.blade.php`) dibuat mengikuti struktur form yang sama seperti `books/create.blade.php`/`edit.blade.php`.

> Fitur **search nama anggota** yang juga disebut di Tugas Pertemuan 5 tetap belum dikerjakan di langkah ini — itu tetap jadi tugas mandiri terpisah, tidak menghalangi praktikum relasi di pertemuan ini.

### Langkah 1 — Menambahkan Relationship di Semua Model

Enam pasang relasi ditambahkan sesuai desain di bagian F `master-outline.md`:

```php
// File: app/Models/Category.php
use Illuminate\Database\Eloquent\Relations\HasMany;

public function books(): HasMany
{
    return $this->hasMany(Book::class);
}
```

```php
// File: app/Models/Book.php
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

public function category(): BelongsTo
{
    return $this->belongsTo(Category::class);
}

public function loanItems(): HasMany
{
    return $this->hasMany(LoanItem::class);
}
```

```php
// File: app/Models/Member.php
use Illuminate\Database\Eloquent\Relations\HasMany;

public function loans(): HasMany
{
    return $this->hasMany(Loan::class);
}
```

```php
// File: app/Models/User.php
use Illuminate\Database\Eloquent\Relations\HasMany;

public function loans(): HasMany
{
    return $this->hasMany(Loan::class);
}
```

```php
// File: app/Models/Loan.php
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

public function member(): BelongsTo
{
    return $this->belongsTo(Member::class);
}

public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

public function loanItems(): HasMany
{
    return $this->hasMany(LoanItem::class);
}
```

```php
// File: app/Models/LoanItem.php
use Illuminate\Database\Eloquent\Relations\BelongsTo;

public function loan(): BelongsTo
{
    return $this->belongsTo(Loan::class);
}

public function book(): BelongsTo
{
    return $this->belongsTo(Book::class);
}
```

Tipe return `HasMany`/`BelongsTo` di setiap method sifatnya opsional secara fungsional (Eloquent tetap bekerja tanpanya), tapi memberi *autocomplete* yang jauh lebih baik di editor dan mempertegas jenis relasi yang dideklarasikan setiap method.

### Langkah 2 — Mendemokan N+1 Query Problem

Sebelum memperbaiki `BookController`, jalankan dulu perbandingan berikut lewat `php artisan tinker` untuk melihat sendiri selisih jumlah query-nya:

```bash
php artisan tinker
```

```php
use Illuminate\Support\Facades\DB;

DB::enableQueryLog();
$books = App\Models\Book::all();
foreach ($books as $book) { $nama = $book->category->nama_kategori; }
echo count(DB::getQueryLog());   // tanpa eager loading

DB::flushQueryLog();
$books2 = App\Models\Book::with('category')->get();
foreach ($books2 as $book) { $nama = $book->category->nama_kategori; }
echo count(DB::getQueryLog());   // dengan eager loading
```

Dengan 2 baris data buku, hasil sebenarnya dari percobaan ini adalah **3 query** tanpa eager loading (1 untuk buku + 2 untuk kategori masing-masing baris) berbanding **2 query** dengan eager loading (1 untuk buku + 1 untuk seluruh kategori sekaligus) — selisihnya makin lebar seiring makin banyak baris buku yang ada, karena angka "tanpa eager loading" bertambah linear mengikuti jumlah baris, sedangkan angka "dengan eager loading" tetap konstan.

> 📸 *Screenshot: output `tinker` menampilkan dua angka jumlah query yang berbeda, membuktikan N+1 Query Problem terjadi nyata, bukan cuma teori.*

### Langkah 3 — Memperbaiki `BookController` dengan Eager Loading

`index()` dan `show()` di `BookController` diubah untuk memuat relasi `category` sekaligus di query awal:

```php
// File: app/Http/Controllers/BookController.php
public function index()
{
    $books = Book::with('category')->paginate(10);

    return view('books.index', compact('books'));
}

public function show(string $id)
{
    $book = Book::with('category')->findOrFail($id);

    return view('books.show', compact('book'));
}
```

Kolom "ID Kategori" di `books/index.blade.php` dan "ID Kategori" di `books/show.blade.php` diganti jadi "Kategori", mengakses nama aslinya lewat relasi:

```blade
{{-- File: resources/views/books/index.blade.php --}}
<td>{{ $book['category']['nama_kategori'] }}</td>
```

> 📸 *Screenshot: halaman `/books` menampilkan nama kategori ("Novel", "Komik", dst) di kolom kategori, bukan lagi angka ID.*

### Langkah 4 — Menambahkan Relasi ke Halaman Detail Anggota

`MemberController@show` memuat riwayat peminjaman anggota lewat relasi bersarang, supaya buku dan petugas di setiap transaksi ikut termuat sekaligus tanpa N+1:

```php
// File: app/Http/Controllers/MemberController.php
public function show(string $id)
{
    $member = Member::with(['loans.loanItems.book', 'loans.user'])->findOrFail($id);

    return view('members.show', compact('member'));
}
```

View `members/show.blade.php` menampilkan data anggota seperti biasa, ditambah tabel riwayat peminjaman yang di-loop dari `$member['loans']`:

```blade
{{-- File: resources/views/members/show.blade.php --}}
@forelse ($member['loans'] as $loan)
    <tr>
        <td>{{ $loan['tanggal_pinjam'] }}</td>
        <td>{{ $loan['tanggal_kembali'] }}</td>
        <td>{{ $loan['user']['name'] }}</td>
        <td>
            @foreach ($loan['loanItems'] as $item)
                {{ $item['book']['judul'] }}@if (!$loop->last), @endif
            @endforeach
        </td>
        <td>{{ ucfirst($loan['status']) }}</td>
    </tr>
@empty
    <tr><td colspan="5">Anggota ini belum pernah meminjam buku.</td></tr>
@endforelse
```

> 📸 *Screenshot: halaman `/members/{id}` menampilkan data anggota lengkap dengan tabel riwayat peminjaman di bawahnya.*

### Langkah 5 — Membangun CRUD `loans`

Berbeda dari `books`/`categories`/`members` yang masing-masing cuma menyentuh satu tabel, `LoanController@store` perlu menulis ke **dua tabel sekaligus** dalam satu transaksi pengguna: satu baris `loans` (data transaksinya), dan satu baris `loan_items` untuk **setiap** buku yang dipilih (karena satu peminjaman bisa mencakup lebih dari satu buku).

```php
// File: app/Http/Controllers/LoanController.php
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

    return redirect()->route('loans.index')
        ->with('success', 'Transaksi peminjaman berhasil dibuat.');
}
```

`$loan->loanItems()->create([...])` adalah pola penting: karena dipanggil **dari instance `$loan` yang baru saja dibuat**, Eloquent otomatis mengisi `loan_id` di setiap baris `loan_items` yang dibuat — tidak perlu ditulis manual `'loan_id' => $loan->id` di array-nya. Ini berlaku untuk relasi `hasMany` apa pun: memanggil `create()` lewat instance relasi (bukan lewat Model secara langsung) otomatis menghubungkan baris baru itu ke induknya.

`user_id` (petugas yang mencatat transaksi) untuk sementara dipilih manual lewat dropdown di form, karena Autentikasi baru masuk Pertemuan 8 — belum ada `auth()->id()` yang bisa dipakai otomatis. Ini akan disederhanakan begitu login tersedia.

`index()` dan `show()` memuat ketiga relasi sekaligus supaya tidak terjadi N+1 saat menampilkan daftar maupun detail peminjaman:

```php
// File: app/Http/Controllers/LoanController.php
public function index()
{
    $loans = Loan::with(['member', 'user', 'loanItems.book'])->paginate(10);

    return view('loans.index', compact('loans'));
}
```

Form create (`loans/create.blade.php`) menyediakan dropdown anggota dan petugas, input tanggal, serta daftar checkbox buku (`name="book_ids[]"`) supaya satu peminjaman bisa mencakup lebih dari satu buku sekaligus:

```blade
{{-- File: resources/views/loans/create.blade.php --}}
<div class="checkbox-list">
    @foreach ($books as $book)
        <label>
            <input type="checkbox" name="book_ids[]" value="{{ $book['id'] }}">
            {{ $book['judul'] }} (stok: {{ $book['stok'] }})
        </label>
    @endforeach
</div>
```

Method `edit`/`update`/`destroy` melengkapi CRUD standar (mengubah tanggal kembali & status, dan menghapus transaksi). Karena `loan_items` terhubung ke `loans` tanpa `cascadeOnDelete()` di migration-nya, item-nya harus dihapus lebih dulu secara eksplisit sebelum baris `loans` induknya dihapus, kalau tidak MySQL akan menolak lewat error foreign key constraint:

```php
// File: app/Http/Controllers/LoanController.php
public function destroy(string $id)
{
    $loan = Loan::findOrFail($id);
    $loan->loanItems()->delete();
    $loan->delete();

    return redirect()->route('loans.index')
        ->with('success', 'Transaksi peminjaman berhasil dihapus.');
}
```

> 📸 *Screenshot: form `/loans/create` terisi lengkap, lalu halaman `/loans` menampilkan nama anggota, nama petugas, dan judul buku (bukan ID) di baris transaksi yang baru dibuat.*

### Langkah 6 — Ujicoba

1. Buka `/books` — pastikan kolom kategori menampilkan nama, bukan angka.
2. Buka `/members/{id}` milik anggota yang belum pernah meminjam — pastikan tampil pesan "belum pernah meminjam buku".
3. Buka `/loans/create`, pilih anggota, petugas, tanggal pinjam & kembali, centang minimal satu buku, submit — pastikan redirect ke `/loans` dengan flash message sukses dan baris baru muncul dengan nama (bukan ID) di setiap kolom relasinya.
4. Buka kembali `/members/{id}` anggota yang baru saja meminjam di langkah 3 — pastikan transaksi itu muncul di riwayat peminjamannya.
5. Edit transaksi peminjaman itu lewat `/loans/{id}/edit`, ubah status jadi "Dikembalikan", submit, pastikan tersimpan.
6. Hapus transaksi itu lewat tombol "Hapus" di `/loans` — pastikan tidak muncul error foreign key, dan baris beserta item bukunya benar-benar hilang.

---

## Checkpoint GitHub

```bash
git add .
git commit -m "[P-7] implementasi eloquent relationships dan crud loans"
git push origin dev
```

---

## Tugas

Lengkapi dua fitur berikut di atas CRUD `loans` yang sudah berfungsi:

1. **Fitur "kembalikan buku"** — isi method `LoanController@kembalikan` (saat ini masih stub) supaya mengubah `status` transaksi jadi `dikembalikan` dan mengisi `tanggal_dikembalikan` dengan tanggal hari ini (`now()->toDateString()`), lalu redirect kembali ke `/loans` dengan flash message sukses. Tambahkan tombol "Kembalikan" di `loans/index.blade.php` yang hanya muncul kalau `status` transaksi masih `dipinjam` (pakai `@if ($loan['status'] === 'dipinjam')`).
2. **Badge status berwarna** — ganti teks status polos (`{{ ucfirst($loan['status']) }}`) di `loans/index.blade.php` dan `loans/show.blade.php` dengan `<span>` berwarna berbeda per status: hijau untuk `dikembalikan`, kuning/oranye untuk `dipinjam`, merah untuk `terlambat`. Buat class CSS baru di `layouts/app.blade.php` (mengikuti pola `.alert-success` yang sudah ada), jangan inline style.

**Yang dikumpulkan:**
- Link commit GitHub (branch `dev`) yang berisi hasil tugas
- Screenshot `/loans` menampilkan badge status berwarna dan tombol "Kembalikan" yang berfungsi

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-06.md) | [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-08.md)*
