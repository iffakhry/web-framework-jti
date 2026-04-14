# Panduan Praktikum — Laravel File Storage

**Laravel 12.x**

---

> **Catatan Versi:** Panduan ini ditulis khusus untuk **Laravel 12** (dirilis 24 Februari 2025). Terdapat beberapa perubahan (_breaking changes_) di Laravel 12 yang berdampak langsung pada materi File Storage — terutama pada path default disk `local`, perilaku validasi gambar, dan UUID. Perubahan ini dijelaskan secara eksplisit di setiap bagian yang terdampak.

---

## Capaian Pembelajaran

Setelah menyelesaikan praktikum ini, mahasiswa mampu:

1. Menjelaskan konsep dan arsitektur Laravel File Storage (Flysystem).
2. Membedakan karakteristik disk `local` dan `public`, serta memilih yang tepat sesuai kebutuhan.
3. Mengimplementasikan upload file ke disk lokal dan cloud (Cloudinary).
4. Menerapkan validasi file yang aman termasuk deteksi file berbahaya.
5. Mengimplementasikan multiple file upload.
6. Menerapkan rate limiting dan manajemen kuota storage per pengguna.
7. Menggunakan UUID secara konsisten dalam penamaan file.

---

## Dasar Teori

### Apa itu Laravel File Storage?

Laravel File Storage adalah sistem abstraksi penyimpanan file yang dibangun di atas package **Flysystem** milik The PHP League. Dengan abstraksi ini, kode aplikasi tidak perlu mengetahui secara detail _di mana_ file disimpan — apakah di disk lokal, server FTP, Amazon S3, atau layanan cloud lainnya. Cukup gunakan satu API yang seragam:

```php
Storage::put('folder/file.txt', 'Isi konten file');
Storage::get('folder/file.txt');
Storage::delete('folder/file.txt');
```

### Konsep Disk

Laravel menggunakan istilah **disk** untuk merujuk ke sebuah konfigurasi penyimpanan. Satu aplikasi bisa memiliki banyak disk sekaligus, misalnya disk `local` untuk file internal dan disk `s3` untuk file publik. Konfigurasi disk ada di `config/filesystems.php`.

---

## Alat dan Bahan

| Alat                             | Versi / Keterangan                            |
| -------------------------------- | --------------------------------------------- |
| PHP                              | >= 8.2 (wajib untuk Laravel 12)               |
| Laravel                          | 12.x                                          |
| Composer                         | >= 2.x                                        |
| Laravel Sail (Docker) atau XAMPP | Disarankan Sail untuk konsistensi environment |
| Postman / Hoppscotch             | Untuk pengujian endpoint API upload           |
| Git                              | Untuk version control dan pengumpulan tugas   |
| Akun Cloudinary                  | Daftar gratis di cloudinary.com               |

---

## Bagian 1 — Konfigurasi Awal

### 1.1 File Konfigurasi Disk

Buka `config/filesystems.php`. Struktur defaultnya adalah sebagai berikut:

> **⚠️ Breaking Change Laravel 12 — Local Disk Root Path**  
> Di Laravel 11 ke bawah, disk `local` secara default mengarah ke `storage/app`. Di **Laravel 12**, path default berubah menjadi **`storage/app/private`**. Artinya `Storage::disk('local')` sekarang membaca dan menulis ke `storage/app/private`, bukan `storage/app`.  
> Jika tidak mendefinisikan disk `local` secara eksplisit di `config/filesystems.php`, perilakunya akan berbeda dari versi sebelumnya.

```php
return [
    'default' => env('FILESYSTEM_DISK', 'local'),

    'disks' => [

        'local' => [
            'driver' => 'local',
            'root'   => storage_path('app/private'), // Laravel 12: default baru = app/private
            'throw'  => false,
        ],

        'public' => [
            'driver'     => 'local',
            'root'       => storage_path('app/public'),
            'url'        => env('APP_URL') . '/storage',
            'visibility' => 'public',
            'throw'      => false,
        ],

        's3' => [
            'driver'   => 's3',
            'key'      => env('AWS_ACCESS_KEY_ID'),
            'secret'   => env('AWS_SECRET_ACCESS_KEY'),
            'region'   => env('AWS_DEFAULT_REGION'),
            'bucket'   => env('AWS_BUCKET'),
            'url'      => env('AWS_URL'),
            'endpoint' => env('AWS_ENDPOINT'),
        ],

    ],

    'links' => [
        public_path('storage') => storage_path('app/public'),
    ],
];
```

### 1.2 Perbedaan Disk `local` vs `public`

Ini adalah salah satu konsep paling penting yang harus dipahami sebelum mengimplementasikan file upload.

#### Disk `local`

| Aspek                     | Detail                                                                                                                                       |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Lokasi fisik**          | `storage/app/private/` _(default baru di Laravel 12)_                                                                                        |
| **Bisa diakses browser?** | **Tidak.** File berada di luar `public/` sehingga tidak dapat diakses langsung via URL.                                                      |
| **Cara akses**            | Harus melalui controller Laravel yang mengontrol izin akses.                                                                                 |
| **Cocok untuk**           | Dokumen sensitif (KTP, kontrak, laporan keuangan), file yang hanya boleh diunduh oleh pemiliknya, file sementara yang diproses lalu dihapus. |

**Keuntungan disk `local`:**

- Aman — file tidak bisa ditebak atau diakses sembarangan via URL.
- Akses dikendalikan penuh oleh aplikasi (bisa dicek otentikasi, otorisasi, log).
- Cocok untuk kebutuhan compliance dan privasi data.

