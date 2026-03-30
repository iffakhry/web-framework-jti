# Tutorial Laravel 12 API: Sanctum Authentication & Authorization dengan Bearer Token

**Level:** Intermediate  
**Versi Laravel:** 12.x  
**Topik:** RESTful API, Laravel Sanctum, Bearer Token, CRUD User & Books

---

## Daftar Isi

1. [Pendahuluan & Konsep Dasar](#1-pendahuluan--konsep-dasar)
2. [Setup Project Laravel 12](#2-setup-project-laravel-12)
3. [Instalasi & Konfigurasi Sanctum](#3-instalasi--konfigurasi-sanctum)
4. [Database: Migration & Seeder](#4-database-migration--seeder)
5. [Model](#5-model)
6. [Auth API: Register, Login, Logout](#6-auth-api-register-login-logout)
7. [CRUD Books dengan Authorization](#7-crud-books-dengan-authorization)
8. [CRUD User (Admin Only)](#8-crud-user-admin-only)
9. [API Routes & Middleware](#9-api-routes--middleware)
10. [API Resource (Response Formatting)](#10-api-resource-response-formatting)
11. [Testing dengan Postman / cURL](#11-testing-dengan-postman--curl)
12. [Error Handling Global](#12-error-handling-global)
13. [Ringkasan Arsitektur](#13-ringkasan-arsitektur)

---

## 1. Pendahuluan & Konsep Dasar

### Apa itu REST API?

REST (Representational State Transfer) API adalah antarmuka yang memungkinkan komunikasi antar sistem menggunakan protokol HTTP. Setiap _resource_ (data) diakses melalui URL dengan metode HTTP yang semantik:

| Method | Endpoint          | Fungsi           |
| ------ | ----------------- | ---------------- |
| GET    | `/api/books`      | Ambil semua buku |
| GET    | `/api/books/{id}` | Ambil satu buku  |
| POST   | `/api/books`      | Buat buku baru   |
| PUT    | `/api/books/{id}` | Update buku      |
| DELETE | `/api/books/{id}` | Hapus buku       |

### Apa itu Bearer Token?

Bearer Token adalah mekanisme autentikasi berbasis token yang dikirimkan dalam header HTTP:

```
Authorization: Bearer <token>
```

Alur kerja:

1. Client login → Server mengembalikan **token**
2. Client menyimpan token (di app/frontend)
3. Setiap request berikutnya, token dikirim di header `Authorization`
4. Server memvalidasi token → akses diberikan atau ditolak

### Apa itu Laravel Sanctum?

Laravel Sanctum menyediakan dua hal:

- **API Token Authentication** → cocok untuk SPA dan mobile app (yang akan kita gunakan)
- **Session-based SPA Authentication** → untuk frontend yang di-serve oleh Laravel

Sanctum menyimpan token di tabel `personal_access_tokens` di database, bukan di cookie atau session.

### Alur Autentikasi Lengkap

```
[Client]                          [Laravel API]
   |                                    |
   |  POST /api/auth/register           |
   |  { name, email, password }         |
   |  --------------------------------> |
   |                                    | [Buat user, buat token]
   |  { token: "abc123..." }            |
   |  <-------------------------------- |
   |                                    |
   |  GET /api/books                    |
   |  Authorization: Bearer abc123...   |
   |  --------------------------------> |
   |                                    | [Validasi token via middleware]
   |  { data: [...] }                   |
   |  <-------------------------------- |
```

---

## 2. Setup Project Laravel 12

### 2.1 Buat Project Baru

```bash
composer create-project laravel/laravel laravel-api-demo
cd laravel-api-demo
```

### 2.2 Konfigurasi Database

Edit file `.env`:

```env
APP_NAME="Laravel API Demo"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_api_demo
DB_USERNAME=root
DB_PASSWORD=
```

Buat database:

```sql
CREATE DATABASE laravel_api_demo;
```

### 2.3 Konfigurasi Dasar API

Di Laravel 12, file konfigurasi API terpisah. Pastikan file `bootstrap/app.php` sudah ada (default Laravel 12 menggunakan application bootstrapper baru).

Cek `config/app.php` untuk memastikan timezone sesuai:

```php
'timezone' => 'Asia/Jakarta',
'locale' => 'id',
```

---

## 3. Instalasi & Konfigurasi Sanctum

### 3.1 Install Sanctum

```bash
composer require laravel/sanctum
```

### 3.2 Publish Konfigurasi Sanctum

```bash
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
```

Perintah ini akan membuat:

- `config/sanctum.php`
- Migration `personal_access_tokens`

### 3.3 Konfigurasi `config/sanctum.php`

```php
<?php

return [
    /*
    |--------------------------------------------------------------------------
    | Stateful Domains
    |--------------------------------------------------------------------------
    | Domain yang dapat menggunakan cookie-based authentication (SPA).
    | Karena kita menggunakan token-based, bagian ini tidak terlalu kritikal.
    */
    'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
        '%s%s',
        'localhost,localhost:3000,127.0.0.1,127.0.0.1:8000,::1',
        Sanctum::currentApplicationUrlWithPort()
    ))),

    /*
    |--------------------------------------------------------------------------
    | Token Expiration
    |--------------------------------------------------------------------------
    | Waktu kadaluarsa token dalam menit. null = tidak pernah expired.
    | Untuk production, disarankan memberikan batas waktu.
    */
    'expiration' => null, // atau misal: 60 * 24 (24 jam)

    /*
    |--------------------------------------------------------------------------
    | Token Prefix
    |--------------------------------------------------------------------------
    | Prefix untuk token yang disimpan di database.
    */
    'token_prefix' => env('SANCTUM_TOKEN_PREFIX', ''),

    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies' => Illuminate\Cookie\Middleware\EncryptCookies::class,
        'validate_csrf_token' => Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
    ],
];
```

### 3.4 Tambahkan Trait Sanctum ke Model User

Edit `app/Models/User.php` — tambahkan `HasApiTokens`:

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
    // ...
}
```

### 3.5 Konfigurasi Auth Guard

Edit `config/auth.php` — pastikan guard `api` menggunakan `sanctum`:

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'sanctum',  // <-- Pastikan ini ada
        'provider' => 'users',
    ],
],
```

---

## 4. Database: Migration & Seeder

### 4.1 Migration Users (Default + Tambahan Role)

Edit migration `create_users_table` yang sudah ada:

```bash
# Lihat file migration yang ada
ls database/migrations/
```

Edit `database/migrations/xxxx_create_users_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->enum('role', ['admin', 'editor', 'user'])->default('user'); // <-- Tambah
            $table->boolean('is_active')->default(true); // <-- Tambah
            $table->rememberToken();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
    }
};
```

### 4.2 Migration Books

Buat migration baru:

```bash
php artisan make:migration create_books_table
```

Edit `database/migrations/xxxx_create_books_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('books', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->string('author');
            $table->string('isbn', 20)->unique()->nullable();
            $table->text('description')->nullable();
            $table->decimal('price', 10, 2)->default(0);
            $table->integer('stock')->default(0);
            $table->enum('status', ['available', 'unavailable'])->default('available');
            $table->foreignId('created_by')->constrained('users')->onDelete('cascade');
            $table->timestamps();
            $table->softDeletes(); // Untuk soft delete
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('books');
    }
};
```

### 4.3 Jalankan Migration

```bash
php artisan migrate
```

### 4.4 Seeder

Buat seeder untuk data awal:

```bash
php artisan make:seeder UserSeeder
php artisan make:seeder BookSeeder
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
            'name'     => 'Administrator',
            'email'    => 'admin@demo.com',
            'password' => Hash::make('password'),
            'role'     => 'admin',
        ]);

        // Editor
        User::create([
            'name'     => 'Editor Satu',
            'email'    => 'editor@demo.com',
            'password' => Hash::make('password'),
            'role'     => 'editor',
        ]);

        // Regular User
        User::create([
            'name'     => 'User Biasa',
            'email'    => 'user@demo.com',
            'password' => Hash::make('password'),
            'role'     => 'user',
        ]);
    }
}
```

Edit `database/seeders/BookSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\Book;
use Illuminate\Database\Seeder;

class BookSeeder extends Seeder
{
    public function run(): void
    {
        $books = [
            [
                'title'       => 'Clean Code',
                'author'      => 'Robert C. Martin',
                'isbn'        => '978-0132350884',
                'description' => 'Panduan menulis kode yang bersih dan mudah dipahami.',
                'price'       => 250000,
                'stock'       => 10,
                'status'      => 'available',
                'created_by'  => 1,
            ],
            [
                'title'       => 'The Pragmatic Programmer',
                'author'      => 'Andrew Hunt & David Thomas',
                'isbn'        => '978-0201616224',
                'description' => 'Filosofi dan praktik terbaik dalam software development.',
                'price'       => 320000,
                'stock'       => 5,
                'status'      => 'available',
                'created_by'  => 1,
            ],
            [
                'title'       => 'Laravel: Up & Running',
                'author'      => 'Matt Stauffer',
                'isbn'        => '978-1491936030',
                'description' => 'Panduan lengkap membangun aplikasi dengan Laravel.',
                'price'       => 280000,
                'stock'       => 0,
                'status'      => 'unavailable',
                'created_by'  => 2,
            ],
        ];

        foreach ($books as $book) {
            Book::create($book);
        }
    }
}
```

Edit `database/seeders/DatabaseSeeder.php`:

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call([
            UserSeeder::class,
            BookSeeder::class,
        ]);
    }
}
```

Jalankan seeder:

```bash
php artisan db:seed
```

Atau fresh migration + seed sekaligus:

```bash
php artisan migrate:fresh --seed
```

---

## 5. Model

### 5.1 Model User

Edit `app/Models/User.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'role',
        'is_active',
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
            'is_active'         => 'boolean',
        ];
    }

    // Helper method: cek role
    public function isAdmin(): bool
    {
        return $this->role === 'admin';
    }

    public function isEditor(): bool
    {
        return $this->role === 'editor';
    }

    public function hasRole(string|array $roles): bool
    {
        if (is_array($roles)) {
            return in_array($this->role, $roles);
        }
        return $this->role === $roles;
    }

    // Relasi: user bisa membuat banyak buku
    public function books()
    {
        return $this->hasMany(Book::class, 'created_by');
    }
}
```

### 5.2 Model Book

Buat model Book:

```bash
php artisan make:model Book
```

Edit `app/Models/Book.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Book extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'title',
        'author',
        'isbn',
        'description',
        'price',
        'stock',
        'status',
        'created_by',
    ];

    protected function casts(): array
    {
        return [
            'price' => 'decimal:2',
            'stock' => 'integer',
        ];
    }

    // Relasi: buku dimiliki oleh satu user (creator)
    public function creator()
    {
        return $this->belongsTo(User::class, 'created_by');
    }

    // Scope: hanya buku yang available
    public function scopeAvailable($query)
    {
        return $query->where('status', 'available');
    }
}
```

---

## 6. Auth API: Register, Login, Logout

### 6.1 Buat AuthController

```bash
php artisan make:controller Api/AuthController
```

Edit `app/Http/Controllers/Api/AuthController.php`:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    /**
     * Register user baru
     * POST /api/auth/register
     */
    public function register(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name'     => ['required', 'string', 'max:255'],
            'email'    => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password' => ['required', 'confirmed', Password::min(8)],
        ]);

        $user = User::create([
            'name'     => $validated['name'],
            'email'    => $validated['email'],
            'password' => Hash::make($validated['password']),
            'role'     => 'user', // Default role
        ]);

        // Buat token untuk user baru
        $token = $user->createToken('auth_token')->plainTextToken;

        return response()->json([
            'status'  => true,
            'message' => 'Registrasi berhasil.',
            'data'    => [
                'user'         => [
                    'id'    => $user->id,
                    'name'  => $user->name,
                    'email' => $user->email,
                    'role'  => $user->role,
                ],
                'token'        => $token,
                'token_type'   => 'Bearer',
            ],
        ], 201);
    }

    /**
     * Login user
     * POST /api/auth/login
     */
    public function login(Request $request): JsonResponse
    {
        $request->validate([
            'email'    => ['required', 'string', 'email'],
            'password' => ['required', 'string'],
        ]);

        // Cek kredensial
        if (!Auth::attempt(['email' => $request->email, 'password' => $request->password])) {
            return response()->json([
                'status'  => false,
                'message' => 'Email atau password salah.',
            ], 401);
        }

        $user = User::where('email', $request->email)->firstOrFail();

        // Cek apakah user aktif
        if (!$user->is_active) {
            return response()->json([
                'status'  => false,
                'message' => 'Akun Anda telah dinonaktifkan. Hubungi administrator.',
            ], 403);
        }

        // Hapus token lama (opsional — satu device satu token)
        $user->tokens()->delete();

        // Buat token baru dengan nama device/client
        $deviceName = $request->header('X-Device-Name', 'api_client');
        $token = $user->createToken($deviceName)->plainTextToken;

        return response()->json([
            'status'  => true,
            'message' => 'Login berhasil.',
            'data'    => [
                'user'       => [
                    'id'    => $user->id,
                    'name'  => $user->name,
                    'email' => $user->email,
                    'role'  => $user->role,
                ],
                'token'      => $token,
                'token_type' => 'Bearer',
            ],
        ]);
    }

    /**
     * Logout user (hapus token saat ini)
     * POST /api/auth/logout
     */
    public function logout(Request $request): JsonResponse
    {
        // Hapus hanya token yang digunakan saat ini
        $request->user()->currentAccessToken()->delete();

        return response()->json([
            'status'  => true,
            'message' => 'Logout berhasil.',
        ]);
    }

    /**
     * Logout dari semua device
     * POST /api/auth/logout-all
     */
    public function logoutAll(Request $request): JsonResponse
    {
        // Hapus semua token milik user
        $request->user()->tokens()->delete();

        return response()->json([
            'status'  => true,
            'message' => 'Logout dari semua device berhasil.',
        ]);
    }

    /**
     * Lihat profil user yang sedang login
     * GET /api/auth/me
     */
    public function me(Request $request): JsonResponse
    {
        return response()->json([
            'status' => true,
            'data'   => [
                'user' => $request->user(),
            ],
        ]);
    }
}
```

---

## 7. CRUD Books dengan Authorization

### 7.1 Buat BookController

```bash
php artisan make:controller Api/BookController --api
```

Edit `app/Http/Controllers/Api/BookController.php`:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Resources\BookResource;
use App\Models\Book;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class BookController extends Controller
{
    /**
     * GET /api/books
     * Semua user (terautentikasi) bisa melihat daftar buku
     */
    public function index(Request $request): AnonymousResourceCollection
    {
        $query = Book::with('creator:id,name');

        // Filter by status
        if ($request->has('status')) {
            $query->where('status', $request->status);
        }

        // Search by title or author
        if ($request->has('search')) {
            $search = $request->search;
            $query->where(function ($q) use ($search) {
                $q->where('title', 'like', "%{$search}%")
                  ->orWhere('author', 'like', "%{$search}%");
            });
        }

        // Sorting
        $sortBy    = $request->get('sort_by', 'created_at');
        $sortOrder = $request->get('sort_order', 'desc');
        $allowedSorts = ['title', 'author', 'price', 'stock', 'created_at'];

        if (in_array($sortBy, $allowedSorts)) {
            $query->orderBy($sortBy, $sortOrder);
        }

        $books = $query->paginate($request->get('per_page', 10));

        return BookResource::collection($books);
    }

    /**
     * POST /api/books
     * Hanya admin dan editor yang bisa membuat buku baru
     */
    public function store(Request $request): JsonResponse
    {
        // Authorization: cek role
        if (!$request->user()->hasRole(['admin', 'editor'])) {
            return response()->json([
                'status'  => false,
                'message' => 'Anda tidak memiliki izin untuk menambah buku.',
            ], 403);
        }

        $validated = $request->validate([
            'title'       => ['required', 'string', 'max:255'],
            'author'      => ['required', 'string', 'max:255'],
            'isbn'        => ['nullable', 'string', 'max:20', 'unique:books'],
            'description' => ['nullable', 'string'],
            'price'       => ['required', 'numeric', 'min:0'],
            'stock'       => ['required', 'integer', 'min:0'],
            'status'      => ['sometimes', 'in:available,unavailable'],
        ]);

        $validated['created_by'] = $request->user()->id;

        $book = Book::create($validated);

        return response()->json([
            'status'  => true,
            'message' => 'Buku berhasil ditambahkan.',
            'data'    => new BookResource($book->load('creator:id,name')),
        ], 201);
    }

    /**
     * GET /api/books/{id}
     * Semua user (terautentikasi) bisa melihat detail buku
     */
    public function show(Book $book): JsonResponse
    {
        return response()->json([
            'status' => true,
            'data'   => new BookResource($book->load('creator:id,name')),
        ]);
    }

    /**
     * PUT /api/books/{id}
     * Admin bisa update semua buku. Editor hanya bisa update buku miliknya.
     */
    public function update(Request $request, Book $book): JsonResponse
    {
        $user = $request->user();

        // Authorization
        if ($user->isAdmin()) {
            // Admin bisa update buku apapun
        } elseif ($user->isEditor() && $book->created_by === $user->id) {
            // Editor hanya bisa update buku yang dia buat
        } else {
            return response()->json([
                'status'  => false,
                'message' => 'Anda tidak memiliki izin untuk mengubah buku ini.',
            ], 403);
        }

        $validated = $request->validate([
            'title'       => ['sometimes', 'string', 'max:255'],
            'author'      => ['sometimes', 'string', 'max:255'],
            'isbn'        => ['nullable', 'string', 'max:20', 'unique:books,isbn,' . $book->id],
            'description' => ['nullable', 'string'],
            'price'       => ['sometimes', 'numeric', 'min:0'],
            'stock'       => ['sometimes', 'integer', 'min:0'],
            'status'      => ['sometimes', 'in:available,unavailable'],
        ]);

        $book->update($validated);

        return response()->json([
            'status'  => true,
            'message' => 'Buku berhasil diperbarui.',
            'data'    => new BookResource($book->load('creator:id,name')),
        ]);
    }

    /**
     * DELETE /api/books/{id}
     * Hanya admin yang bisa menghapus buku
     */
    public function destroy(Request $request, Book $book): JsonResponse
    {
        if (!$request->user()->isAdmin()) {
            return response()->json([
                'status'  => false,
                'message' => 'Hanya administrator yang dapat menghapus buku.',
            ], 403);
        }

        $book->delete(); // Soft delete

        return response()->json([
            'status'  => true,
            'message' => 'Buku berhasil dihapus.',
        ]);
    }
}
```

---

## 8. CRUD User (Admin Only)

### 8.1 Buat UserController

```bash
php artisan make:controller Api/UserController --api
```

Edit `app/Http/Controllers/Api/UserController.php`:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Resources\UserResource;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;

class UserController extends Controller
{
    /**
     * GET /api/users
     * Daftar semua user — hanya admin
     */
    public function index(Request $request): JsonResponse
    {
        $this->authorizeAdmin($request);

        $users = User::query()
            ->when($request->has('role'), fn($q) => $q->where('role', $request->role))
            ->when($request->has('search'), fn($q) => $q->where(function ($q) use ($request) {
                $q->where('name', 'like', "%{$request->search}%")
                  ->orWhere('email', 'like', "%{$request->search}%");
            }))
            ->orderBy('created_at', 'desc')
            ->paginate($request->get('per_page', 10));

        return response()->json([
            'status' => true,
            'data'   => UserResource::collection($users)->response()->getData(true),
        ]);
    }

    /**
     * POST /api/users
     * Buat user baru — hanya admin
     */
    public function store(Request $request): JsonResponse
    {
        $this->authorizeAdmin($request);

        $validated = $request->validate([
            'name'      => ['required', 'string', 'max:255'],
            'email'     => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password'  => ['required', 'confirmed', Password::min(8)],
            'role'      => ['required', 'in:admin,editor,user'],
            'is_active' => ['boolean'],
        ]);

        $validated['password'] = Hash::make($validated['password']);
        $user = User::create($validated);

        return response()->json([
            'status'  => true,
            'message' => 'User berhasil dibuat.',
            'data'    => new UserResource($user),
        ], 201);
    }

    /**
     * GET /api/users/{id}
     * Detail satu user — hanya admin
     */
    public function show(Request $request, User $user): JsonResponse
    {
        $this->authorizeAdmin($request);

        return response()->json([
            'status' => true,
            'data'   => new UserResource($user),
        ]);
    }

    /**
     * PUT /api/users/{id}
     * Update user — hanya admin
     */
    public function update(Request $request, User $user): JsonResponse
    {
        $this->authorizeAdmin($request);

        $validated = $request->validate([
            'name'      => ['sometimes', 'string', 'max:255'],
            'email'     => ['sometimes', 'string', 'email', 'max:255', 'unique:users,email,' . $user->id],
            'password'  => ['sometimes', 'confirmed', Password::min(8)],
            'role'      => ['sometimes', 'in:admin,editor,user'],
            'is_active' => ['sometimes', 'boolean'],
        ]);

        if (isset($validated['password'])) {
            $validated['password'] = Hash::make($validated['password']);
        }

        $user->update($validated);

        return response()->json([
            'status'  => true,
            'message' => 'User berhasil diperbarui.',
            'data'    => new UserResource($user),
        ]);
    }

    /**
     * DELETE /api/users/{id}
     * Hapus user — hanya admin
     */
    public function destroy(Request $request, User $user): JsonResponse
    {
        $this->authorizeAdmin($request);

        // Admin tidak bisa menghapus dirinya sendiri
        if ($user->id === $request->user()->id) {
            return response()->json([
                'status'  => false,
                'message' => 'Anda tidak dapat menghapus akun Anda sendiri.',
            ], 422);
        }

        // Hapus semua token user yang dihapus
        $user->tokens()->delete();
        $user->delete();

        return response()->json([
            'status'  => true,
            'message' => 'User berhasil dihapus.',
        ]);
    }

    /**
     * Helper: Cek apakah user adalah admin
     */
    private function authorizeAdmin(Request $request): void
    {
        if (!$request->user()->isAdmin()) {
            abort(response()->json([
                'status'  => false,
                'message' => 'Akses ditolak. Fitur ini hanya untuk administrator.',
            ], 403));
        }
    }
}
```

---

## 9. API Routes & Middleware

### 9.1 Definisi Routes

Edit `routes/api.php`:

```php
<?php

use App\Http\Controllers\Api\AuthController;
use App\Http\Controllers\Api\BookController;
use App\Http\Controllers\Api\UserController;
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
|
| Di Laravel 12, routes/api.php di-load secara otomatis dengan prefix /api
| dan menggunakan middleware 'api'.
|
*/

// ========================================
// PUBLIC ROUTES — Tidak perlu autentikasi
// ========================================
Route::prefix('auth')->group(function () {
    Route::post('/register', [AuthController::class, 'register']);
    Route::post('/login',    [AuthController::class, 'login']);
});

// ========================================
// PROTECTED ROUTES — Perlu Bearer Token
// ========================================
Route::middleware('auth:sanctum')->group(function () {

    // Auth routes
    Route::prefix('auth')->group(function () {
        Route::get('/me',           [AuthController::class, 'me']);
        Route::post('/logout',      [AuthController::class, 'logout']);
        Route::post('/logout-all',  [AuthController::class, 'logoutAll']);
    });

    // Books — CRUD (authorization dilakukan di controller)
    Route::apiResource('books', BookController::class);

    // Users — CRUD (admin only, authorization dilakukan di controller)
    Route::apiResource('users', UserController::class);
});
```

> **Catatan Penting Laravel 12:**  
> Di Laravel 12, pastikan `routes/api.php` di-register di `bootstrap/app.php`. Cek file tersebut:
>
> ```php
> ->withRouting(
>     web: __DIR__.'/../routes/web.php',
>     api: __DIR__.'/../routes/api.php',  // <-- Pastikan ini ada
>     commands: __DIR__.'/../routes/console.php',
>     health: '/up',
> )
> ```

### 9.2 Middleware Authorization (Role-Based)

Untuk pendekatan yang lebih clean, kita bisa membuat custom middleware:

```bash
php artisan make:middleware EnsureUserIsAdmin
php artisan make:middleware EnsureUserHasRole
```

Edit `app/Http/Middleware/EnsureUserHasRole.php`:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserHasRole
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        // Pastikan user sudah terautentikasi
        if (!$request->user()) {
            return response()->json([
                'status'  => false,
                'message' => 'Unauthenticated. Silakan login terlebih dahulu.',
            ], 401);
        }

        // Cek apakah role user ada dalam daftar roles yang diizinkan
        if (!in_array($request->user()->role, $roles)) {
            return response()->json([
                'status'  => false,
                'message' => 'Forbidden. Anda tidak memiliki akses ke resource ini.',
                'required_roles' => $roles,
                'your_role'      => $request->user()->role,
            ], 403);
        }

        return $next($request);
    }
}
```

### 9.3 Daftarkan Middleware di `bootstrap/app.php`

Di Laravel 12, middleware alias didaftarkan langsung di `bootstrap/app.php`:

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
        // Daftarkan alias middleware custom
        $middleware->alias([
            'role' => \App\Http\Middleware\EnsureUserHasRole::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

### 9.4 Gunakan Middleware di Routes (Alternatif)

Setelah middleware terdaftar, bisa digunakan langsung di routes:

```php
// Contoh penggunaan middleware role di routes
Route::middleware(['auth:sanctum', 'role:admin'])->group(function () {
    Route::apiResource('users', UserController::class);
});

Route::middleware(['auth:sanctum', 'role:admin,editor'])->group(function () {
    Route::post('/books', [BookController::class, 'store']);
    Route::put('/books/{book}', [BookController::class, 'update']);
    Route::delete('/books/{book}', [BookController::class, 'destroy']);
});

Route::middleware('auth:sanctum')->group(function () {
    Route::get('/books', [BookController::class, 'index']);
    Route::get('/books/{book}', [BookController::class, 'show']);
});
```

---

## 10. API Resource (Response Formatting)

API Resource digunakan untuk memformat dan mengontrol data yang dikirim ke client.

### 10.1 Buat Resources

```bash
php artisan make:resource BookResource
php artisan make:resource UserResource
```

Edit `app/Http/Resources/BookResource.php`:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class BookResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'          => $this->id,
            'title'       => $this->title,
            'author'      => $this->author,
            'isbn'        => $this->isbn,
            'description' => $this->description,
            'price'       => (float) $this->price,
            'price_formatted' => 'Rp ' . number_format($this->price, 0, ',', '.'),
            'stock'       => $this->stock,
            'status'      => $this->status,
            'creator'     => $this->whenLoaded('creator', fn() => [
                'id'   => $this->creator->id,
                'name' => $this->creator->name,
            ]),
            'created_at'  => $this->created_at?->toISOString(),
            'updated_at'  => $this->updated_at?->toISOString(),
        ];
    }
}
```

Edit `app/Http/Resources/UserResource.php`:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'name'       => $this->name,
            'email'      => $this->email,
            'role'       => $this->role,
            'is_active'  => $this->is_active,
            'created_at' => $this->created_at?->toISOString(),
            'updated_at' => $this->updated_at?->toISOString(),
            // Field 'password' tidak ditampilkan (sudah di $hidden model)
        ];
    }
}
```

---

## 11. Testing dengan Postman / cURL

### 11.1 Setup Environment Postman

Buat environment baru di Postman:

- **Variable:** `base_url` = `http://localhost:8000/api`
- **Variable:** `token` = _(diisi setelah login)_

