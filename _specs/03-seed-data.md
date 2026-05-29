# TradePortal — Seed Data Lengkap

> Dokumen ini adalah **referensi konten** untuk semua Seeder.
> Agent wajib menggunakan data di bawah ini sebagai nilai literal di seeder,
> bukan `fake()->...` untuk data yang sudah ditentukan di sini.
> Jalankan dengan: `php artisan migrate:fresh --seed`

---

## 1. Admin (`AdminSeeder`)

```php
[
    'name'              => 'Super Admin',
    'email'             => 'admin@tradeportal.id',
    'password'          => Hash::make('TradePortal@2025!'),  // ganti di production
    'is_admin'          => true,
    'email_verified_at' => now(),
    'provider'          => null,
    'provider_id'       => null,
    'avatar'            => null,
]
```

---

## 2. Produk (`ProductSeeder`)

4 produk sample — satu per tipe. Semua `is_active = true` kecuali produk ke-4 (draft).

---

### Produk 1 — Screener Saham Pro
```php
[
    'name'             => 'Screener Saham Pro',
    'slug'             => 'screener-saham-pro',
    'type'             => 'screener',
    'delivery_type'    => 'auto',
    'price'            => 299000,
    'sort_order'       => 1,
    'is_active'        => true,
    'sold_count'       => 0,

    'short_description' => 'Temukan saham potensi breakout setiap hari dalam hitungan detik. Filter ratusan saham BEI otomatis berdasarkan kriteria teknikal pilihan Anda.',

    'description' => <<<HTML
<h2>Apa itu Screener Saham Pro?</h2>
<p>Screener Saham Pro adalah tools berbasis spreadsheet Google Sheets yang terhubung langsung ke data pasar real-time BEI. Setiap pagi sebelum market buka, Anda sudah tahu daftar saham mana yang layak dipantau hari ini.</p>

<h3>Fitur Utama</h3>
<ul>
  <li>Filter otomatis berdasarkan RSI, MACD, volume anomali, dan pola candlestick</li>
  <li>Update data setiap 15 menit saat market buka</li>
  <li>Watchlist personal — tandai dan pantau saham pilihan Anda</li>
  <li>Alert email otomatis jika saham di watchlist memenuhi kriteria entry</li>
  <li>Kompatibel dengan semua broker Indonesia</li>
</ul>

<h3>Cocok untuk Siapa?</h3>
<p>Trader aktif yang tidak punya waktu scan ratusan saham manual setiap hari. Cukup buka screener, lihat shortlist, eksekusi.</p>

<h3>Yang Anda Dapatkan</h3>
<ul>
  <li>Akses seumur hidup ke Google Sheets screener</li>
  <li>Panduan setup dan cara baca sinyal (PDF)</li>
  <li>Update kriteria filter setiap kuartal</li>
  <li>Grup diskusi eksklusif pengguna screener</li>
</ul>
HTML,

    'email_subject'  => 'Akses Screener Saham Pro Anda — {order_code}',

    'email_template' => <<<HTML
<!DOCTYPE html>
<html lang="id">
<head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"></head>
<body style="margin:0;padding:0;background:#0A0B0E;font-family:'Segoe UI',Arial,sans-serif;">
  <table width="100%" cellpadding="0" cellspacing="0" style="background:#0A0B0E;padding:40px 20px;">
    <tr><td align="center">
      <table width="560" cellpadding="0" cellspacing="0" style="background:#111318;border-radius:12px;border:1px solid rgba(255,255,255,0.08);overflow:hidden;">

        <!-- Header -->
        <tr><td style="background:#3B82F6;padding:24px 32px;">
          <p style="margin:0;color:#fff;font-size:20px;font-weight:600;">TradePortal</p>
          <p style="margin:4px 0 0;color:rgba(255,255,255,0.8);font-size:13px;">Platform Trading Tools Terpercaya</p>
        </td></tr>

        <!-- Body -->
        <tr><td style="padding:32px;">
          <p style="margin:0 0 8px;color:#8B8FA8;font-size:13px;">Halo,</p>
          <p style="margin:0 0 24px;color:#F1F2F5;font-size:22px;font-weight:600;">{user_name} 👋</p>

          <p style="margin:0 0 16px;color:#F1F2F5;font-size:15px;line-height:1.6;">
            Terima kasih sudah membeli <strong>Screener Saham Pro</strong>. Pembayaran Anda telah diverifikasi dan akses sudah aktif.
          </p>

          <!-- Access Box -->
          <table width="100%" cellpadding="0" cellspacing="0" style="background:#1A1D26;border-radius:8px;border:1px solid rgba(59,130,246,0.3);margin:0 0 24px;">
            <tr><td style="padding:20px 24px;">
              <p style="margin:0 0 12px;color:#3B82F6;font-size:12px;font-weight:600;text-transform:uppercase;letter-spacing:0.5px;">Informasi Akses Anda</p>
              <p style="margin:0 0 8px;color:#8B8FA8;font-size:13px;">Kode Order</p>
              <p style="margin:0 0 16px;color:#F1F2F5;font-size:15px;font-family:monospace;font-weight:600;">{order_code}</p>
              <p style="margin:0 0 4px;color:#8B8FA8;font-size:13px;">Link Google Sheets Screener</p>
              <p style="margin:0;color:#F1F2F5;font-size:14px;">📋 Cek dashboard Anda untuk detail akses lengkap</p>
            </td></tr>
          </table>

          <p style="margin:0 0 24px;color:#8B8FA8;font-size:14px;line-height:1.6;">
            Semua detail akses (link Google Sheets, panduan, dan link grup) tersedia di <strong style="color:#F1F2F5;">dashboard TradePortal</strong> Anda kapan saja.
          </p>

          <!-- CTA Button -->
          <table cellpadding="0" cellspacing="0" style="margin:0 0 24px;">
            <tr><td style="background:#3B82F6;border-radius:8px;">
              <a href="https://tradeportal.id/dashboard" style="display:inline-block;padding:14px 28px;color:#fff;font-size:14px;font-weight:600;text-decoration:none;">
                Buka Dashboard →
              </a>
            </td></tr>
          </table>

          <p style="margin:0;color:#4B4F63;font-size:13px;line-height:1.6;">
            Ada pertanyaan? Hubungi kami via WhatsApp atau balas email ini.
          </p>
        </td></tr>

        <!-- Footer -->
        <tr><td style="padding:20px 32px;border-top:1px solid rgba(255,255,255,0.06);">
          <p style="margin:0;color:#4B4F63;font-size:12px;">© 2025 TradePortal · {order_code}</p>
        </td></tr>

      </table>
    </td></tr>
  </table>
</body>
</html>
HTML,

    'delivery_content' => <<<HTML
<h2>Selamat datang di Screener Saham Pro!</h2>
<p>Berikut semua yang Anda butuhkan untuk mulai menggunakan screener:</p>

<h3>🔗 Link Akses</h3>
<ul>
  <li><strong>Google Sheets Screener:</strong> <a href="https://docs.google.com/spreadsheets/d/GANTI_DENGAN_LINK_ASLI" target="_blank">Klik untuk buka screener →</a></li>
  <li><strong>Cara duplikat ke Google Drive Anda:</strong> File → Make a copy</li>
</ul>

<h3>📖 Panduan Setup</h3>
<ol>
  <li>Buka link screener di atas</li>
  <li>Klik <strong>File → Make a copy</strong> untuk menyimpan ke Google Drive Anda</li>
  <li>Ikuti instruksi di sheet "SETUP" untuk menghubungkan data market</li>
  <li>Selesai — screener siap digunakan setiap hari market buka</li>
</ol>

<h3>💬 Grup Diskusi Eksklusif</h3>
<p>Join grup Telegram pengguna Screener Saham Pro: <a href="https://t.me/GANTI_DENGAN_LINK_GRUP" target="_blank">t.me/ScreenerSahamPro</a></p>

<h3>📞 Butuh Bantuan?</h3>
<p>Hubungi admin via WhatsApp untuk pertanyaan teknis atau bantuan setup.</p>
HTML,
]
```

