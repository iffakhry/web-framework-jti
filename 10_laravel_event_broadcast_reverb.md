# Event Broadcast & Laravel Reverb

|                   |                                  |
| ----------------- | -------------------------------- |
| **Pokok Bahasan** | Event Broadcast & Laravel Reverb |

---

## d. Dasar Teori

### 1. Laravel Reverb

Laravel Reverb adalah server WebSocket resmi yang dikembangkan oleh tim Laravel, diperkenalkan bersama Laravel 11 dan terus disempurnakan di Laravel 12. Reverb memungkinkan komunikasi dua arah secara real-time antara server dan klien tanpa ketergantungan pada layanan pihak ketiga berbayar seperti Pusher.

Reverb menggunakan protokol WebSocket yang kompatibel dengan Pusher, sehingga dapat bekerja langsung bersama Laravel Echo — library JavaScript resmi untuk mendengarkan event real-time di sisi klien.

**Keunggulan Reverb di Laravel 12:**

- Terintegrasi native dengan ekosistem Laravel
- Mandiri (self-hosted) dan bebas biaya langganan
- Mendukung horizontal scaling via Redis Pub/Sub
- Monitoring bawaan melalui Laravel Pulse
- Tersedia sebagai managed service di Laravel Cloud

### 2. Tiga Jenis Channel Broadcasting

> **[DITAMBAH]** — Modul sebelumnya hanya menggunakan Public Channel. Pemahaman ketiga jenis channel adalah fondasi penting dalam merancang sistem real-time yang aman.

| Jenis Channel        | Autentikasi                         | Kapan Digunakan                                                               |
| -------------------- | ----------------------------------- | ----------------------------------------------------------------------------- |
| **Public Channel**   | Tidak diperlukan                    | Data yang boleh diterima siapa saja (siaran publik, harga saham, berita umum) |
| **Private Channel**  | Wajib login + otorisasi             | Data spesifik user/grup (notifikasi personal, pesan pribadi)                  |
| **Presence Channel** | Wajib login + otorisasi + info user | Fitur "siapa yang sedang online" (collaborative editing, chat room)           |

Cara berlangganan di sisi JavaScript (Laravel Echo):

```javascript
// Public Channel — siapa saja bisa subscribe
Echo.channel("announcements").listen("AnnouncementPosted", (e) => {});

// Private Channel — harus login
Echo.private("chat.room.1").listen("MessageSent", (e) => {});

// Presence Channel — harus login + mendapat info member
Echo.join("classroom.1")
  .here((users) => {}) // daftar user yang sedang online
  .joining((user) => {}) // user baru masuk
  .leaving((user) => {}) // user keluar
  .listen("MessageSent", (e) => {});
```

### 3. Interface ShouldBroadcast vs ShouldBroadcastNow

> **[DIUBAH]** — Modul sebelumnya menggunakan `ShouldBroadcastNow` tanpa penjelasan konsekuensinya. Pemilihan interface yang tepat berdampak besar pada performa aplikasi.

| Interface            | Cara Kerja                                                             | Kapan Digunakan                                     |
| -------------------- | ---------------------------------------------------------------------- | --------------------------------------------------- |
| `ShouldBroadcast`    | Event dikirim ke **queue**, diproses oleh queue worker secara asinkron | **Produksi** — tidak memblokir request HTTP         |
| `ShouldBroadcastNow` | Event dikirim **langsung** (sinkron), melewati queue                   | **Development/debugging** saja — memblokir response |

Pada praktikum ini kita menggunakan `ShouldBroadcast` sesuai praktik industri, sehingga queue worker harus dijalankan secara bersamaan.

### 4. Method Kontrol Payload: broadcastAs() dan broadcastWith()

> **[DITAMBAH]** — Tanpa kedua method ini, semua public property pada class Event akan dikirim ke browser dalam format nama class PHP yang panjang. Ini berisiko membocorkan struktur internal aplikasi.

```php
// Tanpa broadcastAs() — nama event di JavaScript menjadi:
// "App\\Events\\ClassroomMessageSent" (tidak praktis)

// Dengan broadcastAs() — nama event menjadi bersih:
public function broadcastAs(): string
{
    return 'message.sent'; // nama pendek untuk JavaScript
}

// Tanpa broadcastWith() — SEMUA public property dikirim ke client
// termasuk data yang tidak perlu atau sensitif

// Dengan broadcastWith() — hanya data yang dipilih yang dikirim
public function broadcastWith(): array
{
    return [
        'id'         => $this->message->id,
        'content'    => $this->message->content,
        'user_name'  => $this->message->user->name,
        'created_at' => $this->message->created_at->toISOString(),
        // user->password, user->email, dsb. TIDAK dikirim
    ];
}
```

---

## e. Alat dan Bahan

