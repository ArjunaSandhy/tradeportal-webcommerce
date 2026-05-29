# TradePortal — Test Scenarios End-to-End

> Gunakan dokumen ini di **Step 18** (Testing end-to-end).
> Jalankan setiap skenario secara manual setelah `migrate:fresh --seed`.
> Semua skenario diasumsikan berjalan dengan queue worker aktif (`php artisan queue:work`).

---

## Setup Sebelum Testing

```bash
php artisan migrate:fresh --seed
php artisan storage:link
php artisan queue:work --daemon &
php artisan serve &
cd frontend && npm run dev &
```

Akun yang tersedia setelah seed:
- **Admin:** `admin@tradeportal.id` / `admin123` → `/admin`
- **User test:** login via Google OAuth (gunakan akun Google testing)

---

## Skenario 1 — Happy Path Checkout (Auto Delivery)

**Tujuan:** Memastikan seluruh alur transaksi dari awal sampai user bisa akses produk berjalan tanpa hambatan.

**Prasyarat:** Produk "Screener Saham Pro" (`delivery_type = auto`, `is_active = true`) sudah ada dari seeder.

### Langkah:

| # | Aksi | Yang Diverifikasi |
|---|------|-------------------|
| 1 | Buka `/` sebagai guest | Grid produk tampil, harga dalam format Rupiah |
| 2 | Klik produk "Screener Saham Pro" | Halaman detail tampil, tombol "Beli Sekarang" aktif |
| 3 | Klik "Beli Sekarang" tanpa login | Redirect ke Google OAuth |
| 4 | Login via Google OAuth | Redirect kembali ke halaman produk setelah login |
| 5 | Klik "Beli Sekarang" | Redirect ke `/checkout/{slug}` |
| 6 | Di checkout: nama + email user tampil (read-only) | Data dari OAuth benar |
| 7 | Klik "Konfirmasi & Bayar" tanpa kupon | Order dibuat, redirect ke `/payment/{code}` |
| 8 | Verifikasi halaman payment | `unique_amount` tampil dengan font-mono, BUKAN `final_price` |
| 9 | Verifikasi `unique_amount` | Nilai = `final_price + [100-999]`, bukan sama persis dengan harga produk |
| 10 | Verifikasi countdown timer | Menunjukkan ~24 jam, format `HH:MM:SS` |
| 11 | Upload file gambar dummy (< 5MB) sebagai bukti | Preview gambar muncul sebelum submit |
| 12 | Submit bukti transfer | Redirect ke `/order/{code}/success` |
| 13 | Cek database: `orders.status` | Harus `paid` |
| 14 | Cek database: `payments` record | Ada record dengan `proof_image` terisi |
| 15 | Buka `/admin` sebagai admin | Order muncul di OrderResource dengan status "paid" (badge kuning) |
| 16 | Klik "Lihat Bukti" pada order | Modal preview gambar muncul |
| 17 | Klik "Verify Payment" | Status berubah ke `verified`, queue job di-dispatch |
| 18 | Tunggu queue worker proses (~5 detik) | Status berubah ke `completed` |
| 19 | Cek inbox email user | Email delivery diterima dengan subject dan konten benar, placeholder sudah ter-replace |
| 20 | Buka `/dashboard` sebagai user | Produk muncul di "Produk Saya" dengan badge "Akses Aktif" |
| 21 | Klik produk di dashboard | Halaman `/dashboard/products/{order_id}` tampil dengan konten akses |
| 22 | Kembali ke `/products/{slug}` | Tombol "Beli Sekarang" **diganti** teks "Kamu sudah memiliki produk ini" |

**Expected final state:**
- `orders.status = completed`
- `product_deliveries.status = sent`
- `order_logs`: 4 records (order_created, payment_uploaded, payment_verified, delivery_sent)
- `products.sold_count` bertambah 1

---

## Skenario 2 — Race Condition Checkout

**Tujuan:** Memastikan dua request checkout bersamaan untuk produk + user yang sama hanya menghasilkan SATU order.

**Prasyarat:** User sudah login. Produk aktif tersedia.

### Langkah:

