# Project Charter — AiRadar

| Field | Value |
| --- | --- |
| Nama proyek | AiRadar |
| Versi dokumen | 3.0 |
| Tanggal | 2026-08-28 |
| Fase SDLC | Fase 0 — Project Charter & Automation Map |
| Status | Draft — menunggu penutupan Open Question OQ-101..OQ-104 |
| Arsitektur | Browser SPA → AiRadar BFF (Node.js) → OpenSky Network |
| Sumber data eksternal | OpenSky Network REST API — `GET /states/all` |
| Mode akses | OAuth2 client credentials (terautentikasi) |
| Titik acuan geospasial | PT PLN (Persero) UPDL Bogor — `-6.6564, 106.8760` |
| Perubahan dari v2.0 | Penambahan backend, OAuth2, CI, dan lapisan abstraksi provider. Lihat bagian 8. |

---

## 1. Problem Statement, Actor, Outcome, dan Scope

### 1.1 Problem Statement

UPDL Bogor berada di bawah salah satu koridor udara tersibuk di Indonesia. Dalam radius 500 km
terdapat Soekarno-Hatta, Halim Perdanakusuma, Husein Sastranegara, Kertajati, serta jalur lintas
Jawa–Sumatera. Meskipun demikian, tidak ada satu pun tampilan yang menjawab pertanyaan sederhana
berikut secara langsung:

> "Pesawat apa saja yang sedang berada di udara di sekitar kita saat ini, seberapa jauh, setinggi
> apa, dan menuju arah mana?"

Layanan pelacakan penerbangan yang tersedia bersifat global dan berpusat pada peta dunia. Pengguna
harus menafsirkan sendiri tampilan global untuk memperoleh jawaban yang sebenarnya bersifat lokal.
Tidak ada konsep "di sekitar saya" dengan batas yang tegas dan titik acuan yang tetap.

**AiRadar** menyelesaikan masalah itu dengan satu tampilan web yang memusatkan seluruh data
penerbangan langsung pada satu titik acuan tetap — UPDL Bogor — dengan batas radius 500 km yang
dihitung secara eksplisit dan ditampilkan secara visual.

> **Catatan cakupan:** proyek ini memiliki **satu fitur fungsional**. Seluruh nilai tambah
> diarahkan pada kualitas eksekusi — kelengkapan state, ketahanan terhadap kegagalan API,
> kualitas pengalaman pengguna, dan kekuatan bukti proses — bukan pada penambahan fitur.

### 1.2 Bukti verifikasi teknis (Fase 0)

Seluruh keputusan arsitektur pada charter ini bertumpu pada pengujian nyata, bukan asumsi. Bukti
disimpan pada `docs/automation-evidence.md` dan `data/fixtures/`.

**Temuan 1 — CORS memblokir akses langsung dari browser.**

Permintaan `fetch` dari origin browser ke `https://opensky-network.org/api/states/all` ditolak.
Server mengembalikan `Access-Control-Allow-Origin: https://opensky-network.org`, yaitu origin
miliknya sendiri, bukan wildcard. Tidak ada origin aplikasi mana pun yang dapat lolos.

Konsekuensi: **arsitektur frontend-only tidak dapat diwujudkan.** Backend bukan pilihan gaya
melainkan keharusan teknis. Ini diformalkan sebagai ADR-001.

**Temuan 2 — Kepadatan data memadai.**

Satu permintaan bounding box mengembalikan 29 state vector. Seluruh 29-nya berada di dalam radius
500 km; tidak ada satu pun yang jatuh di sudut kotak di luar lingkaran.

| Pengukuran | Hasil |
| --- | --- |
| State vector dikembalikan | 29 |
| Di dalam radius 500 km | 29 |
| Di luar radius (sudut bounding box) | 0 |
| Jarak terdekat | 37,9 km |
| Jarak terjauh di dalam radius | 479,6 km |
| Negara registrasi | Indonesia 26, Filipina 1, Singapura 1, Arab Saudi 1 |
| `on_ground = true` | 2 |

**Temuan 3 — Bentuk data dan pola null yang sebenarnya.**

Tuple berisi **17 elemen**, bukan 18. Indeks 17 (`category`) hanya muncul bila parameter
`extended=1` dikirim. Pola null pada 29 record nyata:

| Indeks | Field | Temuan |
| --- | --- | --- |
| 1 | `callsign` | **2 record berupa string kosong `""`, bukan `null`** |
| 7 | `baro_altitude` | null ×2 |
| 9 | `velocity` | null ×1 |
| 11 | `vertical_rate` | null ×2 |
| 12 | `sensors` | null ×29 (seluruhnya) |
| 13 | `geo_altitude` | null ×2 |
| 14 | `squawk` | null ×5 |

Callsign berupa string kosong adalah temuan yang berkonsekuensi langsung: pemeriksaan `null` saja
tidak cukup. Normalisasi wajib memangkas spasi dan memperlakukan hasil kosong sebagai tidak
diketahui. Ini menjadi DATA rule di Fase 4.

**Temuan 4 — Anggaran credit tidak mendukung penyegaran 30 detik tanpa cache.**

Tier terautentikasi memberi 4.000 credit/hari pada bucket `/states/*`. Bounding box kami berluas
81,43 derajat persegi, masuk kelas tarif 25–100 sq°, sehingga berbiaya **2 credit** per permintaan.