---

### Produk 2 — Indikator Supply & Demand MT4/MT5
```php
[
    'name'             => 'Indikator Supply & Demand MT4/MT5',
    'slug'             => 'indikator-supply-demand-mt4-mt5',
    'type'             => 'indicator',
    'delivery_type'    => 'auto',
    'price'            => 199000,
    'sort_order'       => 2,
    'is_active'        => true,
    'sold_count'       => 0,

    'short_description' => 'Indikator custom MetaTrader yang otomatis mendeteksi dan menggambar zona Supply & Demand di semua timeframe. Tidak perlu gambar manual lagi.',

    'description' => <<<HTML
<h2>Indikator Supply & Demand Otomatis</h2>
<p>Hentikan kebiasaan gambar zona supply & demand secara manual yang makan waktu berjam-jam. Indikator ini bekerja di background dan menggambar semua zona penting secara otomatis di chart MetaTrader Anda.</p>

<h3>Fitur</h3>
<ul>
  <li>Deteksi otomatis zona Supply (penawaran) dan Demand (permintaan) di semua timeframe</li>
  <li>Zona di-grade berdasarkan kekuatan: Strong / Moderate / Weak</li>
  <li>Alert popup + suara saat harga mendekati zona</li>
  <li>Kompatibel MT4 dan MT5</li>
  <li>Bekerja di semua pair forex, indeks, dan komoditas</li>
  <li>Tidak repaint</li>
</ul>

<h3>Yang Anda Dapatkan</h3>
<ul>
  <li>File indikator (.ex4 dan .ex5)</li>
  <li>Panduan instalasi dan konfigurasi (PDF + video)</li>
  <li>Panduan cara trading dengan zona supply & demand</li>
  <li>Update gratis seumur hidup</li>
</ul>
HTML,

    'email_subject'  => 'File Indikator Supply & Demand Siap Diunduh — {order_code}',

    'email_template' => <<<HTML
<!DOCTYPE html>
<html lang="id">
<head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"></head>
<body style="margin:0;padding:0;background:#0A0B0E;font-family:'Segoe UI',Arial,sans-serif;">
  <table width="100%" cellpadding="0" cellspacing="0" style="background:#0A0B0E;padding:40px 20px;">
    <tr><td align="center">
      <table width="560" cellpadding="0" cellspacing="0" style="background:#111318;border-radius:12px;border:1px solid rgba(255,255,255,0.08);overflow:hidden;">
        <tr><td style="background:#3B82F6;padding:24px 32px;">
          <p style="margin:0;color:#fff;font-size:20px;font-weight:600;">TradePortal</p>
          <p style="margin:4px 0 0;color:rgba(255,255,255,0.8);font-size:13px;">Platform Trading Tools Terpercaya</p>
        </td></tr>
        <tr><td style="padding:32px;">
          <p style="margin:0 0 8px;color:#8B8FA8;font-size:13px;">Halo,</p>
          <p style="margin:0 0 24px;color:#F1F2F5;font-size:22px;font-weight:600;">{user_name} 👋</p>
          <p style="margin:0 0 16px;color:#F1F2F5;font-size:15px;line-height:1.6;">
            Pembelian <strong>Indikator Supply & Demand MT4/MT5</strong> Anda sudah diverifikasi. File siap diunduh dari dashboard.
          </p>
          <table width="100%" cellpadding="0" cellspacing="0" style="background:#1A1D26;border-radius:8px;border:1px solid rgba(59,130,246,0.3);margin:0 0 24px;">
            <tr><td style="padding:20px 24px;">
              <p style="margin:0 0 12px;color:#3B82F6;font-size:12px;font-weight:600;text-transform:uppercase;letter-spacing:0.5px;">Detail Pembelian</p>
              <p style="margin:0 0 4px;color:#8B8FA8;font-size:13px;">Kode Order</p>
              <p style="margin:0 0 16px;color:#F1F2F5;font-size:15px;font-family:monospace;font-weight:600;">{order_code}</p>
              <p style="margin:0;color:#8B8FA8;font-size:13px;">File .ex4, .ex5, panduan PDF, dan video tutorial tersedia di dashboard.</p>
            </td></tr>
          </table>
          <table cellpadding="0" cellspacing="0" style="margin:0 0 24px;">
            <tr><td style="background:#3B82F6;border-radius:8px;">
              <a href="https://tradeportal.id/dashboard" style="display:inline-block;padding:14px 28px;color:#fff;font-size:14px;font-weight:600;text-decoration:none;">
                Unduh File Indikator →
              </a>
            </td></tr>
          </table>
          <p style="margin:0;color:#4B4F63;font-size:13px;">Ada kendala instalasi? Hubungi admin via WhatsApp.</p>
        </td></tr>
        <tr><td style="padding:20px 32px;border-top:1px solid rgba(255,255,255,0.06);">
          <p style="margin:0;color:#4B4F63;font-size:12px;">© 2025 TradePortal · {order_code}</p>
        </td></tr>
      </table>
    </td></tr>
  </table>
</body>
</html>
HTML,

    'delivery_content' => <<<HTML
<h2>File Indikator Supply & Demand Anda</h2>
<p>Terima kasih sudah membeli! Berikut link unduhan dan panduan instalasi:</p>

<h3>📥 Link Unduhan</h3>
<ul>
  <li><strong>MT4 (.ex4):</strong> <a href="https://drive.google.com/GANTI_LINK_EX4" target="_blank">Unduh SupplyDemand_v2.ex4</a></li>
  <li><strong>MT5 (.ex5):</strong> <a href="https://drive.google.com/GANTI_LINK_EX5" target="_blank">Unduh SupplyDemand_v2.ex5</a></li>
  <li><strong>Panduan PDF:</strong> <a href="https://drive.google.com/GANTI_LINK_PDF" target="_blank">Unduh Panduan Lengkap.pdf</a></li>
  <li><strong>Video Tutorial:</strong> <a href="https://youtu.be/GANTI_VIDEO_ID" target="_blank">Tonton di YouTube (Private)</a></li>
</ul>

<h3>⚙️ Cara Instalasi MT4</h3>
<ol>
  <li>Buka MetaTrader 4 → File → Open Data Folder</li>
  <li>Masuk ke folder <code>MQL4/Indicators</code></li>
  <li>Copy file <code>SupplyDemand_v2.ex4</code> ke folder tersebut</li>
  <li>Restart MetaTrader</li>
  <li>Indikator akan muncul di Navigator → Custom Indicators</li>
</ol>

<h3>⚙️ Cara Instalasi MT5</h3>
<ol>
  <li>Buka MetaTrader 5 → File → Open Data Folder</li>
  <li>Masuk ke folder <code>MQL5/Indicators</code></li>
  <li>Copy file <code>SupplyDemand_v2.ex5</code></li>
  <li>Restart MetaTrader</li>
</ol>

<h3>📞 Butuh Bantuan Instalasi?</h3>
<p>Hubungi admin via WhatsApp — kami siap bantu remote jika diperlukan.</p>
HTML,
]
```

