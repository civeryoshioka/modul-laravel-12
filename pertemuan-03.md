# Pertemuan 3 — Controller, Request & Validation

> **Sebelumnya:** Semua route resource (`books`, `categories`, `members`, `loans`) sudah terdaftar dan mengarah ke Controller yang masih mengembalikan string dummy.
> **Pertemuan ini:** Controller diisi logika sungguhan — mengambil data dari `Request`, memvalidasi input dengan Form Request, dan mengembalikan `view()` atau `redirect()` sesuai hasilnya.

---

## Capaian Pembelajaran

Setelah menyelesaikan pertemuan ini, mahasiswa mampu:
1. Menjelaskan peran Controller dalam MVC sebagai penghubung Model dan View
2. Membuat Resource Controller dan memahami tujuan setiap method-nya
3. Menggunakan class `Request` untuk mengambil berbagai jenis data HTTP
4. Mengimplementasikan validasi input beserta penanganan errornya

---

## Konsep: Mengapa Controller, Request & Validation Ada?

Di Pertemuan 2, kita sudah melihat bahwa route hanya bertugas memetakan URL ke sebuah tujuan — dia tahu *ke mana* request harus pergi, tapi tidak tahu *apa yang harus dilakukan* di sana. Kalau logika bisnis (mengambil input, mengecek apakah valid, menyimpan data, menyiapkan response) ditulis langsung di closure route, file `web.php` akan cepat membengkak menjadi ratusan baris kode yang bercampur aduk dengan definisi URL. Controller lahir untuk memisahkan dua tanggung jawab ini: route berurusan dengan "jalur masuk", Controller berurusan dengan "apa yang terjadi setelah masuk". Pemisahan ini bukan sekadar soal kerapian — ini soal *testability* dan *reusability*. Sebuah method Controller bisa diuji secara terpisah dari sistem routing, dan bisa dipanggil dari command line (misalnya lewat `artisan tinker`) tanpa harus melalui HTTP sama sekali.

Konsep ini universal di hampir semua framework backend. Di Express.js, pemisahan yang sama dilakukan lewat "route handler" yang biasanya dipindahkan ke file controller terpisah begitu aplikasi membesar. Di Django, komponennya disebut "View" (sedikit membingungkan karena istilahnya berbeda dari Blade view di Laravel) tapi perannya identik: menerima request, memproses, mengembalikan response. Di Spring Boot, ada anotasi `@Controller` dan `@RestController` yang menandai class mana yang berperan sebagai penghubung antara HTTP layer dan business logic. Semua framework ini sampai pada solusi arsitektur yang sama karena masalah yang mereka selesaikan juga sama: request masuk lewat satu pintu, tapi logic-nya berbeda-beda tergantung endpoint.

Setelah data sampai di Controller, masalah berikutnya muncul: bagaimana kita tahu data yang dikirim user itu benar? User bisa mengirim form kosong, mengetik huruf di kolom yang seharusnya angka, atau bahkan mengirim request palsu lewat Postman tanpa lewat form sama sekali. Di sinilah *validation* menjadi lapisan pertahanan wajib. Prinsip keamanan yang berlaku di semua sistem — bukan cuma Laravel — adalah **jangan pernah mempercayai input dari luar**. Validasi bukan formalitas, dia mencegah data rusak masuk ke database, mencegah error tak terduga di tengah proses, dan memberi feedback yang jelas ke user tentang apa yang salah. Django punya `Form` dan `Serializer` untuk tujuan yang sama, Express.js biasanya memakai library seperti `express-validator` atau `Joi`, dan Spring Boot punya anotasi `@Valid` dengan Bean Validation. Laravel menonjol karena menyediakan dua gaya sekaligus: validasi inline langsung di Controller (`$request->validate()`), dan Form Request — class terpisah yang membungkus aturan validasi supaya Controller tetap ringkas.

Form Request layak dibahas lebih dalam karena dia mendemonstrasikan prinsip *single responsibility* dengan jelas. Alih-alih Controller memikirkan "apa saja aturan validasi untuk form ini", tanggung jawab itu dipindahkan ke class `StoreBookRequest` misalnya, yang isinya hanya dua hal: aturan (`rules()`) dan pesan error kustom (`messages()`). Laravel secara otomatis menjalankan validasi ini *sebelum* method Controller sempat dieksekusi — kalau gagal, request langsung dilempar balik ke halaman sebelumnya lengkap dengan pesan error dan input lama, tanpa satu baris kode pun ditulis di Controller untuk menangani kegagalan itu. Ini adalah contoh konkret bagaimana framework mengambil alih "boilerplate" yang berulang di hampir semua aplikasi web, supaya developer fokus ke logika yang benar-benar unik untuk aplikasinya.