| # | Aksi | Yang Diverifikasi |
|---|------|-------------------|
| 1 | Buka dua tab browser dengan halaman checkout produk yang sama | Keduanya tampil normal |
| 2 | Klik "Konfirmasi & Bayar" di kedua tab **hampir bersamaan** (< 1 detik) | — |
| 3 | Satu tab berhasil redirect ke `/payment/{code}` | Order pertama dibuat |
| 4 | Tab lain menampilkan error | Pesan: "Kamu masih memiliki order aktif untuk produk ini" |
| 5 | Error message berisi link ke order aktif | Link mengarah ke `/payment/{code}` yang sama |
| 6 | Cek database: `orders` untuk user + produk ini | Hanya **satu** record dengan status `pending` |

**Catatan teknis yang diverifikasi:** `SELECT ... FOR UPDATE` dalam transaction mencegah insert duplikat.

---

## Skenario 3 — Order Expiry & Coupon Decrement

**Tujuan:** Memastikan order `pending` yang melewati 24 jam otomatis di-cancel, dan `coupons.used_count` dikembalikan.

**Prasyarat:** Kupon `DISKON10` dengan `used_count = 0`, `max_uses = 10` tersedia dari seeder.

### Langkah:

| # | Aksi | Yang Diverifikasi |
|---|------|-------------------|
| 1 | Buat order baru dengan kupon `DISKON10` | Order berhasil dibuat dengan status `pending` |
| 2 | Cek database: `coupons.used_count` untuk `DISKON10` | Harus `1` (increment saat order dibuat) |
| 3 | Cek database: `orders.expired_at` | Harus = `created_at + 24 jam` |
| 4 | **Simulasi expiry:** update `expired_at` ke masa lalu via tinker:<br>`Order::latest()->first()->update(['expired_at' => now()->subMinutes(1)])` | — |
| 5 | Jalankan job manual: `php artisan schedule:run` atau `php artisan tinker` → `(new \App\Jobs\ExpireOrdersJob)->handle()` | — |
| 6 | Cek database: `orders.status` | Harus `cancelled` |
| 7 | Cek database: `coupons.used_count` untuk `DISKON10` | Harus kembali ke `0` (decrement) |
| 8 | Cek database: `order_logs` untuk order ini | Ada record dengan `action = order_expired`, `user_id = null` |
| 9 | Cek inbox email user | Email notifikasi "Order kedaluwarsa" diterima |
| 10 | Buka `/orders` sebagai user | Order tampil dengan badge "Dibatalkan" |
| 11 | User mencoba buat order baru untuk produk yang sama | Berhasil — order lama sudah cancelled |

**Tambahan — verifikasi idempotency:**
- Jalankan `ExpireOrdersJob` dua kali berturut-turut
- `coupons.used_count` tidak boleh menjadi `-1` (decrement tidak boleh double)

---

## Skenario 4 — Manual Delivery

**Tujuan:** Memastikan alur delivery manual berjalan, admin bisa isi konten custom, dan user melihat konten yang benar di dashboard.

**Prasyarat:** Produk "Kelas Trading Pemula" dengan `delivery_type = manual` tersedia dari seeder.

### Langkah:

| # | Aksi | Yang Diverifikasi |
|---|------|-------------------|
| 1 | User checkout & bayar produk "Kelas Trading Pemula" | Order `pending` → `paid` normal |
| 2 | Admin verify payment di Filament | Status berubah ke `verified` |
| 3 | Verifikasi: tombol "Kirim Ulang Email" TIDAK muncul untuk auto delivery | — |
| 4 | Verifikasi: tombol **"Isi & Kirim Delivery"** muncul karena `delivery_type = manual` | Tombol ada di row order |
| 5 | Klik "Isi & Kirim Delivery" | Modal dengan rich text editor muncul |
| 6 | Coba klik "Kirim" tanpa mengisi konten | Validasi error: field wajib diisi |
| 7 | Isi konten: `<p>Username: trader_123<br>Password: Secret@456</p>` | — |
| 8 | Klik "Kirim" | Modal tutup, email delivery di-dispatch ke queue |
| 9 | Tunggu queue worker | Status order menjadi `completed` |
| 10 | Buka `/dashboard/products/{order_id}` sebagai user | Konten menampilkan `trader_123 / Secret@456` |
| 11 | Cek database: `product_deliveries.custom_delivery_content` | Berisi konten yang diisi admin |
| 12 | Buat order lain untuk produk yang sama (user berbeda) | — |
| 13 | Admin isi konten berbeda untuk order kedua | Konten: `Username: trader_456 / Password: Other@789` |
| 14 | Verifikasi kedua user melihat konten yang berbeda di dashboard | User 1 ≠ User 2 ✓ |