---

### Produk 3 — Kelas Trading Pemula: Dari Nol ke Profit Konsisten
```php
[
    'name'             => 'Kelas Trading Pemula: Dari Nol ke Profit Konsisten',
    'slug'             => 'kelas-trading-pemula-dari-nol-ke-profit-konsisten',
    'type'             => 'kelas',
    'delivery_type'    => 'manual',   // admin isi akses per murid
    'price'            => 499000,
    'sort_order'       => 3,
    'is_active'        => true,
    'sold_count'       => 0,

    'short_description' => 'Kelas online intensif 4 minggu untuk pemula yang ingin belajar trading saham dari dasar sampai bisa konsisten profit. Live session + rekaman + mentoring langsung.',

    'description' => <<<HTML
<h2>Kelas Trading Pemula</h2>
<p>Bukan sekadar video recording. Ini program belajar terstruktur dengan live session, rekaman, tugas, dan sesi tanya jawab langsung bersama mentor berpengalaman.</p>

<h3>Kurikulum (4 Minggu)</h3>
<ul>
  <li><strong>Minggu 1:</strong> Dasar pasar modal, cara baca laporan keuangan, pilih broker</li>
  <li><strong>Minggu 2:</strong> Analisis teknikal dasar — candlestick, support/resistance, tren</li>
  <li><strong>Minggu 3:</strong> Manajemen risiko, position sizing, psikologi trading</li>
  <li><strong>Minggu 4:</strong> Strategi entry & exit, backtest, bangun trading plan pribadi</li>
</ul>

<h3>Format Belajar</h3>
<ul>
  <li>4x live session via Zoom (Sabtu malam, 2 jam)</li>
  <li>Rekaman semua sesi — tonton ulang kapan saja</li>
  <li>Grup Telegram eksklusif peserta untuk diskusi harian</li>
  <li>2x sesi Q&A dengan mentor (30 menit, jadwal fleksibel)</li>
  <li>Materi PDF + worksheet per minggu</li>
</ul>

<h3>Batch Berikutnya</h3>
<p>Kelas dibuka setiap bulan. Setelah pembelian, admin akan menghubungi Anda untuk konfirmasi jadwal batch.</p>
HTML,

    'email_subject'  => 'Selamat! Pendaftaran Kelas Trading Pemula Anda Diterima — {order_code}',

    // Template ini untuk delivery_type = manual
    // Admin akan melengkapi dengan link Zoom, jadwal batch, dan link grup di Filament
    'email_template' => <<<HTML
<!DOCTYPE html>
<html lang="id">
<head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"></head>
<body style="margin:0;padding:0;background:#0A0B0E;font-family:'Segoe UI',Arial,sans-serif;">
  <table width="100%" cellpadding="0" cellspacing="0" style="background:#0A0B0E;padding:40px 20px;">
    <tr><td align="center">
      <table width="560" cellpadding="0" cellspacing="0" style="background:#111318;border-radius:12px;border:1px solid rgba(255,255,255,0.08);overflow:hidden;">
        <tr><td style="background:#3B82F6;padding:24px 32px;">
          <p style="margin:0;color:#fff;font-size:20px;font-weight:600;">TradePortal</p>
          <p style="margin:4px 0 0;color:rgba(255,255,255,0.8);font-size:13px;">Platform Trading Tools Terpercaya</p>
        </td></tr>
        <tr><td style="padding:32px;">
          <p style="margin:0 0 8px;color:#8B8FA8;font-size:13px;">Halo,</p>
          <p style="margin:0 0 24px;color:#F1F2F5;font-size:22px;font-weight:600;">{user_name} 👋</p>
          <p style="margin:0 0 16px;color:#F1F2F5;font-size:15px;line-height:1.6;">
            Selamat! Pendaftaran Anda di <strong>Kelas Trading Pemula: Dari Nol ke Profit Konsisten</strong> telah dikonfirmasi.
          </p>
          <table width="100%" cellpadding="0" cellspacing="0" style="background:#1A1D26;border-radius:8px;border:1px solid rgba(59,130,246,0.3);margin:0 0 24px;">
            <tr><td style="padding:20px 24px;">
              <p style="margin:0 0 12px;color:#3B82F6;font-size:12px;font-weight:600;text-transform:uppercase;letter-spacing:0.5px;">Detail Pendaftaran</p>
              <p style="margin:0 0 4px;color:#8B8FA8;font-size:13px;">Kode Order</p>
              <p style="margin:0 0 16px;color:#F1F2F5;font-size:15px;font-family:monospace;font-weight:600;">{order_code}</p>
              <p style="margin:0;color:#8B8FA8;font-size:13px;">
                Informasi batch, jadwal live session, link Zoom, dan link grup Telegram tersedia di dashboard Anda.
              </p>
            </td></tr>
          </table>
          <table cellpadding="0" cellspacing="0" style="margin:0 0 24px;">
            <tr><td style="background:#3B82F6;border-radius:8px;">
              <a href="https://tradeportal.id/dashboard" style="display:inline-block;padding:14px 28px;color:#fff;font-size:14px;font-weight:600;text-decoration:none;">
                Lihat Detail Kelas →
              </a>
            </td></tr>
          </table>
          <p style="margin:0;color:#4B4F63;font-size:13px;line-height:1.6;">
            Ada pertanyaan tentang jadwal atau materi? Hubungi admin via WhatsApp.
          </p>
        </td></tr>
        <tr><td style="padding:20px 32px;border-top:1px solid rgba(255,255,255,0.06);">
          <p style="margin:0;color:#4B4F63;font-size:12px;">© 2025 TradePortal · {order_code}</p>
        </td></tr>
      </table>
    </td></tr>
  </table>
</body>
</html>
HTML,

    // delivery_content ini adalah TEMPLATE DEFAULT
    // Admin wajib override via custom_delivery_content per peserta (isi batch, link zoom, link grup)
    'delivery_content' => <<<HTML
<h2>Informasi Kelas Anda</h2>
<p>Admin sedang menyiapkan detail batch untuk Anda. Halaman ini akan diperbarui segera.</p>
<p>Jika dalam 24 jam belum ada update, silakan hubungi admin via WhatsApp.</p>
HTML,
]
```