Ini membawa kita ke satu prinsip penting yang sering disebut *"skinny controller, fat model"* — Controller idealnya **kurus**: dia hanya mengorkestrasi, bukan mengerjakan semuanya sendiri. Tugasnya cuma tiga: terima input (lewat `Request`), delegasikan pekerjaan berat ke tempat yang tepat (validasi ke Form Request, logika data ke Model), lalu siapkan response (`view()` atau `redirect()`). Kalau sebuah method Controller mulai berisi query database yang rumit, perhitungan bisnis yang panjang, atau logika bercabang yang berbelit, itu tanda tanggung jawab itu seharusnya dipindahkan ke lapisan lain (Model, atau nanti class Service terpisah). Prinsip ini bukan aturan kaku Laravel — ini prinsip desain software yang berlaku di Django, Express.js, Spring Boot, atau framework apa pun yang menganut MVC: semakin sedikit yang dikerjakan Controller sendirian, semakin mudah kode itu diuji, dibaca ulang, dan diubah tanpa merusak bagian lain.

---

## Materi

### Mengapa Logika Tidak Boleh di Route Closure

Route closure cocok untuk percobaan cepat atau endpoint yang sangat sederhana, tapi begitu ada logika seperti validasi, pengambilan data, atau lebih dari beberapa baris kode, dia harus dipindah ke Controller. Alasannya: closure tidak bisa di-cache oleh Laravel (route caching), sulit di-reuse, dan membuat file route sulit dibaca sebagai daftar endpoint.

### Resource Controller dan 7 Method-nya

```bash
php artisan make:controller BookController --resource
```

Perintah ini sudah dijalankan di Pertemuan 2. Tujuh method yang dihasilkan otomatis terpetakan ke kombinasi HTTP method dan URI berikut:

| Method Controller | HTTP Method | URI               | Fungsi                          |
|---|---|---|---|
| `index()`   | GET    | `/books`          | Tampilkan semua data            |
| `create()`  | GET    | `/books/create`   | Tampilkan form tambah data      |
| `store()`   | POST   | `/books`          | Simpan data baru                |
| `show()`    | GET    | `/books/{id}`     | Tampilkan detail satu data      |
| `edit()`    | GET    | `/books/{id}/edit`| Tampilkan form edit data        |
| `update()`  | PUT/PATCH | `/books/{id}`  | Perbarui data                   |
| `destroy()` | DELETE | `/books/{id}`     | Hapus data                      |

### Kenapa 7 Method, Bukan 4?

Operasi dasar terhadap data cuma ada empat — biasa disingkat **CRUD**: Create, Read, Update, Delete. Tapi tabel di atas punya 7 method, bukan 4. Selisihnya (`create`, `edit`, dan `index` sebagai bagian dari Read) bukan operasi data — mereka adalah method yang tugasnya **menampilkan form atau tampilan**, bukan mengubah data itu sendiri.

Coba petakan begini:
- **Create** (operasi data) → dikerjakan oleh `store()`. Tapi sebelum user bisa mengirim data untuk disimpan, dia perlu melihat formnya dulu — itu tugas `create()`.
- **Read** (operasi data) → dikerjakan oleh `index()` (banyak data) dan `show()` (satu data).
- **Update** (operasi data) → dikerjakan oleh `update()`. Sama seperti Create, user butuh melihat form yang sudah terisi data lama dulu — itu tugas `edit()`.
- **Delete** (operasi data) → dikerjakan oleh `destroy()`. Tidak butuh form, karena hapus biasanya cuma butuh konfirmasi, bukan input data baru.

Jadi pola pikirnya: `create` adalah "halaman menuju `store`", dan `edit` adalah "halaman menuju `update`". Ini kenapa route resource selalu menghasilkan 7, bukan sekadar 4 — karena aplikasi web butuh *tampilan* di antara *aksi*, sesuatu yang tidak dibutuhkan kalau kamu langsung memanggil API tanpa antarmuka form (nanti terlihat lagi di Pertemuan 9, REST API murni cuma butuh 5 endpoint karena tidak ada `create()`/`edit()` — API tidak mengembalikan halaman HTML berisi form).