**Kekurangan disk `local`:**

- Setiap request untuk mengunduh file harus melalui PHP — lebih berat di sisi server.
- Tidak bisa langsung ditampilkan di `<img src="...">` atau dijadikan link langsung.

#### Disk `public`

| Aspek                     | Detail                                                                                  |
| ------------------------- | --------------------------------------------------------------------------------------- |
| **Lokasi fisik**          | `storage/app/public/`                                                                   |
| **Bisa diakses browser?** | **Ya**, setelah menjalankan `php artisan storage:link`.                                 |
| **Cara akses**            | Langsung via URL, misal: `https://domain.com/storage/photos/uuid.jpg`                   |
| **Cocok untuk**           | Foto profil, thumbnail produk, gambar artikel, file yang memang dimaksudkan untuk umum. |

**Keuntungan disk `public`:**

- Ringan — file disajikan langsung oleh web server (Nginx/Apache) tanpa melalui PHP.
- Bisa langsung digunakan di tag `<img>`, `<a>`, atau CSS.
- Performa lebih baik untuk file statis yang sering diakses.

**Kekurangan disk `public`:**

- Siapapun yang mengetahui URL dapat mengakses file tersebut.
- Tidak ada kontrol otentikasi/otorisasi pada level file — hanya bisa dibatasi via konfigurasi web server.

**Perintah wajib untuk disk `public`:**

```bash
php artisan storage:link
```

Perintah ini membuat symlink dari `public/storage` ke `storage/app/public` sehingga file bisa diakses via URL.

> **Catatan Industri:** Untuk aplikasi production, file publik yang sering diakses idealnya diunggah ke cloud object storage (S3, GCS, Cloudflare R2) dan disajikan melalui CDN agar latensi lebih rendah dan tidak membebani server utama.

---

## Bagian 2 — Upload ke Disk `local` (File Privat)

Skenario: Mahasiswa mengunggah dokumen tugas (PDF). File hanya boleh diunduh oleh dosen yang login.

### 2.1 Migration

```bash
php artisan make:migration create_uploads_table
```

```php
// database/migrations/xxxx_create_uploads_table.php
public function up(): void
{
    Schema::create('uploads', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->onDelete('cascade');
        $table->string('uuid')->unique();          // nama file di disk
        $table->string('original_name');           // nama asli dari pengguna
        $table->string('mime_type');
        $table->unsignedBigInteger('size');        // dalam bytes
        $table->string('disk')->default('local');  // disk yang digunakan
        $table->string('path');                    // path relatif di dalam disk
        $table->timestamps();
    });
}
```

```bash
php artisan migrate
```

### 2.2 Model

```php
// app/Models/Upload.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Upload extends Model
{
    protected $fillable = [
        'user_id', 'uuid', 'original_name',
        'mime_type', 'size', 'disk', 'path',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

### 2.3 Form View (Blade)

```html
<!-- resources/views/upload/local.blade.php -->
<form
  action="{{ route('upload.local.store') }}"
  method="POST"
  enctype="multipart/form-data"
>
  @csrf
  <div>
    <label>Pilih Dokumen (PDF, maks. 5MB)</label>
    <input type="file" name="document" accept=".pdf" />
    @error('document')
    <p style="color:red">{{ $message }}</p>
    @enderror
  </div>
  <button type="submit">Upload ke Local</button>
</form>
```

### 2.4 Controller — Upload ke `local`

```php
// app/Http/Controllers/LocalUploadController.php
namespace App\Http\Controllers;

use App\Models\Upload;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class LocalUploadController extends Controller
{
    /**
     * Daftar MIME type yang diizinkan beserta magic bytes-nya.
     * Magic bytes digunakan untuk memverifikasi isi file,
     * bukan hanya ekstensinya.
     */
    private const ALLOWED_MIME_TYPES = [
        'application/pdf' => ["\x25\x50\x44\x46"], // %PDF
    ];

    public function store(Request $request)
    {
        // --- 1. Validasi input dasar ---
        $request->validate([
            'document' => [
                'required',
                'file',
                'mimes:pdf',       // cek ekstensi
                'mimetypes:application/pdf', // cek MIME type dari konten
                'max:5120',        // maks 5MB (dalam kilobytes)
            ],
        ]);

        $file = $request->file('document');

        // --- 2. Validasi magic bytes (deteksi file berbahaya) ---
        $this->validateMagicBytes($file);

        // --- 3. Buat UUID unik sebagai nama file ---
        $uuid      = (string) Str::uuid();
        $extension = $file->getClientOriginalExtension();
        $filename  = $uuid . '.' . $extension;

        // --- 4. Simpan file ke disk local ---
        // File tersimpan di: storage/app/documents/{uuid}.pdf
        $path = $file->storeAs('documents', $filename, 'local');

        // --- 5. Simpan metadata ke database ---
        $upload = Upload::create([
            'user_id'       => Auth::id(),
            'uuid'          => $uuid,
            'original_name' => $file->getClientOriginalName(),
            'mime_type'     => $file->getMimeType(),
            'size'          => $file->getSize(),
            'disk'          => 'local',
            'path'          => $path,
        ]);

        return redirect()->back()->with('success', "File berhasil diupload. ID: {$upload->uuid}");
    }

    /**
     * Endpoint download — hanya pemilik file atau admin yang boleh mengunduh.
     * File disajikan melalui controller, BUKAN via URL langsung.
     */
    public function download(Upload $upload)
    {
        // Otorisasi: hanya pemilik yang boleh mengunduh
        if (Auth::id() !== $upload->user_id) {
            abort(403, 'Tidak diizinkan mengakses file ini.');
        }

        if (! Storage::disk('local')->exists($upload->path)) {
            abort(404, 'File tidak ditemukan.');
        }

        return Storage::disk('local')->download(
            $upload->path,
            $upload->original_name // nama file saat diunduh = nama asli
        );
    }

    /**
     * Validasi magic bytes untuk memastikan isi file sesuai dengan tipenya.
     * Mencegah file PHP/shell yang disamarkan dengan ekstensi .pdf
     */
    private function validateMagicBytes($file): void
    {
        $handle  = fopen($file->getRealPath(), 'rb');
        $header  = fread($handle, 8);
        fclose($handle);

        $mimeType    = $file->getMimeType();
        $allowedSigs = self::ALLOWED_MIME_TYPES[$mimeType] ?? null;

        if ($allowedSigs === null) {
            abort(422, 'Tipe file tidak diizinkan.');
        }

        $valid = false;
        foreach ($allowedSigs as $signature) {
            if (str_starts_with($header, $signature)) {
                $valid = true;
                break;
            }
        }

        if (! $valid) {
            abort(422, 'Konten file tidak sesuai dengan ekstensinya. File mungkin berbahaya.');
        }
    }
}
```

### 2.5 Route

```php
// routes/web.php
use App\Http\Controllers\LocalUploadController;

