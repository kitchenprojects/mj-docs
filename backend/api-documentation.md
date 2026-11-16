# 📡 Spesifikasi Referensi API (Backend API Reference)

Dokumen ini menyajikan panduan referensi lengkap untuk seluruh endpoint REST API pada backend **Dapur Nekti (MJ Kitchen)**.

- **Base URL Pengembangan**: `http://localhost:8080/api`
- **Base URL Produksi**: `https://mj-kitchen-489615087978.asia-southeast2.run.app/api`
- **Format Pertukaran Data**: `application/json`
- **Autentikasi**: Header `Authorization: Bearer <JWT_ACCESS_TOKEN>`

---

## 📑 Ringkasan Kategori Endpoint

| Kategori | Prefix URL | Hak Akses | Keterangan |
|---|---|---|---|
| **Menu & Kategori** | `/api/menu` | Publik & Admin | Katalog hidangan, kategori masakan, dan foto galeri |
| **Add-ons** | `/api/addons` | Publik & Admin | Pelengkap makanan (extra sambal, nasi, lauk tambahan) |
| **Pesanan (Orders)** | `/api/orders` | Customer & Admin | Checkout pesanan, riwayat belanja, dan status order |
| **Pembayaran & Webhook**| `/api/orders` | Publik, Auth, Admin | Webhook notifikasi Midtrans & update status bayar |
| **Laporan & Dashboard** | `/api/orders/admin` | Admin | Statistik KPI penjualan dan ekspor laporan periodik |
| **Pengguna & Profil** | `/api/users` | Customer & Admin | Profil akun, hak akses role, dan buku alamat |
| **Pengiriman (Shipping)**| `/api/shipping` | Publik | Kalkulasi ongkos kirim real-time & tarif tier |

---

## 1. Modul Menu & Kategori (`/api/menu`)

### 1.1. Mengambil Seluruh Kategori
`GET /api/menu/categories`
- **Akses**: Publik
- **Respon Sukses (200 OK)**:
```json
[
  {
    "category_id": 1,
    "category_name": "Nasi Box",
    "description": "Paket nasi lengkap dengan lauk pauk higienis",
    "image_url": "https://supabase.co/storage/v1/object/public/menu-images/nasibox.jpg"
  }
]
```

### 1.2. Menambah Kategori Baru
`POST /api/menu/categories`
- **Akses**: Terproteksi (Admin)
- **Request Body**:
```json
{
  "category_name": "Tumpeng Mini",
  "description": "Sajian tumpeng porsi personal untuk perayaan",
  "image_url": "https://..."
}
```
- **Respon Sukses (201 Created)**: Mengembalikan objek kategori yang baru dibuat.

### 1.3. Mengubah Kategori
`PUT /api/menu/categories/:id`
- **Akses**: Terproteksi (Admin)
- **Path Parameter**: `id` (ID kategori)
- **Request Body**: Sama seperti POST Kategori.
- **Respon Sukses (200 OK)**: Objek kategori yang diperbarui.

### 1.4. Menghapus Kategori
`DELETE /api/menu/categories/:id`
- **Akses**: Terproteksi (Admin)
- **Respon Sukses (200 OK)**: `{"success": true}`

---

### 1.5. Mengambil Daftar Menu Makanan
`GET /api/menu/items`
- **Akses**: Publik
- **Query Parameter (Opsional)**: `category_id` (Integer - Filter berdasarkan ID kategori)
- **Respon Sukses (200 OK)**:
```json
[
  {
    "menu_id": 10,
    "menu_name": "Paket Nasi Timbel Komplit",
    "description": "Nasi timbel dibungkus daun pisang, ayam goreng lengkuas, tahu, tempe, lalap, sambal terasi",
    "price": "35000.00",
    "is_available": true,
    "category_id": 1,
    "category_name": "Nasi Box",
    "images": [
      {
        "gallery_id": 4,
        "menu_id": 10,
        "image_url": "https://...",
        "caption": "Tampak atas paket komplit"
      }
    ]
  }
]
```

### 1.6. Mengambil Detail Menu
`GET /api/menu/items/:id`
- **Akses**: Publik
- **Path Parameter**: `id` (ID menu)
- **Respon Sukses (200 OK)**: Objek menu tunggal beserta array gambar `images`.

### 1.7. Membuat Menu Baru
`POST /api/menu/items`
- **Akses**: Terproteksi (Admin)
- **Request Body**:
```json
{
  "category_id": 1,
  "menu_name": "Nasi Liwet Solo",
  "description": "Nasi liwet gurih dengan suwiran ayam opor dan sayur labu",
  "price": 30000,
  "is_available": true,
  "images": [
    "https://storage.supabase.co/.../image1.jpg"
  ]
}
```
- **Respon Sukses (201 Created)**: Objek menu baru.

