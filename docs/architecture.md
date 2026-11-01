# Arsitektur Rombay

Dokumen ini berisi diagram arsitektur, relasi tabel, dan alur proses Rombay. Semua diagram memakai Mermaid sehingga dirender langsung oleh GitHub dan ikut versi kode.

Daftar isi:

1. [Arsitektur runtime](#1-arsitektur-runtime)
2. [Dari repo ke container](#2-dari-repo-ke-container)
3. [Relasi tabel rombay-core](#3-relasi-tabel-rombay-core)
4. [Relasi tabel rombay-ledger (usulan)](#4-relasi-tabel-rombay-ledger-usulan)
5. [Alur pembayaran sukses](#5-alur-pembayaran-sukses)
6. [Jalur keputusan request pembayaran](#6-jalur-keputusan-request-pembayaran)
7. [Siklus status pembayaran](#7-siklus-status-pembayaran)

---

## 1. Arsitektur runtime

Hanya `rombay-gateway` yang terbuka ke luar. Service lain hanya dapat dijangkau lewat network internal `rombay-net`. Garis putus-putus adalah webhook ke merchant.

```mermaid
flowchart LR
  M["Merchant<br/>app + server"]

  subgraph NET["Docker network: rombay-net"]
    G["rombay-gateway<br/>Go :8080"]
    R[("redis<br/>rate limit + cache")]
    C["rombay-core<br/>Spring Boot :8081"]
    CDB[("core-db<br/>PostgreSQL")]
    K{{"kafka<br/>payment.created / settled / failed"}}
    L["rombay-ledger<br/>Rust :8082"]
    LDB[("ledger-db<br/>PostgreSQL")]
  end

  M -->|POST /payments| G
  G -.->|webhook| M
  G --> R
  G --> C
  C --> CDB
  C -->|payment.created| K
  K -->|payment.created| L
  L --> LDB
  L -->|settled / failed| K
  K -->|settled / failed| C
  K -->|status event| G
```

Core juga membaca event hasil dari ledger untuk menyelesaikan status dan saldo, sedangkan gateway membacanya untuk mengirim webhook.

---

## 2. Dari repo ke container

Setiap repo service punya CI sendiri yang menghasilkan image di GHCR. Repo hub `rombay` hanya berisi orkestrasi. Pada mode dev, compose membangun langsung dari folder repo di sebelahnya.

```mermaid
flowchart LR
  core["rombay-core<br/>Spring Boot"] --> CI["GitHub Actions<br/>test, build, push"]
  gw["rombay-gateway<br/>Go"] --> CI
  led["rombay-ledger<br/>Rust"] --> CI
  CI --> GHCR["GHCR<br/>image per service"]
  hub["rombay<br/>compose + dokumen"] -->|docker-compose.yml| DC["docker compose<br/>di mesin Anda"]
  GHCR -->|pull image| DC
  core -.->|build lokal, mode dev| DC
  gw -.->|build lokal, mode dev| DC
  led -.->|build lokal, mode dev| DC
```

---

## 3. Relasi tabel rombay-core

Skema lengkap ada di `rombay-core/src/main/resources/db/migration/V1__init.sql`. Garis putus-putus adalah hubungan logis tanpa foreign key.

```mermaid
erDiagram
  ACCOUNTS ||--o| MERCHANTS : "akun merchant"
  MERCHANTS ||--o{ PAYMENTS : membuat
  ACCOUNTS ||--o{ PAYMENTS : pengirim
  ACCOUNTS ||--o{ PAYMENTS : penerima
  PAYMENTS ||--o{ PAYMENT_STATUS_HISTORY : riwayat
  PAYMENTS ||..o{ OUTBOX_EVENTS : "aggregate_id"

  ACCOUNTS {
    uuid id PK
    string account_number UK
    string type
    bigint balance
    bigint held_amount
    bigint version
  }
  MERCHANTS {
    uuid id PK
    uuid account_id FK
    string api_key_hash UK
    string webhook_url
    string status
  }
  PAYMENTS {
    uuid id PK
    string public_id UK
    uuid merchant_id FK
    uuid from_account_id FK
    uuid to_account_id FK
    bigint amount
    bigint fee_amount
    string status
    string idempotency_key
    string request_hash
    bigint version
  }
  PAYMENT_STATUS_HISTORY {
    bigint id PK
    uuid payment_id FK
    string from_status
    string to_status
    string reason
    timestamp created_at
  }
  OUTBOX_EVENTS {
    uuid id PK
    string aggregate_type
    uuid aggregate_id
    string event_type
    jsonb payload
    timestamp published_at
  }
  PROCESSED_EVENTS {
    uuid event_id PK
    string event_type
    timestamp processed_at
  }
```

| Tabel | Kegunaan |
|---|---|
| `accounts` | Saldo operasional (`balance`) dan dana tertahan (`held_amount`) |
| `merchants` | Identitas pemanggil API dan konfigurasi webhook |
| `payments` | Inti transaksi. Menunjuk ke satu merchant dan dua akun (pengirim dan penerima) |
| `payment_status_history` | Jejak audit setiap perubahan status |
| `outbox_events` | Antrean event ke Kafka. Terhubung ke pembayaran lewat `aggregate_id` tanpa foreign key |
| `processed_events` | Penanda event masuk yang sudah diproses |

---

## 4. Relasi tabel rombay-ledger (usulan)

Skema ledger belum difinalkan. Diagram ini adalah usulan awal untuk Fase 3.

```mermaid
erDiagram
  LEDGER_ACCOUNTS ||--o{ JOURNAL_LINES : "dicatat pada"
  JOURNAL_ENTRIES ||--|{ JOURNAL_LINES : "terdiri dari"
  RECON_RUNS ||--o{ RECON_ITEMS : berisi
  JOURNAL_ENTRIES |o--o{ RECON_ITEMS : dicocokkan

  LEDGER_ACCOUNTS {
    uuid id PK
    string account_number UK
    string type
    string currency
  }
  JOURNAL_ENTRIES {
    uuid id PK
    uuid payment_id UK
    uuid source_event_id UK
    timestamp posted_at
  }
  JOURNAL_LINES {
    bigint id PK
    uuid entry_id FK
    uuid account_id FK
    string direction
    bigint amount
  }
  RECON_RUNS {
    uuid id PK
    date business_date
    string bank_file
    string status
  }
  RECON_ITEMS {
    uuid id PK
    uuid run_id FK
    uuid entry_id FK
    string bank_reference
    string result
    bigint ledger_amount
    bigint bank_amount
  }
  PROCESSED_EVENTS {
    uuid event_id PK
    string event_type
    timestamp processed_at
  }
  OUTBOX_EVENTS {
    uuid id PK
    string event_type
    jsonb payload
    timestamp published_at
  }
```

Ide utama:

- Satu pembayaran menghasilkan satu `journal_entries` (`payment_id` unik) dengan minimal dua `journal_lines`. Kolom `direction` bernilai `DEBIT` atau `CREDIT`, dan total debit harus sama dengan total kredit sebelum ditulis.
- `source_event_id` yang unik mencegah event Kafka yang sama dicatat dua kali.
- Rekonsiliasi dipisah menjadi `recon_runs` (satu eksekusi per hari) dan `recon_items` (hasil per transaksi: cocok, hilang di bank, hilang di ledger, atau nominal berbeda).

---

## 5. Alur pembayaran sukses

Merchant menerima `202 PENDING` jauh sebelum ledger selesai. Hasil akhir datang lewat webhook, atau lewat `GET /payments/{id}` jika merchant ingin mengecek sendiri.

```mermaid
sequenceDiagram
  participant M as Merchant
  participant G as Gateway
  participant C as Core
  participant K as Kafka
  participant L as Ledger

  M->>G: POST /payments
  Note over G: cek JWT, rate limit, idempotency
  G->>C: buat pembayaran
  Note over C: tahan dana, simpan PENDING + outbox
  C-->>G: PENDING
  G-->>M: 202 PENDING
  C->>K: payment.created
  K->>L: payment.created
  Note over L: catat jurnal, debit = kredit
  L->>K: payment.settled
  K->>C: payment.settled
  Note over C: saldo diselesaikan, status SUCCESS
  K->>G: payment.settled
  G-->>M: webhook sukses
```

---

## 6. Jalur keputusan request pembayaran

Pemeriksaan diurutkan dari yang paling murah ke yang paling mahal, sehingga request yang salah ditolak sedini mungkin tanpa membebani core.

```mermaid
flowchart TD
  A["POST /payments"] --> B{"Token valid?<br/>gateway"}
  B -- tidak --> B1["401 ditolak"]
  B -- ya --> C{"Dalam batas rate limit?<br/>gateway"}
  C -- tidak --> C1["429 terlalu banyak"]
  C -- ya --> D{"Idempotency key baru?<br/>gateway Redis + core"}
  D -- tidak --> D1["Hasil lama (isi sama)<br/>atau 422 (isi beda)"]
  D -- ya --> E{"Saldo tersedia cukup?<br/>core"}
  E -- tidak --> E1["FAILED<br/>INSUFFICIENT_FUNDS"]
  E -- ya --> F["Simpan PENDING<br/>tahan dana + tulis outbox"]
```

---

## 7. Siklus status pembayaran

PENDING dapat berakhir di tiga status, dan ketiganya final.

```mermaid
stateDiagram-v2
  [*] --> PENDING: dana ditahan
  PENDING --> SUCCESS: payment.settled
  PENDING --> FAILED: payment.failed
  PENDING --> EXPIRED: timeout
  SUCCESS --> [*]
  FAILED --> [*]
  EXPIRED --> [*]
```

Efek tiap transisi pada kolom `accounts`:

| Transisi | Akun pembeli | Akun merchant dan fee | Event |
|---|---|---|---|
| Dibuat (PENDING) | `held_amount` naik | tidak berubah | `payment.created` |
| SUCCESS | `balance` dan `held_amount` turun sebesar `amount` | `balance` naik (merchant: `amount - fee`, akun fee: `fee`) | `payment.settled` |
| FAILED | `held_amount` turun (dana dilepas) | tidak berubah | `payment.failed` |
| EXPIRED | `held_amount` turun (dana dilepas) | tidak berubah | `payment.expired` (belum ada di kontrak) |

### Hal yang masih terbuka

- Job yang menandai PENDING lama menjadi EXPIRED belum didesain. Indeks `idx_payments_pending` di migrasi sudah disiapkan untuk itu.
- Event `payment.expired` perlu ditambahkan ke kontrak di `docs/events/`. Ledger tidak perlu mencatat apa pun untuk status ini karena dana tidak pernah berpindah.