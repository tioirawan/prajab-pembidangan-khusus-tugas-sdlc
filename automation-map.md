# Automation Map — AiRadar

| Field | Value |
| --- | --- |
| Dokumen induk | `project-charter.md` v3.0 |
| Versi | 3.0 |
| Tanggal | 2026-08-28 |
| Arsitektur | Browser SPA → AiRadar BFF (Node.js) → OpenSky Network |
| Perubahan dari v2.0 | Penambahan gate backend, database, CI, dan abstraksi provider. Lihat bagian 9. |

---

## 1. Prinsip

1. **Setiap fase memiliki generator dan validator.** Tidak ada fase yang hanya mengandalkan
   penilaian manusia, dan tidak ada fase yang hanya mengandalkan output AI.
2. **Validator bersifat deterministik.** Menjalankan validator dua kali pada input yang sama
   menghasilkan keputusan yang sama. Penilaian LLM digunakan sebagai *reviewer*, bukan *gate*.
3. **Artefak hulu adalah input bagi artefak hilir.** Jika artefak hulu berubah, artefak hilir
   diregenerasi atau disinkronkan pada perubahan yang sama.
4. **Kegagalan gate memperbaiki sumber, bukan gejala.**
5. **Fixture berasal dari respons nyata.** Bentuk respons OpenSky adalah array posisional yang
   tidak self-describing; AI tidak dapat menebak urutan indeks dengan andal.
6. **Seluruh otomasi berjalan tanpa kredensial dan tanpa memakai credit.** Ini bukan efek
   samping melainkan syarat: CI tidak boleh memerlukan rahasia, dan test tidak boleh
   bergantung pada layanan pihak ketiga yang dapat berubah atau kehabisan kuota.

---

## 2. Peta rantai artefak

```
Project Brief
   │  [Claude]                          validator: review manusia + markdownlint
   ▼
Project Charter ─────────────────────────────────────────────────┐
   │  ▲                                                           │
   │  └── Verifikasi API nyata: CORS, kepadatan, tuple, credit    │
   │      (Fase 0 tidak ditutup tanpa bukti ini)                  │
   │  [Claude sebagai requirements engineer]                      │
   ▼                                    validator: audit ID       │
Specification (REQ-nnn, BR-nnn)                                   │
   │  [Claude]                          validator: gherkin parser │
   ▼                                                              │
Gherkin .feature                                                  │
   │                                                              │
   ├────────────┬──────────────────┬──────────────┐               │
   ▼            ▼                  ▼              ▼               │
UI Contract  OpenAPI internal   Data Model   Provider Interface   │
UI-nnn       API-nnn            DATA-nnn     (ADR-003)            │
   │            │                  │              │               │
 axe-core   redocly lint      migrasi SQLite  Zod tuple           │
 Storybook  Prism mock        dari kosong     ← fixture nyata     │
   │            │                  │              │               │
   └────────────┴────────┬─────────┴──────────────┘               │
                         │  [Claude]   validator: trace:audit     │
                         ▼                                        │
              Implementation Plan (TASK-nnn)                      │
                         │  [Claude Code]                         │
                         ▼             validator: tsc/lint/build  │
                  Source Code (server + web)                      │
                         │  [Claude + Playwright codegen]         │
                         ▼                                        │
                  Test Suite (TST-nnn)                            │
                         │  validator: vitest/playwright/axe/stryker
                         ▼                                        │
              GitHub Actions CI quality gate                      │
                         │                                        │
                         ▼                                        │
              Traceability Matrix ◄────────────────────────────────┘
                         │  validator: trace:audit + audit akhir
                         ▼
                 final-review.md (READY / NOT READY)
```

---

## 3. Matriks otomasi per fase