### 1.8. Mengubah Menu
`PUT /api/menu/items/:id`
- **Akses**: Terproteksi (Admin)
- **Request Body**:
```json
{
  "category_id": 1,
  "menu_name": "Nasi Liwet Solo Spesial",
  "description": "Dengan tambahan telur pindang",
  "price": 35000,
  "is_available": true
}
```

### 1.9. Menghapus Menu
`DELETE /api/menu/items/:id`
- **Akses**: Terproteksi (Admin)
- **Perilaku**: Otomatis menghapus relasi galeri gambar terkait terlebih dahulu, kemudian menghapus menu.

---

### 1.10. Manajemen Galeri Menu
- `POST /api/menu/:menu_id/gallery` (Admin): Menambahkan foto ke menu (`{"image_url": "...", "caption": "..."}`).
- `PUT /api/menu/gallery/:gallery_id` (Admin): Mengubah URL foto atau caption.
- `DELETE /api/menu/gallery/:gallery_id` (Admin): Menghapus foto dari galeri.

---

## 2. Modul Add-ons (`/api/addons`)

### 2.1. Mengambil Seluruh Item Add-on
`GET /api/addons`
- **Akses**: Publik
- **Kriteria Database**: Mengambil menu yang `is_available = true` dan memiliki nama mengandung `%tambah%`, `%extra%`, `%add%`, atau harga `<= 10000`.
- **Respon Sukses (200 OK)**:
```json
[
  {
    "menu_id": 99,
    "menu_name": "Ekstra Sambal Dadak",
    "description": "Sambal terasi segar pedas gurih",
    "price": "5000.00",
    "category_id": 3,
    "image_url": "https://..."
  }
]
```

### 2.2. Membuat Item Add-on Baru
`POST /api/addons`
- **Akses**: Terproteksi (Admin)
- **Request Body**: `{"menu_name": "...", "description": "...", "price": 5000, "category_id": 1, "image_url": "..."}`

---

## 3. Modul Pesanan (Orders) & Pembayaran

### 3.1. Membuat Pesanan Baru (Checkout)
`POST /api/orders`
- **Akses**: Terproteksi (Customer/Auth)
- **Request Body**:
```json
{
  "user_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "address_id": 3,
  "items": [
    {
      "menu_id": 10,
      "menu_name": "Paket Nasi Timbel Komplit",
      "price": 35000,
      "quantity": 2
    }
  ],
  "shipping_cost": 15000,
  "payment_method_id": "midtrans_snap"
}
```
- **Respon Sukses (201 Created)**:
```json
{
  "order_id": "c1f76d49-411a-4228-b8ce-3ffcfae5ec84",
  "total_amount": 85000,
  "status": "Pending",
  "snap_token": "79b32c69-2323-455b-b91c-99a38ef672e1"
}
```

### 3.2. Mengambil Riwayat Pesanan Saya
`GET /api/orders/mine/:userId`
- **Akses**: Terproteksi (Auth)
- **Respon Sukses (200 OK)**: Array pesanan milik pengguna beserta array nested `items`, `snap_token`, dan `payment_status`.

### 3.3. Mengambil Detail Pesanan (Untuk Struk / Invoice)
`GET /api/orders/:orderId`
- **Akses**: Terproteksi (Auth)
- **Respon Sukses (200 OK)**: Detail pesanan lengkap mencakup data pemesan (`user_name`, `user_phone`), alamat pengiriman (`street`, `city`, `postal_code`), status pembayaran (`payment_status`, `payment_date`), dan daftar rincian item (`items`).

### 3.4. Mengambil Semua Pesanan (Admin)
`GET /api/orders`
- **Akses**: Terproteksi (Admin)
- **Respon Sukses (200 OK)**: Seluruh order dari semua pelanggan terurut dari yang terbaru.

### 3.5. Mengubah Status Operasional Pesanan
`PUT /api/orders/:id/status`
- **Akses**: Terproteksi (Admin)
- **Request Body**:
```json
{
  "status": "Out for Delivery"
}
```
- **Nilai Status Valid**: `Pending`, `Confirmed`, `Processing`, `Out for Delivery`, `Delivered`, `Cancelled`.

### 3.6. Memperbarui Status Pembayaran (Client Callback)
`PUT /api/orders/:id/payment`
- **Akses**: Terproteksi (Auth)
- **Request Body**: `{"payment_status": "Paid"}`
- **Logika**: Memvalidasi UUID order, memperbarui record di tabel `payment` dengan `payment_date = NOW()`, serta mengubah status order menjadi `Confirmed`.

### 3.7. Midtrans Webhook Notifikasi
`POST /api/orders/midtrans-notification`
- **Akses**: Publik (Dipanggil langsung oleh server Midtrans)
- **Request Body**: JSON payload notifikasi transaksi Midtrans.
- **Logika**:
  - `settlement` atau `capture (accept)` -> payment `Paid`, order `Confirmed`.
  - `cancel`, `deny`, `expire` -> payment `Failed`, order `Cancelled`.
  - `pending` -> payment `Pending`, order `Pending`.