1. PC / Laptop
2. RAM 4 GB (Minimal) / 8 GB (Rekomendasi)
3. PHP 8.2 atau lebih baru
4. Composer 2.x
5. Node.js 18+ dan NPM
6. Laravel 12
7. Text Editor: Visual Studio Code
8. Browser modern (Chrome/Firefox)
9. Terminal / Command Prompt

---

## f. Prosedur Kerja

### Skenario Studi Kasus

**ClassRoom Chat — Sistem Chat Ruang Kelas Real-Time**

Politeknik Negeri Jember ingin membangun fitur diskusi real-time di dalam Learning Management System (LMS) internal. Setiap mata kuliah memiliki ruang chat kelas tersendiri. Hanya mahasiswa dan dosen yang terdaftar pada mata kuliah tersebut yang dapat mengakses ruang chat. Sistem harus menampilkan siapa saja yang sedang aktif online di ruang chat saat ini.

**Fitur yang akan dibangun:**

- Mahasiswa/dosen login dan membuka ruang chat mata kuliah
- Pesan dikirim dan langsung muncul di semua browser anggota tanpa refresh
- Daftar member yang sedang online diperbarui secara real-time
- Hanya anggota mata kuliah yang terdaftar yang bisa mengakses chat

---

### Langkah 1 — Instalasi Project Laravel 12

Buat project Laravel 12 baru dan masuk ke direktori project:

```bash
composer create-project laravel/laravel classroom-chat
cd classroom-chat
```

Install Laravel Breeze untuk autentikasi (diperlukan untuk Private dan Presence Channel):

```bash
composer require laravel/breeze --dev
php artisan breeze:install blade
npm install && npm run build
```

### Langkah 2 — Instalasi dan Konfigurasi Broadcasting

> **[DIUBAH]** — Perintah ini sudah ada di modul sebelumnya dan masih valid di Laravel 12. Namun perlu dipahami apa yang terjadi di baliknya.

Jalankan perintah instalasi broadcasting:

```bash
php artisan install:broadcasting
```

Perintah ini secara otomatis melakukan:

- Menginstal package `laravel/reverb` via Composer
- Menginstal `laravel-echo` dan `pusher-js` via NPM
- Membuat file `config/broadcasting.php`
- Membuat file `config/reverb.php`
- Membuat file `routes/channels.php`
- Menambahkan variabel `REVERB_*` ke file `.env`
- Membuat file `resources/js/echo.js`

### Langkah 3 — Konfigurasi File .env

> **[DIUBAH]** — Nilai `REVERB_APP_ID`, `REVERB_APP_KEY`, dan `REVERB_APP_SECRET` sudah di-generate otomatis oleh perintah `install:broadcasting`. Tidak perlu mengisi manual kecuali untuk keperluan khusus. Yang perlu dipastikan adalah baris berikut:

Buka file `.env` dan pastikan konfigurasi berikut sudah ada dan benar:

```env
# [DIUBAH] Gunakan 'reverb' bukan 'log' atau 'pusher'
BROADCAST_CONNECTION=reverb

# Nilai berikut sudah di-generate otomatis — jangan diubah kecuali perlu
REVERB_APP_ID=your_generated_app_id
REVERB_APP_KEY=your_generated_app_key
REVERB_APP_SECRET=your_generated_app_secret
REVERB_HOST="localhost"
REVERB_PORT=8080
REVERB_SCHEME=http

# Prefix VITE_ agar nilai dapat dibaca di sisi JavaScript/Vite
VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST="${REVERB_HOST}"
VITE_REVERB_PORT="${REVERB_PORT}"
VITE_REVERB_SCHEME="${REVERB_SCHEME}"

# [DITAMBAH] Queue driver — gunakan 'database' untuk development
# Ganti 'redis' untuk produksi
QUEUE_CONNECTION=database
```

Buat tabel queue di database:

```bash
php artisan queue:table
php artisan migrate
```

### Langkah 4 — Membuat Struktur Database

Buat migration untuk tabel `classrooms` (ruang kelas) dan `messages` (pesan chat):

```bash
php artisan make:migration create_classrooms_table
php artisan make:migration create_classroom_user_table
php artisan make:migration create_messages_table
```

**Migration `create_classrooms_table`:**

```php
// database/migrations/xxxx_create_classrooms_table.php
public function up(): void
{
    Schema::create('classrooms', function (Blueprint $table) {
        $table->id();
        $table->string('name');           // nama mata kuliah
        $table->string('code')->unique(); // kode mata kuliah, mis. TI-401
        $table->text('description')->nullable();
        $table->timestamps();
    });
}
```

**Migration `create_classroom_user_table` (pivot):**

```php
// database/migrations/xxxx_create_classroom_user_table.php
public function up(): void
{
    Schema::create('classroom_user', function (Blueprint $table) {
        $table->id();
        $table->foreignId('classroom_id')->constrained()->cascadeOnDelete();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->string('role')->default('student'); // 'student' atau 'lecturer'
        $table->timestamps();
    });
}
```

**Migration `create_messages_table`:**

