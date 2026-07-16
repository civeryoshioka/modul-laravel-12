# Git & GitHub — Dasar yang Perlu Diketahui

> Materi tambahan, di luar urutan Pertemuan 1–11. Modul ini **mengasumsikan mahasiswa sudah bisa pakai Git** (branch `dev`/`main`, commit per checkpoint, push ke GitHub — lihat konvensi di setiap `pertemuan-XX.md`). Kalau belum pernah pakai Git sama sekali, baca ini dulu sebelum mulai Pertemuan 1.

---

## Mengapa Version Control Ada?

Bayangkan mengerjakan `app-perpustakaan` tanpa Git. Setiap kali mau mencoba sesuatu yang berisiko, satu-satunya cara aman adalah copy-paste seluruh folder project jadi `app-perpustakaan-backup`, `app-perpustakaan-backup2`, `app-perpustakaan-final`, `app-perpustakaan-final-fix`, dan seterusnya. Kalau kerja tim, masalah lebih parah lagi: dua orang mengedit file yang sama, lalu bingung siapa yang menimpa pekerjaan siapa lewat email atau flashdisk.

**Version control** (dan Git adalah implementasinya yang paling populer) menyelesaikan masalah ini dengan mencatat *riwayat perubahan* kode secara terstruktur — setiap perubahan direkam sebagai satu titik (disebut **commit**) yang bisa dilihat lagi, dibandingkan, atau bahkan dibatalkan kapan saja, tanpa perlu folder duplikat manual.

Konsep ini berlaku universal di luar Laravel — tim developer yang pakai Node.js, Python, atau Java pun tetap pakai Git untuk masalah yang sama persis: melacak riwayat perubahan kode dan mengoordinasikan kerja banyak orang di kode yang sama.

---

## Git vs GitHub — Dua Hal yang Berbeda

Ini sering tertukar oleh pemula:

| | Git | GitHub |
|---|---|---|
| Apa itu | Software version control, jalan di komputer sendiri | Platform hosting online untuk menyimpan repository Git |
| Instalasi | Diinstal di laptop (`git --version` untuk cek) | Tidak perlu instalasi, cukup akun web |
| Fungsi utama | Mencatat riwayat perubahan (commit), branch, merge | Menyimpan salinan repository di server, kolaborasi, review kode (Pull Request) |
| Analogi | Mesin ketik yang menyimpan setiap draft tulisanmu | Google Drive tempat draft-draft itu disimpan dan dibagikan |

Bisa pakai Git **tanpa** GitHub (semua riwayat cuma tersimpan di laptop sendiri). Tapi tidak bisa pakai GitHub tanpa Git — GitHub cuma tempat menyimpan hasil kerja Git. Platform sejenis lain: GitLab, Bitbucket — konsep Git-nya identik, bedanya cuma platform hosting-nya.

---

## Tiga Area Kerja Git

Sebelum masuk perintah, pahami dulu 3 area tempat file "berada" dari sudut pandang Git:

```
Working Directory  --git add-->  Staging Area  --git commit-->  Local Repository  --git push-->  Remote (GitHub)
   (file yang kamu edit)          (file yang siap di-commit)     (riwayat commit di laptop)         (riwayat commit online)
```

- **Working Directory** — folder project seperti biasa, file yang kamu edit di editor.
- **Staging Area** — "ruang tunggu" berisi file yang sudah ditandai siap masuk commit berikutnya (lewat `git add`). Berguna kalau kamu mengedit 5 file tapi cuma ingin 3 di antaranya masuk 1 commit.
- **Local Repository** — riwayat commit yang tersimpan di `.git/` folder, di laptop sendiri.
- **Remote** — salinan repository yang tersimpan online (GitHub), supaya bisa diakses dari laptop lain atau dibagikan ke orang lain.

---

## Perintah yang Paling Sering Dipakai

### Setup Awal (sekali saja per laptop)

```bash
git config --global user.name "Nama Kamu"
git config --global user.email "email@kamu.com"
```

Ini menentukan nama yang tercatat sebagai penulis commit — bukan login akun GitHub.

### Memulai Repository

```bash
git init                        # membuat repository Git baru di folder saat ini
git clone <url>                 # menyalin repository yang sudah ada di GitHub ke laptop
```

### Alur Kerja Sehari-hari

```bash
git status                      # lihat file apa saja yang berubah/belum ter-commit
git add nama_file.php           # tandai 1 file spesifik untuk di-commit
git add .                       # tandai SEMUA file yang berubah untuk di-commit
git commit -m "[P-1] pesan singkat"   # simpan perubahan yang sudah ditandai sebagai 1 commit
git log --oneline               # lihat riwayat commit secara ringkas
```