| Interval | Permintaan/hari | Credit/hari | Status |
| --- | --- | --- | --- |
| 15 detik | 5.760 | 11.520 | Melampaui kuota |
| **30 detik** | **2.880** | **5.760** | **Melampaui kuota 44%** |
| 45 detik | 1.920 | 3.840 | Masuk |
| 60 detik | 1.440 | 2.880 | Masuk |

Konsekuensi: penyegaran 30 detik hanya layak bila snapshot di-cache di sisi server dan dibagikan
ke seluruh klien, sehingga biaya menjadi proporsional terhadap **waktu pemakaian**, bukan jumlah
pengguna. Ini diformalkan sebagai ADR-004 dan BR anggaran di Fase 1.

### 1.3 Prinsip arsitektur: abstraksi penyedia data

Penyedia data telah berganti satu kali sebelum implementasi dimulai. Perubahan itu murah karena
belum ada kode. Perubahan berikutnya tidak akan semurah itu.

Karena itu ditetapkan prinsip berikut, yang diformalkan sebagai **ADR-003**:

> Seluruh detail teknis penyedia data — endpoint, autentikasi, bentuk tuple posisional, satuan SI,
> tarif credit — terisolasi di dalam satu lapisan adapter. Model domain, kontrak API internal,
> dan frontend tidak boleh mengetahui bahwa penyedia datanya bernama OpenSky.

Konsekuensi yang mengikat:

1. Tuple posisional tidak pernah keluar dari `server/src/providers/opensky/`.
2. Model domain menggunakan field bernama dengan satuan tampilan yang eksplisit.
3. Kontrak API internal (`openapi/airadar.yaml`) tidak memuat satu pun istilah khas OpenSky.
4. Frontend tidak pernah mengetahui nama penyedia, kecuali pada satu baris atribusi.
5. Terdapat provider kedua yang berfungsi penuh — `FixtureProvider` — yang membaca respons nyata
   tersimpan. Keberadaannya membuktikan abstraksi bekerja, dan sekaligus menjadi mode default
   untuk pengembangan dan seluruh test.

Butir 5 adalah bukti yang dapat didemonstrasikan: mengganti provider dilakukan lewat satu variabel
lingkungan, tanpa mengubah satu baris pun di frontend atau domain.

### 1.4 Actor

| ID | Actor | Tipe | Deskripsi | Kepentingan |
| --- | --- | --- | --- | --- |
| ACT-01 | Pengamat Penerbangan | Primary, manusia | Peserta diklat, pegawai, atau pengunjung UPDL Bogor yang ingin melihat lalu lintas udara di sekitar lokasi. | Mendapat jawaban dalam hitungan detik tanpa registrasi dan tanpa konfigurasi. |
| ACT-02 | Evaluator / Pemateri | Secondary, manusia | Menilai artefak SDLC, traceability, dan hasil quality gate. | Dapat menelusuri satu requirement sampai ke test dan bukti eksekusinya. |
| ACT-03 | AiRadar BFF | Internal system | Backend milik tim. Menyimpan kredensial, mengelola token, menegakkan anggaran, menyajikan kontrak internal. | Menjadi satu-satunya pihak yang berbicara dengan penyedia data. |
| ACT-04 | OpenSky Network API | External system | Penyedia state vector berbasis jaringan receiver ADS-B sukarelawan. | Anggaran credit, masa berlaku token, dan bentuk respons harus dihormati. |
| ACT-05 | Penyedia peta dasar | External system | Penyedia tile peta. | Atribusi wajib ditampilkan. |

### 1.5 Desired Outcome (terukur)

| ID | Outcome | Ukuran keberhasilan |
| --- | --- | --- |
| OUT-01 | Pengguna memperoleh gambaran lalu lintas udara di sekitar UPDL Bogor tanpa langkah konfigurasi apa pun. | Halaman menampilkan hasil (data, empty state, atau error state yang informatif) dalam ≤ 3 detik, tanpa input pengguna. |
| OUT-02 | Pengguna dapat membedakan pesawat yang relevan dari yang tidak. | Setiap pesawat menyertakan jarak dari UPDL Bogor dalam km; daftar terurut menaik berdasarkan jarak. |
| OUT-03 | Pengguna tidak pernah melihat layar yang ambigu. | 100% state terdefinisi memiliki tampilan dan pesan yang dirancang. |
| OUT-04 | Pengguna tahu seberapa mutakhir data yang dilihatnya. | Setiap tampilan menyertakan waktu pengamatan yang berasal dari penyedia data, bukan jam perangkat, dan menandai data basi. |
| OUT-05 | Pengguna dapat memaksa pembaruan ketika membutuhkannya. | Tombol refresh manual tersedia, memberi umpan balik dalam ≤ 1 detik, dan tidak dapat dipakai untuk menghabiskan anggaran. |
| OUT-06 | Anggaran credit tidak pernah habis akibat pemakaian normal. | Konsumsi tercatat, ditegakkan oleh pagu keras di server, dan dapat dibuktikan tidak melampaui batas. |
| OUT-07 | Kegagalan penyedia data tidak membuat aplikasi tidak dapat dipakai. | Ketika upstream gagal, aplikasi menyajikan snapshot terakhir yang diketahui baik beserta penanda usia data. |
| OUT-08 | Pengguna memahami bahwa peta menampilkan cakupan receiver, bukan seluruh penerbangan yang ada. | Aplikasi menyatakan secara eksplisit bahwa data berasal dari jaringan ADS-B sukarelawan. |
| OUT-09 | Penggantian penyedia data tidak menuntut perubahan pada frontend maupun domain. | Dapat didemonstrasikan dengan mengganti satu variabel lingkungan. |

### 1.6 In-Scope

