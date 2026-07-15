# Pertemuan 4 — Blade Templating & Layout System

> **Sebelumnya:** `BookController` dan `CategoryController` sudah berisi logika nyata dengan data dummy, ditampilkan lewat view Blade sederhana yang masing-masing menulis `<!DOCTYPE html>` sendiri-sendiri.
> **Pertemuan ini:** Struktur HTML yang berulang (head, navbar, footer) dipisahkan ke master layout, sehingga setiap view cukup fokus pada kontennya sendiri.

---

## Capaian Pembelajaran

Setelah menyelesaikan pertemuan ini, mahasiswa mampu:
1. Memahami konsep template engine dan keunggulannya dibanding PHP murni
2. Menggunakan sintaks Blade: echo aman, directive kondisi, perulangan
3. Membangun master layout reusable dengan inheritance system Blade
4. Memisahkan komponen tampilan yang berulang dengan `@include`

---

## Konsep: Mengapa Template Engine Ada?

Coba perhatikan tiga file view yang sudah dibuat di Pertemuan 3: `books/index.blade.php`, `books/create.blade.php`, `categories/index.blade.php`. Ketiganya punya `<!DOCTYPE html>`, tag `<head>`, dan blok `<style>` yang nyaris identik, ditulis ulang di setiap file. Kalau di Pertemuan 5 nanti navbar ditambahkan, atau warna tombol perlu diubah, perubahan itu harus dilakukan manual di puluhan file berbeda — dan kalau satu file terlewat, tampilannya jadi tidak konsisten. Ini bukan masalah kecil kosmetik; ini pelanggaran terhadap prinsip **DRY (Don't Repeat Yourself)**, salah satu prinsip paling dasar dalam rekayasa perangkat lunak. Semakin banyak duplikasi, semakin besar kemungkinan terjadi *drift* — bagian yang seharusnya sama tapi diam-diam jadi berbeda karena satu tempat diubah dan tempat lain lupa disentuh.

Masalah kedua yang diselesaikan template engine lebih mendasar: bagaimana caranya menampilkan data dinamis di tengah HTML tanpa membuat kodenya jadi kacau. Di PHP native murni, ini biasanya dilakukan dengan keluar-masuk tag `<?php ?>` berkali-kali di tengah HTML, atau memakai `echo htmlspecialchars($data)` berulang-ulang supaya aman dari serangan XSS (*Cross-Site Scripting*) — celah keamanan yang terjadi kalau data dari user ditampilkan mentah-mentah tanpa di-escape, sehingga user jahat bisa menyisipkan `<script>` untuk dijalankan di browser pengguna lain. Menulis `htmlspecialchars()` di setiap titik output itu gampang lupa, dan sekali lupa, aplikasi jadi rentan. Template engine seperti Blade menyelesaikan ini dengan menyediakan sintaks singkat, `{{ $data }}`, yang **secara otomatis** melakukan escaping — developer tidak perlu mengingat untuk melakukannya sendiri, itu jadi perilaku default.

Ada ironi menarik di balik masalah ini: PHP sendiri, sejak awal dibuat, memang dirancang sebagai bahasa untuk *menyisipkan* logika ke dalam HTML — nama aslinya, "Personal Home Page Tools", literally dibuat untuk keperluan itu di pertengahan 1990-an. Jadi mencampur `<?php ?>` dengan HTML bukan kesalahan pemakaian, itu memang desain awal bahasanya. Masalahnya muncul begitu aplikasi bertumbuh: di Pertemuan 1 kita sudah sepakat Controller mengurus logika (validasi, query, redirect) dan View mengurus tampilan. Tapi kalau file View boleh berisi PHP penuh tanpa batasan, tidak ada yang mencegah seorang developer diam-diam menaruh query database atau logika bisnis di tengah HTML — pelanggaran MVC yang tidak akan ketahuan sampai kode itu susah dites atau diubah. Template engine menutup celah ini bukan lewat aturan tertulis atau disiplin tim, tapi lewat **batasan sintaks**: Blade sengaja tidak menyediakan cara mudah untuk memanggil query Eloquent langsung di dalam `{{ }}` atau `@if`, memaksa logika berat itu tetap tinggal di Controller atau Model tempat seharusnya.

Konsep template engine bukan sesuatu yang eksklusif milik Laravel. Setiap framework backend besar punya solusinya sendiri untuk masalah yang sama persis. Django (Python) punya Django Template Language dengan sintaks `{{ variabel }}` dan `{% tag %}` yang mirip sekali dengan Blade. Express.js (Node.js) biasanya dipasangkan dengan EJS atau Pug, yang juga menyediakan syntax singkat untuk echo dan kontrol alur. Spring Boot (Java) umum memakai Thymeleaf, yang uniknya ditulis sebagai atribut HTML tambahan sehingga filenya tetap valid HTML murni meski belum di-render. Semua solusi ini lahir dari kebutuhan yang identik: memisahkan *logika tampilan* (bagaimana data ditampilkan) dari *logika bisnis* (bagaimana data diproses), sambil tetap memberi cara singkat untuk menyisipkan data dinamis ke HTML.

Blade sendiri punya karakteristik teknis yang layak diketahui: dia bukan engine yang mem-parsing ulang tiap request. File `.blade.php` dikompilasi menjadi kode PHP polos, disimpan sebagai file cache di `storage/framework/views/`, dan hanya dikompilasi ulang kalau file sumbernya berubah. Artinya di production, overhead template engine praktis nol — yang benar-benar dieksekusi setiap request adalah PHP biasa yang sudah dikompilasi sebelumnya. Ini pendekatan yang sama dipakai Jinja2 (Python) yang juga meng-compile template ke bytecode Python. Jadi kemudahan sintaks yang kita nikmati tidak datang dengan mengorbankan performa. (Bagian Materi nanti akan menunjukkan langsung contoh hasil kompilasi ini, supaya terasa nyata, bukan cuma klaim.)

Poin ini layak ditegaskan lebih jauh: keterbatasan Blade dibanding PHP murni bukan kekurangan, itu justru **fitur yang disengaja**. Secara teknis, Blade masih menyediakan directive `@php ... @endphp` untuk menulis PHP mentah kalau benar-benar diperlukan, tapi konvensi di seluruh komunitas Laravel adalah menghindarinya sebisa mungkin — kalau butuh logika rumit di View, itu tanda logika itu seharusnya dipindah ke Controller atau disiapkan lebih dulu sebagai data sederhana. Prinsip yang sama berlaku di Django: Jinja2/Django Template Language sengaja tidak mengizinkan pemanggilan method sembarangan atau akses bebas ke objek Python dari dalam template, justru supaya siapa pun yang menyentuh file template tidak bisa — sengaja atau tidak sengaja — mengubah logika bisnis aplikasi. Batasan ini mirip pagar pembatas di tepi jalan pegunungan: bukan untuk mempersulit perjalanan, tapi supaya kesalahan kecil tidak berujung fatal.

Manfaat ketiga — dan ini yang jadi fokus utama pertemuan ini — adalah *layout inheritance*. Alih-alih setiap halaman menulis struktur HTML lengkap dari nol, satu file "master" mendefinisikan kerangka bersama (head, navbar, area konten, footer), dan halaman lain tinggal "mewarisi" kerangka itu sambil mengisi bagian yang berbeda-beda. Django menyebut mekanisme ini `{% extends %}` dan `{% block %}`, konsepnya identik dengan `@extends` dan `@section` di Blade. Express.js dengan EJS/Pug biasanya tidak punya inheritance sekuat ini secara bawaan, dan lebih mengandalkan *include* murni (menempelkan potongan file ke file lain) untuk mencapai efek yang mirip. Blade menyediakan keduanya — inheritance untuk kerangka besar, dan `@include` untuk potongan kecil yang berulang seperti navbar — sehingga masalah duplikasi yang kita lihat di awal bagian ini terselesaikan dari dua sisi sekaligus.

---

## Materi

### Echo Aman vs Raw Output

Dua cara Blade menampilkan data punya efek keamanan yang sangat berbeda:

```blade
{{-- Contoh ilustrasi konsep — bukan langkah praktikum --}}
{{ $judul }}   {{-- auto-escape, aman dari XSS, ini yang HAMPIR SELALU dipakai --}}
{!! $html !!} {{-- raw output, TIDAK di-escape, hanya untuk HTML yang sudah kamu percaya --}}
```

`{{ }}` secara internal memanggil `htmlspecialchars()`, mengubah karakter seperti `<` dan `>` jadi entitas HTML supaya tidak dieksekusi sebagai tag. `{!! !!}` melewati proses itu sepenuhnya — berguna kalau kamu memang ingin menampilkan HTML mentah (misalnya hasil dari editor WYSIWYG yang sudah disanitasi terpisah), tapi berbahaya kalau dipakai sembarangan untuk data yang berasal dari input user.

Supaya efek bahayanya lebih terasa, bayangkan skenario nyata: seseorang mengisi form tambah buku dengan judul `<script>alert('Data kamu bocor!')</script>` alih-alih judul buku sungguhan. Kalau ditampilkan lewat `{{ $judul }}`, Blade otomatis mengubahnya jadi teks aman `&lt;script&gt;alert('Data kamu bocor!')&lt;/script&gt;` — browser menampilkannya sebagai **teks biasa**, persis seperti yang diketik, tidak dieksekusi apa-apa. Tapi kalau (sengaja atau tidak sengaja) ditampilkan lewat `{!! $judul !!}`, browser benar-benar **menjalankan** script itu — di contoh ini cuma memunculkan kotak `alert()`, tapi di dunia nyata skrip semacam ini bisa mencuri cookie session pengguna lain atau mengubah tampilan halaman untuk menipu (phishing). Karena itu aturan praktisnya sederhana: **selalu pakai `{{ }}` untuk data apa pun yang berasal dari input user**, dan hanya pakai `{!! !!}` untuk HTML yang benar-benar kamu percaya sumbernya — misalnya teks statis yang kamu tulis sendiri di kode, bukan data dari form atau database yang bisa diisi siapa saja.

### Blade Bukan Sihir: Hasil Kompilasi di Balik Layar

Supaya directive-directive Blade tidak terasa seperti sihir, penting untuk tahu apa yang sebenarnya terjadi saat Laravel merender file `.blade.php`. Setiap directive sebenarnya cuma singkatan yang dikompilasi menjadi PHP polos sebelum benar-benar dieksekusi. Potongan Blade ini:

```blade
{{-- Contoh ilustrasi konsep --}}
@if ($book['stok'] > 0)
    <span>Tersedia</span>
@endif

@foreach ($books as $book)
    <p>{{ $book['judul'] }}</p>
@endforeach
```

kira-kira dikompilasi Laravel menjadi kode PHP berikut (disederhanakan):

```php
<?php if ($book['stok'] > 0): ?>
    <span>Tersedia</span>
<?php endif; ?>

<?php foreach ($books as $book): ?>
    <p><?php echo e($book['judul']); ?></p>
<?php endforeach; ?>
```

`e()` di situ adalah helper Laravel yang sebenarnya cuma pembungkus tipis di atas `htmlspecialchars()` — persis mekanisme escaping yang baru saja dibahas. File hasil kompilasi ini benar-benar tersimpan sebagai file PHP biasa di `storage/framework/views/`, dan hanya dibuat ulang kalau file `.blade.php` sumbernya berubah — kamu bisa membukanya sendiri untuk membuktikan tidak ada yang disembunyikan. Jadi setiap kali menulis `@foreach`, yang sebenarnya terjadi hanyalah `foreach` PHP biasa dengan sedikit gula sintaksis di atasnya — bukan bahasa baru yang misterius, cuma cara penulisan yang lebih ringkas dan lebih sulit salah pakai.

### Directive Kondisi dan Perulangan

Blade menyediakan directive yang membungkus struktur kontrol PHP dengan sintaks lebih ringkas:

```blade
{{-- Contoh ilustrasi konsep --}}
@if ($book['stok'] > 0)
    <span>Tersedia</span>
@elseif ($book['stok'] == 0)
    <span>Habis</span>
@else
    <span>Data tidak valid</span>
@endif

@unless ($books->isEmpty())
    <p>Ada data buku.</p>
@endunless

@foreach ($books as $book)
    <p>{{ $book['judul'] }}</p>
@endforeach

@forelse ($books as $book)
    <p>{{ $book['judul'] }}</p>
@empty
    <p>Belum ada data buku.</p>
@endforelse
```

`@forelse` yang sudah dipakai sejak Pertemuan 3 sebenarnya gabungan dari `@foreach` dan pengecekan kosong sekaligus — kalau koleksinya kosong, blok `@empty` yang dirender, sehingga tidak perlu menulis `@if (count($books) === 0)` secara terpisah.

### Variabel Loop `$loop`

Di dalam `@foreach`, Blade otomatis menyediakan variabel `$loop` berisi informasi tentang posisi iterasi saat ini — berguna untuk memberi nomor urut, menandai baris pertama/terakhir, atau memberi warna selang-seling tanpa menghitung modulo manual:

```blade
{{-- Contoh ilustrasi konsep --}}
@foreach ($books as $book)
    <tr class="{{ $loop->even ? 'bg-gray' : '' }}">
        <td>{{ $loop->iteration }}</td> {{-- nomor urut mulai dari 1 --}}
        <td>{{ $book['judul'] }}</td>
        @if ($loop->last)
            <td>Baris terakhir</td>
        @endif
    </tr>
@endforeach
```

Properti `$loop` yang tersedia lebih lengkap dari sekadar `iteration` dan `last`:

| Properti | Tipe | Keterangan |
|---|---|---|
| `$loop->index` | int | Index iterasi, mulai dari **0** |
| `$loop->iteration` | int | Nomor iterasi, mulai dari **1** (cocok untuk nomor urut tabel) |
| `$loop->remaining` | int | Sisa iterasi setelah yang sekarang |
| `$loop->count` | int | Total jumlah item dalam loop |
| `$loop->first` | bool | `true` kalau ini iterasi pertama |
| `$loop->last` | bool | `true` kalau ini iterasi terakhir |
| `$loop->even` / `$loop->odd` | bool | `true` kalau posisi iterasi genap/ganjil (berguna untuk warna selang-seling baris tabel) |
| `$loop->depth` | int | Level kedalaman loop saat ini (1 untuk loop terluar) |
| `$loop->parent` | object | Akses `$loop` milik loop di luarnya — dipakai kalau ada `@foreach` bersarang (nested), misalnya me-loop `$loans` lalu di dalamnya me-loop `$loan->loanItems` |

### Layout Inheritance: `@extends`, `@section`, `@yield`

Inilah inti pertemuan ini. File layout mendefinisikan kerangka bersama dengan `@yield('nama')` sebagai "lubang" yang nanti diisi child view:

```blade
{{-- Contoh ilustrasi konsep --}}
{{-- layouts/app.blade.php --}}
<html>
<head><title>@yield('title', 'Judul Default')</title></head>
<body>
    @yield('content')
</body>
</html>
```

Child view "mewarisi" layout itu dengan `@extends`, lalu mengisi lubang `content` lewat `@section` ... `@endsection`:

```blade
{{-- Contoh ilustrasi konsep --}}
{{-- books/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Daftar Buku')

@section('content')
    <h1>Daftar Buku</h1>
@endsection
```

`@yield('title', 'Judul Default')` menerima argumen kedua sebagai nilai default — dipakai kalau child view tidak mendefinisikan `@section('title', ...)` sama sekali. Ini yang membuat setiap halaman bisa punya `<title>` tab browser yang berbeda tanpa perlu menulis ulang seluruh `<head>`.

### Urutan Eksekusi: Mana yang Dirender Duluan?

Kesalahpahaman paling umum soal layout inheritance: mengira Laravel merender `layouts/app.blade.php` dulu dari atas ke bawah, lalu "menyisipkan" child view begitu ketemu `@yield`. Urutan sebenarnya justru terbalik, dan ini penting dipahami supaya tidak bingung saat sedang debugging tampilan yang tidak sesuai harapan:

1. Controller memanggil `return view('books.index', compact('books'))`.
2. Laravel mulai membaca `books/index.blade.php`. Baris pertama `@extends('layouts.app')` memberi tahu Laravel: "file ini punya induk — proses dulu semua section di bawah, jangan langsung dirender."
3. Laravel lanjut membaca seluruh isi `books/index.blade.php`. Setiap kali ketemu `@section('nama') ... @endsection`, isinya **tidak langsung dicetak**, melainkan disimpan sementara di memori dengan label sesuai nama section-nya (`title`, `content`, dst) — semacam "dikantongi" dulu.
4. Setelah seluruh child view selesai dibaca dan semua section-nya sudah dikantongi, barulah Laravel membuka `layouts/app.blade.php` dan mulai merendernya dari atas ke bawah, sebagai proses yang benar-benar terpisah.
5. Setiap kali proses render layout ketemu `@yield('nama')`, Laravel mengambil isi dari kantong section yang labelnya cocok, lalu menyisipkannya persis di situ. Kalau tidak ada section dengan nama itu, dipakai nilai default (`@yield('title', 'Judul Default')`) atau kosong kalau tidak ada default.
6. Hasil gabungan layout + section itulah yang akhirnya dikirim sebagai satu response HTML utuh ke browser.

Analogi sederhananya: child view seperti mengisi formulir kosong di rumah dulu (kolom "title", kolom "content"), baru formulir itu dibawa ke kantor (layout) yang sudah punya kop surat dan cap resmi (head, navbar, footer) — kantor tinggal menempelkan isian kamu ke tempat yang memang sudah disediakan, bukan sebaliknya.

### Partial View dan Prinsip DRY: `@include`

Kalau `@extends` cocok untuk kerangka besar yang dipakai oleh (hampir) semua halaman, `@include` cocok untuk potongan kecil yang berulang di beberapa tempat tapi tidak harus selalu sama persis strukturnya — misalnya navbar, atau blok alert pesan sukses:

```blade
{{-- Contoh ilustrasi konsep --}}
@include('partials.navbar')
@include('partials.alert')
```

Bedanya dengan `@extends`: `@include` cuma "menempelkan" isi file lain persis di titik itu, tanpa konsep pewarisan lubang seperti `@yield`. Kamu bisa mengoper data tambahan ke file yang di-include lewat argumen kedua: `@include('partials.navbar', ['activeMenu' => 'books'])`.

### Kesalahan Umum Pemula

Empat kesalahan berikut paling sering bikin bingung karena gejalanya tidak selalu berupa error yang jelas:

- **Lupa directive penutup** (`@endforeach`, `@endif`, `@endsection`) — Laravel melempar error parsing yang terdengar asing buat pemula, semacam `syntax error, unexpected end of file`. Penyebabnya hampir selalu satu directive pembuka yang lupa ditutup di suatu tempat.
- **Nama section tidak cocok** antara child dan parent (typo, atau beda besar-kecil huruf) — ini yang paling menjebak karena **tidak memunculkan error sama sekali**. Halaman tetap tampil normal, cuma bagian yang seharusnya terisi jadi kosong begitu saja, karena Blade menganggap section dengan nama itu memang belum pernah didefinisikan, bukan "salah ketik".
- **Lupa menulis `@extends` di baris pertama**, atau menaruh `@section` tanpa `@extends` sama sekali — halaman akan tampil tanpa navbar/footer sama sekali, karena Blade menganggap file itu berdiri sendiri, bukan child dari layout manapun.
- **Memakai `{!! !!}` untuk data dari input user** — tidak memunculkan error apa pun saat development, tapi membuka celah XSS yang baru terasa dampaknya kalau ada yang benar-benar menyalahgunakannya (lihat bagian Echo Aman vs Raw Output di atas).

### Helper yang Sering Dipakai di Blade

```blade
{{-- Contoh ilustrasi konsep --}}
{{ route('books.index') }}          {{-- generate URL dari nama route, bukan hardcode string --}}
{{ url('/books') }}                 {{-- generate URL absolut dari path --}}
{{ asset('css/app.css') }}          {{-- generate URL ke file di folder public/ --}}
{{ auth()->user()->name ?? 'Tamu' }} {{-- akses user yang sedang login (dipakai penuh mulai Pertemuan 8) --}}
```

Memakai `route()` alih-alih menulis path secara manual (`href="/books"`) penting supaya link tidak rusak kalau suatu saat prefix URL diubah — karena yang dirujuk adalah *nama* route, bukan string path-nya.

### Active State Navbar dengan `request()->routeIs()`

`request()->routeIs('books.*')` mengembalikan `true` kalau nama route yang sedang aktif cocok dengan pola yang diberikan (tanda `*` adalah wildcard). Ini dipakai untuk menandai menu navbar mana yang sedang aktif, murni berdasarkan route yang sedang diakses — tidak perlu variabel manual dikirim dari tiap Controller untuk memberi tahu "menu mana yang aktif sekarang".

---

## Praktikum

> **Konteks:** Melanjutkan project `app-perpustakaan` dari Pertemuan 3.
> Pastikan branch aktif adalah `dev`.
> Di akhir praktikum ini kamu akan memiliki: `layouts/app.blade.php` sebagai master layout, `partials/navbar.blade.php` dan `partials/alert.blade.php` sebagai komponen reusable, serta `books/index.blade.php`, `categories/index.blade.php`, dan `members/index.blade.php` yang semuanya sudah memakai `@extends`/`@section`/`@include` alih-alih HTML mandiri.

### Langkah 1 — Membuat Master Layout

Master layout menampung seluruh struktur HTML yang sebelumnya diulang di setiap view: `<head>`, style bersama, navbar, area konten, dan footer.

```blade
{{-- File: resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>@yield('title', 'Perpustakaan Digital Kampus')</title>
    <style>
        /* style bersama untuk seluruh halaman: nav, table, alert, tombol */
    </style>
</head>
<body>
    @include('partials.navbar')

    <main>
        @include('partials.alert')

        @yield('content')
    </main>

    <footer>
        &copy; {{ date('Y') }} Sistem Perpustakaan Digital Kampus
    </footer>
</body>
</html>
```

`@yield('content')` adalah lubang utama yang nanti diisi oleh setiap halaman lewat `@section('content') ... @endsection`. `@include('partials.navbar')` dan `@include('partials.alert')` menempelkan dua komponen kecil yang akan dibuat di langkah berikutnya.

### Langkah 2 — Membuat Partial Navbar

Navbar dipisah ke file sendiri karena isinya (link ke tiap resource) dipakai persis sama di semua halaman, dan active state-nya bergantung pada route yang sedang aktif — bukan pada Controller mana yang memanggilnya.

```blade
{{-- File: resources/views/partials/navbar.blade.php --}}
<nav>
    <div class="brand">📚 Perpustakaan Digital Kampus</div>
    <ul>
        <li><a href="{{ route('books.index') }}" class="{{ request()->routeIs('books.*') ? 'active' : '' }}">Buku</a></li>
        <li><a href="{{ route('categories.index') }}" class="{{ request()->routeIs('categories.*') ? 'active' : '' }}">Kategori</a></li>
        <li><a href="{{ route('members.index') }}" class="{{ request()->routeIs('members.*') ? 'active' : '' }}">Anggota</a></li>
        <li><a href="{{ route('loans.index') }}" class="{{ request()->routeIs('loans.*') ? 'active' : '' }}">Peminjaman</a></li>
    </ul>
</nav>
```

`request()->routeIs('books.*')` mencocokkan nama route saat ini terhadap pola `books.*` (cocok untuk `books.index`, `books.create`, `books.show`, dst). Kalau cocok, class `active` ditambahkan supaya menu itu tersorot secara visual — tanpa perlu Controller mengirim variabel tambahan untuk memberi tahu menu mana yang aktif.

### Langkah 3 — Membuat Partial Alert

Flash message sukses sebelumnya ditulis manual dengan `@if (session('success'))` di tiap file index. Sekarang dipisah jadi komponen tersendiri supaya cukup ditulis sekali di layout lewat `@include('partials.alert')`.

```blade
{{-- File: resources/views/partials/alert.blade.php --}}
@if (session('success'))
    <div class="alert-success">{{ session('success') }}</div>
@endif
```

### Langkah 4 — Merefactor `books/index.blade.php`

Isi `<!DOCTYPE html>`, `<head>`, dan blok alert manual dihapus dari file ini — semuanya sudah diambil alih oleh layout. File ini sekarang hanya berisi konten unik halaman ini.

```blade
{{-- File: resources/views/books/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Daftar Buku')

@section('content')
    <h1>Daftar Buku</h1>

    <p><a href="{{ route('books.create') }}" class="btn">+ Tambah Buku</a></p>

    <table>
        {{-- header tabel dan @forelse ($books as $book) persis seperti Pertemuan 3 --}}
    </table>
@endsection
```

> 📸 *Screenshot: halaman `/books` tampil dengan navbar biru di atas, menu "Buku" tersorot aktif, dan tabel data dummy di bawahnya.*

### Langkah 5 — Merefactor `categories/index.blade.php`

Pola yang sama persis diterapkan ke `categories/index.blade.php`: bungkus konten dengan `@extends('layouts.app')` dan `@section('content')`, hapus HTML boilerplate yang sudah dipindah ke layout.

```blade
{{-- File: resources/views/categories/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Daftar Kategori')

@section('content')
    <h1>Daftar Kategori</h1>
    {{-- tabel @forelse ($categories as $category) persis seperti Pertemuan 3 --}}
@endsection
```

### Langkah 6 — Mengisi `MemberController@index` dan Membuat `members/index.blade.php`

Supaya ketiga resource (buku, kategori, anggota) sama-sama bisa didemonstrasikan memakai layout baru, `MemberController@index` diisi data dummy anggota mengikuti pola persis `BookController`.

```php
// File: app/Http/Controllers/MemberController.php
private array $members = [
    ['id' => 1, 'nama' => 'Siti Aminah', 'nim' => '2310501001', 'email' => 'siti.aminah@pens.ac.id', 'nomor_telepon' => '081234567890', 'status' => 'aktif'],
    ['id' => 2, 'nama' => 'Budi Santoso', 'nim' => '2310501002', 'email' => 'budi.santoso@pens.ac.id', 'nomor_telepon' => '081298765432', 'status' => 'aktif'],
    ['id' => 3, 'nama' => 'Dewi Lestari', 'nim' => '2310501003', 'email' => 'dewi.lestari@pens.ac.id', 'nomor_telepon' => '081211122233', 'status' => 'nonaktif'],
];

public function index()
{
    $members = $this->members;

    return view('members.index', compact('members'));
}
```

Kemudian buat view-nya, langsung memakai master layout sejak awal (tidak seperti `books`/`categories` yang tadinya HTML mandiri lalu di-refactor):

```blade
{{-- File: resources/views/members/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Daftar Anggota')

@section('content')
    <h1>Daftar Anggota</h1>

    <table>
        @forelse ($members as $member)
            <tr>
                <td>{{ $member['nama'] }}</td>
                <td>{{ $member['nim'] }}</td>
                <td>{{ ucfirst($member['status']) }}</td>
            </tr>
        @empty
            <tr><td colspan="3">Belum ada data anggota.</td></tr>
        @endforelse
    </table>
@endsection
```

Method `create()`, `store()`, `edit()`, `update()`, dan `destroy()` di `MemberController` sengaja **belum diisi** — CRUD anggota yang lengkap (termasuk form tambah/edit) baru dikerjakan mulai Pertemuan 5, setelah Migration & Model Eloquent tersedia.

> 📸 *Screenshot: halaman `/members` menampilkan tabel anggota dengan navbar dan footer yang sama seperti `/books` dan `/categories`.*

### Langkah 7 — Ujicoba

Jalankan server dan bandingkan tampilan ketiga halaman:

```bash
php artisan serve
```

1. Buka `http://127.0.0.1:8000/books`, `/categories`, dan `/members` satu per satu — pastikan navbar dan footer identik di ketiganya, hanya konten tabel yang berbeda.
2. Perhatikan menu navbar yang tersorot (class `active`) selalu sesuai dengan halaman yang sedang dibuka.
3. Tambahkan satu buku baru lewat `/books/create` — pastikan setelah redirect ke `/books`, flash message sukses tetap tampil dengan benar (tandanya `partials/alert.blade.php` ter-include dengan baik di layout).
4. Buka `view-source` (lihat kode sumber halaman) di browser dan pastikan tidak ada duplikasi `<html>` atau `<head>` — hanya satu, berasal dari `layouts/app.blade.php`.

---

## Checkpoint GitHub

```bash
git add .
git commit -m "[P-4] tambah master layout blade dan view index semua resource"
git push origin dev
```

---

## Tugas

Refactor `books/create.blade.php` dan `categories/create.blade.php` supaya sama-sama memakai master layout, mengikuti pola yang sama seperti refactor `index.blade.php` di praktikum ini:

1. Ganti `<!DOCTYPE html>` mandiri di kedua file dengan `@extends('layouts.app')`, `@section('title', ...)`, dan `@section('content') ... @endsection`.
2. Pastikan tampilan error validasi (`@error`) dan `old()` tetap berfungsi persis seperti sebelumnya — hanya bungkusnya (HTML di luar form) yang berubah.
3. Tambahkan link "← Kembali ke daftar" di bagian atas `@section('content')` pada kedua file, mengarah ke `route('books.index')` / `route('categories.index')`.
4. Uji lagi submit kosong (pastikan error tetap tampil) dan submit valid (pastikan redirect dan flash message tetap tampil lewat `partials/alert.blade.php`).

**Yang dikumpulkan:**
- Link commit GitHub (branch `dev`) yang berisi hasil tugas
- Screenshot `/books/create` dan `/categories/create` setelah di-refactor, menampilkan navbar yang sama seperti halaman index

---

*Navigasi: [← Pertemuan sebelumnya](./pertemuan-03.md) | [Daftar Isi](./README.md) | [Pertemuan berikutnya →](./pertemuan-05.md)*