```php
// database/migrations/xxxx_create_messages_table.php
public function up(): void
{
    Schema::create('messages', function (Blueprint $table) {
        $table->id();
        $table->foreignId('classroom_id')->constrained()->cascadeOnDelete();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->text('content');
        $table->timestamps();
    });
}
```

Jalankan semua migration:

```bash
php artisan migrate
```

### Langkah 5 — Membuat Model

**Model `Classroom`:**

```php
<?php
// app/Models/Classroom.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Classroom extends Model
{
    protected $fillable = ['name', 'code', 'description'];

    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class)
                    ->withPivot('role')
                    ->withTimestamps();
    }

    public function messages(): HasMany
    {
        return $this->hasMany(Message::class)->latest();
    }
}
```

**Model `Message`:**

> **[DITAMBAH]** — Gunakan trait `BroadcastsEvents` untuk Model Broadcasting. Fitur ini memungkinkan Eloquent model melakukan broadcast secara otomatis saat event created/updated/deleted terjadi, tanpa perlu membuat Event class terpisah.

```php
<?php
// app/Models/Message.php

namespace App\Models;

use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Database\Eloquent\BroadcastsEvents;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Message extends Model
{
    // [DITAMBAH] Trait ini mengaktifkan Model Broadcasting
    use BroadcastsEvents;

    protected $fillable = ['classroom_id', 'user_id', 'content'];

    // [DITAMBAH] Tentukan channel tempat event model di-broadcast
    // Model Broadcasting secara otomatis broadcast pada event:
    // created, updated, deleted, restored, trashed
    public function broadcastOn(string $event): array
    {
        // Hanya broadcast saat pesan baru dibuat
        return match ($event) {
            'created' => [new PrivateChannel('classroom.' . $this->classroom_id)],
            default   => [],
        };
    }

    // [DITAMBAH] Kontrol data yang dikirim ke client
    // Tanpa method ini, seluruh atribut model dikirim termasuk foreign key mentah
    public function broadcastWith(string $event): array
    {
        return [
            'id'         => $this->id,
            'content'    => $this->content,
            'user_name'  => $this->user->name,
            'user_id'    => $this->user_id,
            'created_at' => $this->created_at->format('H:i'),
        ];
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function classroom(): BelongsTo
    {
        return $this->belongsTo(Classroom::class);
    }
}
```

> **Catatan perbedaan Model Broadcasting vs Event Class:**
>
> Dengan `BroadcastsEvents`, kita tidak perlu membuat file `app/Events/MessageCreated.php` secara manual. Model akan otomatis me-broadcast dirinya sendiri. Nama event yang diterima di JavaScript adalah `MessageCreated` (dari nama model + nama event Eloquent).

### Langkah 6 — Mendefinisikan Channel Authorization

> **[DITAMBAH — KRITIS]** — Ini adalah bagian yang **sama sekali tidak ada** di modul sebelumnya. Channel Authorization adalah mekanisme keamanan yang memastikan hanya user yang berhak dapat berlangganan ke sebuah Private atau Presence Channel.

Buka file `routes/channels.php` dan definisikan otorisasi untuk dua channel:

```php
<?php
// routes/channels.php

use App\Models\Classroom;
use Illuminate\Support\Facades\Broadcast;

/*
|--------------------------------------------------------------------------
| Private Channel: classroom.{classroomId}
|--------------------------------------------------------------------------
| Digunakan untuk broadcast pesan chat.
| Hanya user yang terdaftar sebagai anggota classroom yang boleh masuk.
|
| Callback harus mengembalikan: true (boleh) atau false (tolak)
*/
Broadcast::channel('classroom.{classroomId}', function ($user, int $classroomId) {
    // Periksa apakah user adalah anggota classroom ini
    return $user->classrooms()->where('classroom_id', $classroomId)->exists();
});

/*
|--------------------------------------------------------------------------
| Presence Channel: presence-classroom.{classroomId}
|--------------------------------------------------------------------------
| Digunakan untuk fitur "siapa yang sedang online".
| Hanya user yang terdaftar sebagai anggota yang boleh bergabung.
|
| [PENTING] Callback Presence Channel HARUS mengembalikan array (bukan true/false).
| Array ini berisi data user yang akan dibagikan ke semua member lain di channel.
| Kembalikan false atau null untuk menolak akses.
*/
Broadcast::channel('presence-classroom.{classroomId}', function ($user, int $classroomId) {
    $membership = $user->classrooms()
                       ->where('classroom_id', $classroomId)
                       ->first();

    if (! $membership) {
        return false; // tolak — user bukan anggota classroom ini
    }

    // Data ini akan tersedia di JavaScript melalui Echo presence channel
    return [
        'id'   => $user->id,
        'name' => $user->name,
        'role' => $membership->pivot->role,
    ];
});
```

**Mengapa ada dua channel berbeda?**