| ID | Item | Keterangan |
| --- | --- | --- |
| IS-01 | Aplikasi web satu halaman (SPA) | Frontend. |
| IS-02 | Backend for Frontend (BFF) berbasis Node.js | Wajib karena CORS (ADR-001). |
| IS-03 | Autentikasi OAuth2 client credentials ke OpenSky, dengan manajemen masa berlaku token | Token kedaluwarsa 30 menit; disegarkan proaktif dengan margin. |
| IS-04 | Lapisan abstraksi provider dengan dua implementasi: `OpenSkyProvider` dan `FixtureProvider` | ADR-003. |
| IS-05 | Kontrak API internal machine-readable (OpenAPI 3.1) yang dimiliki tim | Bersifat provider-neutral. |
| IS-06 | Cache snapshot di server dengan TTL, dibagikan ke seluruh klien | ADR-004. |
| IS-07 | Ledger anggaran credit yang persisten dan pagu keras harian | Bertahan melintasi restart server. |
| IS-08 | Database persisten dengan migration dan seed data | SQLite. Lihat OQ-102. |
| IS-09 | Penyegaran otomatis 30 detik dan tombol refresh manual | Refresh manual dibatasi interval minimum di server. |
| IS-10 | Query bounding box yang diturunkan dari titik acuan dan radius | `lamin=-11.1530`, `lamax=-2.1598`, `lomin=102.3489`, `lomax=111.4031`. |
| IS-11 | Penyaringan presisi radius 500 km dengan haversine | Bounding box adalah penyaring kasar di upstream; haversine memangkas sudut kotak. |
| IS-12 | Visualisasi peta: posisi pesawat, arah hadap, lingkaran radius, penanda titik acuan | Peta interaktif. |
| IS-13 | Daftar pesawat tersinkronisasi dengan peta | Seleksi pada satu sisi menyorot sisi lain. |
| IS-14 | Panel detail pesawat | Callsign, identitas transponder, negara registrasi, ketinggian, kecepatan, arah, vertical rate, status di darat, sumber posisi, jarak, waktu kontak terakhir. |
| IS-15 | Penanganan seluruh state non-happy-path | Loading, kosong, error jaringan, upstream gagal, anggaran habis, refresh dibatasi, data basi, respons tidak sesuai skema. |
| IS-16 | Executable specification (Gherkin) | Fase 2. |
| IS-17 | Automated test berlapis: unit, komponen, kontrak, integrasi, E2E, aksesibilitas, mutation | Fase 7–8. Bobot rubrik tertinggi. |
| IS-18 | Pipeline CI pada GitHub Actions | Fase 9. |
| IS-19 | Aksesibilitas dasar | Navigasi keyboard, label, kontras, focus visible, pengumuman perubahan status. |
| IS-20 | Traceability matrix dan evidence portfolio | Sepanjang proyek. |

### 1.7 Out-of-Scope

| ID | Item | Alasan |
| --- | --- | --- |
| OOS-01 | Autentikasi pengguna aplikasi | Aplikasi bersifat publik dan anonim bagi pengguna akhir. |
| OOS-02 | Bandara asal, bandara tujuan, nama maskapai, nomor penerbangan komersial, status keterlambatan | **Tidak tersedia pada `/states/all`.** Endpoint `/flights/*` hanya berisi data batch semalam sebelumnya. Callsign bukan nomor penerbangan. |
| OOS-03 | Data historis, trajectory, endpoint `/tracks` | `/tracks` berstatus eksperimental menurut dokumentasi resmi. |
| OOS-04 | Notifikasi, langganan pesawat, alert | Penambahan fitur. |
| OOS-05 | Prediksi lintasan atau analitik lalu lintas | Penambahan fitur. |
| OOS-06 | Titik acuan yang dapat dipilih pengguna | Titik acuan tetap pada UPDL Bogor. |
| OOS-07 | Aplikasi mobile native | Cukup responsive web. |
| OOS-08 | Internasionalisasi | Bahasa Indonesia saja. |
| OOS-09 | Deployment otomatis ke lingkungan produksi | CI berhenti pada quality gate. Lihat OQ-103. |
| OOS-10 | Horizontal scaling, load balancing, high availability | Satu instance BFF sudah memadai untuk cakupan ini. |

---

## 2. Asumsi dan Pertanyaan Terbuka

### 2.1 Asumsi

