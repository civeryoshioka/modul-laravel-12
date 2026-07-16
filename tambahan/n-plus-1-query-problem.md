# N+1 Query Problem

> Halaman ini adalah pendalaman untuk topik N+1 Query Problem yang diperkenalkan di **[Pertemuan 7 — Eloquent Relationships](../pertemuan-07.md)**. Baca ini kalau penjelasan singkat di sana masih kurang mengena, atau kalau ingin analogi, studi kasus tambahan, dan cara mendeteksi N+1 di project nyata.

---

## Mengapa Ini Ada?

Setiap kali aplikasi butuh data dari database, aplikasi harus "bertanya" ke database — ini disebut **query**. Semakin sering aplikasi bertanya (semakin banyak query), semakin lama waktu yang dibutuhkan untuk memuat halaman, karena setiap query butuh waktu tempuh (koneksi) ke database, bukan cuma waktu memproses datanya.

Masalahnya, cara penulisan kode yang terlihat "biasa saja" — misalnya menampilkan data employee beserta nama departemennya di dalam sebuah `foreach` — bisa diam-diam membuat aplikasi bertanya ke database **berkali-kali lipat** tanpa disadari. Inilah yang disebut **N+1 Query Problem**, salah satu penyebab paling umum aplikasi Laravel terasa lambat saat datanya sudah banyak, meskipun terasa cepat-cepat saja saat masih development dengan data sedikit.

Konsep ini penting dipahami karena bukan cuma soal Eloquent — masalah yang sama juga muncul di ORM framework lain seperti Sequelize (Node.js) atau Django ORM (Python).

---

## Analogi Sederhana

Bayangkan kamu belanja ke minimarket untuk membeli 10 barang.

**Cara boros (N+1):**
Kamu pergi ke minimarket, beli 1 barang, pulang. Ingat butuh barang lain, pergi lagi, pulang. Begitu terus sampai 10 kali bolak-balik.

**Cara efisien (Eager Loading):**
Kamu buat daftar belanja dulu, lalu sekali jalan langsung borong semua 10 barang.

Setiap "pergi ke minimarket" itu setara dengan **1 query** ke database. Yang bikin lambat bukan barangnya, tapi jumlah perjalanannya.

---

## Studi Kasus: Employee dan Department

Anggap tabel `employees` menyimpan `department_id` (angka), bukan nama departemennya langsung:

```
employees:
id | nama  | department_id
1  | Budi  | 1
2  | Sari  | 2
3  | Andi  | 1

departments:
id | nama
1  | IT
2  | HR
```

Untuk menampilkan nama departemen (bukan cuma angka ID), aplikasi harus mengambil data dari tabel `departments`.

### Kode yang Bermasalah (N+1)

```php
// Controller
public function index()
{
    $employees = Employee::all(); // Query #1: ambil semua employee
    return view('employees.index', compact('employees'));
}
```

```blade
{{-- View --}}
@foreach($employees as $employee)
    <tr>
        <td>{{ $employee->nama_lengkap }}</td>
        <td>{{ $employee->department->nama_departemen }}</td>
        {{-- Setiap baris ini memicu query baru ke tabel departments! --}}
    </tr>
@endforeach
```

### Apa yang Sebenarnya Terjadi di Database

```
Query #1: SELECT * FROM employees;
Query #2: SELECT * FROM departments WHERE id = 1;   -- untuk Budi
Query #3: SELECT * FROM departments WHERE id = 2;   -- untuk Sari
Query #4: SELECT * FROM departments WHERE id = 1;   -- untuk Andi (duplikat!)
```

Total: **4 query** untuk 3 baris data. Rumusnya:

```
Total Query = 1 (query utama) + N (query tambahan, sebanyak jumlah baris)
```

Kalau employee-nya ada 1.000 baris, query yang ditembak bisa **1.001 kali** hanya untuk satu halaman.

| Jumlah Employee | Jumlah Query |
| --- | --- |
| 10 | 11 |
| 100 | 101 |
| 1.000 | 1.001 |
| 10.000 | 10.001 |

---

## Solusi: Eager Loading dengan `with()`

```php
public function index()
{
    $employees = Employee::with('department')->get();
    return view('employees.index', compact('employees'));
}
```

Yang terjadi di database sekarang:

```
Query #1: SELECT * FROM employees;
Query #2: SELECT * FROM departments WHERE id IN (1, 2);
```

Eloquent mengumpulkan semua `department_id` yang unik dari hasil query pertama, lalu mengambil semuanya sekaligus dalam **satu query** menggunakan `WHERE id IN (...)`.

Total: **selalu 2 query**, tidak peduli jumlah employee-nya 3 atau 10.000.

### Variasi Eager Loading

```php
// Satu relasi
Employee::with('department')->get();

// Beberapa relasi sekaligus
Employee::with(['department', 'position'])->get();

// Relasi bersarang (nested)
Employee::with('department.kepala')->get();

// Eager loading dengan kondisi
Employee::with(['attendance' => function ($query) {
    $query->where('tanggal', today());
}])->get();

// Lazy eager loading (kalau data sudah terlanjur di-load tanpa with())
$employees = Employee::all();
$employees->load('department');
```

---

## Cara Mendeteksi N+1 di Proyek

1. **Laravel Debugbar** — package populer untuk melihat jumlah query yang dijalankan per halaman.
   ```bash
   composer require barryvdh/laravel-debugbar --dev
   ```
2. **DB::listen()** — untuk logging query secara manual saat debugging.
   ```php
   \DB::listen(function ($query) {
       \Log::info($query->sql, $query->bindings);
   });
   ```
3. **Laravel Telescope** — tool resmi Laravel untuk monitoring query, job, dan request.

---

## Contoh Perbandingan Lengkap

```php
// BURUK — N+1 Problem
public function index()
{
    $employees = Employee::paginate(10);
    // Di view: $employee->department->nama_departemen
    //          $employee->position->nama_jabatan
    // Total: 1 + 10 + 10 = 21 query per halaman
    return view('employees.index', compact('employees'));
}

// BAIK — Eager Loading
public function index()
{
    $employees = Employee::with(['department', 'position'])
                          ->latest()
                          ->paginate(10);
    // Total: hanya 3 query, berapapun jumlah barisnya
    return view('employees.index', compact('employees'));
}
```

---

## Ringkasan

| | Tanpa `with()` (N+1) | Dengan `with()` (Eager Loading) |
| --- | --- | --- |
| Analogi | Bolak-balik ke minimarket tiap butuh 1 barang | Bikin daftar belanja, sekali jalan borong semua |
| Jumlah query | 1 + N (naik terus sesuai jumlah data) | Tetap konstan (biasanya 2) |
| Dampak saat data besar | Makin lambat, bisa membebani server | Tetap stabil |

**Poin kunci:** N+1 Problem muncul ketika kode melakukan loop terhadap hasil query, dan di dalam loop tersebut setiap item memicu query baru ke tabel relasi. Solusinya adalah memberi tahu Eloquent di awal relasi apa saja yang dibutuhkan, menggunakan method `with()`, sehingga Eloquent bisa mengambil semua data relasi dalam satu query tambahan saja.

---

*Referensi lain: [Pertemuan 7 — Eloquent Relationships](../pertemuan-07.md) | [Daftar Isi Modul](../README.md)*