- `classroom.{id}` (Private Channel) → digunakan oleh Model `Message` untuk broadcast pesan baru
- `presence-classroom.{id}` (Presence Channel) → digunakan khusus untuk melacak siapa yang sedang online; Presence Channel memerlukan awalan `presence-` di namanya agar Laravel Echo mengenalinya

### Langkah 7 — Menambahkan Relasi pada Model User

Buka `app/Models/User.php` dan tambahkan relasi ke Classroom:

```php
// app/Models/User.php

use Illuminate\Database\Eloquent\Relations\BelongsToMany;

// Tambahkan method berikut di dalam class User
public function classrooms(): BelongsToMany
{
    return $this->belongsToMany(Classroom::class)
                ->withPivot('role')
                ->withTimestamps();
}
```

### Langkah 8 — Membuat Controller

```bash
php artisan make:controller ClassroomController
php artisan make:controller MessageController
```

**`ClassroomController`:**

```php
<?php
// app/Http/Controllers/ClassroomController.php

namespace App\Http\Controllers;

use App\Models\Classroom;
use App\Models\User;
use Illuminate\Http\Request;

class ClassroomController extends Controller
{
    // [DIPERBAIKI] Gunakan Request injection untuk mengakses user yang sedang login.
    //
    // Mengapa tidak pakai auth()->user()?
    // auth()->user() mengembalikan tipe Authenticatable|null — PHP dan IDE tidak
    // dapat memastikan bahwa nilai tersebut bukan null meski route sudah dilindungi
    // middleware 'auth'. Akibatnya muncul error "Call to member function on null".
    //
    // $request->user() juga mengembalikan Authenticatable|null secara teknis,
    // namun dengan menambahkan @var User $user kita memberi tahu PHP bahwa
    // di titik ini user PASTI sudah terautentikasi (dijamin middleware 'auth').
    // Ini adalah pola yang umum digunakan di proyek Laravel produksi.

    public function index(Request $request)
    {
        /** @var User $user */
        $user = $request->user();

        // Tampilkan classroom yang dimiliki user yang sedang login
        $classrooms = $user->classrooms()->get();

        return view('classrooms.index', compact('classrooms'));
    }

    public function show(Request $request, Classroom $classroom)
    {
        /** @var User $user */
        $user = $request->user();

        // Pastikan user adalah anggota classroom ini
        abort_unless(
            $user->classrooms()->where('classroom_id', $classroom->id)->exists(),
            403,
            'Anda bukan anggota mata kuliah ini.'
        );

        // Ambil 50 pesan terakhir (urutan terlama ke terbaru untuk tampilan chat)
        $messages = $classroom->messages()
                              ->with('user')
                              ->latest()
                              ->take(50)
                              ->get()
                              ->reverse()
                              ->values();

        return view('classrooms.show', compact('classroom', 'messages'));
    }
}
```

**`MessageController`:**

> **[DIUBAH — KRITIS]** — Modul sebelumnya menggunakan GET route sebagai trigger broadcast. Ini anti-pattern. Pengiriman pesan harus menggunakan POST dengan validasi form yang benar.

```php
<?php
// app/Http/Controllers/MessageController.php

namespace App\Http\Controllers;

use App\Models\Classroom;
use App\Models\Message;
use App\Models\User;
use Illuminate\Http\Request;

class MessageController extends Controller
{
    // [DIUBAH] Menggunakan POST, bukan GET
    // [DIUBAH] Ada validasi input sebelum menyimpan
    // [DIPERBAIKI] Menggunakan $request->user() dengan type assertion
    public function store(Request $request, Classroom $classroom)
    {
        /** @var User $user */
        $user = $request->user();

        // Pastikan user adalah anggota classroom ini
        abort_unless(
            $user->classrooms()->where('classroom_id', $classroom->id)->exists(),
            403
        );

        $validated = $request->validate([
            'content' => ['required', 'string', 'max:1000'],
        ]);

        // Simpan pesan ke database
        // Model Broadcasting akan otomatis men-trigger broadcast
        // saat metode create() dipanggil (Eloquent 'created' event)
        $message = Message::create([
            'classroom_id' => $classroom->id,
            'user_id'      => $user->id,
            'content'      => $validated['content'],
        ]);

        // Load relasi user agar broadcastWith() dapat mengaksesnya
        $message->load('user');

        // Kembalikan response JSON — front-end tidak perlu reload halaman
        return response()->json([
            'status'  => 'sent',
            'message' => $message->only(['id', 'content', 'created_at']),
        ]);
    }
}
```

### Langkah 9 — Mendefinisikan Route

```php
<?php
// routes/web.php

use App\Http\Controllers\ClassroomController;
use App\Http\Controllers\MessageController;
use Illuminate\Support\Facades\Route;

Route::middleware(['auth'])->group(function () {
    Route::get('/classrooms', [ClassroomController::class, 'index'])
         ->name('classrooms.index');

    Route::get('/classrooms/{classroom}', [ClassroomController::class, 'show'])
         ->name('classrooms.show');

    // [DIUBAH] POST, bukan GET — sesuai prinsip HTTP yang benar
    Route::post('/classrooms/{classroom}/messages', [MessageController::class, 'store'])
         ->name('classrooms.messages.store');
});
```

