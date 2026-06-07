# AutoDeploy Sistem Informasi Institut Teknologi Kalimantan — Panduan Lengkap

> Platform auto-hosting untuk mahasiswa ITK. Push ke GitHub → otomatis live di server.

---

## Daftar Isi

1. [Gambaran Sistem](#gambaran-sistem)
2. [File yang Dibutuhkan](#file-yang-dibutuhkan)
3. [Setup GitHub Repository](#setup-github-repository)
4. [Cara Deploy Pertama Kali](#cara-deploy-pertama-kali)
5. [Install cloudflared](#langkah-1--install-cloudflared-sekali-saja)
6. [Login SSH ke Server](#langkah-2--login-ssh-ke-server)
7. [Setup Laravel](#langkah-3--setup-laravel)
8. [Cara Update Kode](#cara-update-kode)
9. [Perintah Berguna](#perintah-berguna-di-server)
10. [Troubleshooting](#troubleshooting)

---

## Gambaran Sistem

```
Mahasiswa push kode ke GitHub
        │
        ▼
GitHub Actions berjalan otomatis (3 tahapan)
  [1] Detect & Validate
      • Deteksi framework (Laravel / PHP Native)
      • Generate slug nama repo
      • Pemeriksaan sintaks PHP
        │
        ▼
  [2] Package & Deploy
      • Buat ZIP dari kode mahasiswa (.env dikecualikan)
      • Kirim ke server via HTTPS
      • Ekstrak file ke /www/wwwroot/hosting/<repo>
      • Buat user SSH khusus untuk mahasiswa
      • Setup Nginx, Systemd service
      • Daftarkan subdomain di Cloudflare
        │
        ▼
  [3] Notification
      • Tampilkan domain, kredensial SSH, dan panduan
        di Job Summary GitHub Actions
        │
        ▼
Mahasiswa membaca Job Summary → login SSH → setup Laravel
  • Database & .env dibuat otomatis oleh sistem
  • Mahasiswa menjalankan composer install, migrate, restart service
        │
        ▼
Web mahasiswa live di:
  https://<nama-repo>.akhzafachrozy.my.id
```

> Buka GitHub Actions → tab Actions → job terbaru → Job Summary untuk melihat semua informasi dan panduan langkah selanjutnya.

---

## File yang Dibutuhkan

| File | Fungsi |
|------|--------|
| `deploy.yml` | Diletakkan di `.github/workflows/` pada repo mahasiswa |
| `cloudflaredclient.bat` | Install cloudflared di Windows (sekali saja) |
| `ssh.bat` | Login SSH di Windows |
| — | macOS/Linux: gunakan terminal biasa (lihat panduan di bawah) |

---

## Setup GitHub Repository

### Langkah 1 — Tambahkan workflow ke repo mahasiswa

Buat folder dan salin file `deploy.yml`:

```
nama-repo-mahasiswa/
├── .github/
│   └── workflows/
│       └── deploy.yml   ← taruh di sini
├── app/
├── ...
└── artisan
```

### Langkah 2 — Pastikan branch `main` atau `master` ada

Workflow hanya berjalan saat ada push ke branch `main` atau `master`.

### Langkah 3 — Push ke GitHub

```bash
git add .
git commit -m "first deploy"
git push origin main
```

---

## Cara Deploy Pertama Kali

1. Push kode ke GitHub (langkah di atas).
2. Buka **GitHub → tab Actions → workflow terbaru**.
3. Tunggu hingga semua job selesai (centang hijau).
4. Klik job **Package and Deploy** → lihat **Job Summary**.

Di Job Summary akan tersedia:
- Domain web
- Username & password SSH
- Semua langkah selanjutnya

> Selalu baca Job Summary setelah deploy — semua panduan ada di sana, disesuaikan otomatis dengan repo mahasiswa.

---

## Langkah 1 — Install cloudflared (sekali saja)

SSH ke server menggunakan Cloudflare Tunnel. Mahasiswa perlu install `cloudflared` terlebih dahulu.

### Windows

1. Download file `cloudflaredclient.bat` dari github
2. Klik kanan → Run as administrator
3. Tunggu hingga muncul "BERHASIL!"
4. Tutup command prompt

> Jika muncul *"Windows protected your PC"*, klik More info → Run anyway

### macOS

```bash
brew install cloudflare/cloudflare/cloudflared
```

> Belum punya Homebrew? Install dulu di [brew.sh](https://brew.sh)

Verifikasi instalasi:

```bash
cloudflared --version
```

---

## Langkah 2 — Login SSH ke Server

Ambil **username dan password SSH** dari **Job Summary** di GitHub Actions.

### Windows

1. Download file `ssh.bat` dari github
2. Klik kanan → Run as administrator
3. Masukkan username SSH saat diminta → Enter
4. Masukkan password SSH saat diminta → Enter

### macOS / Linux

1. Download file `ssh.sh` dari github
2. `chmod +x ssh.sh`
3. Jalankan dengan `./ssh.sh`
4. Masukkan username SSH saat diminta → Enter
5. Masukkan password SSH saat diminta → Enter

Jika diminta konfirmasi (pertama kali):
```
Are you sure you want to continue connecting? (yes/no)
```
Ketik `yes` lalu Enter.

> Password SSH selalu bisa dilihat kembali di GitHub Actions → tab Actions → klik deploy terbaru → Job Summary.

---

## Langkah 3 — Setup Laravel

> Langkah ini hanya perlu dilakukan **sekali** setelah deploy pertama. Update kode selanjutnya cukup push ke GitHub.

Setelah berhasil login SSH, terminal otomatis masuk ke direktori project dan menampilkan informasi sesi:

```
╔══════════════════════════════════════════════╗
║  AutoDeploy SSH Session — nama-repo          ║
╠══════════════════════════════════════════════╣
║  Web   : https://nama-repo.akhzafachrozy.my.id
║  Dir   : /www/wwwroot/hosting/nama-repo      ║
╠══════════════════════════════════════════════╣
║  Perintah cepat:                             ║
║  php artisan migrate --force                 ║
║  php artisan key:generate                    ║
║  composer install --no-dev                   ║
║  fix-perm   → fix permission                 ║
║  restart-app → restart service               ║
║  log-app    → lihat log live                 ║
╚══════════════════════════════════════════════╝
```

> PATH PHP, alias `php`, `artisan`, `composer`, dan perintah manajemen service sudah diset otomatis — langsung ketik tanpa path penuh.

### 3a. Install dependencies Composer

```bash
composer install --no-dev --optimize-autoloader
```

### 3b. File .env sudah dibuat otomatis

File `.env` sudah dibuat otomatis oleh sistem saat deploy dengan konfigurasi:

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://nama-repo.akhzafachrozy.my.id

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=nama_repo        ← otomatis dari nama repo
DB_USERNAME=deploysiitk      ← sudah diisi otomatis
DB_PASSWORD=deploysiitk2026  ← sudah diisi otomatis
```

Tidak perlu edit `.env` untuk konfigurasi database — sudah otomatis.

Jika ada konfigurasi tambahan (misalnya API key pihak ketiga), edit dengan:

```bash
nano .env
```

### 3c. Generate APP_KEY

```bash
php artisan key:generate
```

### 3d. Jalankan migrasi database

```bash
php artisan migrate --force
```

### 3e. Buat symlink storage

```bash
php artisan storage:link
```

### 3f. Install dependencies NPM & build aset (jika ada `package.json`)

```bash
npm install
npm run build
```

### 3g. Aktifkan web

```bash
restart-app
# atau
sudo systemctl restart autodeploy-nama-repo.service
```

### 3h. Verifikasi service berjalan

```bash
status-app
# atau
sudo systemctl status autodeploy-nama-repo.service
```

Output yang berarti sukses: `active (running)`

---

## Web Mahasiswa Sekarang Live!

Buka di browser: `https://nama-repo.akhzafachrozy.my.id`

---

## Cara Update Kode

Cukup push kode baru ke GitHub — proses deploy berjalan otomatis:

```bash
git add .
git commit -m "update fitur baru"
git push origin main
```

Yang aman saat update (tidak ditimpa):
- `.env` — konfigurasi tetap aman
- `storage/` — file upload user tetap ada
- `vendor/` — dependensi tetap ada
- `node_modules/` — dependensi NPM tetap ada
- `database/database.sqlite` — data SQLite tetap ada

Jika ada migrasi database baru, jalankan via SSH:

```bash
php artisan migrate --force
```

---

## Perintah Berguna di Server

Alias berikut sudah tersedia langsung tanpa perlu ketik path penuh:

```bash
# Manajemen service
restart-app                  # restart service
stop-app                     # stop service
status-app                   # lihat status service
log-app                      # lihat log aplikasi live

# Permission
fix-perm                     # fix permission satu perintah

# Artisan (langsung tanpa path penuh)
php artisan migrate --force
php artisan key:generate
php artisan storage:link
php artisan cache:clear
php artisan optimize:clear

# Composer (langsung tanpa path penuh)
composer install --no-dev --optimize-autoloader
composer dump-autoload

# Log systemd
sudo journalctl -u autodeploy-nama-repo.service -f

# Cek port
ss -tlnp | grep :PORT
```

---

## Troubleshooting

### Web tidak bisa diakses (502 Bad Gateway)

Service belum berjalan atau belum di-setup. Cek dengan:

```bash
status-app
```

Jika status `failed` atau `waiting-setup`, pastikan semua langkah setup Laravel di atas sudah dijalankan.

### SSH gagal terkoneksi

Pastikan `cloudflared` sudah terinstall:

```bash
cloudflared --version
```

Jika tidak ditemukan, ulangi instalasi cloudflared sesuai OS yang digunakan.

### Permission denied saat edit file

```bash
fix-perm
```

### `php artisan migrate` gagal — Access denied

Kemungkinan `.env` belum terbuat otomatis. Cek:

```bash
cat .env | grep DB_
```

Jika DB_USERNAME masih `root` atau DB_DATABASE masih `laravel`, jalankan:

```bash
sed -i 's/^DB_DATABASE=.*/DB_DATABASE=nama_repo/' .env
sed -i 's/^DB_USERNAME=.*/DB_USERNAME=deploysiitk/' .env
sed -i 's/^DB_PASSWORD=.*/DB_PASSWORD=deploysiitk2026/' .env
php artisan config:clear
php artisan migrate --force
```

### `npm install` atau `npm run build` gagal

Cek versi Node.js:

```bash
node --version
npm --version
```

Jika versi terlalu lama atau tidak tersedia, hubungi dosen terkait.

### Lupa password SSH

Buka GitHub → tab Actions → klik deploy terbaru → Job Summary — password selalu ditampilkan di sana.

### Web tampil 404 Not Found (bukan dari Nginx)

Kemungkinan database kosong. Jalankan seeder:

```bash
php artisan db:seed --force
```

Atau jika perlu reset:

```bash
php artisan migrate:fresh --seed --force
```

---

## Bantuan

Jika masih ada masalah:
1. Cek **Job Summary** di GitHub Actions untuk pesan error
2. Lihat log lengkap via SSH:
   ```bash
   log-app
   ```
3. Hubungi dosen terkait dengan menyertakan output log

---

*AutoDeploy v2.6 — Sistem Informasi ITK*
