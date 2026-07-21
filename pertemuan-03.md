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

Ganti isi `authorize()`, `rules()`, dan `messages()` di file yang baru dibuat. `authorize()` diubah jadi `true` karena autentikasi baru diimplementasikan di Pertemuan 8 — untuk saat ini semua orang "diizinkan" mengisi form. Berikut isi **lengkap** `StoreBookRequest` — ganti seluruh isi class-nya persis seperti ini:

```php
// File: app/Http/Requests/StoreBookRequest.php
<?php

namespace App\Http\Requests;

use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;

class StoreBookRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    /**
     * @return array<string, ValidationRule|array<mixed>|string>
     */
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
            'judul.max' => 'Judul buku maksimal 200 karakter.',
            'penulis.required' => 'Nama penulis wajib diisi.',
            'penerbit.required' => 'Nama penerbit wajib diisi.',
            'tahun_terbit.required' => 'Tahun terbit wajib diisi.',
            'tahun_terbit.integer' => 'Tahun terbit harus berupa angka.',
            'tahun_terbit.min' => 'Tahun terbit tidak valid.',
            'tahun_terbit.max' => 'Tahun terbit tidak boleh lebih dari tahun sekarang.',
            'isbn.max' => 'ISBN maksimal 20 karakter.',
            'stok.required' => 'Stok wajib diisi.',
            'stok.integer' => 'Stok harus berupa angka.',
            'stok.min' => 'Stok tidak boleh kurang dari 0.',
            'category_id.required' => 'Kategori wajib dipilih.',
        ];
    }
}
```

> ⚠️ **Penting:** `messages()` harus diisi untuk **setiap kombinasi field dan rule** yang ingin ditampilkan dalam Bahasa Indonesia. Project ini belum mengatur `APP_LOCALE` ke `id`, jadi field yang tidak dikustomisasi pesannya akan tetap tampil pesan default Bahasa Inggris dari Laravel — kalau salah satu baris di atas kelewatan, hasilnya bercampur Indonesia-Inggris.

Lakukan hal yang sama untuk `StoreCategoryRequest` — ini juga isi lengkapnya:

```php
// File: app/Http/Requests/StoreCategoryRequest.php
<?php

namespace App\Http\Requests;

use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;

class StoreCategoryRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    /**
     * @return array<string, ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'nama_kategori' => 'required|string|max:100',
            'deskripsi' => 'nullable|string',
        ];
    }

    public function messages(): array
    {
        return [
            'nama_kategori.required' => 'Nama kategori wajib diisi.',
            'nama_kategori.max' => 'Nama kategori maksimal 100 karakter.',
        ];
    }
}
```

### Langkah 2 — Mengisi BookController Secara Penuh

Karena Migration & Model Eloquent baru dibuat di Pertemuan 5, data buku untuk sementara disimpan sebagai array dummy di dalam Controller. **Semua 7 method resource diisi logika nyata** — tidak ada lagi yang mengembalikan string dummy. Perhatikan baris `use App\Http\Requests\StoreBookRequest;` di bagian atas file — ini **wajib ditambahkan sendiri**, karena skeleton Controller dari Pertemuan 2 cuma punya `use Illuminate\Http\Request;` bawaan. Tanpa baris ini, `store()` akan gagal dengan error `Class "App\Http\Controllers\StoreBookRequest" not found` begitu form disubmit.

Berikut isi **lengkap** `BookController.php` setelah diubah:

```php
// File: app/Http/Controllers/BookController.php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StoreBookRequest;
use Illuminate\Http\Request;

class BookController extends Controller
{
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

    public function create()
    {
        $categories = $this->categories;

        return view('books.create', compact('categories'));
    }

    public function store(StoreBookRequest $request)
    {
        $validated = $request->validated();

        return redirect()->route('books.index')
            ->with('success', "Buku \"{$validated['judul']}\" berhasil ditambahkan (data dummy, belum tersimpan ke database).");
    }

    public function show(string $id)
    {
        $book = collect($this->books)->firstWhere('id', (int) $id);

        abort_if(! $book, 404);

        return view('books.show', compact('book'));
    }

    public function edit(string $id)
    {
        $book = collect($this->books)->firstWhere('id', (int) $id);

        abort_if(! $book, 404);

        $categories = $this->categories;

        return view('books.edit', compact('book', 'categories'));
    }

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

    public function destroy(string $id)
    {
        return redirect()->route('books.index')
            ->with('success', "Buku dengan id {$id} berhasil dihapus (data dummy, belum tersimpan ke database).");
    }
}
```

Beberapa hal yang perlu diperhatikan dari kode di atas:
- `create()` dan `edit()` sama-sama mengirim `$categories` ke view, karena keduanya menampilkan dropdown pilih kategori.
- `show()` dan `edit()` mencari buku dari array dummy dengan `collect(...)->firstWhere('id', ...)`, lalu `abort_if(! $book, 404)` melempar halaman 404 kalau id tidak ditemukan — perilaku ini nanti otomatis digantikan `findOrFail()` saat Eloquent masuk di Pertemuan 5.
- `update()` sengaja ditulis memakai validasi **inline** (`$request->validate()`), bukan Form Request, supaya kamu bisa membandingkan langsung kedua gaya di project yang sama.
- `destroy()` untuk saat ini hanya mengembalikan flash message sukses tanpa benar-benar menghapus apa pun dari `$books`, karena array ini dibuat ulang dari nol setiap request (lihat bagian "Lifecycle Controller" di atas) — belum ada penyimpanan data yang sungguhan.

### Langkah 3 — Membuat View Buku (Tanpa Master Layout Dulu)

Master layout (`layouts/app.blade.php`) baru dibangun di Pertemuan 4. Untuk saat ini, view ditulis sebagai HTML biasa lengkap dengan `<!DOCTYPE html>` sendiri-sendiri, cukup untuk mendemonstrasikan data dummy dan validasi.

Berikut isi lengkap `books/index.blade.php` — tabel daftar buku, flash message sukses, dan link aksi (Detail/Edit/Hapus) di tiap baris:

```blade
{{-- File: resources/views/books/index.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Daftar Buku</title>
    <style>
        body { font-family: sans-serif; margin: 40px; }
        table { border-collapse: collapse; width: 100%; margin-top: 16px; }
        th, td { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
        .success { background: #d1fae5; color: #065f46; padding: 10px 14px; border-radius: 4px; margin-top: 16px; }
        .btn { display: inline-block; padding: 6px 14px; background: #2563eb; color: #fff; text-decoration: none; border-radius: 4px; }
        form.inline { display: inline; }
    </style>
</head>
<body>
    <h1>Daftar Buku</h1>

    @if (session('success'))
        <div class="success">{{ session('success') }}</div>
    @endif

    <p><a href="{{ route('books.create') }}" class="btn">+ Tambah Buku</a></p>

    <table>
        <thead>
            <tr>
                <th>ID</th>
                <th>Judul</th>
                <th>Penulis</th>
                <th>Penerbit</th>
                <th>Tahun</th>
                <th>Stok</th>
                <th>Kategori</th>
                <th>Aksi</th>
            </tr>
        </thead>
        <tbody>
            @forelse ($books as $book)
                <tr>
                    <td>{{ $book['id'] }}</td>
                    <td>{{ $book['judul'] }}</td>
                    <td>{{ $book['penulis'] }}</td>
                    <td>{{ $book['penerbit'] }}</td>
                    <td>{{ $book['tahun_terbit'] }}</td>
                    <td>{{ $book['stok'] }}</td>
                    <td>{{ $book['kategori'] }}</td>
                    <td>
                        <a href="{{ route('books.show', $book['id']) }}">Detail</a>
                        |
                        <a href="{{ route('books.edit', $book['id']) }}">Edit</a>
                        |
                        <form class="inline" action="{{ route('books.destroy', $book['id']) }}" method="POST">
                            @csrf
                            @method('DELETE')
                            <button type="submit">Hapus</button>
                        </form>
                    </td>
                </tr>
            @empty
                <tr>
                    <td colspan="8">Belum ada data buku.</td>
                </tr>
            @endforelse
        </tbody>
    </table>

    <p><em>Catatan: data di atas masih data dummy (array statis di Controller), belum dari database. Migration &amp; Model Eloquent baru dibuat di Pertemuan 5.</em></p>
</body>
</html>
```