### Lifecycle Controller: Instance Baru Setiap Request

Satu hal yang gampang disalahpahami: `$books` yang dideklarasikan sebagai property di `BookController` (lihat Langkah 2 di bagian Praktikum) itu **bukan penyimpanan permanen**. Setiap kali ada request masuk ke `/books`, Laravel membuat instance `BookController` yang benar-benar baru dari nol, menjalankan satu method yang relevan, lalu instance itu dibuang setelah response dikirim. Tidak ada "memori" yang bertahan antar-request.

Ini menjelaskan sesuatu yang mungkin sudah kamu perhatikan sendiri saat praktikum: klik "Hapus" pada salah satu buku memang memunculkan flash message "berhasil dihapus", tapi begitu halaman di-refresh, buku itu muncul lagi di daftar. Itu bukan bug — itu konsekuensi logis dari sifat *stateless* ini. `destroy()` memang dieksekusi, tapi dia tidak benar-benar mengubah apa pun karena `$books` yang dihapus cuma salinan sementara milik instance Controller yang sudah dibuang begitu response dikirim; request berikutnya mendapat instance baru dengan `$books` yang dideklarasikan ulang persis seperti semula.

Sifat stateless ini bukan kelemahan Laravel — ini fondasi bagaimana hampir semua web framework bekerja (PHP native, Express.js, Django, Spring Boot — semuanya menciptakan ulang konteks request dari nol setiap kali). Justru karena Controller tidak menyimpan apa pun secara permanen, satu server bisa melayani ribuan request dari user berbeda secara bersamaan tanpa data satu user "bocor" ke user lain. Penyimpanan yang benar-benar permanen — yang bertahan antar-request — adalah tanggung jawab lapisan lain: database (lewat Model, mulai Pertemuan 5) atau session (untuk data sementara per-user, seperti flash message `session('success')` yang sudah kamu pakai sejak awal pertemuan ini).

### Mengambil Data dari Request

Class `Illuminate\Http\Request` di-inject otomatis oleh Laravel ke method Controller (disebut *dependency injection*). Ini bagian dari prinsip "Controller cuma menerima, tidak mencari sendiri": kamu tidak perlu menulis kode untuk membaca `$_POST` atau `$_GET` seperti di PHP native — cukup tulis tipe `Request $request` sebagai parameter method, dan Laravel otomatis menyiapkan objek itu, sudah berisi seluruh data request saat ini, sebelum method-mu sempat dijalankan. Mekanisme yang sama persis dipakai untuk meng-inject `StoreBookRequest` yang akan kamu lihat di bagian Validasi — Laravel cukup melihat tipe parameter untuk tahu instance apa yang harus disiapkan dan divalidasi terlebih dahulu.

Beberapa method yang paling sering dipakai dari objek `Request`:

```php
$request->input('judul');     // ambil satu field
$request->all();              // ambil semua field sebagai array
$request->only(['judul', 'penulis']); // ambil field tertentu saja
$request->except(['_token']); // ambil semua kecuali field tertentu
$request->file('sampul');     // ambil file upload
$request->method();           // GET, POST, dst
$request->url();              // URL saat ini
```

### Validasi Inline vs Form Request

Validasi inline ditulis langsung di dalam method Controller memakai `$request->validate([...])`. Cocok untuk validasi sederhana atau ketika aturan validasinya sangat spesifik untuk satu method saja.

```php
// Contoh ilustrasi konsep — validasi inline di Controller
$validated = $request->validate([
    'judul' => 'required|string|max:200',
    'stok' => 'required|integer|min:0',
]);
```

Form Request memindahkan aturan validasi ke class terpisah. Laravel otomatis menjalankan validasi ini sebelum method Controller dieksekusi:

```bash
php artisan make:request StoreBookRequest
```

```php
// Contoh ilustrasi konsep — struktur Form Request
class StoreBookRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // belum ada autentikasi, jadi selalu diizinkan untuk saat ini
    }

    public function rules(): array
    {
        return [
            'judul' => 'required|string|max:200',
        ];
    }
}
```