| Fase | Input | Generator | Output kandidat | Validator deterministik | Reviewer (LLM) | Gate |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | Project brief | Claude | Charter, automation map, repository instructions, DoD | `markdownlint`; checklist; **panggilan API nyata** untuk membuktikan CORS, kepadatan data, bentuk tuple, dan biaya credit | Audit charter terhadap brief (invented scope) | Open question = 0; tool coverage = 100%; setiap keputusan arsitektur punya bukti |
| 1 | Charter | Claude (requirements engineer) | `specification.md` | Skrip audit ID (unik, berurutan, terpakai); `markdownlint` | Reviewer agent sesi bersih: ambiguity, contradiction, unverifiable, invented scope — khususnya requirement yang mengasumsikan data bandara atau maskapai (OOS-02) | Completeness ≥ 90%; testability 100%; ambiguity tinggi = 0; scope integrity = 0 |
| 2 | Specification | Claude | `features/*.feature` | `@cucumber/gherkin` parser; `gherkin-lint`; BDD dry-run | Reviewer agent: vague Then, multi-behaviour scenario, missing negative case | REQ/BR coverage 100%; parse validity 100%; negative coverage 100% |
| 3 | Spec + Gherkin | Claude + Storybook | UI contract, state matrix, prototype | `axe-core`; interaction test; pemetaan state ↔ Gherkin | Reviewer agent membandingkan prototype dengan spec, bukan estetika | Journey coverage 100%; state coverage 100%; a11y critical/serious = 0 |
| 4 | Spec + Gherkin + UI contract + **fixture nyata** | Claude | OpenAPI internal, ADR-001..005, data model, provider interface | `redocly lint`; Prism mock; **skema Zod tuple diuji terhadap 29 record nyata**; migrasi SQLite dari database kosong; uji seed | Reviewer agent: setiap aksi UI punya jalur data; setiap BR punya titik penegakan; istilah penyedia tidak bocor ke kontrak internal | Contract validity 0 error; action coverage 100%; example coverage 100%; migrasi berhasil; kontrak internal bebas istilah penyedia |
| 5 | Artefak Fase 1–4 | Claude | `implementation-plan.md`, `tasks.md` | `npm run trace:audit` | Reviewer agent: task terlalu besar, acceptance criteria hilang | Task coverage 100%; orphan = 0; cycle = 0 |
| 6 | TASK + artefak sumber | Claude Code | Pull request | `tsc --noEmit`; ESLint (termasuk aturan batas impor); Prettier; build; test terkait | Reviewer agent membaca diff terhadap TASK dan REQ | Build pass; lint/type error = 0; traceability 100%; komentar severity tinggi terbuka = 0 |
| 7 | OpenAPI + data model + Gherkin | Claude | Contract & integration test | Vitest + MSW; validasi respons BFF terhadap OpenAPI; uji migrasi dari database kosong; uji ledger anggaran; uji isolasi urutan acak | Reviewer agent: kasus kegagalan yang belum tercakup | Contract conformance 100%; data rule coverage 100%; migrasi pass; ledger pass |
| 8 | Seluruh artefak + kode | Claude + Playwright codegen | Test berlapis | Vitest, Testing Library, Playwright, `axe-core`, Stryker, coverage; suite dijalankan 2× | Reviewer agent: assertion tautologis, test tanpa ID sumber | Acceptance coverage 100%; pass rate 100%; flaky 0%; coverage ≥ 80%; mutation score sesuai target |
| 9 | Repository | GitHub Actions | Pipeline run | Job berurutan; fault injection pada pull request nyata | — | Pipeline hijau pada kode benar; merah pada FI-01..FI-05; artefak laporan tersedia; durasi diukur |

---

## 4. Pemetaan tool coverage

| Fase | Punya generator? | Punya validator deterministik? | Status |
| --- | --- | --- | --- |
| 0 | Ya — Claude | Ya — markdownlint + checklist + panggilan API nyata | Terpenuhi |
| 1 | Ya — Claude | Ya — audit ID + markdownlint | Terpenuhi |
| 2 | Ya — Claude | Ya — gherkin parser + lint + dry-run | Terpenuhi |
| 3 | Ya — Claude + Storybook | Ya — axe-core + interaction test | Terpenuhi |
| 4 | Ya — Claude | Ya — redocly + Prism + Zod tuple vs fixture nyata + migrasi SQLite | Terpenuhi |
| 5 | Ya — Claude | Ya — trace:audit | Terpenuhi |
| 6 | Ya — Claude Code | Ya — tsc + eslint + build | Terpenuhi |
| 7 | Ya — Claude | Ya — vitest + MSW + validasi OpenAPI + uji migrasi | Terpenuhi |
| 8 | Ya — Claude + codegen | Ya — vitest/playwright/axe/stryker | Terpenuhi |
| 9 | Ya — GitHub Actions | Ya — fault injection | Terpenuhi |

**Tool coverage = 10/10 = 100%.**

---

## 5. Otomasi khusus arsitektur

### 5.1 Menegakkan batas abstraksi provider (ADR-003)

Abstraksi yang hanya ditulis di dokumen akan bocor. Karena itu batasnya ditegakkan secara
otomatis, bukan lewat kesepakatan.