---

### Produk 4 — Draft (Tidak Aktif)
```php
[
    'name'             => 'Paket Screener + Kelas Bundle',
    'slug'             => 'paket-screener-kelas-bundle',
    'type'             => 'other',
    'delivery_type'    => 'auto',
    'price'            => 699000,
    'sort_order'       => 99,
    'is_active'        => false,        // DRAFT — tidak muncul di publik
    'sold_count'       => 0,
    'short_description' => 'Bundling Screener Saham Pro + Kelas Trading Pemula dengan harga spesial.',
    'description'      => '<p>Segera hadir.</p>',
    'email_subject'    => null,
    'email_template'   => null,
    'delivery_content' => null,
]
```

---

## 3. Affiliate (`AffiliateSeeder`)

```php
[
    ['name' => 'Budi Santoso',   'code' => 'BUDI',    'is_active' => true],
    ['name' => 'Rizky Pratama',  'code' => 'RIZKY',   'is_active' => true],
    ['name' => 'Channels Media', 'code' => 'CHANNELS','is_active' => true],
]
```

---

## 4. Kupon (`CouponSeeder`)

```php
// Kupon 1 — Diskon persen, aktif, milik affiliate BUDI
[
    'code'         => 'BUDI10',
    'affiliate_id' => 1,   // Budi Santoso
    'type'         => 'percent',
    'value'        => 10,
    'min_purchase' => null,
    'max_uses'     => 100,
    'used_count'   => 0,
    'expired_at'   => null,   // tidak expired
    'is_active'    => true,
]

// Kupon 2 — Diskon flat, aktif, milik affiliate RIZKY
[
    'code'         => 'RIZKY50',
    'affiliate_id' => 2,   // Rizky Pratama
    'type'         => 'flat',
    'value'        => 50000,
    'min_purchase' => 200000,
    'max_uses'     => 50,
    'used_count'   => 0,
    'expired_at'   => null,
    'is_active'    => true,
]

// Kupon 3 — Promo umum, aktif, tanpa affiliate
[
    'code'         => 'TRADEPORTAL15',
    'affiliate_id' => null,
    'type'         => 'percent',
    'value'        => 15,
    'min_purchase' => null,
    'max_uses'     => 200,
    'used_count'   => 0,
    'expired_at'   => Carbon::now()->addMonths(3),
    'is_active'    => true,
]

// Kupon 4 — Untuk testing: sudah expired
[
    'code'         => 'EXPIRED99',
    'affiliate_id' => null,
    'type'         => 'percent',
    'value'        => 99,
    'min_purchase' => null,
    'max_uses'     => null,
    'used_count'   => 0,
    'expired_at'   => Carbon::now()->subDay(),
    'is_active'    => true,
]

// Kupon 5 — Untuk testing: sudah habis kuota
[
    'code'         => 'HABISQUOTA',
    'affiliate_id' => null,
    'type'         => 'flat',
    'value'        => 100000,
    'min_purchase' => null,
    'max_uses'     => 5,
    'used_count'   => 5,   // sudah penuh
    'expired_at'   => null,
    'is_active'    => true,
]

// Kupon 6 — Untuk testing: is_active = false
[
    'code'         => 'NONAKTIF',
    'affiliate_id' => null,
    'type'         => 'percent',
    'value'        => 25,
    'min_purchase' => null,
    'max_uses'     => null,
    'used_count'   => 0,
    'expired_at'   => null,
    'is_active'    => false,
]
```

