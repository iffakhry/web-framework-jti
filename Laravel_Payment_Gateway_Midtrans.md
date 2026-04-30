# Payment Gateway dengan Midtrans di Laravel 12

---

## C. Dasar Teori

### C.1 Apa itu Payment Gateway?

Payment Gateway adalah layanan perantara yang memfasilitasi transaksi elektronik antara aplikasi web (merchant) dengan penyedia layanan pembayaran (bank, e-wallet, dll). Ketika pengguna melakukan pembayaran, payment gateway:

1. Mengenkripsi data kartu/rekening
2. Mengirim ke bank/provider untuk otorisasi
3. Mengembalikan status ke merchant secara real-time

### C.2 Midtrans dan Produk-Produknya

Midtrans (bagian dari GoTo Financial) adalah payment gateway Indonesia terkemuka yang mendukung:

| Kategori        | Metode Pembayaran                      |
| --------------- | -------------------------------------- |
| Kartu           | Visa, Mastercard, JCB (kredit & debit) |
| Virtual Account | BCA, BNI, BRI, Mandiri, Permata, CIMB  |
| E-Wallet        | GoPay, ShopeePay, OVO, DANA            |
| QRIS            | Semua aplikasi berbasis QRIS           |
| Gerai Retail    | Indomaret, Alfamart                    |
| Cicilan         | Kartu kredit cicilan 0%                |

**Dua produk utama Midtrans:**

| Produk       | Deskripsi                                  | Cocok untuk                                 |
| ------------ | ------------------------------------------ | ------------------------------------------- |
| **Snap**     | UI popup/redirect siap pakai dari Midtrans | Praktikum ini — cepat, mudah diintegrasikan |
| **Core API** | API primitif, UI dibuat sendiri            | Kustomisasi penuh, butuh effort lebih       |

### C.3 Arsitektur Alur Pembayaran Snap

```
[Pengguna]
    |
    | 1. Klik "Bayar"
    v
[Laravel Backend]
    |
    | 2. Kirim order data ke Midtrans API
    |    POST https://app.sandbox.midtrans.com/snap/v1/transactions
    v
[Midtrans Server]
    |
    | 3. Kembalikan snap_token
    v
[Laravel Backend → Frontend]
    |
    | 4. Frontend panggil window.snap.pay(snap_token)
    v
[Midtrans Popup/Redirect]
    |
    | 5. Pengguna pilih metode & bayar
    v
[Midtrans Server]
    |
    | 6. Kirim webhook notification ke Laravel
    |    POST /midtrans/notification
    v
[Laravel Backend]
    |
    | 7. Validasi signature, update status order
    v
[Database — status: settlement/pending/failed]
```

### C.4 Mengapa Konfigurasi Midtrans Tidak Boleh di Constructor Controller?

Pada implementasi sebelumnya `Config::$serverKey = ...` di dalam `__construct()` controller. Ini **tidak disarankan** karena:

- Setiap controller yang butuh Midtrans harus copy-paste konfigurasi yang sama
- Sulit di-test (unit test sulit mock static property)
- Melanggar prinsip DRY (Don't Repeat Yourself)

**Praktik yang baik:** Taruh konfigurasi Midtrans di `AppServiceProvider::boot()` — dieksekusi sekali saat aplikasi booting.

---

## D. Alat dan Bahan

| No  | Komponen       | Spesifikasi                                |
| --- | -------------- | ------------------------------------------ |
| 1   | PC / Laptop    | RAM minimal 8 GB (rekomendasi)             |
| 2   | Sistem Operasi | Windows 10/11, macOS, atau Linux           |
| 3   | PHP            | >= 8.2 (requirement Laravel 12)            |
| 4   | Composer       | >= 2.x                                     |
| 5   | Laravel        | 12.x                                       |
| 6   | Database       | MySQL 8.x / MariaDB / SQLite               |
| 7   | Akun Midtrans  | Sandbox — daftar di dashboard.midtrans.com |
| 8   | Text Editor    | VS Code (rekomendasi)                      |
| 9   | Browser        | Chrome / Firefox (untuk testing)           |
| 10  | (Opsional)     | Ngrok untuk expose webhook ke publik       |

---

## E. Prosedur Kerja

### BAGIAN 1 — Persiapan Akun dan Project (Acara 37)

---

#### Langkah 1: Daftar Akun Midtrans Sandbox

1. Buka [https://dashboard.midtrans.com](https://dashboard.midtrans.com)
2. Klik **"Sign Up"** dan daftarkan akun
3. Setelah login, pastikan berada di mode **Sandbox** (toggle di pojok kiri atas)
4. Navigasi ke **Settings → Access Keys**
5. Catat dua kunci berikut:
   - `Server Key` (contoh: `SB-Mid-server-xxxxxxxxxxxxxxxx`)
   - `Client Key` (contoh: `SB-Mid-client-xxxxxxxxxxxxxxxx`)

> ⚠️ **Penting:** Server Key bersifat rahasia dan **tidak boleh** diekspos ke frontend/JavaScript. Hanya boleh digunakan di backend.

---

#### Langkah 2: Buat Project Laravel 12

```bash
composer create-project laravel/laravel toko-digital
cd toko-digital
```

Buat database baru (contoh: `toko_digital`) dan sesuaikan `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=toko_digital
DB_USERNAME=root
DB_PASSWORD=
```

---

#### Langkah 3: Install Midtrans PHP SDK

```bash
composer require midtrans/midtrans-php
```

> **Versi saat ini:** `midtrans/midtrans-php` v**2.6.2** (dirilis 18 Maret 2025)
> Versi ini menambahkan dukungan **Snap-BI** (Standar Nasional Open API Pembayaran).

---

#### Langkah 4: Konfigurasi Environment

Tambahkan variabel berikut ke file `.env`:

```env
MIDTRANS_SERVER_KEY=SB-Mid-server-GANTI_DENGAN_KEY_ANDA
MIDTRANS_CLIENT_KEY=SB-Mid-client-GANTI_DENGAN_KEY_ANDA
MIDTRANS_IS_PRODUCTION=false
MIDTRANS_IS_SANITIZED=true
MIDTRANS_IS_3DS=true
```

---

#### Langkah 5: Buat File Konfigurasi Midtrans

Buat file `config/midtrans.php`:

```php
<?php

return [
    'server_key'    => env('MIDTRANS_SERVER_KEY'),
    'client_key'    => env('MIDTRANS_CLIENT_KEY'),
    'is_production' => env('MIDTRANS_IS_PRODUCTION', false),
    'is_sanitized'  => env('MIDTRANS_IS_SANITIZED', true),
    'is_3ds'        => env('MIDTRANS_IS_3DS', true),
];
```

> [DIUBAH] Nama key menggunakan `snake_case` yang konsisten (`server_key`, bukan `serverKey`) untuk mengikuti konvensi Laravel.

---

#### Langkah 6: Registrasi Konfigurasi di AppServiceProvider [DITAMBAH]

Buka `app/Providers/AppServiceProvider.php` dan tambahkan inisialisasi Midtrans di method `boot()`:

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Midtrans\Config;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        //
    }

    public function boot(): void
    {
        // [DITAMBAH] Inisialisasi Midtrans sekali saat aplikasi booting
        Config::$serverKey    = config('midtrans.server_key');
        Config::$isProduction = config('midtrans.is_production');
        Config::$isSanitized  = config('midtrans.is_sanitized');
        Config::$is3ds        = config('midtrans.is_3ds');
    }
}
```

> **Mengapa di `boot()`?** Karena semua config sudah ter-load saat `boot()` dipanggil, berbeda dengan `register()`. Dengan cara ini, kita tidak perlu mengulangi konfigurasi di setiap controller.

---

### BAGIAN 2 — Membangun Studi Kasus: Toko Produk Digital (Acara 38)

**Studi Kasus:** Aplikasi toko e-book sederhana. Pengguna memilih produk, melakukan checkout, membayar via Midtrans, dan sistem otomatis memperbarui status pesanan.

---

#### Langkah 7: Buat Model dan Migrasi

**7a. Model dan Migrasi untuk Produk:**

```bash
php artisan make:model Product -m
```

Edit migrasi `database/migrations/xxxx_create_products_table.php`:

```php
public function up(): void
{
    Schema::create('products', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->text('description')->nullable();
        $table->unsignedBigInteger('price'); // Harga dalam Rupiah (integer, bukan desimal)
        $table->string('category')->default('ebook');
        $table->timestamps();
    });
}
```

**7b. Model dan Migrasi untuk Order:**

```bash
php artisan make:model Order -m
```

Edit migrasi `database/migrations/xxxx_create_orders_table.php`:

```php
public function up(): void
{
    Schema::create('orders', function (Blueprint $table) {
        $table->id();
        // [DITAMBAH] order_id unik sebagai referensi ke Midtrans
        $table->string('order_id')->unique();
        $table->foreignId('product_id')->constrained()->cascadeOnDelete();
        $table->string('customer_name');
        $table->string('customer_email');
        // [DITAMBAH] Status mengikuti nilai yang dikembalikan Midtrans
        $table->enum('status', [
            'pending',
            'settlement',
            'capture',
            'deny',
            'cancel',
            'expire',
            'failure',
        ])->default('pending');
        $table->unsignedBigInteger('gross_amount');
        // [DITAMBAH] Simpan snap_token untuk referensi/retry payment
        $table->string('snap_token')->nullable();
        $table->timestamps();
    });
}
```

**7c. Jalankan migrasi:**

```bash
php artisan migrate
```

**7d. Sesuaikan Model Order** (`app/Models/Order.php`):

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Order extends Model
{
    protected $fillable = [
        'order_id',
        'product_id',
        'customer_name',
        'customer_email',
        'status',
        'gross_amount',
        'snap_token',
    ];

    public function product()
    {
        return $this->belongsTo(Product::class);
    }

    // [DITAMBAH] Helper method untuk mengecek apakah order sudah dibayar
    public function isPaid(): bool
    {
        return in_array($this->status, ['settlement', 'capture']);
    }
}
```