### 11.2 Register

```
POST {{base_url}}/auth/register
Content-Type: application/json

{
    "name": "Mahasiswa Baru",
    "email": "mahasiswa@demo.com",
    "password": "password123",
    "password_confirmation": "password123"
}
```

**Response sukses (201):**

```json
{
  "status": true,
  "message": "Registrasi berhasil.",
  "data": {
    "user": {
      "id": 4,
      "name": "Mahasiswa Baru",
      "email": "mahasiswa@demo.com",
      "role": "user"
    },
    "token": "1|abc123xyz...",
    "token_type": "Bearer"
  }
}
```

### 11.3 Login

```
POST {{base_url}}/auth/login
Content-Type: application/json

{
    "email": "admin@demo.com",
    "password": "password"
}
```

**Response sukses (200):**

```json
{
  "status": true,
  "message": "Login berhasil.",
  "data": {
    "user": {
      "id": 1,
      "name": "Administrator",
      "email": "admin@demo.com",
      "role": "admin"
    },
    "token": "2|def456uvw...",
    "token_type": "Bearer"
  }
}
```

> **Tips Postman:** Salin nilai `token` dan simpan ke variable `token` di environment. Tambahkan script di tab "Tests":
>
> ```javascript
> if (pm.response.code === 200) {
>   const data = pm.response.json();
>   pm.environment.set("token", data.data.token);
> }
> ```

