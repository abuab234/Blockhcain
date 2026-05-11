# 🧊 Es Gieta Smart-Ops V2

<div align="center">

![Version](https://img.shields.io/badge/version-2.0.0-blue?style=for-the-badge)
![Laravel](https://img.shields.io/badge/Laravel-11-FF2D20?style=for-the-badge&logo=laravel&logoColor=white)
![React](https://img.shields.io/badge/React-18-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![License](https://img.shields.io/badge/license-Proprietary-red?style=for-the-badge)

**Sistem Manajemen Operasional F&B Terintegrasi**
*Role-Based Access Control · Automated Financial Reporting · Digital Receipt*

</div>

---

## 📋 Daftar Isi

- [Deskripsi Proyek](#-deskripsi-proyek)
- [Arsitektur Sistem](#️-arsitektur-sistem)
- [Diagram Alur Data](#-diagram-alur-data)
- [Skema Database](#️-skema-database)
- [Manajemen Hak Akses (RBAC)](#-manajemen-hak-akses-rbac)
- [Tech Stack](#️-tech-stack)
- [Struktur Direktori](#-struktur-direktori)
- [Instalasi & Setup](#-instalasi--setup)
- [Dokumentasi API](#-dokumentasi-api)
- [Roadmap Pengembangan](#-roadmap-pengembangan)
- [Lisensi](#️-lisensi)

---

## 📌 Deskripsi Proyek

**Es Gieta Smart-Ops V2** adalah platform manajemen operasional internal untuk bisnis F&B yang mengintegrasikan alur kerja antara Admin, Karyawan, dan Pelanggan dalam satu ekosistem terpusat. Sistem dirancang dengan prinsip **data integrity first** — setiap transaksi yang terkonfirmasi secara atomik memperbarui status pesanan, menerbitkan invoice digital, dan mencatat entri keuangan dalam satu operasi database yang tidak dapat diinterupsi.

**Problem yang diselesaikan:**
- Eliminasi pencatatan keuangan manual yang rawan human-error
- Sentralisasi pemantauan pesanan dari channel WhatsApp
- Transparansi laporan finansial real-time untuk pengambilan keputusan bisnis
- Audit trail lengkap untuk setiap pergerakan stok dan transaksi

---

## 🏗️ Arsitektur Sistem

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Admin SPA   │  │ Karyawan SPA │  │   User Page      │  │
│  │  (React+Vite)│  │ (React+Vite) │  │  (React+Vite)    │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
└─────────┼─────────────────┼───────────────────┼─────────────┘
          │                 │                   │
          ▼                 ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                    API GATEWAY LAYER                         │
│              Laravel Sanctum (Token Auth)                    │
│              Rate Limiting · CORS · Middleware               │
│              Route Prefix: /api/v1/                          │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Admin Routes │ │  Staff Routes│ │  User Routes │
│  /dashboard  │ │  /orders     │ │  /catalog    │
│  /reports    │ │  /invoices   │ │  /tracker    │
│  /products   │ │  /products   │ │  /wa-order   │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                   SERVICE / BUSINESS LOGIC                   │
│   OrderService · InvoiceService · FinancialLogService        │
│   StockService · ReportAggregatorService                     │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│                    DATABASE LAYER                            │
│                    MySQL 8.0                                 │
│   users · roles · products · orders · order_items           │
│   invoices · financial_logs · stock_movements               │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 Diagram Alur Data

### 1. Alur Utama: Order → Invoice → Financial Log

```mermaid
flowchart TD
    A([👤 Pelanggan]) -->|Kirim pesan WhatsApp| B[WhatsApp Channel]
    B -->|Karyawan terima notifikasi| C[Dashboard Karyawan]
    
    C --> D{Validasi Pesanan}
    D -->|Data tidak lengkap| E[❌ Tolak / Minta klarifikasi]
    D -->|Data valid| F[Input Detail Transaksi ke Sistem]
    
    F --> G[Pilih Item dari Catalog]
    G --> H[Set Estimasi Waktu Selesai]
    H --> I[Submit Order]
    
    I --> J[(DB: orders + order_items\nSTATUS: processing)]
    J --> K[Live Tracker Update → Pelanggan]
    
    K --> L{Pesanan Selesai?}
    L -->|Belum| K
    L -->|Selesai| M[Karyawan: Generate Invoice]
    
    M --> N{{\n🔒 DB Transaction\nAtomic Operation\n}}
    
    N --> N1[(Buat record\ninvoices)]
    N --> N2[(Catat entri\nfinancial_logs)]
    N --> N3[(Update orders\nSTATUS: success)]
    
    N1 & N2 & N3 -->|Semua berhasil| O[✅ Commit Transaction]
    N1 & N2 & N3 -->|Salah satu gagal| P[🔄 Rollback Transaction]
    P --> M
    
    O --> Q[Generate PDF Struk\nvia DomPDF]
    Q --> R[Struk dikirim / dicetak]
    R --> S([📊 Admin: Data terekam\ndi Laporan Keuangan])

    style N fill:#1a1a2e,color:#e0e0ff,stroke:#7c3aed
    style O fill:#064e3b,color:#d1fae5,stroke:#10b981
    style P fill:#7f1d1d,color:#fee2e2,stroke:#ef4444
```

---

### 2. Alur Autentikasi & RBAC

```mermaid
flowchart TD
    A([User Request]) --> B[POST /api/v1/auth/login]
    B --> C{Validasi Kredensial}
    C -->|Gagal| D[401 Unauthorized]
    C -->|Berhasil| E[Generate Sanctum Token]
    E --> F[Return: token + role]
    
    F --> G[Client simpan token\ndi memory/httpOnly cookie]
    G --> H[Request ke Protected Endpoint\nBearer: token]
    
    H --> I[Middleware: auth:sanctum]
    I -->|Token invalid/expired| J[401 → Redirect Login]
    I -->|Token valid| K[Middleware: CheckRole]
    
    K --> L{Role Check}
    L -->|role: admin| M[✅ Akses Admin Routes]
    L -->|role: karyawan| N[✅ Akses Karyawan Routes]
    L -->|role: user| O[✅ Akses User Routes]
    L -->|Role tidak cocok| P[403 Forbidden]

    style M fill:#1e3a5f,color:#bfdbfe,stroke:#3b82f6
    style N fill:#1a3a2a,color:#bbf7d0,stroke:#22c55e
    style O fill:#3b2a1a,color:#fde68a,stroke:#f59e0b
    style P fill:#7f1d1d,color:#fee2e2,stroke:#ef4444
```

---

### 3. Alur Manajemen Stok

```mermaid
flowchart LR
    A[Order Confirmed] -->|order_items qty| B[StockService::deduct]
    B --> C{Stok Cukup?}
    C -->|Tidak| D[❌ Error: Insufficient Stock\nOrder ditolak]
    C -->|Ya| E[UPDATE products.stock]
    E --> F[INSERT stock_movements\ntype: deduction\nref: order_id]
    
    G[Admin: Update Stok Manual] --> H[PUT /api/v1/products/:id/stock]
    H --> I[UPDATE products.stock]
    I --> J[INSERT stock_movements\ntype: restock\nref: admin_id]
    
    F & J --> K[(stock_movements\naudit trail)]

    style D fill:#7f1d1d,color:#fee2e2,stroke:#ef4444
```

---

### 4. Alur Agregasi Laporan Keuangan (Admin)

```mermaid
flowchart TD
    A[Admin: Buka Dashboard] --> B[GET /api/v1/admin/reports]
    B --> C[ReportAggregatorService]
    
    C --> D{Filter Parameter}
    D -->|daily| E[WHERE DATE = today]
    D -->|monthly| F[WHERE MONTH = selected]
    D -->|custom range| G[WHERE date BETWEEN start AND end]
    
    E & F & G --> H[(SELECT SUM amount\nFROM financial_logs\nJOIN invoices ON...)]
    
    H --> I[Return: total_revenue,\ntotal_orders, success_rate,\ntop_products]
    
    I --> J[Dashboard Visualisasi]
    J --> K[Chart: Revenue Trend]
    J --> L[Chart: Order Conversion]
    J --> M[Table: Transaction Detail]
    
    M --> N[Admin: Export PDF Report\nvia DomPDF]

    style H fill:#1a1a2e,color:#e0e0ff,stroke:#7c3aed
```

---

### 5. Alur Live Tracker (Perspektif Pelanggan)

```mermaid
sequenceDiagram
    actor P as Pelanggan
    participant F as Frontend (React)
    participant API as Laravel API
    participant DB as MySQL

    P->>F: Buka halaman tracker\n/track/{order_id}
    F->>API: GET /api/v1/orders/{id}/status
    API->>DB: SELECT status, estimated_done_at\nFROM orders WHERE id = ?
    DB-->>API: { status: "processing", eta: "14:30" }
    API-->>F: 200 OK + order data
    F-->>P: Tampilkan status & countdown timer

    loop Polling setiap 30 detik
        F->>API: GET /api/v1/orders/{id}/status
        API->>DB: SELECT status...
        DB-->>API: current status
        API-->>F: updated data
        F-->>P: Re-render status
    end

    Note over F,API: Karyawan update status
    API->>DB: UPDATE orders SET status='ready'
    DB-->>API: affected rows: 1
    API-->>F: { status: "ready" }
    F-->>P: 🔔 Notifikasi: Pesanan siap!
```

---

## 🗄️ Skema Database

### Entity Relationship Overview

```mermaid
erDiagram
    ROLES {
        bigint id PK
        string name
        timestamp created_at
        timestamp updated_at
    }

    USERS {
        bigint id PK
        bigint role_id FK
        string name
        string email
        string password
        string wa_number
        timestamp email_verified_at
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    PRODUCTS {
        bigint id PK
        string name
        text description
        decimal price
        int stock
        boolean is_available
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    ORDERS {
        bigint id PK
        bigint user_id FK
        bigint karyawan_id FK
        string status
        timestamp estimated_done_at
        text notes
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    ORDER_ITEMS {
        bigint id PK
        bigint order_id FK
        bigint product_id FK
        int qty
        decimal unit_price
        decimal subtotal
    }

    INVOICES {
        bigint id PK
        bigint order_id FK
        string invoice_number
        decimal total_amount
        string pdf_path
        timestamp issued_at
        timestamp created_at
        timestamp updated_at
    }

    FINANCIAL_LOGS {
        bigint id PK
        bigint invoice_id FK
        decimal amount
        string type
        string description
        timestamp logged_at
    }

    STOCK_MOVEMENTS {
        bigint id PK
        bigint product_id FK
        bigint reference_id
        string reference_type
        int qty_change
        string type
        string note
        timestamp created_at
    }

    ROLES ||--o{ USERS : "has many"
    USERS ||--o{ ORDERS : "places"
    USERS ||--o{ ORDERS : "handles (as karyawan)"
    ORDERS ||--o{ ORDER_ITEMS : "contains"
    PRODUCTS ||--o{ ORDER_ITEMS : "referenced in"
    ORDERS ||--|| INVOICES : "generates"
    INVOICES ||--|| FINANCIAL_LOGS : "records"
    PRODUCTS ||--o{ STOCK_MOVEMENTS : "tracked in"
```

### Detail Kolom Penting

#### Tabel `orders` — kolom `status`
| Value | Deskripsi |
|---|---|
| `pending` | Pesanan masuk dari WhatsApp, belum divalidasi karyawan |
| `processing` | Karyawan konfirmasi, sedang diproses |
| `ready` | Pesanan selesai dibuat, siap diambil |
| `success` | Invoice diterbitkan, transaksi selesai |
| `cancelled` | Pesanan dibatalkan |

#### Tabel `financial_logs` — kolom `type`
| Value | Deskripsi |
|---|---|
| `credit` | Pemasukan dari penjualan |
| `debit` | Pengeluaran (refund, dll) |

#### Tabel `stock_movements` — kolom `type`
| Value | Deskripsi |
|---|---|
| `deduction` | Stok berkurang karena order |
| `restock` | Stok bertambah karena input manual admin |
| `adjustment` | Koreksi manual |

---

## 👥 Manajemen Hak Akses (RBAC)

Implementasi menggunakan **Spatie Laravel-Permission** dengan middleware per-route.

```
┌─────────────────────────────────────────────────────────────┐
│ ROLE: admin                                                  │
├─────────────────────────────────────────────────────────────┤
│ ✅ GET  /api/v1/admin/dashboard   — Statistik agregat       │
│ ✅ GET  /api/v1/admin/reports     — Seluruh laporan keuangan│
│ ✅ POST /api/v1/products          — Tambah produk           │
│ ✅ PUT  /api/v1/products/:id      — Update harga & stok     │
│ ✅ DEL  /api/v1/products/:id      — Soft delete produk      │
│ ✅ GET  /api/v1/admin/orders      — Lihat semua pesanan      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ ROLE: karyawan                                               │
├─────────────────────────────────────────────────────────────┤
│ ✅ GET  /api/v1/orders            — Pesanan masuk (own)     │
│ ✅ POST /api/v1/orders            — Input detail pesanan    │
│ ✅ PUT  /api/v1/orders/:id/status — Update status pesanan   │
│ ✅ POST /api/v1/invoices          — Generate invoice/struk  │
│ ✅ GET  /api/v1/products          — Lihat catalog & stok    │
│ ❌ Semua route /admin/*           — Diblokir 403            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ ROLE: user (Pelanggan)                                       │
├─────────────────────────────────────────────────────────────┤
│ ✅ GET  /api/v1/catalog           — Lihat menu tersedia     │
│ ✅ GET  /api/v1/orders/:id/status — Live tracker pesanan    │
│ ✅ GET  /api/v1/orders/:id/wa     — Redirect ke WhatsApp    │
│ ❌ Semua route create/update/delete — Diblokir 403          │
└─────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Layer | Teknologi | Versi | Keterangan |
|---|---|---|---|
| **Frontend** | React JS + Vite | 18.x / 5.x | SPA dengan code-splitting per role |
| **Styling** | Tailwind CSS | 3.x | Utility-first, JIT compiler |
| **Backend** | Laravel | 11.x | REST API, Service Layer pattern |
| **Auth** | Laravel Sanctum | built-in | Token-based auth untuk SPA |
| **RBAC** | Spatie Laravel-Permission | 6.x | DB-driven roles & permissions |
| **Database** | MySQL | 8.0+ | Relational, ACID-compliant |
| **PDF** | barryvdh/laravel-dompdf | 3.x | HTML template → PDF invoice |
| **ORM** | Eloquent | built-in | Dengan SoftDeletes & Transactions |

---

## 📁 Struktur Direktori

```
es-gieta-smart-ops/
│
├── backend/                          # Laravel 11
│   ├── app/
│   │   ├── Http/
│   │   │   ├── Controllers/
│   │   │   │   ├── Admin/
│   │   │   │   │   ├── DashboardController.php
│   │   │   │   │   ├── ReportController.php
│   │   │   │   │   └── ProductController.php
│   │   │   │   ├── Staff/
│   │   │   │   │   ├── OrderController.php
│   │   │   │   │   └── InvoiceController.php
│   │   │   │   └── User/
│   │   │   │       ├── CatalogController.php
│   │   │   │       └── TrackerController.php
│   │   │   └── Middleware/
│   │   │       └── CheckRole.php
│   │   ├── Models/
│   │   │   ├── User.php
│   │   │   ├── Product.php
│   │   │   ├── Order.php
│   │   │   ├── OrderItem.php
│   │   │   ├── Invoice.php
│   │   │   ├── FinancialLog.php
│   │   │   └── StockMovement.php
│   │   └── Services/
│   │       ├── OrderService.php
│   │       ├── InvoiceService.php        # Atomic transaction logic
│   │       ├── FinancialLogService.php
│   │       ├── StockService.php
│   │       └── ReportAggregatorService.php
│   ├── database/
│   │   ├── migrations/
│   │   └── seeders/
│   │       ├── RoleSeeder.php
│   │       └── ProductSeeder.php
│   └── routes/
│       └── api.php                       # Versioned: /api/v1/
│
└── frontend/                         # React 18 + Vite
    ├── src/
    │   ├── pages/
    │   │   ├── admin/
    │   │   │   ├── Dashboard.jsx
    │   │   │   ├── Reports.jsx
    │   │   │   └── Products.jsx
    │   │   ├── staff/
    │   │   │   ├── Orders.jsx
    │   │   │   └── InvoiceGenerator.jsx
    │   │   └── user/
    │   │       ├── Catalog.jsx
    │   │       └── Tracker.jsx
    │   ├── components/
    │   ├── services/                     # Axios API calls per module
    │   └── hooks/
│
└── README.md
```

---

## 🚀 Instalasi & Setup

### Prerequisites

```
- PHP >= 8.2
- Composer >= 2.x
- Node.js >= 20.x + npm
- MySQL 8.0+
```

### Backend (Laravel)

```bash
# 1. Clone repository
git clone <repo-url> && cd es-gieta-smart-ops/backend

# 2. Install dependencies
composer install

# 3. Environment setup
cp .env.example .env
php artisan key:generate

# 4. Konfigurasi .env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=esgieta_db
DB_USERNAME=root
DB_PASSWORD=your_password

# 5. Migrasi & Seeder
php artisan migrate
php artisan db:seed --class=RoleSeeder

# 6. Jalankan server
php artisan serve
```

### Frontend (React + Vite)

```bash
cd es-gieta-smart-ops/frontend

# Install dependencies
npm install

# Konfigurasi API base URL
# Edit src/services/api.js → baseURL: 'http://localhost:8000/api/v1'

# Jalankan dev server
npm run dev
```

---

## 📡 Dokumentasi API

### Authentication

| Method | Endpoint | Deskripsi | Auth |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | Login, return Sanctum token | ❌ |
| `POST` | `/api/v1/auth/logout` | Revoke token | ✅ |
| `GET` | `/api/v1/auth/me` | Data user aktif | ✅ |

### Admin

| Method | Endpoint | Deskripsi | Role |
|---|---|---|---|
| `GET` | `/api/v1/admin/dashboard` | Statistik agregat | admin |
| `GET` | `/api/v1/admin/reports` | Laporan keuangan + filter | admin |
| `GET` | `/api/v1/admin/reports/export` | Export PDF laporan | admin |
| `GET` | `/api/v1/products` | List produk + stok | admin |
| `POST` | `/api/v1/products` | Tambah produk | admin |
| `PUT` | `/api/v1/products/{id}` | Update produk | admin |
| `DELETE` | `/api/v1/products/{id}` | Soft delete produk | admin |

### Karyawan

| Method | Endpoint | Deskripsi | Role |
|---|---|---|---|
| `GET` | `/api/v1/orders` | List pesanan masuk | karyawan |
| `POST` | `/api/v1/orders` | Input pesanan baru | karyawan |
| `PUT` | `/api/v1/orders/{id}/status` | Update status + ETA | karyawan |
| `POST` | `/api/v1/invoices` | Generate invoice (atomic) | karyawan |
| `GET` | `/api/v1/invoices/{id}/download` | Download PDF struk | karyawan |

### User (Pelanggan)

| Method | Endpoint | Deskripsi | Role |
|---|---|---|---|
| `GET` | `/api/v1/catalog` | Menu tersedia | user |
| `GET` | `/api/v1/orders/{id}/status` | Live tracker status | user |
| `GET` | `/api/v1/orders/{id}/wa` | Redirect WhatsApp order | user |

---

## 📅 Roadmap Pengembangan

> Urutan berdasarkan dependency chain — setiap fase adalah prerequisite fase berikutnya.

```
FASE 1 — FOUNDATION
├── [x] Inisialisasi project Laravel 11 + React Vite
├── [ ] Setup MySQL & konfigurasi .env
├── [ ] Perancangan & eksekusi migrasi database
│         users, roles, products, orders, order_items,
│         invoices, financial_logs, stock_movements
└── [ ] RoleSeeder (admin, karyawan, user)

FASE 2 — AUTENTIKASI & RBAC
├── [ ] Install & konfigurasi Laravel Sanctum
├── [ ] Install & konfigurasi Spatie Laravel-Permission
├── [ ] AuthController (login, logout, me)
├── [ ] Middleware CheckRole per route group
└── [ ] Testing: isolasi akses antar role

FASE 3 — CORE BUSINESS LOGIC
├── [ ] CRUD Products (Admin) + StockService
├── [ ] OrderController + OrderService (Karyawan)
├── [ ] Validasi stok saat order dibuat
└── [ ] Status update pipeline (pending → processing → ready)

FASE 4 — INVOICE & KEUANGAN (KRITIKAL)
├── [ ] InvoiceService dengan DB::transaction atomic
│         Buat invoice + catat financial_log + update order
├── [ ] Install DomPDF (barryvdh/laravel-dompdf)
├── [ ] Template HTML untuk struk digital
└── [ ] Endpoint download PDF invoice

FASE 5 — DASHBOARD & REPORTING (ADMIN)
├── [ ] ReportAggregatorService (daily/monthly/custom)
├── [ ] Dashboard API: total pesanan, konversi, revenue
├── [ ] Export PDF laporan keuangan
└── [ ] Visualisasi chart di frontend (React)

FASE 6 — USER INTERFACE
├── [ ] Frontend: Auth pages + token management
├── [ ] Frontend: Admin dashboard + laporan
├── [ ] Frontend: Karyawan order management + invoice generator
├── [ ] Frontend: User catalog + WhatsApp redirect
└── [ ] Frontend: Live Tracker dengan polling 30 detik

FASE 7 — HARDENING & DEPLOYMENT
├── [ ] Rate limiting pada semua endpoint publik
├── [ ] Soft deletes verification (products, orders)
├── [ ] Environment separation (APP_DEBUG=false production)
├── [ ] Database backup strategy (mysqldump cron)
└── [ ] Final QA: RBAC audit, financial data integrity test
```

---

## ⚙️ Keputusan Teknis & Rationale

| Keputusan | Pilihan | Alternatif | Alasan |
|---|---|---|---|
| Auth mechanism | Laravel Sanctum | Passport (OAuth2) | Sanctum cukup untuk SPA internal; Passport overkill |
| RBAC | Spatie Permission | Gate & Policies | DB-driven, scalable tanpa deploy ulang |
| PDF | DomPDF | Browsershot | DomPDF zero dependency eksternal; Browsershot butuh Node.js di server |
| Polling vs WebSocket | HTTP Polling (30s) | Laravel Echo + Pusher | Cukup untuk use case tracker; WebSocket butuh infra tambahan |
| Soft Deletes | Aktif di semua tabel | Hard delete | Financial data tidak boleh dihapus permanen |
| Atomic Transaction | `DB::transaction()` | Manual rollback | Mencegah partial write pada financial_logs |

---

## 🛡️ Lisensi

```
Copyright © 2026 abuab. All Rights Reserved.

Sistem ini adalah perangkat lunak proprietary.
Dilarang keras mendistribusikan, memodifikasi, atau menggunakan
ulang kode ini tanpa izin tertulis dari pemilik.
```

---

<div align="center">
  <sub>Es Gieta Smart-Ops V2 · Built with Laravel 11 + React 18 · © 2026 abuab</sub>
</div>
