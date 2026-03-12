# Tutorial Laravel 12 Authentication

### Middleware, Role-Based Access Control & Best Practices Industri

> **Mata Kuliah:** Rekayasa Perangkat Lunak  
> **Level:** Menengah (Semester 4)  
> **Durasi Estimasi:** 3–4 jam praktikum  
> **Prasyarat:** PHP dasar, MVC, Composer, database MySQL

---

## Daftar Isi

1. [Pendahuluan & Konsep](#1-pendahuluan--konsep)
2. [Persiapan Lingkungan](#2-persiapan-lingkungan)
3. [Instalasi Laravel 12 + Starter Kit](#3-instalasi-laravel-12--starter-kit)
4. [Struktur Authentication Bawaan](#4-struktur-authentication-bawaan)
5. [Migrasi Database & User Model](#5-migrasi-database--user-model)
6. [Implementasi Role-Based Access Control (RBAC)](#6-implementasi-role-based-access-control-rbac)
7. [Middleware Custom](#7-middleware-custom)
8. [Gate & Policy](#8-gate--policy)
9. [Proteksi Route](#9-proteksi-route)
10. [Demo Alur Lengkap](#10-demo-alur-lengkap)
11. [Pengujian Manual](#11-pengujian-manual)
12. [Best Practices Industri](#12-best-practices-industri)
13. [Referensi](#13-referensi)

---

## 1. Pendahuluan & Konsep

### Apa itu Authentication vs Authorization?

| Konsep             | Pertanyaan                     | Contoh                                       |
| ------------------ | ------------------------------ | -------------------------------------------- |
| **Authentication** | _Siapa kamu?_                  | Login dengan email & password                |
| **Authorization**  | _Apa yang boleh kamu lakukan?_ | Admin bisa hapus data, user biasa tidak bisa |

### Komponen Utama Auth di Laravel 12

```
Guards      → Menentukan BAGAIMANA user diautentikasi (session, token)
Providers   → Menentukan DARI MANA user diambil (Eloquent, database)
Middleware  → Lapisan penjaga sebelum request masuk ke controller
Gates       → Otorisasi berbasis closure (sederhana)
Policies    → Otorisasi berbasis class (kompleks, per-model)
```

### Perubahan Penting di Laravel 12

- **Laravel Breeze & Jetstream** mulai deprecated → digantikan **Starter Kit baru** (React, Vue, Livewire)
- Starter kit baru sudah include **WorkOS AuthKit** (opsional) untuk social login & passkeys
- Membutuhkan **PHP 8.2 atau lebih tinggi**
- Struktur direktori lebih rapi berbasis domain-driven design

---

## 2. Persiapan Lingkungan

### Kebutuhan Sistem

```bash
# Cek versi PHP (minimal 8.2)
php -v

# Cek Composer
composer --version

# Cek Node.js & npm (untuk frontend assets)
node -v
npm -v
```

### Stack yang Digunakan dalam Tutorial Ini

```
Backend  : Laravel 12 (PHP 8.2+)
Auth Kit : Laravel Breeze (Blade) — tetap tersedia di Laravel 12
Database : MySQL / SQLite
Frontend : Blade + Tailwind CSS
RBAC     : Custom role column + Middleware + Gates
```

---

## 3. Instalasi Laravel 12 + Starter Kit

### Langkah 1 — Buat Project Baru

```bash
# Menggunakan Laravel Installer
composer create-project laravel/laravel auth-demo

# Masuk ke direktori project
cd auth-demo
```

### Langkah 2 — Install Laravel Breeze

```bash
# Install Breeze sebagai dev dependency
composer require laravel/breeze --dev

# Install scaffolding Blade (paling sederhana untuk belajar)
php artisan breeze:install blade

# Install dependencies frontend
npm install && npm run build
```

> **Catatan:** Saat ditanya pilihan stack, pilih `blade` untuk tutorial ini.

### Langkah 3 — Konfigurasi Database

Edit file `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=auth_demo
DB_USERNAME=root
DB_PASSWORD=
```

Buat database di MySQL:

```sql
CREATE DATABASE auth_demo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Langkah 4 — Jalankan Migrasi

```bash
php artisan migrate
```

### Langkah 5 — Jalankan Server

```bash
php artisan serve
```

Buka browser: `http://127.0.0.1:8000` — tampilan login/register sudah tersedia.

---

## 4. Struktur Authentication Bawaan

Setelah instalasi Breeze, Laravel membuat file-file berikut secara otomatis:

### Routes

```
routes/auth.php
├── GET  /register      → RegisteredUserController@create
├── POST /register      → RegisteredUserController@store
├── GET  /login         → AuthenticatedSessionController@create
├── POST /login         → AuthenticatedSessionController@store
├── POST /logout        → AuthenticatedSessionController@destroy
├── GET  /forgot-password
├── POST /forgot-password
├── GET  /reset-password/{token}
└── POST /reset-password
```

### Controllers (app/Http/Controllers/Auth/)

```
AuthenticatedSessionController.php  → Proses login & logout
RegisteredUserController.php        → Proses registrasi
PasswordResetLinkController.php     → Kirim link reset password
NewPasswordController.php           → Ganti password baru
EmailVerificationController.php     → Verifikasi email
```

### Cara Kerja Auth (Diagram Alur)

```
Browser Request
      │
      ▼
  Middleware (auth)
      │
   ┌──┴──────────────┐
   │ Sudah login?    │
   └──┬──────────────┘
      │ Ya                Tidak
      │                     │
      ▼                     ▼
  Controller          Redirect ke /login
      │
      ▼
  Response / View
```

---

## 5. Migrasi Database & User Model

### Tambah Kolom Role ke Tabel Users

Buat migration baru:

```bash
php artisan make:migration add_role_to_users_table --table=users
```

Edit file migrasi yang baru dibuat di `database/migrations/`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->enum('role', ['admin', 'editor', 'user'])
                  ->default('user')
                  ->after('email');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('role');
        });
    }
};
```

Jalankan migrasi:

```bash
php artisan migrate
```

### Update User Model

Edit `app/Models/User.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use HasFactory, Notifiable;

    /**
     * Kolom yang boleh diisi massal (mass assignment)
     */
    protected $fillable = [
        'name',
        'email',
        'password',
        'role',          // ← Tambahkan ini
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password'          => 'hashed',
        ];
    }

    // ─── Helper Methods ────────────────────────────────────────────

    /**
     * Cek apakah user adalah admin
     */
    public function isAdmin(): bool
    {
        return $this->role === 'admin';
    }

    /**
     * Cek apakah user adalah editor
     */
    public function isEditor(): bool
    {
        return $this->role === 'editor';
    }

    /**
     * Cek apakah user memiliki salah satu role
     */
    public function hasRole(string|array $roles): bool
    {
        if (is_string($roles)) {
            return $this->role === $roles;
        }

        return in_array($this->role, $roles);
    }
}
```

### Buat Seeder untuk Data Dummy

```bash
php artisan make:seeder UserSeeder
```

Edit `database/seeders/UserSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        // Admin
        User::create([
            'name'     => 'Admin User',
            'email'    => 'admin@demo.com',
            'password' => Hash::make('password'),
            'role'     => 'admin',
        ]);

        // Editor
        User::create([
            'name'     => 'Editor User',
            'email'    => 'editor@demo.com',
            'password' => Hash::make('password'),
            'role'     => 'editor',
        ]);

        // User Biasa
        User::create([
            'name'     => 'Regular User',
            'email'    => 'user@demo.com',
            'password' => Hash::make('password'),
            'role'     => 'user',
        ]);
    }
}
```

Edit `database/seeders/DatabaseSeeder.php`:

```php
public function run(): void
{
    $this->call([
        UserSeeder::class,
    ]);
}
```

Jalankan seeder:

```bash
php artisan db:seed
```

---

## 6. Implementasi Role-Based Access Control (RBAC)

RBAC adalah pola di mana hak akses diberikan berdasarkan **peran (role)** pengguna, bukan kepada individu secara langsung.

### Struktur Role dalam Tutorial Ini

```
admin   → Akses penuh: CRUD semua data, kelola user
editor  → Akses terbatas: buat & edit konten, tidak bisa hapus user
user    → Akses standar: hanya lihat konten, kelola profil sendiri
```

---

## 7. Middleware Custom

Middleware adalah kode yang berjalan **sebelum** request sampai ke controller.

### Buat Middleware Role

```bash
php artisan make:middleware RoleMiddleware
```

Edit `app/Http/Middleware/RoleMiddleware.php`:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class RoleMiddleware
{
    /**
     * Periksa apakah user memiliki role yang dibutuhkan
     *
     * @param  string|string[]  $roles  Role yang diizinkan (pisahkan dengan koma atau array)
     */
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        // Pastikan user sudah login
        if (! $request->user()) {
            return redirect()->route('login');
        }

        // Cek apakah role user ada di daftar role yang diizinkan
        if (! in_array($request->user()->role, $roles)) {
            abort(403, 'Anda tidak memiliki izin untuk mengakses halaman ini.');
        }

        return $next($request);
    }
}
```

### Daftarkan Middleware di Bootstrap

Di Laravel 12, middleware didaftarkan di `bootstrap/app.php`:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {

        // ← Tambahkan alias middleware di sini
        $middleware->alias([
            'role' => \App\Http\Middleware\RoleMiddleware::class,
        ]);

    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

---

## 8. Gate & Policy

### Gate — Otorisasi Sederhana Berbasis Closure

Gates didefinisikan di `app/Providers/AppServiceProvider.php`:

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;
use App\Models\User;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Gate: hanya admin yang bisa mengakses panel admin
        Gate::define('access-admin-panel', function (User $user): bool {
            return $user->isAdmin();
        });

        // Gate: admin & editor bisa mengelola konten
        Gate::define('manage-content', function (User $user): bool {
            return $user->hasRole(['admin', 'editor']);
        });

        // Gate: hanya admin yang bisa mengelola user lain
        Gate::define('manage-users', function (User $user): bool {
            return $user->isAdmin();
        });

        // Super gate: admin bisa melakukan SEMUA aksi
        Gate::before(function (User $user, string $ability): ?bool {
            if ($user->isAdmin()) {
                return true; // Admin bypass semua gate
            }
            return null; // Lanjut ke gate spesifik
        });
    }
}
```

### Cara Menggunakan Gate di Controller

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;

class AdminController extends Controller
{
    public function index()
    {
        // Cara 1: abort jika tidak punya akses
        Gate::authorize('access-admin-panel');

        return view('admin.dashboard');
    }

    public function manageUsers()
    {
        // Cara 2: cek manual
        if (Gate::denies('manage-users')) {
            abort(403, 'Hanya admin yang dapat mengelola pengguna.');
        }

        $users = \App\Models\User::all();
        return view('admin.users', compact('users'));
    }
}
```

### Cara Menggunakan Gate di Blade View

```blade
{{-- Tampilkan menu hanya untuk admin --}}
@can('access-admin-panel')
    <a href="{{ route('admin.dashboard') }}" class="nav-link">
        🔧 Admin Panel
    </a>
@endcan

{{-- Tampilkan tombol edit hanya untuk admin & editor --}}
@can('manage-content')
    <a href="{{ route('posts.edit', $post) }}" class="btn btn-sm btn-warning">
        Edit
    </a>
@endcan

{{-- Tampilkan konten berbeda berdasarkan role --}}
@if(auth()->user()->isAdmin())
    <span class="badge bg-danger">Admin</span>
@elseif(auth()->user()->isEditor())
    <span class="badge bg-warning">Editor</span>
@else
    <span class="badge bg-secondary">User</span>
@endif
```

### Policy — Otorisasi Berbasis Model

Policy cocok untuk otorisasi yang berkaitan dengan model tertentu (misalnya: hanya pemilik post yang bisa mengedit).

```bash
php artisan make:policy PostPolicy --model=Post
```

Edit `app/Policies/PostPolicy.php`:

```php
<?php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    /**
     * Admin bisa melakukan semua aksi
     */
    public function before(User $user, string $ability): bool|null
    {
        if ($user->isAdmin()) {
            return true; // Admin bypass semua cek policy
        }
        return null;
    }

    /**
     * Siapa saja yang sudah login bisa melihat post
     */
    public function view(User $user, Post $post): bool
    {
        return true;
    }

    /**
     * Hanya admin & editor yang bisa membuat post
     */
    public function create(User $user): bool
    {
        return $user->hasRole(['admin', 'editor']);
    }

    /**
     * Hanya pemilik post atau admin yang bisa mengedit
     */
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->hasRole(['admin', 'editor']);
    }

    /**
     * Hanya pemilik post atau admin yang bisa menghapus
     */
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->isAdmin();
    }
}
```

---

## 9. Proteksi Route

### routes/web.php — Contoh Lengkap

```php
<?php

use App\Http\Controllers\ProfileController;
use App\Http\Controllers\DashboardController;
use App\Http\Controllers\AdminController;
use App\Http\Controllers\PostController;
use Illuminate\Support\Facades\Route;

// ─── Route Publik ────────────────────────────────────────────────
Route::get('/', function () {
    return view('welcome');
});

// ─── Route untuk User yang Sudah Login ──────────────────────────
Route::middleware(['auth', 'verified'])->group(function () {

    // Dashboard umum — semua user yang sudah login
    Route::get('/dashboard', [DashboardController::class, 'index'])
         ->name('dashboard');

    // Profile — semua user yang sudah login
    Route::get('/profile', [ProfileController::class, 'edit'])->name('profile.edit');
    Route::patch('/profile', [ProfileController::class, 'update'])->name('profile.update');
    Route::delete('/profile', [ProfileController::class, 'destroy'])->name('profile.destroy');

    // ─── Posts — Admin & Editor ──────────────────────────────────
    Route::middleware('role:admin,editor')->group(function () {
        Route::resource('posts', PostController::class)
             ->except(['index', 'show']); // index & show tetap publik
    });

    // ─── Admin Panel — Hanya Admin ───────────────────────────────
    Route::middleware('role:admin')->prefix('admin')->name('admin.')->group(function () {

        Route::get('/dashboard', [AdminController::class, 'index'])
             ->name('dashboard');

        Route::get('/users', [AdminController::class, 'users'])
             ->name('users');

        Route::patch('/users/{user}/role', [AdminController::class, 'updateRole'])
             ->name('users.update-role');

        Route::delete('/users/{user}', [AdminController::class, 'destroyUser'])
             ->name('users.destroy');
    });
});

// ─── Route Posts Publik ─────────────────────────────────────────
Route::get('/posts', [PostController::class, 'index'])->name('posts.index');
Route::get('/posts/{post}', [PostController::class, 'show'])->name('posts.show');

// ─── Auth Routes (login, register, logout, dst.) ────────────────
require __DIR__.'/auth.php';
```

---

## 10. Demo Alur Lengkap

### Buat Controller untuk Demo

```bash
php artisan make:controller AdminController
php artisan make:controller DashboardController
```

**`app/Http/Controllers/DashboardController.php`:**

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class DashboardController extends Controller
{
    public function index(Request $request)
    {
        $user = $request->user();

        // Redirect ke halaman sesuai role
        return match ($user->role) {
            'admin'  => redirect()->route('admin.dashboard'),
            default  => view('dashboard'),
        };
    }
}
```

**`app/Http/Controllers/AdminController.php`:**

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;

class AdminController extends Controller
{
    public function index()
    {
        Gate::authorize('access-admin-panel');

        $stats = [
            'total_users'   => User::count(),
            'total_admins'  => User::where('role', 'admin')->count(),
            'total_editors' => User::where('role', 'editor')->count(),
        ];

        return view('admin.dashboard', compact('stats'));
    }

    public function users()
    {
        Gate::authorize('manage-users');

        $users = User::orderBy('role')->orderBy('name')->get();
        return view('admin.users', compact('users'));
    }

    public function updateRole(Request $request, User $user)
    {
        Gate::authorize('manage-users');

        $request->validate([
            'role' => ['required', 'in:admin,editor,user'],
        ]);

        // Jangan ubah role diri sendiri
        if ($user->id === auth()->id()) {
            return back()->with('error', 'Anda tidak dapat mengubah role diri sendiri.');
        }

        $user->update(['role' => $request->role]);

        return back()->with('success', "Role {$user->name} berhasil diubah menjadi {$request->role}.");
    }

    public function destroyUser(User $user)
    {
        Gate::authorize('manage-users');

        if ($user->id === auth()->id()) {
            return back()->with('error', 'Anda tidak dapat menghapus akun sendiri.');
        }

        $user->delete();
        return back()->with('success', "Akun {$user->name} berhasil dihapus.");
    }
}
```

### Buat Views

**`resources/views/admin/dashboard.blade.php`:**

```blade
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            🔧 Admin Dashboard
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">

            {{-- Alert Success/Error --}}
            @if(session('success'))
                <div class="mb-4 p-4 bg-green-100 text-green-700 rounded">
                    {{ session('success') }}
                </div>
            @endif

            {{-- Stats Cards --}}
            <div class="grid grid-cols-3 gap-6 mb-8">
                <div class="bg-white p-6 rounded-lg shadow text-center">
                    <p class="text-4xl font-bold text-blue-600">{{ $stats['total_users'] }}</p>
                    <p class="text-gray-500 mt-1">Total Pengguna</p>
                </div>
                <div class="bg-white p-6 rounded-lg shadow text-center">
                    <p class="text-4xl font-bold text-red-600">{{ $stats['total_admins'] }}</p>
                    <p class="text-gray-500 mt-1">Admin</p>
                </div>
                <div class="bg-white p-6 rounded-lg shadow text-center">
                    <p class="text-4xl font-bold text-yellow-600">{{ $stats['total_editors'] }}</p>
                    <p class="text-gray-500 mt-1">Editor</p>
                </div>
            </div>

            {{-- Quick Links --}}
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg p-6">
                <h3 class="font-bold text-lg mb-4">Menu Admin</h3>
                <div class="flex gap-4">
                    <a href="{{ route('admin.users') }}"
                       class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600">
                        👥 Kelola Pengguna
                    </a>
                    <a href="{{ route('posts.index') }}"
                       class="bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600">
                        📝 Kelola Konten
                    </a>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

**`resources/views/admin/users.blade.php`:**

```blade
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            👥 Manajemen Pengguna
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg p-6">

                @if(session('success'))
                    <div class="mb-4 p-4 bg-green-100 text-green-700 rounded">
                        {{ session('success') }}
                    </div>
                @endif

                <table class="w-full text-left border-collapse">
                    <thead>
                        <tr class="border-b bg-gray-50">
                            <th class="py-3 px-4">#</th>
                            <th class="py-3 px-4">Nama</th>
                            <th class="py-3 px-4">Email</th>
                            <th class="py-3 px-4">Role</th>
                            <th class="py-3 px-4">Aksi</th>
                        </tr>
                    </thead>
                    <tbody>
                        @foreach($users as $user)
                        <tr class="border-b hover:bg-gray-50">
                            <td class="py-3 px-4">{{ $loop->iteration }}</td>
                            <td class="py-3 px-4">{{ $user->name }}</td>
                            <td class="py-3 px-4">{{ $user->email }}</td>
                            <td class="py-3 px-4">
                                <span class="px-2 py-1 rounded text-sm font-medium
                                    @if($user->role === 'admin') bg-red-100 text-red-700
                                    @elseif($user->role === 'editor') bg-yellow-100 text-yellow-700
                                    @else bg-gray-100 text-gray-700 @endif">
                                    {{ ucfirst($user->role) }}
                                </span>
                            </td>
                            <td class="py-3 px-4">
                                @if($user->id !== auth()->id())
                                    {{-- Form Ganti Role --}}
                                    <form action="{{ route('admin.users.update-role', $user) }}"
                                          method="POST" class="inline-flex items-center gap-2">
                                        @csrf
                                        @method('PATCH')
                                        <select name="role" class="text-sm border rounded px-2 py-1">
                                            <option value="user"   @selected($user->role === 'user')>User</option>
                                            <option value="editor" @selected($user->role === 'editor')>Editor</option>
                                            <option value="admin"  @selected($user->role === 'admin')>Admin</option>
                                        </select>
                                        <button type="submit" class="bg-blue-500 text-white px-2 py-1 rounded text-sm">
                                            Simpan
                                        </button>
                                    </form>

                                    {{-- Tombol Hapus --}}
                                    <form action="{{ route('admin.users.destroy', $user) }}"
                                          method="POST" class="inline"
                                          onsubmit="return confirm('Hapus pengguna ini?')">
                                        @csrf
                                        @method('DELETE')
                                        <button type="submit" class="bg-red-500 text-white px-2 py-1 rounded text-sm ml-2">
                                            Hapus
                                        </button>
                                    </form>
                                @else
                                    <span class="text-gray-400 text-sm italic">Akun Anda</span>
                                @endif
                            </td>
                        </tr>
                        @endforeach
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</x-app-layout>
```

---

## 11. Pengujian Manual

### Skenario Uji Coba

Setelah menjalankan seeder, coba login dengan akun berikut:

| Email             | Password   | Role   | Akses                          |
| ----------------- | ---------- | ------ | ------------------------------ |
| `admin@demo.com`  | `password` | admin  | Semua halaman                  |
| `editor@demo.com` | `password` | editor | Dashboard, Posts (create/edit) |
| `user@demo.com`   | `password` | user   | Dashboard, Profil saja         |

### Checklist Pengujian

```
□ Login sebagai admin → bisa akses /admin/dashboard
□ Login sebagai editor → /admin/dashboard menampilkan 403
□ Login sebagai user → /admin/dashboard menampilkan 403
□ Login sebagai editor → bisa akses /posts/create
□ Login sebagai user → /posts/create menampilkan 403
□ Admin bisa ubah role pengguna lain
□ Admin tidak bisa ubah role dirinya sendiri
□ Logout → redirect ke halaman login
□ Akses /dashboard tanpa login → redirect ke /login
```

### Cek Auth di Tinker (REPL Laravel)

```bash
php artisan tinker
```

```php
// Simulasi login
$user = \App\Models\User::where('email', 'admin@demo.com')->first();
auth()->login($user);

// Cek gate
\Illuminate\Support\Facades\Gate::allows('access-admin-panel'); // true
\Illuminate\Support\Facades\Gate::allows('manage-users');       // true

// Cek helper
auth()->user()->isAdmin();    // true
auth()->user()->isEditor();   // false
auth()->user()->hasRole(['admin', 'editor']); // true
```

---

## 12. Best Practices Industri

### ✅ DO — Yang Harus Dilakukan

```
1. SELALU gunakan middleware untuk proteksi route, bukan hanya cek di controller
2. Gunakan Gate::authorize() bukan abort(403) manual — lebih ekspresif
3. Hash password dengan bcrypt (sudah otomatis di Laravel via 'hashed' cast)
4. Gunakan $table->enum() untuk kolom role — mencegah nilai tidak valid masuk database
5. Validasi input di setiap form — terutama saat ganti role
6. Beri pesan error yang ramah pengguna, bukan stack trace mentah
7. Pisahkan logika otorisasi ke Gates/Policies — jangan campur di view
8. Gunakan named routes — mudah maintenance dan refactor
```

### ❌ DON'T — Yang Harus Dihindari

```
1. JANGAN simpan role sebagai angka tanpa label (0, 1, 2) — susah dibaca
2. JANGAN cek role hanya di frontend/Blade — mudah di-bypass
3. JANGAN hardcode email admin di config — gunakan database
4. JANGAN expose pesan error detail ke pengguna umum di production
5. JANGAN lupa logout invalidasi session di server (bukan hanya hapus cookie)
6. JANGAN izinkan mass assignment tanpa $fillable atau $guarded
```

### Keamanan Tambahan yang Umum di Industri

```php
// 1. Rate limiting pada login (sudah ada di Breeze, tapi pastikan aktif)
// routes/auth.php — Breeze sudah menerapkan ThrottleRequests

// 2. Password strength validation (Laravel 12 - secureValidate)
$request->validate([
    'password' => ['required', 'confirmed', \Illuminate\Validation\Rules\Password::defaults()],
]);

// 3. Regenerate session setelah login (cegah session fixation)
// Sudah otomatis di AuthenticatedSessionController bawaan Breeze

// 4. Logout dari semua device
Auth::logoutOtherDevices($request->password);
```

---

## 13. Referensi

| Sumber                                        | URL                                          |
| --------------------------------------------- | -------------------------------------------- |
| Dokumentasi Resmi Laravel 12 — Authentication | https://laravel.com/docs/12.x/authentication |
| Dokumentasi Resmi Laravel 12 — Authorization  | https://laravel.com/docs/12.x/authorization  |
| Dokumentasi Resmi Laravel 12 — Middleware     | https://laravel.com/docs/12.x/middleware     |
| Laravel 12 Release Notes                      | https://laravel.com/docs/12.x/releases       |
| Laravel Breeze GitHub                         | https://github.com/laravel/breeze            |

---

## Ringkasan Konsep

```
Authentication  = Verifikasi identitas (login)
Authorization   = Verifikasi hak akses (role & gate)
Guard           = Mekanisme autentikasi (session/token)
Middleware      = Penjaga akses sebelum masuk controller
Gate            = Otorisasi closure sederhana
Policy          = Otorisasi class berbasis model
RBAC            = Role-Based Access Control — hak akses via peran
```

---

_Tutorial ini dibuat untuk keperluan pembelajaran praktikum di Politeknik Negeri Jember._  
_Laravel versi: 12.x | PHP: 8.2+ | Terakhir diperbarui: Maret 2026_