| ID | Asumsi | Dampak jika salah | Cara verifikasi |
| --- | --- | --- | --- |
| ASM-01 | Titik acuan UPDL Bogor adalah `-6.6564, 106.8760`. | Seluruh perhitungan jarak dan bounding box bergeser. | Konfirmasi ke pemateri. |
| ASM-02 | Radius 500 km diukur sebagai jarak great-circle ke proyeksi posisi pesawat di permukaan bumi; ketinggian diabaikan. | Himpunan pesawat yang ditampilkan berbeda. | Business rule eksplisit di Fase 1. |
| ASM-03 | Akun tim berstatus *standard user* dengan anggaran 4.000 credit/hari pada bucket `/states/*`. | Seluruh perhitungan anggaran meleset. | Amati `X-Rate-Limit-Remaining` pada permintaan pertama hari itu. |
| ASM-04 | Bounding box berluas 81,43 sq°, masuk kelas tarif 25–100 sq°, berbiaya 2 credit. | Anggaran meleset dua kali lipat. | Bandingkan `X-Rate-Limit-Remaining` sebelum dan sesudah satu permintaan. |
| ASM-05 | Mesin yang menjalankan BFF dapat meresolusi DNS dan menjangkau `opensky-network.org` serta `auth.opensky-network.org`. | Backend tidak dapat berfungsi. | **Pengujian awal dari Node menghasilkan `ENOTFOUND`.** Wajib diuji ulang dari mesin target sebelum Fase 1 ditutup. |
| ASM-06 | Token OAuth2 berlaku 30 menit dan dapat disegarkan dengan `grant_type=client_credentials`. | Strategi manajemen token perlu dirancang ulang. | Amati field `expires_in` pada respons token. |
| ASM-07 | Tuple berisi 17 elemen selama `extended=1` tidak dikirim. | Skema tuple gagal memvalidasi. | Sudah terverifikasi pada 29 record nyata. |
| ASM-08 | Penyedia tile peta dapat digunakan tanpa API key untuk penggunaan edukasional bervolume rendah, dengan atribusi. | Perlu penyedia alternatif. | Baca ketentuan penggunaan penyedia terpilih. |
| ASM-09 | Kepadatan data pada waktu demo serupa dengan hasil pengamatan (29 pesawat). | Empty state menjadi tampilan yang dominan. | Sampling tambahan pada waktu berbeda, termasuk dini hari. |
| ASM-10 | Tim terdiri dari 1–4 orang dengan durasi kerja 1–2 minggu. | Estimasi Fase 5 tidak valid. | Konfirmasi komposisi tim. |

### 2.2 Pertanyaan Terbuka

> **Quality gate Fase 0 mensyaratkan 0 pertanyaan terbuka material sebelum Fase 1 dimulai.**

| ID | Pertanyaan | Kepada | Mengapa material | Rekomendasi tim | Status |
| --- | --- | --- | --- | --- | --- |
| OQ-101 | Mengapa `node check.js` menghasilkan `ENOTFOUND opensky-network.org` sementara browser berhasil? | Verifikasi teknis | BFF berjalan di Node. Jika Node tidak dapat meresolusi host tersebut pada mesin target, backend tidak akan berfungsi sama sekali. | Uji `curl` dan `node` dari mesin yang akan menjalankan BFF. Kemungkinan penyebab: DNS resolver lokal, proxy korporat, atau IPv6. Bukan penolakan dari OpenSky, karena browser di mesin yang sama berhasil. | OPEN |
| OQ-102 | Apakah SQLite diterima sebagai pemenuhan syarat "database persisten dengan migration dan seed data"? | Pemateri | Menentukan bentuk lapisan persistensi. | **Ya, gunakan SQLite.** Kebutuhannya nyata, bukan dibuat-buat: ledger anggaran credit harus bertahan melintasi restart server (jika tidak, restart mengembalikan pagu ke nol dan anggaran dapat terlampaui), dan snapshot terakhir yang diketahui baik harus tersedia saat upstream gagal (OUT-07). Migration runner sederhana dan seed dari fixture nyata. | OPEN |
| OQ-103 | Sejauh mana pipeline GitHub Actions dijalankan? | Tim + pemateri | Menentukan cakupan Fase 9 dan bukti quality gate. | Jalankan lint, typecheck, build, unit, component, contract, integration, dan E2E pada setiap pull request. Mutation testing dijalankan lokal karena lambat. **Tanpa deployment otomatis** (OOS-09). Seluruh job berjalan dengan `FixtureProvider`, sehingga CI tidak memerlukan kredensial dan tidak memakai satu credit pun. | OPEN |
| OQ-104 | Di mana BFF dijalankan saat demo? | Tim | Menentukan langkah demo dan kebutuhan jaringan. | Jalankan lokal (`npm run dev` menjalankan BFF dan frontend bersamaan). Hindari ketergantungan pada layanan hosting saat penilaian. | OPEN |

**Pertanyaan terbuka yang sudah ditutup pada v3.0:**

| ID lama | Pertanyaan | Keputusan | Bukti |
| --- | --- | --- | --- |
| OQ-001 | Apakah CORS mengizinkan akses langsung dari browser? | **Tidak.** `Access-Control-Allow-Origin` bernilai origin milik OpenSky sendiri. Backend menjadi keharusan. | Log console browser, terlampir pada evidence. |
| OQ-002 | Apakah peniadaan CI menurunkan nilai maksimum menjadi 75? | Tidak berlaku lagi. CI dikembalikan ke cakupan (IS-18). | Keputusan tim. |
| OQ-003 | Bentuk apa yang diterima sebagai database persisten? | Dilanjutkan sebagai OQ-102 dengan rekomendasi SQLite, kini layak karena ada backend. | — |
| OQ-004 | Mode anonim atau terautentikasi? | **Terautentikasi (OAuth2).** Aman karena kredensial berada di server. Anggaran naik dari 400 ke 4.000 credit/hari. | Akun sudah terdaftar. |
| OQ-005 | Interval penyegaran dan pagu credit? | **30 detik dengan cache server**, pagu 1.600 permintaan/hari. Lihat bagian 1.2 Temuan 4. | Perhitungan anggaran. |
| OQ-006 | Bagaimana perilaku pada cakupan ADS-B yang tipis? | Kepadatan terbukti memadai: 29 pesawat, seluruhnya dalam radius. Empty state tetap dirancang untuk kondisi dini hari. | Respons nyata, 29 record. |

---

## 3. Arsitektur Tingkat Tinggi