### 11.4 Menggunakan Bearer Token

Untuk semua request yang membutuhkan autentikasi, tambahkan header:

```
Authorization: Bearer {{token}}
```

Atau di tab **Authorization** Postman → pilih **Bearer Token** → isi dengan `{{token}}`.

### 11.5 CRUD Books

**GET semua buku:**

```
GET {{base_url}}/books
Authorization: Bearer {{token}}
```

**GET buku dengan filter:**

```
GET {{base_url}}/books?status=available&search=clean&sort_by=price&sort_order=asc&per_page=5
Authorization: Bearer {{token}}
```

**POST buat buku baru (harus login sebagai admin/editor):**

```
POST {{base_url}}/books
Authorization: Bearer {{token}}
Content-Type: application/json

{
    "title": "Domain-Driven Design",
    "author": "Eric Evans",
    "isbn": "978-0321125217",
    "description": "Tackling Complexity in the Heart of Software.",
    "price": 350000,
    "stock": 8,
    "status": "available"
}
```

**PUT update buku:**

```
PUT {{base_url}}/books/1
Authorization: Bearer {{token}}
Content-Type: application/json

{
    "price": 275000,
    "stock": 15
}
```

**DELETE hapus buku (harus admin):**

```
DELETE {{base_url}}/books/1
Authorization: Bearer {{token}}
```