Selanjutnya `books/create.blade.php` — form tambah buku dengan dropdown kategori dan `@error` di setiap field:

```blade
{{-- File: resources/views/books/create.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Tambah Buku</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 500px; }
        label { display: block; margin-top: 12px; font-weight: bold; }
        input, select { width: 100%; padding: 6px; margin-top: 4px; box-sizing: border-box; }
        .error { color: #b91c1c; font-size: 14px; margin-top: 4px; }
        .btn { margin-top: 20px; padding: 8px 16px; background: #2563eb; color: #fff; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <h1>Tambah Buku</h1>
    <p><a href="{{ route('books.index') }}">&larr; Kembali ke daftar buku</a></p>

    <form action="{{ route('books.store') }}" method="POST">
        @csrf

        <label for="judul">Judul</label>
        <input type="text" name="judul" id="judul" value="{{ old('judul') }}">
        @error('judul')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="penulis">Penulis</label>
        <input type="text" name="penulis" id="penulis" value="{{ old('penulis') }}">
        @error('penulis')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="penerbit">Penerbit</label>
        <input type="text" name="penerbit" id="penerbit" value="{{ old('penerbit') }}">
        @error('penerbit')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="tahun_terbit">Tahun Terbit</label>
        <input type="number" name="tahun_terbit" id="tahun_terbit" value="{{ old('tahun_terbit') }}">
        @error('tahun_terbit')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="isbn">ISBN (opsional)</label>
        <input type="text" name="isbn" id="isbn" value="{{ old('isbn') }}">
        @error('isbn')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="stok">Stok</label>
        <input type="number" name="stok" id="stok" value="{{ old('stok', 1) }}">
        @error('stok')
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
        @error('category_id')
            <div class="error">{{ $message }}</div>
        @enderror

        <button type="submit" class="btn">Simpan</button>
    </form>
</body>
</html>
```

> 📸 *Screenshot: form "Tambah Buku" setelah submit kosong — setiap field menampilkan pesan error berwarna merah di bawahnya.*

`books/edit.blade.php` isinya mirip `create.blade.php`, tapi ada tiga perbedaan penting: `<form>` mengarah ke `route('books.update', $book['id'])`, ditambahkan `@method('PUT')` (karena form HTML biasa cuma bisa mengirim GET/POST, bukan PUT), dan setiap input memakai `old('field', $book['field'])` — jadi kalau belum ada input lama (baru pertama kali buka form), yang tampil adalah data buku yang mau diedit; kalau validasi gagal, yang tampil adalah input terakhir yang diketik user:

```blade
{{-- File: resources/views/books/edit.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Edit Buku</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 500px; }
        label { display: block; margin-top: 12px; font-weight: bold; }
        input, select { width: 100%; padding: 6px; margin-top: 4px; box-sizing: border-box; }
        .error { color: #b91c1c; font-size: 14px; margin-top: 4px; }
        .btn { margin-top: 20px; padding: 8px 16px; background: #2563eb; color: #fff; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <h1>Edit Buku</h1>
    <p><a href="{{ route('books.index') }}">&larr; Kembali ke daftar buku</a></p>

    <form action="{{ route('books.update', $book['id']) }}" method="POST">
        @csrf
        @method('PUT')

        <label for="judul">Judul</label>
        <input type="text" name="judul" id="judul" value="{{ old('judul', $book['judul']) }}">
        @error('judul')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="penulis">Penulis</label>
        <input type="text" name="penulis" id="penulis" value="{{ old('penulis', $book['penulis']) }}">
        @error('penulis')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="penerbit">Penerbit</label>
        <input type="text" name="penerbit" id="penerbit" value="{{ old('penerbit', $book['penerbit']) }}">
        @error('penerbit')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="tahun_terbit">Tahun Terbit</label>
        <input type="number" name="tahun_terbit" id="tahun_terbit" value="{{ old('tahun_terbit', $book['tahun_terbit']) }}">
        @error('tahun_terbit')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="isbn">ISBN (opsional)</label>
        <input type="text" name="isbn" id="isbn" value="{{ old('isbn', $book['isbn']) }}">
        @error('isbn')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="stok">Stok</label>
        <input type="number" name="stok" id="stok" value="{{ old('stok', $book['stok']) }}">
        @error('stok')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="category_id">Kategori</label>
        <select name="category_id" id="category_id">
            <option value="">-- Pilih Kategori --</option>
            @foreach ($categories as $category)
                <option value="{{ $category['id'] }}" @selected(old('category_id', $book['category_id']) == $category['id'])>
                    {{ $category['nama_kategori'] }}
                </option>
            @endforeach
        </select>
        @error('category_id')
            <div class="error">{{ $message }}</div>
        @enderror

        <button type="submit" class="btn">Perbarui</button>
    </form>
</body>
</html>
```

> ⚠️ Perhatikan `@selected(old('category_id', $book['category_id']) == $category['id'])` — bagian `$book['category_id']` di situ yang membuat dropdown otomatis ter-select ke kategori buku yang sedang diedit. Kalau ditulis `old('category_id')` saja tanpa fallback, dropdown akan selalu kembali ke "-- Pilih Kategori --" setiap form dibuka pertama kali, meskipun buku itu sebenarnya sudah punya kategori.

Terakhir `books/show.blade.php` — halaman detail satu buku, ditampilkan sebagai tabel key-value:

```blade
{{-- File: resources/views/books/show.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Detail Buku</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 500px; }
        table { border-collapse: collapse; width: 100%; margin-top: 16px; }
        th, td { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
        th { width: 160px; background: #f3f4f6; }
    </style>
</head>
<body>
    <h1>Detail Buku</h1>
    <p><a href="{{ route('books.index') }}">&larr; Kembali ke daftar buku</a></p>

    <table>
        <tr>
            <th>Judul</th>
            <td>{{ $book['judul'] }}</td>
        </tr>
        <tr>
            <th>Penulis</th>
            <td>{{ $book['penulis'] }}</td>
        </tr>
        <tr>
            <th>Penerbit</th>
            <td>{{ $book['penerbit'] }}</td>
        </tr>
        <tr>
            <th>Tahun Terbit</th>
            <td>{{ $book['tahun_terbit'] }}</td>
        </tr>
        <tr>
            <th>ISBN</th>
            <td>{{ $book['isbn'] ?? '-' }}</td>
        </tr>
        <tr>
            <th>Stok</th>
            <td>{{ $book['stok'] }}</td>
        </tr>
        <tr>
            <th>Kategori</th>
            <td>{{ $book['kategori'] }}</td>
        </tr>
    </table>
</body>
</html>
```

### Langkah 4 — Mengisi CategoryController (index & store)

Sesuai target pertemuan ini, hanya `index()`, `create()`, dan `store()` yang diisi logika nyata. `edit()`, `update()`, `destroy()` **tetap dibiarkan seperti kerangka Pertemuan 2** (return string dummy) — CRUD kategori yang lengkap baru selesai di Pertemuan 5. Sama seperti `BookController`, jangan lupa tambahkan baris `use App\Http\Requests\StoreCategoryRequest;` di bagian atas file.

Berikut isi lengkap `CategoryController.php`:

```php
// File: app/Http/Controllers/CategoryController.php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StoreCategoryRequest;
use Illuminate\Http\Request;

class CategoryController extends Controller
{
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

    public function edit(string $id)
    {
        return "CategoryController@edit, id: {$id}";
    }

    public function update(Request $request, string $id)
    {
        return "CategoryController@update, id: {$id}";
    }

    public function destroy(string $id)
    {
        return "CategoryController@destroy, id: {$id}";
    }
}
```