Route::middleware(['auth'])->group(function () {
    Route::get('/upload/local',           [LocalUploadController::class, 'create'])->name('upload.local.create');
    Route::post('/upload/local',          [LocalUploadController::class, 'store'])->name('upload.local.store');
    Route::get('/upload/local/{upload}/download', [LocalUploadController::class, 'download'])->name('upload.local.download');
});
```

---

## Bagian 3 — Upload ke Disk `public` (File Publik)

Skenario: Pengguna mengunggah foto profil. Gambar perlu ditampilkan di halaman profil menggunakan tag `<img>`.

> **⚠️ Breaking Change Laravel 12 — Validasi Gambar**  
> Di Laravel 11 ke bawah, rule `image` menerima SVG secara default. Di **Laravel 12**, SVG **tidak lagi diterima** oleh rule `image` secara default karena potensi XSS (SVG bisa mengandung JavaScript). Panduan ini menggunakan `File::image()` — cara modern dan eksplisit yang direkomendasikan di Laravel 12.

### 3.1 Controller — Upload ke `public`

```php
// app/Http/Controllers/PublicUploadController.php
namespace App\Http\Controllers;

use App\Models\Upload;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;
use Illuminate\Validation\Rules\File; // <-- import baru di Laravel 12

class PublicUploadController extends Controller
{
    /**
     * Magic bytes untuk gambar yang diizinkan.
     */
    private const ALLOWED_MIME_TYPES = [
        'image/jpeg' => ["\xFF\xD8\xFF"],                     // JPEG
        'image/png'  => ["\x89\x50\x4E\x47\x0D\x0A\x1A\x0A"], // PNG
        'image/webp' => ["RIFF", "WEBP"],                     // WebP
    ];

    public function store(Request $request)
    {
        // --- 1. Validasi ---
        $request->validate([
            // Menggunakan File::image() — cara yang direkomendasikan di Laravel 12.
            // SVG TIDAK diizinkan secara default (breaking change Laravel 12).
            // Gunakan File::image(allowSvg: true) HANYA jika benar-benar diperlukan.
            'photo' => [
                'required',
                File::image()          // validasi gambar gaya Laravel 12 (JPEG, PNG, WebP, GIF, BMP)
                    ->max(2 * 1024)    // maks 2MB (dalam kilobytes)
                    ->dimensions(minWidth: 100, minHeight: 100, maxWidth: 4096, maxHeight: 4096),
                'mimetypes:image/jpeg,image/png,image/webp', // double-check MIME type
            ],
        ]);

        $file = $request->file('photo');

        // --- 2. Validasi magic bytes ---
        $this->validateMagicBytes($file);

        // --- 3. Hapus foto profil lama (jika ada) ---
        $existing = Upload::where('user_id', Auth::id())
                          ->where('disk', 'public')
                          ->first();

        if ($existing && Storage::disk('public')->exists($existing->path)) {
            Storage::disk('public')->delete($existing->path);
            $existing->delete();
        }

        // --- 4. Generate UUID dan simpan file ---
        $uuid      = (string) Str::uuid();
        $extension = $file->getClientOriginalExtension();
        $filename  = $uuid . '.' . $extension;

        // File tersimpan di: storage/app/public/photos/{uuid}.jpg
        // Dapat diakses via: https://domain.com/storage/photos/{uuid}.jpg
        $path = $file->storeAs('photos', $filename, 'public');

        // --- 5. Simpan metadata ---
        $upload = Upload::create([
            'user_id'       => Auth::id(),
            'uuid'          => $uuid,
            'original_name' => $file->getClientOriginalName(),
            'mime_type'     => $file->getMimeType(),
            'size'          => $file->getSize(),
            'disk'          => 'public',
            'path'          => $path,
        ]);

        // URL publik yang bisa langsung digunakan di <img src="...">
        $publicUrl = Storage::disk('public')->url($path);

        return redirect()->back()->with([
            'success' => 'Foto profil berhasil diupload.',
            'photo_url' => $publicUrl,
        ]);
    }

