# PRD — TradePortal (Web + Admin)
> Digital product store untuk trading tools (screener, indicator, kelas, dll)
> Stack: Laravel 12 + React 19 + Filament 5 + MySQL
> Auth: Laravel Socialite (Google OAuth)
> Storage: VPS Local Storage

---

## 1. Ringkasan Sistem

Website penjualan produk digital trading. User login via Google OAuth, checkout cepat tanpa hambatan, bayar via QRIS manual, upload bukti transfer, lalu admin verifikasi. Setelah diverifikasi, sistem otomatis mengirimkan akses produk ke user via **Email**. User memiliki dashboard pribadi untuk mengakses semua produk yang sudah dibeli kapan saja.

**Prinsip desain:**
- Tidak menggunakan payment gateway otomatis
- Tidak ada keranjang belanja — satu order = satu produk
- Affiliate tracking murni berbasis kupon, tanpa cookie/referral link
- Delivery produk sepenuhnya otomatis via email setelah admin verifikasi payment
- User dashboard sebagai pusat akses produk yang sudah dibeli — tidak bergantung pada inbox email
- WhatsApp hanya sebagai kontak support admin (link statis), tidak terintegrasi ke sistem
- Tidak ada refund — produk digital tidak memiliki cacat produksi
- Order pending yang tidak dibayar dalam 24 jam otomatis dibatalkan

---

## 2. Tech Stack

| Layer | Teknologi |
|---|---|
| Backend | Laravel 12 |
| Frontend public | React 19 + TypeScript + Vite |
| Admin panel | Filament 5 |
| Database | MySQL 8.0+ |
| Auth | Laravel Socialite (Google OAuth) |
| File storage | VPS Local Storage |
| Queue | Laravel Queue (database driver) |
| PHP | PHP 8.2+ |
| Email delivery | Resend (atau Mailgun / Brevo) |
| Error tracking | Sentry (opsional, direkomendasikan production) |

---

## 3. Arsitektur Sistem

```
[User Browser]
      ↓
[React 19 Frontend] ←→ [Laravel 12 API]
                               ↓
                        [MySQL Database]
                               ↓
                        [Laravel Queue]
                               ↓
                          [Email Job]
                               ↓
                          [Resend API]
                               ↓
                          [Email User]

[Admin]
  ↓
[Filament 5 Admin Panel] ←→ [Laravel 12]
```

---

## 4. Database Schema

**Total: 11 tabel**

### 4.1 Tabel `users`
```sql
id                bigint PK AUTO_INCREMENT
name              string
email             string UNIQUE
avatar            string NULLABLE        -- URL dari OAuth provider
provider          enum('google') NULLABLE
provider_id       string NULLABLE
password          string NULLABLE        -- null jika pure OAuth user, wajib diisi untuk akun admin
is_admin          boolean DEFAULT false  -- true = bisa login ke Filament panel admin
                                         -- selalu di-set false saat register via Google OAuth
                                         -- tidak ada endpoint API untuk mengubah field ini
email_verified_at timestamp NULLABLE
remember_token    string NULLABLE
created_at        timestamp
updated_at        timestamp
```

---

### 4.2 Tabel `products`
```sql
id                bigint PK AUTO_INCREMENT
name              string
slug              string UNIQUE
type              enum('screener','indicator','kelas','other')
short_description text NULLABLE
description       text
price             decimal(12,2)
thumbnail         string NULLABLE        -- path ke VPS storage
images            json NULLABLE          -- array of image paths
delivery_type     enum('auto','manual') DEFAULT 'auto'
                  -- auto: sistem langsung kirim email setelah verified
                  -- manual: admin perlu proses dulu sebelum delivery dikirim
email_subject     string NULLABLE        -- subject email delivery, bisa pakai placeholder
email_template    text NULLABLE          -- body email delivery, bisa pakai placeholder
                  -- Placeholder: {user_name}, {product_name}, {order_code}
delivery_content  text NULLABLE          -- konten akses produk yang ditampilkan di dashboard user
                                         -- bisa berupa HTML/markdown: link, instruksi, credentials, dll
is_active         boolean DEFAULT true
sold_count        integer DEFAULT 0    -- increment otomatis saat order completed, read-only
sort_order        integer DEFAULT 0
created_at        timestamp
updated_at        timestamp
deleted_at        timestamp NULLABLE     -- soft deletes
```

**Placeholder yang tersedia di template:**
- `{user_name}` — nama user
- `{product_name}` — nama produk
- `{order_code}` — kode order

---

### 4.2b Tabel `product_price_histories`
```sql
id           bigint PK AUTO_INCREMENT
product_id   bigint FK products
old_price    decimal(12,2)
new_price    decimal(12,2)
changed_by   bigint FK users   -- admin yang mengubah harga
changed_at   timestamp
notes        text NULLABLE     -- alasan perubahan harga (opsional, diisi admin)
```
> Dicatat otomatis setiap kali `products.price` diubah via Filament ProductResource.
> Berguna untuk audit dan resolusi dispute harga. Admin tidak bisa mengedit atau menghapus
> riwayat ini secara langsung — read-only di Filament.

---

### 4.3 Tabel `orders`
```sql
id                  bigint PK AUTO_INCREMENT
code                string UNIQUE          -- format: ORD-YYYYMMDD-XXXX
user_id             bigint FK users
coupon_id           bigint FK coupons NULLABLE
                    -- affiliate ter-track via coupons.affiliate_id
subtotal            decimal(12,2)
discount_amount     decimal(12,2) DEFAULT 0
final_price         decimal(12,2)          -- harga setelah diskon, sebelum nominal unik
unique_amount       decimal(12,2)          -- final_price + 3 digit acak (e.g. 299000 + 847 = 299847)
                                           -- ini yang ditransfer user dan dicek admin di mutasi
status              enum('pending','paid','verified','completed','cancelled') DEFAULT 'pending'
notes               text NULLABLE
expired_at          timestamp NULLABLE     -- di-set saat order dibuat: created_at + 24 jam
                                           -- null jika status sudah berubah dari pending
admin_verified_at   timestamp NULLABLE
admin_verified_by   bigint FK users NULLABLE
completed_at        timestamp NULLABLE
completed_by        bigint FK users NULLABLE
created_at          timestamp
updated_at          timestamp
```

**Affiliate tracking (pure coupon-based):**
- Affiliate ter-track jika user input kupon → `orders.coupon_id` → `coupons.affiliate_id`
- Tidak ada cookie, tidak ada referral link

**Nominal unik:**
- Di-generate saat order dibuat: `unique_amount = final_price + random(100, 999)`
- Harus unik di antara semua order berstatus `pending` dan `paid` pada hari yang sama (cek sebelum simpan, retry jika collision)
- Selisih 3 digit dicatat sebagai pembulatan — tidak ditagih terpisah, tidak dikembalikan ke user
- Yang ditampilkan ke user di halaman pembayaran adalah `unique_amount`, bukan `final_price`
- Memudahkan admin cocokkan pembayaran dari mutasi rekening tanpa perlu lihat bukti satu per satu

**Order expiry:**
- `expired_at` di-set saat order dibuat: `created_at + 24 jam`
- Laravel Scheduler menjalankan job `ExpireOrdersJob` setiap 5 menit: auto-cancel order `pending` yang sudah melewati `expired_at`
- Saat auto-cancel: update status → `cancelled`, catat `order_logs` (action: `order_expired`, user_id: null), decrement `coupons.used_count` jika order pakai kupon
- Order yang sudah `paid` (sudah upload bukti) **tidak** di-cancel otomatis meski melewati `expired_at` — admin yang memutuskan

---

### 4.4 Tabel `order_items`
```sql
id          bigint PK AUTO_INCREMENT
order_id    bigint FK orders
product_id  bigint FK products
price       decimal(12,2)   -- snapshot harga saat transaksi
created_at  timestamp
updated_at  timestamp
```

---

### 4.5 Tabel `payments`
```sql
id           bigint PK AUTO_INCREMENT
order_id     bigint FK orders UNIQUE
proof_image  string              -- path ke VPS storage
submitted_at timestamp
verified_at  timestamp NULLABLE
verified_by  bigint FK users NULLABLE
notes        text NULLABLE
created_at   timestamp
updated_at   timestamp
```

---

### 4.6 Tabel `product_deliveries`
```sql
id                      bigint PK AUTO_INCREMENT
order_id                bigint FK orders
product_id              bigint FK products
custom_delivery_content text NULLABLE   -- konten akses khusus per order (untuk delivery_type = 'manual')
                                        -- jika diisi, dashboard user menampilkan ini
                                        -- jika null, fallback ke products.delivery_content
status                  enum('pending','sent','failed') DEFAULT 'pending'
sent_at                 timestamp NULLABLE
error_message           text NULLABLE
retry_count             integer DEFAULT 0      -- max 3
created_at              timestamp
updated_at              timestamp
```
> Satu order menghasilkan satu record delivery (email). Field `custom_delivery_content` diisi admin
> untuk produk `manual` — memungkinkan konten akses berbeda per pembeli (contoh: username/password
> unik, link grup eksklusif, serial key). Untuk produk `auto`, field ini null dan dashboard
> menampilkan `products.delivery_content` yang sama untuk semua pembeli.

