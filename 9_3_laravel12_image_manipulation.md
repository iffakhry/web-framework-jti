# Panduan Praktikum: Laravel 12 Image Manipulation

**Mata Kuliah:** Workshop Sistem Informasi Web Framework  
**Acara:** 31 — Laravel Image Manipulation (Modernisasi)  
**Topik:** Manipulasi Gambar dengan Intervention Image v3 di Laravel 12

---

## Capaian Pembelajaran

Setelah menyelesaikan praktikum ini, mahasiswa mampu:

1. Menginstal dan mengkonfigurasi **Intervention Image v3** pada proyek Laravel 12.
2. Membuat **Form Request** untuk validasi upload gambar yang aman dan terstruktur.
3. Menerapkan **Separation of Concerns** dengan memisahkan logika pemrosesan ke dalam **Service Class**.
4. Menerapkan berbagai teknik manipulasi gambar: resize, crop, rotate, watermark, dan konversi format.
5. Menghasilkan output gambar dalam format modern **WebP** dan **AVIF**.
6. Menerapkan verifikasi MIME berbasis konten file untuk keamanan upload.
7. _(Opsional)_ Menghasilkan **multi-variant thumbnail** dari satu gambar yang diupload.

---

## Dasar Teori

### Intervention Image v3

**Intervention Image** adalah library PHP untuk manipulasi gambar yang menyediakan API yang bersih dan mudah digunakan. Versi 3 (terbaru) memperkenalkan perubahan signifikan dibandingkan v2:

| Aspek            | v2 (Lama / di Modul)              | v3 (Terbaru)                               |
| ---------------- | --------------------------------- | ------------------------------------------ |
| Package          | `intervention/image`              | `intervention/image-laravel`               |
| Cara buka file   | `Image::make($file)`              | `Image::read($file)`                       |
| Service Provider | Daftar manual di `config/app.php` | Auto-discovered otomatis                   |
| Driver           | GD / Imagick                      | GD / Imagick (konfigurasi via config file) |
| Encode output    | `->encode('webp')`                | `->toWebp(quality: 80)`                    |

### Separation of Concerns

Dalam pengembangan profesional, logika tidak boleh dijejalkan seluruhnya ke dalam Controller. Pola yang digunakan adalah:

- **Form Request** — menangani validasi input dari user.
- **Service Class** — menangani logika bisnis (dalam kasus ini, pemrosesan gambar).
- **Controller** — hanya sebagai orkestrator: menerima request, memanggil service, mengembalikan response.

### Format Gambar Modern

| Format | Dukungan Browser    | Ukuran vs JPEG      | Rekomendasi                    |
| ------ | ------------------- | ------------------- | ------------------------------ |
| JPEG   | Semua               | —                   | Legacy, masih umum             |
| PNG    | Semua               | Lebih besar         | Untuk gambar transparan        |
| WebP   | 96%+ browser modern | ~30-50% lebih kecil | **Direkomendasikan**           |
| AVIF   | 90%+ browser modern | ~50-80% lebih kecil | Terbaik, butuh dukungan server |

### Keamanan Upload Gambar

Validasi ekstensi file saja tidak cukup. Penyerang dapat mengganti ekstensi file berbahaya menjadi `.jpg`. Langkah keamanan yang benar:

1. Validasi MIME type berdasarkan **konten** file (bukan ekstensi).
2. Strip metadata **EXIF** untuk melindungi privasi pengguna (lokasi GPS, dll.).
3. Simpan dengan nama file yang di-generate server (bukan nama dari user).
4. Gunakan **Laravel Storage** dengan disk yang terkonfigurasi (bukan `public_path` langsung).

---

## Alat dan Bahan

- PC / Laptop dengan RAM minimal 4 GB
- PHP 8.2 atau lebih baru
- Composer
- Laravel 12 (sudah terinstal)
- Extension PHP: `ext-gd` atau `ext-imagick`
- Visual Studio Code atau editor pilihan

---

## Prosedur Kerja

### Langkah 1 — Persiapan Proyek Laravel 12

Jika belum memiliki proyek, buat terlebih dahulu:

```bash
composer create-project laravel/laravel image-app
cd image-app
```

