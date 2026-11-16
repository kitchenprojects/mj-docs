# ⚙️ Arsitektur & Desain Rekayasa Backend (Backend Architecture & Flow)

Dokumen ini menjelaskan struktur rekayasa perangkat lunak pada repositori **`mj-backend`**, pola desain (*design patterns*), siklus hidup request-response (*request lifecycle*), konfigurasi koneksi database, dan sistem middleware.

---

## 1. Struktur Direktori Backend

Kode sumber backend dirancang menggunakan arsitektur modular **Controller-Route-Middleware**:

```
mj-backend/
├── .github/
│   └── workflows/
│       └── main.yml           # CI/CD GitHub Actions ke Google Cloud Run
├── config/
│   ├── db.js                  # Pool koneksi PostgreSQL (pg)
│   └── midtrans.js            # Inisialisasi SDK Midtrans Snap & CoreApi
├── controllers/
│   ├── addonController.js     # Logika bisnis item add-on
│   ├── menuController.js      # Logika menu, kategori, & galeri foto
│   ├── orderController.js     # Logika transaksi order, payment, & reporting
│   ├── shippingController.js  # Integrasi Distance Matrix AI untuk ongkir
│   └── userController.js      # Manajemen profil pengguna & alamat
├── middleware/
│   ├── adminOnly.js           # Guard otorisasi Role-Based Access Control (Admin)
│   └── auth.js                # Verifikasi JWT Supabase & injeksi req.profile
├── routes/
│   ├── addonRoutes.js         # Endpoint /api/addons
│   ├── menuRoutes.js          # Endpoint /api/menu
│   ├── orderRoutes.js         # Endpoint /api/orders
│   ├── shippingRoutes.js      # Endpoint /api/shipping
│   └── userRoutes.js          # Endpoint /api/users
├── .gitignore
├── package.json               # Konfigurasi dependensi ESM ("type": "module")
├── server.js                  # Entry point Express.js HTTP Server
├── swagger.js                 # Konfigurasi OpenAPI 3.0 JSDoc
└── swagger-docs.js            # Anotasi dokumentasi endpoint Swagger
```

---

## 2. Inisialisasi Server & Siklus Request-Response