---

#### Langkah 8: Seeder Data Produk

```bash
php artisan make:seeder ProductSeeder
```

Edit `database/seeders/ProductSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\Product;
use Illuminate\Database\Seeder;

class ProductSeeder extends Seeder
{
    public function run(): void
    {
        $products = [
            [
                'name'        => 'E-Book: Laravel 12 untuk Pemula',
                'description' => 'Panduan lengkap membangun aplikasi web modern dengan Laravel 12.',
                'price'       => 75000,
                'category'    => 'ebook',
            ],
            [
                'name'        => 'E-Book: Desain UI/UX dengan Figma',
                'description' => 'Belajar membuat desain antarmuka profesional dari nol.',
                'price'       => 50000,
                'category'    => 'ebook',
            ],
            [
                'name'        => 'Template SaaS Dashboard Premium',
                'description' => 'Template Bootstrap 5 siap pakai untuk proyek SaaS.',
                'price'       => 150000,
                'category'    => 'template',
            ],
        ];

        foreach ($products as $product) {
            Product::create($product);
        }
    }
}
```

Jalankan seeder:

```bash
php artisan db:seed --class=ProductSeeder
```

---

#### Langkah 9: Buat PaymentService (Separation of Concerns) [DITAMBAH]

Daripada menaruh logika Midtrans langsung di controller, kita buat Service class khusus.

```bash
mkdir -p app/Services
```

Buat file `app/Services/MidtransService.php`:

```php
<?php

namespace App\Services;

use App\Models\Order;
use App\Models\Product;
use Midtrans\Snap;
use Illuminate\Support\Str;

class MidtransService
{
    /**
     * Membuat order baru dan mendapatkan Snap Token dari Midtrans.
     *
     * @return array{order: Order, snap_token: string}
     */
    public function createTransaction(Product $product, array $customerData): array
    {
        // Generate order_id unik menggunakan UUID pendek
        $orderId = 'ORDER-' . strtoupper(Str::random(8)) . '-' . time();

        // Parameter yang dikirim ke Midtrans
        $params = [
            'transaction_details' => [
                'order_id'     => $orderId,
                'gross_amount' => $product->price,
            ],
            'item_details' => [
                [
                    'id'       => (string) $product->id,
                    'price'    => $product->price,
                    'quantity' => 1,
                    'name'     => $product->name,
                    'category' => $product->category,
                ],
            ],
            'customer_details' => [
                'first_name' => $customerData['name'],
                'email'      => $customerData['email'],
            ],
        ];

        // Request snap token ke Midtrans API
        $snapToken = Snap::getSnapToken($params);

        // Simpan order ke database
        $order = Order::create([
            'order_id'       => $orderId,
            'product_id'     => $product->id,
            'customer_name'  => $customerData['name'],
            'customer_email' => $customerData['email'],
            'status'         => 'pending',
            'gross_amount'   => $product->price,
            'snap_token'     => $snapToken,
        ]);

        return compact('order', 'snapToken');
    }

    /**
     * Memproses notifikasi webhook dari Midtrans.
     * Mengembalikan order yang diperbarui, atau null jika validasi gagal.
     */
    public function handleNotification(): ?Order
    {
        $notification = new \Midtrans\Notification();

        // [DITAMBAH] Validasi signature key untuk keamanan
        $serverKey         = config('midtrans.server_key');
        $orderId           = $notification->order_id;
        $statusCode        = $notification->status_code;
        $grossAmount       = $notification->gross_amount;
        $receivedSignature = $notification->signature_key;

        $expectedSignature = hash(
            'sha512',
            $orderId . $statusCode . $grossAmount . $serverKey
        );

        if ($receivedSignature !== $expectedSignature) {
            // Signature tidak valid — kemungkinan request palsu
            return null;
        }

        $transactionStatus = $notification->transaction_status;
        $fraudStatus       = $notification->fraud_status ?? null;

        // Tentukan status final berdasarkan kombinasi transaction_status dan fraud_status
        $finalStatus = match(true) {
            $transactionStatus === 'capture' && $fraudStatus === 'accept' => 'capture',
            $transactionStatus === 'settlement'                           => 'settlement',
            $transactionStatus === 'pending'                              => 'pending',
            in_array($transactionStatus, ['deny', 'cancel', 'expire', 'failure']) => $transactionStatus,
            default => 'pending',
        };

        $order = Order::where('order_id', $orderId)->first();

        if ($order) {
            $order->update(['status' => $finalStatus]);
        }

        return $order;
    }
}
```

---

#### Langkah 10: Buat Controller

```bash
php artisan make:controller PaymentController
```

Edit `app/Http/Controllers/PaymentController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Order;
use App\Models\Product;
use App\Services\MidtransService;
use Illuminate\Http\Request;

class PaymentController extends Controller
{
    public function __construct(
        private readonly MidtransService $midtransService
    ) {}

    /**
     * Tampilkan daftar produk.
     */
    public function index()
    {
        $products = Product::all();
        return view('products.index', compact('products'));
    }

    /**
     * Tampilkan form checkout untuk produk tertentu.
     */
    public function checkout(Product $product)
    {
        return view('payment.checkout', compact('product'));
    }

    /**
     * Proses checkout: buat order dan dapatkan snap_token dari Midtrans.
     */
    public function process(Request $request, Product $product)
    {
        $validated = $request->validate([
            'name'  => 'required|string|max:255',
            'email' => 'required|email|max:255',
        ]);

        try {
            $result = $this->midtransService->createTransaction($product, $validated);

            return response()->json([
                'snap_token' => $result['snapToken'],
                'order_id'   => $result['order']->order_id,
            ]);
        } catch (\Exception $e) {
            return response()->json([
                'error' => 'Gagal membuat transaksi: ' . $e->getMessage(),
            ], 500);
        }
    }

    /**
     * Halaman sukses setelah pembayaran.
     */
    public function success(Request $request)
    {
        $order = Order::where('order_id', $request->query('order_id'))->first();
        return view('payment.success', compact('order'));
    }

    /**
     * Halaman gagal/expired pembayaran.
     */
    public function failed(Request $request)
    {
        $order = Order::where('order_id', $request->query('order_id'))->first();
        return view('payment.failed', compact('order'));
    }

    /**
     * Endpoint webhook — menerima notifikasi dari Midtrans.
     * [DITAMBAH] Termasuk validasi signature.
     */
    public function notification(Request $request)
    {
        $order = $this->midtransService->handleNotification();

        if (!$order) {
            return response()->json(['message' => 'Invalid signature'], 403);
        }

        return response()->json([
            'message' => 'Notification handled',
            'order'   => $order->order_id,
            'status'  => $order->status,
        ]);
    }
}
```