Aturan validasi yang paling sering dipakai: `required`, `string`, `integer`, `max`, `min`, `email`, `unique`, `date`, `in`, `nullable`. Aturan bisa digabung dengan tanda `|` atau ditulis sebagai array.

### Menampilkan Error Validasi di Blade

Ketika validasi gagal, Laravel otomatis redirect kembali ke halaman asal dengan session berisi `$errors` (instance `MessageBag`) dan input lama (`old()`). Blade menyediakan directive `@error` untuk menampilkan pesan error per field:

```blade
<input type="text" name="judul" value="{{ old('judul') }}">
@error('judul')
    <div class="error">{{ $message }}</div>
@enderror
```

### Response Types

Tiga jenis response yang paling umum dipakai dari Controller:

```php
return view('books.index', compact('books'));        // render Blade view
return redirect()->route('books.index')->with('success', '...'); // redirect + flash message
return back()->withInput();                            // kembali ke halaman sebelumnya (otomatis dipicu saat validasi gagal)
```

---

## Praktikum

> **Konteks:** Melanjutkan project `app-perpustakaan` dari Pertemuan 2.
> Pastikan branch aktif adalah `dev`.
> Di akhir praktikum ini kamu akan memiliki: `BookController` dengan semua method berfungsi (data dummy array), `CategoryController` dengan `index` dan `store` tervalidasi, dua Form Request class, dan tampilan Blade sederhana (belum pakai master layout — itu baru dibuat di Pertemuan 4) yang menampilkan data dan error validasi.

### Langkah 1 — Membuat Form Request

Form Request dibuat lewat artisan, sama seperti membuat Controller.

```bash
php artisan make:request StoreBookRequest
php artisan make:request StoreCategoryRequest
```

Isi `rules()` dan `messages()` pada `StoreBookRequest` sesuai kolom-kolom tabel `books` yang sudah dirancang di awal modul (lihat desain database di Pertemuan 5 nanti). `authorize()` diubah jadi `true` karena autentikasi baru diimplementasikan di Pertemuan 8 — untuk saat ini semua orang "diizinkan" mengisi form.

```php
// File: app/Http/Requests/StoreBookRequest.php
public function authorize(): bool
{
    return true;
}

public function rules(): array
{
    return [
        'judul' => 'required|string|max:200',
        'penulis' => 'required|string|max:100',
        'penerbit' => 'required|string|max:100',
        'tahun_terbit' => 'required|integer|min:1900|max:'.date('Y'),
        'isbn' => 'nullable|string|max:20',
        'stok' => 'required|integer|min:0',
        'category_id' => 'required|integer',
    ];
}

public function messages(): array
{
    return [
        'judul.required' => 'Judul buku wajib diisi.',
        'category_id.required' => 'Kategori wajib dipilih.',
        // pesan lain menyusul pola yang sama: field.rule => pesan
    ];
}
```

Lakukan hal yang sama untuk `StoreCategoryRequest` sesuai kolom tabel `categories` (`nama_kategori`, `deskripsi`).

```php
// File: app/Http/Requests/StoreCategoryRequest.php
public function rules(): array
{
    return [
        'nama_kategori' => 'required|string|max:100',
        'deskripsi' => 'nullable|string',
    ];
}
```

### Langkah 2 — Mengisi BookController Secara Penuh

Karena Migration & Model Eloquent baru dibuat di Pertemuan 5, data buku untuk sementara disimpan sebagai array dummy di dalam Controller. Semua method diisi logika nyata — tidak ada lagi yang mengembalikan string dummy.

```php
// File: app/Http/Controllers/BookController.php
private array $categories = [
    ['id' => 1, 'nama_kategori' => 'Fiksi'],
    ['id' => 2, 'nama_kategori' => 'Teknologi'],
    ['id' => 3, 'nama_kategori' => 'Sejarah'],
];

private array $books = [
    ['id' => 1, 'judul' => 'Laskar Pelangi', 'penulis' => 'Andrea Hirata', 'penerbit' => 'Bentang Pustaka', 'tahun_terbit' => 2005, 'isbn' => '9789793062792', 'stok' => 5, 'category_id' => 1, 'kategori' => 'Fiksi'],
    ['id' => 2, 'judul' => 'Bumi Manusia', 'penulis' => 'Pramoedya Ananta Toer', 'penerbit' => 'Hasta Mitra', 'tahun_terbit' => 1980, 'isbn' => '9789794330746', 'stok' => 3, 'category_id' => 1, 'kategori' => 'Fiksi'],
    ['id' => 3, 'judul' => 'Clean Code', 'penulis' => 'Robert C. Martin', 'penerbit' => 'Prentice Hall', 'tahun_terbit' => 2008, 'isbn' => '9780132350884', 'stok' => 7, 'category_id' => 2, 'kategori' => 'Teknologi'],
];

public function index()
{
    $books = $this->books;

    return view('books.index', compact('books'));
}

public function store(StoreBookRequest $request)
{
    $validated = $request->validated();

    return redirect()->route('books.index')
        ->with('success', "Buku \"{$validated['judul']}\" berhasil ditambahkan (data dummy, belum tersimpan ke database).");
}
```

