# 🚀 Panduan DevOps, CI/CD & Deployment Cloud (DevOps Guide)

Dokumen ini menjelaskan konfigurasi continuous integration & continuous deployment (CI/CD), arsitektur deployment cloud, manajemen variabel lingkungan (*environment variables*), dan prosedur operasional untuk **Dapur Nekti (MJ Kitchen)**.

---

## 1. Topologi Deployment Cloud

```mermaid
flowchart LR
    subgraph Development ["Lingkungan Pengembangan"]
        Dev["Developer (Git Push)"]
    end

    subgraph CI_CD ["GitHub Actions CI/CD"]
        GHActions["main.yml Workflow"]
    end

    subgraph GCP ["Google Cloud Platform (asia-southeast2)"]
        GCPAuth["Google Auth (@v2)"]
        CloudBuild["Cloud Build (Buildpacks)"]
        CloudRun["Google Cloud Run\n(Service: mj-kitchen)"]
    end

    subgraph Vercel_Platform ["Vercel Edge Platform"]
        VercelGit["Vercel GitHub Integration"]
        VercelCDN["Vercel Global CDN (SPA)"]
    end

    Dev -->|"Push ke branch 'main'"| GHActions
    Dev -->|"Push frontend repo"| VercelGit

    GHActions --> GCPAuth
    GCPAuth --> CloudBuild
    CloudBuild --> CloudRun

    VercelGit --> VercelCDN
```

---

## 2. Pipeline Backend CI/CD (Google Cloud Run)

Backend dideploy secara otomatis menggunakan **GitHub Actions** yang didefinisikan pada [`.github/workflows/main.yml`](file:///d:/mj-kitchen/mj-backend/.github/workflows/main.yml).

### Alur Kerja Deployment:
1. **Trigger**: Terpicu setiap kali terdapat perubahan kode yang di-*push* ke branch `main`.
2. **Checkout**: Menggunakan `actions/checkout@v4`.
3. **Autentikasi GCP**: Menggunakan `google-github-actions/auth@v2` dengan kredensial service account JSON dari GitHub Secret `GOOGLE_CREDENTIALS`.
4. **Deploy Serverless**:
   Mengeksekusi perintah deployment Google Cloud Run menggunakan **Google Cloud Buildpacks** (tanpa perlu mengelola Dockerfile manual):
   ```bash
   gcloud run deploy mj-kitchen \
     --source . \
     --region asia-southeast2 \
     --platform managed \
     --allow-unauthenticated \
     --service-account=renz-cloud@${{ secrets.GCP_PROJECT_ID }}.iam.gserviceaccount.com \
     --set-env-vars=SUPABASE_URL=${{ secrets.SUPABASE_URL }},...
   ```
5. **Region**: `asia-southeast2` (Jakarta, Indonesia) untuk memastikan latensi minimal bagi pengguna lokal di wilayah Jawa Barat dan sekitarnya.
6. **Verifikasi Layanan**: Menjalankan `gcloud run services describe mj-kitchen --region asia-southeast2`.

---

## 3. Deployment Frontend (Vercel)

Frontend React dihosting pada platform **Vercel** untuk memanfaatkan Edge Caching dan CDN global.

### Konfigurasi Routing Single Page Application (`vercel.json`):
Karena React Router DOM menggunakan *client-side routing* (History API), semua URL rute browser harus diarahkan kembali ke `index.html`:
```json
{
  "rewrites": [
    {
      "source": "/(.*)",
      "destination": "/index.html"
    }
  ]
}
```

### Parameter Build Vercel:
- **Framework Preset**: Vite
- **Build Command**: `npm run build`
- **Output Directory**: `dist`
- **Install Command**: `npm install`

---

## 4. Matriks Variabel Lingkungan (Environment Variables Matrix)

Berikut adalah daftar variabel lingkungan yang wajib dikonfigurasi pada masing-masing layanan:

### 4.1. Backend (`mj-backend/.env` & GitHub Actions Secrets)

| Nama Variabel | Wajib | Contoh Nilai | Deskripsi & Keamanan |
|---|---|---|---|
| `PORT` | Tidak | `8080` | Port listen server Express (diisi otomatis oleh Cloud Run) |
| `DATABASE_URL` | **YA** | `postgresql://postgres.[ref]:[pass]@aws-0-ap-southeast-1.pooler.supabase.com:6543/postgres` | Connection string PostgreSQL Supabase dengan pooling |
| `DB_SSL` | **YA** | `true` | Mengaktifkan koneksi SSL untuk database pooler |
| `SUPABASE_URL` | **YA** | `https://xyzproject.supabase.co` | URL proyek Supabase |
| `SUPABASE_SERVICE_ROLE_KEY` | **YA** | `eyJhbGciOi...` | **Kunci Rahasia Utama**: Digunakan backend untuk memverifikasi token JWT dan bypass RLS |
| `MIDTRANS_SERVER_KEY` | **YA** | `SB-Mid-server-xxxx` | Kunci server otorisasi transaksi Midtrans |
| `MIDTRANS_CLIENT_KEY` | **YA** | `SB-Mid-client-xxxx` | Kunci publik client Midtrans |
| `MIDTRANS_IS_PRODUCTION` | **YA** | `false` | `false` untuk Sandbox, `true` untuk Production |
| `DISTANCE_MATRIX_API_KEY` | **YA** | `AIzaSy...` | Kunci API Distance Matrix AI untuk perhitungan jarak antar |

---

### 4.2. Frontend (`mj-frontend/.env` & Vercel Environment Variables)

| Nama Variabel | Wajib | Contoh Nilai | Deskripsi & Keamanan |
|---|---|---|---|
| `VITE_API_URL` | **YA** | `https://mj-kitchen-489615087978.asia-southeast2.run.app/api` | Base URL endpoint REST backend |
| `VITE_SUPABASE_URL` | **YA** | `https://xyzproject.supabase.co` | URL proyek Supabase publik |
| `VITE_SUPABASE_ANON_KEY` | **YA** | `eyJhbGciOi...` | Public Anonymous Key Supabase untuk auth klien & upload Storage |

---

## 5. Konfigurasi Webhook Midtrans di Dashboard Merchant

Agar notifikasi perubahan status pembayaran berhasil diterima backend secara real-time, daftarkan URL berikut pada portal Midtrans Merchant Dashboard:

1. Masuk ke **Midtrans Dashboard** -> Pilih Environment (*Sandbox* atau *Production*).
2. Masuk ke menu **Settings** -> **Configuration**.
3. Pada kolom **Payment Notification URL**, masukkan:
   ```
   https://mj-kitchen-489615087978.asia-southeast2.run.app/api/orders/midtrans-notification
   ```
4. Pastikan flag HTTP 200 diaktifkan.

---

## 6. Prosedur Uji Kesehatan Sistem (Health Check & Diagnostics)

### 6.1. Uji Endpoint Backend
Jalankan perintah HTTP GET pada root backend:
```bash
curl -i https://mj-kitchen-489615087978.asia-southeast2.run.app/
```
**Respon yang diharapkan**:
```
HTTP/2 200
content-type: text/html; charset=utf-8

Dapur Nekti API OK - Docs at /api-docs
```

### 6.2. Memeriksa Log Cloud Run
Untuk memantau log transaksi dan log notifikasi Midtrans secara real-time:
```bash
gcloud beta run services logs tail mj-kitchen --region asia-southeast2
```
Atau melalui Google Cloud Console -> **Cloud Run** -> **mj-kitchen** -> **Logs**.