---

## Skenario 5 — Resend Delivery (Retry Logic & Rate Limit)

**Tujuan:** Memastikan user bisa minta kirim ulang email saat delivery gagal, dengan rate limit yang berfungsi.

**Prasyarat:** Ada order `completed` dengan `product_deliveries.status = failed` (dari seeder `OrderSeeder`).

### Langkah:

| # | Aksi | Yang Diverifikasi |
|---|------|-------------------|
| 1 | Buka `/dashboard/products/{order_id}` untuk order dengan delivery failed | Banner warning "Email akses produk gagal terkirim" tampil |
| 2 | Tombol "Kirim Ulang Email" aktif (tidak disabled) | Belum pernah dipakai dalam 1 jam terakhir |
| 3 | Klik "Kirim Ulang Email" | Toast konfirmasi muncul, tombol disabled dengan countdown |
| 4 | Cek database: `product_deliveries.retry_count` | Bertambah 1 |
| 5 | Coba klik tombol lagi (masih dalam 1 jam) | Tombol tetap disabled + countdown timer aktif |
| 6 | Panggil API langsung: `POST /api/orders/{code}/resend-delivery` dalam 1 jam | HTTP 429 Too Many Requests |
| 7 | **Simulasi email berhasil:** pastikan RESEND_API_KEY valid, queue worker jalan | Status berubah ke `sent` |
| 8 | Refresh halaman dashboard | Banner warning **hilang**, konten akses tampil normal |

**Tambahan — verifikasi batas retry:**
- Set `product_deliveries.retry_count = 2` via tinker
- Trigger resend → job jalan → gagal lagi
- `retry_count` menjadi `3`, status menjadi `failed` (tidak retry lagi)
- Cek `order_logs`: ada record `delivery_failed`

---

## Skenario 6 — Polling Status di Halaman Payment

**Tujuan:** Memastikan halaman payment melakukan polling dan redirect otomatis saat order completed.

**Prasyarat:** Ada order `paid` (sudah upload bukti, menunggu verifikasi admin).

### Langkah:

| # | Aksi | Yang Diverifikasi |
|---|------|-------------------|
| 1 | Buka `/payment/{code}` untuk order yang sudah `paid` | Halaman tampil dengan status polling aktif |
| 2 | Buka browser DevTools → Network tab | Filter: XHR/Fetch, cari request ke `/api/orders/{code}/status` |
| 3 | Tunggu 30 detik | Request polling muncul setiap ~30 detik |
| 4 | **Tab tidak aktif:** pindah ke tab lain | Polling **berhenti** (Page Visibility API) |
| 5 | Kembali ke tab payment | Polling **resume** |
| 6 | Admin verify payment di Filament (biarkan queue worker jalan) | — |
| 7 | Tunggu queue worker selesai (~5 detik) | — |
| 8 | Halaman payment mendeteksi status `completed` via polling | Auto-redirect ke `/dashboard` |
| 9 | Toast "Akses produk kamu sudah aktif!" muncul di dashboard | Notifikasi sukses tampil |
| 10 | Cek rate limit: panggil `/api/orders/{code}/status` > 120x dalam 1 jam | HTTP 429 di request ke-121 |

---

## Checklist Pasca Semua Skenario

Setelah semua skenario dijalankan, verifikasi kondisi final:

- [ ] `order_logs` terisi untuk setiap perubahan status di semua order
- [ ] `product_price_histories` mencatat perubahan harga (coba ubah harga produk via Filament)
- [ ] `sold_count` akurat sesuai jumlah order `completed` per produk
- [ ] Widget dashboard admin menampilkan data yang benar (revenue, pending orders, delivery gagal)
- [ ] Queue health widget menunjukkan tidak ada stuck job
- [ ] Sitemap `/sitemap.xml` hanya berisi produk aktif
- [ ] Endpoint `GET /api/config` mengembalikan `admin_whatsapp` dan `qris_image_url`
- [ ] Semua halaman yang `noindex` tidak muncul di sitemap
- [ ] `is_admin` tidak bisa diubah via API (coba PATCH/PUT ke `/api/profile`)
- [ ] User biasa tidak bisa akses `/admin` meski login
