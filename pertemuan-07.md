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

`MemberController` diubah dari array dummy ke Eloquent penuh, mengikuti pola persis `CategoryController` di Pertemuan 5. Validasi saat membuat anggota baru memakai Form Request tersendiri:

```php
// File: app/Http/Requests/StoreMemberRequest.php
<?php

namespace App\Http\Requests;

use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;

class StoreMemberRequest extends FormRequest
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
            'nama' => 'required|string|max:100',
            'nim' => 'required|string|max:20|unique:members,nim',
            'email' => 'required|email|max:100|unique:members,email',
            'nomor_telepon' => 'required|string|max:15',
            'alamat' => 'required|string',
            'status' => 'required|in:aktif,nonaktif',
        ];
    }

    public function messages(): array
    {
        return [
            'nama.required' => 'Nama anggota wajib diisi.',
            'nim.required' => 'NIM wajib diisi.',
            'nim.unique' => 'NIM sudah terdaftar.',
            'email.required' => 'Email wajib diisi.',
            'email.email' => 'Format email tidak valid.',
            'email.unique' => 'Email sudah terdaftar.',
            'nomor_telepon.required' => 'Nomor telepon wajib diisi.',
            'alamat.required' => 'Alamat wajib diisi.',
            'status.required' => 'Status wajib dipilih.',
            'status.in' => 'Status tidak valid.',
        ];
    }
}
```

Dibuat lewat `php artisan make:request StoreMemberRequest`, lalu isi seperti di atas. Setelah itu, seluruh method `MemberController` diisi — `index`/`store` memakai pola yang sama seperti `CategoryController`, sedangkan `edit`/`update`/`destroy` memakai `findOrFail()` seperti biasa (method `show` sengaja dibahas terpisah di Langkah 4, karena di situ baru masuk eager loading relasi `loans`):

```php
// File: app/Http/Controllers/MemberController.php
use App\Http\Requests\StoreMemberRequest;
use App\Models\Member;
use Illuminate\Http\Request;

public function index()
{
    $members = Member::paginate(10);

    return view('members.index', compact('members'));
}

public function create()
{
    return view('members.create');
}

public function store(StoreMemberRequest $request)
{
    $validated = $request->validated();

    Member::create($validated);

    return redirect()->route('members.index')
        ->with('success', "Anggota \"{$validated['nama']}\" berhasil ditambahkan.");
}

public function edit(string $id)
{
    $member = Member::findOrFail($id);

    return view('members.edit', compact('member'));
}

public function update(Request $request, string $id)
{
    $member = Member::findOrFail($id);

    $validated = $request->validate([
        'nama' => 'required|string|max:100',
        'nim' => 'required|string|max:20|unique:members,nim,'.$member->id,
        'email' => 'required|email|max:100|unique:members,email,'.$member->id,
        'nomor_telepon' => 'required|string|max:15',
        'alamat' => 'required|string',
        'status' => 'required|in:aktif,nonaktif',
    ]);

    $member->update($validated);

    return redirect()->route('members.index')
        ->with('success', "Anggota \"{$validated['nama']}\" berhasil diperbarui.");
}

public function destroy(string $id)
{
    $member = Member::findOrFail($id);
    $member->delete();

    return redirect()->route('members.index')
        ->with('success', 'Anggota berhasil dihapus.');
}
```

`nim`/`email` di `update()` menambahkan `,'.$member->id` di akhir aturan `unique` — supaya validasi tidak menolak anggota itu sendiri saat NIM/email-nya tidak berubah (Laravel `unique` perlu tahu baris mana yang boleh dikecualikan saat sedang mengedit baris itu sendiri).

View-nya dibuat dengan struktur halaman lengkap yang sama seperti `books/create.blade.php`/`edit.blade.php` (HTML polos, bukan master layout — cuma isi field yang disesuaikan ke kolom `members`):