---

## 5. User Sample (`UserSeeder`)

5 user biasa untuk testing. Semua pure OAuth (password null, is_admin false).

```php
[
    ['name' => 'Ahmad Fauzi',    'email' => 'ahmad@example.com'],
    ['name' => 'Sari Dewi',      'email' => 'sari@example.com'],
    ['name' => 'Doni Prasetyo',  'email' => 'doni@example.com'],
    ['name' => 'Linda Kurniati', 'email' => 'linda@example.com'],
    ['name' => 'Reza Mahendra',  'email' => 'reza@example.com'],
]
// Semua: provider='google', provider_id=fake()->numerify('####################'),
//        password=null, is_admin=false, email_verified_at=now()
```

---

## 6. Order Sample (`OrderSeeder`)

5 order untuk mengisi widget dashboard admin langsung setelah seed.

```php
// Order 1 — Pending (belum bayar), user Ahmad, produk Screener
// → Untuk test tampilan countdown & halaman payment
[status: pending, user: ahmad, product: screener-saham-pro, no coupon]

// Order 2 — Paid (sudah upload bukti, menunggu verifikasi), user Sari, produk Indikator, pakai kupon BUDI10
// → Untuk test widget "Order Pending Verifikasi" di admin dashboard
[status: paid, user: sari, product: indikator-supply-demand, coupon: BUDI10]

// Order 3 — Completed dengan delivery sent, user Doni, produk Screener
// → Untuk test tampilan /dashboard/products dan riwayat transaksi
[status: completed, delivery: sent, user: doni, product: screener-saham-pro]

// Order 4 — Cancelled, user Linda, produk Kelas
// → Untuk test badge "Dibatalkan" di riwayat transaksi
[status: cancelled, user: linda, product: kelas-trading-pemula]

// Order 5 — Completed dengan delivery FAILED, user Reza, produk Indikator
// → Untuk test widget "Delivery Gagal" di admin & banner resend di dashboard user
[status: completed, delivery: failed, retry_count: 3, user: reza, product: indikator-supply-demand]
```