### Langkah 10 — Membuat Tampilan Blade

> **[DITAMBAH]** — Modul sebelumnya hanya menampilkan output ke `console.log`. Berikut ini adalah tampilan UI nyata yang merender pesan secara real-time di DOM.

**View daftar classroom (`resources/views/classrooms/index.blade.php`):**

```html
{{-- resources/views/classrooms/index.blade.php --}}
<x-app-layout>
  <x-slot name="header">
    <h2 class="font-semibold text-xl text-gray-800 leading-tight">
      Mata Kuliah Saya
    </h2>
  </x-slot>

  <div class="py-12">
    <div class="max-w-4xl mx-auto sm:px-6 lg:px-8">
      @forelse($classrooms as $classroom)
      <a
        href="{{ route('classrooms.show', $classroom) }}"
        class="block bg-white rounded-lg shadow p-5 mb-4 hover:bg-indigo-50 transition"
      >
        <div class="font-semibold text-lg text-indigo-700">
          {{ $classroom->code }} — {{ $classroom->name }}
        </div>
        <div class="text-sm text-gray-500 mt-1">
          {{ $classroom->description }}
        </div>
      </a>
      @empty
      <div class="text-center text-gray-500 py-12">
        Anda belum terdaftar di mata kuliah manapun.
      </div>
      @endforelse
    </div>
  </div>
</x-app-layout>
```

**View ruang chat (`resources/views/classrooms/show.blade.php`):**

```html
{{-- resources/views/classrooms/show.blade.php --}}
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            {{ $classroom->code }} — {{ $classroom->name }}
        </h2>
    </x-slot>

    {{--
        [PENTING] Variabel config untuk JavaScript diletakkan di sini — SEBELUM konten halaman,
        bukan di @push('scripts').

        Alasan: chat.js di-bundle oleh Vite dan dimuat via tag @vite di layout utama.
        Jika variabel diletakkan di @push('scripts'), ada risiko chat.js sudah dieksekusi
        sebelum script @push di-render, sehingga SEND_URL, CLASSROOM_ID, dll. belum terdefinisi
        dan muncul error "ReferenceError: SEND_URL is not defined".

        Dengan meletakkan <script> ini di atas konten halaman, variabel sudah tersedia
        di window scope sebelum JavaScript bundle apapun dieksekusi.
    --}}
    <script>
        const CLASSROOM_ID  = {{ $classroom->id }};
        const AUTH_USER_ID  = {{ auth()->id() }};
        const SEND_URL      = "{{ route('classrooms.messages.store', $classroom) }}";
        const CSRF_TOKEN    = "{{ csrf_token() }}";
    </script>
    </x-slot>

    <div class="py-6">
        <div class="max-w-5xl mx-auto sm:px-6 lg:px-8 flex gap-4">

            {{-- Panel Chat Utama --}}
            <div class="flex-1 bg-white rounded-lg shadow flex flex-col" style="height: 75vh;">

                {{-- Header chat --}}
                <div class="px-4 py-3 border-b border-gray-200">
                    <p class="text-sm text-gray-500">Ruang Diskusi Real-Time</p>
                </div>

                {{-- Area pesan --}}
                <div id="chat-messages"
                     class="flex-1 overflow-y-auto p-4 space-y-3">

                    @foreach($messages as $message)
                        <div class="flex items-start gap-3 {{ $message->user_id === auth()->id() ? 'flex-row-reverse' : '' }}">
                            <div class="w-8 h-8 rounded-full bg-indigo-500 flex items-center justify-center text-white text-xs font-bold shrink-0">
                                {{ strtoupper(substr($message->user->name, 0, 1)) }}
                            </div>
                            <div class="{{ $message->user_id === auth()->id() ? 'items-end' : 'items-start' }} flex flex-col max-w-xs">
                                <span class="text-xs text-gray-400 mb-1">
                                    {{ $message->user->name }} · {{ $message->created_at->format('H:i') }}
                                </span>
                                <div class="px-3 py-2 rounded-lg text-sm
                                    {{ $message->user_id === auth()->id()
                                        ? 'bg-indigo-600 text-white'
                                        : 'bg-gray-100 text-gray-800' }}">
                                    {{ $message->content }}
                                </div>
                            </div>
                        </div>
                    @endforeach
                </div>

                {{-- Form input pesan --}}
                <div class="border-t border-gray-200 p-3">
                    <form id="message-form" class="flex gap-2">
                        @csrf
                        <input type="text"
                               id="message-input"
                               placeholder="Ketik pesan..."
                               autocomplete="off"
                               class="flex-1 rounded-lg border border-gray-300 px-3 py-2 text-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none"
                               required>
                        <button type="submit"
                                class="bg-indigo-600 hover:bg-indigo-700 text-white px-4 py-2 rounded-lg text-sm font-medium transition">
                            Kirim
                        </button>
                    </form>
                </div>
            </div>

            {{-- Sidebar: Member Online --}}
            <div class="w-56 bg-white rounded-lg shadow p-4" style="height: fit-content;">
                <h3 class="font-semibold text-sm text-gray-700 mb-3">
                    Sedang Online
                    <span id="online-count"
                          class="ml-1 bg-green-100 text-green-700 text-xs px-1.5 py-0.5 rounded-full">
                        0
                    </span>
                </h3>
                <ul id="online-members" class="space-y-2 text-sm text-gray-600">
                    {{-- Diisi oleh JavaScript --}}
                </ul>
            </div>

        </div>
    </div>
</x-app-layout>
```