Pastikan storage link sudah dibuat agar file yang disimpan bisa diakses publik:

```bash
php artisan storage:link
```

### Langkah 2 — Instalasi Intervention Image v3

Untuk Laravel 12, gunakan package `intervention/image-laravel` (bukan `intervention/image` seperti di modul lama):

```bash
composer require intervention/image-laravel
```

Package ini sudah mendukung **auto-discovery**, sehingga tidak perlu mendaftarkan Service Provider atau Facade secara manual di `config/app.php`.

#### 2a. Publish Konfigurasi (Opsional tapi Direkomendasikan)

```bash
php artisan vendor:publish --provider="Intervention\Image\Laravel\ServiceProvider"
```

Perintah ini akan menghasilkan file `config/image.php`. Buka file tersebut untuk memilih driver:

```php
// config/image.php
return [
    /*
    |--------------------------------------------------------------------------
    | Image Driver
    |--------------------------------------------------------------------------
    | Pilih "gd" (default, sudah ada di sebagian besar instalasi PHP)
    | atau "imagick" (lebih powerful, perlu ext-imagick terinstal)
    */
    'driver' => \Intervention\Image\Drivers\Gd\Driver::class,
    // 'driver' => \Intervention\Image\Drivers\Imagick\Driver::class,
];
```

#### 2b. Verifikasi Driver Aktif

Pastikan extension GD atau Imagick aktif di PHP:

```bash
php -m | grep -E "gd|imagick"
```

---

### Langkah 3 — Siapkan Model dan Migration

Buat model `Image` beserta migration-nya:

```bash
php artisan make:model Image -m
```

Edit file migration yang baru dibuat (`database/migrations/xxxx_create_images_table.php`):

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('images', function (Blueprint $table) {
            $table->id();
            $table->string('original_name');       // Nama asli file dari user
            $table->string('path');                // Path file utama yang disimpan
            $table->string('format', 10);          // Format output: webp, avif, jpeg
            $table->integer('width')->nullable();
            $table->integer('height')->nullable();
            $table->unsignedBigInteger('size')->nullable(); // Ukuran file dalam bytes
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('images');
    }
};
```

Jalankan migration:

```bash
php artisan migrate
```

Edit model `app/Models/Image.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Image extends Model
{
    protected $fillable = [
        'original_name',
        'path',
        'format',
        'width',
        'height',
        'size',
    ];
}
```

---

### Langkah 4 — Buat Form Request untuk Validasi

Form Request memisahkan logika validasi dari controller sehingga kode lebih bersih dan mudah diuji.

```bash
php artisan make:request UploadImageRequest
```

Edit file `app/Http/Requests/UploadImageRequest.php`:

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class UploadImageRequest extends FormRequest
{
    /**
     * Menentukan apakah user berhak melakukan request ini.
     * Ubah ke true, atau tambahkan logika autentikasi jika diperlukan.
     */
    public function authorize(): bool
    {
        return true;
    }

    /**
     * Aturan validasi untuk request ini.
     */
    public function rules(): array
    {
        return [
            'photo' => [
                'required',
                'file',
                // Validasi bahwa file adalah gambar (berdasarkan konten, bukan ekstensi)
                'image',
                // Izinkan hanya format tertentu
                'mimes:jpeg,jpg,png,webp',
                // Ukuran maksimal: 5 MB (5120 KB)
                'max:5120',
                // Validasi dimensi gambar
                'dimensions:min_width=50,min_height=50,max_width=6000,max_height=6000',
            ],
            // Opsi format output (opsional, default ke webp jika tidak diisi)
            'output_format' => [
                'nullable',
                'string',
                'in:webp,avif,jpeg',
            ],
            // Lebar target untuk resize (opsional)
            'resize_width' => [
                'nullable',
                'integer',
                'min:50',
                'max:3000',
            ],
        ];
    }

    /**
     * Pesan error yang lebih ramah pengguna.
     */
    public function messages(): array
    {
        return [
            'photo.required'        => 'Silakan pilih file gambar untuk diupload.',
            'photo.image'           => 'File yang diupload harus berupa gambar.',
            'photo.mimes'           => 'Format gambar yang diizinkan: JPEG, PNG, WebP.',
            'photo.max'             => 'Ukuran gambar tidak boleh melebihi 5 MB.',
            'photo.dimensions'      => 'Dimensi gambar tidak valid (min: 50x50px, maks: 6000x6000px).',
            'output_format.in'      => 'Format output harus salah satu dari: webp, avif, jpeg.',
            'resize_width.integer'  => 'Lebar resize harus berupa angka.',
            'resize_width.min'      => 'Lebar resize minimal 50 pixel.',
            'resize_width.max'      => 'Lebar resize maksimal 3000 pixel.',
        ];
    }
}
```