Entry point aplikasi berada pada file [`server.js`](file:///d:/mj-kitchen/mj-backend/server.js). Alur pemrosesan request masuk adalah sebagai berikut:

```mermaid
flowchart TD
    ClientReq["Client HTTP Request"] --> ExpressApp["Express Application Core"]
    ExpressApp --> CorsMiddleware["CORS Middleware (Allow Origins)"]
    ExpressApp --> JsonParser["express.json() Body Parser"]
    
    ExpressApp --> RouteDispatcher{"Route Dispatcher"}
    
    RouteDispatcher -->|"/api-docs"| SwaggerDocs["Swagger UI Documentation"]
    RouteDispatcher -->|"Public Routes (/api/menu, /api/shipping)"| ControllerPub["Public Controllers"]
    RouteDispatcher -->|"Protected Routes"| AuthMiddleware["Auth Middleware (auth.js)"]
    
    AuthMiddleware -->|Token Invalid/Expired| Return401["HTTP 401 Unauthorized"]
    AuthMiddleware -->|Token Valid| AttachContext["Decorate req.user & req.profile"]
    
    AttachContext --> AdminCheck{"Requires Admin?"}
    AdminCheck -->|Yes| AdminOnlyMiddleware["AdminOnly Guard (adminOnly.js)"]
    AdminCheck -->|No| ControllerUser["User Controllers"]
    
    AdminOnlyMiddleware -->|Role != 'admin'| Return403["HTTP 403 Forbidden"]
    AdminOnlyMiddleware -->|Role == 'admin'| ControllerAdmin["Admin Controllers"]
    
    ControllerPub & ControllerUser & ControllerAdmin --> PgPool["PostgreSQL Client Execution"]
    PgPool --> SuccessResponse["HTTP 200/201 JSON Response"]
    
    RouteDispatcher -->|Route Not Found| NotFoundHandler["HTTP 404 Handler"]
    ExpressApp -->|Unhandled Exception| ErrorHandler["HTTP 500 Global Error Handler"]
```

### Konfigurasi Cloud Run Port Binding
Google Cloud Run menyuntikkan environment variable `PORT` secara dinamis pada saat container start. Server mengikat listener pada interface `0.0.0.0`:
```javascript
const PORT = process.env.PORT || 8080;
app.listen(PORT, '0.0.0.0', () => {
  console.log(`🚀 Dapur Nekti API running on port ${PORT}`);
});
```

---

## 3. Konfigurasi Koneksi Database (`config/db.js`)

Koneksi ke PostgreSQL di Supabase dikelola menggunakan **Connection Pooling** dari package `pg` (node-postgres).

### Fitur & Keandalan Pool:
1. **Dual Configuration Mode**:
   - Memprioritaskan satu connection string penuh via `DATABASE_URL` (format: `postgres://user:pass@host:port/dbname`).
   - Fallback ke konfigurasi granular (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`).
2. **SSL Encryption**:
   - Jika `DB_SSL === 'true'`, koneksi SSL diaktifkan dengan opsi `{ rejectUnauthorized: false }` agar kompatibel dengan sertifikat pooler Supabase.
3. **Resiliensi Koneksi**:
   - `connectionTimeoutMillis: 5000`: Mencegah server hang jika database tidak dapat dijangkau lebih dari 5 detik.
   - `idleTimeoutMillis: 30000`: Koneksi yang menganggur selama 30 detik dilepas secara otomatis untuk membebaskan batas pool database.
4. **Event Listeners**:
   - `pool.on('connect')`: Mencatat status log saat koneksi baru berhasil dibuat.
   - `pool.on('error')`: Mencegah crash fatal saat terjadi unexpected error pada idle client.

---

## 4. Sistem Middleware

### 4.1. Authentication Middleware (`middleware/auth.js`)
Middleware ini bertindak sebagai penjaga identitas untuk seluruh rute terproteksi:

1. **Ekstraksi Token**: Mengambil header `Authorization` dan memotong prefix `Bearer `.
2. **Verifikasi via Supabase Service Role**:
   ```javascript
   const { data, error } = await supabase.auth.getUser(token);
   ```
   Supabase client diinisialisasi dengan `SUPABASE_SERVICE_ROLE_KEY` sehingga memiliki otorisasi penuh untuk memvalidasi token JWT pengguna tanpa persistensi session lokal.
3. **Pemuatan Profil Database**:
   Melakukan query ke tabel `profile` menggunakan ID pengguna `user.id`.
4. **Auto-Profile Recovery (Fallback)**:
   Jika akun sudah ada di `auth.users` tetapi record tabel `profile` belum terbentuk (misal karena trigger Supabase tertunda), middleware secara otomatis mengeksekusi insert record profil baru dengan default role `'customer'`.
5. **Injeksi Request Context**:
   - `req.user`: Objek pengguna dari Supabase Auth (email, user metadata).
   - `req.profile`: Record profil lengkap dari database PostgreSQL (id, role, name, phone_number, email).

### 4.2. Authorization Guard (`middleware/adminOnly.js`)
Dijalankan setelah `auth.js`. Memeriksa apakah `req.profile?.role` bernilai `'admin'`. Jika bukan admin, mengembalikan respon status `403 Forbidden`:
```javascript
const adminOnly = (req, res, next) => {
  const role = req.profile?.role;
  if (role && role.toLowerCase() === 'admin') {
    return next();
  }
  res.status(403).json({ message: 'Forbidden: Admin access required' });
};
```

---

## 5. Dokumentasi Interaktif OpenAPI / Swagger UI

Backend dilengkapi Swagger UI interaktif yang dapat diakses langsung pada endpoint browser:
`http://localhost:8080/api-docs` atau di URL Cloud Run:
`https://mj-kitchen-489615087978.asia-southeast2.run.app/api-docs`

Dikonfigurasi pada [`swagger.js`](file:///d:/mj-kitchen/mj-backend/swagger.js) dan [`swagger-docs.js`](file:///d:/mj-kitchen/mj-backend/swagger-docs.js) dengan skema komponen lengkap untuk `MenuItem`, `Category`, `Order`, `OrderItem`, `User`, `Address`, dan `ShippingResult`.