### 11.6 Response Error: Token Tidak Valid

Jika request dilakukan **tanpa token** atau token **tidak valid**:

```
GET {{base_url}}/books
(tanpa header Authorization)
```

**Response (401):**

```json
{
  "message": "Unauthenticated."
}
```

### 11.7 Response Error: Akses Ditolak

Jika user biasa mencoba membuat buku:

```
POST {{base_url}}/books
Authorization: Bearer <token-user-biasa>
```

**Response (403):**

```json
{
  "status": false,
  "message": "Anda tidak memiliki izin untuk menambah buku."
}
```

### 11.8 Testing dengan cURL

**Login via cURL:**

```bash
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@demo.com","password":"password"}'
```

**GET books dengan token:**

```bash
curl -X GET http://localhost:8000/api/books \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Accept: application/json"
```

---

## 12. Error Handling Global

### 12.1 Custom Exception Handler

Di Laravel 12, exception handling dikonfigurasi di `bootstrap/app.php`. Tambahkan handler untuk respons API yang konsisten:

```php
->withExceptions(function (Exceptions $exceptions) {

    // Handle ValidationException → 422
    $exceptions->render(function (\Illuminate\Validation\ValidationException $e, $request) {
        if ($request->is('api/*') || $request->wantsJson()) {
            return response()->json([
                'status'  => false,
                'message' => 'Data yang diberikan tidak valid.',
                'errors'  => $e->errors(),
            ], 422);
        }
    });

    // Handle AuthenticationException → 401
    $exceptions->render(function (\Illuminate\Auth\AuthenticationException $e, $request) {
        if ($request->is('api/*') || $request->wantsJson()) {
            return response()->json([
                'status'  => false,
                'message' => 'Unauthenticated. Silakan login terlebih dahulu.',
            ], 401);
        }
    });

    // Handle ModelNotFoundException → 404
    $exceptions->render(function (\Illuminate\Database\Eloquent\ModelNotFoundException $e, $request) {
        if ($request->is('api/*') || $request->wantsJson()) {
            $model = class_basename($e->getModel());
            return response()->json([
                'status'  => false,
                'message' => "{$model} tidak ditemukan.",
            ], 404);
        }
    });

    // Handle RouteNotFoundException → 404
    $exceptions->render(function (\Symfony\Component\HttpKernel\Exception\NotFoundHttpException $e, $request) {
        if ($request->is('api/*') || $request->wantsJson()) {
            return response()->json([
                'status'  => false,
                'message' => 'Endpoint tidak ditemukan.',
            ], 404);
        }
    });

    // Handle MethodNotAllowedHttpException → 405
    $exceptions->render(function (\Symfony\Component\HttpKernel\Exception\MethodNotAllowedHttpException $e, $request) {
        if ($request->is('api/*') || $request->wantsJson()) {
            return response()->json([
                'status'  => false,
                'message' => 'Method HTTP tidak diizinkan untuk endpoint ini.',
            ], 405);
        }
    });

})
```