### Langkah 5 — Membuat View Kategori

Berikut isi lengkap `categories/index.blade.php`:

```blade
{{-- File: resources/views/categories/index.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Daftar Kategori</title>
    <style>
        body { font-family: sans-serif; margin: 40px; }
        table { border-collapse: collapse; width: 100%; margin-top: 16px; }
        th, td { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
        .success { background: #d1fae5; color: #065f46; padding: 10px 14px; border-radius: 4px; margin-top: 16px; }
        .btn { display: inline-block; padding: 6px 14px; background: #2563eb; color: #fff; text-decoration: none; border-radius: 4px; }
    </style>
</head>
<body>
    <h1>Daftar Kategori</h1>

    @if (session('success'))
        <div class="success">{{ session('success') }}</div>
    @endif

    <p><a href="{{ route('categories.create') }}" class="btn">+ Tambah Kategori</a></p>

    <table>
        <thead>
            <tr>
                <th>ID</th>
                <th>Nama Kategori</th>
                <th>Deskripsi</th>
                <th>Aksi</th>
            </tr>
        </thead>
        <tbody>
            @forelse ($categories as $category)
                <tr>
                    <td>{{ $category['id'] }}</td>
                    <td>{{ $category['nama_kategori'] }}</td>
                    <td>{{ $category['deskripsi'] ?? '-' }}</td>
                    <td>
                        <a href="{{ route('categories.edit', $category['id']) }}">Edit</a>
                        |
                        <form style="display:inline" action="{{ route('categories.destroy', $category['id']) }}" method="POST">
                            @csrf
                            @method('DELETE')
                            <button type="submit">Hapus</button>
                        </form>
                    </td>
                </tr>
            @empty
                <tr>
                    <td colspan="4">Belum ada data kategori.</td>
                </tr>
            @endforelse
        </tbody>
    </table>

    <p><em>Catatan: data di atas masih data dummy (array statis di Controller), belum dari database. Migration &amp; Model Eloquent baru dibuat di Pertemuan 5.</em></p>
</body>
</html>
```

> ⚠️ Link "Edit" dan tombol "Hapus" di halaman ini mengarah ke `CategoryController@edit`/`@destroy` yang **masih dummy** (lihat Langkah 4) — klik keduanya masih akan menampilkan teks polos seperti `CategoryController@edit, id: 1`, bukan halaman edit sungguhan. Ini memang disengaja untuk pertemuan ini; CRUD kategori lengkap baru dikerjakan di Pertemuan 5.

Dan `categories/create.blade.php`:

```blade
{{-- File: resources/views/categories/create.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Tambah Kategori</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 500px; }
        label { display: block; margin-top: 12px; font-weight: bold; }
        input, textarea { width: 100%; padding: 6px; margin-top: 4px; box-sizing: border-box; }
        .error { color: #b91c1c; font-size: 14px; margin-top: 4px; }
        .btn { margin-top: 20px; padding: 8px 16px; background: #2563eb; color: #fff; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <h1>Tambah Kategori</h1>
    <p><a href="{{ route('categories.index') }}">&larr; Kembali ke daftar kategori</a></p>

    <form action="{{ route('categories.store') }}" method="POST">
        @csrf

        <label for="nama_kategori">Nama Kategori</label>
        <input type="text" name="nama_kategori" id="nama_kategori" value="{{ old('nama_kategori') }}">
        @error('nama_kategori')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="deskripsi">Deskripsi (opsional)</label>
        <textarea name="deskripsi" id="deskripsi" rows="4">{{ old('deskripsi') }}</textarea>
        @error('deskripsi')
            <div class="error">{{ $message }}</div>
        @enderror

        <button type="submit" class="btn">Simpan</button>
    </form>
</body>
</html>
```

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