```blade
{{-- File: resources/views/members/create.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Tambah Anggota</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 500px; }
        label { display: block; margin-top: 12px; font-weight: bold; }
        input, select, textarea { width: 100%; padding: 6px; margin-top: 4px; box-sizing: border-box; }
        .error { color: #b91c1c; font-size: 14px; margin-top: 4px; }
        .btn { margin-top: 20px; padding: 8px 16px; background: #2563eb; color: #fff; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <h1>Tambah Anggota</h1>
    <p><a href="{{ route('members.index') }}">&larr; Kembali ke daftar anggota</a></p>

    <form action="{{ route('members.store') }}" method="POST">
        @csrf

        <label for="nama">Nama</label>
        <input type="text" name="nama" id="nama" value="{{ old('nama') }}">
        @error('nama')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="nim">NIM</label>
        <input type="text" name="nim" id="nim" value="{{ old('nim') }}">
        @error('nim')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="email">Email</label>
        <input type="email" name="email" id="email" value="{{ old('email') }}">
        @error('email')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="nomor_telepon">Nomor Telepon</label>
        <input type="text" name="nomor_telepon" id="nomor_telepon" value="{{ old('nomor_telepon') }}">
        @error('nomor_telepon')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="alamat">Alamat</label>
        <textarea name="alamat" id="alamat" rows="3">{{ old('alamat') }}</textarea>
        @error('alamat')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="status">Status</label>
        <select name="status" id="status">
            <option value="aktif" @selected(old('status', 'aktif') == 'aktif')>Aktif</option>
            <option value="nonaktif" @selected(old('status') == 'nonaktif')>Nonaktif</option>
        </select>
        @error('status')
            <div class="error">{{ $message }}</div>
        @enderror

        <button type="submit" class="btn">Simpan</button>
    </form>
</body>
</html>
```

`edit.blade.php` sama persis strukturnya, cuma judul, `action` diarahkan ke `members.update`, ditambah `@method('PUT')`, dan tiap `value`/isi field memakai `old('field', $member['field'])` supaya ter-prefill data lama:

```blade
{{-- File: resources/views/members/edit.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Edit Anggota</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 500px; }
        label { display: block; margin-top: 12px; font-weight: bold; }
        input, select, textarea { width: 100%; padding: 6px; margin-top: 4px; box-sizing: border-box; }
        .error { color: #b91c1c; font-size: 14px; margin-top: 4px; }
        .btn { margin-top: 20px; padding: 8px 16px; background: #2563eb; color: #fff; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <h1>Edit Anggota</h1>
    <p><a href="{{ route('members.index') }}">&larr; Kembali ke daftar anggota</a></p>

    <form action="{{ route('members.update', $member['id']) }}" method="POST">
        @csrf
        @method('PUT')

        <label for="nama">Nama</label>
        <input type="text" name="nama" id="nama" value="{{ old('nama', $member['nama']) }}">
        @error('nama')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="nim">NIM</label>
        <input type="text" name="nim" id="nim" value="{{ old('nim', $member['nim']) }}">
        @error('nim')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="email">Email</label>
        <input type="email" name="email" id="email" value="{{ old('email', $member['email']) }}">
        @error('email')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="nomor_telepon">Nomor Telepon</label>
        <input type="text" name="nomor_telepon" id="nomor_telepon" value="{{ old('nomor_telepon', $member['nomor_telepon']) }}">
        @error('nomor_telepon')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="alamat">Alamat</label>
        <textarea name="alamat" id="alamat" rows="3">{{ old('alamat', $member['alamat']) }}</textarea>
        @error('alamat')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="status">Status</label>
        <select name="status" id="status">
            <option value="aktif" @selected(old('status', $member['status']) == 'aktif')>Aktif</option>
            <option value="nonaktif" @selected(old('status', $member['status']) == 'nonaktif')>Nonaktif</option>
        </select>
        @error('status')
            <div class="error">{{ $message }}</div>
        @enderror

        <button type="submit" class="btn">Perbarui</button>
    </form>
</body>
</html>
```

Terakhir, `members/index.blade.php` yang sebelumnya cuma menampilkan data dummy tanpa aksi apa pun, diperbarui supaya benar-benar bisa dipakai untuk masuk ke `create`/`show`/`edit`/`destroy` — tanpa langkah ini, mahasiswa tidak akan punya cara mengklik ke halaman-halaman yang baru dibuat:

```blade
{{-- File: resources/views/members/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Daftar Anggota')

@section('content')
    <h1>Daftar Anggota</h1>

    <p><a href="{{ route('members.create') }}" class="btn">+ Tambah Anggota</a></p>

    <table>
        <thead>
            <tr>
                <th>ID</th>
                <th>Nama</th>
                <th>NIM</th>
                <th>Email</th>
                <th>No. Telepon</th>
                <th>Status</th>
                <th>Aksi</th>
            </tr>
        </thead>
        <tbody>
            @forelse ($members as $member)
                <tr>
                    <td>{{ $member['id'] }}</td>
                    <td>{{ $member['nama'] }}</td>
                    <td>{{ $member['nim'] }}</td>
                    <td>{{ $member['email'] }}</td>
                    <td>{{ $member['nomor_telepon'] }}</td>
                    <td>{{ ucfirst($member['status']) }}</td>
                    <td>
                        <a href="{{ route('members.show', $member['id']) }}">Detail</a>
                        |
                        <a href="{{ route('members.edit', $member['id']) }}">Edit</a>
                        |
                        <form class="inline" action="{{ route('members.destroy', $member['id']) }}" method="POST">
                            @csrf
                            @method('DELETE')
                            <button type="submit">Hapus</button>
                        </form>
                    </td>
                </tr>
            @empty
                <tr>
                    <td colspan="7">Belum ada data anggota.</td>
                </tr>
            @endforelse
        </tbody>
    </table>

    {{ $members->links() }}
@endsection
```

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
echo 'Tanpa eager loading: '.count(DB::getQueryLog()).' query'.PHP_EOL;

DB::flushQueryLog();
$books2 = App\Models\Book::with('category')->get();
foreach ($books2 as $book) { $nama = $book->category->nama_kategori; }
echo 'Dengan eager loading: '.count(DB::getQueryLog()).' query'.PHP_EOL;
```

> Tambahkan `.PHP_EOL` dan label teks seperti di atas — kalau cuma `echo count(...)` dipanggil dua kali berturut-turut, hasilnya nempel jadi satu angka (mis. "32") di terminal tanpa pemisah baris, membingungkan dibaca padahal sebenarnya dua angka terpisah (3 lalu 2).

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

Kolom "ID Kategori" di `books/index.blade.php` dan "ID Kategori" di `books/show.blade.php` diganti jadi "Kategori" — baik teks header (`<th>`) maupun cara mengambil datanya, mengakses nama aslinya lewat relasi:

```blade
{{-- File: resources/views/books/index.blade.php --}}
<th>Kategori</th>  {{-- sebelumnya: <th>ID Kategori</th> --}}
```

```blade
{{-- File: resources/views/books/index.blade.php --}}
<td>{{ $book['category']['nama_kategori'] }}</td>  {{-- sebelumnya: {{ $book['category_id'] }} --}}
```

Baris paling bawah tabel info juga ikut berubah polanya sama di `books/show.blade.php` (halaman ini masih HTML polos tanpa master layout, jadi strukturnya tabel dua kolom `<th>`/`<td>`, bukan tabel daftar seperti index):

```blade
{{-- File: resources/views/books/show.blade.php --}}
<tr>
    <th>Kategori</th>  {{-- sebelumnya: <th>ID Kategori</th> --}}
    <td>{{ $book['category']['nama_kategori'] }}</td>  {{-- sebelumnya: {{ $book['category_id'] }} --}}