### 12.2 Format Response yang Konsisten

Selalu gunakan format response yang seragam:

```json
// Sukses
{
    "status": true,
    "message": "Pesan sukses",
    "data": { ... }
}

// Sukses dengan paginasi
{
    "status": true,
    "data": {
        "current_page": 1,
        "data": [ ... ],
        "per_page": 10,
        "total": 50
    }
}

// Error
{
    "status": false,
    "message": "Pesan error",
    "errors": { ... }  // Opsional untuk validation errors
}
```

---

## 13. Ringkasan Arsitektur

### Struktur File yang Dibuat

```
app/
├── Http/
│   ├── Controllers/
│   │   └── Api/
│   │       ├── AuthController.php     ← Register, Login, Logout, Me
│   │       ├── BookController.php     ← CRUD Books
│   │       └── UserController.php     ← CRUD Users (Admin only)
│   ├── Middleware/
│   │   └── EnsureUserHasRole.php      ← Custom role middleware
│   └── Resources/
│       ├── BookResource.php           ← Format response buku
│       └── UserResource.php           ← Format response user
├── Models/
│   ├── User.php                       ← HasApiTokens, role helpers
│   └── Book.php                       ← SoftDeletes, scopes
│
bootstrap/
└── app.php                            ← Routing, middleware alias, exception handling
│
config/
├── auth.php                           ← Guard api: sanctum
└── sanctum.php                        ← Token expiration
│
database/
├── migrations/
│   ├── ..._create_users_table.php     ← +role, +is_active
│   └── ..._create_books_table.php
└── seeders/
    ├── UserSeeder.php
    └── BookSeeder.php
│
routes/
└── api.php                            ← Semua API routes
```