---

### 4.7 Tabel `affiliates`
```sql
id         bigint PK AUTO_INCREMENT
name       string
code       string UNIQUE    -- selalu uppercase
user_id    bigint FK users NULLABLE
is_active  boolean DEFAULT true
created_at timestamp
updated_at timestamp
```

---

### 4.8 Tabel `coupons`
```sql
id           bigint PK AUTO_INCREMENT
affiliate_id bigint FK affiliates NULLABLE
code         string UNIQUE            -- selalu uppercase
type         enum('percent','flat')
value        decimal(10,2)
min_purchase decimal(12,2) NULLABLE
max_uses     integer NULLABLE
used_count   integer DEFAULT 0
expired_at   timestamp NULLABLE
is_active    boolean DEFAULT true
created_at   timestamp
updated_at   timestamp
```

---

### 4.9 Tabel `order_logs`
```sql
id          bigint PK AUTO_INCREMENT
order_id    bigint FK orders
user_id     bigint FK users NULLABLE  -- null jika aksi dilakukan sistem
action      string
            -- 'order_created', 'payment_uploaded', 'payment_verified',
            -- 'delivery_sent', 'delivery_failed', 'order_completed', 'order_cancelled'
from_status enum('pending','paid','verified','completed','cancelled') NULLABLE
to_status   enum('pending','paid','verified','completed','cancelled') NULLABLE
notes       text NULLABLE
created_at  timestamp
```

---

### 4.10 Tabel `notifications`
```sql
id            bigint PK AUTO_INCREMENT
user_id       bigint FK users
order_id      bigint FK orders NULLABLE
type          string
              -- 'order_confirmed' (ke user, saat order dibuat)
              -- 'payment_verified' (ke user)
              -- 'order_expired' (ke user, saat order auto-cancel)
              -- 'order_cancelled' (ke user)
              -- 'new_order_admin' (ke admin, saat user upload bukti)
payload       json            -- isi email yang dikirim
status        enum('pending','sent','failed') DEFAULT 'pending'
sent_at       timestamp NULLABLE
error_message text NULLABLE
created_at    timestamp
updated_at    timestamp
```
> Tabel ini untuk notifikasi status order via email (bukan delivery produk). Delivery produk dicatat di `product_deliveries`.

---

## 5. Alur Autentikasi

### 5.1 Login / Register via Google OAuth
1. User klik "Login dengan Google"
2. Redirect ke Google OAuth
3. Callback: cek apakah `email` sudah ada di `users`
   - **Sudah ada** → update `avatar`, `provider`, `provider_id` jika berubah → login
   - **Belum ada** → buat user baru dari data OAuth
4. Redirect ke halaman tujuan sebelumnya (atau `/dashboard`)

### 5.2 Middleware
- `auth` — route yang butuh login (checkout, payment, dashboard, orders) redirect ke OAuth jika belum login

---

## 5.3 Admin Authentication (Filament 5)

### Keputusan Arsitektur

Admin menggunakan **model dan guard terpisah** dari user biasa. Ini dipilih karena:
- Admin login via email + password (bukan Google OAuth)
- Admin tidak pernah bisa menjadi pembeli, dan user biasa tidak pernah bisa mengakses panel admin
- Memisahkan guard mencegah privilege escalation — session user publik tidak bisa dipakai masuk ke `/admin`

### Implementasi

**Kolom tambahan di tabel `users`:**
```sql
is_admin   boolean DEFAULT false   -- true = bisa login ke Filament panel
```

> Gunakan satu tabel `users` yang sama, tapi akses Filament dikontrol via kolom `is_admin`.
> Ini lebih sederhana daripada model Admin terpisah mengingat skala proyek ini kecil.

**Guard Filament:**
Filament 5 dikonfigurasi dengan guard `web` (default Laravel), tapi akses panel dibatasi via method `canAccessPanel()` di model `User`:

```php
// app/Models/User.php
use Filament\Models\Contracts\FilamentUser;
use Filament\Panel;

class User extends Authenticatable implements FilamentUser
{
    public function canAccessPanel(Panel $panel): bool
    {
        return $this->is_admin === true;
    }
}
```

**Login Admin:**
- URL panel admin: `/admin`
- Admin login menggunakan **email + password** (bukan Google OAuth)
- Field `password` di tabel `users` tidak null untuk akun admin
- Akun admin dibuat via `php artisan make:filament-user` atau Seeder (lihat section 22)
- User biasa yang mencoba akses `/admin` akan di-redirect ke halaman login Filament, dan ditolak meski berhasil login karena `canAccessPanel()` return false

**Middleware:**
- Route `/admin/*` dilindungi oleh Filament middleware bawaan — tidak perlu konfigurasi tambahan
- Route `/api/*` tetap menggunakan guard `web` dengan session cookie (Sanctum SPA)

### Role Admin

Untuk skala proyek ini, **tidak ada multi-role** — semua admin memiliki akses penuh ke semua Resource di Filament. Jika di masa depan butuh pembatasan (misal: staff hanya bisa verify payment, tidak bisa hapus produk), tambahkan kolom `admin_role enum('super_admin','staff')` dan override method `canCreate()`, `canEdit()`, `canDelete()` di masing-masing Resource.

### Catatan Keamanan Admin Auth
- Admin tidak bisa login via Google OAuth — kolom `provider` dan `provider_id` null untuk akun admin
- Kolom `is_admin` tidak boleh bisa diubah oleh user sendiri — tidak ada endpoint API untuk ini
- Saat membuat user baru via OAuth callback, `is_admin` selalu di-set `false` secara eksplisit

---

## 6. Alur Checkout

```
[Halaman Produk]
  → Klik "Beli Sekarang"
  → Belum login → redirect OAuth → kembali ke halaman produk
  → Sudah login → halaman checkout

[Halaman Checkout]
  Tampil:
  - Ringkasan produk (thumbnail, nama, harga)
  - Data user (nama + email) — read only
  - Field kode kupon (opsional) + tombol "Pakai" — validasi real-time
  - Total harga (update otomatis saat kupon dipakai)
  - Tombol "Konfirmasi & Bayar"

  Saat "Konfirmasi & Bayar" diklik:
  1. Validasi produk aktif: cek `product->is_active` di server
     - Jika tidak aktif → return error: "Produk ini sedang tidak tersedia."
  2. Validasi order aktif: cek apakah user sudah memiliki order dengan status `pending` atau `paid`
     untuk produk yang sama
     - Jika ada → tampilkan error: "Kamu masih memiliki order aktif untuk produk ini.
       Selesaikan atau batalkan order tersebut sebelum membuat order baru."
       + sertakan link ke halaman order aktif tersebut (`/payment/{order:code}`)
     - Jika tidak ada → lanjut ke step berikutnya
  3. Validasi kupon jika ada
  4. Hitung final_price = subtotal - discount_amount
  5. Dalam satu database transaction:
     a. Re-validasi order aktif (cegah race condition jika user klik dua kali bersamaan):
        SELECT COUNT(*) FROM orders
        JOIN order_items ON order_items.order_id = orders.id
        WHERE orders.user_id = {user_id}
        AND order_items.product_id = {product_id}
        AND orders.status IN ('pending', 'paid')
        → Jika ada → rollback + return error
     b. Generate order_code (ORD-YYYYMMDD-XXXX)
     c. Generate unique_amount = final_price + random(100, 999)
        → Cek collision di orders where status IN ('pending','paid') AND DATE(created_at) = today
        → Retry generate jika collision, max 10x
     d. Buat record orders (status: pending, simpan final_price + unique_amount)
     e. Buat record order_items (snapshot harga)
     f. Increment coupons.used_count jika pakai kupon
     g. Catat order_logs: action = 'order_created'
     h. Dispatch queue job: kirim email konfirmasi order ke user
  6. Redirect ke halaman pembayaran

[Halaman Pembayaran QRIS]
  Tampil:
  - Gambar QR statis (dari config QRIS_IMAGE_PATH)
  - Nominal UNIK yang harus ditransfer: unique_amount (BUKAN final_price)
    → Tampilkan juga keterangan: "Transfer tepat nominal ini agar pembayaran mudah diidentifikasi"
  - Kode order
  - Form upload foto bukti transfer (image/*, max 5MB)
    → Preview gambar sebelum submit
  - Tombol "Saya Sudah Bayar & Upload Bukti"

  Setelah upload:
  1. Validasi file (MIME, ukuran, ekstensi)
  2. Simpan ke storage/app/public/proofs/{order_code}.{ext}
  3. Buat record payments
  4. Update orders.status → 'paid'
  5. Catat order_logs: action = 'payment_uploaded'
  6. Dispatch queue job: notifikasi ke admin (ada order perlu diverifikasi)
  7. Redirect ke halaman sukses

[Halaman Sukses]
  - Pesan konfirmasi bukti diterima
  - Detail order (kode, produk, total)
  - Info bahwa akses produk akan dikirim via email setelah diverifikasi admin
  - Tombol shortcut ke Dashboard
```

