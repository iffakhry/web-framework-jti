# Modul Praktikum — Laporan PDF dengan DomPDF

## Workshop Sistem Informasi Web Framework · Laravel 12

---

## Daftar Isi

1. [Capaian Pembelajaran](#1-capaian-pembelajaran)
2. [Dasar Teori](#2-dasar-teori)
3. [Alat dan Bahan](#3-alat-dan-bahan)
4. [Persiapan Project](#4-persiapan-project)
5. [Langkah Kerja](#5-langkah-kerja)
   - [5.1 Instalasi DomPDF](#51-instalasi-dompdf)
   - [5.2 Membuat Database & Model](#52-membuat-database--model)
   - [5.3 Membuat Controller](#53-membuat-controller)
   - [5.4 Membuat View Blade](#54-membuat-view-blade)
   - [5.5 Mendaftarkan Route](#55-mendaftarkan-route)
   - [5.6 Menjalankan & Pengujian](#56-menjalankan--pengujian)
6. [Konfigurasi Lanjutan DomPDF](#6-konfigurasi-lanjutan-dompdf)
7. [Fitur Tambahan](#7-fitur-tambahan)
   - [7.1 Header & Footer Tiap Halaman](#71-header--footer-tiap-halaman)
   - [7.2 Orientasi Landscape](#72-orientasi-landscape)
   - [7.3 Filter Data via Query String](#73-filter-data-via-query-string)
8. [Kendala Umum & Solusi](#8-kendala-umum--solusi)
9. [Perbandingan Library PDF di Laravel](#9-perbandingan-library-pdf-di-laravel)
10. [Kesimpulan](#10-kesimpulan)
11. [Tugas & Rubrik Penilaian](#11-tugas--rubrik-penilaian)

---

## 1. Capaian Pembelajaran

Setelah menyelesaikan praktikum ini, mahasiswa mampu:

1. Menginstal dan mengintegrasikan package **barryvdh/laravel-dompdf** ke dalam proyek Laravel 12 menggunakan Composer.
2. Memahami perbedaan cara penggunaan DomPDF pada **Laravel versi lama** vs **Laravel 12** (auto-discovery, namespace facade).
3. Membuat laporan PDF berbasis HTML dan CSS menggunakan template Blade.
4. Mengambil data dari database melalui **Eloquent ORM** dan menyusunnya ke dalam laporan PDF.
5. Mengatur layout laporan: ukuran kertas, orientasi, header, footer, dan nomor halaman.
6. Melindungi endpoint generate PDF menggunakan **middleware autentikasi**.

---

## 2. Dasar Teori

### 2.1 Apa itu DomPDF?

**DomPDF** adalah library PHP yang mengkonversi dokumen HTML dan CSS menjadi file PDF. Cara kerjanya adalah dengan mem-_parsing_ HTML, menerapkan styling CSS, lalu me-_render_ hasilnya ke dalam format PDF menggunakan engine rendering berbasis PHP murni — tanpa dependensi binary eksternal seperti wkhtmltopdf atau Chrome.

Di ekosistem Laravel, DomPDF digunakan melalui wrapper resmi bernama **`barryvdh/laravel-dompdf`**. Package ini mengintegrasikan DomPDF dengan sistem View (Blade), Facade, dan konfigurasi Laravel secara mulus.

**Kegunaan umum di industri:**

- Invoice dan faktur pembelian
- Laporan data pegawai / keuangan
- Sertifikat dan dokumen resmi
- Ekspor data dalam format siap cetak

### 2.2 Perbedaan DomPDF di Laravel Versi Lama vs Laravel 12

Ini adalah poin paling penting yang sering menyebabkan kebingungan. Terdapat dua perubahan utama:

**a. Cara import Facade**

| Versi               | Kode                              | Keterangan                                                 |
| ------------------- | --------------------------------- | ---------------------------------------------------------- |
| Laravel ≤ 8 (lama)  | `use PDF;`                        | Menggunakan alias facade yang didaftarkan manual           |
| Laravel 9–12 (baru) | `use Barryvdh\DomPDF\Facade\Pdf;` | Menggunakan namespace lengkap, otomatis via auto-discovery |

```php
// ❌ CARA LAMA — tidak direkomendasikan di Laravel 12
use PDF;

$pdf = PDF::loadView('laporan', $data);

// ✅ CARA BENAR untuk Laravel 12
use Barryvdh\DomPDF\Facade\Pdf;

$pdf = Pdf::loadView('laporan', $data);
```

**b. Registrasi Service Provider**

Di Laravel versi lama, kita harus menambahkan baris berikut secara manual ke `config/app.php`:

```php
// ❌ Tidak perlu dilakukan di Laravel 12
'providers' => [
    Barryvdh\DomPDF\ServiceProvider::class,
],
'aliases' => [
    'PDF' => Barryvdh\DomPDF\Facade::class,
],
```

Laravel 12 menggunakan fitur **Package Auto-Discovery** — package yang sudah mendukung fitur ini (termasuk `barryvdh/laravel-dompdf`) akan otomatis terdaftar tanpa konfigurasi manual apapun.

### 2.3 Alur Kerja Generate PDF

```
Request → Route → Middleware (auth) → Controller → Eloquent (ambil data)
       → Blade View (render HTML) → DomPDF (konversi ke PDF)
       → Response (stream/download)
```

### 2.4 Keterbatasan CSS pada DomPDF

DomPDF tidak mendukung semua spesifikasi CSS modern. Berikut panduan cepat:

| Fitur CSS                               | Dukungan DomPDF                    |
| --------------------------------------- | ---------------------------------- |
| Box model (margin, padding, border)     | ✅ Didukung penuh                  |
| Font dasar, text-align, color           | ✅ Didukung penuh                  |
| Tabel (table, tr, td, th)               | ✅ Didukung penuh                  |
| Float                                   | ✅ Didukung (untuk layout 2 kolom) |
| `position: fixed` (untuk header/footer) | ✅ Didukung                        |
| Flexbox (`display: flex`)               | ❌ Tidak didukung                  |
| CSS Grid (`display: grid`)              | ❌ Tidak didukung                  |
| CSS Variables (`var(--warna)`)          | ❌ Tidak didukung                  |
| `box-shadow`, `text-shadow`             | ❌ Tidak didukung                  |
| `border-radius`                         | ⚠️ Terbatas                        |

**Kesimpulan:** Gunakan pendekatan layout berbasis `float` atau `table` untuk dokumen PDF. Hindari Flexbox dan Grid.

---

## 3. Alat dan Bahan

| No  | Perangkat / Software    | Versi            | Keterangan                              |
| --- | ----------------------- | ---------------- | --------------------------------------- |
| 1   | PC / Laptop             | —                | RAM minimal 4 GB, disarankan 8 GB       |
| 2   | PHP                     | 8.2 atau 8.3     | Syarat Laravel 12                       |
| 3   | Composer                | 2.x              | Manajemen dependency PHP                |
| 4   | Laravel                 | 12.x             | Framework yang digunakan                |
| 5   | barryvdh/laravel-dompdf | 3.x              | Library PDF                             |
| 6   | SQLite atau MySQL       | —                | Database (SQLite lebih mudah untuk dev) |
| 7   | Visual Studio Code      | Terbaru          | Text editor yang disarankan             |
| 8   | Browser                 | Chrome / Firefox | Untuk pengujian                         |
| 9   | Terminal / CMD          | —                | Menjalankan perintah Artisan            |

> **Catatan:** Pastikan ekstensi PHP berikut aktif: `php-gd`, `php-mbstring`, `php-xml`, `php-zip`. Ekstensi ini dibutuhkan oleh DomPDF.

---

## 4. Persiapan Project

Jika belum memiliki project Laravel 12, buat project baru dengan perintah berikut:

```bash
composer create-project laravel/laravel latihan-pdf
cd latihan-pdf
```

Konfigurasi database di file `.env`. Untuk kemudahan, gunakan SQLite:

```dotenv
DB_CONNECTION=sqlite
# Hapus atau komentari baris DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD
```

Buat file database SQLite:

```bash
touch database/database.sqlite
```

Verifikasi Laravel berjalan:

```bash
php artisan serve
# Buka: http://127.0.0.1:8000
```

---

## 5. Langkah Kerja

### 5.1 Instalasi DomPDF

Install package melalui Composer:

```bash
composer require barryvdh/laravel-dompdf
```

Tunggu hingga proses selesai. Karena Laravel 12 mendukung **auto-discovery**, package ini langsung siap digunakan — tidak perlu mengedit `config/app.php`.

**(Opsional)** Publish file konfigurasi DomPDF jika ingin menyesuaikan pengaturan:

```bash
php artisan vendor:publish --provider="Barryvdh\DomPDF\ServiceProvider"
```

Perintah ini akan membuat file `config/dompdf.php` di project Anda.

**Verifikasi instalasi** — pastikan package muncul di `composer.json`:

```bash
composer show barryvdh/laravel-dompdf
```

---

### 5.2 Membuat Database & Model

#### a. Buat Migration

```bash
php artisan make:migration create_pegawais_table
```

Buka file migration yang baru dibuat di folder `database/migrations/` dan isi dengan:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('pegawais', function (Blueprint $table) {
            $table->id();
            $table->string('nama');
            $table->string('nip')->unique();
            $table->string('jabatan');
            $table->string('divisi');
            $table->decimal('gaji_pokok', 15, 2);
            $table->date('tanggal_masuk');
            $table->enum('status', ['aktif', 'cuti', 'nonaktif'])->default('aktif');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('pegawais');
    }
};
```

#### b. Buat Model Eloquent

```bash
php artisan make:model Pegawai
```

Buka `app/Models/Pegawai.php` dan isi dengan:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Pegawai extends Model
{
    use HasFactory;

    protected $fillable = [
        'nama', 'nip', 'jabatan', 'divisi',
        'gaji_pokok', 'tanggal_masuk', 'status',
    ];

    protected $casts = [
        'tanggal_masuk' => 'date',
        'gaji_pokok'    => 'decimal:2',
    ];

    // Scope: filter hanya pegawai aktif
    public function scopeAktif($query)
    {
        return $query->where('status', 'aktif');
    }

    // Accessor: format gaji menjadi Rupiah
    public function getGajiFormatAttribute(): string
    {
        return 'Rp ' . number_format($this->gaji_pokok, 0, ',', '.');
    }

    // Accessor: hitung lama bekerja dalam tahun
    public function getLamaKerjaAttribute(): string
    {
        return $this->tanggal_masuk->diffInYears(now()) . ' tahun';
    }
}
```

> **Penjelasan Accessor:** `getGajiFormatAttribute()` membuat properti baru `$pegawai->gaji_format` yang mengembalikan nilai terformat. Ini adalah cara Eloquent untuk menyiapkan data agar siap ditampilkan tanpa logika di Blade.

#### c. Buat Factory untuk Data Dummy

```bash
php artisan make:factory PegawaiFactory --model=Pegawai
```

Buka `database/factories/PegawaiFactory.php` dan isi:

```php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;

class PegawaiFactory extends Factory
{
    public function definition(): array
    {
        $jabatanPerDivisi = [
            'IT'          => ['Software Engineer', 'System Analyst', 'DevOps Engineer'],
            'Keuangan'    => ['Akuntan', 'Auditor Internal', 'Staf Keuangan'],
            'SDM'         => ['Rekrutmen Specialist', 'Training Coordinator'],
            'Operasional' => ['Supervisor', 'Staf Operasional'],
        ];

        $divisi  = $this->faker->randomElement(array_keys($jabatanPerDivisi));
        $jabatan = $this->faker->randomElement($jabatanPerDivisi[$divisi]);

        static $nip = 1;

        return [
            'nama'          => $this->faker->name(),
            'nip'           => 'NIP' . str_pad($nip++, 5, '0', STR_PAD_LEFT),
            'jabatan'       => $jabatan,
            'divisi'        => $divisi,
            'gaji_pokok'    => $this->faker->numberBetween(4_000_000, 15_000_000),
            'tanggal_masuk' => $this->faker->dateTimeBetween('-8 years', '-6 months'),
            'status'        => $this->faker->randomElement(['aktif', 'aktif', 'aktif', 'cuti', 'nonaktif']),
        ];
    }
}
```

#### d. Buat Seeder

```bash
php artisan make:seeder PegawaiSeeder
```

Buka `database/seeders/PegawaiSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\Pegawai;
use Illuminate\Database\Seeder;

class PegawaiSeeder extends Seeder
{
    public function run(): void
    {
        Pegawai::factory(25)->create();
    }
}
```

Daftarkan seeder di `database/seeders/DatabaseSeeder.php`:

```php
public function run(): void
{
    $this->call([
        PegawaiSeeder::class,
    ]);
}
```

#### e. Jalankan Migration dan Seeder

```bash
php artisan migrate --seed
```

Verifikasi data masuk ke database:

```bash
php artisan tinker
>>> App\Models\Pegawai::count()  # Harus menampilkan: 25
```

---

### 5.3 Membuat Controller

```bash
php artisan make:controller LaporanController
```

Buka `app/Http/Controllers/LaporanController.php` dan isi dengan:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Pegawai;
use Barryvdh\DomPDF\Facade\Pdf;  // ✅ Import yang benar untuk Laravel 12
use Illuminate\Http\Request;

class LaporanController extends Controller
{
    /**
     * Tampilkan laporan PDF langsung di browser (tanpa download).
     */
    public function stream(Request $request)
    {
        $pdf = $this->buatPdf($request);

        // stream() = PDF tampil langsung di tab browser
        return $pdf->stream('laporan-pegawai.pdf');
    }

    /**
     * Download laporan PDF ke komputer pengguna.
     */
    public function download(Request $request)
    {
        $pdf = $this->buatPdf($request);

        $namaFile = 'laporan-pegawai-' . now()->format('Ymd') . '.pdf';

        // download() = paksa unduh file
        return $pdf->download($namaFile);
    }

    /**
     * Tampilkan preview laporan dalam format HTML.
     * Berguna saat development agar tidak perlu reload PDF terus-menerus.
     */
    public function preview(Request $request)
    {
        $data = $this->siapkanData($request);

        return view('laporan.pegawai', $data);
    }

    // -------------------------------------------------------
    // Private helper — hindari duplikasi kode
    // -------------------------------------------------------

    /**
     * Ambil data dari database dan siapkan variabel untuk view.
     */
    private function siapkanData(Request $request): array
    {
        // Ambil data dari database menggunakan Eloquent ORM
        $query = Pegawai::query();

        // Filter berdasarkan divisi jika ada di query string
        if ($request->filled('divisi')) {
            $query->where('divisi', $request->input('divisi'));
        }

        // Default: hanya tampilkan pegawai aktif
        // Gunakan scope yang sudah didefinisikan di Model
        $pegawai = $query->aktif()->orderBy('divisi')->orderBy('nama')->get();

        return [
            'pegawai'       => $pegawai,
            'divisi'        => $request->input('divisi', 'Semua Divisi'),
            'total_pegawai' => $pegawai->count(),
            'total_gaji'    => $pegawai->sum('gaji_pokok'),
            'tanggal_cetak' => now()->translatedFormat('d F Y'),
            'dicetak_oleh'  => auth()->user()?->name ?? 'Sistem',
        ];
    }

    /**
     * Buat instance PDF dari Blade view.
     */
    private function buatPdf(Request $request): \Barryvdh\DomPDF\PDF
    {
        $data = $this->siapkanData($request);

        $pdf = Pdf::loadView('laporan.pegawai', $data)
            ->setPaper('a4', 'landscape')           // kertas A4, orientasi landscape
            ->setOption('isHtml5ParserEnabled', true) // parser HTML5 lebih andal
            ->setOption('isRemoteEnabled', false);    // matikan akses URL eksternal

        return $pdf;
    }
}
```

> **Mengapa ada method `preview()`?** Saat development, membuka PDF setiap kali ada perubahan sangat lambat. Method ini merender HTML yang sama ke browser biasa sehingga hot-reload bekerja dengan cepat.

---

### 5.4 Membuat View Blade

Buat folder dan file view:

```bash
mkdir -p resources/views/laporan
```

Buat file `resources/views/laporan/pegawai.blade.php`:

```html
<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <title>Laporan Pegawai</title>
    <style>
      /*
         * PENTING: Hanya gunakan CSS yang didukung DomPDF.
         * HINDARI: flexbox, grid, CSS variables, box-shadow.
         * GUNAKAN: float, table, margin, padding, border, color.
         */

      * {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
      }

      body {
        font-family: "DejaVu Sans", sans-serif; /* font aman untuk DomPDF */
        font-size: 9pt;
        color: #2d2d2d;
        line-height: 1.5;
      }

      /* ================================================
           HEADER TETAP — muncul di setiap halaman
           ================================================ */
      header {
        position: fixed; /* DomPDF mendukung fixed untuk header/footer */
        top: -40px;
        left: 0;
        right: 0;
        height: 50px;
        border-bottom: 2px solid #1a4f91;
        padding-bottom: 6px;
      }

      .header-nama-instansi {
        font-size: 13pt;
        font-weight: bold;
        color: #1a4f91;
      }

      .header-keterangan {
        font-size: 8pt;
        color: #666;
      }

      /* ================================================
           FOOTER TETAP — muncul di setiap halaman
           ================================================ */
      footer {
        position: fixed;
        bottom: -25px;
        left: 0;
        right: 0;
        height: 22px;
        border-top: 1px solid #ddd;
        font-size: 7.5pt;
        color: #999;
        padding-top: 3px;
      }

      /* Nomor halaman otomatis DomPDF menggunakan CSS counter */
      .nomor-halaman:before {
        content: counter(page);
      }
      .total-halaman:before {
        content: counter(pages);
      }

      /* ================================================
           KONTEN UTAMA
           ================================================ */
      .konten {
        margin-top: 55px; /* ruang untuk header */
        margin-bottom: 30px; /* ruang untuk footer */
      }

      /* Judul laporan */
      .judul {
        text-align: center;
        margin-bottom: 14px;
      }

      .judul h1 {
        font-size: 14pt;
        font-weight: bold;
        color: #1a4f91;
        margin-bottom: 3px;
      }

      .judul p {
        font-size: 8.5pt;
        color: #666;
      }

      /* Informasi cetak — layout 2 kolom menggunakan float */
      .info-cetak {
        margin-bottom: 12px;
        overflow: hidden; /* clearfix untuk float */
      }

      .info-kiri {
        float: left;
        width: 50%;
      }
      .info-kanan {
        float: right;
        width: 50%;
      }

      .info-baris {
        font-size: 8.5pt;
        margin-bottom: 3px;
      }

      .info-label {
        display: inline-block;
        width: 110px;
        color: #666;
      }

      /* ================================================
           TABEL DATA
           ================================================ */
      table {
        width: 100%;
        border-collapse: collapse;
        margin-bottom: 14px;
      }

      thead tr {
        background-color: #1a4f91;
        color: #ffffff;
      }

      thead th {
        padding: 7px 8px;
        text-align: left;
        font-size: 8pt;
        font-weight: bold;
      }

      /* Zebra striping: baris genap berwarna beda */
      tbody tr:nth-child(even) {
        background-color: #f0f4ff;
      }

      tbody td {
        padding: 5px 8px;
        font-size: 8.5pt;
        border-bottom: 0.5px solid #e0e0e0;
        vertical-align: middle;
      }

      .teks-kanan {
        text-align: right;
      }
      .teks-tengah {
        text-align: center;
      }

      /* Baris total di bagian bawah tabel */
      .baris-total td {
        font-weight: bold;
        background-color: #dce8f5;
        border-top: 2px solid #1a4f91;
        padding: 7px 8px;
      }

      /* Badge status */
      .badge {
        display: inline-block;
        font-size: 7pt;
        padding: 1px 5px;
      }
      .badge-aktif {
        background: #d4edda;
        color: #155724;
      }
      .badge-cuti {
        background: #fff3cd;
        color: #856404;
      }
      .badge-nonaktif {
        background: #f8d7da;
        color: #721c24;
      }

      /* ================================================
           TANDA TANGAN
           ================================================ */
      .area-ttd {
        margin-top: 20px;
        overflow: hidden;
      }

      .kotak-ttd {
        float: right;
        width: 200px;
        text-align: center;
        font-size: 8.5pt;
      }

      .ruang-ttd {
        height: 45px; /* ruang kosong untuk tanda tangan */
      }

      .garis-ttd {
        border-top: 1px solid #333;
        padding-top: 3px;
        font-weight: bold;
      }
    </style>
  </head>
  <body>
    {{-- HEADER: tampil di setiap halaman --}}
    <header>
      <div class="header-nama-instansi">Politeknik Negeri Jember</div>
      <div class="header-keterangan">
        Dokumen Resmi · Laporan Data Kepegawaian
      </div>
    </header>

    {{-- FOOTER: tampil di setiap halaman --}}
    <footer>
      <span style="float:left;">
        Dicetak: {{ $tanggal_cetak }} · Oleh: {{ $dicetak_oleh }}
      </span>
      <span style="float:right;">
        Halaman <span class="nomor-halaman"></span> dari
        <span class="total-halaman"></span>
      </span>
    </footer>

    {{-- KONTEN UTAMA --}}
    <div class="konten">
      {{-- Judul --}}
      <div class="judul">
        <h1>LAPORAN DATA PEGAWAI</h1>
        <p>
          Divisi: {{ $divisi }} &nbsp;|&nbsp; Periode: {{
          now()->translatedFormat('F Y') }}
        </p>
      </div>

      {{-- Informasi Cetak --}}
      <div class="info-cetak">
        <div class="info-kiri">
          <div class="info-baris">
            <span class="info-label">Tanggal Cetak</span>:
            <strong>{{ $tanggal_cetak }}</strong>
          </div>
          <div class="info-baris">
            <span class="info-label">Dicetak Oleh</span>:
            <strong>{{ $dicetak_oleh }}</strong>
          </div>
        </div>
        <div class="info-kanan">
          <div class="info-baris">
            <span class="info-label">Total Pegawai</span>:
            <strong>{{ $total_pegawai }} orang</strong>
          </div>
          <div class="info-baris">
            <span class="info-label">Total Gaji Pokok</span>:
            <strong>Rp {{ number_format($total_gaji, 0, ',', '.') }}</strong>
          </div>
        </div>
      </div>

      {{-- Tabel Data Pegawai --}}
      <table>
        <thead>
          <tr>
            <th style="width:28px">No</th>
            <th style="width:65px">NIP</th>
            <th>Nama Pegawai</th>
            <th style="width:80px">Divisi</th>
            <th>Jabatan</th>
            <th style="width:95px">Gaji Pokok</th>
            <th style="width:65px">Tgl Masuk</th>
            <th style="width:60px">Lama Kerja</th>
            <th style="width:50px">Status</th>
          </tr>
        </thead>
        <tbody>
          @forelse ($pegawai as $index => $p)
          <tr>
            <td class="teks-tengah">{{ $index + 1 }}</td>
            <td>{{ $p->nip }}</td>
            <td>{{ $p->nama }}</td>
            <td>{{ $p->divisi }}</td>
            <td>{{ $p->jabatan }}</td>
            <td class="teks-kanan">{{ $p->gaji_format }}</td>
            <td class="teks-tengah">
              {{ $p->tanggal_masuk->format('d/m/Y') }}
            </td>
            <td class="teks-tengah">{{ $p->lama_kerja }}</td>
            <td class="teks-tengah">
              <span class="badge badge-{{ $p->status }}"
                >{{ ucfirst($p->status) }}</span
              >
            </td>
          </tr>
          @empty
          <tr>
            <td
              colspan="9"
              class="teks-tengah"
              style="padding: 20px; color: #999;"
            >
              Tidak ada data pegawai ditemukan.
            </td>
          </tr>
          @endforelse {{-- Baris Total --}} @if($pegawai->isNotEmpty())
          <tr class="baris-total">
            <td colspan="5" class="teks-kanan">Total</td>
            <td class="teks-kanan">
              Rp {{ number_format($total_gaji, 0, ',', '.') }}
            </td>
            <td colspan="3"></td>
          </tr>
          @endif
        </tbody>
      </table>

      {{-- Area Tanda Tangan --}}
      <div class="area-ttd">
        <div class="kotak-ttd">
          <p>{{ $tanggal_cetak }}</p>
          <p>Mengetahui, Kepala Divisi SDM</p>
          <div class="ruang-ttd"></div>
          <div class="garis-ttd">( _________________________ )</div>
        </div>
      </div>
    </div>
  </body>
</html>
```

---

### 5.5 Mendaftarkan Route

Buka `routes/web.php` dan tambahkan route berikut:

```php
<?php

use App\Http\Controllers\LaporanController;
use Illuminate\Support\Facades\Route;

// Route untuk halaman utama (biarkan yang sudah ada)
Route::get('/', function () {
    return view('welcome');
});

// -------------------------------------------------------
// Route Laporan PDF
// -------------------------------------------------------

// Preview dalam format HTML (tanpa autentikasi — hanya untuk development)
Route::get('/laporan/preview', [LaporanController::class, 'preview'])
    ->name('laporan.preview');

// Tampilkan PDF di browser (butuh login)
Route::get('/laporan/pdf', [LaporanController::class, 'stream'])
    ->middleware('auth')
    ->name('laporan.pdf');

// Download PDF ke komputer (butuh login)
Route::get('/laporan/download', [LaporanController::class, 'download'])
    ->middleware('auth')
    ->name('laporan.download');
```

> **Catatan:** Middleware `auth` memastikan hanya pengguna yang sudah login yang bisa mengakses endpoint PDF. Ini penting karena laporan pegawai mengandung data sensitif.

---

### 5.6 Menjalankan & Pengujian

**Langkah 1:** Pastikan server berjalan:

```bash
php artisan serve
```

**Langkah 2:** Uji endpoint preview terlebih dahulu (tidak butuh login):

```
http://127.0.0.1:8000/laporan/preview
```

Jika tampilan HTML sudah sesuai, lanjut ke pengujian PDF.

**Langkah 3:** Untuk menguji endpoint yang dilindungi `auth`, install Breeze (scaffolding auth) terlebih dahulu:

```bash
composer require laravel/breeze --dev
php artisan breeze:install blade
php artisan migrate
npm install && npm run dev
```

**Langkah 4:** Daftarkan akun di `/register`, lalu uji endpoint PDF:

```
http://127.0.0.1:8000/laporan/pdf       → PDF tampil di browser
http://127.0.0.1:8000/laporan/download  → File PDF terunduh
```

**Langkah 5:** Uji filter berdasarkan divisi:

```
http://127.0.0.1:8000/laporan/pdf?divisi=IT
http://127.0.0.1:8000/laporan/pdf?divisi=Keuangan
```

---

## 6. Konfigurasi Lanjutan DomPDF

Publish file konfigurasi jika belum:

```bash
php artisan vendor:publish --provider="Barryvdh\DomPDF\ServiceProvider"
```

Buka `config/dompdf.php` dan sesuaikan bagian `options`:

```php
'options' => [

    // Font default jika font tidak ditemukan
    'defaultFont' => 'DejaVu Sans',

    // Aktifkan parser HTML5 untuk kompatibilitas lebih baik
    'isHtml5ParserEnabled' => true,

    // MATIKAN akses URL remote (keamanan — hindari SSRF)
    'isRemoteEnabled' => false,

    // DPI rendering (96 = standar layar, 150 = cetak lebih tajam)
    'dpi' => 96,

    // Ukuran kertas default
    'defaultPaperSize' => 'a4',

    // Aktifkan subsetting font untuk file PDF lebih kecil
    'isFontSubsettingEnabled' => true,
],
```

> **Keamanan:** `isRemoteEnabled => false` penting di produksi untuk mencegah DomPDF mengakses URL eksternal yang berpotensi membahayakan server.

---

## 7. Fitur Tambahan

### 7.1 Header & Footer Tiap Halaman

Header dan footer per-halaman di DomPDF menggunakan `position: fixed` pada CSS, yang sudah diimplementasikan di Blade template pada Langkah 5.4.

Mekanismenya:

- `position: fixed; top: -Npx;` → elemen ditempatkan di area margin atas, muncul berulang tiap halaman.
- `position: fixed; bottom: -Npx;` → elemen ditempatkan di area margin bawah.
- `counter(page)` dan `counter(pages)` → nomor halaman otomatis dari engine DomPDF.

```css
/* Contoh implementasi header tetap */
header {
  position: fixed;
  top: -40px; /* nilai negatif = masuk ke area margin atas */
  left: 0;
  right: 0;
  height: 50px;
  border-bottom: 2px solid #1a4f91;
}

/* Nomor halaman otomatis */
.nomor-halaman:before {
  content: counter(page);
}
.total-halaman:before {
  content: counter(pages);
}
```

### 7.2 Orientasi Landscape

Atur orientasi melalui method `setPaper()` di controller:

```php
// Portrait (vertikal) — default
$pdf = Pdf::loadView('laporan.pegawai', $data)->setPaper('a4', 'portrait');

// Landscape (horizontal) — cocok untuk tabel banyak kolom
$pdf = Pdf::loadView('laporan.pegawai', $data)->setPaper('a4', 'landscape');

// Ukuran kertas lain yang tersedia:
// 'letter', 'legal', 'a3', 'a5'
```

### 7.3 Filter Data via Query String

Controller sudah mendukung filter melalui parameter URL. Tambahkan link di halaman untuk memudahkan akses:

```blade
{{-- Di halaman Blade biasa (bukan template PDF) --}}

<a href="{{ route('laporan.pdf') }}">Semua Divisi</a>
<a href="{{ route('laporan.pdf', ['divisi' => 'IT']) }}">Divisi IT</a>
<a href="{{ route('laporan.pdf', ['divisi' => 'Keuangan']) }}">Divisi Keuangan</a>
<a href="{{ route('laporan.download') }}">Download PDF</a>
```

---

## 8. Kendala Umum & Solusi

| No  | Gejala                           | Penyebab                           | Solusi                                                               |
| --- | -------------------------------- | ---------------------------------- | -------------------------------------------------------------------- |
| 1   | File PDF kosong / blank          | Syntax error di Blade template     | Gunakan `/laporan/preview` untuk debug tampilan HTML terlebih dahulu |
| 2   | Gambar tidak muncul di PDF       | Path gambar relatif tidak terbaca  | Gunakan `public_path('images/logo.png')` bukan path relatif          |
| 3   | Font aneh / karakter salah       | Font tidak ter-embed               | Gunakan `'DejaVu Sans'` atau font yang tersedia di DomPDF            |
| 4   | Layout berantakan                | Menggunakan Flexbox / Grid         | Ganti dengan layout `float` atau `table`                             |
| 5   | Error `Class 'PDF' not found`    | Menggunakan alias lama             | Ganti dengan `use Barryvdh\DomPDF\Facade\Pdf;`                       |
| 6   | Error `Call to undefined method` | Salah nama method Facade           | Periksa dokumentasi: `Pdf::loadView()`, bukan `PDF::load()`          |
| 7   | PDF terlalu lambat generate      | Data terlalu banyak / gambar besar | Batasi jumlah data per halaman, kompres gambar                       |
| 8   | Header/footer tidak muncul       | Nilai `top`/`bottom` tidak tepat   | Sesuaikan nilai negatif dengan margin halaman                        |
| 9   | `isRemoteEnabled` error          | Gambar dari URL eksternal          | Simpan gambar ke `public/` dan akses via `public_path()`             |
| 10  | Karakter Indonesia (é, ñ) salah  | Encoding tidak sesuai              | Pastikan file Blade tersimpan dalam encoding **UTF-8**               |

### Cara Debug Efektif

Selalu mulai debugging dari preview HTML, bukan dari PDF langsung:

```
1. Buka /laporan/preview
2. Perbaiki tampilan hingga benar di browser
3. Buka /laporan/pdf
4. Jika masih bermasalah di PDF, cek keterbatasan CSS DomPDF (lihat tabel di Bagian 2.4)
```

---

## 9. Perbandingan Library PDF di Laravel

Meskipun praktikum ini menggunakan DomPDF, penting untuk mengetahui alternatif lain agar dapat memilih tool yang tepat sesuai kebutuhan proyek:

| Kriteria            | DomPDF                     | Browsershot                  | mPDF                 |
| ------------------- | -------------------------- | ---------------------------- | -------------------- |
| **Package**         | `barryvdh/laravel-dompdf`  | `spatie/browsershot`         | `mpdf/mpdf`          |
| **Dependensi**      | PHP murni                  | Node.js + Puppeteer (Chrome) | PHP murni            |
| **Dukungan CSS**    | Terbatas (no flex/grid)    | Penuh (CSS modern)           | Cukup baik           |
| **Kecepatan**       | Cepat                      | Lambat (spawn browser)       | Menengah             |
| **Kualitas render** | Cukup                      | Sangat tinggi                | Baik                 |
| **Instalasi**       | Mudah                      | Perlu Node.js + Chrome       | Mudah                |
| **Cocok untuk**     | Laporan sederhana, invoice | Laporan kompleks, grafik     | Dokumen multilingual |
| **Ukuran file PDF** | Kecil                      | Besar                        | Menengah             |

**Kapan memilih DomPDF (rekomendasi default):**

- Laporan tabel sederhana
- Invoice dengan styling minimal
- Server shared hosting tanpa akses install binary
- Kebutuhan performa tinggi (generate banyak PDF)

**Kapan mempertimbangkan Browsershot:**

- Laporan dengan grafik Chart.js / D3.js
- Layout menggunakan Flexbox/Grid yang kompleks
- Tampilan harus identik persis dengan halaman web

---

## 10. Kesimpulan

Melalui praktikum ini, mahasiswa telah mempelajari dan mengimplementasikan:

1. **Instalasi DomPDF yang benar di Laravel 12** — menggunakan `use Barryvdh\DomPDF\Facade\Pdf;` dan tanpa registrasi manual Service Provider berkat fitur auto-discovery.

2. **Arsitektur MVC yang bersih** — data diambil dari database via Eloquent ORM (bukan hardcode), diproses di Controller, dan ditampilkan via Blade View.

3. **Template PDF profesional** — lengkap dengan header tetap, footer dengan nomor halaman otomatis, tabel data bergaris, badge status berwarna, dan area tanda tangan.

4. **Keamanan endpoint** — route PDF dilindungi `middleware('auth')` sehingga hanya pengguna terautentikasi yang bisa mengaksesnya.

5. **Alur debug yang efektif** — menggunakan endpoint `/preview` untuk memeriksa tampilan HTML sebelum di-render ke PDF, menghemat waktu pengembangan.

---