### Tabel Ringkasan Endpoint & Akses

| Method | Endpoint             | Auth | Role Yang Diizinkan |
| ------ | -------------------- | ---- | ------------------- |
| POST   | `/api/auth/register` | ❌   | Public              |
| POST   | `/api/auth/login`    | ❌   | Public              |
| GET    | `/api/auth/me`       | ✅   | Semua               |
| POST   | `/api/auth/logout`   | ✅   | Semua               |
| GET    | `/api/books`         | ✅   | Semua               |
| GET    | `/api/books/{id}`    | ✅   | Semua               |
| POST   | `/api/books`         | ✅   | Admin, Editor       |
| PUT    | `/api/books/{id}`    | ✅   | Admin, Editor\*     |
| DELETE | `/api/books/{id}`    | ✅   | Admin               |
| GET    | `/api/users`         | ✅   | Admin               |
| POST   | `/api/users`         | ✅   | Admin               |
| GET    | `/api/users/{id}`    | ✅   | Admin               |
| PUT    | `/api/users/{id}`    | ✅   | Admin               |
| DELETE | `/api/users/{id}`    | ✅   | Admin               |

\*Editor hanya bisa update buku yang dia buat sendiri.

### Alur Bearer Token (Ringkasan)

```
1. Client → POST /api/auth/login { email, password }
2. Server → Validasi kredensial → Buat token → Return { token: "..." }
3. Client → Simpan token (localStorage / app state)
4. Client → GET /api/books
           Header: Authorization: Bearer <token>
5. Server → middleware auth:sanctum memvalidasi token
          → Jika valid, request dilanjutkan ke controller
          → Jika tidak valid, return 401 Unauthenticated
6. Controller → Cek role (opsional) → Proses request → Return response
7. Client → POST /api/auth/logout
           Header: Authorization: Bearer <token>
8. Server → Token dihapus dari database
9. Client → Token di sisi client dihapus
```

---

## Referensi

- [Laravel 12 Documentation](https://laravel.com/docs/12.x)
- [Laravel Sanctum Documentation](https://laravel.com/docs/12.x/sanctum)
- [Laravel API Resources](https://laravel.com/docs/12.x/eloquent-resources)
- [RFC 6750 — OAuth 2.0 Bearer Token Usage](https://tools.ietf.org/html/rfc6750)

---

_Maret 2026_