`update()` sengaja ditulis memakai validasi **inline** (`$request->validate()`), bukan Form Request, supaya kamu bisa membandingkan langsung kedua gaya di project yang sama:

```php
// File: app/Http/Controllers/BookController.php
public function update(Request $request, string $id)
{
    $validated = $request->validate([
        'judul' => 'required|string|max:200',
        'penulis' => 'required|string|max:100',
        'penerbit' => 'required|string|max:100',
        'tahun_terbit' => 'required|integer|min:1900|max:'.date('Y'),
        'isbn' => 'nullable|string|max:20',
        'stok' => 'required|integer|min:0',
        'category_id' => 'required|integer',
    ]);

    return redirect()->route('books.index')
        ->with('success', "Buku \"{$validated['judul']}\" berhasil diperbarui (data dummy, belum tersimpan ke database).");
}
```

`show()` dan `edit()` mencari buku dari array dummy berdasarkan id, lalu melempar 404 kalau tidak ditemukan — perilaku ini nanti otomatis digantikan `findOrFail()` saat Eloquent masuk di Pertemuan 5:

```php
// File: app/Http/Controllers/BookController.php
public function show(string $id)
{
    $book = collect($this->books)->firstWhere('id', (int) $id);

    abort_if(! $book, 404);

    return view('books.show', compact('book'));
}
```

`destroy()` untuk saat ini hanya mengembalikan flash message sukses tanpa benar-benar menghapus apa pun, karena belum ada penyimpanan data sungguhan.

### Langkah 3 — Membuat View Buku (Tanpa Master Layout Dulu)

Master layout (`layouts/app.blade.php`) baru dibangun di Pertemuan 4. Untuk saat ini, view ditulis sebagai HTML biasa lengkap dengan `<!DOCTYPE html>` sendiri-sendiri, cukup untuk mendemonstrasikan data dummy dan validasi.

```blade
{{-- File: resources/views/books/index.blade.php --}}
@if (session('success'))
    <div class="success">{{ session('success') }}</div>
@endif

<table>
    @foreach ($books as $book)
        <tr>
            <td>{{ $book['judul'] }}</td>
            <td>{{ $book['kategori'] }}</td>
        </tr>
    @endforeach
</table>
```

```blade
{{-- File: resources/views/books/create.blade.php --}}
<form action="{{ route('books.store') }}" method="POST">
    @csrf

    <label for="judul">Judul</label>
    <input type="text" name="judul" id="judul" value="{{ old('judul') }}">
    @error('judul')
        <div class="error">{{ $message }}</div>
    @enderror

    <label for="category_id">Kategori</label>
    <select name="category_id" id="category_id">
        <option value="">-- Pilih Kategori --</option>
        @foreach ($categories as $category)
            <option value="{{ $category['id'] }}" @selected(old('category_id') == $category['id'])>
                {{ $category['nama_kategori'] }}
            </option>
        @endforeach
    </select>

    <button type="submit">Simpan</button>
</form>
```

> 📸 *Screenshot: form "Tambah Buku" setelah submit kosong — setiap field menampilkan pesan error berwarna merah di bawahnya.*

`books/edit.blade.php` mirip dengan `create.blade.php`, hanya saja `<form>`-nya mengarah ke `route('books.update', $book['id'])`, menambahkan `@method('PUT')`, dan setiap input memakai `old('field', $book['field'])` supaya nilai lama muncul saat pertama kali form dibuka. `books/show.blade.php` cukup menampilkan detail satu buku dalam bentuk tabel key-value.

### Langkah 4 — Mengisi CategoryController (index & store)

