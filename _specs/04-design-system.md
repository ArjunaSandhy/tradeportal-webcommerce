# TradePortal — Design System Specification

> Tambahkan dokumen ini ke agent prompt saat mengerjakan **Step 14** (React frontend).
> Stack: React 19 + TypeScript + Tailwind CSS + shadcn/ui

---

## 1. Prinsip Desain

- **Dark mode default** — background gelap, konten terang. Tidak ada light mode toggle.
- **Tech-forward, trader aesthetic** — bukan e-commerce biasa. Rasa seperti TradingView atau platform SaaS trading.
- **Rich & informative** — info produk, harga, status, dan aksi tampil lengkap dalam satu layar tanpa harus scroll jauh.
- **Zero friction checkout** — langkah sesedikit mungkin. Setiap halaman punya satu primary action yang jelas.
- **Tidak ada dekorasi berlebihan** — tidak ada gradient warna-warni, animasi berlebihan, atau ilustrasi. Clean, tajam, purposeful.

---

## 2. Warna (Color Tokens)

Implementasikan sebagai CSS custom properties di `frontend/src/index.css` dan daftarkan di `tailwind.config.ts`.

```css
:root {
  /* Background layers */
  --bg-base:      #0A0B0E;   /* halaman utama — paling gelap */
  --bg-surface:   #111318;   /* card, panel */
  --bg-elevated:  #1A1D26;   /* modal, dropdown, popover */
  --bg-overlay:   #222636;   /* hover state, selected row */

  /* Brand accent */
  --accent:       #3B82F6;   /* biru elektrik — CTA, link aktif, badge */
  --accent-hover: #2563EB;
  --accent-muted: rgba(59,130,246,0.15); /* background badge, highlight */

  /* Text */
  --text-primary:   #F1F2F5;
  --text-secondary: #8B8FA8;
  --text-muted:     #4B4F63;
  --text-inverse:   #0A0B0E;  /* teks di atas tombol berwarna */

  /* Semantic */
  --success:       #10B981;
  --success-muted: rgba(16,185,129,0.12);
  --warning:       #F59E0B;
  --warning-muted: rgba(245,158,11,0.12);
  --danger:        #EF4444;
  --danger-muted:  rgba(239,68,68,0.12);

  /* Border */
  --border:        rgba(255,255,255,0.08);
  --border-strong: rgba(255,255,255,0.16);
}
```

**Tailwind config mapping (`tailwind.config.ts`):**
```typescript
colors: {
  bg: {
    base:     'var(--bg-base)',
    surface:  'var(--bg-surface)',
    elevated: 'var(--bg-elevated)',
    overlay:  'var(--bg-overlay)',
  },
  accent: {
    DEFAULT: 'var(--accent)',
    hover:   'var(--accent-hover)',
    muted:   'var(--accent-muted)',
  },
  text: {
    primary:   'var(--text-primary)',
    secondary: 'var(--text-secondary)',
    muted:     'var(--text-muted)',
  },
  border: {
    DEFAULT: 'var(--border)',
    strong:  'var(--border-strong)',
  },
  success: 'var(--success)',
  warning: 'var(--warning)',
  danger:  'var(--danger)',
}
```

---

## 3. Tipografi

