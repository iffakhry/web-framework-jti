# Deployment Aplikasi Laravel 12 ke Server (Shared Hosting & VPS)

## a. Dasar Teori

### 1. Apa itu Deployment?

**Deployment** adalah proses memindahkan aplikasi dari lingkungan pengembangan (development) ke lingkungan produksi (production) agar dapat diakses pengguna akhir melalui internet. Dalam konteks Laravel, deployment mencakup:

- Transfer file proyek ke server
- Konfigurasi environment produksi (`.env`)
- Instalasi dependency (`composer install`)
- Kompilasi asset frontend (`npm run build`)
- Migrasi dan seeding database
- Optimasi caching framework
- Konfigurasi web server dan domain

### 2. Perbandingan Shared Hosting vs VPS

| Aspek                  | Shared Hosting                  | VPS (Virtual Private Server)            |
| ---------------------- | ------------------------------- | --------------------------------------- |
| **Akses server**       | Terbatas (cPanel/hPanel)        | Penuh (root SSH)                        |
| **Resource**           | Dibagi dengan user lain         | Terdedikasi/terjamin                    |
| **Harga**              | Lebih murah (Rp 15rb–100rb/bln) | Lebih mahal (Rp 50rb–500rb/bln)         |
| **Skalabilitas**       | Rendah                          | Tinggi                                  |
| **Konfigurasi server** | Tidak bisa                      | Bebas (Nginx, PHP versi, dll.)          |
| **Cocok untuk**        | Proyek kecil, demo, belajar     | Proyek production, skala menengah-besar |
| **Composer via SSH**   | Kadang tersedia (terbatas)      | Selalu tersedia                         |
| **Contoh provider**    | Niagahoster, Hostinger Shared   | DigitalOcean, Vultr, Hostinger VPS      |

### 3. Persyaratan Server Laravel 12

Laravel 12 memerlukan:

- **PHP ≥ 8.2** (direkomendasikan PHP 8.3)
- Ekstensi PHP: `Ctype`, `cURL`, `DOM`, `Fileinfo`, `Filter`, `Hash`, `Mbstring`, `OpenSSL`, `PCRE`, `PDO`, `Session`, `Tokenizer`, `XML`
- **Composer** (dependency manager PHP)
- **MySQL 8.0+** atau MariaDB 10.10+
- Web server: **Nginx** (VPS) atau **Apache** (shared hosting)

### 4. Struktur Direktori Laravel yang Penting untuk Deployment

```
my-laravel-app/
├── app/              ← Logika aplikasi (TIDAK diekspos ke publik)
├── bootstrap/
│   └── cache/        ← Harus writable (permission 775)
├── config/           ← Konfigurasi (TIDAK diekspos ke publik)
├── database/
├── public/           ← INI yang menjadi Document Root / Web Root
│   ├── index.php     ← Entry point aplikasi
│   └── .htaccess
├── resources/
├── routes/
├── storage/          ← Harus writable (permission 775)
│   ├── app/
│   ├── framework/
│   └── logs/
├── vendor/           ← Hasil composer install
└── .env              ← Konfigurasi environment (JANGAN diekspos publik)
```

> **⚠️ PENTING:** Hanya folder `public/` yang boleh dijadikan web root. Folder lainnya (terutama `.env`, `config/`, `app/`) TIDAK boleh dapat diakses langsung dari browser.

### 5. Perintah Optimasi Production

Laravel menyediakan perintah untuk mengoptimalkan aplikasi sebelum deployment:

```bash
# Jalankan semua optimasi sekaligus (Laravel 12)
php artisan optimize

# Atau secara manual:
php artisan config:cache    # Cache konfigurasi .env
php artisan route:cache     # Cache routing
php artisan view:cache      # Cache Blade template
php artisan event:cache     # Cache event listeners (baru di Laravel 12)
```

---

## b. Alat dan Bahan

**Perangkat Keras:**

- PC / Laptop (RAM minimal 4 GB, rekomendasi 8 GB)
- Koneksi internet

**Perangkat Lunak:**