---

## 7. Alur Delivery Produk

Ini adalah alur inti setelah admin verifikasi payment.

```
[Admin klik "Verify Payment" di Filament]
         ↓
[Laravel: update orders.status → 'verified']
[Catat order_logs: action = 'payment_verified']
         ↓
[Cek products.delivery_type]
         ↓
   ┌─────┴──────┐
  auto        manual
   ↓             ↓
[Dispatch     [Admin dapat notif:
DeliveryJob]   "Produk ini butuh
               proses manual dulu"
               Admin isi konten delivery
               di Filament, lalu klik
               "Kirim Delivery"]
                ↓
           [Dispatch DeliveryJob]
                ↓
[DeliveryJob berjalan di queue]
   ↓
   Buat 1 record product_deliveries (email)
   ↓
[EmailDeliveryJob]
   ↓
Render template email dengan placeholder
   ↓
Kirim via Resend API
   ↓
Berhasil → update product_deliveries status = 'sent', isi sent_at
           → update orders.status = 'completed'
           → catat order_logs: action = 'delivery_sent'
           → konten delivery tersedia di user dashboard (/dashboard)
   ↓
Gagal → retry max 3x, catat error_message
        Jika retry >= 3 → status = 'failed', catat order_logs
```

**Catatan penting:**
- Admin bisa trigger **"Kirim Ulang Email"** dari Filament jika delivery failed
- Konten akses produk (`products.delivery_content`) selalu tersedia di dashboard user setelah order `completed`, terlepas dari status email delivery — sehingga user tidak kehilangan akses meski email masuk spam

---

## 8. Alur Notifikasi Status Order

Terpisah dari delivery produk. Ini notifikasi perubahan status untuk admin dan user, semua via email.

| Trigger | Penerima | Isi |
|---|---|---|
| Order dibuat (status: pending) | User | Konfirmasi order + **nominal unik** yang harus ditransfer + kode order + instruksi upload bukti + link ke halaman pembayaran |
| User upload bukti (status: paid) | Admin | Ada order baru perlu diverifikasi + nominal unik + link ke Filament |
| Admin verify payment | User | "Pembayaran diterima, akses produk segera dikirim ke email ini" |
| Order auto-expired | User | "Order kamu telah kedaluwarsa karena tidak ada pembayaran dalam 24 jam" + info cara order ulang |
| Order cancelled (manual) | User | "Order dibatalkan" + nominal yang sudah dibayar (jika ada) + link kontak admin |

**Email konfirmasi order (pertama)** adalah yang paling penting — berisi:
- Nama produk + nominal unik yang harus ditransfer
- Instruksi QRIS + kode QR (jika memungkinkan)
- Kode order sebagai referensi
- Link langsung ke halaman pembayaran untuk upload bukti
- Berlaku sebagai **invoice digital** yang sah

> Semua notifikasi dikirim via email. WhatsApp tidak digunakan dalam sistem — hanya tersedia sebagai link kontak admin statis di halaman publik.

---

## 9. Alur Admin (Filament 5)

### 9.1 OrderResource

Kolom tabel:
- `code` (order code)
- `user.name` + `user.email`
- Nama produk (dari order_items)
- Kupon + affiliator (jika ada)
- `unique_amount` — nominal unik yang harus ditransfer user (ini yang dicek admin di mutasi)
- `final_price` — harga asli setelah diskon (tampil sebagai secondary info)
- `status` — badge: pending=abu, paid=kuning, verified=hijau, completed=biru, cancelled=merah
- Status delivery email (icon ✓/✗/⏳)
- `created_at` + `expired_at` (untuk order pending)

Filter:
- By `status`
- By `affiliate` (via kupon)
- By rentang tanggal
- By delivery status (ada yang failed)

Actions per row:
- **"Lihat Bukti"** → modal preview gambar bukti transfer
- **"Detail Delivery"** → modal menampilkan record `product_deliveries`: status, sent_at, retry_count, error_message — untuk keperluan support
- **"Verify Payment"** → update status ke 'verified', trigger DeliveryJob (jika auto), catat order_logs
- **"Isi & Kirim Delivery"** → muncul jika `delivery_type = 'manual'` dan status = 'verified'.
  Admin membuka modal dengan rich text editor untuk mengisi `custom_delivery_content` (konten akses
  khusus untuk pembeli ini — contoh: username, password, link grup). Wajib diisi sebelum bisa kirim.
  Setelah simpan → dispatch EmailDeliveryJob, konten ini yang dikirim via email dan ditampilkan di dashboard user.
- **"Kirim Ulang Email"** → trigger ulang EmailDeliveryJob untuk order ini
- **"Batalkan Order"** → modal konfirmasi → status = 'cancelled', catat order_logs

### 9.2 ProductResource
CRUD lengkap. Fields:
- `name`, `slug` (auto-generate)
- `short_description`, `description` (rich text)
- `price`, `type`, `sort_order`
- `thumbnail`, `images` (upload)
- `delivery_type` (toggle: auto / manual)
- `email_subject` — subject email delivery
- `email_template` — rich text editor, support placeholder
- `delivery_content` — rich text editor, konten akses produk untuk semua pembeli (produk `auto`)
  atau konten default jika admin tidak mengisi konten khusus (produk `manual`)
- `sold_count` — read-only, increment otomatis saat order completed
- `is_active` (toggle)

> Tampilkan preview placeholder yang tersedia di bawah field template.

**Tombol "Preview Email":** render `email_template` dengan data dummy (`{user_name}` = "John Doe", `{product_name}` = nama produk, `{order_code}` = "ORD-20250101-ABCD") dan tampilkan di modal. Berguna untuk cek layout dan placeholder sebelum produk aktif.

**Validasi sebelum `is_active = true`:**
- `email_subject` tidak boleh kosong
- `email_template` tidak boleh kosong
- `delivery_content` tidak boleh kosong
- Jika salah satu kosong → blokir simpan + tampilkan error: "Produk tidak dapat diaktifkan sebelum email template dan konten akses produk diisi."

**Aturan soft delete produk:**
- Produk tidak boleh di-soft delete jika masih ada order dengan status `pending`, `paid`, atau `verified`
- Jika ada order aktif → tampilkan error: "Produk tidak dapat dihapus karena masih memiliki X order aktif."
- Produk yang sudah di-soft delete tetap bisa diakses oleh order lama (delivery ulang tetap berfungsi)

**Tab "Riwayat Harga"** di halaman detail produk:
- Tabel read-only: tanggal perubahan, harga lama, harga baru, diubah oleh (nama admin), catatan
- Admin tidak bisa edit atau hapus riwayat ini
- Dicatat otomatis via `ProductObserver` setiap kali field `price` berubah

**Tab "Pembeli"** di halaman detail produk:
- Tabel list user yang memiliki order `completed` untuk produk ini
- Kolom: nama user, email, kode order, tanggal beli, status delivery email
- Berguna untuk keperluan support dan audit

### 9.3 AffiliateResource
CRUD lengkap: `name`, `code` (auto-uppercase), `user_id` (optional), `is_active`.
Tampilkan total order ter-track di tabel.

### 9.4 CouponResource
CRUD lengkap: `code`, `affiliate_id`, `type`, `value`, `min_purchase`, `max_uses`, `expired_at`, `is_active`.
Tampilkan `used_count` / `max_uses`.

> **Field `used_count`** dapat diedit manual oleh admin — berguna jika terjadi lonjakan pembelian
> atau perlu reset. Tampilkan peringatan kuning di form edit:
> _"Mengubah used_count secara manual dapat mempengaruhi batas pemakaian kupon.
> Pastikan perubahan ini disengaja."_

### 9.5 Dashboard Widgets
- **Stats**: revenue bulan ini, total order, order pending verifikasi (badge merah jika > 0), order expired hari ini
- **Order terbaru**: 10 order terakhir
- **Top affiliator**: by jumlah order bulan ini
- **Delivery gagal**: jumlah `product_deliveries` dengan status 'failed' — badge merah, klik langsung ke filter
- **Queue health**: tampilkan jumlah job pending di queue dan waktu job terlama menunggu — badge merah jika ada job pending > 30 menit (indikasi queue worker mati)

---

## 10. Halaman Public

### 10.1 `/` — Beranda
- Hero section
- Grid produk aktif, urut by `sort_order`
- Kartu: thumbnail, nama, tipe, harga, tombol "Beli Sekarang"
- Tombol kontak admin (link `wa.me/{ADMIN_WHATSAPP}`) — statis, tidak terintegrasi sistem

### 10.2 `/products/{slug}` — Detail Produk
- Gallery gambar, nama, tipe, harga
- Short description + deskripsi lengkap
- Tombol "Beli Sekarang"
  - Jika user belum login → redirect ke OAuth saat diklik
  - Jika user sudah memiliki order `completed` untuk produk ini → tombol diganti teks non-interaktif: **"Kamu sudah memiliki produk ini"** + link ke `/dashboard`
  - Jika user memiliki order `pending`/`paid` untuk produk ini → tombol diganti teks + link: **"Kamu memiliki order aktif untuk produk ini"** → arahkan ke `/payment/{order:code}`

