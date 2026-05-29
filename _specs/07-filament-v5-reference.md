# TradePortal — Filament v5 Quick Reference

> **Tujuan dokumen ini:** Mencegah agent menggunakan syntax Filament v3 yang sudah tidak berlaku.
> Filament v5 memiliki breaking change besar dari v3 — terutama pada struktur Resource, Schema, dan Table.
> Gunakan dokumen ini sebagai acuan pertama sebelum menulis kode apapun yang menyentuh `/admin`.

---

## ⚠️ PERBEDAAN KRITIS v3 → v5 (Baca ini dulu)

| Aspek | Filament v3 (JANGAN PAKAI) | Filament v5 (WAJIB DIPAKAI) |
|---|---|---|
| Form schema | `public static function form(Form $form)` | Terpisah di `Schemas/XxxForm.php` |
| Table definition | `public static function table(Table $table)` | Terpisah di `Tables/XxxTable.php` |
| Infolist | inline di Resource | Terpisah di `Schemas/XxxInfolist.php` |
| `getFormSchema()` | dipakai di v3 | **tidak ada di v5** |
| `getTableColumns()` | dipakai di v3 | **tidak ada di v5** |
| Widget stats | `StatsOverviewWidget` | `StatsOverviewWidget` (sama, tapi cara register berbeda) |

---

## Struktur File Resource v5

Setelah `php artisan make:filament-resource Order`, struktur yang dihasilkan:

```
app/Filament/Resources/
└── Orders/
    ├── OrderResource.php          ← class utama (routing, nav, auth)
    ├── Pages/
    │   ├── CreateOrder.php
    │   ├── EditOrder.php
    │   └── ListOrders.php
    ├── Schemas/
    │   ├── OrderForm.php          ← definisi form fields
    │   └── OrderInfolist.php      ← definisi infolist entries (view page)
    └── Tables/
        └── OrdersTable.php        ← definisi columns, filters, actions
```

---

## Dokumentasi Live (Selalu Cek Ini)

Filament v5 menyediakan index dokumentasi khusus untuk AI agent:

```
https://filamentphp.com/docs/llms.txt
```

**Cara pakai:** Setiap kali mengerjakan Step 5 atau Step 13, fetch URL di atas untuk mendapatkan daftar semua halaman dokumentasi, lalu fetch halaman spesifik yang relevan dengan fitur yang sedang diimplementasikan.

**Halaman paling sering dibutuhkan untuk TradePortal:**

| Fitur | URL Dokumentasi |
|---|---|
| Resource overview | `https://filamentphp.com/docs/5.x/resources/overview.md` |
| Table actions | `https://filamentphp.com/docs/5.x/tables/actions.md` |
| Action modals | `https://filamentphp.com/docs/5.x/actions/modals.md` |
| Form fields | `https://filamentphp.com/docs/5.x/forms/overview.md` |
| Widgets | `https://filamentphp.com/docs/5.x/widgets/overview.md` |
| Stats widget | `https://filamentphp.com/docs/5.x/widgets/stats-overview.md` |
| Custom pages | `https://filamentphp.com/docs/5.x/resources/custom-pages.md` |
| Schema tabs | `https://filamentphp.com/docs/5.x/schemas/tabs.md` |
| Infolist | `https://filamentphp.com/docs/5.x/infolists/overview.md` |
| Authorization | `https://filamentphp.com/docs/5.x/resources/overview.md#authorization` |

---

## Scaffold Commands v5

```bash
# Resource biasa (dengan list, create, edit pages)
php artisan make:filament-resource Order

# Resource simple (semua via modal, satu halaman)
php artisan make:filament-resource Affiliate --simple

# Widget stats
php artisan make:filament-widget RevenueStatsWidget --stats-overview

# Widget tabel
php artisan make:filament-widget RecentOrdersWidget --table

# Widget custom (chart, dsb)
php artisan make:filament-widget QueueHealthWidget

# Buat admin user
php artisan make:filament-user
```

---

## Pattern: Resource Utama (OrderResource)