</tr>
```

Baris catatan "kolom kategori masih menampilkan ID, dipelajari di Pertemuan 7" yang sebelumnya ada di bawah tabel `books/index.blade.php` juga dihapus — sudah tidak relevan sejak langkah ini.

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

View `members/show.blade.php` (halaman baru, belum ada sebelum langkah ini) menampilkan data anggota di tabel info seperti pola `books/show.blade.php`, ditambah tabel riwayat peminjaman yang di-loop dari `$member['loans']`:

```blade
{{-- File: resources/views/members/show.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Detail Anggota</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 700px; }
        table { border-collapse: collapse; width: 100%; margin-top: 16px; }
        th, td { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
        .info th { width: 160px; background: #f3f4f6; }
    </style>
</head>
<body>
    <h1>Detail Anggota</h1>
    <p><a href="{{ route('members.index') }}">&larr; Kembali ke daftar anggota</a></p>

    <table class="info">
        <tr>
            <th>Nama</th>
            <td>{{ $member['nama'] }}</td>
        </tr>
        <tr>
            <th>NIM</th>
            <td>{{ $member['nim'] }}</td>
        </tr>
        <tr>
            <th>Email</th>
            <td>{{ $member['email'] }}</td>
        </tr>
        <tr>
            <th>Nomor Telepon</th>
            <td>{{ $member['nomor_telepon'] }}</td>
        </tr>
        <tr>
            <th>Alamat</th>
            <td>{{ $member['alamat'] }}</td>
        </tr>
        <tr>
            <th>Status</th>
            <td>{{ ucfirst($member['status']) }}</td>
        </tr>
    </table>

    <h2>Riwayat Peminjaman</h2>
    <p><em>Diambil lewat relasi <code>$member->loans</code> — satu anggota bisa punya banyak transaksi peminjaman.</em></p>

    <table>
        <thead>
            <tr>
                <th>Tanggal Pinjam</th>
                <th>Tanggal Kembali</th>
                <th>Petugas</th>
                <th>Buku</th>
                <th>Status</th>
            </tr>
        </thead>
        <tbody>
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
                <tr>
                    <td colspan="5">Anggota ini belum pernah meminjam buku.</td>
                </tr>
            @endforelse
        </tbody>
    </table>
</body>
</html>
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

Sebelum `store()` bisa dipakai, form-nya perlu data anggota, buku, dan petugas untuk mengisi dropdown/checkbox — ini tugas `create()`, yang **wajib dibuat lebih dulu** sebelum `/loans/create` bisa diakses sama sekali (tanpa method ini, view akan error karena variabel `$members`/`$books`/`$users` belum dikirim):

```php
// File: app/Http/Controllers/LoanController.php
public function create()
{
    $members = Member::all();
    $books = Book::all();
    $users = User::all();

    return view('loans.create', compact('members', 'books', 'users'));
}
```

`use App\Models\Book;`, `use App\Models\Member;`, dan `use App\Models\User;` ditambahkan di bagian atas `LoanController.php` untuk mendukung `create()` di atas.

Form create (`loans/create.blade.php`) menyediakan dropdown anggota dan petugas, input tanggal, serta daftar checkbox buku (`name="book_ids[]"`) supaya satu peminjaman bisa mencakup lebih dari satu buku sekaligus:

```blade
{{-- File: resources/views/loans/create.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Tambah Peminjaman</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 500px; }
        label { display: block; margin-top: 12px; font-weight: bold; }
        input, select { width: 100%; padding: 6px; margin-top: 4px; box-sizing: border-box; }
        .checkbox-list { border: 1px solid #ccc; border-radius: 4px; padding: 10px; margin-top: 4px; max-height: 200px; overflow-y: auto; }
        .checkbox-list label { display: block; font-weight: normal; margin-top: 0; }
        .error { color: #b91c1c; font-size: 14px; margin-top: 4px; }
        .btn { margin-top: 20px; padding: 8px 16px; background: #2563eb; color: #fff; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <h1>Tambah Peminjaman</h1>
    <p><a href="{{ route('loans.index') }}">&larr; Kembali ke daftar peminjaman</a></p>

    <form action="{{ route('loans.store') }}" method="POST">
        @csrf

        <label for="member_id">Anggota</label>
        <select name="member_id" id="member_id">
            <option value="">-- Pilih Anggota --</option>
            @foreach ($members as $member)
                <option value="{{ $member['id'] }}" @selected(old('member_id') == $member['id'])>
                    {{ $member['nama'] }} ({{ $member['nim'] }})
                </option>
            @endforeach
        </select>
        @error('member_id')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="user_id">Petugas</label>
        <select name="user_id" id="user_id">
            <option value="">-- Pilih Petugas --</option>
            @foreach ($users as $user)
                <option value="{{ $user['id'] }}" @selected(old('user_id') == $user['id'])>
                    {{ $user['name'] }}
                </option>
            @endforeach
        </select>
        @error('user_id')
            <div class="error">{{ $message }}</div>
        @enderror
        <p><em>Catatan: dropdown petugas dipilih manual karena login belum ada — otomatis dari user yang login mulai Pertemuan 8.</em></p>

        <label for="tanggal_pinjam">Tanggal Pinjam</label>
        <input type="date" name="tanggal_pinjam" id="tanggal_pinjam" value="{{ old('tanggal_pinjam') }}">
        @error('tanggal_pinjam')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="tanggal_kembali">Tanggal Kembali</label>
        <input type="date" name="tanggal_kembali" id="tanggal_kembali" value="{{ old('tanggal_kembali') }}">
        @error('tanggal_kembali')
            <div class="error">{{ $message }}</div>
        @enderror

        <label>Buku yang Dipinjam</label>
        <div class="checkbox-list">
            @forelse ($books as $book)
                <label>
                    <input type="checkbox" name="book_ids[]" value="{{ $book['id'] }}"
                        @checked(in_array($book['id'], old('book_ids', [])))>
                    {{ $book['judul'] }} (stok: {{ $book['stok'] }})
                </label>
            @empty
                <p>Belum ada data buku.</p>
            @endforelse
        </div>
        @error('book_ids')
            <div class="error">{{ $message }}</div>
        @enderror

        <button type="submit" class="btn">Simpan</button>
    </form>
</body>
</html>
```

`index()` dan `show()` memuat ketiga relasi sekaligus supaya tidak terjadi N+1 saat menampilkan daftar maupun detail peminjaman:

```php
// File: app/Http/Controllers/LoanController.php
public function index()
{
    $loans = Loan::with(['member', 'user', 'loanItems.book'])->paginate(10);

    return view('loans.index', compact('loans'));
}

public function show(string $id)
{
    $loan = Loan::with(['member', 'user', 'loanItems.book'])->findOrFail($id);

    return view('loans.show', compact('loan'));
}
```

`loans/index.blade.php` pakai master layout seperti `books/index.blade.php`, menampilkan nama anggota/petugas/buku lewat relasi (bukan ID), dan daftar judul buku digabung koma kalau lebih dari satu lewat `$loop->last`:

```blade
{{-- File: resources/views/loans/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Daftar Peminjaman')

@section('content')
    <h1>Daftar Peminjaman</h1>

    <p><a href="{{ route('loans.create') }}" class="btn">+ Tambah Peminjaman</a></p>

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
                <th>Aksi</th>
            </tr>
        </thead>
        <tbody>
            @forelse ($loans as $loan)
                <tr>
                    <td>{{ $loan['id'] }}</td>
                    <td>{{ $loan['member']['nama'] }}</td>
                    <td>{{ $loan['user']['name'] }}</td>
                    <td>
                        @foreach ($loan['loanItems'] as $item)
                            {{ $item['book']['judul'] }}@if (!$loop->last), @endif
                        @endforeach
                    </td>
                    <td>{{ $loan['tanggal_pinjam'] }}</td>
                    <td>{{ $loan['tanggal_kembali'] }}</td>
                    <td>{{ ucfirst($loan['status']) }}</td>
                    <td>
                        <a href="{{ route('loans.show', $loan['id']) }}">Detail</a>
                        |
                        <a href="{{ route('loans.edit', $loan['id']) }}">Edit</a>
                        |
                        <form class="inline" action="{{ route('loans.destroy', $loan['id']) }}" method="POST">
                            @csrf
                            @method('DELETE')
                            <button type="submit">Hapus</button>
                        </form>
                    </td>
                </tr>
            @empty
                <tr>
                    <td colspan="8">Belum ada data peminjaman.</td>
                </tr>
            @endforelse
        </tbody>
    </table>

    {{ $loans->links() }}
@endsection
```

`loans/show.blade.php` (halaman baru) menampilkan detail satu transaksi beserta seluruh buku yang dipinjam di transaksi itu:

```blade
{{-- File: resources/views/loans/show.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Detail Peminjaman</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 600px; }
        table { border-collapse: collapse; width: 100%; margin-top: 16px; }
        th, td { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
        .info th { width: 160px; background: #f3f4f6; }
    </style>
</head>
<body>
    <h1>Detail Peminjaman</h1>
    <p><a href="{{ route('loans.index') }}">&larr; Kembali ke daftar peminjaman</a></p>

    <table class="info">
        <tr>
            <th>Anggota</th>
            <td>{{ $loan['member']['nama'] }} ({{ $loan['member']['nim'] }})</td>
        </tr>
        <tr>
            <th>Petugas</th>
            <td>{{ $loan['user']['name'] }}</td>
        </tr>
        <tr>
            <th>Tanggal Pinjam</th>
            <td>{{ $loan['tanggal_pinjam'] }}</td>
        </tr>
        <tr>
            <th>Tanggal Kembali</th>
            <td>{{ $loan['tanggal_kembali'] }}</td>
        </tr>
        <tr>
            <th>Tanggal Dikembalikan</th>
            <td>{{ $loan['tanggal_dikembalikan'] ?? '-' }}</td>
        </tr>
        <tr>
            <th>Status</th>
            <td>{{ ucfirst($loan['status']) }}</td>
        </tr>
    </table>

    <h2>Buku yang Dipinjam</h2>
    <table>
        <thead>
            <tr>
                <th>Judul</th>
                <th>Penulis</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($loan['loanItems'] as $item)
                <tr>
                    <td>{{ $item['book']['judul'] }}</td>
                    <td>{{ $item['book']['penulis'] }}</td>
                </tr>
            @endforeach
        </tbody>
    </table>
</body>
</html>
```

Method `edit`/`update`/`destroy` melengkapi CRUD standar. `edit()` sengaja **tidak** mengizinkan mengganti anggota atau buku yang dipinjam — cuma tanggal kembali dan status yang bisa diubah, karena mengubah anggota/buku di tengah transaksi yang sudah berjalan lebih berisiko menimbulkan data tidak konsisten daripada bermanfaat untuk kasus penggunaan aplikasi ini:

```php
// File: app/Http/Controllers/LoanController.php
public function edit(string $id)
{
    $loan = Loan::with(['member', 'user', 'loanItems.book'])->findOrFail($id);

    return view('loans.edit', compact('loan'));
}

public function update(Request $request, string $id)
{
    $loan = Loan::findOrFail($id);

    $validated = $request->validate([
        'tanggal_kembali' => 'required|date|after_or_equal:tanggal_pinjam',
        'status' => 'required|in:dipinjam,dikembalikan,terlambat',
    ], [
        'tanggal_kembali.required' => 'Tanggal kembali wajib diisi.',
        'tanggal_kembali.after_or_equal' => 'Tanggal kembali tidak boleh sebelum tanggal pinjam.',
    ]);

    $loan->update($validated);

    return redirect()->route('loans.index')
        ->with('success', 'Transaksi peminjaman berhasil diperbarui.');
}

public function destroy(string $id)
{
    $loan = Loan::findOrFail($id);
    $loan->loanItems()->delete();
    $loan->delete();

    return redirect()->route('loans.index')
        ->with('success', 'Transaksi peminjaman berhasil dihapus.');
}
```

Karena `loan_items` terhubung ke `loans` tanpa `cascadeOnDelete()` di migration-nya, item-nya harus dihapus lebih dulu secara eksplisit sebelum baris `loans` induknya dihapus di `destroy()`, kalau tidak MySQL akan menolak lewat error foreign key constraint.

`loans/edit.blade.php` menampilkan anggota dan buku sebagai teks baca-saja (bukan input, sesuai keputusan di atas), lalu cuma dua field yang benar-benar bisa diedit — perhatikan input `hidden` untuk `tanggal_pinjam`: field ini **wajib ada** meski tidak ditampilkan ke pengguna, karena aturan validasi `after_or_equal:tanggal_pinjam` di `update()` butuh nilai itu ada di data yang dikirim form, bukan diam-diam diambil dari database:

```blade
{{-- File: resources/views/loans/edit.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Edit Peminjaman</title>
    <style>
        body { font-family: sans-serif; margin: 40px; max-width: 500px; }
        label { display: block; margin-top: 12px; font-weight: bold; }
        input, select { width: 100%; padding: 6px; margin-top: 4px; box-sizing: border-box; }
        .error { color: #b91c1c; font-size: 14px; margin-top: 4px; }
        .btn { margin-top: 20px; padding: 8px 16px; background: #2563eb; color: #fff; border: none; border-radius: 4px; cursor: pointer; }
        .readonly { background: #f3f4f6; padding: 8px; border-radius: 4px; margin-top: 4px; }
    </style>
</head>
<body>
    <h1>Edit Peminjaman</h1>
    <p><a href="{{ route('loans.index') }}">&larr; Kembali ke daftar peminjaman</a></p>

    <label>Anggota</label>
    <div class="readonly">{{ $loan['member']['nama'] }} ({{ $loan['member']['nim'] }})</div>

    <label>Buku</label>
    <div class="readonly">
        @foreach ($loan['loanItems'] as $item)
            {{ $item['book']['judul'] }}@if (!$loop->last), @endif
        @endforeach
    </div>

    <form action="{{ route('loans.update', $loan['id']) }}" method="POST">
        @csrf
        @method('PUT')
        <input type="hidden" name="tanggal_pinjam" value="{{ $loan['tanggal_pinjam'] }}">

        <label for="tanggal_kembali">Tanggal Kembali</label>
        <input type="date" name="tanggal_kembali" id="tanggal_kembali" value="{{ old('tanggal_kembali', $loan['tanggal_kembali']) }}">
        @error('tanggal_kembali')
            <div class="error">{{ $message }}</div>
        @enderror

        <label for="status">Status</label>
        <select name="status" id="status">
            <option value="dipinjam" @selected(old('status', $loan['status']) == 'dipinjam')>Dipinjam</option>
            <option value="dikembalikan" @selected(old('status', $loan['status']) == 'dikembalikan')>Dikembalikan</option>
            <option value="terlambat" @selected(old('status', $loan['status']) == 'terlambat')>Terlambat</option>
        </select>
        @error('status')
            <div class="error">{{ $message }}</div>
        @enderror

        <button type="submit" class="btn">Perbarui</button>
    </form>
</body>
</html>
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