### Langkah 11 — Mengintegrasikan Laravel Echo di JavaScript

> **[DIUBAH]** — File `resources/js/echo.js` sudah dibuat otomatis oleh `install:broadcasting`. Kita perlu menambahkan logika chat di file JavaScript terpisah yang dipanggil dari Blade.

**Pertama, buat file kosong `chat.js` terlebih dahulu:**

```bash
touch resources/js/chat.js
```

> **Penting:** file ini harus dibuat sebelum menjalankan `npm run build`. Jika file belum ada, Vite akan gagal dengan error `Could not resolve "./chat"` saat proses bundling.

Kemudian isi file `resources/js/chat.js` dengan kode berikut:

```javascript
// resources/js/chat.js

import "./echo"; // pastikan Echo sudah terinisialisasi

// Seluruh logika chat dibungkus dalam kondisi ini.
// chat.js dimuat di semua halaman karena di-import di app.js,
// tapi kode di dalamnya hanya berjalan jika elemen #chat-messages ada di DOM.
// Dengan cara ini tidak ada error yang dilempar ke halaman lain.
if (document.getElementById("chat-messages")) {
  // ─── Referensi elemen DOM ───────────────────────────────────────────────────
  const chatMessages = document.getElementById("chat-messages");
  const messageForm = document.getElementById("message-form");
  const messageInput = document.getElementById("message-input");
  const onlineMembers = document.getElementById("online-members");
  const onlineCount = document.getElementById("online-count");

  // ─── Helper: render satu pesan ke DOM ──────────────────────────────────────
  function renderMessage(data, isSelf = false) {
    const wrapper = document.createElement("div");
    wrapper.className = `flex items-start gap-3 ${isSelf ? "flex-row-reverse" : ""}`;

    const initial = data.user_name.charAt(0).toUpperCase();
    const bubbleColor = isSelf
      ? "bg-indigo-600 text-white"
      : "bg-gray-100 text-gray-800";

    wrapper.innerHTML = `
        <div class="w-8 h-8 rounded-full bg-indigo-500 flex items-center justify-center text-white text-xs font-bold shrink-0">
            ${initial}
        </div>
        <div class="${isSelf ? "items-end" : "items-start"} flex flex-col max-w-xs">
            <span class="text-xs text-gray-400 mb-1">
                ${data.user_name} · ${data.created_at}
            </span>
            <div class="px-3 py-2 rounded-lg text-sm ${bubbleColor}">
                ${escapeHtml(data.content)}
            </div>
        </div>
    `;

    chatMessages.appendChild(wrapper);
    chatMessages.scrollTop = chatMessages.scrollHeight;
  }

  // Helper: hindari XSS
  function escapeHtml(text) {
    const div = document.createElement("div");
    div.appendChild(document.createTextNode(text));
    return div.innerHTML;
  }

  // ─── Kirim pesan via fetch (AJAX) ──────────────────────────────────────────
  messageForm.addEventListener("submit", async (e) => {
    e.preventDefault();

    const content = messageInput.value.trim();
    if (!content) return;

    messageInput.value = "";
    messageInput.focus();

    try {
      await fetch(SEND_URL, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "X-CSRF-TOKEN": CSRF_TOKEN,
          Accept: "application/json",
        },
        body: JSON.stringify({ content }),
      });
      // Pesan yang dikirim sendiri akan muncul melalui broadcast,
      // bukan langsung di sini, agar konsisten dengan pesan dari user lain.
    } catch (err) {
      console.error("Gagal mengirim pesan:", err);
    }
  });

  // ─── Private Channel: Menerima pesan baru ──────────────────────────────────
  //
  // [DIUBAH] Menggunakan Private Channel, bukan Public Channel
  // Nama event: '.MessageCreated' — awalan titik menunjukkan ini event dari Model Broadcasting
  // Format nama event Model Broadcasting: .<NamaModel><NamaEventEloquent>
  //
  Echo.private(`classroom.${CLASSROOM_ID}`).listen(".MessageCreated", (e) => {
    // e berisi data dari broadcastWith() di model Message
    const isSelf = e.user_id === AUTH_USER_ID;
    renderMessage(e, isSelf);
  });

  // ─── Presence Channel: Melacak siapa yang online ───────────────────────────
  //
  // [DITAMBAH] Presence Channel memberikan tiga callback:
  //   here()    — dipanggil sekali saat pertama bergabung, berisi semua member
  //   joining() — dipanggil setiap kali ada member baru masuk
  //   leaving() — dipanggil setiap kali ada member yang keluar
  //
  let onlineUsers = {};

  function updateOnlineList() {
    const users = Object.values(onlineUsers);
    onlineCount.textContent = users.length;
    onlineMembers.innerHTML = users
      .map(
        (u) => `
        <li class="flex items-center gap-2">
            <span class="w-2 h-2 bg-green-400 rounded-full"></span>
            <span>${escapeHtml(u.name)}</span>
            ${u.role === "lecturer" ? '<span class="text-xs text-indigo-500">(Dosen)</span>' : ""}
        </li>
    `,
      )
      .join("");
  }

  Echo.join(`presence-classroom.${CLASSROOM_ID}`)
    .here((users) => {
      // Dipanggil sekali saat kita bergabung ke channel
      // 'users' adalah array semua member yang sedang online
      onlineUsers = {};
      users.forEach((u) => {
        onlineUsers[u.id] = u;
      });
      updateOnlineList();
    })
    .joining((user) => {
      // Dipanggil saat ada member baru bergabung
      onlineUsers[user.id] = user;
      updateOnlineList();
    })
    .leaving((user) => {
      // Dipanggil saat ada member yang menutup halaman
      delete onlineUsers[user.id];
      updateOnlineList();
    })
    .error((error) => {
      console.error("Presence channel error:", error);
    });
} // akhir if (document.getElementById('chat-messages'))
```

