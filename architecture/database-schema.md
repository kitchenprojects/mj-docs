# 🗄️ Skema Database & ERD (Database Schema)

Dokumen ini mendeskripsikan struktur relasional basis data PostgreSQL yang di-host pada platform Supabase untuk aplikasi **Dapur Nekti (MJ Kitchen)**, mencakup diagram entitas relasi (ERD), definisi kamus data (*data dictionary*), constraint, dan trigger.

---

## 1. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    AUTH_USERS ||--|| PROFILE : "1:1 owns (Supabase Auth)"
    PROFILE ||--o{ USER_ADDRESS : "1:N has addresses"
    PROFILE ||--o{ ORDER : "1:N places orders"
    
    CATEGORY ||--o{ MENU : "1:N classifies"
    MENU ||--o{ GALLERY : "1:N displays photos"
    
    USER_ADDRESS ||--o{ ORDER : "1:N delivers to"
    ORDER ||--|{ ORDER_ITEM : "1:N contains items"
    ORDER ||--|| PAYMENT : "1:1 payment record"
    
    MENU ||--o{ ORDER_ITEM : "1:N ordered as"

    PROFILE {
        uuid id PK "Matches auth.users.id"
        varchar role "Role ('admin' | 'customer')"
        varchar name "Nama lengkap pengguna"
        varchar phone_number "Nomor WhatsApp / telepon"
        timestamp created_at "Waktu registrasi"
        timestamp updated_at "Waktu pembaruan profil"
    }

    USER_ADDRESS {
        uuid address_id PK "Unique address ID"
        uuid user_id FK "References profile.id"
        varchar label "Label (Rumah, Kantor, dll)"
        text street "Alamat lengkap jalan/nomor"
        varchar city "Nama kota / kabupaten"
        varchar postal_code "Kode pos"
        boolean is_default "Penanda alamat utama"
    }

    CATEGORY {
        int category_id PK "Auto-increment ID"
        varchar category_name "Nama kategori (Nasi Box, dll)"
        text description "Deskripsi kategori"
        text image_url "URL ikon / gambar sampul"
    }

    MENU {
        int menu_id PK "Auto-increment ID"
        int category_id FK "References category.category_id"
        varchar menu_name "Nama menu / hidangan"
        text description "Deskripsi komposisi hidangan"
        numeric price "Harga per porsi (IDR)"
        boolean is_available "Status ketersediaan menu"
    }

    GALLERY {
        int gallery_id PK "Auto-increment ID"
        int menu_id FK "References menu.menu_id (CASCADE)"
        text image_url "Public URL Supabase Storage"
        text caption "Keterangan foto menu"
    }

    ORDER {
        uuid order_id PK "UUID unik pesanan"
        uuid user_id FK "References profile.id"
        uuid address_id FK "References user_address.address_id"
        timestamp order_date "Waktu pembuatan order"
        varchar status "Pending | Confirmed | Out for Delivery | Delivered | Cancelled"
        numeric total_amount "Total keseluruhan (Items + Ongkir)"
        numeric shipping_cost "Biaya pengiriman"
    }

    ORDER_ITEM {
        int order_item_id PK "Auto-increment ID"
        uuid order_id FK "References order.order_id (CASCADE)"
        int menu_id FK "References menu.menu_id"
        int quantity "Jumlah porsi dipesan"
        numeric price_each "Harga satuan saat transaksi"
        numeric subtotal "Subtotal (quantity * price_each)"
    }

    PAYMENT {
        int payment_id PK "Auto-increment ID"
        uuid order_id FK "References order.order_id (CASCADE)"
        varchar payment_method_id "Metode bayar (Opsional)"
        timestamp payment_date "Waktu pembayaran berhasil diverifikasi"
        numeric amount_paid "Nominal yang dibayarkan"
        varchar payment_status "Unpaid | Pending | Paid | Failed"
        text snap_token "Token transaksi Midtrans Snap"
    }
```

---

## 2. Kamus Data (Data Dictionary)

### 2.1. Tabel `profile`
Menyimpan atribut profil pengguna publik yang terikat langsung dengan `auth.users` di Supabase.

| Kolom | Tipe Data | Nullable | Default | Keterangan |
|---|---|---|---|---|
| `id` | `UUID` | **NO** | - | Primary Key, merujuk ke `auth.users.id` (ON DELETE CASCADE) |
| `name` | `VARCHAR(255)` | **NO** | - | Nama lengkap pelanggan atau admin |
| `phone_number` | `VARCHAR(50)` | YES | `NULL` | Nomor telepon / nomor kontak WhatsApp |
| `role` | `VARCHAR(20)` | **NO** | `'customer'` | Nilai valid: `'admin'` atau `'customer'` |
| `created_at` | `TIMESTAMPTZ` | **NO** | `NOW()` | Waktu pembuatan akun |
| `updated_at` | `TIMESTAMPTZ` | YES | `NULL` | Waktu pembaruan data akun |

> **Catatan Alur Sinkronisasi**: Email disimpan di `auth.users` Supabase. Pada query backend `auth.js`, jika record profil belum terbentuk di PostgreSQL saat login pertama kali, middleware backend menyertakan fallback insert otomatis dengan nilai role `'customer'`.

---

### 2.2. Tabel `user_address`
Menyimpan buku alamat (*address book*) milik pelanggan untuk pengiriman pesanan.

| Kolom | Tipe Data | Nullable | Default | Keterangan |
|---|---|---|---|---|
| `address_id` | `SERIAL` / `UUID` | **NO** | - | Primary Key |
| `user_id` | `UUID` | **NO** | - | Foreign Key ke `profile(id)` (ON DELETE CASCADE) |
| `label` | `VARCHAR(100)` | **NO** | - | Label alamat (misal: "Rumah", "Kantor", "Gedung A") |
| `street` | `TEXT` | **NO** | - | Detail alamat jalan, nomor rumah, RT/RW, kelurahan |
| `city` | `VARCHAR(100)` | **NO** | - | Nama kota / kabupaten (digunakan juga untuk kalkulasi ongkir) |
| `postal_code` | `VARCHAR(20)` | YES | `NULL` | Kode pos |
| `is_default` | `BOOLEAN` | **NO** | `false` | Menandai apakah alamat ini merupakan alamat pengiriman utama |

> **Logika Default**: Ketika alamat ditandai `is_default = true`, sistem menjalankan perintah SQL untuk mereset seluruh alamat lain milik `user_id` tersebut menjadi `is_default = false` sebelum menyimpan.

---

### 2.3. Tabel `category`
Menyimpan kelompok klasifikasi menu makanan.

| Kolom | Tipe Data | Nullable | Default | Keterangan |
|---|---|---|---|---|
| `category_id` | `SERIAL` / `INT` | **NO** | - | Primary Key |
| `category_name` | `VARCHAR(100)` | **NO** | - | Nama kategori makanan (contoh: "Nasi Box", "Tumpeng", "Snack Box") |
| `description` | `TEXT` | YES | `NULL` | Deskripsi penjelasan paket kategori |
| `image_url` | `TEXT` | YES | `NULL` | URL gambar sampul/banner kategori |

---

### 2.4. Tabel `menu`
Menyimpan item hidangan/makanan yang tersedia untuk dipesan.

| Kolom | Tipe Data | Nullable | Default | Keterangan |
|---|---|---|---|---|
| `menu_id` | `SERIAL` / `INT` | **NO** | - | Primary Key |
| `category_id` | `INT` | **NO** | - | Foreign Key ke `category(category_id)` |
| `menu_name` | `VARCHAR(255)` | **NO** | - | Nama hidangan makanan |
| `description` | `TEXT` | YES | `NULL` | Komposisi lauk, deskripsi rasa, dan porsi |
| `price` | `NUMERIC(12,2)` | **NO** | - | Harga jual per porsi dalam Rupiah |
| `is_available` | `BOOLEAN` | **NO** | `true` | `true` = menu tampil dan dapat dipesan, `false` = stok habis / nonaktif |

> **Konvensi Add-On**: Item add-on (seperti extra sambal, nasi tambahan, kerupuk) juga disimpan di tabel `menu` dengan kriteria harga `<= 10000` atau memiliki kata kunci nama menu `%tambah%`, `%extra%`, `%add%`.

---

### 2.5. Tabel `gallery`
Menyimpan relasi 1:N gambar/foto makanan untuk setiap item menu.

| Kolom | Tipe Data | Nullable | Default | Keterangan |
|---|---|---|---|---|
| `gallery_id` | `SERIAL` / `INT` | **NO** | - | Primary Key |
| `menu_id` | `INT` | **NO** | - | Foreign Key ke `menu(menu_id)` (ON DELETE CASCADE) |
| `image_url` | `TEXT` | **NO** | - | URL publik gambar yang tersimpan di Supabase Storage (`menu-images`) |
| `caption` | `VARCHAR(255)` | YES | `NULL` | Keterangan tambahan atau label gambar |

---

### 2.6. Tabel `order`
Menyimpan pesanan induk dari pelanggan.

| Kolom | Tipe Data | Nullable | Default | Keterangan |
|---|---|---|---|---|
| `order_id` | `UUID` | **NO** | - | Primary Key (UUID v4) |
| `user_id` | `UUID` | **NO** | - | Foreign Key ke `profile(id)` |
| `address_id` | `UUID` / `INT` | YES | `NULL` | Foreign Key ke `user_address(address_id)` |
| `order_date` | `TIMESTAMPTZ` | **NO** | `NOW()` | Waktu pemesanan dilakukan |
| `status` | `VARCHAR(50)` | **NO** | `'Pending'` | Status pesanan: `Pending`, `Confirmed`, `Processing`, `Out for Delivery`, `Delivered`, `Cancelled` |
| `total_amount` | `NUMERIC(14,2)` | **NO** | - | Total tagihan akhir: Subtotal item + `shipping_cost` |
| `shipping_cost` | `NUMERIC(12,2)` | **NO** | `0` | Biaya ongkos kirim berdasarkan perhitungan jarak |

---

### 2.7. Tabel `order_item`
Menyimpan rincian item-item menu dan kuantitas dalam suatu pesanan.

| Kolom | Tipe Data | Nullable | Default | Keterangan |
|---|---|---|---|---|
| `order_item_id` | `SERIAL` / `INT` | **NO** | - | Primary Key |
| `order_id` | `UUID` | **NO** | - | Foreign Key ke `order(order_id)` (ON DELETE CASCADE) |
| `menu_id` | `INT` | **NO** | - | Foreign Key ke `menu(menu_id)` |
| `quantity` | `INT` | **NO** | `1` | Jumlah porsi makanan yang dipesan |
| `price_each` | `NUMERIC(12,2)` | **NO** | - | *Snapshot* harga satuan menu saat order dibuat |
| `subtotal` | `NUMERIC(14,2)` | **NO** | - | Hasil perkalian `quantity * price_each` |

---

### 2.8. Tabel `payment`
Menyimpan informasi transaksi pembayaran dan integrasi Midtrans Snap.

| Kolom | Tipe Data | Nullable | Default | Keterangan |
|---|---|---|---|---|
| `payment_id` | `SERIAL` / `INT` | **NO** | - | Primary Key |
| `order_id` | `UUID` | **NO** | - | Foreign Key ke `order(order_id)` (ON DELETE CASCADE) |
| `payment_method_id` | `VARCHAR(50)` | YES | `NULL` | Metode pembayaran (opsional) |
| `payment_date` | `TIMESTAMPTZ` | YES | `NULL` | Timestamp kapan status berubah menjadi `'Paid'` |
| `amount_paid` | `NUMERIC(14,2)` | **NO** | - | Nilai uang yang wajib/sudah dibayar |
| `payment_status` | `VARCHAR(50)` | **NO** | `'Unpaid'` | Status bayar: `'Unpaid'`, `'Pending'`, `'Paid'`, `'Failed'` |
| `snap_token` | `TEXT` | YES | `NULL` | Token transaksi Snap yang di-generate oleh Midtrans SDK |

---

## 3. Aturan Relasi & Integritas Data

1. **Cascade Delete pada Menu**:
   - Jika satu baris di tabel `menu` dihapus oleh Admin, seluruh baris terkait di tabel `gallery` akan dihapus secara otomatis (`ON DELETE CASCADE`).
2. **Cascade Delete pada Order**:
   - Jika order dihapus, seluruh data `order_item` dan `payment` yang terikat pada `order_id` tersebut akan terhapus otomatis.
3. **Immutability Harga Order Item**:
   - Kolom `price_each` pada tabel `order_item` mencatat harga menu saat transaksi terjadi. Perubahan harga menu di kemudian hari pada tabel `menu` tidak akan mengubah data historis pesanan yang telah berlalu.
