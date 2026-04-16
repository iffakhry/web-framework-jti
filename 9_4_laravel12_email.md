# BKPM – Workshop Sistem Informasi Web Framework

## Acara 32 — Laravel Email (Revisi Laravel 12)

**Pokok Bahasan:** Kirim Email dengan Laravel
**Versi Laravel:** 12.x  
**Versi PHP:** 8.3+

---

## d. Dasar Teori

Laravel Email adalah fitur dalam framework Laravel yang mempermudah pengiriman email dari aplikasi web PHP. Laravel menyediakan API yang bersih dan ekspresif yang didukung oleh **Symfony Mailer**, memungkinkan pengiriman email melalui berbagai layanan seperti SMTP, Resend, Mailgun, Postmark, Amazon SES, dan lainnya.

### Komponen Utama

- **Mailable Class** — Merepresentasikan satu jenis email (welcome email, invoice, notifikasi, dll).
- **Mail Facade** — Antarmuka untuk mengirim email dari controller atau service.
- **Markdown Mail** — Template email berbasis komponen Blade yang responsif.
- **Notifications** — Cara alternatif mengirim email yang lebih sederhana untuk notifikasi singkat.
- **Queue** — Mekanisme untuk mengirim email secara asinkron agar tidak memperlambat respons aplikasi.

### Perubahan Arsitektur di Laravel 11/12

> **`[DIUBAH]`** — Laravel 11 memperkenalkan struktur Mailable yang baru. Method `build()` yang digunakan di Laravel 9/10 **telah dihapus sepenuhnya**. Sebagai gantinya, setiap Mailable kini menggunakan tiga method terpisah: `envelope()`, `content()`, dan `attachments()`. Perubahan ini membuat struktur email lebih eksplisit, mudah dibaca, dan lebih mudah di-test.

```
Struktur Mailable Lama (Laravel ≤9):          Struktur Mailable Baru (Laravel 11/12):
─────────────────────────────────             ─────────────────────────────────────
build()                                       envelope()   → subject, from, replyTo
  ↳ subject()                                content()    → view, markdown, with
  ↳ view()                                   attachments() → file attachments
  ↳ attach()
```

---

## e. Alat dan Bahan

> **`[DIUBAH]`** — Spesifikasi sistem diperbarui menyesuaikan kebutuhan Laravel 12 dan PHP 8.3+. Windows 10 diganti menjadi Windows 11 karena Windows 10 telah memasuki fase End of Life (EOL) pada Oktober 2025.