---

### Langkah 5 — Buat ImageService

Service class bertugas menampung **seluruh logika pemrosesan gambar**. Buat folder `Services` terlebih dahulu, lalu buat file-nya:

```bash
mkdir -p app/Services
```

Buat file `app/Services/ImageService.php`:

```php
<?php

namespace App\Services;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;
use Intervention\Image\Laravel\Facades\Image;
use RuntimeException;

class ImageService
{
    /**
     * Format MIME yang diizinkan, diverifikasi dari konten file.
     */
    private const ALLOWED_MIME_TYPES = [
        'image/jpeg',
        'image/png',
        'image/webp',
    ];

    /**
     * Verifikasi MIME type berdasarkan konten aktual file,
     * bukan hanya dari ekstensi yang bisa dimanipulasi.
     *
     * @throws RuntimeException Jika MIME type tidak diizinkan.
     */
    public function verifyMimeType(UploadedFile $file): void
    {
        $realMimeType = $file->getMimeType();

        if (!in_array($realMimeType, self::ALLOWED_MIME_TYPES, strict: true)) {
            throw new RuntimeException(
                "Tipe file tidak diizinkan. Terdeteksi: {$realMimeType}. " .
                "Hanya JPEG, PNG, dan WebP yang diperbolehkan."
            );
        }
    }

    /**
     * Proses gambar: verifikasi, resize, encode ke format modern, simpan.
     *
     * @param  UploadedFile  $file          File gambar dari request
     * @param  string        $outputFormat  Format output: 'webp', 'avif', atau 'jpeg'
     * @param  int|null      $resizeWidth   Lebar target (null = tidak di-resize)
     * @param  string        $disk          Laravel Storage disk ('public', 's3', dll.)
     * @return array{path: string, format: string, width: int, height: int, size: int, url: string}
     *
     * @throws RuntimeException
     */
    public function process(
        UploadedFile $file,
        string $outputFormat = 'webp',
        ?int $resizeWidth = null,
        string $disk = 'public'
    ): array {
        // ── Langkah 1: Verifikasi MIME dari konten file ──────────────────────
        $this->verifyMimeType($file);

        // ── Langkah 2: Baca gambar menggunakan Intervention Image v3 ─────────
        $image = Image::read($file->getRealPath());

        // ── Langkah 3: Resize jika lebar target diberikan ────────────────────
        if ($resizeWidth !== null) {
            // scaleDown menjaga aspek rasio dan tidak memperbesar gambar kecil
            $image->scaleDown(width: $resizeWidth);
        }

        // ── Langkah 4: Encode ke format yang dipilih ─────────────────────────
        $encoded = match ($outputFormat) {
            'avif'  => $image->toAvif(quality: 72),
            'jpeg'  => $image->toJpeg(quality: 85),
            default => $image->toWebp(quality: 80),   // webp sebagai default
        };

        // ── Langkah 5: Generate nama file yang aman (dari server, bukan user) ─
        $filename  = Str::uuid()->toString() . '.' . $outputFormat;
        $directory = 'images/' . now()->format('Y/m');
        $storagePath = "{$directory}/{$filename}";

        // ── Langkah 6: Simpan ke Laravel Storage ─────────────────────────────
        Storage::disk($disk)->put($storagePath, $encoded);

        // ── Langkah 7: Ambil metadata dari hasil encode ───────────────────────
        $savedPath   = Storage::disk($disk)->path($storagePath);
        [$width, $height] = @getimagesize($savedPath) ?: [0, 0];
        $fileSize    = Storage::disk($disk)->size($storagePath);

        return [
            'path'   => $storagePath,
            'format' => $outputFormat,
            'width'  => $width,
            'height' => $height,
            'size'   => $fileSize,
            'url'    => Storage::disk($disk)->url($storagePath),
        ];
    }

    /**
     * Hapus file gambar dari storage.
     */
    public function delete(string $path, string $disk = 'public'): bool
    {
        if (Storage::disk($disk)->exists($path)) {
            return Storage::disk($disk)->delete($path);
        }

        return false;
    }
}
```