```
┌──────────────────────────────────────────────────────────────┐
│  Browser — AiRadar SPA                                       │
│  Peta · Daftar · Detail · Tombol refresh · Indikator usia    │
│  Tidak mengetahui nama penyedia data                         │
└───────────────────────────┬──────────────────────────────────┘
                            │  GET  /api/v1/airspace
                            │  POST /api/v1/airspace/refresh
                            │  Kontrak milik tim, provider-neutral
┌───────────────────────────▼──────────────────────────────────┐
│  AiRadar BFF — Node.js                                       │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Lapisan HTTP        validasi permintaan, kontrak       │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ Snapshot cache      TTL 30 detik, dibagi seluruh klien │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ Budget guard        pagu harian, batas refresh manual  │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ Domain              haversine, filter radius, satuan   │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ Provider interface  ◄── batas abstraksi (ADR-003)      │  │
│  │   ├── OpenSkyProvider   OAuth · tuple · Zod · mapper   │  │
│  │   └── FixtureProvider   respons nyata tersimpan        │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ Persistensi         SQLite · migration · seed          │  │
│  │                     ledger credit · snapshot terakhir  │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────┬──────────────────────────────────┘
                            │  Bearer token · bounding box
┌───────────────────────────▼──────────────────────────────────┐
│  OpenSky Network — GET /states/all                           │
└──────────────────────────────────────────────────────────────┘
```

Empat tanggung jawab BFF, masing-masing dengan alasan yang dapat diuji:

| Tanggung jawab | Alasan |
| --- | --- |
| Menembus CORS | Browser tidak diizinkan memanggil upstream (Temuan 1). |
| Menyimpan kredensial | `client_secret` tidak boleh berada di bundel browser. |
| Menyatukan biaya | Satu snapshot melayani N klien, sehingga biaya sebanding dengan waktu pemakaian, bukan jumlah pengguna (Temuan 4). |
| Mengisolasi penyedia | Frontend dan domain tidak mengetahui bentuk data upstream (ADR-003). |

---

## 4. Toolchain per Fase SDLC

Prinsip: **AI menghasilkan kandidat artefak; validator deterministik yang memutuskan lulus atau
tidak.**

| Fase | Generator / Otomasi | Validator deterministik | Artefak keluaran |
| --- | --- | --- | --- |
| 0 — Charter | Claude | `markdownlint`; checklist; **panggilan API nyata** | Charter, automation map, repository instructions, DoD |
| 1 — Requirement | Claude sebagai requirements engineer | Skrip audit ID; `markdownlint` | `specification.md`, `spec-review-report.md` |
| 2 — Executable spec | Claude → Gherkin | `@cucumber/gherkin` parser; `gherkin-lint`; BDD dry-run | `features/*.feature`, scenario catalog |
| 3 — Prototype & UI | Claude + Storybook | `axe-core`; interaction test; pemetaan state ↔ Gherkin | `ui-contract.md`, `state-matrix.md` |
| 4 — Arsitektur & kontrak | Claude → OpenAPI 3.1 + ADR | `redocly lint`; Prism mock; skema Zod tuple diuji terhadap fixture nyata; migrasi SQLite dari database kosong | `openapi/airadar.yaml`, `architecture-decisions/`, `data-model.md` |
| 5 — Perencanaan | Claude → task list | `npm run trace:audit` | `implementation-plan.md`, `tasks.md` |
| 6 — Implementasi | Claude Code | `tsc --noEmit`; ESLint; Prettier; build; review diff manusia | Kode sumber, pull request |
| 7 — Verifikasi integrasi | Claude → contract & integration test | Vitest + MSW; validasi respons BFF terhadap OpenAPI; uji migrasi dari database kosong; uji ledger anggaran | `tests/contract/*`, `tests/integration/*` |
| 8 — Testing berlapis | Claude + Playwright codegen | Vitest, Testing Library, Playwright, `axe-core`, Stryker, coverage | `tests/**`, `reports/**` |
| 9 — CI quality gate | GitHub Actions | Pipeline pada setiap pull request; fault injection | `.github/workflows/`, artefak run |

### 4.1 Stack

| Lapisan | Pilihan | Alasan |
| --- | --- | --- |
| Bahasa | TypeScript `strict`, di frontend dan backend | Tipe tuple menjadikan urutan indeks sebagai kontrak yang diperiksa compiler. |
| Backend runtime | Node.js 20 LTS | Sesuai arahan. |
| Backend framework | Fastify | Ringan, validasi skema bawaan, dukungan OpenAPI baik. |
| Database | SQLite via `better-sqlite3` | Satu berkas, tanpa layanan terpisah, mendukung migration dan seed dengan sederhana. |
| Validasi runtime | Zod | Skema tuple pada adapter; skema kontrak pada batas HTTP. |
| Frontend build | Vite | Dev server cepat, proxy ke BFF bawaan. |
| UI framework | React | Ekosistem test paling matang. |
| Peta | MapLibre GL JS | Sumber terbuka, tanpa API key, mendukung rotasi marker sesuai arah. |
| Data fetching | TanStack Query | Deduplikasi, penjedaan saat tab tidak aktif, retry terkendali. |
| Styling | Tailwind CSS | Iterasi cepat, konsisten dengan token desain Fase 3. |
| CI | GitHub Actions | Sesuai arahan. |

---

## 5. Repository Convention dan Perintah Standar

### 5.1 Struktur direktori