---

#### Langkah 11: Daftarkan Route

Edit `routes/web.php`:

```php
<?php

use App\Http\Controllers\PaymentController;
use Illuminate\Support\Facades\Route;

// Halaman utama: daftar produk
Route::get('/', [PaymentController::class, 'index'])->name('products.index');

// Halaman checkout per produk
Route::get('/checkout/{product}', [PaymentController::class, 'checkout'])->name('payment.checkout');

// Proses checkout (AJAX) — kembalikan snap_token
Route::post('/checkout/{product}/process', [PaymentController::class, 'process'])->name('payment.process');

// Halaman setelah pembayaran
Route::get('/payment/success', [PaymentController::class, 'success'])->name('payment.success');
Route::get('/payment/failed', [PaymentController::class, 'failed'])->name('payment.failed');
```

Edit `routes/api.php` (atau tambah di web.php):

```php
// Webhook Midtrans — WAJIB POST, WAJIB dikecualikan dari CSRF
Route::post('/midtrans/notification', [PaymentController::class, 'notification'])
    ->name('midtrans.notification');
```

---

#### Langkah 12: Kecualikan Webhook dari CSRF [DITAMBAH — WAJIB]

Midtrans mengirim POST request dari servernya ke endpoint webhook kita. Request ini **tidak memiliki CSRF token** Laravel, sehingga akan ditolak secara default.

