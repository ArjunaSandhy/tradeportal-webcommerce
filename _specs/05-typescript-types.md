# TradePortal — TypeScript API Types

> Letakkan file ini di `frontend/src/types/api.ts`.
> Semua response API harus menggunakan interface ini. Tidak boleh ada `any`.
> Agent wajib menggunakan file ini sebagai referensi saat menulis service layer dan komponen.

---

## File: `frontend/src/types/api.ts`

```typescript
// ============================================================
// BASE
// ============================================================

export interface PaginatedResponse<T> {
  data: T[]
  meta: {
    current_page: number
    last_page: number
    per_page: number
    total: number
  }
}

export interface ApiError {
  message: string
  errors?: Record<string, string[]>       // validation errors (422)
  active_order_url?: string               // khusus error order aktif
}

// ============================================================
// USER & AUTH
// ============================================================

export interface User {
  id: number
  name: string
  email: string
  avatar: string | null
  provider: 'google' | null
  is_admin: false                          // frontend tidak pernah lihat is_admin: true
  email_verified_at: string | null
  created_at: string
}

export interface ProfileResponse {
  user: User
}

// ============================================================
// PRODUCT
// ============================================================

export type ProductType = 'screener' | 'indicator' | 'kelas' | 'other'
export type DeliveryType = 'auto' | 'manual'

export interface Product {
  id: number
  name: string
  slug: string
  type: ProductType
  short_description: string | null
  description: string
  price: number
  thumbnail: string | null
  images: string[] | null
  delivery_type: DeliveryType
  is_active: boolean
  sold_count: number
  sort_order: number
  created_at: string
  updated_at: string
}

export interface ProductDetail extends Product {
  // tambahan untuk halaman detail saat user login
  ownership_status: 'none' | 'active_order' | 'owned' | null
  active_order_code: string | null        // jika ownership_status = 'active_order'
}

export interface ProductListResponse extends PaginatedResponse<Product> {}

// ============================================================
// ORDER
// ============================================================

export type OrderStatus = 'pending' | 'paid' | 'verified' | 'completed' | 'cancelled'
export type DeliveryStatus = 'pending' | 'sent' | 'failed'

export interface OrderItem {
  id: number
  product_id: number
  product_name: string
  product_slug: string
  product_thumbnail: string | null
  price: number
}

export interface Order {
  id: number
  code: string
  user_id: number
  coupon_code: string | null
  subtotal: number
  discount_amount: number
  final_price: number
  unique_amount: number
  status: OrderStatus
  notes: string | null
  expired_at: string | null
  completed_at: string | null
  created_at: string
  updated_at: string
  items: OrderItem[]
  delivery_status: DeliveryStatus | null  // null jika belum ada delivery record
}

export interface CreateOrderRequest {
  product_id: number
  coupon_code?: string
}

export interface CreateOrderResponse {
  order_code: string
  unique_amount: number
  final_price: number
  expired_at: string
  payment_url: string
}

export interface OrderStatusResponse {
  status: OrderStatus
  completed_at: string | null
}

export interface OrderListResponse extends PaginatedResponse<Order> {}

// ============================================================
// PAYMENT
// ============================================================

export interface Payment {
  id: number
  order_id: number
  proof_image: string
  submitted_at: string
  verified_at: string | null
}

// Upload via FormData — tidak ada interface khusus, tapi selalu validasi di sisi client:
// - File: image/jpeg | image/png | image/webp
// - Max size: 5MB (5 * 1024 * 1024 bytes)

// ============================================================
// COUPON
// ============================================================

export interface ValidateCouponRequest {
  coupon_code: string
  product_id: number
}

export interface ValidateCouponResponse {
  valid: boolean
  discount_amount: number
  final_price: number
  coupon_code: string
  coupon_type: 'percent' | 'flat'
  coupon_value: number
}

// ============================================================
// DASHBOARD
// ============================================================

export interface ActiveOrder {
  id: number
  code: string
  product_name: string
  product_thumbnail: string | null
  unique_amount: number
  status: 'pending' | 'paid'
  expired_at: string | null
}

export interface OwnedProduct {
  order_id: number
  order_code: string
  product_id: number
  product_name: string
  product_slug: string
  product_thumbnail: string | null
  product_type: ProductType
  completed_at: string
  delivery_status: DeliveryStatus
}

export interface DashboardResponse {
  active_orders: ActiveOrder[]
  owned_products: OwnedProduct[]
}

export interface ProductAccessResponse {
  order_id: number
  product_id: number
  product_name: string
  product_type: ProductType
  product_thumbnail: string | null
  order_code: string
  completed_at: string
  delivery_content: string              // sudah resolved: custom_content ?? product.delivery_content
  delivery_status: DeliveryStatus
  last_resend_at: string | null         // untuk rate limit UI (1x/jam)
}

// ============================================================
// CONFIG
// ============================================================

export interface AppConfig {
  admin_whatsapp: string
  qris_image_url: string
}

// ============================================================
// RESEND / NOTIFICATION
// ============================================================

export interface ResendResponse {
  message: string
  next_available_at: string | null       // ISO datetime, untuk countdown rate limit di UI
}

// ============================================================
// AXIOS SERVICE LAYER PATTERN
// ============================================================
// Semua fungsi di frontend/src/services/ harus mengikuti pola ini:
//
// import axios from 'axios'
// import type { CreateOrderResponse, ApiError } from '@/types/api'
//
// export async function createOrder(data: CreateOrderRequest): Promise<CreateOrderResponse> {
//   const response = await axios.post<CreateOrderResponse>('/api/orders', data)
//   return response.data
// }
//
// Error handling di komponen:
// try {
//   const result = await createOrder({ product_id: 1 })
// } catch (err) {
//   if (axios.isAxiosError(err)) {
//     const apiErr = err.response?.data as ApiError
//     // tampilkan apiErr.message ke user
//   }
// }
```

---

## Catatan untuk Agent

- Tidak boleh ada `any` atau `unknown` kecuali di error boundary paling luar.
- Semua `id` adalah `number` (bigint MySQL di-cast ke number oleh Laravel JSON response).
- Semua `timestamp` dari API adalah string ISO 8601 dengan timezone `+07:00` (WIB).
- Untuk tampilkan waktu: gunakan `new Date(timestamp).toLocaleString('id-ID', { timeZone: 'Asia/Jakarta' })`.
- Nominal uang: gunakan `new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', minimumFractionDigits: 0 }).format(amount)`.
- `delivery_content` bisa berisi HTML — render dengan `dangerouslySetInnerHTML` di dalam container `prose dark:prose-invert`.