- **Respon Sukses (200 OK)**: `{"message": "OK"}`

---

## 4. Modul Laporan & Statistik Admin

### 4.1. Dashboard KPI Stats
`GET /api/orders/admin/dashboard-stats`
- **Akses**: Terproteksi (Admin)
- **Respon Sukses (200 OK)**:
```json
{
  "totals": {
    "orders": 124,
    "revenue": 14500000.00,
    "customers": 45,
    "pendingOrders": 6
  },
  "today": {
    "orders": 12,
    "revenue": 1250000.00
  },
  "thisMonth": {
    "orders": 85,
    "revenue": 9800000.00
  },
  "statusBreakdown": {
    "Pending": 4,
    "Confirmed": 12,
    "Delivered": 105,
    "Cancelled": 3
  },
  "recentOrders": [...],
  "popularItems": [...]
}
```

### 4.2. Laporan Penjualan (Sales Report)
`GET /api/orders/admin/sales-report`
- **Akses**: Terproteksi (Admin)
- **Query Parameter**:
  - `startDate`: Format `YYYY-MM-DD`
  - `endDate`: Format `YYYY-MM-DD`
- **Respon Sukses (200 OK)**: Objek agregat penjualan berisikan `summary`, daftar transaksi `orders`, performa menu `topItems`, dan tren omzet harian `dailySales` untuk grafik.

---

## 5. Modul Pengguna & Buku Alamat (`/api/users`)

### 5.1. Profil Pengguna Saya
- `GET /api/users/me` (Auth): Mengambil data profil yang terhubung dengan akun yang sedang login.
- `PUT /api/users/me` (Auth): Memperbarui nama dan nomor telepon pengguna.
- `DELETE /api/users/me` (Auth): Menghapus akun sendiri dari tabel `profile`.

### 5.2. Buku Alamat Saya
- `GET /api/users/me/addresses` (Auth): Daftar alamat pengguna terurut berdasarkan `is_default DESC`.
- `POST /api/users/me/addresses` (Auth): Tambah alamat baru (`label`, `street`, `city`, `postal_code`, `is_default`).
- `PUT /api/users/me/addresses/:address_id` (Auth): Perbarui alamat yang sudah ada.
- `DELETE /api/users/me/addresses/:address_id` (Auth): Hapus alamat.

### 5.3. Rute Manajemen Pengguna oleh Admin
- `GET /api/users` (Admin): Menampilkan seluruh pengguna yang terdaftar.
- `PUT /api/users/:id` (Admin): Mengubah nama, nomor telepon, dan role akun (`admin` / `customer`).
- `DELETE /api/users/:id` (Admin): Menghapus pengguna secara paksa.
- `GET /api/users/admin/:user_id/addresses` (Admin): Melihat alamat milik user tertentu.
- `POST /api/users/admin/:user_id/addresses` (Admin): Menambahkan alamat untuk user tertentu.
- `PUT /api/users/admin/:user_id/addresses/:address_id` (Admin): Mengubah alamat user tertentu.
- `DELETE /api/users/admin/:user_id/addresses/:address_id` (Admin): Menghapus alamat user tertentu.

---

## 6. Modul Pengiriman (Shipping)

### 6.1. Kalkulasi Ongkos Kirim Presisi
`GET /api/shipping/calculate`
- **Akses**: Publik
- **Query Parameter**: `destination` (String alamat tujuan lengkap)
- **Contoh Request**:
  `GET /api/shipping/calculate?destination=Jl.+Ir.+H.+Juanda+No.+100+Bandung`
- **Respon Sukses (200 OK)**:
```json
{
  "success": true,
  "origin": "Setiabudi Regency no.G 110, Cihideung, Kec. Parongpong, Kabupaten Bandung Barat, Jawa Barat 40559",
  "destination": "Jl. Ir. H. Juanda No. 100, Dago, Kecamatan Coblong, Kota Bandung",
  "distance": {
    "text": "11.2 km",
    "value": 11200,
    "km": 11.2
  },
  "duration": {
    "text": "28 mins",
    "value": 1680,
    "minutes": 28
  },
  "shipping": {
    "cost": 33600,
    "pricePerKm": 3000,
    "freeThreshold": 500000,
    "formatted": "Rp 33.600"
  }
}
```

### 6.2. Tarif Pengiriman Tier (Fallback)
`GET /api/shipping/tiers`
- **Akses**: Publik
- **Respon Sukses (200 OK)**: Daftar tarif flat berdasarkan zona radius (0-5 km: Rp 15.000; 5-10 km: Rp 30.000; 10-20 km: Rp 50.000; 20-50 km: Rp 100.000).
