# 📖 Dapur Nekti (MJ Kitchen) — Dokumentasi Sistem Lengkap

Selamat datang di dokumentasi resmi sistem aplikasi **Dapur Nekti (MJ Kitchen)**. Repositori ini berisi dokumentasi teknis dan operasional komprehensif yang mencakup arsitektur sistem, desain antarmuka, spesifikasi API backend, skema database, alur bisnis checkout & payment gateway, manajemen state frontend, hingga panduan deployment CI/CD.

---

## 📌 Daftar Isi Dokumentasi

Dokumentasi ini dibagi menjadi beberapa bagian utama:

| No | Modul Dokumentasi | Deskripsi | Lokasi File |
|---|---|---|---|
| 1 | **System Architecture** | Arsitektur topologi sistem, relasi Client-Server, integrasi third-party (Supabase, Midtrans, Distance Matrix), dan keamanan jaringan. | [system-architecture.md](file:///d:/mj-kitchen/mj-docs/architecture/system-architecture.md) |
| 2 | **Database Schema & ERD** | Skema lengkap PostgreSQL di Supabase, relasi antar tabel (ERD), constraint, indeks, dan trigger otomatis pembuatan profil. | [database-schema.md](file:///d:/mj-kitchen/mj-docs/architecture/database-schema.md) |
| 3 | **Backend Architecture & Flow** | Arsitektur Express.js (ESM), PostgreSQL Connection Pool, middleware authentication & authorization, struktur folder backend. | [architecture-and-flow.md](file:///d:/mj-kitchen/mj-docs/backend/architecture-and-flow.md) |
| 4 | **Backend API Reference** | Spesifikasi lengkap seluruh endpoint REST API, parameter query, payload request, kode status, dan contoh respon JSON. | [api-documentation.md](file:///d:/mj-kitchen/mj-docs/backend/api-documentation.md) |
| 5 | **Backend Business Logic** | Logika transaksi ACID pemesanan (Order), webhook state machine Midtrans, algoritma perhitungan ongkir Distance Matrix, dan kalkulasi add-on. | [business-logic.md](file:///d:/mj-kitchen/mj-docs/backend/business-logic.md) |
| 6 | **Frontend Architecture & Design** | Desain antarmuka (Design System & Color Palette), konfigurasi Tailwind CSS, Vite + React 19, sistem routing & Route Guards. | [architecture-and-design.md](file:///d:/mj-kitchen/mj-docs/frontend/architecture-and-design.md) |
| 7 | **Frontend State & Services** | Manajemen state global menggunakan Zustand (`authStore`, `cartStore` dengan local persistence), integrasi Axios Interceptors & Supabase JS client. | [state-management.md](file:///d:/mj-kitchen/mj-docs/frontend/state-management.md) |
| 8 | **Frontend Pages & Components** | Dokumentasi interaksi halaman pelanggan (Menu, Cart, Tracking, Receipt) serta modul panel Admin (Dashboard, Menu, Order, Report Excel). | [pages-and-components.md](file:///d:/mj-kitchen/mj-docs/frontend/pages-and-components.md) |
| 9 | **DevOps & Cloud Deployment** | Panduan deployment otomatis via GitHub Actions ke Google Cloud Run, konfigurasi hosting Vercel, dan kamus variabel lingkungan (`.env`). | [devops-guide.md](file:///d:/mj-kitchen/mj-docs/devops/devops-guide.md) |
| 10 | **Testing & System Evaluation** | Laporan evaluasi pengujian komprehensif: Skenario Midtrans (Success, Pending, Failed), pengujian batas transisi status pengiriman (*State Machine*), dan analisis nilai batas radius pengiriman (*Boundary Value Analysis*). | [testing-evaluation.md](file:///d:/mj-kitchen/mj-docs/testing/testing-evaluation.md) |

---

## 🏢 Gambaran Umum Sistem (System Overview)

**Dapur Nekti** (dikenal secara internal sebagai **MJ Kitchen**) adalah platform *e-commerce food & catering* modern yang dirancang untuk melayani pemesanan makanan, nasi box, dan katering dalam skala satuan hingga ribuan porsi secara dadakan (*on-demand catering*).

```
   Tagline Brand: "Pesan Dadakan? Tanpa Deg-degan!"
   Alamat Dapur Utama: Setiabudi Regency no.G 110, Cihideung, Parongpong, Kab. Bandung Barat
```

### 🌟 Fitur Unggulan Sistem:
1. **Katalog Menu Dinamis & Galeri**: Pengelolaan kategori makanan, item menu, foto produk berbasis Supabase Storage, ketersediaan menu (*availability flag*), dan add-on item (sambal, nasi, kerupuk, dll.).
2. **Kalkulator Ongkos Kirim Real-Time**: Perhitungan tarif jarak presisi menggunakan integrasi Google Distance Matrix AI / Distance Matrix API berdasarkan koordinat dan rute jalan raya riil ke alamat tujuan pelanggan.
3. **Multi-Alamat Pelanggan**: Dukungan penyimpanan banyak alamat per pelanggan dengan penanda alamat utama (*default address*).
4. **Keranjang Belanja Fleksibel**: Keranjang belanja modular yang menyimpan kustomisasi catatan khusus (*notes* per item) dan pilihan varian/add-on, dipersistensikan di `localStorage`.
5. **Midtrans Snap & Core API Integration**: Pembayaran digital instan melalui QRIS, Virtual Account (BCA, BNI, BRI, Mandiri, Permata), Kartu Kredit, dan e-Wallet dengan sinkronisasi status otomatis via Webhook Notification.
6. **Lacak Status Pesanan & Struk Digital**: Pelacakan status pemrosesan hidangan secara transparan (Pending -> Confirmed -> Processing -> Out for Delivery -> Delivered) serta cetak struk pesanan digital (*thermal printable*).
7. **Backoffice Admin Lengkap**:
   - Dashboard KPI & Analisis Performa Penjualan.
   - Manajemen Pesanan & Pembaruan Status Real-time.
   - Manajemen Menu, Kategori, Galeri, dan Add-on.
   - Manajemen Pelanggan, Alamat, dan Pengaturan Role (Admin / Customer).
   - Laporan Penjualan Akuntansi dengan Ekspor Multi-Sheet Excel (`.xlsx`) lengkap dengan Ringkasan Eksekutif, Daftar Transaksi, dan Performa Menu Terlaris.

---

## 🛠️ Ringkasan Teknologi (Technology Stack)

```
[ Frontend ]
  ├── React 19 + Vite 7 (Single Page Application)
  ├── Tailwind CSS v3 (Custom Teal Branding & Responsive Layout)
  ├── Zustand v5 (Reactive Store with LocalStorage Persistence)
  ├── React Router DOM v7 (Data & Component Routing with Guards)
  ├── Axios (HTTP Client with Bearer Token Interceptors)
  ├── @supabase/supabase-js (Client Auth & Object Storage Uploads)
  ├── SweetAlert2 (Interactive Dialogs & Toast Notifications)
  └── SheetJS (xlsx) & date-fns (Excel Report Export & Date Formatting)

[ Backend ]
  ├── Node.js (v18+) Runtime (ESM - ECMAScript Modules)
  ├── Express.js v4.21 (RESTful API Server)
  ├── PostgreSQL Database (Hosted on Supabase) via 'pg' Pool
  ├── @supabase/supabase-js (Admin Service Role Auth Token Verification)
  ├── Midtrans Client SDK (Snap & CoreApi Engine)
  ├── Swagger UI Express & Swagger-JSDoc (OpenAPI 3.0 Documentation)
  └── CORS & Dotenv (Security and Environment Configuration)

[ Infrastructure & 3rd Party APIs ]
  ├── Google Cloud Run (Containerless Serverless Backend Deployment)
  ├── Vercel (Frontend Global CDN Hosting)
  ├── GitHub Actions (Automated CI/CD Pipeline on Main Branch Push)
  ├── Distance Matrix AI (Origin-to-Destination Distance Calculation)
  └── Midtrans Payment Gateway (Indonesian Payment Gateway)
```

---

## 🚀 Panduan Memulai Cepat (Quick Start)

### 1. Prasyarat
- Node.js versi 18 ke atas
- Akun Supabase (PostgreSQL + Auth + Storage bucket `menu-images`)
- Akun Midtrans (Sandbox / Production)
- API Key Distance Matrix AI

### 2. Menjalankan Backend (`mj-backend`)
```bash
cd mj-backend
npm install

# Buat file .env dan lengkapi konfigurasi (lihat panduan devops)
npm start
# Server berjalan di http://localhost:8080
# Dokumentasi Swagger dapat diakses di http://localhost:8080/api-docs
```

### 3. Menjalankan Frontend (`mj-frontend`)
```bash
cd mj-frontend
npm install

# Buat file .env dan isi VITE_API_URL, VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY
npm run dev
# Frontend berjalan di http://localhost:5173
```