Setelah `chat.js` berisi kode di atas, daftarkan sebagai import di `resources/js/app.js`:

```javascript
// resources/js/app.js
import "./bootstrap";
import "./chat"; // [DITAMBAH]
```

Baru kemudian jalankan build — pastikan urutan ini diikuti: **buat file → isi kode → tambah import → build**:

```bash
npm run build
```

### Langkah 12 — Menjalankan Semua Service yang Diperlukan

> **[DITAMBAH — KRITIS]** — Ini adalah salah satu poin yang tidak ada di modul sebelumnya. Sistem broadcasting memerlukan **tiga proses** yang berjalan secara bersamaan. Buka tiga jendela terminal secara terpisah.

**Terminal 1 — Server Laravel (PHP):**

```bash
php artisan serve
```

**Terminal 2 — Server WebSocket Reverb:**

```bash
# --debug menampilkan log koneksi WebSocket secara real-time
php artisan reverb:start --debug
```

**Terminal 3 — Queue Worker:**

> **[DITAMBAH]** — Queue Worker adalah proses yang mengambil job dari antrian database dan menjalankannya, termasuk mengirim event broadcast ke Reverb. Tanpa ini, pesan yang dikirim tidak akan pernah di-broadcast meski sudah tersimpan di database.

```bash
php artisan queue:work
```

**Terminal 4 (opsional) — Vite Dev Server untuk development:**

```bash
npm run dev
```

**Penjelasan alur kerja keempat proses:**

```
Browser ──[POST /messages]──► Laravel (artisan serve)
                                    │
                                    ▼ Message::create() dipanggil
                              Eloquent 'created' event
                                    │
                                    ▼ BroadcastsEvents men-dispatch job ke queue
                              Queue Table (database)
                                    │
                              Queue Worker mengambil job
                                    │
                                    ▼
                              Reverb WebSocket Server
                                    │
                              ──[WebSocket push]──► Semua Browser yang tersubscribe
```

### Langkah 13 — Membuat Data Seeder untuk Pengujian

Buat seeder agar mudah menguji fitur tanpa mengisi data manual:

```bash
php artisan make:seeder ClassroomSeeder
```

```php
<?php
// database/seeders/ClassroomSeeder.php

namespace Database\Seeders;

use App\Models\Classroom;
use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class ClassroomSeeder extends Seeder
{
    public function run(): void
    {
        // Buat 3 user: 1 dosen, 2 mahasiswa
        // Password di-hash menggunakan Hash::make() — JANGAN simpan password plaintext di database.
        // Hash::make() menggunakan algoritma bcrypt secara default, sesuai konfigurasi Laravel.
        $lecturer = User::factory()->create([
            'name'     => 'Dosen Informatika',
            'email'    => 'dosen@polije.ac.id',
            'password' => Hash::make('password123'),
        ]);

        $student1 = User::factory()->create([
            'name'     => 'Mahasiswa A',
            'email'    => 'mahasiswa.a@polije.ac.id',
            'password' => Hash::make('password123'),
        ]);

        $student2 = User::factory()->create([
            'name'     => 'Mahasiswa B',
            'email'    => 'mahasiswa.b@polije.ac.id',
            'password' => Hash::make('password123'),
        ]);

        // Buat 1 classroom
        $classroom = Classroom::create([
            'name'        => 'Workshop Sistem Informasi Web Framework',
            'code'        => 'TI-401',
            'description' => 'Praktikum Web Framework D4 Teknik Informatika',
        ]);

        // Daftarkan anggota ke classroom
        $classroom->users()->attach($lecturer->id,  ['role' => 'lecturer']);
        $classroom->users()->attach($student1->id,  ['role' => 'student']);
        $classroom->users()->attach($student2->id,  ['role' => 'student']);
    }
}
```