---

### Langkah 6 — Buat Controller

```bash
php artisan make:controller ImageController
```

Edit `app/Http/Controllers/ImageController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\UploadImageRequest;
use App\Models\Image;
use App\Services\ImageService;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;
use RuntimeException;

class ImageController extends Controller
{
    /**
     * Inject ImageService melalui constructor (Dependency Injection).
     * Laravel akan secara otomatis menyelesaikan dependensi ini.
     */
    public function __construct(private readonly ImageService $imageService)
    {
    }

    /**
     * Tampilkan halaman utama dengan daftar gambar.
     */
    public function index(): View
    {
        $images = Image::latest()->paginate(12);

        return view('images.index', compact('images'));
    }

    /**
     * Tampilkan form upload.
     */
    public function create(): View
    {
        return view('images.create');
    }

    /**
     * Proses upload dan manipulasi gambar.
     *
     * Alur:
     * 1. Request sudah divalidasi oleh UploadImageRequest sebelum masuk ke sini.
     * 2. Panggil ImageService untuk memproses gambar.
     * 3. Simpan metadata ke database.
     * 4. Redirect dengan pesan sukses.
     */
    public function store(UploadImageRequest $request): RedirectResponse
    {
        try {
            $outputFormat = $request->input('output_format', 'webp');
            $resizeWidth  = $request->integer('resize_width') ?: null;

            // Delegasikan semua pemrosesan gambar ke Service
            $result = $this->imageService->process(
                file: $request->file('photo'),
                outputFormat: $outputFormat,
                resizeWidth: $resizeWidth,
            );

            // Simpan metadata ke database
            Image::create([
                'original_name' => $request->file('photo')->getClientOriginalName(),
                'path'          => $result['path'],
                'format'        => $result['format'],
                'width'         => $result['width'],
                'height'        => $result['height'],
                'size'          => $result['size'],
            ]);

            return redirect()
                ->route('images.index')
                ->with('success', 'Gambar berhasil diupload dan diproses!');

        } catch (RuntimeException $e) {
            // Tangani error keamanan (mis. MIME tidak valid) dengan pesan yang jelas
            return redirect()
                ->back()
                ->withInput()
                ->with('error', $e->getMessage());
        }
    }

    /**
     * Hapus gambar dari storage dan database.
     */
    public function destroy(Image $image): RedirectResponse
    {
        $this->imageService->delete($image->path);
        $image->delete();

        return redirect()
            ->route('images.index')
            ->with('success', 'Gambar berhasil dihapus.');
    }
}
```

---

### Langkah 7 — Daftarkan Routes

Edit `routes/web.php`:

```php
<?php

use App\Http\Controllers\ImageController;
use Illuminate\Support\Facades\Route;

Route::get('/', fn () => redirect()->route('images.index'));

Route::resource('images', ImageController::class)
    ->only(['index', 'create', 'store', 'destroy']);
```

---

### Langkah 8 — Buat View

#### 8a. Layout Utama

Buat `resources/views/layouts/app.blade.php`:

```html
<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@yield('title', 'Image Manipulation — Laravel 12')</title>
    <link
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
      rel="stylesheet"
    />
  </head>
  <body class="bg-light">
    <nav class="navbar navbar-dark bg-dark mb-4">
      <div class="container">
        <a class="navbar-brand" href="{{ route('images.index') }}">
          🖼️ Image App
        </a>
        <a
          href="{{ route('images.create') }}"
          class="btn btn-sm btn-outline-light"
        >
          + Upload Gambar
        </a>
      </div>
    </nav>

    <div class="container py-2">
      {{-- Notifikasi sukses --}} @if (session('success'))
      <div class="alert alert-success alert-dismissible fade show" role="alert">
        {{ session('success') }}
        <button
          type="button"
          class="btn-close"
          data-bs-dismiss="alert"
        ></button>
      </div>
      @endif {{-- Notifikasi error --}} @if (session('error'))
      <div class="alert alert-danger alert-dismissible fade show" role="alert">
        {{ session('error') }}
        <button
          type="button"
          class="btn-close"
          data-bs-dismiss="alert"
        ></button>
      </div>
      @endif @yield('content')
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
  </body>
</html>
```

#### 8b. Halaman Daftar Gambar

Buat `resources/views/images/index.blade.php`:

```html
@extends('layouts.app') @section('title', 'Daftar Gambar') @section('content')
<h4 class="mb-4">Galeri Gambar</h4>

@if ($images->isEmpty())
<div class="text-center text-muted py-5">
  <p>
    Belum ada gambar.
    <a href="{{ route('images.create') }}">Upload sekarang</a>.
  </p>
</div>
@else
<div class="row row-cols-1 row-cols-md-3 row-cols-lg-4 g-4">
  @foreach ($images as $image)
  <div class="col">
    <div class="card h-100 shadow-sm">
      <img
        src="{{ Storage::url($image->path) }}"
        class="card-img-top"
        style="height: 180px; object-fit: cover;"
        alt="{{ $image->original_name }}"
        loading="lazy"
      />
      <div class="card-body p-2">
        <p
          class="card-text small text-muted mb-1 text-truncate"
          title="{{ $image->original_name }}"
        >
          {{ $image->original_name }}
        </p>
        <div class="d-flex justify-content-between align-items-center">
          <span class="badge bg-secondary text-uppercase"
            >{{ $image->format }}</span
          >
          <small class="text-muted">
            {{ $image->width }}×{{ $image->height }}px
          </small>
        </div>
        <small class="text-muted d-block mt-1">
          {{ number_format($image->size / 1024, 1) }} KB
        </small>
      </div>
      <div class="card-footer p-2 bg-transparent border-top-0">
        <form
          method="POST"
          action="{{ route('images.destroy', $image) }}"
          onsubmit="return confirm('Hapus gambar ini?')"
        >
          @csrf @method('DELETE')
          <button class="btn btn-sm btn-outline-danger w-100">Hapus</button>
        </form>
      </div>
    </div>
  </div>
  @endforeach
</div>

<div class="mt-4">{{ $images->links() }}</div>
@endif @endsection
```

#### 8c. Form Upload

Buat `resources/views/images/create.blade.php`:

```html
@extends('layouts.app')

@section('title', 'Upload Gambar')

@section('content')
<div class="row justify-content-center">
    <div class="col-md-7">
        <div class="card shadow-sm">
            <div class="card-header bg-white">
                <h5 class="mb-0">Upload & Manipulasi Gambar</h5>
            </div>
            <div class="card-body">
                <form method="POST" action="{{ route('images.store') }}" enctype="multipart/form-data">
                    @csrf

                    {{-- Upload file --}}
                    <div class="mb-3">
                        <label for="photo" class="form-label fw-semibold">
                            File Gambar <span class="text-danger">*</span>
                        </label>
                        <input
                            type="file"
                            name="photo"
                            id="photo"
                            accept="image/jpeg,image/png,image/webp"
                            class="form-control @error('photo') is-invalid @enderror"
                        >
                        <div class="form-text">
                            Format: JPEG, PNG, WebP. Ukuran maks: 5 MB. Dimensi: 50×50 s/d 6000×6000 px.
                        </div>
                        @error('photo')
                            <div class="invalid-feedback">{{ $message }}</div>
                        @enderror
                    </div>

                    {{-- Format output --}}
                    <div class="mb-3">
                        <label for="output_format" class="form-label fw-semibold">Format Output</label>
                        <select name="output_format" id="output_format"
                                class="form-select @error('output_format') is-invalid @enderror">
                            <option value="webp" {{ old('output_format') === 'webp' ? 'selected' : '' }}>
                                WebP — Direkomendasikan (~30-50% lebih kecil dari JPEG)
                            </option>
                            <option value="avif" {{ old('output_format') === 'avif' ? 'selected' : '' }}>
                                AVIF — Terbaik (~50-80% lebih kecil, butuh PHP 8.1+ dengan libavif)
                            </option>
                            <option value="jpeg" {{ old('output_format') === 'jpeg' ? 'selected' : '' }}>
                                JPEG — Kompatibel semua browser
                            </option>
                        </select>
                        @error('output_format')
                            <div class="invalid-feedback">{{ $message }}</div>
                        @enderror
                    </div>

                    {{-- Resize opsional --}}
                    <div class="mb-4">
                        <label for="resize_width" class="form-label fw-semibold">
                            Resize Lebar (opsional)
                        </label>
                        <div class="input-group">
                            <input
                                type="number"
                                name="resize_width"
                                id="resize_width"
                                value="{{ old('resize_width') }}"
                                placeholder="Biarkan kosong untuk ukuran asli"
                                min="50"
                                max="3000"
                                class="form-control @error('resize_width') is-invalid @enderror"
                            >
                            <span class="input-group-text">px</span>
                        </div>
                        <div class="form-text">
                            Aspek rasio dipertahankan secara otomatis. Gambar tidak akan diperbesar.
                        </div>
                        @error('resize_width')
                            <div class="invalid-feedback">{{ $message }}</div>
                        @enderror
                    </div>

                    <div class="d-flex gap-2">
                        <button type="submit" class="btn btn-primary">
                            Upload & Proses Gambar
                        </button>
                        <a href="{{ route('images.index') }}" class="btn btn-outline-secondary">
                            Batal
                        </a>
                    </div>
                </form>
            </div>
        </div>
    </div>
</div>
@endsection
```

---

### Langkah 9 — Uji Coba Aplikasi

Jalankan server development:

```bash
php artisan serve
```

Buka browser dan akses `http://127.0.0.1:8000`. Lakukan pengujian berikut:

1. **Upload gambar valid** (JPEG/PNG/WebP ukuran wajar) → gambar harus muncul di galeri.
2. **Upload dengan format output berbeda** (WebP, AVIF, JPEG) → bandingkan ukuran file.
3. **Upload dengan resize** → isi kolom lebar, cek apakah dimensi output sesuai.
4. **Upload file bukan gambar** (mis. rename `.exe` jadi `.jpg`) → harus ditolak dengan error.
5. **Upload file terlalu besar** (> 5 MB) → harus ditolak validasi.
6. **Upload gambar terlalu kecil** (< 50×50 px) → harus ditolak validasi.

---

## ⚙️ Fitur Opsional — Multi-variant Thumbnails

Fitur ini berguna jika aplikasi membutuhkan beberapa ukuran gambar dari satu file upload, misalnya: thumbnail kecil untuk daftar, ukuran sedang untuk preview, dan ukuran penuh untuk tampilan detail.

### Tambahkan Method di ImageService

Tambahkan method berikut ke `app/Services/ImageService.php`:

```php
/**
 * Proses satu gambar menjadi beberapa variant ukuran sekaligus.
 *
 * @param  UploadedFile  $file
 * @param  string        $outputFormat  'webp', 'avif', atau 'jpeg'
 * @param  string        $disk
 * @return array<string, array{path: string, url: string, width: int, height: int, size: int}>
 *         Contoh key: 'thumb', 'medium', 'large'
 *
 * @throws RuntimeException
 */
public function processMultiVariant(
    UploadedFile $file,
    string $outputFormat = 'webp',
    string $disk = 'public'
): array {
    // Verifikasi MIME sekali di awal
    $this->verifyMimeType($file);

    // Definisi variant: nama => lebar maksimal (px)
    $variants = [
        'thumb'  => 200,
        'medium' => 600,
        'large'  => 1200,
    ];

    $baseUuid  = Str::uuid()->toString();
    $directory = 'images/' . now()->format('Y/m');
    $results   = [];

    foreach ($variants as $variantName => $maxWidth) {
        // Baca ulang setiap iterasi agar tidak menggunakan instance yang sudah dimodifikasi
        $image   = Image::read($file->getRealPath());
        $image->scaleDown(width: $maxWidth);

        $encoded = match ($outputFormat) {
            'avif'  => $image->toAvif(quality: 72),
            'jpeg'  => $image->toJpeg(quality: 85),
            default => $image->toWebp(quality: 80),
        };

        $filename    = "{$baseUuid}_{$variantName}.{$outputFormat}";
        $storagePath = "{$directory}/{$filename}";

        Storage::disk($disk)->put($storagePath, $encoded);

        $savedPath        = Storage::disk($disk)->path($storagePath);
        [$width, $height] = @getimagesize($savedPath) ?: [0, 0];

        $results[$variantName] = [
            'path'   => $storagePath,
            'url'    => Storage::disk($disk)->url($storagePath),
            'width'  => $width,
            'height' => $height,
            'size'   => Storage::disk($disk)->size($storagePath),
        ];
    }

    return $results;
}
```