> **Kenapa harus `git add` dulu, tidak langsung `git commit`?** Supaya kamu bisa memilih mana yang masuk commit ini dan mana yang belum — misalnya kamu sedang mengerjakan 2 fitur sekaligus, tapi cuma 1 yang sudah selesai dan siap di-commit.

### Bekerja dengan Remote (GitHub)

```bash
git remote add origin <url>     # menghubungkan repository lokal ke repository GitHub (sekali saja)
git push origin dev             # kirim commit dari branch dev di laptop ke GitHub
git pull origin dev             # ambil commit terbaru dari GitHub ke laptop (kebalikan dari push)
```

### Branch — Bekerja di "Cabang" Terpisah

**Branch** adalah cara mengembangkan sesuatu secara terpisah dari kode utama, tanpa mengganggu versi yang sudah stabil. Project `app-perpustakaan` di modul ini memakai 2 branch:

```
main  ← kode stabil, hanya menerima kode yang sudah teruji lewat merge
dev   ← branch aktif tempat semua pekerjaan baru dilakukan
```

```bash
git branch                      # lihat daftar branch, tanda * menunjukkan branch aktif
git checkout dev                # pindah ke branch dev
git checkout -b fitur-baru       # buat branch baru sekaligus langsung pindah ke sana
git merge dev                   # gabungkan branch dev ke branch yang sedang aktif (misal: sedang di main)
```

Alur yang dipakai di modul ini: seluruh checkpoint `[P-1]` s.d. `[P-10]` dikerjakan dan di-commit di branch `dev`. Baru di akhir (Tugas P10), `dev` di-merge ke `main` supaya `main` berisi versi final yang stabil untuk didemokan di UAS.

### Melihat Perubahan

```bash
git diff                        # lihat detail baris yang berubah, belum di-add
git diff --staged               # lihat detail baris yang sudah di-add, belum di-commit
```

---

## `.gitignore` — File yang Sengaja Tidak Ikut Di-commit

Tidak semua file di project layak masuk Git. Contoh di project Laravel: folder `vendor/` (hasil `composer install`, bisa dibuat ulang), `node_modules/` (hasil `npm install`), dan `.env` (berisi password database — **jangan pernah commit file ini**, karena kalau repository publik, kredensial ikut ter-expose ke siapa saja).

File `.gitignore` mendaftar pola nama file/folder yang harus diabaikan Git:

```
/vendor
/node_modules
.env
```

Laravel sudah menyediakan `.gitignore` default yang mencakup ini semua — biasanya tidak perlu diedit manual kecuali menambah file spesifik project.

---

## Konvensi Commit Message yang Baik

Commit message yang jelas membantu diri sendiri (dan orang lain) memahami riwayat perubahan tanpa harus membuka isi kodenya satu per satu.

```bash
# Kurang jelas
git commit -m "update"
git commit -m "fix bug"
git commit -m "asdf"

# Jelas — konvensi yang dipakai di modul ini
git commit -m "[P-5] migrasi semua tabel dan crud books categories selesai"
git commit -m "[P-8] implementasi autentikasi login logout middleware"
```

Prinsipnya: pesan commit menjelaskan **apa** yang berubah, ditulis dalam bentuk kalimat yang masuk akal kalau dibaca berurutan di `git log`.

---

## Kesalahan Umum Pemula

- **Lupa `git add` sebelum commit** — muncul pesan "nothing to commit", padahal file sudah diedit. Cek dulu dengan `git status`.
- **Commit terlalu besar** — mengerjakan 5 fitur sekaligus baru commit sekali dengan pesan "update banyak hal". Riwayat jadi tidak berguna karena tidak bisa dilacak per fitur. Biasakan commit kecil, sering, per bagian yang selesai (seperti pola checkpoint `[P-X]` di modul ini).
- **Commit file `.env` atau `vendor/`** — cek `.gitignore` sebelum `git add .` pertama kali di project baru.
- **Push ke branch yang salah** — pastikan branch aktif benar (`git branch` untuk cek tanda `*`) sebelum `git push`.
- **Panik saat merge conflict** — ini normal terjadi kalau 2 orang (atau 2 branch) mengedit baris yang sama. Git akan menandai bagian yang bentrok di file, tinggal edit manual pilih versi mana yang benar, lalu `git add` dan `git commit` lagi.

---

## Ringkasan Alur Kerja Project Ini

```bash
# Sekali di awal project
git init
git remote add origin <url-repository-github>
git checkout -b dev

# Berulang setiap selesai 1 checkpoint pertemuan
git add .
git commit -m "[P-X] deskripsi singkat perubahan"
git push origin dev

# Di akhir semester (Tugas P10), setelah dev stabil
git checkout main
git merge dev
git push origin main
```

---

*Referensi lain: [Pertemuan 1 — Framework, MVC & Setup Proyek](../pertemuan-01.md) | [Daftar Isi Modul](../README.md)*