| Mekanisme | Fungsi |
| --- | --- |
| Aturan lint batas impor | Modul di luar `server/src/providers/opensky/` dilarang mengimpor apa pun dari dalamnya. Pelanggaran menggagalkan `npm run lint`. |
| Skema tuple terbatas pada adapter | Tipe tuple posisional tidak diekspor keluar direktori adapter. |
| Uji pergantian provider | Test integrasi menjalankan skenario yang sama dengan `FixtureProvider` dan `OpenSkyProvider` bertopeng, dan memeriksa keluaran domain identik. |
| Pemindaian istilah pada kontrak internal | Skrip memeriksa `openapi/airadar.yaml` tidak memuat istilah khas penyedia (`icao24`, `baro_altitude`, `true_track`, `states`, `opensky`). |
| `FixtureProvider` sebagai bukti hidup | Provider kedua yang berfungsi penuh. Jika abstraksi bocor, provider ini berhenti bekerja — kegagalannya langsung terlihat. |

### 5.2 Validasi tuple posisional

`/states/all` mengembalikan array dua dimensi dengan makna ditentukan oleh posisi indeks.
Verifikasi Fase 0 menunjukkan tuple berisi **17 elemen**.

| Mekanisme | Fungsi |
| --- | --- |
| Skema Zod tuple | Menetapkan panjang, tipe, dan nullability tiap indeks sebagai kontrak yang dieksekusi. |
| Pemetaan tuple → objek domain bernama | Indeks mentah tidak pernah bocor keluar dari adapter. |
| Contract test per indeks | Setiap indeks memiliki assertion tersendiri, sehingga pergeseran satu posisi terdeteksi spesifik. |
| Fixture 29 record nyata | Memuat seluruh pola null yang teramati: callsign string kosong ×2, `baro_altitude` null ×2, `velocity` null ×1, `vertical_rate` null ×2, `sensors` null ×29, `geo_altitude` null ×2, `squawk` null ×5. |

### 5.3 Disiplin anggaran credit

| Mekanisme | Fungsi |
| --- | --- |
| `FixtureProvider` sebagai default pengembangan, test, dan CI | Nol credit terpakai di luar demo. |
| Cache snapshot TTL 30 detik di server | Satu snapshot melayani seluruh klien; biaya sebanding waktu pemakaian, bukan jumlah pengguna. |
| Ledger persisten di SQLite | Pagu harian bertahan melintasi restart server. Tanpa ini, restart mengembalikan hitungan ke nol. |
| Pagu keras 1.600 permintaan/hari | 3.200 dari 4.000 credit, menyisakan 800 credit cadangan. |
| Batas interval minimum pada refresh manual | Mencegah penekanan tombol berulang menghabiskan anggaran. |
| Pembacaan `X-Rate-Limit-Remaining` | Merekonsiliasi hitungan internal dengan hitungan penyedia. |
| Uji otomatis konsumsi nol | Test gagal jika ada panggilan jaringan keluar selama suite berjalan. |

### 5.4 Perlindungan kredensial

| Mekanisme | Fungsi |
| --- | --- |
| Kredensial dibaca di satu modul saja | Mempersempit permukaan kebocoran. |
| Pemetaan ulang error upstream | Pesan error penyedia tidak diteruskan mentah ke klien. |
| Uji pemindaian respons dan log | Test gagal jika substring kredensial atau token muncul di respons HTTP atau keluaran log. |
| CI tanpa secret | Seluruh job berjalan dengan `FixtureProvider`; tidak ada secret yang perlu dikonfigurasi. |

---

## 6. Pipeline CI — GitHub Actions

Dijalankan pada setiap pull request dan setiap push ke `main`. Seluruh job berjalan dengan
`FixtureProvider` dan tidak memerlukan satu secret pun.

| Urutan | Job | Isi | Gagal berarti |
| --- | --- | --- | --- |
| 1 | `setup` | Checkout, setup Node 20, cache dependensi, `npm ci` | Dependensi tidak dapat direproduksi |
| 2 | `static` | `lint`, `typecheck`, `openapi:lint`, `bdd` | Kode atau kontrak tidak memenuhi standar |
| 3 | `build` | Build server dan web | Artefak tidak dapat dibangun |
| 4 | `test-fast` | `test:unit`, `test:component`, `test:contract` | Logika atau kontrak rusak |
| 5 | `test-integration` | Migrasi database dari kosong, seed, `test:integration` | Integrasi atau persistensi rusak |
| 6 | `test-e2e` | Jalankan BFF + web, `test:e2e`, `test:a11y` | Alur pengguna atau aksesibilitas rusak |
| 7 | `audit` | `trace:audit` | Traceability tidak lengkap |
| 8 | `report` | Unggah coverage, laporan Playwright, laporan a11y sebagai artefak | — |

Job 2 sampai 6 diparalelkan sejauh dependensinya memungkinkan. Mutation testing **tidak**
dijalankan di CI karena lambat; ia dijalankan lokal sebelum Fase 8 ditutup, dan laporannya
disimpan pada `reports/`.