```
airadar/
├── README.md
├── .github/workflows/ci.yml
├── docs/
│   ├── project-charter.md · automation-map.md
│   ├── repository-instructions.md · definition-of-done.md
│   ├── specification.md · architecture-decisions/
│   ├── traceability-matrix.md · automation-evidence.md
│   └── final-review.md
├── features/                      # Gherkin
├── openapi/airadar.yaml           # Kontrak internal milik tim
├── server/
│   ├── src/
│   │   ├── http/                  # Route, validasi, penanganan error
│   │   ├── domain/                # Haversine, bounding box, radius, satuan
│   │   ├── providers/
│   │   │   ├── provider.ts        # Interface — batas abstraksi
│   │   │   ├── opensky/           # OAuth, tuple, Zod, mapper
│   │   │   └── fixture/           # Respons nyata tersimpan
│   │   ├── cache/                 # Snapshot cache TTL
│   │   ├── budget/                # Ledger credit, pagu, batas refresh
│   │   └── db/
│   │       ├── migrations/        # 001_*.sql, 002_*.sql
│   │       └── seed/
│   └── tests/{unit,contract,integration}/
├── web/
│   ├── src/{app,components,api,domain}/
│   └── tests/{unit,component}/
├── tests/e2e/                     # Playwright, lintas frontend + BFF
├── data/fixtures/                 # Respons nyata OpenSky
├── reports/
└── scripts/                       # verify.mjs, trace-audit.mjs
```

### 5.2 Konvensi ID artefak

| Artefak | Prefiks | Lokasi |
| --- | --- | --- |
| Functional requirement | `REQ-` | `docs/specification.md` |
| Business rule | `BR-` | `docs/specification.md` |
| UI state / komponen | `UI-` | `docs/ui-contract.md` |
| Operasi API internal | `API-` | `openapi/airadar.yaml` (`operationId`) |
| Aturan data | `DATA-` | `docs/data-model.md` |
| Implementation task | `TASK-` | `docs/tasks.md` |
| Automated test | `TST-` | tag/judul test |
| Architecture decision | `ADR-` | `docs/architecture-decisions/` |
| Risiko | `RISK-` | charter ini |

### 5.3 Konvensi branch dan commit

- Branch utama `main`, dilindungi. Merge hanya melalui pull request dengan CI hijau.
- Branch kerja: `feat/TASK-nnn-slug`, `fix/TASK-nnn-slug`, `docs/fase-n-slug`.
- Conventional Commits dengan ID traceability wajib:

  ```
  feat(server/domain): turunkan bounding box dari titik acuan [TASK-003][REQ-002][BR-002]
  feat(server/providers): normalisasi callsign kosong menjadi null [TASK-008][DATA-004]
  test(contract): assertion per indeks tuple state vector [TASK-006][TST-011]
  ```

- Satu task = satu branch = satu pull request.

### 5.4 Perintah standar

| Perintah | Fungsi |
| --- | --- |
| `npm install` | Memasang dependensi seluruh workspace. |
| `npm run dev` | Menjalankan BFF dan frontend bersamaan. |
| `npm run build` | Build backend dan frontend. |
| `npm run db:migrate` | Menjalankan migration sampai versi terbaru. |
| `npm run db:seed` | Memuat seed data. |
| `npm run db:reset` | Menghapus database, migrate ulang, seed ulang. |
| `npm run lint` | ESLint + Prettier. |
| `npm run typecheck` | `tsc --noEmit` pada seluruh workspace. |
| `npm run openapi:lint` | Validasi kontrak internal. |
| `npm run bdd` | Gherkin parse + dry-run. |
| `npm run test:unit` | Unit test. |
| `npm run test:component` | Component test. |
| `npm run test:contract` | Contract test. |
| `npm run test:integration` | Integration test BFF + database. |
| `npm run test:e2e` | End-to-end test. |
| `npm run test:a11y` | Pemindaian aksesibilitas. |
| `npm run test:mutation` | Mutation testing `server/src/domain`. |
| `npm run trace:audit` | Audit traceability. |
| `npm run verify` | Seluruh gate berurutan, sama dengan yang dijalankan CI. |

Seluruh perintah di atas berjalan dengan `FixtureProvider` dan **tidak memakai satu credit pun**.

### 5.5 Kredensial

| Aturan | Ketentuan |
| --- | --- |
| Lokasi | `server/.env.local`, ter-`gitignore`. Tidak pernah di-commit. |
| Isi | `OPENSKY_CLIENT_ID`, `OPENSKY_CLIENT_SECRET`. |
| Paparan | Kredensial tidak pernah meninggalkan proses server. Tidak dikirim ke browser, tidak masuk log, tidak masuk pesan error. |
| Frontend | Tidak memiliki variabel rahasia sama sekali. |
| CI | Tidak memerlukan kredensial, karena seluruh job berjalan dengan `FixtureProvider`. |
| Rotasi | Kredensial dirotasi setelah demo. |
| Penegakan | Uji otomatis memastikan respons BFF dan log tidak pernah memuat substring `client_secret` atau token. |

---

## 6. Definition of Done

Ringkasan; rincian pada `definition-of-done.md`.

### 6.1 Functional gate
1. Perilaku dapat ditelusuri ke minimal satu REQ atau BR.
2. Tidak ada perilaku tanpa sumber requirement.
3. Tidak ada field data yang tidak disediakan penyedia (OOS-02).
4. Skenario Gherkin terkait lulus, bukan dilewati.

### 6.2 Quality gate
5. `npm run lint` = 0 error.
6. `npm run typecheck` = 0 error.
7. `npm run build` berhasil dari checkout bersih.
8. Tidak ada `TODO`, `FIXME`, `console.log`, atau kode dinonaktifkan pada diff.
9. Istilah dan bentuk data khas penyedia tidak muncul di luar direktori adapter.