---

## 7. Email Notifikasi Status Order

Email-email ini dikirim via tabel `notifications` (bukan `product_deliveries`).
Gunakan sebagai template di `OrderExpiryService` dan `OAuthController`/observer terkait.

### 7.1 Email Konfirmasi Order (dikirim saat order dibuat, status: pending)

**Subject:** `Konfirmasi Order {order_code} — Selesaikan Pembayaran Sebelum {expired_at}`

**Body (HTML):**
```html
<!DOCTYPE html>
<html lang="id">
<head><meta charset="UTF-8"></head>
<body style="margin:0;padding:0;background:#0A0B0E;font-family:'Segoe UI',Arial,sans-serif;">
  <table width="100%" cellpadding="0" cellspacing="0" style="background:#0A0B0E;padding:40px 20px;">
    <tr><td align="center">
      <table width="560" cellpadding="0" cellspacing="0" style="background:#111318;border-radius:12px;border:1px solid rgba(255,255,255,0.08);overflow:hidden;">
        <tr><td style="background:#3B82F6;padding:24px 32px;">
          <p style="margin:0;color:#fff;font-size:20px;font-weight:600;">TradePortal</p>
        </td></tr>
        <tr><td style="padding:32px;">
          <p style="margin:0 0 16px;color:#F1F2F5;font-size:16px;">Halo <strong>{{user_name}}</strong>,</p>
          <p style="margin:0 0 24px;color:#8B8FA8;font-size:14px;line-height:1.6;">
            Order Anda telah dibuat. Selesaikan pembayaran sebelum waktu habis.
          </p>

          <!-- Nominal Box — menonjolkan unique_amount -->
          <table width="100%" cellpadding="0" cellspacing="0" style="background:#1A1D26;border-radius:8px;border:1px solid rgba(245,158,11,0.4);margin:0 0 24px;">
            <tr><td style="padding:20px 24px;text-align:center;">
              <p style="margin:0 0 6px;color:#F59E0B;font-size:12px;font-weight:600;text-transform:uppercase;letter-spacing:0.5px;">Transfer Tepat Sebesar</p>
              <p style="margin:0 0 6px;color:#F1F2F5;font-size:32px;font-weight:700;font-family:monospace;">Rp {{unique_amount}}</p>
              <p style="margin:0;color:#8B8FA8;font-size:12px;">Nominal berbeda tiap transaksi untuk identifikasi otomatis</p>
            </td></tr>
          </table>

          <table width="100%" cellpadding="0" cellspacing="0" style="margin:0 0 24px;">
            <tr>
              <td style="padding:8px 0;color:#8B8FA8;font-size:13px;">Produk</td>
              <td style="padding:8px 0;color:#F1F2F5;font-size:13px;text-align:right;font-weight:500;">{{product_name}}</td>
            </tr>
            <tr style="border-top:1px solid rgba(255,255,255,0.06);">
              <td style="padding:8px 0;color:#8B8FA8;font-size:13px;">Kode Order</td>
              <td style="padding:8px 0;color:#F1F2F5;font-size:13px;text-align:right;font-family:monospace;font-weight:600;">{{order_code}}</td>
            </tr>
            <tr style="border-top:1px solid rgba(255,255,255,0.06);">
              <td style="padding:8px 0;color:#8B8FA8;font-size:13px;">Batas Waktu</td>
              <td style="padding:8px 0;color:#EF4444;font-size:13px;text-align:right;font-weight:500;">{{expired_at}}</td>
            </tr>
          </table>

          <table cellpadding="0" cellspacing="0" style="width:100%;margin:0 0 16px;">
            <tr><td style="background:#3B82F6;border-radius:8px;text-align:center;">
              <a href="{{payment_url}}" style="display:block;padding:14px;color:#fff;font-size:14px;font-weight:600;text-decoration:none;">
                Upload Bukti Transfer →
              </a>
            </td></tr>
          </table>

          <p style="margin:0;color:#4B4F63;font-size:12px;line-height:1.6;">
            Gunakan QR QRIS yang tersedia di halaman pembayaran. Pastikan transfer tepat nominal di atas.
          </p>
        </td></tr>
        <tr><td style="padding:20px 32px;border-top:1px solid rgba(255,255,255,0.06);">
          <p style="margin:0;color:#4B4F63;font-size:12px;">© 2025 TradePortal · {{order_code}}</p>
        </td></tr>
      </table>
    </td></tr>
  </table>
</body>
</html>
```