1. PC / Laptop
2. **Windows 11** atau Ubuntu 22.04 LTS / macOS Ventura ke atas
3. RAM 4 GB (minimal) / 8 GB (rekomendasi)
4. SSD 256 GB ke atas
5. PHP **8.3** atau lebih baru
6. Composer 2.x
7. Node.js 20 LTS (untuk asset build)
8. Visual Studio Code
9. **Laravel Herd** (opsional, sangat direkomendasikan sebagai pengganti XAMPP)
10. Akun Mailtrap (gratis) — [https://mailtrap.io](https://mailtrap.io)

---

## f. Prosedur Kerja

---

### Langkah 0 — Persiapan Project Laravel 12

Buat project Laravel 12 baru dan install starter kit Breeze untuk scaffolding autentikasi.

```bash
# Buat project baru
composer create-project laravel/laravel laravel-email-demo

cd laravel-email-demo

# Verifikasi versi Laravel
php artisan --version
# Output: Laravel Framework 12.x.x

# Install Laravel Breeze (scaffolding autentikasi modern)
composer require laravel/breeze --dev

# Install Breeze dengan stack Blade
php artisan breeze:install blade

# Install dependensi Node dan build asset
npm install && npm run build

# Buat database SQLite dan jalankan migrasi
touch database/database.sqlite
php artisan migrate
```

> **`[DITAMBAH]`** — Laravel 12 tidak lagi menyertakan scaffolding autentikasi bawaan (`Auth::routes()`). Cara yang direkomendasikan industri adalah menggunakan **Laravel Breeze** yang menghasilkan kode autentikasi yang bersih, modern, dan sudah mencakup email verification secara bawaan.

---

### Langkah 1 — Konfigurasi Email di Laravel

#### a) Konfigurasi `.env`

Buka file `.env` dan sesuaikan konfigurasi email. Gunakan **Mailtrap** untuk keperluan development.

> **`[DIUBAH]`** — Host Mailtrap yang lama (`smtp.mailtrap.io`) sudah berubah. Host yang benar untuk sandbox Mailtrap saat ini adalah `sandbox.smtp.mailtrap.io` dengan port `587`.

```dotenv
MAIL_MAILER=smtp
MAIL_HOST=sandbox.smtp.mailtrap.io
MAIL_PORT=587
MAIL_USERNAME=<isi_dari_dashboard_mailtrap>
MAIL_PASSWORD=<isi_dari_dashboard_mailtrap>
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="noreply@example.com"
MAIL_FROM_NAME="${APP_NAME}"
```

**Cara mendapatkan kredensial Mailtrap:**

1. Daftar/login di [https://mailtrap.io](https://mailtrap.io)
2. Pilih menu **Email Testing → Inboxes**
3. Klik inbox yang tersedia, lalu pilih tab **SMTP Settings**
4. Salin `Username` dan `Password` ke file `.env`

#### b) Alternatif: Mail Driver `log` untuk Development Cepat

Jika tidak ingin menggunakan Mailtrap, gunakan driver `log`. Email tidak dikirim ke mana-mana, melainkan ditulis ke file `storage/logs/laravel.log`.

```dotenv
MAIL_MAILER=log
```

#### c) Alternatif Modern: Driver Resend

> **`[DITAMBAH]`** — Laravel 12 menambahkan dukungan bawaan untuk **Resend** ([resend.com](https://resend.com)), sebuah transactional email service modern yang populer di industri. Ini adalah pilihan yang sangat baik untuk produksi.

```dotenv
MAIL_MAILER=resend
RESEND_KEY=re_xxxxxxxxxxxx
```

```bash
# Install package Resend
composer require resend/resend-laravel
```

#### d) Testing Konfigurasi via Tinker

```bash
php artisan tinker
```

```php
Mail::raw('Test email dari Laravel 12!', function ($m) {
    $m->to('test@example.com')->subject('Test');
});
// Cek inbox Mailtrap atau file log jika berhasil
```

---

### Langkah 2 — Membuat dan Mengirim Email dengan Mailable

#### a) Membuat Mailable Class

```bash
php artisan make:mail WelcomeMail
```

File akan dibuat di `app/Mail/WelcomeMail.php`.

#### b) Mengisi `WelcomeMail.php`

> **`[DIUBAH — KRUSIAL]`** — Struktur Mailable **wajib** menggunakan `envelope()`, `content()`, dan `attachments()`. Method `build()` sudah **dihapus total** di Laravel 11/12 dan akan menyebabkan error jika masih digunakan. Ini adalah perubahan paling penting di modul ini.

```php
<?php

namespace App\Mail;

use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Content;
use Illuminate\Mail\Mailables\Envelope;
use Illuminate\Queue\SerializesModels;

class WelcomeMail extends Mailable
{
    use Queueable, SerializesModels;

    /**
     * Buat instance Mailable baru.
     */
    public function __construct(
        public readonly User $user,  // PHP 8.1+ constructor property promotion
    ) {}

    /**
     * Metadata email: subject, from, replyTo, cc, bcc.
     */
    public function envelope(): Envelope
    {
        return new Envelope(
            subject: 'Selamat Datang di ' . config('app.name') . '!',
        );
    }

    /**
     * Konten email: template view dan data yang diteruskan.
     */
    public function content(): Content
    {
        return new Content(
            view: 'emails.welcome',
        );
    }

    /**
     * Lampiran file (kosong jika tidak ada).
     */
    public function attachments(): array
    {
        return [];
    }
}
```

**Perbedaan pendekatan passing data:**

Karena `$user` dideklarasikan sebagai `public`, properti ini otomatis tersedia di view Blade tanpa perlu memanggil `with()`.

```php
// Opsi alternatif: menggunakan with() di content()
public function content(): Content
{
    return new Content(
        view: 'emails.welcome',
        with: [
            'userName' => $this->user->name,
            'loginUrl' => route('login'),
        ],
    );
}
```

#### c) Membuat View Email

Buat file `resources/views/emails/welcome.blade.php`:

```bash
mkdir -p resources/views/emails
```

```html
<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Selamat Datang</title>
    <style>
      body {
        font-family: Arial, sans-serif;
        background: #f4f4f4;
        padding: 20px;
      }
      .container {
        max-width: 600px;
        margin: auto;
        background: #fff;
        padding: 32px;
        border-radius: 8px;
      }
      .btn {
        display: inline-block;
        padding: 12px 24px;
        background: #4f46e5;
        color: #fff;
        text-decoration: none;
        border-radius: 6px;
        margin-top: 16px;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <h1>Selamat Datang, {{ $user->name }}!</h1>
      <p>
        Terima kasih telah mendaftar di
        <strong>{{ config('app.name') }}</strong>.
      </p>
      <p>
        Akun Anda telah berhasil dibuat dengan email:
        <strong>{{ $user->email }}</strong>
      </p>
      <a href="{{ route('login') }}" class="btn">Masuk ke Akun</a>
      <p style="margin-top: 24px; font-size: 12px; color: #666;">
        Jika Anda tidak mendaftar, abaikan email ini.
      </p>
    </div>
  </body>
</html>
```

#### d) Preview Email di Browser

> **`[DITAMBAH]`** — Fitur ini tidak ada di modul sebelumnya namun sangat berguna saat development. Tambahkan route sementara untuk melihat tampilan email langsung di browser tanpa perlu mengirimnya.

Tambahkan di `routes/web.php` (hapus sebelum ke production):

```php
// DEVELOPMENT ONLY — hapus sebelum deploy ke production
Route::get('/mail-preview', function () {
    $user = \App\Models\User::first() ?? new \App\Models\User([
        'name' => 'Budi Santoso',
        'email' => 'budi@example.com',
    ]);
    return new \App\Mail\WelcomeMail($user);
});
```

Akses `http://localhost/mail-preview` di browser untuk melihat pratinjau.

#### e) Mengirim Email dari Controller

```php
<?php

namespace App\Http\Controllers\Auth;

use App\Mail\WelcomeMail;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Mail;

class RegisterController extends Controller
{
    public function store(Request $request)
    {
        $validated = $request->validate([
            'name'     => ['required', 'string', 'max:255'],
            'email'    => ['required', 'email', 'unique:users'],
            'password' => ['required', 'min:8', 'confirmed'],
        ]);

        $user = User::create([
            'name'     => $validated['name'],
            'email'    => $validated['email'],
            'password' => bcrypt($validated['password']),
        ]);

        // Kirim email selamat datang
        Mail::to($user->email)->send(new WelcomeMail($user));

        return redirect()->route('dashboard')
                         ->with('success', 'Registrasi berhasil! Cek email Anda.');
    }
}
```

---

### Langkah 3 — Email dengan Template Markdown

Markdown Mail memungkinkan pembuatan email yang responsif dan konsisten menggunakan komponen Blade bawaan Laravel.

```bash
php artisan make:mail OrderConfirmation --markdown=emails.order-confirmation
```

> **`[DIUBAH]`** — Sama seperti Mailable biasa, struktur menggunakan `envelope()` dan `content()`, bukan `build()`.

```php
// app/Mail/OrderConfirmation.php
public function envelope(): Envelope
{
    return new Envelope(
        subject: 'Konfirmasi Pesanan #' . $this->order->id,
    );
}

public function content(): Content
{
    return new Content(
        markdown: 'emails.order-confirmation',  // gunakan 'markdown', bukan 'view'
        with: [
            'orderNumber' => $this->order->id,
            'totalAmount' => $this->order->total,
            'actionUrl'   => route('orders.show', $this->order),
        ],
    );
}
```

Edit view Markdown di `resources/views/emails/order-confirmation.blade.php`:

```blade
<x-mail::message>
# Konfirmasi Pesanan

Halo **{{ $user->name ?? 'Pelanggan' }}**,

Pesanan Anda dengan nomor **#{{ $orderNumber }}** telah kami terima.

**Total:** Rp {{ number_format($totalAmount, 0, ',', '.') }}

<x-mail::button :url="$actionUrl" color="primary">
Lihat Detail Pesanan
</x-mail::button>

Terima kasih telah berbelanja di {{ config('app.name') }}.

Salam,
Tim {{ config('app.name') }}
</x-mail::message>
```

**Publish komponen Markdown untuk kustomisasi:**

```bash
php artisan vendor:publish --tag=laravel-mail
# File ditempatkan di: resources/views/vendor/mail/
```

---

### Langkah 4 — Verifikasi Email

#### a) Setup Model User

Tambahkan implementasi `MustVerifyEmail` di model User:

```php
<?php

// app/Models/User.php
namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable implements MustVerifyEmail
{
    use Notifiable;

    // ...
}
```

#### b) Aktifkan Verifikasi di Breeze

> **`[DIUBAH]`** — Cara mengaktifkan verifikasi email berubah total di Laravel 11/12. `Auth::routes(['verify' => true])` sudah tidak ada. Dengan Breeze, cukup aktifkan `requireEmailVerification` saat instalasi, atau tambahkan middleware secara manual.

Jika Breeze sudah terinstall, pastikan routing dilindungi dengan middleware `verified`:

```php
// routes/web.php
use Illuminate\Support\Facades\Route;

// Route yang memerlukan email terverifikasi
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', function () {
        return view('dashboard');
    })->name('dashboard');
});
```

#### c) Kirim Ulang Email Verifikasi

```php
// routes/web.php
use Illuminate\Http\Request;

Route::post('/email/verification-notification', function (Request $request) {
    $request->user()->sendEmailVerificationNotification();

    return back()->with('status', 'verification-link-sent');

})->middleware(['auth', 'throttle:6,1'])->name('verification.send');
```

> **Catatan:** Jika menggunakan Breeze, route ini sudah dibuat secara otomatis di `routes/auth.php`.

---

### Langkah 5 — Mengirim Email via Queue (Asinkron)

Mengirim email secara langsung (synchronous) akan memperlambat respons aplikasi. Gunakan queue agar proses pengiriman berjalan di background.

#### a) Setup Queue Database

```bash
# Buat tabel queue
php artisan queue:table
php artisan migrate
```

Perbarui `.env`:

```dotenv
QUEUE_CONNECTION=database
```

#### b) Implementasi ShouldQueue di Mailable

```php
// app/Mail/WelcomeMail.php
use Illuminate\Contracts\Queue\ShouldQueue;

class WelcomeMail extends Mailable implements ShouldQueue
{
    use Queueable, SerializesModels;

    // Konfigurasi queue (opsional)
    public int $tries = 3;        // Jumlah percobaan ulang jika gagal
    public int $backoff = 60;     // Tunggu 60 detik antar percobaan

    // ... sisa kode sama
}
```

#### c) Mengirim ke Queue

```php
// Otomatis masuk queue jika Mailable implements ShouldQueue
Mail::to($user->email)->send(new WelcomeMail($user));

// Atau eksplisit: kirim ke queue
Mail::to($user->email)->queue(new WelcomeMail($user));

// Tunda pengiriman selama 5 menit
Mail::to($user->email)->later(now()->addMinutes(5), new WelcomeMail($user));
```

#### d) Menjalankan Queue Worker

```bash
# Jalankan worker (biarkan berjalan di terminal terpisah saat development)
php artisan queue:work

# Untuk production, gunakan supervisor atau systemd
# Tampilkan status queue
php artisan queue:monitor
```

---

### Langkah 6 — Mengirim ke Beberapa Penerima (CC, BCC, Reply-To)

> **`[DITAMBAH]`** — Fitur ini adalah kebutuhan umum di dunia kerja (misalnya: kirim invoice ke pelanggan dan CC ke tim keuangan). Ditambahkan ke modul untuk mempersiapkan mahasiswa menghadapi kebutuhan industri nyata.

```php
// Kirim ke beberapa penerima sekaligus
Mail::to($user->email)
    ->cc('manager@company.com')
    ->bcc('archive@company.com')
    ->send(new WelcomeMail($user));

// Atau, definisikan di dalam envelope() untuk konsistensi
use Illuminate\Mail\Mailables\Address;

public function envelope(): Envelope
{
    return new Envelope(
        from: new Address('noreply@app.com', config('app.name')),
        replyTo: [
            new Address('support@app.com', 'Tim Support'),
        ],
        cc: [
            new Address('admin@app.com', 'Admin'),
        ],
        subject: 'Selamat Datang!',
    );
}
```

---

### Langkah 7 — Melampirkan File (Attachment)

> **`[DITAMBAH]`** — Mengirim lampiran seperti PDF invoice atau dokumen adalah kebutuhan industri yang sangat umum.

```php
use Illuminate\Mail\Mailables\Attachment;

public function attachments(): array
{
    return [
        // Lampiran dari path file
        Attachment::fromPath(storage_path('app/invoices/invoice-001.pdf'))
                  ->as('Invoice-Januari-2026.pdf')
                  ->withMime('application/pdf'),

        // Lampiran dari storage disk
        Attachment::fromStorageDisk('s3', 'reports/monthly.pdf')
                  ->as('Laporan-Bulanan.pdf'),
    ];
}
```

---

### Langkah 8 — Notifikasi via Email

Laravel Notifications menyediakan cara yang lebih ringkas untuk email notifikasi sederhana.

```bash
php artisan make:notification InvoicePaid
```

```php
// app/Notifications/InvoicePaid.php
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification
{
    public function __construct(
        private readonly float $amount,
        private readonly string $invoiceUrl,
    ) {}

    public function via(object $notifiable): array
    {
        return ['mail'];
    }

    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('Invoice Telah Dibayar')
            ->greeting('Halo ' . $notifiable->name . '!')
            ->line('Invoice Anda sebesar Rp ' . number_format($this->amount, 0, ',', '.') . ' telah dibayar.')
            ->action('Lihat Invoice', $this->invoiceUrl)
            ->line('Terima kasih telah menggunakan layanan kami.');
    }
}
```

**Kirim notifikasi:**

```php
// Kirim ke satu user
$user->notify(new InvoicePaid(amount: 150000, invoiceUrl: route('invoices.show', $invoice)));

// Kirim ke banyak user sekaligus (menggunakan Notification facade)
use Illuminate\Support\Facades\Notification;
Notification::send($users, new InvoicePaid(150000, $invoiceUrl));
```

---

### Langkah 9 — Testing Email

> **`[DITAMBAH]`** — Testing email adalah praktik standar di industri yang tidak ada di modul sebelumnya. Dengan testing, mahasiswa dapat memverifikasi bahwa email terkirim dengan benar tanpa perlu mengirim email sungguhan.

Laravel 12 menggunakan **Pest PHP** sebagai test runner default.

```bash
# Buat test file
php artisan make:test MailTest
```

```php
<?php

// tests/Feature/MailTest.php
use App\Mail\WelcomeMail;
use App\Models\User;
use Illuminate\Support\Facades\Mail;

test('email selamat datang terkirim setelah registrasi', function () {
    // Intercept semua email (tidak benar-benar dikirim)
    Mail::fake();

    $user = User::factory()->create();

    // Trigger aksi yang mengirim email
    Mail::to($user->email)->send(new WelcomeMail($user));

    // Verifikasi bahwa email terkirim
    Mail::assertSent(WelcomeMail::class, function (WelcomeMail $mail) use ($user) {
        return $mail->hasTo($user->email);
    });
});

test('email selamat datang tidak terkirim ke alamat yang salah', function () {
    Mail::fake();

    $user = User::factory()->create();
    Mail::to($user->email)->send(new WelcomeMail($user));

    // Pastikan tidak dikirim ke alamat lain
    Mail::assertNotSent(WelcomeMail::class, function (WelcomeMail $mail) {
        return $mail->hasTo('orang-lain@example.com');
    });
});

test('envelope berisi subject yang benar', function () {
    Mail::fake();

    $user = User::factory()->create();
    Mail::to($user->email)->send(new WelcomeMail($user));

    Mail::assertSent(WelcomeMail::class, function (WelcomeMail $mail) {
        return $mail->hasSubject('Selamat Datang di ' . config('app.name') . '!');
    });
});
```

**Jalankan test:**

```bash
php artisan test
# atau
./vendor/bin/pest
```

---

### Langkah 10 — Testing Email di Localhost

> **`[DIUBAH]`** — Mailhog dihapus dari daftar rekomendasi karena sudah tidak aktif dikembangkan sejak 2020. Penggantinya adalah **Mailpit** yang aktif dikembangkan, lebih cepat, dan terintegrasi langsung dengan Laravel Herd.

**Opsi 1: Mailtrap (direkomendasikan untuk semua OS)**

- Akses [https://mailtrap.io](https://mailtrap.io)
- Gratis untuk development, tidak perlu install apapun
- Email ditangkap di inbox online

**Opsi 2: Mailpit (pengganti Mailhog)**

> **`[DITAMBAH]`** — Mailpit adalah suksesor Mailhog yang aktif dikembangkan.

```bash
# Install via Laravel Herd (otomatis tersedia)
# Atau install manual:

# macOS
brew install axllent/apps/mailpit

# Windows (via Scoop)
scoop install mailpit

# Linux
curl -sL https://raw.githubusercontent.com/axllent/mailpit/develop/install.sh | bash
```

Konfigurasi `.env` untuk Mailpit:

```dotenv
MAIL_MAILER=smtp
MAIL_HOST=127.0.0.1
MAIL_PORT=1025
MAIL_ENCRYPTION=null
```

Akses UI Mailpit di `http://localhost:8025`

**Opsi 3: Driver Log (paling simpel)**

```dotenv
MAIL_MAILER=log
```

Cek email di `storage/logs/laravel.log`:

```bash
tail -f storage/logs/laravel.log | grep -A 20 "Message-Id"
```

---

## g. Ringkasan Perubahan dari Modul Sebelumnya

| No  | Item                                     | Status          | Alasan                                                                            |
| --- | ---------------------------------------- | --------------- | --------------------------------------------------------------------------------- |
| 1   | Method `build()` di Mailable             | **Dihapus**     | Removed di Laravel 11/12. Ganti dengan `envelope()`, `content()`, `attachments()` |
| 2   | `Auth::routes(['verify' => true])`       | **Dihapus**     | Tidak ada di Laravel 11/12. Gunakan Breeze + middleware `verified`                |
| 3   | Mailhog sebagai testing tool             | **Dihapus**     | Tidak aktif dikembangkan sejak 2020. Ganti dengan Mailpit                         |
| 4   | Windows 10 di alat dan bahan             | **Diperbarui**  | EOL Oktober 2025. Ganti dengan Windows 11                                         |
| 5   | Host Mailtrap (`smtp.mailtrap.io`)       | **Diperbarui**  | Host berubah menjadi `sandbox.smtp.mailtrap.io` port 587                          |
| 6   | Struktur Mailable dengan public property | **Diperbarui**  | Tetap bisa digunakan, namun sebaiknya gunakan constructor promotion PHP 8.1+      |
| 7   | Laravel Breeze untuk verifikasi email    | **Ditambahkan** | Standar industri modern, lebih clean dari scaffolding manual                      |
| 8   | Driver Resend                            | **Ditambahkan** | Driver bawaan baru di Laravel 12, populer di industri 2026                        |
| 9   | Preview email di browser                 | **Ditambahkan** | Sangat membantu proses development dan praktikum                                  |
| 10  | CC, BCC, Reply-To via `Envelope`         | **Ditambahkan** | Kebutuhan umum di dunia kerja nyata                                               |
| 11  | Attachment file                          | **Ditambahkan** | Kebutuhan industri yang sangat umum (invoice, laporan)                            |
| 12  | Testing dengan `Mail::fake()`            | **Ditambahkan** | Standar profesional yang wajib diketahui di industri                              |
| 13  | Mailpit sebagai local testing            | **Ditambahkan** | Pengganti aktif untuk Mailhog                                                     |

---

## h. Hasil dan Pembahasan

### Format Laporan

- Kertas A4
- Format `*.pdf`
- Struktur Laporan: Cover, Pendahuluan, Hasil Praktik, Kesimpulan, dan Daftar Pustaka
- Berikan Identitas Diri: NIM, Nama, Golongan, Tugas Minggu Ke-X
- Penamaan File: `ACARA-32_GOL_NIM_NAMA.pdf`
  - **Contoh:** `ACARA-32_A_E41234567_BUDI.pdf`
- Kumpulkan pada [https://elearning-jti.polije.ac.id/](https://elearning-jti.polije.ac.id/)

### Checklist Hasil Praktik yang Harus Didokumentasikan

- [ ] Screenshot konfigurasi `.env` (sensor username/password)
- [ ] Screenshot kode `WelcomeMail.php` dengan struktur baru
- [ ] Screenshot email masuk di Mailtrap/Mailpit
- [ ] Screenshot preview email di browser (route `/mail-preview`)
- [ ] Screenshot output `php artisan test` yang sukses

---

## i. Kesimpulan

Dengan menyelesaikan praktikum ini, mahasiswa telah mempelajari cara kerja sistem email di Laravel 12 menggunakan API terbaru. Perbedaan utama dari versi sebelumnya adalah penggunaan `envelope()` dan `content()` menggantikan `build()`, penggunaan Breeze sebagai scaffolding autentikasi modern, serta penerapan `Mail::fake()` untuk testing — semua ini mencerminkan praktik terbaik yang digunakan di industri pengembangan web profesional saat ini.

---

## j. Rubrik Penilaian

| No        | Indikator Penilaian                                                         | Nilai   |
| --------- | --------------------------------------------------------------------------- | ------- |
| 1         | Konfigurasi `.env` benar dan email berhasil masuk ke Mailtrap/Mailpit       | 25      |
| 2         | Mailable class menggunakan struktur `envelope()` dan `content()` yang benar | 25      |
| 3         | Verifikasi email terimplementasi dengan middleware `verified`               | 25      |
| 4         | Test email menggunakan `Mail::fake()` berhasil dijalankan                   | 25      |
| **Total** |                                                                             | **100** |

---

## k. Referensi

- [Laravel 12 Documentation — Mail](https://laravel.com/docs/12.x/mail)
- [Laravel 12 Documentation — Email Verification](https://laravel.com/docs/12.x/verification)
- [Laravel 12 Documentation — Notifications](https://laravel.com/docs/12.x/notifications)
- [Laravel Breeze Documentation](https://laravel.com/docs/12.x/starter-kits#laravel-breeze)
- [Mailtrap — Email Testing for Development](https://mailtrap.io)
- [Mailpit — Local Email Testing Tool](https://mailpit.axllent.org)
- [Resend — Transactional Email for Developers](https://resend.com)
- [Pest PHP — Elegant Testing for PHP](https://pestphp.com)

---