### 6.3 Test gate
10. Setiap business rule memiliki unit test positif dan negatif.
11. Setiap state UI memiliki component test.
12. Setiap indeks tuple yang dikonsumsi memiliki assertion tersendiri.
13. Setiap operasi API internal memiliki contract test, termasuk jalur kegagalan.
14. Critical journey tercakup E2E test.
15. Suite lulus dua kali berturut-turut (flaky rate 0%).
16. Coverage pada kode yang diubah ≥ 80%.
17. Mutation score `server/src/domain` mencapai target yang ditetapkan.

### 6.4 Security gate
18. Tidak ada kredensial pada kode, log, respons, maupun riwayat commit.
19. Temuan `axe-core` critical/serious = 0.
20. Seluruh fungsi dapat dioperasikan dengan keyboard; focus terlihat.
21. Perubahan status penting diumumkan kepada teknologi bantu.

### 6.5 Budget gate
22. Konsumsi credit selama seluruh test dan CI = 0.
23. Pagu harian ditegakkan di server dan diuji.
24. Batas refresh manual ditegakkan di server dan diuji.

### 6.6 Documentation & reproducibility gate
25. Traceability matrix diperbarui.
26. ADR ditulis untuk setiap keputusan yang sulit dibalik.
27. Evidence log fase diisi lengkap.
28. Output AI yang ditolak atau dikoreksi dicatat beserta alasan.
29. Anggota lain dapat menjalankan `npm install && npm run verify` pada mesin bersih tanpa kredensial.
30. Aplikasi dapat didemonstrasikan penuh dari fixture.

---

## 7. Risiko dan Titik Persetujuan Manusia

### 7.1 Register risiko

| ID | Risiko | Kategori | Dampak | Kemungkinan | Mitigasi | Pemilik |
| --- | --- | --- | --- | --- | --- | --- |
| RISK-01 | Node pada mesin target tidak dapat meresolusi `opensky-network.org` (terbukti `ENOTFOUND` pada pengujian awal). | Infrastruktur | **Kritis** | **Sedang** | Investigasi OQ-101 sebelum Fase 1 ditutup. Uji `curl` dan `node` dari mesin target. Periksa DNS, proxy, dan IPv6. Demo tetap dapat berjalan penuh dengan `FixtureProvider`. | Tech lead |
| RISK-02 | Anggaran 4.000 credit/hari habis akibat penyegaran berlebih. | Eksternal | Tinggi | Sedang | Cache server TTL 30 detik; pagu keras 1.600 permintaan/hari; ledger persisten; refresh manual dibatasi interval minimum; sisa anggaran ditampilkan. | Tech lead |
| RISK-03 | `client_secret` bocor melalui log, pesan error, atau respons. | Keamanan | Tinggi | Rendah | Kredensial hanya dibaca di satu modul; error upstream dipetakan ulang sebelum dikembalikan; uji otomatis memindai respons dan log. | Tech lead |
| RISK-04 | Token kedaluwarsa di tengah operasi menyebabkan 401 beruntun. | Teknis | Sedang | Sedang | Penyegaran proaktif dengan margin 30 detik sebelum kedaluwarsa; satu percobaan ulang pada 401; kegagalan menjadi error state terdefinisi. | Developer |
| RISK-05 | Bentuk respons berupa array posisional. Perubahan urutan indeks merusak aplikasi secara diam-diam. | Teknis | Tinggi | Rendah | Skema Zod tuple; contract test per indeks; fault injection FI-03 menukar longitude dan latitude. | Developer |
| RISK-06 | AI menghasilkan field, endpoint, atau parameter yang tidak ada pada API sebenarnya. | Otomasi | Tinggi | Tinggi | Fixture berasal dari respons nyata; setiap field diverifikasi terhadap dokumentasi resmi dan data terlampir. | QA |
| RISK-07 | AI menambahkan data bandara, maskapai, atau nomor penerbangan yang tidak tersedia. | Otomasi | Sedang | **Tinggi** | OOS-02 dinyatakan eksplisit pada charter dan repository instructions; reviewer agent memeriksa invented scope; DoD butir 3. | Product owner |
| RISK-08 | Detail penyedia merembes ke domain atau frontend, sehingga abstraksi ADR-003 hanya di atas kertas. | Arsitektur | Sedang | Sedang | Lint rule melarang impor lintas batas; DoD butir 9; `FixtureProvider` sebagai bukti hidup; uji pergantian provider lewat variabel lingkungan. | Tech lead |
| RISK-09 | Test yang dihasilkan AI menegaskan implementasi, bukan requirement. | Otomasi | Tinggi | Tinggi | Mutation testing pada domain; setiap test merujuk ID sumber. | QA |
| RISK-10 | Kepadatan data sangat rendah pada waktu demo, misalnya dini hari. | Data | Sedang | Sedang | Sampling pada beberapa waktu; empty state yang jujur; demo utama memakai fixture dengan 29 pesawat. | Tech lead |
| RISK-11 | Perhitungan radius salah pada kasus batas. | Teknis | Sedang | Sedang | Boundary test 499,9 / 500,0 / 500,1 km; property test bounding box ⊇ lingkaran. | Developer |
| RISK-12 | Dokumen turunan menjadi basi ketika specification berubah. | Proses | Sedang | Tinggi | Perubahan requirement wajib memperbarui artefak turunan pada pull request yang sama. | Tech lead |
| RISK-13 | CI menjadi lambat sehingga tim tergoda melewatinya. | Proses | Sedang | Sedang | Mutation testing dijalankan lokal, bukan di CI; job diparalelkan; durasi diukur dan dilaporkan. | Tech lead |
| RISK-14 | Penyedia tile peta membatasi akses. | Eksternal | Rendah | Rendah | Pilih penyedia dengan ketentuan jelas; tampilkan atribusi; siapkan satu cadangan. | Developer |