**Placeholder yang diisi oleh sistem:**
- `{{user_name}}` → `$order->user->name`
- `{{product_name}}` → `$order->items->first()->product->name`
- `{{order_code}}` → `$order->code`
- `{{unique_amount}}` → `number_format($order->unique_amount, 0, ',', '.')`
- `{{expired_at}}` → `$order->expired_at->timezone('Asia/Jakarta')->format('d M Y, H:i') . ' WIB'`
- `{{payment_url}}` → `config('app.url') . '/payment/' . $order->code`

---

### 7.2 Email Order Kedaluwarsa (dikirim saat order auto-expired)

**Subject:** `Order {order_code} Kedaluwarsa — Buat Order Baru untuk Melanjutkan`

**Body (HTML):**
```html
<table width="560" ...>
  ...header sama...
  <tr><td style="padding:32px;">
    <p style="color:#F1F2F5;font-size:16px;">Halo <strong>{{user_name}}</strong>,</p>
    <p style="color:#8B8FA8;font-size:14px;line-height:1.6;">
      Order <strong style="color:#F1F2F5;font-family:monospace;">{{order_code}}</strong>
      untuk <strong style="color:#F1F2F5;">{{product_name}}</strong>
      telah kedaluwarsa karena tidak ada pembayaran dalam 24 jam.
    </p>
    <table width="100%" style="background:#1A1D26;border-radius:8px;border:1px solid rgba(239,68,68,0.3);margin:0 0 24px;">
      <tr><td style="padding:16px 20px;">
        <p style="margin:0;color:#EF4444;font-size:13px;">
          ⚠️ Order dibatalkan otomatis pada {{cancelled_at}} WIB
        </p>
      </td></tr>
    </table>
    <p style="color:#8B8FA8;font-size:14px;line-height:1.6;">
      Anda bisa membuat order baru kapan saja. Harga produk berlaku saat order dibuat.
    </p>
    <table cellpadding="0" cellspacing="0" style="width:100%;margin:0 0 16px;">
      <tr><td style="background:#3B82F6;border-radius:8px;text-align:center;">
        <a href="{{product_url}}" style="display:block;padding:14px;color:#fff;font-size:14px;font-weight:600;text-decoration:none;">
          Beli Lagi →
        </a>
      </td></tr>
    </table>
  </td></tr>
</table>
```

---

### 7.3 Email Order Dibatalkan Admin

**Subject:** `Order {order_code} Dibatalkan`

Isi: informasi order dibatalkan, nama produk, kode order, ajakan hubungi admin via WhatsApp jika ada pertanyaan.

---

## 8. Catatan untuk Agent

- Semua `price` dan `value` adalah angka **tanpa titik/koma** — MySQL menyimpan sebagai `decimal(12,2)`.
- Semua `HTML` di `description`, `email_template`, `delivery_content` disimpan sebagai string biasa di kolom `text`.
- `delivery_content` produk Kelas (`delivery_type = manual`) sengaja dibuat minimal — admin yang melengkapi per peserta via Filament.
- Kupon `BUDI10` dan `RIZKY50` sudah ter-link ke affiliate dengan `affiliate_id` yang sesuai urutan insert di `AffiliateSeeder`.
- Pastikan `AffiliateSeeder` dijalankan **sebelum** `CouponSeeder` karena ada FK constraint.
- Email template menggunakan `{{placeholder}}` dengan kurung kurawal ganda — ini untuk notifikasi status (bukan delivery produk). Delivery produk memakai `{placeholder}` tunggal sesuai spesifikasi di agent prompt.