### Cara Memanggil di Controller

```php
// Di method store() controller, ganti pemanggilan process() dengan:
$variants = $this->imageService->processMultiVariant(
    file: $request->file('photo'),
    outputFormat: $outputFormat,
);

// Akses masing-masing variant
$thumbUrl  = $variants['thumb']['url'];
$mediumUrl = $variants['medium']['url'];
$largeUrl  = $variants['large']['url'];

// Simpan semua path ke database (sesuaikan kolom migration jika perlu)
Image::create([
    'original_name' => $request->file('photo')->getClientOriginalName(),
    'path'          => $variants['large']['path'],     // path utama
    'path_thumb'    => $variants['thumb']['path'],
    'path_medium'   => $variants['medium']['path'],
    'format'        => $outputFormat,
    'width'         => $variants['large']['width'],
    'height'        => $variants['large']['height'],
    'size'          => $variants['large']['size'],
]);
```

> **Catatan:** Jika menggunakan multi-variant, tambahkan kolom `path_thumb` dan `path_medium` ke migration dan model `Image`.

---

## 💡 Informasi Lanjutan — Pemrosesan Asinkron

Pada aplikasi nyata dengan traffic tinggi atau gambar berukuran besar, memproses gambar secara langsung di dalam request-response cycle dapat memperlambat respons untuk pengguna.

Pendekatan yang umum digunakan di industri adalah **Laravel Queue + Jobs**:

1. Controller hanya menyimpan file sementara dan mengembalikan respons `202 Accepted` langsung ke user.
2. Sebuah **Job** (`ProcessUploadedImage`) didispatch ke antrian (queue).
3. **Queue worker** memproses gambar di background secara asinkron.
4. Setelah selesai, sistem bisa mengirim notifikasi ke user (email, WebSocket, dll.).

Topik ini mencakup konfigurasi **Laravel Horizon** (untuk monitoring queue), **Redis** sebagai queue driver, dan **broadcasting** untuk notifikasi real-time. Materi tersebut akan dibahas di modul terpisah.

---

## Referensi

- [Intervention Image v3 — Dokumentasi Resmi](https://image.intervention.io/v3)
- [Laravel 12 — File Storage](https://laravel.com/docs/12.x/filesystem)
- [Laravel 12 — Form Request Validation](https://laravel.com/docs/12.x/validation#form-request-validation)
- [WebP — Google Developers](https://developers.google.com/speed/webp)
- [AVIF — MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types#avif_image)
- [Spatie Laravel MediaLibrary](https://spatie.be/docs/laravel-medialibrary) — Alternatif enterprise untuk media management

---

## Format Laporan

- Kertas: A4, format PDF
- Struktur: Cover → Pendahuluan → Hasil Praktik (screenshot tiap langkah) → Analisis & Pembahasan → Kesimpulan → Daftar Pustaka
- Identitas: NIM, Nama, Golongan, Tugas Minggu Ke-9
- Penamaan file: `ACARA-31_GOL_NIM_NAMA.pdf`
- Kumpulkan di: [https://elearning-jti.polije.ac.id](https://elearning-jti.polije.ac.id)

---