Font: **Inter** (Google Fonts). Import di `index.html`:
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap" rel="stylesheet">
```

| Token       | Size    | Weight | Line-height | Penggunaan                        |
|-------------|---------|--------|-------------|-----------------------------------|
| `text-hero` | 36px    | 600    | 1.2         | Headline beranda                  |
| `text-h1`   | 24px    | 600    | 1.3         | Nama produk di halaman detail     |
| `text-h2`   | 18px    | 600    | 1.4         | Section title, card header        |
| `text-h3`   | 15px    | 500    | 1.4         | Sub-section, label grup           |
| `body`      | 14px    | 400    | 1.6         | Body text default                 |
| `text-sm`   | 13px    | 400    | 1.5         | Metadata, timestamp, helper text  |
| `text-xs`   | 12px    | 500    | 1.4         | Badge, label, tag                 |
| `mono`      | 13px    | 500    | 1.4         | Kode order, nominal transfer (font-mono) |

Untuk nominal unik dan kode order, gunakan `font-mono` agar mudah dibaca dan tidak ambigu.

---

## 4. shadcn/ui — Setup & Konfigurasi

```bash
cd frontend
npx shadcn@latest init
# Pilih: TypeScript=yes, Tailwind=yes, Style=default, BaseColor=slate, CSSVariables=yes
```

Komponen yang perlu di-install (`npx shadcn@latest add ...`):
```
button badge card dialog input label
select separator skeleton tabs toast
alert progress avatar dropdown-menu
```

**Override shadcn CSS variables** di `globals.css` agar sesuai dark theme TradePortal:
```css
.dark {
  --background: 10 11 14;          /* #0A0B0E */
  --foreground: 241 242 245;       /* #F1F2F5 */
  --card: 17 19 24;                /* #111318 */
  --card-foreground: 241 242 245;
  --border: 255 255 255 / 0.08;
  --input: 26 29 38;               /* #1A1D26 */
  --primary: 59 130 246;           /* #3B82F6 */
  --primary-foreground: 10 11 14;
  --muted: 26 29 38;
  --muted-foreground: 139 143 168; /* #8B8FA8 */
  --accent: 34 38 54;              /* #222636 */
  --accent-foreground: 241 242 245;
  --destructive: 239 68 68;
  --ring: 59 130 246;
}
```

Tambahkan `className="dark"` di `<html>` tag pada `index.html`.

---

## 5. Komponen & Pattern Per Halaman

### 5.1 Layout Global

```
┌─────────────────────────────────────────────────────┐
│  Navbar (sticky, bg-surface, border-bottom)         │
│  Logo kiri  ·  Nav links  ·  Login/Avatar kanan     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  <main> (max-w-6xl mx-auto px-4 py-8)              │
│                                                     │
├─────────────────────────────────────────────────────┤
│  Footer minimal (bg-surface, text-muted, links)     │
└─────────────────────────────────────────────────────┘
```

### 5.2 Beranda (`/`)

**Hero section:**
- Background: `bg-base` dengan subtle grid pattern (CSS `background-image: linear-gradient`)
- Headline 36px, subline 16px text-secondary
- CTA button: biru elektrik, ukuran besar, full-radius

**Product grid:**
- `grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4`
- Setiap card: `bg-surface border border-border rounded-xl p-5 hover:border-border-strong hover:bg-elevated transition-all`
- Card berisi: thumbnail (aspect-ratio 16:9, rounded-lg), badge tipe produk, nama, harga, tombol beli

**Badge tipe produk:**
```
screener  → bg-accent-muted text-accent       border border-accent/20
indicator → bg-success-muted text-success     border border-success/20
kelas     → bg-warning-muted text-warning     border border-warning/20
other     → bg-bg-overlay    text-text-secondary
```

### 5.3 Halaman Detail Produk (`/products/{slug}`)

Layout 2 kolom di desktop, stack di mobile:
- **Kiri (60%):** gallery gambar, deskripsi lengkap dengan `prose dark:prose-invert`
- **Kanan (40%) sticky:** card harga + tombol beli — sticky saat scroll

Card harga kanan:
```
┌─────────────────────────┐
│ Rp 299.000              │  ← text-h1, font-mono
│ [Badge tipe]            │
│ ─────────────────────── │
│ ✓ Akses seumur hidup    │  ← checklist value props
│ ✓ Dashboard pribadi     │
│ ✓ Update gratis         │
│ ─────────────────────── │
│ [Beli Sekarang]  CTA    │
│ [Hubungi Admin]  ghost  │
└─────────────────────────┘
```

### 5.4 Checkout (`/checkout/{slug}`)

Single-column, max-w-lg, centered:
- Ringkasan produk: thumbnail kecil + nama + harga di atas
- Info user (nama + email) dalam card read-only dengan icon lock
- Input kode kupon: `flex gap-2`, input + tombol "Pakai"
  - Saat valid: border hijau + badge diskon muncul
  - Saat invalid: border merah + pesan error inline
- Total: animasi harga berubah saat kupon dipakai
- Tombol CTA full-width di bawah

### 5.5 Halaman Pembayaran (`/payment/{code}`)

Layout 2 kolom:
- **Kiri:** QR QRIS (card putih kecil agar QR terlihat), instruksi transfer
- **Kanan:** Detail order card

**Nominal unik — tampilkan menonjol:**
```
Transfer tepat sebesar
┌─────────────────────────┐
│  Rp 299.847             │  ← text-hero, font-mono, text-accent
└─────────────────────────┘
Nominal berbeda tiap transaksi untuk identifikasi otomatis
```

**Countdown timer:**
- Format: `23:45:12` — font-mono, text-warning saat < 1 jam, text-danger saat < 10 menit
- Bar progress di bawahnya (shadcn `<Progress>`)

**Upload bukti:**
- Dropzone: border dashed, border-border, hover border-accent, rounded-xl
- Preview thumbnail setelah file dipilih
- Max size indicator: "Maks 5MB · JPG, PNG, WEBP"

**Banner polling setelah upload:**
- `bg-accent-muted border border-accent/30 rounded-lg p-4`
- Icon loading spinner + "Menunggu verifikasi admin..."

### 5.6 Dashboard (`/dashboard`)

**Banner order aktif** (jika ada):
- `bg-warning-muted border border-warning/30 rounded-xl p-4 mb-6`
- Nama produk + nominal + countdown + tombol "Lanjut Bayar"

**Grid produk yang dimiliki:**
- `grid grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4` — lebih padat dari halaman publik
- Card lebih kecil: thumbnail + nama + badge "Akses Aktif" (hijau)
- Empty state: ilustrasi minimal + CTA ke beranda

### 5.7 Detail Produk Dibeli (`/dashboard/products/{order_id}`)

- Konten akses di dalam card `bg-surface` dengan `prose dark:prose-invert max-w-none`
- Jika delivery failed: banner danger dengan tombol "Kirim Ulang Email" + countdown rate limit
- Tombol "Hubungi Admin" (WhatsApp) selalu ada di bawah sebagai fallback

### 5.8 Riwayat Transaksi (`/orders`)

Tabel responsif (shadcn `<Table>`):

| Kode Order | Produk | Nominal | Status | Delivery | Tanggal |
|---|---|---|---|---|---|

**Status badges:**
```
pending   → bg-bg-overlay    text-text-secondary  "Menunggu Bayar"
paid      → bg-warning-muted text-warning         "Diverifikasi"
verified  → bg-accent-muted  text-accent          "Diproses"
completed → bg-success-muted text-success         "Selesai"
cancelled → bg-danger-muted  text-danger          "Dibatalkan"
```

Di mobile, collapse ke card list (bukan tabel).

---

## 6. Interaksi & Feedback

| Situasi | Feedback |
|---|---|
| Tombol submit (loading) | Spinner + disabled, teks berubah: "Memproses..." |
| API error | Toast merah (`useToast`) muncul kanan bawah, auto-dismiss 5 detik |
| Sukses action | Toast hijau |
| Form validation | Error inline di bawah field, border merah |
| Kupon valid | Border hijau + badge diskon slide-in |
| Polling aktif | Pulse animation pada status indicator |

---

## 7. Instruksi untuk Agent

Saat menulis kode React frontend:

1. **Selalu gunakan dark theme** — `className="dark"` di root, tidak ada conditional theme.
2. **Import shadcn** dari `@/components/ui/...`, jangan buat komponen primitif dari scratch.
3. **Gunakan Tailwind class** yang sudah di-mapping ke design tokens di atas.
4. **Nominal unik dan kode order** selalu pakai `font-mono` dan tampilkan dengan `toLocaleString('id-ID')`.
5. **Semua loading state** harus ditangani — skeleton loader untuk data, spinner untuk action.
6. **Mobile-first** — gunakan Tailwind responsive prefix (`sm:`, `lg:`) untuk layout yang berbeda.
7. **Tidak ada hardcoded hex** di JSX — gunakan class Tailwind atau CSS variable.
8. **Konsistensi spacing:** gunakan kelipatan 4 (`p-4`, `gap-4`, `p-6`, `gap-6`).