```php
// app/Filament/Resources/Orders/OrderResource.php
namespace App\Filament\Resources\Orders;

use Filament\Resources\Resource;
use App\Models\Order;

class OrderResource extends Resource
{
    protected static ?string $model = Order::class;
    protected static ?string $navigationIcon = 'heroicon-o-shopping-cart';
    protected static ?string $navigationLabel = 'Orders';
    protected static ?int $navigationSort = 1;

    public static function canAccess(): bool
    {
        return auth()->user()?->is_admin === true;
    }

    public static function getPages(): array
    {
        return [
            'index'  => Pages\ListOrders::route('/'),
            'create' => Pages\CreateOrder::route('/create'),
            'edit'   => Pages\EditOrder::route('/{record}/edit'),
        ];
    }
}
```

---

## Pattern: Table dengan Custom Actions (v5)

```php
// app/Filament/Resources/Orders/Tables/OrdersTable.php
namespace App\Filament\Resources\Orders\Tables;

use Filament\Tables\Table;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Columns\BadgeColumn;
use Filament\Tables\Actions\Action;
use Filament\Tables\Actions\ActionGroup;

class OrdersTable
{
    public static function make(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('code')
                    ->label('Kode Order')
                    ->searchable()
                    ->fontFamily('mono'),

                TextColumn::make('user.name')
                    ->label('User')
                    ->searchable(),

                TextColumn::make('unique_amount')
                    ->label('Nominal Transfer')
                    ->money('IDR')
                    ->fontFamily('mono'),

                TextColumn::make('status')
                    ->label('Status')
                    ->badge()
                    ->color(fn (string $state): string => match ($state) {
                        'pending'   => 'gray',
                        'paid'      => 'warning',
                        'verified'  => 'info',
                        'completed' => 'success',
                        'cancelled' => 'danger',
                    }),

                TextColumn::make('created_at')
                    ->label('Tanggal')
                    ->dateTime('d M Y, H:i')
                    ->timezone('Asia/Jakarta'),
            ])
            ->filters([
                // tambahkan filter status di sini
            ])
            ->actions([
                ActionGroup::make([
                    Action::make('verify')
                        ->label('Verify Payment')
                        ->icon('heroicon-o-check-circle')
                        ->color('success')
                        ->visible(fn ($record) => $record->status === 'paid')
                        ->requiresConfirmation()
                        ->action(fn ($record) => /* dispatch job */ null),

                    Action::make('view_proof')
                        ->label('Lihat Bukti')
                        ->icon('heroicon-o-photo')
                        ->visible(fn ($record) => $record->payment !== null)
                        ->modalContent(fn ($record) => view('filament.modals.proof-image', [
                            'url' => Storage::url($record->payment->proof_image),
                        ]))
                        ->modalSubmitAction(false),

                    Action::make('cancel')
                        ->label('Batalkan Order')
                        ->icon('heroicon-o-x-circle')
                        ->color('danger')
                        ->visible(fn ($record) => in_array($record->status, ['pending', 'paid']))
                        ->requiresConfirmation()
                        ->action(fn ($record) => /* cancel logic */ null),
                ]),
            ])
            ->defaultSort('created_at', 'desc');
    }
}
```

---

## Pattern: Form Schema (v5)