    private function validateMagicBytes($file): void
    {
        $handle = fopen($file->getRealPath(), 'rb');
        $header = fread($handle, 12);
        fclose($handle);

        $mimeType    = $file->getMimeType();
        $allowedSigs = self::ALLOWED_MIME_TYPES[$mimeType] ?? null;

        if ($allowedSigs === null) {
            abort(422, 'Tipe file tidak diizinkan.');
        }

        $valid = false;
        foreach ($allowedSigs as $signature) {
            if (str_contains($header, $signature)) {
                $valid = true;
                break;
            }
        }

        if (! $valid) {
            abort(422, 'Konten file tidak sesuai dengan ekstensinya.');
        }
    }
}
```

### 3.2 Menampilkan File Publik di Blade

```html
{{-- resources/views/profile/show.blade.php --}} @php $upload =
auth()->user()->uploads()->where('disk', 'public')->first(); $photoUrl = $upload
? Storage::disk('public')->url($upload->path) :
asset('images/default-avatar.png'); @endphp

<img src="{{ $photoUrl }}" alt="Foto Profil" width="150" height="150" />
```

---

## Bagian 4 — Upload ke Cloudinary (Cloud Storage)

Cloudinary adalah layanan cloud untuk penyimpanan dan transformasi media (gambar dan video). Di industri, layanan seperti ini digunakan agar file tidak menghabiskan storage server dan dapat ditransformasi (resize, crop, format) secara on-demand melalui URL.

### 4.1 Instalasi Package

```bash
composer require cloudinary-labs/cloudinary-laravel
```

Publish konfigurasi:

```bash
php artisan vendor:publish --provider="CloudinaryLabs\CloudinaryLaravel\CloudinaryServiceProvider"
```

### 4.2 Konfigurasi di `.env`

Masukkan kredensial dari dashboard Cloudinary (cloudinary.com → Settings → API Keys):

```dotenv
CLOUDINARY_URL=cloudinary://API_KEY:API_SECRET@CLOUD_NAME
CLOUDINARY_UPLOAD_PRESET=your_upload_preset
```

Atau secara terpisah:

```dotenv
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

### 4.3 Tambahkan Disk Cloudinary (Opsional)

Di `config/filesystems.php`, tambahkan disk khusus Cloudinary:

```php
'cloudinary' => [
    'driver'        => 'cloudinary',
    'cloud_name'    => env('CLOUDINARY_CLOUD_NAME'),
    'api_key'       => env('CLOUDINARY_API_KEY'),
    'api_secret'    => env('CLOUDINARY_API_SECRET'),
    'secure'        => true,
],
```

### 4.4 Migration (tambahkan kolom `cloudinary_public_id`)

Tambahkan kolom ke tabel `uploads` agar kita bisa menghapus file dari Cloudinary nantinya:

```bash
php artisan make:migration add_cloudinary_id_to_uploads_table
```

```php
public function up(): void
{
    Schema::table('uploads', function (Blueprint $table) {
        $table->string('cloudinary_public_id')->nullable()->after('path');
    });
}
```

### 4.5 Controller — Upload ke Cloudinary

```php
// app/Http/Controllers/CloudinaryUploadController.php
namespace App\Http\Controllers;

use App\Models\Upload;
use CloudinaryLabs\CloudinaryLaravel\Facades\Cloudinary;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Str;
use Illuminate\Validation\Rules\File;

class CloudinaryUploadController extends Controller
{
    private const ALLOWED_MIME_TYPES = [
        'image/jpeg' => ["\xFF\xD8\xFF"],
        'image/png'  => ["\x89\x50\x4E\x47\x0D\x0A\x1A\x0A"],
        'image/webp' => ["RIFF"],
    ];

    public function store(Request $request)
    {
        // --- 1. Validasi (gaya Laravel 12 dengan File::image()) ---
        $request->validate([
            'image' => [
                'required',
                File::image()->max(5 * 1024), // maks 5MB, SVG tidak diizinkan (default Laravel 12)
                'mimetypes:image/jpeg,image/png,image/webp',
            ],
        ]);

        $file = $request->file('image');

        // --- 2. Validasi magic bytes ---
        $this->validateMagicBytes($file);

        // --- 3. Buat UUID untuk identifikasi unik ---
        $uuid = (string) Str::uuid();

        // --- 4. Upload ke Cloudinary ---
        // Cloudinary menghasilkan public_id sendiri, tapi kita bisa tentukan folder
        // dan gunakan UUID kita sebagai nama file di Cloudinary
        $uploadedFile = Cloudinary::upload(
            $file->getRealPath(),
            [
                'folder'          => 'praktikum/' . Auth::id(),
                'public_id'       => $uuid,          // UUID sebagai nama file
                'resource_type'   => 'image',
                'overwrite'       => true,
                // Transformasi otomatis: konversi ke WebP untuk efisiensi
                'transformation'  => [
                    ['quality' => 'auto', 'fetch_format' => 'auto'],
                ],
            ]
        );

        // Ambil hasil upload
        $cloudinaryPublicId = $uploadedFile->getPublicId();
        $secureUrl          = $uploadedFile->getSecurePath(); // HTTPS URL
        $width              = $uploadedFile->getWidth();
        $height             = $uploadedFile->getHeight();
        $fileSize           = $uploadedFile->getSize();

        // --- 5. Simpan metadata ke database ---
        $upload = Upload::create([
            'user_id'              => Auth::id(),
            'uuid'                 => $uuid,
            'original_name'        => $file->getClientOriginalName(),
            'mime_type'            => $file->getMimeType(),
            'size'                 => $fileSize ?? $file->getSize(),
            'disk'                 => 'cloudinary',
            'path'                 => $secureUrl,         // URL lengkap Cloudinary
            'cloudinary_public_id' => $cloudinaryPublicId,
        ]);

        return response()->json([
            'message'    => 'File berhasil diupload ke Cloudinary.',
            'uuid'       => $uuid,
            'url'        => $secureUrl,
            'dimensions' => "{$width}x{$height}",
        ]);
    }

    /**
     * Hapus file dari Cloudinary dan database.
     */
    public function destroy(Upload $upload)
    {
        if (Auth::id() !== $upload->user_id) {
            abort(403);
        }

        if ($upload->cloudinary_public_id) {
            Cloudinary::destroy($upload->cloudinary_public_id);
        }

        $upload->delete();

        return redirect()->back()->with('success', 'File berhasil dihapus dari Cloudinary.');
    }

    /**
     * Contoh transformasi URL Cloudinary on-demand.
     * Cloudinary bisa mengubah ukuran gambar hanya dengan mengubah URL-nya.
     */
    public function getThumbnailUrl(Upload $upload): string
    {
        // Menggunakan Cloudinary URL transformation
        // w_300,h_300,c_fill = resize ke 300x300 dengan crop fill
        return cloudinary_url($upload->cloudinary_public_id, [
            'transformation' => [
                'width'   => 300,
                'height'  => 300,
                'crop'    => 'fill',
                'quality' => 'auto',
                'format'  => 'webp',
            ],
        ]);
    }

    private function validateMagicBytes($file): void
    {
        $handle = fopen($file->getRealPath(), 'rb');
        $header = fread($handle, 12);
        fclose($handle);

        $mimeType    = $file->getMimeType();
        $allowedSigs = self::ALLOWED_MIME_TYPES[$mimeType] ?? null;

        if ($allowedSigs === null) {
            abort(422, 'Tipe file tidak diizinkan.');
        }

        $valid = false;
        foreach ($allowedSigs as $signature) {
            if (str_contains($header, $signature)) {
                $valid = true;
                break;
            }
        }

        if (! $valid) {
            abort(422, 'Konten file tidak sesuai dengan ekstensinya.');
        }
    }
}
```