Jalankan seeder:

```bash
php artisan db:seed --class=ClassroomSeeder
```

### Langkah 14 — Pengujian Fitur Real-Time

**Kredensial login yang dibuat oleh seeder:**

| Nama              | Email                      | Password      |
| ----------------- | -------------------------- | ------------- |
| Dosen Informatika | `dosen@polije.ac.id`       | `password123` |
| Mahasiswa A       | `mahasiswa.a@polije.ac.id` | `password123` |
| Mahasiswa B       | `mahasiswa.b@polije.ac.id` | `password123` |

1. Buka browser pertama, login sebagai `mahasiswa.a@polije.ac.id` dengan password `password123`
2. Buka browser kedua (mode incognito atau browser berbeda), login sebagai `mahasiswa.b@polije.ac.id` dengan password `password123`
3. Pada kedua browser, buka halaman `http://localhost:8000/classrooms`
4. Klik classroom **TI-401** di kedua browser
5. Perhatikan sidebar "Sedang Online" — kedua nama mahasiswa harus muncul
6. Ketik dan kirim pesan dari browser pertama
7. Pesan harus muncul di browser kedua **tanpa refresh halaman**
8. Tutup salah satu browser — nama user yang menutup halaman harus hilang dari daftar online

**Verifikasi di Terminal Reverb:**

Saat koneksi WebSocket berhasil, terminal Reverb (--debug) akan menampilkan log seperti:

```
INFO  Server running at 0.0.0.0:8080

  [2026-04-20 07:00:00] Connection id 1 established.
  [2026-04-20 07:00:01] Connection id 2 established.
  [2026-04-20 07:00:05] Message received on connection 1.
  [2026-04-20 07:00:05] Broadcasting to channel private-classroom.1
```

---

## g. Ringkasan Perbedaan dengan Modul Sebelumnya

| Aspek                       | Modul Lama                     | Modul Baru (Laravel 12)                       |
| --------------------------- | ------------------------------ | --------------------------------------------- |
| Trigger broadcast           | GET route `/send-message`      | POST via Controller + validasi                |
| Jenis channel               | Public saja                    | Public, Private, Presence                     |
| Return type `broadcastOn()` | Tanpa type hint                | `array` (PHP 8.2+)                            |
| Interface                   | `ShouldBroadcastNow` (sinkron) | `ShouldBroadcast` + Queue Worker              |
| Nama event di JS            | FQCN panjang                   | Custom via `broadcastAs()`                    |
| Payload                     | Semua public property          | Dikontrol via `broadcastWith()`               |
| Otorisasi channel           | Tidak ada (`return true`)      | Berbasis Eloquent + user membership           |
| Pembuatan event             | Manual Event class             | Model Broadcasting (`BroadcastsEvents`)       |
| Output front-end            | `console.log` saja             | Tampilan UI Blade + JavaScript fetch          |
| Proses server               | 1 (artisan serve)              | 3 (artisan serve + reverb:start + queue:work) |

---

## h. Kesimpulan

Pada praktikum ini mahasiswa telah berhasil membangun sistem chat ruang kelas real-time menggunakan Laravel Reverb dan Event Broadcasting di Laravel 12. Beberapa konsep penting yang telah dipelajari:

1. **Tiga jenis channel** (Public, Private, Presence) memiliki fungsi dan tingkat keamanan yang berbeda. Pemilihan channel yang tepat adalah bagian dari desain sistem, bukan hanya implementasi.

2. **Channel Authorization** di `routes/channels.php` adalah lapisan keamanan fundamental yang memastikan hanya pengguna yang berhak dapat berlangganan ke sebuah channel.

3. **Model Broadcasting** dengan trait `BroadcastsEvents` menyederhanakan implementasi dengan menghilangkan kebutuhan membuat Event class terpisah untuk setiap perubahan model.

4. **`broadcastAs()` dan `broadcastWith()`** adalah praktik keamanan yang baik untuk mengontrol nama event dan data apa saja yang boleh diterima oleh klien JavaScript.

5. **Queue Worker** adalah komponen wajib saat menggunakan `ShouldBroadcast` (bukan `ShouldBroadcastNow`). Tanpa queue worker, event broadcast tidak akan pernah terkirim ke Reverb.

6. **Presence Channel** memungkinkan fitur "siapa yang sedang online" yang membangun pengalaman pengguna kolaboratif dan interaktif.

---