```php
// app/Filament/Resources/Products/Schemas/ProductForm.php
namespace App\Filament\Resources\Products\Schemas;

use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\RichEditor;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\Toggle;
use Filament\Forms\Components\FileUpload;
use Filament\Forms\Components\Tabs;
use Filament\Schemas\Schema;

class ProductForm
{
    public static function make(Schema $schema): Schema
    {
        return $schema->components([
            Tabs::make()->tabs([

                Tabs\Tab::make('Info Produk')->schema([
                    TextInput::make('name')
                        ->label('Nama Produk')
                        ->required()
                        ->live(onBlur: true)
                        ->afterStateUpdated(fn ($state, $set) =>
                            $set('slug', Str::slug($state))
                        ),

                    TextInput::make('slug')
                        ->label('Slug')
                        ->required()
                        ->unique(ignoreRecord: true),

                    Select::make('type')
                        ->label('Tipe')
                        ->options([
                            'screener'  => 'Screener',
                            'indicator' => 'Indicator',
                            'kelas'     => 'Kelas',
                            'other'     => 'Other',
                        ])
                        ->required(),

                    Select::make('delivery_type')
                        ->options(['auto' => 'Auto', 'manual' => 'Manual'])
                        ->required(),

                    TextInput::make('price')
                        ->label('Harga')
                        ->numeric()
                        ->prefix('Rp')
                        ->required(),

                    Toggle::make('is_active')->label('Aktif'),
                ]),

                Tabs\Tab::make('Deskripsi')->schema([
                    Textarea::make('short_description')->rows(3),
                    RichEditor::make('description')->required(),
                ]),

                Tabs\Tab::make('Email Delivery')->schema([
                    TextInput::make('email_subject')->label('Subject Email'),
                    RichEditor::make('email_template')->label('Template Email'),
                    RichEditor::make('delivery_content')->label('Konten Dashboard'),
                ]),

            ])->columnSpanFull(),
        ]);
    }
}
```

---

## Pattern: Action dengan Modal Form (v5)

Untuk tombol "Isi & Kirim Delivery" di OrderResource:

```php
Action::make('fill_delivery')
    ->label('Isi & Kirim Delivery')
    ->icon('heroicon-o-paper-airplane')
    ->color('primary')
    ->visible(fn ($record) =>
        $record->status === 'verified' &&
        $record->items->first()?->product?->delivery_type === 'manual'
    )
    ->form([
        RichEditor::make('custom_delivery_content')
            ->label('Konten Akses Produk')
            ->required()
            ->helperText('Isi konten yang akan ditampilkan di dashboard user dan dikirim via email.'),
    ])
    ->action(function ($record, array $data): void {
        // update product_deliveries.custom_delivery_content
        // dispatch EmailDeliveryJob
    })
    ->modalHeading('Isi & Kirim Konten Delivery')
    ->modalSubmitActionLabel('Kirim Sekarang'),
```

---

## Pattern: Stats Overview Widget (v5)

```php
// app/Filament/Widgets/RevenueStatsWidget.php
namespace App\Filament\Widgets;

use Filament\Widgets\StatsOverviewWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;

class RevenueStatsWidget extends StatsOverviewWidget
{
    protected function getStats(): array
    {
        return [
            Stat::make('Revenue Bulan Ini', 'Rp ' . number_format(
                Order::whereMonth('completed_at', now()->month)
                    ->where('status', 'completed')
                    ->sum('final_price'),
                0, ',', '.'
            ))->color('success'),

            Stat::make('Order Pending Verifikasi',
                Order::where('status', 'paid')->count()
            )->color('warning'),

            Stat::make('Delivery Gagal',
                ProductDelivery::where('status', 'failed')->count()
            )->color('danger'),
        ];
    }
}
```

---

## Pattern: canAccessPanel() di User Model

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

---

## Hal yang Berubah dari v3 — Checklist Anti-Salah

- [ ] Tidak ada `public static function form(Form $form)` di dalam class Resource — pindahkan ke `Schemas/XxxForm.php`
- [ ] Tidak ada `public static function table(Table $table)` di dalam class Resource — pindahkan ke `Tables/XxxTable.php`
- [ ] Import `Filament\Schemas\Schema` (bukan `Filament\Forms\Form`) untuk method schema di v5
- [ ] `canAccess()` menggantikan beberapa method gate lama — cek dokumentasi authorization
- [ ] `Action` di tabel diimport dari `Filament\Tables\Actions\Action`, bukan `Filament\Actions\Action`
- [ ] Widget harus di-register di Panel provider atau `getWidgets()` di halaman yang tepat

---

## Catatan Tambahan

- Jika ragu dengan syntax apapun, **fetch URL dokumentasi spesifik** dari daftar di atas sebelum menulis kode
- Halaman "AI-assisted development" di docs resmi: `https://filamentphp.com/docs/5.x/introduction/ai`
- Filament v5 juga mendukung `composer require laravel/boost --dev` + `php artisan boost:install` untuk integrasi AI lebih dalam — opsional tapi berguna untuk Claude Code / Cursor