### 7.2 Titik yang memerlukan persetujuan manusia

| ID | Titik keputusan | Fase | Mengapa manusia harus memutuskan | Bukti persetujuan |
| --- | --- | --- | --- | --- |
| HA-01 | Penutupan OQ-101..OQ-104. | 0 → 1 | Menentukan kelayakan teknis. AI tidak dapat memverifikasi jaringan mesin target. | Catatan keputusan + bukti uji. |
| HA-02 | Penerimaan `specification.md` sebagai sumber kebenaran. | 1 → 2 | Kesalahan di sini menyebar ke seluruh artefak turunan. | Sign-off pada spec review report. |
| HA-03 | Penetapan business rule radius, interval, pagu anggaran, dan batas refresh manual. | 1 | Keputusan bisnis, bukan teknis. | BR bernomor dengan alasan tertulis. |
| HA-04 | Konfirmasi setiap indeks tuple terhadap respons nyata. | 4 | Mitigasi langsung RISK-05 dan RISK-06. | Fixture nyata + hasil validasi skema. |
| HA-05 | Persetujuan ADR-001 sampai ADR-004. | 4 | Konsekuensi melampaui satu task. | ADR berstatus Accepted. |
| HA-06 | Review diff sebelum merge ke `main`. | 6 | AI dapat menghasilkan kode yang lulus test tetapi salah maksud. | Review comment pada pull request. |
| HA-07 | Penerimaan mutant yang bertahan. | 8 | Menentukan apakah mutant tidak relevan atau test lemah. | Justifikasi per mutant. |
| HA-08 | Perubahan cakupan setelah Fase 1 ditutup. | Semua | Mencegah scope creep. | Change record pada charter. |
| HA-09 | Pernyataan READY / NOT READY pada audit akhir. | Final | Penilaian kesiapan tidak dapat didelegasikan ke AI. | `final-review.md` bertanda tangan. |

### 7.3 Daftar Architecture Decision Record

| ID | Keputusan | Alasan ringkas | Status |
| --- | --- | --- | --- |
| ADR-001 | Menggunakan backend (BFF), bukan frontend-only. | CORS memblokir akses langsung dari browser (Temuan 1). Keharusan teknis, bukan preferensi. | Diusulkan |
| ADR-002 | Menggunakan OAuth2 client credentials, bukan mode anonim. | Kredensial aman di server; anggaran naik dari 400 ke 4.000 credit/hari; resolusi data 5 detik. | Diusulkan |
| ADR-003 | Mengisolasi seluruh detail penyedia di balik interface provider. | Penyedia sudah berganti sekali; abstraksi membuat pergantian berikutnya murah dan menjaga domain tetap bersih. | Diusulkan |
| ADR-004 | Cache snapshot di server dengan TTL, bukan pemanggilan per klien. | Penyegaran 30 detik tanpa cache melampaui kuota 44% (Temuan 4). Cache membuat biaya sebanding dengan waktu pemakaian, bukan jumlah pengguna. | Diusulkan |
| ADR-005 | Menggunakan SQLite untuk ledger anggaran dan snapshot terakhir. | Ledger harus bertahan melintasi restart; snapshot terakhir dibutuhkan saat upstream gagal (OUT-07). | Diusulkan, bergantung OQ-102 |

---

## 8. Riwayat Perubahan Charter

| Versi | Tanggal | Perubahan | Alasan |
| --- | --- | --- | --- |
| 1.0 | 2026-08-28 | Charter awal berbasis aviationstack, arsitektur frontend-only. | Inisiasi proyek. |
| 2.0 | 2026-08-28 | Penggantian penyedia data ke OpenSky Network; bounding box menjadi mekanisme query utama; mode anonim tanpa kredensial. | Aviationstack tidak memiliki parameter geospasial. |
| 3.0 | 2026-08-28 | Perubahan arsitektur menyeluruh berdasarkan hasil verifikasi Fase 0: (a) **penambahan backend Node.js (BFF)** karena CORS terbukti memblokir akses langsung dari browser; (b) **OAuth2 client credentials** menggantikan mode anonim, sehingga anggaran naik menjadi 4.000 credit/hari; (c) **pipeline GitHub Actions dikembalikan ke cakupan**, sehingga automatic cap rubrik tidak lagi berlaku; (d) **lapisan abstraksi provider (ADR-003)** ditetapkan sebagai prinsip arsitektur, dengan `FixtureProvider` sebagai bukti hidup; (e) **cache snapshot di server (ADR-004)** karena penyegaran 30 detik tanpa cache melampaui kuota 44%; (f) penambahan **SQLite** untuk ledger anggaran dan snapshot terakhir, sekaligus memenuhi syarat database persisten; (g) penambahan tombol refresh manual (OUT-05) dan penyajian data terakhir yang diketahui baik saat upstream gagal (OUT-07); (h) OQ-001 sampai OQ-006 ditutup dengan bukti, digantikan OQ-101 sampai OQ-104; (i) risiko baru RISK-01 (kegagalan DNS di Node), RISK-03 (kebocoran kredensial), RISK-04 (kedaluwarsa token), RISK-08 (kebocoran abstraksi). | Hasil verifikasi teknis Fase 0 dan arahan tim. |