### 10.3 `/checkout/{product:slug}` — Checkout
- Ringkasan produk + data user (nama + email, read only)
- Field kupon + validasi real-time
- Total harga dinamis
- Tombol "Konfirmasi & Bayar"

### 10.4 `/payment/{order:code}` — Pembayaran
- QR QRIS statis + nominal unik + kode order
- Upload bukti transfer (preview sebelum submit)
- Validasi: max 5MB, image/*
- Countdown timer `expired_at`
- **Banner warning email konfirmasi**: jika `notifications` record dengan `type = 'order_confirmed'`
  dan `order_id` ini berstatus `failed`, tampilkan banner:
  _"Email konfirmasi order belum berhasil dikirim ke inbox kamu."_ + tombol "Kirim Ulang Email Konfirmasi"
  → memanggil `POST /api/orders/{code}/resend-confirmation` (rate limit: 1x per 10 menit)
  → Nominal unik tetap selalu terlihat di halaman ini sehingga user tidak bergantung pada email
- **Polling status order**: setelah user upload bukti, halaman otomatis polling `GET /api/orders/{code}/status` setiap 30 detik
  - Jika status berubah menjadi `completed` → redirect otomatis ke `/dashboard` dengan toast notifikasi "Akses produk kamu sudah aktif!"
  - Jika status `cancelled` → tampilkan pesan order dibatalkan
  - Polling berhenti otomatis setelah redirect atau jika tab tidak aktif (Page Visibility API)

### 10.5 `/order/{order:code}/success` — Sukses
- Pesan konfirmasi bukti diterima
- Detail order (kode, produk, total)
- Info bahwa akses produk akan dikirim via email setelah diverifikasi admin
- Tombol shortcut ke Dashboard

### 10.6 `/dashboard` — Dashboard User (halaman utama setelah login)
Halaman pusat bagi user yang sudah login. Terdiri dari:

**Section "Order Aktif"** (tampil jika ada order `pending` atau `paid`):
- Banner/card di atas halaman
- Tampil: nama produk, nominal, status, countdown expired_at (jika pending)
- Tombol: "Lanjut Bayar" (ke `/payment/{order:code}`)

**Section "Produk Saya"**:
- Grid kartu semua produk yang sudah dibeli (order `completed`)
- Per kartu: thumbnail + nama produk
- Klik kartu → halaman detail produk yang dibeli (`/dashboard/products/{order_id}`)
- Jika belum ada produk → tampilkan empty state dengan CTA ke halaman beranda

### 10.7 `/dashboard/products/{order_id}` — Detail Produk yang Dibeli
- Thumbnail + nama produk
- Tanggal pembelian + kode order
- **Konten akses produk** — ditampilkan dengan prioritas:
  1. Jika `product_deliveries.custom_delivery_content` terisi → tampilkan ini (konten khusus per pembeli)
  2. Jika kosong → fallback ke `products.delivery_content` (konten umum semua pembeli)
- Tombol "Hubungi Admin" → link `wa.me/{ADMIN_WHATSAPP}` (statis)
- **Jika `product_deliveries.status = 'failed'`**: tampilkan banner warning "Email akses produk gagal terkirim" + tombol "Kirim Ulang Email"
  - Rate limit: max 1x per jam per order
  - Tombol disabled dengan countdown jika sudah dipakai dalam 1 jam terakhir
  - Aksi ini memanggil `POST /api/orders/{code}/resend-delivery`

### 10.8 `/orders` — Riwayat Transaksi
- List semua order milik user (semua status)
- Tampil: kode order, nama produk, nominal, status (badge), status delivery email (icon), tanggal
- Order `pending` tampil dengan countdown `expired_at`
- Klik order → detail order + link ke halaman pembayaran jika masih `pending`/`paid`

### 10.9 `/profile` — Profil User
- Tampil: nama, email, avatar (read only — dari Google)

---

## 11. API Endpoints

Semua endpoint diawali `/api`. Auth menggunakan Laravel Sanctum session cookie (SPA).
Endpoint bertanda 🔒 memerlukan user login.

### Config
| Method | Path | Auth | Keterangan |
|---|---|---|---|
| GET | `/api/config` | — | Mengembalikan config publik: `admin_whatsapp`, `qris_image_url` |

### Auth
| Method | Path | Auth | Keterangan |
|---|---|---|---|
| GET | `/auth/google` | — | Redirect ke Google OAuth |
| GET | `/auth/google/callback` | — | Callback dari Google, set session |
| POST | `/auth/logout` | 🔒 | Hapus session |

### Products
| Method | Path | Auth | Keterangan |
|---|---|---|---|
| GET | `/api/products` | — | List produk aktif (paginated, default 15) |
| GET | `/api/products/{slug}` | — | Detail produk + status kepemilikan jika login |

### Coupons
| Method | Path | Auth | Keterangan |
|---|---|---|---|
| POST | `/api/coupons/validate` | 🔒 | Validasi kode kupon. Body: `{coupon_code, product_id}`. Return: `{valid, discount_amount, final_price}` |

### Orders & Checkout
| Method | Path | Auth | Keterangan |
|---|---|---|---|
| POST | `/api/orders` | 🔒 | Buat order baru. Body: `{product_id, coupon_code?}`. Return: `{order_code, unique_amount, expired_at}` |
| GET | `/api/orders` | 🔒 | List semua order milik user (paginated) |
| GET | `/api/orders/{code}` | 🔒 | Detail order milik user |
| GET | `/api/orders/{code}/status` | 🔒 | Polling status. Return: `{status, completed_at}`. Rate limit: 120 req/jam |

### Payments
| Method | Path | Auth | Keterangan |
|---|---|---|---|
| POST | `/api/orders/{code}/payment` | 🔒 | Upload bukti transfer. Body: `multipart/form-data {proof_image}`. Rate limit: 3x per order |

### Delivery
| Method | Path | Auth | Keterangan |
|---|---|---|---|
| POST | `/api/orders/{code}/resend-delivery` | 🔒 | Kirim ulang email akses produk (hanya jika status `failed`). Rate limit: 1x/jam per order |
| POST | `/api/orders/{code}/resend-confirmation` | 🔒 | Kirim ulang email konfirmasi order. Rate limit: 1x/10 menit per order |

### Dashboard
| Method | Path | Auth | Keterangan |
|---|---|---|---|
| GET | `/api/dashboard` | 🔒 | Data dashboard: `{active_orders[], owned_products[]}` |
| GET | `/api/dashboard/products/{order_id}` | 🔒 | Konten akses produk yang sudah dibeli |

### Profile
| Method | Path | Auth | Keterangan |
|---|---|---|---|
| GET | `/api/profile` | 🔒 | Data profil user |

---

**Contoh response `GET /api/orders/{code}/status`:**
```json
{
  "status": "completed",
  "completed_at": "2025-01-15T10:30:00+07:00"
}
```

**Contoh response `POST /api/orders` (sukses):**
```json
{
  "order_code": "ORD-20250115-A3K9",
  "unique_amount": 299847,
  "final_price": 299000,
  "expired_at": "2025-01-16T10:00:00+07:00",
  "payment_url": "/payment/ORD-20250115-A3K9"
}
```

**Contoh response error (422):**
```json
{
  "message": "Kamu masih memiliki order aktif untuk produk ini.",
  "active_order_url": "/payment/ORD-20250114-B7X2"
}
```

---

## 12. Service Classes

### `CouponValidator`
```
Input : coupon_code (string|null), subtotal (decimal)
Output: ['coupon' => Coupon|null, 'discount' => decimal, 'affiliate_id' => int|null]

Validasi: aktif, belum expired, used_count < max_uses, subtotal >= min_purchase
```

### `DeliveryTemplateRenderer`
```
Input : product (Product), order (Order), user (User), custom_content (string|null)
Output: ['email_subject', 'email_body'] — placeholder sudah diganti nilai nyata

Logika konten di email body:
- Jika custom_content terisi → gunakan custom_content sebagai konten akses di email
- Jika null → gunakan products.delivery_content

Placeholder: {user_name}, {product_name}, {order_code}
```

### `OrderCodeGenerator`
```
Format : ORD-{YYYYMMDD}-{4 char random uppercase alphanumeric}
Retry loop max 10x jika collision, fallback ke microtime
```

### `UniqueAmountGenerator`
```
Input : final_price (decimal)
Output: unique_amount (decimal)

Logika:
1. Generate suffix = random integer antara 100–999
2. unique_amount = final_price + suffix (dalam satuan rupiah penuh, bukan desimal)
   Contoh: 299000.00 + 847 = 299847.00
3. Cek collision: SELECT COUNT(*) FROM orders
   WHERE unique_amount = {unique_amount}
   AND status IN ('pending', 'paid')
   AND DATE(created_at) = CURDATE()
4. Jika ada collision → ulangi dari step 1, max 10x
5. Jika 10x masih collision (sangat jarang) → gunakan suffix 4 digit (1000–9999)
```

### `DeliveryDispatcher`
```
Input : order_id
Output: void

Logika:
1. Load order beserta product dan user
2. Render template via DeliveryTemplateRenderer
3. Buat 1 record product_deliveries (email, status: pending)
4. Dispatch EmailDeliveryJob ke queue
```
### `OrderExpiryService`
```
Dipanggil oleh: ExpireOrdersJob (scheduler setiap 5 menit)

Logika:
1. Ambil semua orders WHERE status = 'pending' AND expired_at <= NOW()
2. Untuk setiap order:
   a. Update status → 'cancelled'
   b. Catat order_logs: action = 'order_expired', user_id = null
   c. Decrement coupons.used_count jika order pakai kupon
   d. Kirim email notifikasi ke user bahwa order expired
```

---

## 13. Queue Jobs

### `EmailDeliveryJob`
```
- Load product_deliveries record
- Kirim email via Resend API menggunakan rendered template
- Berhasil → update status = 'sent', isi sent_at
           → update orders.status = 'completed', isi completed_at
           → catat order_logs: action = 'delivery_sent'
- Gagal → increment retry_count, update error_message
  - Jika retry_count < 3 → release job ke queue (delay 5 menit)
  - Jika retry_count >= 3 → update status = 'failed', catat order_logs
```

### `AdminNotificationJob`
```
- Kirim pesan notifikasi ke Telegram Group via Telegram Bot API (BUKAN email)
- Package: irazasyed/telegram-bot-sdk ^3.0
- Trigger: saat user upload bukti transfer (order status berubah ke 'paid')
- Target: TELEGRAM_ADMIN_GROUP_ID di .env (nilai negatif, contoh: -1001234567890)
- Parse mode: Markdown
- Format pesan:

  🛒 *Order Baru Perlu Diverifikasi*

  👤 *User:* {nama user} ({email})
  📦 *Produk:* {nama produk}
  💰 *Nominal Transfer:* Rp {unique_amount}
  🔖 *Kode Order:* {order_code}
  🏷️ *Kupon:* {kode kupon atau "—"}

  🔗 [Lihat di Admin Panel]({APP_URL}/admin/orders/{order_id})

- Jika gagal kirim: catat error ke Laravel Log, jangan throw exception
  Notifikasi Telegram bersifat best-effort — tidak boleh menghentikan flow order
- Tidak ada retry otomatis untuk job ini
```

### `ExpireOrdersJob`
```
- Dijalankan via Laravel Scheduler setiap 5 menit
- Panggil OrderExpiryService untuk auto-cancel order pending yang sudah melewati expired_at
- Log jumlah order yang di-cancel ke Laravel Log
```

---

## 14. Kontak Admin (WhatsApp Statis)

WhatsApp tidak terintegrasi ke sistem. Nomor WA admin hanya digunakan sebagai link kontak statis di beberapa titik UI:

- Tombol "Hubungi Admin" di beranda
- Tombol "Hubungi Admin" di halaman `/dashboard/products/{order_id}`
- Footer halaman publik

Format link: `https://wa.me/{ADMIN_WHATSAPP}?text=Halo+Admin+TradePortal,...`

Nilai `ADMIN_WHATSAPP` dikonfigurasi di `.env` dan di-expose ke frontend via API config endpoint atau env variable build time.

---

## 15. Aturan Bisnis

1. **Satu order = satu produk.** Tidak ada multi-item checkout.
2. **Affiliate tracking via kupon saja.** `orders.coupon_id` → `coupons.affiliate_id`.
3. **Snapshot harga** di `order_items.price` — perubahan harga produk tidak mempengaruhi order lama.
4. **Nominal unik (`unique_amount`)** di-generate saat order dibuat. Yang ditampilkan dan ditransfer user adalah `unique_amount`, bukan `final_price`. Selisih 3 digit adalah pembulatan, bukan biaya tambahan.
5. **`unique_amount` harus unik** di antara order berstatus `pending` dan `paid` pada hari yang sama.
6. **`used_count` kupon** di-increment saat order dibuat (status pending).
7. **Status flow:** `pending` → `paid` → `verified` → `completed`. Semua status bisa → `cancelled`.
8. **Setiap perubahan status** dicatat di `order_logs`.
9. **Email konfirmasi order** dikirim otomatis saat order dibuat — berisi nominal unik, instruksi transfer, dan link halaman pembayaran. Nominal unik selalu tampil di halaman `/payment` sehingga user tidak bergantung pada email.
10. **Delivery via email** dikirim otomatis setelah status = `verified` (jika `delivery_type = auto`).
11. **Delivery `manual`** — admin wajib mengisi `custom_delivery_content` per order sebelum delivery dikirim. Konten ini bersifat unik per pembeli (contoh: username/password, link grup eksklusif).
12. **Konten akses di dashboard** menggunakan prioritas: `product_deliveries.custom_delivery_content` jika ada, fallback ke `products.delivery_content`.
13. **Order `completed`** setelah email delivery berhasil terkirim.
14. **Konten produk tersedia di dashboard** setelah order `completed` — user tidak bergantung pada inbox email untuk akses produk yang dibeli.
15. **WhatsApp tidak terintegrasi sistem** — hanya link statis ke nomor admin untuk keperluan support manual.
16. **Satu order aktif per produk per user.** User tidak boleh membuat order baru untuk produk yang sama jika masih ada order berstatus `pending` atau `paid`. Order baru hanya diizinkan setelah order sebelumnya `completed` atau `cancelled`.
17. **Produk yang sudah dimiliki.** Jika user sudah memiliki order `completed` untuk suatu produk, tombol "Beli Sekarang" diganti teks non-interaktif: "Kamu sudah memiliki produk ini".
18. **Order expiry.** Order berstatus `pending` otomatis di-cancel setelah 24 jam jika user tidak upload bukti transfer. Order yang sudah `paid` tidak ikut di-cancel otomatis — admin yang memutuskan.
19. **Tidak ada refund.** Produk bersifat digital dan tidak memiliki cacat produksi. Tidak ada mekanisme pengembalian dana dalam bentuk apapun.
20. **Upload bukti transfer** hanya bisa dilakukan sekali per order. Jika perlu mengganti, user menghubungi admin — admin yang menghapus record lama via Filament.
21. **Produk tidak memiliki stok.** Produk bersifat digital — tidak ada batasan jumlah pembeli. Field `sold_count` hanya sebagai informasi statistik, bukan pembatas transaksi.
22. **Riwayat perubahan harga** dicatat otomatis di `product_price_histories` setiap kali admin mengubah harga produk. Riwayat ini read-only dan tidak bisa dihapus.
23. **Produk tidak dapat diaktifkan** (`is_active = true`) jika `email_subject`, `email_template`, atau `delivery_content` masih kosong.
24. **Validasi produk aktif saat checkout** dilakukan di sisi server — bukan hanya di frontend. Jika produk di-nonaktifkan saat user sedang di halaman checkout, server menolak order dengan error yang jelas.
25. **Session OAuth berlaku 30 hari.** User tidak perlu login ulang selama masih dalam periode tersebut. Session diperbarui otomatis setiap ada aktivitas.
26. **Polling status order** berjalan di halaman `/payment` setelah user upload bukti — setiap 30 detik cek status via API. Otomatis redirect ke dashboard saat order `completed`.
27. **Kirim ulang email oleh user** diizinkan dari halaman detail produk di dashboard, hanya jika `product_deliveries.status = 'failed'`. Rate limit: max 1x per jam per order.
28. **Timezone sistem: Asia/Jakarta (WIB).** Semua operasi waktu (expired_at, scheduler, countdown di frontend) menggunakan WIB.

---

## 16. Keamanan & Validasi

- **Rate limiting** pada endpoint upload bukti: max 3x per order
- **Rate limiting** pada endpoint resend delivery oleh user: max 1x per jam per order
- **Rate limiting** pada endpoint polling status order: max 120 request per jam per user (setiap 30 detik)
- **File validation**: MIME type, ekstensi, dan ukuran (max 5MB)
- **Order ownership policy**: user hanya bisa akses order miliknya — semua endpoint order divalidasi via Policy
- **Coupon validation** dalam database transaction bersama pembuatan order (cegah race condition)
- **Duplikasi order**: validasi order aktif dilakukan dua kali — sekali sebelum masuk transaction (early return), sekali di dalam transaction dengan SELECT ... FOR UPDATE (cegah race condition klik ganda)
- **Validasi produk aktif**: `product->is_active` dicek di server saat checkout diproses, bukan hanya di frontend
- **Validasi kelengkapan produk**: `is_active` tidak bisa di-set `true` jika template atau delivery_content kosong
- **Upload bukti transfer**: satu order hanya boleh punya satu record aktif di tabel `payments`. Jika admin menghapus record lama, user bisa upload ulang — ini dicatat di `order_logs`
- **Soft delete produk**: diblokir jika masih ada order aktif untuk produk tersebut — validasi dilakukan di ProductResource sebelum eksekusi delete
- **CSRF protection** pada semua form POST
- **CORS** hanya untuk domain frontend yang terdaftar di `.env`
- **Pagination** wajib pada semua endpoint yang mengembalikan list (orders, products) — default 15 per halaman

---

## 17. File Storage

```
storage/app/public/
├── proofs/       -- Bukti transfer: {order_code}.{ext}
└── products/     -- Gambar produk (thumbnail + gallery)
```

---

## 17.1 SEO & Meta Tags

Halaman publik yang perlu dioptimasi untuk mesin pencari:

**Halaman yang di-index:**
- `/` — Beranda: title, meta description, Open Graph image
- `/products/{slug}` — Detail Produk: title = nama produk, meta description = `short_description`, OG image = thumbnail

**Halaman yang TIDAK di-index (noindex):**
- Semua halaman dashboard, checkout, payment, orders, profile

**Sitemap:**
- `GET /sitemap.xml` — digenerate otomatis oleh Laravel, berisi semua URL produk aktif
- Di-update otomatis saat ada produk baru diaktifkan atau di-nonaktifkan

**Open Graph tags** di halaman produk:
```html
<meta property="og:title" content="{product.name} — TradePortal" />
<meta property="og:description" content="{product.short_description}" />
<meta property="og:image" content="{product.thumbnail_url}" />
<meta property="og:type" content="product" />
```

---

## 18. Environment Variables

```env
APP_NAME=TradePortal
APP_URL=https://yoursite.com
FRONTEND_URL=https://yoursite.com
APP_TIMEZONE=Asia/Jakarta

# Database
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=tradeportal
DB_USERNAME=
DB_PASSWORD=

# Storage
FILESYSTEM_DISK=public

# Session
SESSION_LIFETIME=43200    -- 30 hari dalam menit

# Google OAuth
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=https://yoursite.com/auth/google/callback

# QRIS
QRIS_IMAGE_PATH=public/qris/qr-statis.png

# Email delivery (Resend)
RESEND_API_KEY=
MAIL_FROM_ADDRESS=order@yoursite.com
MAIL_FROM_NAME=TradePortal

# Admin kontak
ADMIN_WHATSAPP=628xxxxxxxxxx
ADMIN_EMAIL=admin@yoursite.com        # dipakai untuk backup notification saja

# Telegram Bot (notifikasi admin)
TELEGRAM_BOT_TOKEN=                   # dari @BotFather
TELEGRAM_ADMIN_GROUP_ID=              # ID group telegram admin (negatif, contoh: -1001234567890)

# SEO
APP_DESCRIPTION="Platform trading tools terbaik — screener, indikator, dan kelas"

# Backup (opsional, jika pakai spatie/laravel-backup)
BACKUP_NOTIFICATION_EMAIL=admin@yoursite.com

# Sentry (opsional)
SENTRY_LARAVEL_DSN=
```

---

## 19. Server Requirements

- PHP 8.2+, Extensions: BCMath, Ctype, cURL, DOM, Fileinfo, JSON, Mbstring, OpenSSL, PCRE, PDO, Tokenizer, XML, GD/Imagick
- MySQL 8.0+
- Composer 2.5+, NPM 9+
- Supervisor (untuk queue worker + scheduler)

### VPS Minimum
- RAM: 2GB, Storage: 20GB, CPU: 2 core, Bandwidth: 1TB/bulan

### Backup Strategy
- Database: backup otomatis harian via `spatie/laravel-backup` → simpan ke direktori terpisah atau cloud storage (S3/R2)
- Storage (bukti transfer + gambar produk): ikut backup bersama database
- Retensi: simpan 7 backup terakhir, hapus yang lebih lama otomatis
- Notifikasi email ke admin jika backup gagal

### Composer Dependencies
```json
{
  "require": {
    "php": "^8.2",
    "laravel/framework": "^12.0",
    "laravel/socialite": "^5.0",
    "filament/filament": "^5.0",
    "resend/resend-laravel": "^0.10",
    "irazasyed/telegram-bot-sdk": "^3.0",
    "spatie/laravel-backup": "^9.0"
  }
}
```

### NPM Dependencies (frontend — di /frontend)
```json
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router-dom": "^7.0.0",
    "axios": "^1.7.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vite": "^6.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0"
  }
}
```

---

## 19.1 Prerequisites Checklist Sebelum Mulai Coding

Status prerequisite yang sudah dan belum siap:

| # | Prerequisite | Status | Aksi yang Diperlukan |
|---|---|---|---|
| 1 | VPS / server lokal (PHP 8.2+, MySQL 8) | ✅ Siap | — |
| 2 | Resend / Mailgun API key | ✅ Siap | Isi `RESEND_API_KEY` di `.env` |
| 3 | Google OAuth credentials | ❌ Belum | Lihat panduan di bawah |
| 4 | File `.env` sudah diisi lengkap | ❌ Belum | Isi setelah Google OAuth selesai |

### Cara Setup Google OAuth (wajib sebelum step 6 development)

1. Buka [console.cloud.google.com](https://console.cloud.google.com)
2. Buat project baru → **"TradePortal"**
3. Di sidebar: **APIs & Services → OAuth consent screen**
   - User Type: **External**
   - Isi App name, support email, developer email → Save
4. Di sidebar: **APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID**
   - Application type: **Web application**
   - Name: **TradePortal**
   - Authorized redirect URIs — tambahkan keduanya:
     ```
     http://localhost:8000/auth/google/callback   ← untuk development
     https://yourdomain.com/auth/google/callback  ← untuk production
     ```
5. Copy **Client ID** dan **Client Secret** → isi di `.env`:
   ```env
   GOOGLE_CLIENT_ID=xxxxxxxxxxxx.apps.googleusercontent.com
   GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxx
   GOOGLE_REDIRECT_URI=http://localhost:8000/auth/google/callback
   ```

> ⚠ Saat development lokal, pastikan app Laravel berjalan di port 8000 (`php artisan serve`), atau sesuaikan redirect URI dengan port yang dipakai.

---

## 19.2 Struktur Repo — Monorepo (Laravel + React dalam Satu Repo)

**Keputusan: Monorepo.** Laravel dan React berada dalam satu repository Git. Ini valid dan umum dipakai — React tidak di-serve oleh Laravel, melainkan berjalan sebagai SPA terpisah (dev server Vite), tapi semua kodenya ada dalam satu folder project.

### Struktur Direktori

```
tradeportal/                        ← root repo, ini adalah project Laravel 12
├── app/
│   ├── Filament/                   ← Resources, Widgets, Pages Filament
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── Api/                ← semua API controller (JSON response)
│   │   │   └── Auth/               ← OAuthController
│   │   ├── Middleware/
│   │   └── Requests/
│   ├── Models/                     ← 11 model
│   ├── Observers/
│   ├── Policies/
│   ├── Services/                   ← 6 service class
│   └── Jobs/                       ← 3 queue jobs
├── database/
│   ├── factories/                  ← 11 factory
│   ├── migrations/                 ← 11 migration
│   └── seeders/                    ← 7 seeder
├── routes/
│   ├── api.php                     ← semua route /api/*
│   └── web.php                     ← route OAuth + fallback ke React SPA
├── storage/
│   └── app/public/
│       ├── proofs/                 ← bukti transfer
│       └── products/               ← gambar produk
├── resources/
│   └── views/
│       └── app.blade.php           ← satu-satunya blade view, entry point React SPA
│
├── frontend/                       ← ⬅ React 19 + TypeScript + Vite ada di sini
│   ├── src/
│   │   ├── pages/                  ← satu file per halaman (Home, Checkout, Payment, dll)
│   │   ├── components/             ← komponen reusable
│   │   ├── hooks/                  ← custom hooks (useAuth, useOrder, dll)
│   │   ├── services/               ← axios API calls
│   │   ├── types/                  ← TypeScript type definitions
│   │   └── main.tsx                ← entry point React
│   ├── public/
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
│
├── .env
├── .env.example
├── composer.json
└── artisan
```

### Cara Kerja Monorepo Ini

**Development (dua terminal):**
```bash
# Terminal 1 — Laravel API server
php artisan serve          # berjalan di http://localhost:8000

# Terminal 2 — React dev server
cd frontend
npm run dev                # berjalan di http://localhost:5173
```

React (port 5173) memanggil Laravel API (port 8000). CORS dikonfigurasi di Laravel agar menerima request dari `http://localhost:5173`.

**Production (satu domain):**
```bash
# Build React
cd frontend && npm run build
# Output masuk ke /public/build/ (dikonfigurasi di vite.config.ts)
```

Di production, Nginx melayani semua request:
- `/api/*` dan `/auth/*` → diteruskan ke Laravel (PHP-FPM)
- Semua route lain → `index.html` hasil build React (SPA routing)

```nginx
# Nginx config production (ringkasan)
location /api { try_files $uri $uri/ /index.php?$query_string; }
location /auth { try_files $uri $uri/ /index.php?$query_string; }
location /admin { try_files $uri $uri/ /index.php?$query_string; }
location / {
    root /path/to/tradeportal/public/build;
    try_files $uri $uri/ /index.html;   # fallback ke React SPA
}
```

**`vite.config.ts` di `/frontend`:**
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': 'http://localhost:8000',       // proxy API ke Laravel saat dev
      '/auth': 'http://localhost:8000',
      '/sanctum': 'http://localhost:8000',
    }
  },
  build: {
    outDir: '../public/build',               // output build masuk ke Laravel public/
    emptyOutDir: true,
  }
})
```

> Dengan proxy di Vite, saat development React dan Laravel berjalan di port berbeda tapi React memanggil `/api/...` tanpa perlu tulis `http://localhost:8000` — tidak ada CORS issue di development.

### `.env` — Tambahan untuk Monorepo
```env
# Tambahkan ini ke .env untuk CORS (React dev server)
FRONTEND_URL=http://localhost:5173        # development
# FRONTEND_URL=https://yourdomain.com    # production (uncomment saat deploy)
```

### `config/cors.php` — yang Perlu Dikonfigurasi
```php
'allowed_origins' => [env('FRONTEND_URL', 'http://localhost:5173')],
'supports_credentials' => true,   // wajib true untuk session cookie (Sanctum SPA)
```

---

## 20. Urutan Development

1. Setup Laravel 12 + konfigurasi dasar (session, queue, scheduler, timezone Asia/Jakarta) + init React 19 di `/frontend` (Vite + TypeScript) + konfigurasi `vite.config.ts` untuk proxy dan build output (section 19.2) + konfigurasi CORS
2. Buat 11 migration sesuai schema (termasuk `product_price_histories`, kolom `is_admin` di `users`)
3. Buat 11 Model + Relationship + Policy
4. **Buat 11 Factory + 7 Seeder (section 22)** — jalankan `migrate:fresh --seed` untuk verifikasi semua relasi benar
5. Setup Filament 5: konfigurasi `canAccessPanel()` di model User + `AdminSeeder` (section 5.3)
6. Setup Google OAuth credentials (section 19.1) + OAuthController + middleware `auth` — pastikan `is_admin = false` di-set eksplisit saat register
7. Buat service classes: `CouponValidator`, `UniqueAmountGenerator`, `DeliveryTemplateRenderer`, `OrderCodeGenerator`, `DeliveryDispatcher`, `OrderExpiryService`
8. Buat Observer: `OrderObserver` (order code, expired_at, sold_count), `ProductObserver` (price history, sitemap cache)
9. Buat Queue Jobs: `EmailDeliveryJob`, `AdminNotificationJob`, `ExpireOrdersJob`
10. Setup Laravel Scheduler untuk `ExpireOrdersJob` (setiap 5 menit) + backup harian
11. Buat API controllers sesuai section 11 (semua endpoint terdokumentasi)
12. Tambah endpoint: polling, resend-delivery, resend-confirmation, dashboard, config
13. Setup Filament 5: 4 Resource + 5 Widget + custom actions (modal Detail Delivery, Preview Email, Isi & Kirim Delivery, tab Pembeli, tab Riwayat Harga)
14. Setup React 19 frontend: routing, auth flow, semua halaman (dashboard, polling, banner email gagal)
15. Implementasi SEO: meta tags, Open Graph, sitemap.xml
16. Integrasi Resend API untuk email delivery
17. Setup spatie/laravel-backup + konfigurasi backup harian
18. Testing end-to-end: flow transaksi, manual delivery (custom_delivery_content), order expiry, delivery retry, polling, resend, validasi produk
19. Setup production: Nginx, SSL, Supervisor untuk queue worker + scheduler

---

## 22. Database Seeders & Factories

Semua Factory dan Seeder harus dibuat **setelah migration dan model selesai (step 3)**, sebelum development fitur apapun. Tanpa data dummy, testing setiap endpoint harus dilakukan manual — sangat membuang waktu.

### 22.1 Factories (11 Factory)

Setiap model memiliki Factory-nya sendiri. Gunakan `php artisan make:factory NamaFactory`.

**`UserFactory`**
```php
// Mengextend default Laravel UserFactory
'name'              => fake()->name(),
'email'             => fake()->unique()->safeEmail(),
'avatar'            => fake()->imageUrl(100, 100),
'provider'          => 'google',
'provider_id'       => fake()->numerify('####################'),
'password'          => null,   // pure OAuth user
'email_verified_at' => now(),
'is_admin'          => false,
```
> Tambahkan state `admin()` untuk membuat akun admin:
> `'password' => Hash::make('password'), 'is_admin' => true, 'provider' => null`

**`ProductFactory`**
```php
'name'             => fake()->words(3, true),
'slug'             => fn(array $attr) => Str::slug($attr['name']),
'type'             => fake()->randomElement(['screener','indicator','kelas','other']),
'short_description'=> fake()->sentence(),
'description'      => fake()->paragraphs(3, true),
'price'            => fake()->randomElement([99000, 149000, 199000, 299000, 499000]),
'thumbnail'        => null,
'images'           => null,
'delivery_type'    => 'auto',
'email_subject'    => 'Akses {product_name} — Kode Order {order_code}',
'email_template'   => 'Halo {user_name}, terima kasih sudah membeli {product_name}. Berikut akses Anda.',
'delivery_content' => '<p>Akses produk: <a href="https://example.com">klik di sini</a></p>',
'is_active'        => true,
'sold_count'       => 0,
'sort_order'       => fake()->numberBetween(0, 100),
```
> Tambahkan state `inactive()` dan `manual()` (delivery_type = manual).

**`AffiliateFactory`**
```php
'name'      => fake()->name(),
'code'      => strtoupper(fake()->unique()->lexify('???')),
'user_id'   => null,
'is_active' => true,
```

**`CouponFactory`**
```php
'affiliate_id' => null,
'code'         => strtoupper(fake()->unique()->lexify('??????')),
'type'         => fake()->randomElement(['percent', 'flat']),
'value'        => fn(array $attr) => $attr['type'] === 'percent' ? fake()->numberBetween(5, 30) : fake()->randomElement([10000, 20000, 50000]),
'min_purchase' => null,
'max_uses'     => fake()->optional()->numberBetween(10, 100),
'used_count'   => 0,
'expired_at'   => fake()->optional()->dateTimeBetween('+1 month', '+6 months'),
'is_active'    => true,
```

**`OrderFactory`** — state lengkap untuk semua status
```php
'code'       => fn() => 'ORD-' . now()->format('Ymd') . '-' . strtoupper(Str::random(4)),
'user_id'    => User::factory(),
'coupon_id'  => null,
'subtotal'   => 299000,
'discount_amount' => 0,
'final_price'=> 299000,
'unique_amount' => fn(array $attr) => $attr['final_price'] + fake()->numberBetween(100, 999),
'status'     => 'pending',
'expired_at' => now()->addHours(24),
```
> State: `pending()`, `paid()`, `verified()`, `completed()`, `cancelled()` — masing-masing set status + timestamp yang relevan.

**`OrderItemFactory`**
```php
'order_id'   => Order::factory(),
'product_id' => Product::factory(),
'price'      => 299000,
```

**`PaymentFactory`**
```php
'order_id'    => Order::factory()->paid(),
'proof_image' => 'proofs/dummy-proof.jpg',
'submitted_at'=> now(),
'verified_at' => null,
'verified_by' => null,
```

**`ProductDeliveryFactory`**
```php
'order_id'               => Order::factory()->completed(),
'product_id'             => Product::factory(),
'custom_delivery_content'=> null,
'status'                 => 'sent',
'sent_at'                => now(),
'error_message'          => null,
'retry_count'            => 0,
```
> State: `pending()`, `sent()`, `failed()`.

**`NotificationFactory`**
```php
'user_id'  => User::factory(),
'order_id' => null,
'type'     => 'order_confirmed',
'payload'  => ['subject' => 'Konfirmasi Order', 'body' => 'Test notifikasi'],
'status'   => 'sent',
'sent_at'  => now(),
```

**`OrderLogFactory`**
```php
'order_id'    => Order::factory(),
'user_id'     => null,
'action'      => 'order_created',
'from_status' => null,
'to_status'   => 'pending',
'notes'       => null,
```

**`ProductPriceHistoryFactory`**
```php
'product_id' => Product::factory(),
'old_price'  => 199000,
'new_price'  => 299000,
'changed_by' => User::factory()->admin(),
'changed_at' => now(),
'notes'      => null,
```

---

### 22.2 Seeders

**`DatabaseSeeder`** — entry point utama, urutan penting karena FK constraints:
```php
public function run(): void
{
    $this->call([
        AdminSeeder::class,          // 1. Admin user dulu
        ProductSeeder::class,        // 2. Produk
        AffiliateSeeder::class,      // 3. Affiliate
        CouponSeeder::class,         // 4. Kupon (butuh affiliate)
        UserSeeder::class,           // 5. Sample user biasa
        OrderSeeder::class,          // 6. Sample orders (butuh user + produk + kupon)
        DummyProofImageSeeder::class,// 7. Copy dummy proof image ke storage
    ]);
}
```

**`AdminSeeder`** — wajib ada, dijalankan pertama kali di production juga:
```php
User::create([
    'name'              => 'Super Admin',
    'email'             => 'admin@tradeportal.id',
    'password'          => Hash::make('admin123'),  // ganti di production!
    'is_admin'          => true,
    'email_verified_at' => now(),
    'provider'          => null,
    'provider_id'       => null,
]);
```
> ⚠ Di production, jalankan `AdminSeeder` sekali, lalu segera ganti password via Filament.

**`ProductSeeder`** — 4 produk sample, satu per type:
```php
Product::factory()->create(['name' => 'Screener Saham Pro', 'type' => 'screener', 'price' => 299000]);
Product::factory()->create(['name' => 'Indikator RSI Custom', 'type' => 'indicator', 'price' => 149000]);
Product::factory()->create(['name' => 'Kelas Trading Pemula', 'type' => 'kelas', 'price' => 499000, 'delivery_type' => 'manual']);
Product::factory()->inactive()->create(['name' => 'Produk Draft', 'type' => 'other', 'price' => 99000]);
```

**`AffiliateSeeder`**:
```php
Affiliate::factory()->count(3)->create();
```

**`CouponSeeder`** — variasi kupon untuk testing semua skenario:
```php
// Kupon persen aktif
Coupon::factory()->create(['code' => 'DISKON10', 'type' => 'percent', 'value' => 10]);
// Kupon flat aktif
Coupon::factory()->create(['code' => 'HEMAT50K', 'type' => 'flat', 'value' => 50000, 'min_purchase' => 100000]);
// Kupon sudah expired
Coupon::factory()->create(['code' => 'EXPIRED', 'type' => 'percent', 'value' => 20, 'expired_at' => now()->subDay()]);
// Kupon sudah habis pemakaian
Coupon::factory()->create(['code' => 'HABIS', 'type' => 'percent', 'value' => 15, 'max_uses' => 5, 'used_count' => 5]);
// Kupon nonaktif
Coupon::factory()->create(['code' => 'NONAKTIF', 'type' => 'percent', 'value' => 25, 'is_active' => false]);
```

**`UserSeeder`** — 5 sample user biasa:
```php
User::factory()->count(5)->create();
```

**`OrderSeeder`** — sample orders untuk semua status agar widget dashboard admin langsung terisi:
```php
$users    = User::where('is_admin', false)->get();
$products = Product::where('is_active', true)->get();

// Pending order (belum bayar)
Order::factory()->pending()->has(OrderItem::factory()->for($products->first()))->for($users->first())->create();

// Paid order (sudah upload bukti, belum diverifikasi)
Order::factory()->paid()->has(OrderItem::factory()->for($products->get(1)))->for($users->get(1))->create();

// Completed order (sudah selesai, ada delivery)
Order::factory()->completed()
    ->has(OrderItem::factory()->for($products->get(2)))
    ->has(ProductDelivery::factory()->sent()->for($products->get(2)))
    ->for($users->get(2))
    ->create();

// Cancelled order
Order::factory()->cancelled()->has(OrderItem::factory()->for($products->first()))->for($users->get(3))->create();

// Completed order dengan delivery failed (untuk test widget "Delivery gagal")
Order::factory()->completed()
    ->has(OrderItem::factory()->for($products->first()))
    ->has(ProductDelivery::factory()->failed()->for($products->first()))
    ->for($users->get(4))
    ->create();
```

**`DummyProofImageSeeder`** — copy file dummy ke storage agar bukti transfer bisa ditampilkan di Filament:
```php
// Buat direktori jika belum ada
Storage::makeDirectory('public/proofs');
// Copy dari resources/testing/dummy-proof.jpg ke storage
// Sertakan file dummy-proof.jpg di resources/testing/ dalam repo
```

---

### 22.3 Perintah Artisan untuk Development

```bash
# Reset penuh + seed ulang (development only)
php artisan migrate:fresh --seed

# Jalankan seeder tertentu saja
php artisan db:seed --class=ProductSeeder

# Buat akun admin baru via CLI (Filament 5)
php artisan make:filament-user

# Production: jalankan AdminSeeder saja (sekali)
php artisan db:seed --class=AdminSeeder
```

---

## 21. Catatan Implementasi

### Versi & Kompatibilitas — BACA INI PERTAMA

> ⚠ **Setiap instruksi coding harus menyebut versi eksplisit.** Banyak library di stack ini punya breaking change besar antar versi. Jangan asumsikan agent/developer tahu versi mana yang dipakai.

| Library | Versi yang Dipakai | Breaking Change Utama vs Versi Sebelumnya |
|---|---|---|
| **Laravel** | **12.x** | Constructor promotion wajib, typed properties, beberapa helper deprecated |
| **Filament** | **5.x** | Syntax Resource, Form, Table, Action **berbeda total dari v3** — jangan pakai contoh kode v3 |
| **React** | **19.x** | `use()` hook baru, compiler, beberapa deprecated pattern dari v18 |
| **Laravel Socialite** | **5.x** | API sama, tapi registrasi provider berbeda |
| **Resend Laravel** | **0.10.x** | — |

**Cara menyebut versi di prompt agent:**
```
Gunakan Laravel 12 (bukan 10 atau 11).
Gunakan Filament 5 (bukan v3 — syntaxnya BERBEDA TOTAL).
Gunakan React 19 dengan TypeScript.
Jangan gunakan sintaks atau method yang deprecated.
```

**Referensi dokumentasi yang harus selalu dicek:**
- Filament 5: https://filamentphp.com/docs (pastikan versi dropdown = v5)
- Laravel 12: https://laravel.com/docs/12.x
- React 19: https://react.dev
- Telegram Bot SDK: https://telegram-bot-sdk.readme.io/docs

---

### Catatan Teknis

- `UniqueAmountGenerator` dipanggil dalam database transaction yang sama dengan pembuatan order — pastikan tidak ada race condition
- `unique_amount` adalah yang ditampilkan ke user di halaman pembayaran dan email konfirmasi, bukan `final_price`
- Admin melihat `unique_amount` di kolom utama OrderResource — ini yang dicocokkan dengan mutasi rekening
- `DeliveryDispatcher` dipanggil dari dalam queue job, bukan langsung dari controller
- Template delivery dirender oleh `DeliveryTemplateRenderer` sebelum disimpan ke `product_deliveries`
- `DeliveryTemplateRenderer` menggunakan `custom_delivery_content` jika ada, fallback ke `products.delivery_content`
- Untuk produk `manual`, admin wajib mengisi `custom_delivery_content` via modal "Isi & Kirim Delivery" — field ini wajib diisi (tidak boleh kosong) sebelum delivery bisa dikirim
- `order_logs` mencatat semua aksi termasuk yang dilakukan sistem (user_id null)
- `expired_at` di-set oleh `OrderObserver` saat event `creating`: `now()->addHours(24)` (timezone: Asia/Jakarta)
- `ExpireOrdersJob` harus idempotent — aman dijalankan berkali-kali tanpa efek samping ganda
- `ProductObserver` mencatat `product_price_histories` setiap kali field `price` berubah — inject auth user sebagai `changed_by`
- `sold_count` di-increment via `OrderObserver` saat order status berubah ke `completed`
- Preview email di ProductResource menggunakan data dummy — tidak boleh trigger pengiriman nyata
- Queue health widget membaca dari tabel `jobs` (Laravel Queue database driver) — tampilkan jumlah dan job terlama
- Endpoint polling `GET /api/orders/{code}/status` hanya mengembalikan `{status, completed_at}` — tidak ada data sensitif lain
- Resend delivery oleh user (`POST /api/orders/{code}/resend-delivery`) hanya diizinkan jika `product_deliveries.status = 'failed'` dan sudah lebih dari 1 jam sejak percobaan terakhir — validasi di server
- Resend confirmation (`POST /api/orders/{code}/resend-confirmation`) menggunakan record `notifications` dengan type `order_confirmed` untuk cek status dan rate limit
- Validasi `is_active` produk dilakukan di `CheckoutController` sebelum masuk transaction, bukan hanya di frontend
- Validasi kelengkapan produk (`email_template`, `delivery_content`) dilakukan di `ProductResource` via Filament lifecycle hook `saving`
- Sitemap.xml di-cache dan di-invalidate otomatis saat ada perubahan `is_active` produk via `ProductObserver`
- Pagination default 15 per halaman, bisa di-override via query param `?per_page=` (max 50)
- `APP_TIMEZONE=Asia/Jakarta` — semua `now()`, `Carbon::now()`, dan timestamp database menggunakan WIB
- Frontend menampilkan semua waktu dalam WIB — tidak perlu konversi timezone di sisi client
- **Filament 5** — sintaks berbeda dari v3, pastikan mengacu dokumentasi Filament 5
- **Laravel 12** — gunakan typed properties dan constructor property promotion