Sesuai target pertemuan ini, hanya `index()`, `create()`, dan `store()` yang diisi logika nyata. `edit()`, `update()`, `destroy()` tetap seperti kerangka Pertemuan 2 — CRUD kategori yang lengkap baru selesai di Pertemuan 5.

```php
// File: app/Http/Controllers/CategoryController.php
private array $categories = [
    ['id' => 1, 'nama_kategori' => 'Fiksi', 'deskripsi' => 'Buku cerita rekaan seperti novel dan kumpulan cerpen.'],
    ['id' => 2, 'nama_kategori' => 'Teknologi', 'deskripsi' => 'Buku seputar teknologi, pemrograman, dan ilmu komputer.'],
    ['id' => 3, 'nama_kategori' => 'Sejarah', 'deskripsi' => 'Buku bertema sejarah dan biografi tokoh.'],
];

public function index()
{
    $categories = $this->categories;

    return view('categories.index', compact('categories'));
}

public function create()
{
    return view('categories.create');
}

public function store(StoreCategoryRequest $request)
{
    $validated = $request->validated();

    return redirect()->route('categories.index')
        ->with('success', "Kategori \"{$validated['nama_kategori']}\" berhasil ditambahkan (data dummy, belum tersimpan ke database).");
}
```

### Langkah 5 — Membuat View Kategori

```blade
{{-- File: resources/views/categories/create.blade.php --}}
<form action="{{ route('categories.store') }}" method="POST">
    @csrf

    <label for="nama_kategori">Nama Kategori</label>
    <input type="text" name="nama_kategori" id="nama_kategori" value="{{ old('nama_kategori') }}">
    @error('nama_kategori')
        <div class="error">{{ $message }}</div>
    @enderror

    <label for="deskripsi">Deskripsi (opsional)</label>
    <textarea name="deskripsi" id="deskripsi">{{ old('deskripsi') }}</textarea>

    <button type="submit">Simpan</button>
</form>
```

`categories/index.blade.php` mengikuti pola yang sama seperti `books/index.blade.php`: menampilkan flash message sukses, lalu me-loop `$categories`.

### Langkah 6 — Ujicoba

Jalankan server dan uji langsung di browser:

```bash
php artisan serve
```

1. Buka `http://127.0.0.1:8000/books` — pastikan tabel data dummy tampil.
2. Buka `http://127.0.0.1:8000/books/create`, klik **Simpan** tanpa mengisi apa pun — pastikan setiap field menampilkan pesan error dalam Bahasa Indonesia.
3. Isi form dengan data valid, submit — pastikan ter-redirect ke `/books` dengan flash message sukses berwarna hijau.
4. Ulangi langkah yang sama untuk `http://127.0.0.1:8000/categories/create`.
5. Buka `http://127.0.0.1:8000/books/2/edit` — pastikan form terisi otomatis dengan data buku id 2, termasuk dropdown kategori yang ter-select dengan benar.

> 📸 *Screenshot: halaman `/books` menampilkan flash message hijau setelah berhasil menambah buku.*

---

## Checkpoint GitHub

```bash
git add .
git commit -m "[P-3] tambah controller book category dan validasi form"
git push origin dev
```

---

## Tugas

Lengkapi `MemberController` mengikuti pola yang sama seperti `BookController`:

1. Buat `StoreMemberRequest` lewat `php artisan make:request StoreMemberRequest`, isi `rules()` sesuai desain tabel `members` (`nama`, `nim`, `email`, `nomor_telepon`, `alamat`, `status`).
2. Isi `MemberController@index` dengan data dummy array anggota, kembalikan lewat `view('members.index', compact('members'))`.
3. Isi `MemberController@create` mengembalikan view form tambah anggota.
4. Isi `MemberController@store` memakai `StoreMemberRequest`, lalu **redirect ke `members.index` dengan flash message sukses** — persis pola yang dipakai `BookController@store`.
5. Buat `resources/views/members/index.blade.php` dan `resources/views/members/create.blade.php` mengikuti gaya view yang sudah dibuat untuk `books` (HTML biasa, belum pakai master layout).

**Yang dikumpulkan:**
- Link commit GitHub (branch `dev`) yang berisi hasil tugas
- Screenshot form `/members/create` setelah submit kosong (menampilkan error validasi) dan setelah submit valid (menampilkan flash message sukses)

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-02.md) | [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-04.md)*