Buka `bootstrap/app.php` (Laravel 12 tidak lagi menggunakan `Kernel.php`):

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        // [DITAMBAH] Kecualikan endpoint webhook dari verifikasi CSRF
        $middleware->validateCsrfTokens(except: [
            'midtrans/notification',
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

> ⚠️ **Keamanan:** Mengecualikan CSRF aman di sini karena kita memvalidasi `signature_key` dari Midtrans sebagai pengganti CSRF. **Jangan pernah mengecualikan CSRF tanpa validasi signature.**

---

#### Langkah 13: Buat View — Daftar Produk

Buat `resources/views/products/index.blade.php`:

```html
<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Toko Digital</title>
    <link
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
      rel="stylesheet"
    />
  </head>
  <body class="bg-light">
    <div class="container py-5">
      <h1 class="mb-4 text-center fw-bold">📚 Toko Produk Digital</h1>
      <div class="row g-4">
        @foreach ($products as $product)
        <div class="col-md-4">
          <div class="card h-100 shadow-sm">
            <div class="card-body">
              <span class="badge bg-primary mb-2"
                >{{ ucfirst($product->category) }}</span
              >
              <h5 class="card-title">{{ $product->name }}</h5>
              <p class="card-text text-muted small">
                {{ $product->description }}
              </p>
              <p class="fw-bold text-success fs-5">
                Rp {{ number_format($product->price, 0, ',', '.') }}
              </p>
            </div>
            <div class="card-footer bg-transparent border-0 pb-3">
              <a
                href="{{ route('payment.checkout', $product) }}"
                class="btn btn-primary w-100"
              >
                🛒 Beli Sekarang
              </a>
            </div>
          </div>
        </div>
        @endforeach
      </div>
    </div>
  </body>
</html>
```

---

#### Langkah 14: Buat View — Halaman Checkout

Buat `resources/views/payment/checkout.blade.php`:

```html
<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Checkout — {{ $product->name }}</title>
    <link
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
      rel="stylesheet"
    />
    <meta name="csrf-token" content="{{ csrf_token() }}" />
  </head>
  <body class="bg-light">
    <div class="container py-5" style="max-width: 540px;">
      <div class="card shadow">
        <div class="card-header bg-primary text-white">
          <h4 class="mb-0">🛒 Checkout</h4>
        </div>
        <div class="card-body">
          <!-- Info Produk -->
          <div class="alert alert-info">
            <strong>Produk:</strong> {{ $product->name }}<br />
            <strong>Harga:</strong>
            <span class="fw-bold text-success">
              Rp {{ number_format($product->price, 0, ',', '.') }}
            </span>
          </div>

          <!-- Form Checkout -->
          <div id="checkout-form">
            <div class="mb-3">
              <label for="name" class="form-label fw-semibold"
                >Nama Lengkap</label
              >
              <input
                type="text"
                id="name"
                class="form-control"
                placeholder="Masukkan nama Anda"
                required
              />
            </div>
            <div class="mb-3">
              <label for="email" class="form-label fw-semibold">Email</label>
              <input
                type="email"
                id="email"
                class="form-control"
                placeholder="Masukkan email Anda"
                required
              />
            </div>
            <button id="pay-button" class="btn btn-success w-100 py-2 fw-bold">
              💳 Bayar Sekarang
            </button>
          </div>

          <!-- Alert untuk pesan error -->
          <div id="error-alert" class="alert alert-danger mt-3 d-none"></div>
          <div id="loading" class="text-center mt-3 d-none">
            <div class="spinner-border text-primary" role="status"></div>
            <p class="mt-2 text-muted">Menghubungi Midtrans...</p>
          </div>
        </div>
      </div>
      <div class="text-center mt-3">
        <a
          href="{{ route('products.index') }}"
          class="text-decoration-none text-muted"
        >
          ← Kembali ke Daftar Produk
        </a>
      </div>
    </div>

    {{-- Midtrans Snap.js SDK --}} {{-- Ganti "sandbox" dengan "app" untuk
    production --}}
    <script
      src="https://app.sandbox.midtrans.com/snap/snap.js"
      data-client-key="{{ config('midtrans.client_key') }}"
    ></script>

    <script>
      document
        .getElementById("pay-button")
        .addEventListener("click", function () {
          const name = document.getElementById("name").value.trim();
          const email = document.getElementById("email").value.trim();
          const errorAlert = document.getElementById("error-alert");
          const loading = document.getElementById("loading");

          // Validasi sederhana di sisi klien
          if (!name || !email) {
            errorAlert.textContent = "Nama dan email wajib diisi.";
            errorAlert.classList.remove("d-none");
            return;
          }

          errorAlert.classList.add("d-none");
          loading.classList.remove("d-none");
          this.disabled = true;

          // Request snap_token ke backend
          fetch('{{ route("payment.process", $product) }}', {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
              "X-CSRF-TOKEN": document.querySelector('meta[name="csrf-token"]')
                .content,
            },
            body: JSON.stringify({ name, email }),
          })
            .then((res) => res.json())
            .then((data) => {
              loading.classList.add("d-none");

              if (data.error) {
                errorAlert.textContent = data.error;
                errorAlert.classList.remove("d-none");
                this.disabled = false;
                return;
              }

              // Tampilkan popup Midtrans Snap
              window.snap.pay(data.snap_token, {
                onSuccess: function (result) {
                  window.location.href =
                    '{{ route("payment.success") }}?order_id=' + data.order_id;
                },
                onPending: function (result) {
                  alert(
                    "Pembayaran tertunda. Selesaikan pembayaran sebelum batas waktu.",
                  );
                  window.location.href =
                    '{{ route("payment.success") }}?order_id=' + data.order_id;
                },
                onError: function (result) {
                  window.location.href =
                    '{{ route("payment.failed") }}?order_id=' + data.order_id;
                },
                onClose: function () {
                  // Pengguna menutup popup tanpa bayar
                  document.getElementById("pay-button").disabled = false;
                  loading.classList.add("d-none");
                },
              });
            })
            .catch((err) => {
              loading.classList.add("d-none");
              errorAlert.textContent = "Terjadi kesalahan jaringan. Coba lagi.";
              errorAlert.classList.remove("d-none");
              document.getElementById("pay-button").disabled = false;
            });
        });
    </script>
  </body>
</html>
```

---

#### Langkah 15: Buat View — Sukses & Gagal

**`resources/views/payment/success.blade.php`:**

```html
<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <title>Pembayaran Berhasil</title>
    <link
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
      rel="stylesheet"
    />
  </head>
  <body class="bg-light">
    <div class="container py-5 text-center" style="max-width: 480px;">
      <div class="card shadow">
        <div class="card-body py-5">
          <div class="display-1 mb-3">✅</div>
          <h2 class="fw-bold text-success">Terima Kasih!</h2>
          @if ($order)
          <p class="text-muted">
            Order <strong>{{ $order->order_id }}</strong> sedang diproses.<br />
            Status saat ini:
            <span class="badge bg-warning">{{ $order->status }}</span>
          </p>
          <p class="small text-muted">
            Konfirmasi akan dikirim ke
            <strong>{{ $order->customer_email }}</strong>
            setelah pembayaran diverifikasi.
          </p>
          @endif
          <a href="{{ route('products.index') }}" class="btn btn-primary mt-3">
            Kembali ke Toko
          </a>
        </div>
      </div>
    </div>
  </body>
</html>
```

**`resources/views/payment/failed.blade.php`:**

```html
<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <title>Pembayaran Gagal</title>
    <link
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
      rel="stylesheet"
    />
  </head>
  <body class="bg-light">
    <div class="container py-5 text-center" style="max-width: 480px;">
      <div class="card shadow">
        <div class="card-body py-5">
          <div class="display-1 mb-3">❌</div>
          <h2 class="fw-bold text-danger">Pembayaran Gagal</h2>
          @if ($order)
          <p class="text-muted">
            Order <strong>{{ $order->order_id }}</strong> tidak berhasil.<br />
            Status: <span class="badge bg-danger">{{ $order->status }}</span>
          </p>
          @endif
          <a
            href="{{ route('products.index') }}"
            class="btn btn-outline-primary mt-3"
          >
            Coba Lagi
          </a>
        </div>
      </div>
    </div>
  </body>
</html>
```

---

#### Langkah 16: Menguji dengan Sandbox Midtrans [DITAMBAH]

Jalankan server development:

```bash
php artisan serve
```

Buka `http://localhost:8000` di browser.

**Kartu Kredit Test untuk Sandbox:**

| Skenario               | Nomor Kartu           | Expired | CVV | OTP    |
| ---------------------- | --------------------- | ------- | --- | ------ |
| **Pembayaran Sukses**  | `4811 1111 1111 1114` | 01/39   | 123 | 112233 |
| **Pembayaran Ditolak** | `4911 1111 1111 1113` | 01/39   | 123 | 112233 |

**Langkah pengujian:**

1. Buka halaman produk, klik "Beli Sekarang"
2. Isi nama dan email, klik "Bayar Sekarang"
3. Popup Midtrans muncul → pilih "Credit Card"
4. Masukkan nomor kartu test di atas
5. Klik "Bayar" → masukkan OTP `112233`
6. Amati redirect ke halaman sukses

---

#### Langkah 17: Menguji Webhook dengan Ngrok [DITAMBAH]

Midtrans perlu mengirim notifikasi ke URL yang bisa diakses dari internet. Untuk development lokal, gunakan ngrok:

```bash
# Install ngrok: https://ngrok.com/download
ngrok http 8000
```

Ngrok akan memberikan URL publik seperti `https://abc123.ngrok.io`.

1. Login ke [dashboard.midtrans.com](https://dashboard.midtrans.com)
2. Navigasi ke **Settings → Configuration**
3. Isi **Payment Notification URL** dengan:
   ```
   https://abc123.ngrok.io/midtrans/notification
   ```
4. Klik Save, lalu lakukan pembayaran test

Cek log Laravel untuk melihat notifikasi masuk:

```bash
tail -f storage/logs/laravel.log
```

---

### BAGIAN 3 — Catatan Penting & Keamanan

---

#### Catatan A: Status Transaksi Midtrans

Memahami status sangat penting untuk menentukan kapan konten digital boleh diberikan ke pembeli:

| Status       | Artinya                           | Aksi yang Disarankan              |
| ------------ | --------------------------------- | --------------------------------- |
| `pending`    | Menunggu pembayaran               | Tampilkan instruksi pembayaran    |
| `settlement` | Pembayaran diterima (VA/retail)   | ✅ Berikan akses produk           |
| `capture`    | Kartu kredit diotorisasi          | ✅ Berikan akses produk           |
| `deny`       | Pembayaran ditolak bank           | Minta coba metode lain            |
| `cancel`     | Dibatalkan oleh pengguna/merchant | Reset ke pending atau hapus order |
| `expire`     | Melebihi batas waktu pembayaran   | Buat ulang order jika perlu       |
| `failure`    | Gagal (error teknis)              | Hubungi support                   |

> ⚠️ Untuk kartu kredit, status `capture` (bukan `settlement`) yang menandakan pembayaran berhasil, **terutama jika fraud status = `accept`**.

---

#### Catatan B: Perbedaan Sandbox vs Production

| Aspek       | Sandbox                        | Production                    |
| ----------- | ------------------------------ | ----------------------------- |
| URL Snap.js | `app.sandbox.midtrans.com`     | `app.midtrans.com`            |
| Server Key  | Prefix `SB-Mid-server-`        | Tanpa prefix SB               |
| Client Key  | Prefix `SB-Mid-client-`        | Tanpa prefix SB               |
| `.env`      | `MIDTRANS_IS_PRODUCTION=false` | `MIDTRANS_IS_PRODUCTION=true` |
| Transaksi   | Tidak nyata, untuk testing     | Transaksi nyata               |

---

## C. Troubleshooting

| Masalah                                  | Kemungkinan Penyebab                                | Solusi                                                                              |
| ---------------------------------------- | --------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `cURL error 60: SSL certificate problem` | SSL tidak dikonfigurasi                             | Tambahkan `Config::$curlOptions = [CURLOPT_SSL_VERIFYPEER => false]` di development |
| Webhook tidak diterima (404)             | Route salah atau CSRF memblokir                     | Pastikan route POST ada dan dikecualikan dari CSRF                                  |
| `Signature key mismatch`                 | Server Key salah atau `gross_amount` format berbeda | Gunakan nilai `gross_amount` persis dari notifikasi Midtrans                        |
| Popup Snap tidak muncul                  | `client_key` salah atau Snap.js tidak ter-load      | Periksa console browser dan `MIDTRANS_CLIENT_KEY` di `.env`                         |
| Status tidak terupdate                   | Webhook URL tidak dapat diakses Midtrans            | Gunakan ngrok untuk development lokal                                               |
| `Class "Midtrans\Config" not found`      | Autoloader belum diperbarui                         | Jalankan `composer dump-autoload`                                                   |

---

## D. Kesimpulan

Integrasi payment gateway Midtrans dengan Laravel 12 dapat dilakukan secara terstruktur dengan mengikuti pola arsitektur yang baik:

- **Konfigurasi terpusat** di `AppServiceProvider` menghindari duplikasi kode
- **Service class** (`MidtransService`) memisahkan logika bisnis dari controller
- **Validasi signature** pada webhook memastikan notifikasi hanya diproses jika benar-benar berasal dari Midtrans
- **Penanganan semua status** transaksi menjamin pengalaman pengguna yang konsisten

Midtrans Snap menyederhanakan pengalaman pembayaran dengan menyediakan UI siap pakai yang mendukung puluhan metode pembayaran populer di Indonesia, menjadikannya pilihan ideal untuk aplikasi e-commerce dan layanan digital.

---

## E. Referensi

1. Midtrans. (2025). _Official PHP Library for Midtrans Payment API_ (v2.6.2). GitHub. https://github.com/Midtrans/midtrans-php
2. Midtrans. (2026). _Midtrans Technical Documentation_. https://docs.midtrans.com
3. Laravel. (2024). _Laravel 12.x Documentation_. https://laravel.com/docs/12.x
4. Midtrans. (2026). _Sandbox Testing_. https://docs.midtrans.com/docs/testing-payment-on-sandbox
5. Midtrans. (2026). _Handling Notifications / Webhooks_. https://docs.midtrans.com/docs/post-transaction-notification

---
