# GitHub Project: Rombay

Satu GitHub Project (Projects v2) melacak pekerjaan di keempat repo (`rombay`, `rombay-core`, `rombay-gateway`, `rombay-ledger`). Project berada di tingkat akun, bukan di dalam satu repo, sehingga issue dari repo mana pun bisa muncul di papan yang sama.

## 1. Membuat project

1. Buka profil GitHub Anda, tab **Projects**, lalu **New project**.
2. Pilih template **Board** (atau mulai dari kosong), beri nama `Rombay`.
3. Di pengaturan project, tautkan keempat repo (**Settings → Manage access / Linked repositories**) agar project muncul di tab Projects tiap repo.

## 2. Kolom status

| Status | Arti |
|---|---|
| Backlog | Ide atau tugas yang belum dijadwalkan |
| Ready | Sudah jelas dan siap dikerjakan |
| In progress | Sedang dikerjakan (batasi 1-2 item sekaligus) |
| In review | PR terbuka atau sedang diuji |
| Done | Selesai dan sudah di-merge |

## 3. Field khusus

Tambahkan field berikut lewat **Settings → Fields**:

| Field | Tipe | Nilai |
|---|---|---|
| Service | Single select | core, gateway, ledger, infra, docs |
| Fase | Single select | Fase 1, Fase 2, Fase 3, Fase 4 |
| Prioritas | Single select | P0, P1, P2 |
| Ukuran | Single select | S, M, L |
| Target | Date | Tanggal target selesai (untuk tampilan Roadmap) |

## 4. Tampilan (views)

| View | Layout | Pengaturan |
|---|---|---|
| Papan | Board | Dikelompokkan per Status |
| Per service | Table | Dikelompokkan per Service, diurutkan per Prioritas |
| Roadmap | Roadmap | Memakai field Target, dikelompokkan per Fase |
| Sedang berjalan | Table | Filter: `status:"In progress","In review"` |

## 5. Otomatisasi

Di **Settings → Workflows**:

- **Auto-add to project:** aktifkan untuk tiap repo dengan filter `is:issue,pr`, sehingga issue dan PR baru langsung masuk papan.
- **Item closed:** ubah Status menjadi Done.
- **Pull request merged:** ubah Status menjadi Done.
- **Item reopened:** kembalikan Status ke In progress.

## 6. Label seragam di keempat repo

| Label | Kegunaan |
|---|---|
| `feature`, `bug`, `chore`, `docs` | Jenis pekerjaan |
| `svc:core`, `svc:gateway`, `svc:ledger`, `svc:infra` | Service terkait |
| `contract` | Perubahan pada API atau skema event (wajib ditinjau lintas repo) |
| `good-first-step` | Tugas kecil untuk pemanasan |

## 7. Backlog awal

Buat issue berikut di repo yang sesuai, lalu isi field Service dan Fase.

### Fase 1: rombay-core

- [ ] Inisialisasi project Spring Boot dan Dockerfile multi-stage
- [ ] Migrasi Flyway `V1__init.sql` (accounts, merchants, payments, outbox)
- [ ] Endpoint internal pembuatan pembayaran dengan penahanan dana atomik
- [ ] Idempotency di level database (`merchant_id` + `idempotency_key`)
- [ ] Outbox relay ke Kafka
- [ ] Riwayat status pembayaran
- [ ] Integration test dengan Testcontainers (PostgreSQL, Kafka)

### Fase 2: rombay-gateway

- [ ] Kerangka HTTP server, `/healthz`, `/metrics`
- [ ] Middleware JWT
- [ ] Rate limiter berbasis Redis
- [ ] Idempotency di Redis dan deteksi key sama dengan isi berbeda
- [ ] Reverse proxy ke core
- [ ] Webhook dispatcher: worker pool, HMAC signature, exponential backoff
- [ ] Dead-letter queue untuk webhook gagal
- [ ] Consumer event status dari Kafka

### Fase 3: rombay-ledger

- [ ] Inisialisasi crate Rust dan Dockerfile
- [ ] Skema ledger (jurnal double-entry, append-only)
- [ ] Consumer `payment.created` dengan penanganan idempoten
- [ ] Validasi debit sama dengan kredit sebelum menulis
- [ ] Penerbitan `payment.settled` dan `payment.failed`
- [ ] CLI rekonsiliasi (`recon`): parse CSV settlement bank dan cocokkan
- [ ] Laporan selisih rekonsiliasi

### Fase 4: lintas service (`rombay`)

- [ ] OpenAPI gateway di `docs/openapi.yaml`
- [ ] Skema event di `docs/events/`
- [ ] Dashboard Grafana dan alert dasar
- [ ] Distributed tracing (OpenTelemetry)
- [ ] Load test k6 untuk `POST /payments`
- [ ] Skenario end-to-end: sukses, request ganda, saldo kurang, merchant down

## 8. Deskripsi singkat dan topik tiap repo

Isi kolom **About** di halaman utama masing-masing repo agar mudah ditemukan:

| Repo | Deskripsi (About) | Topics |
|---|---|---|
| `rombay` | Hub project Rombay: docker-compose, kontrak API/event, dan dokumentasi core payment platform | `payments`, `fintech`, `microservices`, `docker-compose`, `event-driven` |
| `rombay-gateway` | API gateway Rombay (Go): JWT, rate limit, idempotency, webhook dispatcher | `golang`, `api-gateway`, `webhook`, `redis`, `fintech` |
| `rombay-core` | Core transaksi Rombay (Spring Boot): akun, pembayaran, outbox event | `spring-boot`, `java`, `payments`, `kafka`, `postgresql` |
| `rombay-ledger` | Ledger double-entry dan rekonsiliasi Rombay (Rust) | `rust`, `ledger`, `double-entry`, `reconciliation`, `fintech` |

Sematkan (pin) keempat repo di profil GitHub Anda dan tautkan GitHub Project-nya dari README repo `rombay`.