### 4.6 Keunggulan Cloudinary vs Disk Lokal

| Fitur                   | Disk Lokal                                  | Cloudinary                                    |
| ----------------------- | ------------------------------------------- | --------------------------------------------- |
| **Penyimpanan**         | Server sendiri (terbatas)                   | Cloud (hampir tidak terbatas)                 |
| **Transformasi gambar** | Butuh library tambahan (Intervention Image) | Otomatis via URL                              |
| **CDN**                 | Tidak ada (perlu konfigurasi terpisah)      | Sudah termasuk                                |
| **Backup**              | Tanggung jawab developer                    | Ditangani Cloudinary                          |
| **Biaya**               | Gratis (tapi bayar storage server)          | Gratis (kuota terbatas), berbayar untuk lebih |
| **Performa global**     | Bergantung lokasi server                    | CDN global, cepat di mana saja                |

---

## Bagian 5 — Validasi File Berbahaya

Validasi file yang benar terdiri dari **tiga lapis** yang saling melengkapi:

### Lapis 1: Validasi Ekstensi & Ukuran (Laravel Validator)

```php
$request->validate([
    'file' => 'required|file|mimes:jpg,png,pdf|max:5120',
]);
```

Ini adalah lapisan pertama — cepat, tapi mudah dibypass karena hanya mengecek nama ekstensi.

### Lapis 2: Validasi MIME Type dari Konten (mimetypes)

```php
$request->validate([
    'file' => 'required|file|mimes:jpg,png,pdf|mimetypes:image/jpeg,image/png,application/pdf|max:5120',
]);
```

Rule `mimetypes:` membaca beberapa byte pertama file (magic bytes) secara internal melalui PHP `finfo`. Lebih handal dari `mimes:` saja.

### Lapis 3: Validasi Magic Bytes Secara Eksplisit

Ini adalah validasi paling aman. Kita membaca header file secara langsung dan mencocokkannya dengan tanda tangan (signature) biner yang diketahui:

```php
// app/Services/FileSecurityService.php
namespace App\Services;

use Illuminate\Http\UploadedFile;
use Illuminate\Validation\ValidationException;

class FileSecurityService
{
    /**
     * Tanda tangan biner (magic bytes) untuk setiap MIME type yang diizinkan.
     * Sumber referensi: https://en.wikipedia.org/wiki/List_of_file_signatures
     */
    private const MAGIC_BYTES = [
        'image/jpeg'      => ["\xFF\xD8\xFF"],
        'image/png'       => ["\x89\x50\x4E\x47\x0D\x0A\x1A\x0A"],
        'image/gif'       => ["GIF87a", "GIF89a"],
        'image/webp'      => ["RIFF"],                             // RIFF....WEBP
        'application/pdf' => ["\x25\x50\x44\x46"],                // %PDF
        'application/zip' => ["\x50\x4B\x03\x04"],                // PK..
    ];

    /**
     * Daftar ekstensi berbahaya yang sama sekali tidak boleh lolos,
     * bahkan jika pengguna mencoba menyamarkannya.
     */
    private const DANGEROUS_EXTENSIONS = [
        'php', 'php3', 'php4', 'php5', 'php7', 'phtml', 'phar',
        'asp', 'aspx', 'jsp', 'cgi', 'pl', 'py', 'rb', 'sh', 'bash',
        'exe', 'bat', 'cmd', 'ps1', 'vbs', 'js', 'jar',
    ];

    /**
     * Validasi file secara menyeluruh.
     * Melempar ValidationException jika file dianggap berbahaya.
     */
    public function validate(UploadedFile $file, array $allowedMimeTypes): void
    {
        $this->checkDangerousExtension($file);
        $this->checkMagicBytes($file, $allowedMimeTypes);
        $this->checkDoubleExtension($file);
    }

    /**
     * Cek apakah ekstensi file termasuk dalam daftar berbahaya.
     */
    private function checkDangerousExtension(UploadedFile $file): void
    {
        $extension = strtolower($file->getClientOriginalExtension());

        if (in_array($extension, self::DANGEROUS_EXTENSIONS, true)) {
            throw ValidationException::withMessages([
                'file' => "Ekstensi .{$extension} tidak diizinkan diunggah.",
            ]);
        }
    }

    /**
     * Cek magic bytes — apakah isi file sesuai dengan MIME type-nya.
     */
    private function checkMagicBytes(UploadedFile $file, array $allowedMimeTypes): void
    {
        $mimeType = $file->getMimeType();

        if (! in_array($mimeType, $allowedMimeTypes, true)) {
            throw ValidationException::withMessages([
                'file' => "Tipe file {$mimeType} tidak diizinkan.",
            ]);
        }

        $signatures = self::MAGIC_BYTES[$mimeType] ?? null;

        if ($signatures === null) {
            throw ValidationException::withMessages([
                'file' => 'Tipe file ini tidak terdaftar dalam whitelist yang diizinkan.',
            ]);
        }

        $handle = fopen($file->getRealPath(), 'rb');
        $header = fread($handle, 16);
        fclose($handle);

        $isValid = false;
        foreach ($signatures as $sig) {
            if (str_contains($header, $sig)) {
                $isValid = true;
                break;
            }
        }

        if (! $isValid) {
            throw ValidationException::withMessages([
                'file' => 'Konten file tidak sesuai dengan tipenya. Upload ditolak.',
            ]);
        }
    }

    /**
     * Cek double extension seperti "shell.php.jpg" atau "virus.jpg.php".
     */
    private function checkDoubleExtension(UploadedFile $file): void
    {
        $originalName = $file->getClientOriginalName();
        $parts        = explode('.', $originalName);

        // Hapus elemen pertama (nama file)
        array_shift($parts);

        foreach ($parts as $ext) {
            if (in_array(strtolower($ext), self::DANGEROUS_EXTENSIONS, true)) {
                throw ValidationException::withMessages([
                    'file' => 'Nama file mengandung ekstensi berbahaya.',
                ]);
            }
        }
    }
}
```

**Cara menggunakan `FileSecurityService` di controller:**

```php
use App\Services\FileSecurityService;

class UploadController extends Controller
{
    public function __construct(private FileSecurityService $fileSecurity) {}

    public function store(Request $request)
    {
        $request->validate([
            'file' => 'required|file|mimes:jpg,png,pdf|max:5120',
        ]);

        // Validasi keamanan mendalam
        $this->fileSecurity->validate(
            $request->file('file'),
            ['image/jpeg', 'image/png', 'application/pdf']
        );

        // Lanjut proses upload ...
        $uuid = (string) Str::uuid();
        // ...
    }
}
```

---

## Bagian 6 — Multiple File Upload

Skenario: Pengguna mengunggah beberapa lampiran sekaligus (hingga 5 file).

### 6.1 Form View

```html
<!-- resources/views/upload/multiple.blade.php -->
<form
  action="{{ route('upload.multiple.store') }}"
  method="POST"
  enctype="multipart/form-data"
>
  @csrf {{-- 'name' menggunakan array notation: files[] --}}
  <label>Pilih File (maks. 5 file, masing-masing 2MB)</label>
  <input type="file" name="files[]" multiple accept=".jpg,.jpeg,.png,.pdf" />

  @if ($errors->any())
  <ul style="color:red">
    @foreach ($errors->all() as $error)
    <li>{{ $error }}</li>
    @endforeach
  </ul>
  @endif

  <button type="submit">Upload Semua File</button>
</form>
```

### 6.2 Controller — Multiple Upload