`main` dilindungi: merge hanya diizinkan ketika seluruh required check hijau.

### 6.1 Fault injection

Efektivitas gate dibuktikan dengan sengaja memasukkan kesalahan melalui pull request nyata,
sehingga bukti berupa run CI yang merah — bukan klaim.

| Injeksi | Perubahan yang sengaja salah | Job yang harus merah | Membuktikan |
| --- | --- | --- | --- |
| FI-01 | Variabel tidak terpakai / pelanggaran lint | `static` | Gate gaya kode bekerja |
| FI-02 | Membalik operator pada filter radius (`<=` menjadi `>=`) | `test-fast` | Unit test business rule bermakna |
| FI-03 | **Menukar indeks 5 dan 6 pada mapper (longitude ↔ latitude)** | `test-fast` | Contract test per indeks bekerja — kesalahan ini lolos type-check karena kedua field bertipe sama |
| FI-04 | Menghapus penegakan pagu anggaran harian | `test-integration` | Budget guard benar-benar diuji |
| FI-05 | **Mengimpor tipe tuple OpenSky langsung dari lapisan domain** | `static` | Batas abstraksi ADR-003 ditegakkan mesin, bukan sekadar kesepakatan |

Jika salah satu injeksi tetap menghasilkan CI hijau, itu adalah **false green**, dan test yang
bersangkutan harus diperkuat sebelum fase dinyatakan selesai.

---

## 7. Strategi evidence

Setiap fase menghasilkan satu baris pada `docs/automation-evidence.md`, diisi saat fase berjalan.

| Fase | Input | Tool / otomasi | Output | Validasi | Koreksi manusia | Status |
| --- | --- | --- | --- | --- | --- | --- |
| | | | | | | Lulus / Ulangi |

Bukti yang wajib disimpan:

1. **Prompt yang diberikan** — persis seperti dikirim, termasuk artefak yang ditempelkan.
2. **Output mentah AI** — sebelum koreksi manusia.
3. **Hasil validator** — log atau laporan, bukan ringkasan naratif.
4. **Diff koreksi manusia** — apa yang diubah dan mengapa.
5. **Output yang ditolak** — kandidat yang dibuang beserta alasannya.
6. **Tautan run CI** — termasuk run yang merah pada fault injection.
7. **Catatan setiap panggilan API nyata** — waktu, tujuan, credit terpakai, sisa anggaran.

Butir 5 bernilai tinggi pada rubrik. Assignment secara eksplisit menyatakan jumlah koreksi
manusia **dilaporkan, bukan harus nol**.

### 7.1 Evidence Fase 0 yang sudah terkumpul

| Bukti | Isi | Menutup |
| --- | --- | --- |
| Log console browser | Penolakan CORS beserta nilai header `Access-Control-Allow-Origin` | OQ-001 → ADR-001 |
| Respons nyata 29 state vector | Kepadatan data, bentuk tuple 17 elemen, pola null lengkap | OQ-006, ASM-07 |
| Perhitungan anggaran | Interval 30 detik tanpa cache melampaui kuota 44% | OQ-005 → ADR-004 |
| Analisis jarak | 29 dari 29 di dalam radius; terdekat 37,9 km; terjauh 479,6 km | Dasar rancangan empty state |
| Log kegagalan Node | `ENOTFOUND opensky-network.org` | Membuka RISK-01 dan OQ-101 |

---

## 8. Riwayat Perubahan

| Versi | Tanggal | Perubahan |
| --- | --- | --- |
| 1.0 | 2026-08-28 | Automation map awal berbasis aviationstack. |
| 2.0 | 2026-08-28 | Disesuaikan untuk OpenSky: otomasi validasi tuple posisional, disiplin credit, verifikasi cakupan geografis. |
| 3.0 | 2026-08-28 | Disesuaikan untuk arsitektur BFF: (a) bagian 5.1 penegakan otomatis batas abstraksi provider, termasuk aturan lint batas impor dan pemindaian istilah pada kontrak internal; (b) bagian 5.4 perlindungan kredensial dengan uji pemindaian respons dan log; (c) bagian 6 pipeline GitHub Actions delapan job tanpa secret; (d) FI-04 dan FI-05 ditambahkan pada fault injection; (e) validator Fase 4 diperluas dengan migrasi SQLite dan pemeriksaan kebocoran istilah penyedia; (f) validator Fase 7 diperluas dengan uji ledger anggaran; (g) bagian 7.1 mencatat evidence Fase 0 yang sudah terkumpul beserta open question yang ditutupnya. |