- Git (https://git-scm.com/downloads)
- Terminal / Command Prompt / PowerShell
- Text editor (VS Code)
- FileZilla FTP Client (opsional, untuk shared hosting alternatif)
- Termius / PuTTY (untuk koneksi SSH ke VPS)

**Akun / Layanan:**

- Akun shared hosting dengan cPanel (disediakan instruktur)
- **ATAU** akun VPS Ubuntu 24.04 LTS dengan akses SSH root
- Repository Git (GitHub / GitLab)
- Proyek Laravel 12 yang sudah siap di-deploy

---

## c. Persiapan Sebelum Deployment (Wajib untuk Semua Metode)

Langkah-langkah ini dilakukan di komputer lokal sebelum upload ke server.

### Langkah 1 — Pastikan Proyek Berjalan di Lokal

```bash
# Cek versi Laravel
php artisan --version
# Output: Laravel Framework 12.x.x

# Cek versi PHP
php --version
# Output: PHP 8.2.x atau lebih baru
```

### Langkah 2 — Buat File `.env` untuk Production

Buat salinan `.env` khusus production. Jangan copy langsung `.env` development.

```env
APP_NAME="Nama Aplikasi Saya"
APP_ENV=production
APP_KEY=base64:xxxx...  # Akan di-generate di server
APP_DEBUG=false          # WAJIB false di production!
APP_URL=https://domain-anda.com

LOG_CHANNEL=daily
LOG_LEVEL=error

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=nama_database
DB_USERNAME=nama_user_db
DB_PASSWORD=password_db_kuat

FILESYSTEM_DISK=public
SESSION_DRIVER=file
CACHE_STORE=file
QUEUE_CONNECTION=database
```

### Langkah 3 — Kompilasi Asset Frontend

```bash
# Install dependency Node.js (jika belum)
npm install

# Build untuk production (Vite)
npm run build
```

> Perintah ini menghasilkan folder `public/build/` yang berisi CSS dan JS yang sudah diminifikasi. **Pastikan folder ini ikut diupload ke server.**

### Langkah 4 — Persiapkan Composer untuk Production

```bash
# Hapus dev-dependency, optimasi autoloader
composer install --optimize-autoloader --no-dev
```

### Langkah 5 — Buat Repository Git (Jika Belum Ada)

```bash
cd /path/to/proyek-laravel

# Inisialisasi Git
git init

# Pastikan .gitignore sudah benar (Laravel sudah include ini secara default)
# File yang TIDAK perlu di-push: /vendor, /node_modules, .env, /public/build
cat .gitignore

# Tambahkan semua file
git add .
git commit -m "Initial production commit"

# Push ke GitHub/GitLab
git remote add origin https://github.com/username/nama-repo.git
git push -u origin main
```

---

## d. Prosedur Kerja — Bagian A: Deployment ke Shared Hosting

### Topologi Shared Hosting

```
Browser Pengguna
      │
      ▼
 [Domain/Subdomain]
      │
      ▼
 [Apache Web Server]  ← dikonfigurasi oleh hosting provider
      │
      ▼
 [public_html/]  ← Document Root hosting
      │ (perlu diarahkan ke)
      ▼
 [Laravel public/]  ← Entry point aplikasi
```

### A.1 — Siapkan Database di cPanel

1. Login ke **cPanel** hosting Anda.
2. Cari menu **MySQL Databases** atau **Database Wizard**.
3. Buat database baru:
   - **Database name:** `namauser_laraveldb` (prefix otomatis dari hosting)
4. Buat user database baru:
   - **Username:** `namauser_dbuser`
   - **Password:** buat password kuat (gunakan generator)
5. Tambahkan user ke database → beri semua privilege (**ALL PRIVILEGES**).
6. **Catat** nama database, username, dan password — akan digunakan di `.env`.

### A.2 — Upload File Proyek via Git (Metode Direkomendasikan)

Jika shared hosting menyediakan **SSH Terminal** (biasanya di cPanel → Terminal atau SSH Access):

```bash
# Masuk ke direktori home
cd ~

# Clone repository dari GitHub
git clone https://github.com/username/nama-repo.git nama-folder

# Masuk ke folder proyek
cd nama-folder

# Install dependency (pastikan PHP dan Composer tersedia)
composer install --optimize-autoloader --no-dev
```

### A.3 — Upload File via FTP (Metode Alternatif)

Jika SSH tidak tersedia, gunakan **FileZilla** atau **cPanel File Manager**:

1. Buka FileZilla → masukkan kredensial FTP dari cPanel.
2. Upload **seluruh isi folder proyek Laravel** ke direktori pilihan, misalnya `/home/namauser/laravel/`.
   - Pastikan termasuk folder `public/build/` (hasil `npm run build`).
   - Upload juga folder `vendor/` (hasil `composer install` lokal).
3. Jangan upload: `.env` (nanti dibuat manual di server), `node_modules/`.

### A.4 — Konfigurasi Document Root (Arahkan Domain ke `public/`)

Ini adalah langkah paling kritis di shared hosting. Ada dua cara:

#### Cara 1 — Subdomain / Addon Domain (Direkomendasikan)

Di cPanel:

1. Buka **Subdomains** atau **Addon Domains**.
2. Buat subdomain/domain baru.
3. Pada kolom **Document Root**, arahkan langsung ke folder `public/` dari proyek Laravel:
   ```
   /home/namauser/laravel/public
   ```
4. Klik **Add Subdomain / Add Domain**.

Cara ini adalah yang **paling aman** karena folder di luar `public/` tidak dapat diakses browser sama sekali.

#### Cara 2 — `.htaccess` Redirect (Jika Harus Pakai public_html)

Jika proyek harus ditempatkan di dalam `public_html/`, buat file `.htaccess` di root `public_html/`:

```apache
# File: public_html/.htaccess
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^(.*)$ laravel/public/$1 [L]
</IfModule>
```

Lalu tambahkan pengamanan `.env` di file `.htaccess` yang sama:

```apache
# Lindungi file sensitif
<Files .env>
    Order allow,deny
    Deny from all
</Files>

<Files composer.json>
    Order allow,deny
    Deny from all
</Files>

# Nonaktifkan directory listing
Options -Indexes
```

### A.5 — Konfigurasi File `.env` di Server

1. Di cPanel File Manager, navigasi ke folder root proyek Laravel.
2. Buat file baru bernama `.env` (atau upload dari komputer lokal).
3. Isi dengan konfigurasi production:

```env
APP_NAME="Nama Aplikasi"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://subdomain.domain.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=namauser_laraveldb
DB_USERNAME=namauser_dbuser
DB_PASSWORD=password_database_anda

FILESYSTEM_DISK=public
SESSION_DRIVER=file
CACHE_STORE=file
```

### A.6 — Generate Application Key

Melalui SSH Terminal di cPanel:

```bash
cd ~/laravel
php artisan key:generate
```

Jika tidak ada SSH, generate key secara manual:

```bash
# Di komputer lokal
php artisan key:generate --show
# Salin output: base64:xxxxxxxxxxxxxxxxxxxxxxxx=
```

Lalu tempel nilai tersebut di `.env` server pada baris `APP_KEY=`.

### A.7 — Jalankan Migrasi Database

```bash
# Via SSH Terminal cPanel
cd ~/laravel
php artisan migrate --force
```

Flag `--force` diperlukan karena Laravel akan meminta konfirmasi di environment production.

Jika ingin sekalian jalankan seeder:

```bash
php artisan migrate --seed --force
```

### A.8 — Buat Storage Link

```bash
php artisan storage:link
```

Perintah ini membuat symbolic link dari `public/storage` → `storage/app/public`, sehingga file yang diupload pengguna dapat diakses via URL.

### A.9 — Set Permission Folder

```bash
# Via SSH
chmod -R 755 storage
chmod -R 755 bootstrap/cache

# Atau lebih ketat
find storage -type d -exec chmod 775 {} \;
find storage -type f -exec chmod 664 {} \;
find bootstrap/cache -type d -exec chmod 775 {} \;
```

### A.10 — Jalankan Optimasi Production

```bash
php artisan optimize
```

Atau secara terpisah:

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
```

### A.11 — Verifikasi Deployment Shared Hosting

Buka browser dan akses domain/subdomain Anda. Periksa:

- [ ] Halaman utama terbuka tanpa error
- [ ] Login/registrasi berfungsi (koneksi database OK)
- [ ] Upload file berfungsi (storage link OK)
- [ ] File `.env` TIDAK bisa diakses via browser (`https://domain.com/.env` → 403 Forbidden)
- [ ] Tidak ada folder listing yang terbuka

---

## e. Prosedur Kerja — Bagian B: Deployment ke VPS (Ubuntu 24.04 LTS + Nginx)

### Topologi VPS

```
Browser Pengguna
      │
      ▼
 [Domain → IP Publik VPS]
      │
      ▼
 [Nginx Web Server]  ← kita konfigurasikan sendiri
      │
      ▼
 [PHP-FPM 8.3]  ← kita install sendiri
      │
      ▼
 [Laravel 12 → /var/www/myapp/public]
      │
      ▼
 [MySQL 8.0]  ← kita install sendiri
```

### B.1 — Koneksi SSH ke VPS

```bash
# Dari terminal komputer lokal
ssh root@IP_VPS_ANDA

# Contoh
ssh root@103.x.x.x
```

> **Rekomendasi Keamanan:** Setelah login pertama kali, segera buat user non-root:
>
> ```bash
> adduser deployer
> usermod -aG sudo deployer
> ```
>
> Selanjutnya gunakan user `deployer` untuk semua operasi.

### B.2 — Update Sistem

```bash
apt update && apt upgrade -y
```

### B.3 — Install PHP 8.3 dan Ekstensi yang Diperlukan

```bash
# Install repository PHP terbaru
apt install -y software-properties-common
add-apt-repository ppa:ondrej/php -y
apt update

# Install PHP 8.3 dan ekstensi Laravel
apt install -y php8.3 php8.3-fpm php8.3-cli \
    php8.3-mysql php8.3-mbstring php8.3-xml \
    php8.3-curl php8.3-zip php8.3-bcmath \
    php8.3-tokenizer php8.3-ctype php8.3-fileinfo \
    php8.3-dom php8.3-opcache

# Cek versi PHP
php --version
```

### B.4 — Install Composer

```bash
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
chmod +x /usr/local/bin/composer

# Verifikasi
composer --version
```

### B.5 — Install Nginx

```bash
apt install -y nginx

# Aktifkan dan jalankan Nginx
systemctl enable nginx
systemctl start nginx
systemctl status nginx
```

### B.6 — Install MySQL 8.0

```bash
apt install -y mysql-server

# Amankan instalasi MySQL
mysql_secure_installation

# Login ke MySQL
mysql -u root -p

# Buat database dan user untuk Laravel
CREATE DATABASE laravel_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'laravel_user'@'localhost' IDENTIFIED BY 'PasswordKuat123!';
GRANT ALL PRIVILEGES ON laravel_db.* TO 'laravel_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### B.7 — Install Git dan Clone Repository

```bash
apt install -y git

# Pindah ke direktori web
cd /var/www/

# Clone repository Laravel
git clone https://github.com/username/nama-repo.git myapp

# Masuk ke folder proyek
cd myapp
```

### B.8 — Install Dependency Composer

```bash
# Di dalam folder proyek
composer install --optimize-autoloader --no-dev
```

### B.9 — Konfigurasi File `.env`

```bash
# Salin dari file example
cp .env.example .env

# Edit file .env
nano .env
```

Isi konfigurasi production:

```env
APP_NAME="Aplikasi Laravel Saya"
APP_ENV=production
APP_KEY=                     # Akan di-generate
APP_DEBUG=false
APP_URL=http://domain-atau-ip-anda.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_db
DB_USERNAME=laravel_user
DB_PASSWORD=PasswordKuat123!

FILESYSTEM_DISK=public
SESSION_DRIVER=file
CACHE_STORE=file
```

Simpan dan keluar (`Ctrl+X` → `Y` → `Enter`).

### B.10 — Generate Application Key

```bash
php artisan key:generate
```

### B.11 — Set Permission File dan Folder

```bash
# Set kepemilikan ke user web server
chown -R www-data:www-data /var/www/myapp

# Set permission
chmod -R 755 /var/www/myapp
chmod -R 775 /var/www/myapp/storage
chmod -R 775 /var/www/myapp/bootstrap/cache
```

### B.12 — Jalankan Migrasi Database

```bash
php artisan migrate --force
# Jika ada seeder:
# php artisan migrate --seed --force
```

### B.13 — Buat Storage Link

```bash
php artisan storage:link
```

### B.14 — Konfigurasi Nginx Virtual Host

Buat file konfigurasi Nginx untuk proyek:

```bash
nano /etc/nginx/sites-available/myapp
```

Isi dengan konfigurasi berikut (sesuai dokumentasi resmi Laravel 12):

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name domain-anda.com www.domain-anda.com;
    # Jika menggunakan IP langsung:
    # server_name 103.x.x.x;

    root /var/www/myapp/public;
    index index.php;

    charset utf-8;

    # Header keamanan
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    # Lindungi file sensitif
    location ~ /\. {
        deny all;
    }

    location ~ /\.env {
        deny all;
    }

    error_page 404 /index.php;

    # Proses PHP via PHP-FPM
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }
}
```

### B.15 — Aktifkan Virtual Host

```bash
# Buat symlink ke sites-enabled
ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# Hapus konfigurasi default (opsional)
rm /etc/nginx/sites-enabled/default

# Test konfigurasi Nginx
nginx -t

# Reload Nginx
systemctl reload nginx
```

### B.16 — Jalankan Optimasi Production

```bash
php artisan optimize
```

### B.17 — Verifikasi Deployment VPS

```bash
# Cek status PHP-FPM
systemctl status php8.3-fpm

# Cek status Nginx
systemctl status nginx

# Cek log error (jika ada masalah)
tail -n 50 /var/log/nginx/error.log
tail -n 50 /var/www/myapp/storage/logs/laravel.log
```

Buka browser dan akses `http://IP_VPS` atau `http://domain-anda.com`. Periksa:

- [ ] Halaman utama terbuka
- [ ] Database terhubung (coba login/registrasi)
- [ ] Storage link berfungsi (coba upload file)
- [ ] `.env` tidak bisa diakses via browser

---

## f. Langkah Bonus — Deploy Ulang (Update Kode)

Ketika ada perubahan kode, proses update di server menjadi lebih mudah dengan Git:

### Update di VPS

```bash
cd /var/www/myapp

# Aktifkan mode maintenance (pengguna lihat halaman "Sedang dalam pemeliharaan")
php artisan down --refresh=15 --retry=30

# Ambil kode terbaru dari repository
git pull origin main

# Update dependency jika ada perubahan composer.json
composer install --optimize-autoloader --no-dev

# Jalankan migrasi terbaru (jika ada)
php artisan migrate --force

# Rebuild cache
php artisan optimize

# Matikan mode maintenance
php artisan up
```

### Update di Shared Hosting (via SSH)

```bash
cd ~/laravel
php artisan down
git pull origin main
composer install --optimize-autoloader --no-dev
php artisan migrate --force
php artisan optimize
php artisan up
```

---

## g. Troubleshooting Umum

| Gejala Error                       | Kemungkinan Penyebab                                            | Solusi                                                                                 |
| ---------------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `500 Internal Server Error`        | `APP_DEBUG=true` (lihat detail) atau permission salah           | Set `APP_DEBUG=true` sementara untuk debug, cek `storage/logs/laravel.log`             |
| `419 Page Expired`                 | Session tidak berjalan atau CSRF token tidak valid              | Cek konfigurasi `SESSION_DRIVER` di `.env`, jalankan `php artisan config:cache`        |
| Halaman terbuka tapi CSS/JS hilang | `APP_URL` salah atau asset tidak ter-compile                    | Cek `APP_URL` di `.env`, pastikan `npm run build` sudah dijalankan                     |
| `Class not found`                  | `composer install` belum dijalankan atau autoload belum di-dump | Jalankan `composer install` lalu `composer dump-autoload`                              |
| File upload tidak muncul           | `storage:link` belum dibuat                                     | Jalankan `php artisan storage:link`                                                    |
| Database connection refused        | Kredensial DB salah atau DB tidak aktif                         | Verifikasi isi `.env` DB section, cek `systemctl status mysql`                         |
| `.env` bisa diakses dari browser   | Konfigurasi web server salah                                    | Tambahkan proteksi `.htaccess` (Apache) atau `location ~ /\.env { deny all; }` (Nginx) |
| `php artisan` tidak dikenal        | PHP tidak terinstall atau path salah                            | Cek `php --version`, pastikan PHP sudah di PATH                                        |

---

## h. Perbandingan Workflow Deployment

```
SHARED HOSTING                          VPS
────────────────────────────────        ─────────────────────────────────
1. Buat DB di cPanel                    1. Install stack (Nginx, PHP, MySQL)
2. Upload file (FTP/Git/cPanel)         2. Clone repository via Git
3. Konfigurasi .env                     3. Konfigurasi .env
4. Arahkan domain ke /public            4. Konfigurasi Nginx virtual host
5. php artisan key:generate             5. php artisan key:generate
6. php artisan migrate --force          6. php artisan migrate --force
7. php artisan storage:link             7. php artisan storage:link
8. php artisan optimize                 8. php artisan optimize
9. Set permission                       9. Set permission & ownership
10. Verifikasi                          10. Verifikasi
```

---

## n. Referensi

1. Laravel Documentation — Deployment. (2025). https://laravel.com/docs/12.x/deployment
2. Laravel Starter Kits. (2025). _How to Deploy Laravel to Shared Hosting (Step by Step)_. https://1v0.net/blog/how-to-deploy-laravel-to-shared-hosting-step-by-step/
3. Lemanceau, L. (2024). _Nginx Configuration for Laravel_. LEMP Stack Guide.
4. PHP Foundation. (2025). _PHP 8.3 Release Notes_. https://www.php.net/releases/8.3/