```php
// app/Http/Controllers/MultipleUploadController.php
namespace App\Http\Controllers;

use App\Models\Upload;
use App\Services\FileSecurityService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class MultipleUploadController extends Controller
{
    public function __construct(private FileSecurityService $fileSecurity) {}

    public function store(Request $request)
    {
        // --- 1. Validasi kolektif ---
        $request->validate([
            'files'   => 'required|array|min:1|max:5',  // 1 sampai 5 file
            'files.*' => [
                'file',
                'mimes:jpg,jpeg,png,pdf',
                'mimetypes:image/jpeg,image/png,application/pdf',
                'max:2048',  // masing-masing maks 2MB
            ],
        ]);

        $files          = $request->file('files');
        $uploadedFiles  = [];
        $failedFiles    = [];

        // --- 2. Proses setiap file dalam satu transaksi database ---
        DB::beginTransaction();

        try {
            foreach ($files as $file) {
                // Validasi keamanan per file
                $this->fileSecurity->validate(
                    $file,
                    ['image/jpeg', 'image/png', 'application/pdf']
                );

                // UUID unik per file
                $uuid      = (string) Str::uuid();
                $extension = $file->getClientOriginalExtension();
                $filename  = $uuid . '.' . $extension;

                // Tentukan subfolder berdasarkan tipe file
                $folder = str_starts_with($file->getMimeType(), 'image/')
                    ? 'attachments/images'
                    : 'attachments/documents';

                // Simpan ke disk public
                $path = $file->storeAs($folder, $filename, 'public');

                // Simpan metadata
                $upload = Upload::create([
                    'user_id'       => Auth::id(),
                    'uuid'          => $uuid,
                    'original_name' => $file->getClientOriginalName(),
                    'mime_type'     => $file->getMimeType(),
                    'size'          => $file->getSize(),
                    'disk'          => 'public',
                    'path'          => $path,
                ]);

                $uploadedFiles[] = [
                    'uuid'          => $uuid,
                    'original_name' => $file->getClientOriginalName(),
                    'url'           => Storage::disk('public')->url($path),
                ];
            }

            DB::commit();

        } catch (\Throwable $e) {
            DB::rollBack();

            // Hapus file yang sudah terlanjur disimpan ke disk
            foreach ($uploadedFiles as $uploaded) {
                if (Storage::disk('public')->exists($uploaded['path'] ?? '')) {
                    Storage::disk('public')->delete($uploaded['path']);
                }
            }

            return redirect()->back()->withErrors([
                'files' => 'Terjadi kesalahan saat upload: ' . $e->getMessage(),
            ]);
        }

        return redirect()->back()->with([
            'success'        => count($uploadedFiles) . ' file berhasil diupload.',
            'uploaded_files' => $uploadedFiles,
        ]);
    }
}
```

### 6.3 Menampilkan Hasil Upload

```html
@if (session('success'))
<p style="color:green">{{ session('success') }}</p>

<ul>
  @foreach (session('uploaded_files', []) as $file)
  <li>
    {{ $file['original_name'] }} —
    <a href="{{ $file['url'] }}" target="_blank">Lihat File</a>
    (UUID: {{ $file['uuid'] }})
  </li>
  @endforeach
</ul>
@endif
```

---

## Bagian 7 — Rate Limiting & Quota Management

Tanpa pembatasan, endpoint upload rentan terhadap penyalahgunaan: pengguna bisa mengupload ribuan file dan menghabiskan storage server. Kita perlu dua mekanisme: **rate limiting** (batas frekuensi request) dan **quota management** (batas total storage per pengguna).

### 7.1 Rate Limiting — Batas Jumlah Request

Laravel menyediakan middleware `throttle` untuk membatasi jumlah request dalam jendela waktu tertentu.

#### Cara 1 — Throttle sederhana via route (untuk proteksi dasar)

```php
// routes/web.php

// Batasi endpoint upload: maksimal 10 request per menit per pengguna
Route::middleware(['auth', 'throttle:10,1'])->group(function () {
    Route::post('/upload/local',    [LocalUploadController::class, 'store']);
    Route::post('/upload/multiple', [MultipleUploadController::class, 'store']);
});
```

#### Cara 2 — Custom Rate Limiter (lebih fleksibel)

Daftarkan custom limiter di `App\Providers\AppServiceProvider`:

```php
// app/Providers/AppServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    // Limiter untuk upload: 20 upload per jam per user
    RateLimiter::for('file-upload', function (Request $request) {
        return Limit::perHour(20)
                    ->by(optional($request->user())->id ?: $request->ip())
                    ->response(function () {
                        return response()->json([
                            'message' => 'Terlalu banyak upload. Coba lagi dalam beberapa saat.',
                        ], 429);
                    });
    });
}
```

Gunakan di route:

```php
Route::middleware(['auth', 'throttle:file-upload'])->group(function () {
    Route::post('/upload', [UploadController::class, 'store']);
});
```

### 7.2 Quota Management — Batas Total Storage per Pengguna

Rate limiting hanya membatasi frekuensi, bukan total ukuran file yang tersimpan. Kita perlu kuota storage per pengguna.

#### Tambahkan kolom kuota ke tabel `users`

```bash
php artisan make:migration add_storage_quota_to_users_table
```

```php
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        // Kuota dalam bytes: default 50MB
        $table->unsignedBigInteger('storage_quota')->default(50 * 1024 * 1024);
    });
}
```

#### Service untuk mengecek dan menghitung kuota

```php
// app/Services/StorageQuotaService.php
namespace App\Services;

use App\Models\Upload;
use App\Models\User;
use Illuminate\Validation\ValidationException;

class StorageQuotaService
{
    /**
     * Hitung total storage yang sudah digunakan oleh pengguna (dalam bytes).
     */
    public function getUsedStorage(User $user): int
    {
        return Upload::where('user_id', $user->id)->sum('size');
    }

    /**
     * Cek apakah pengguna masih memiliki ruang untuk file baru.
     * Melempar exception jika kuota terlampaui.
     */
    public function ensureQuotaAvailable(User $user, int $fileSizeBytes): void
    {
        $usedBytes  = $this->getUsedStorage($user);
        $quota      = $user->storage_quota;
        $afterBytes = $usedBytes + $fileSizeBytes;

        if ($afterBytes > $quota) {
            $usedMb  = round($usedBytes / 1024 / 1024, 1);
            $quotaMb = round($quota / 1024 / 1024, 1);

            throw ValidationException::withMessages([
                'file' => "Kuota storage habis. Sudah terpakai {$usedMb}MB dari {$quotaMb}MB. "
                        . "Hapus beberapa file sebelum mengupload yang baru.",
            ]);
        }
    }

    /**
     * Kembalikan ringkasan penggunaan storage pengguna.
     */
    public function getSummary(User $user): array
    {
        $usedBytes = $this->getUsedStorage($user);
        $quota     = $user->storage_quota;

        return [
            'used_bytes'       => $usedBytes,
            'quota_bytes'      => $quota,
            'available_bytes'  => max(0, $quota - $usedBytes),
            'used_mb'          => round($usedBytes / 1024 / 1024, 2),
            'quota_mb'         => round($quota / 1024 / 1024, 2),
            'used_percent'     => $quota > 0 ? round(($usedBytes / $quota) * 100, 1) : 0,
        ];
    }
}
```

#### Integrasi ke controller upload

```php
// Tambahkan ke controller mana pun yang menerima upload

use App\Services\StorageQuotaService;

class UploadController extends Controller
{
    public function __construct(
        private FileSecurityService  $fileSecurity,
        private StorageQuotaService  $quota,
    ) {}

    public function store(Request $request)
    {
        $request->validate([
            'file' => 'required|file|mimes:jpg,png,pdf|max:5120',
        ]);

        $file = $request->file('file');

        // Cek kuota SEBELUM menyimpan file
        $this->quota->ensureQuotaAvailable(Auth::user(), $file->getSize());

        // Validasi keamanan
        $this->fileSecurity->validate($file, ['image/jpeg', 'image/png', 'application/pdf']);

        // Lanjutkan proses upload ...
        $uuid     = (string) Str::uuid();
        $filename = $uuid . '.' . $file->getClientOriginalExtension();
        $path     = $file->storeAs('uploads', $filename, 'public');

        Upload::create([
            'user_id'       => Auth::id(),
            'uuid'          => $uuid,
            'original_name' => $file->getClientOriginalName(),
            'mime_type'     => $file->getMimeType(),
            'size'          => $file->getSize(),
            'disk'          => 'public',
            'path'          => $path,
        ]);

        return redirect()->back()->with('success', 'File berhasil diupload.');
    }

    /**
     * Tampilkan informasi penggunaan kuota pengguna.
     */
    public function quotaInfo()
    {
        $summary = $this->quota->getSummary(Auth::user());

        return view('upload.quota', compact('summary'));
    }
}
```

#### View untuk menampilkan penggunaan kuota

```html
{{-- resources/views/upload/quota.blade.php --}}
<div>
  <h3>Storage Saya</h3>
  <p>
    Terpakai: {{ $summary['used_mb'] }} MB dari {{ $summary['quota_mb'] }} MB
  </p>

  {{-- Progress bar --}}
  <div style="background:#eee; border-radius:4px; height:12px; width:100%">
    <div
      style="
            background: {{ $summary['used_percent'] > 80 ? '#e74c3c' : '#2ecc71' }};
            width: {{ $summary['used_percent'] }}%;
            height: 100%;
            border-radius: 4px;
        "
    ></div>
  </div>
  <p>
    {{ $summary['used_percent'] }}% terpakai — tersisa {{
    round($summary['available_bytes'] / 1024 / 1024, 2) }} MB
  </p>
</div>
```

---

## Bagian 8 — Catatan Praktik Industri Lanjutan

Beberapa topik berikut tidak dibahas dalam praktikum ini, namun penting diketahui sebagai referensi untuk pengembangan lanjutan:

### 8.1 Asynchronous / Queued Upload

Untuk file berukuran besar atau proses yang lama (seperti konversi video, scan antivirus, atau resize batch), upload sebaiknya tidak diproses secara sinkron dalam satu HTTP request. Industri menggunakan **Laravel Queue** dengan driver seperti Redis atau database.

Alurnya:

1. File diterima oleh controller → disimpan sementara ke disk.
2. Controller mendispatch sebuah **Job** ke antrian: `ProcessUploadedFile::dispatch($upload)`.
3. Worker (biasanya Laravel Horizon) memproses job di background.
4. Controller langsung mengembalikan response ke pengguna tanpa menunggu proses selesai.

Untuk mempelajari lebih lanjut: [laravel.com/docs/queues](https://laravel.com/docs/queues)

### 8.2 Direct Upload ke Cloud (Bypass Server)

Untuk file sangat besar, teknik terbaik adalah mengunggah langsung dari browser ke cloud (S3, Cloudinary) tanpa melewati server Laravel sama sekali. Laravel hanya bertugas menerbitkan **signed URL sementara**, lalu browser mengunggah langsung ke cloud menggunakan URL tersebut.

### 8.3 Manajemen File Yatim (Orphan Files)

File yang gagal dikaitkan dengan data (misalnya karena error saat menyimpan ke database) akan tersisa di disk tanpa referensi. Gunakan **Scheduled Command** untuk membersihkan file-file yatim secara berkala.

---

## Ringkasan Operasi Storage

| Operasi                 | Contoh Kode                                                         |
| ----------------------- | ------------------------------------------------------------------- |
| Simpan file             | `Storage::put('file.txt', $isi)`                                    |
| Simpan upload dari form | `$request->file('foto')->storeAs('folder', $uuid.'.jpg', 'public')` |
| Cek file ada            | `Storage::disk('public')->exists($path)`                            |
| Ambil isi file          | `Storage::get($path)`                                               |
| Hapus file              | `Storage::disk('public')->delete($path)`                            |
| Ganti nama / pindah     | `Storage::move($pathLama, $pathBaru)`                               |
| Salin file              | `Storage::copy($path, $pathBaru)`                                   |
| Ambil URL publik        | `Storage::disk('public')->url($path)`                               |
| Download via controller | `Storage::disk('local')->download($path, $namaAsli)`                |

---
