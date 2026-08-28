# Repository Instructions — AiRadar

| Field | Value |
| --- | --- |
| Versi | 3.0 |
| Tanggal | 2026-08-28 |
| Dokumen induk | `project-charter.md` v3.0 |

Dokumen ini adalah sumber kebenaran bagi manusia maupun coding agent yang bekerja pada repository
ini. Coding agent wajib membaca dokumen ini sebelum mengubah kode.

---

## 1. Ringkasan proyek

AiRadar menampilkan pesawat yang sedang berada di udara dalam radius 500 km dari PT PLN (Persero)
UPDL Bogor (`-6.6564, 106.8760`).

Arsitektur: **Browser SPA → AiRadar BFF (Node.js) → OpenSky Network.**

Backend bukan pilihan gaya. OpenSky mengembalikan `Access-Control-Allow-Origin` berisi origin
miliknya sendiri, sehingga tidak ada origin browser yang dapat memanggilnya. Ini terverifikasi
pada Fase 0 dan diformalkan sebagai ADR-001.

**Proyek ini memiliki satu fitur.** Jika sebuah perubahan menghasilkan perilaku yang tidak dapat
ditelusuri ke REQ atau BR pada `docs/specification.md`, perubahan itu di luar cakupan.

---

## 2. Aturan arsitektur yang tidak dapat dinegosiasikan

### 2.1 Batas abstraksi provider (ADR-003)

> Detail penyedia data berhenti di `server/src/providers/<nama>/`. Tidak ada yang lolos keluar.

| Lapisan | Boleh mengetahui |
| --- | --- |
| `server/src/providers/opensky/` | Endpoint, OAuth, tuple posisional, satuan SI, tarif credit, nama OpenSky |
| `server/src/providers/provider.ts` | Hanya interface dan model domain |
| `server/src/domain/` | Model domain saja |
| `server/src/http/` | Model domain dan kontrak internal saja |
| `web/` | Kontrak internal saja |

Ditegakkan oleh aturan lint batas impor. Pelanggaran menggagalkan `npm run lint` dan job `static`
di CI. Fault injection FI-05 menguji penegakan ini.

Interface provider:

```ts
export interface FlightDataProvider {
  readonly id: string;
  fetchArea(area: BoundingBox): Promise<AreaSnapshot>;
  estimateCost(area: BoundingBox): number;
}
```

Terdapat dua implementasi. `FixtureProvider` bukan mock untuk test — ia provider yang berfungsi
penuh, menjadi default pengembangan, dan keberadaannya adalah bukti bahwa abstraksi benar-benar
bekerja. Jika abstraksi bocor, provider ini berhenti berfungsi.

### 2.2 Model domain bersifat provider-neutral

Model domain menggunakan nama dan satuan yang dipahami manusia, bukan istilah penyedia:

```ts
export interface Aircraft {
  id: string;                        // identitas stabil lintas pembaruan
  callsign: string | null;           // sudah dinormalisasi; string kosong menjadi null
  registrationCountry: string;
  position: { lat: number; lon: number };
  altitude: { barometricM: number | null; geometricM: number | null };
  groundSpeedMs: number | null;
  headingDeg: number | null;
  verticalRateMs: number | null;
  onGround: boolean;
  positionSource: 'adsb' | 'asterix' | 'mlat' | 'flarm';
  squawk: string | null;
  lastContactAt: Date;
  positionUpdatedAt: Date | null;
}
```

Perhatikan yang tidak ada: `icao24`, `baro_altitude`, `true_track`, `states`. Istilah tersebut
hanya hidup di dalam adapter.

### 2.3 Kredensial tidak pernah meninggalkan server

| Aturan | Ketentuan |
| --- | --- |
| Lokasi | `server/.env.local`, ter-`gitignore` |
| Pembaca | Satu modul saja: `server/src/providers/opensky/auth.ts` |
| Larangan | Tidak masuk log, tidak masuk pesan error, tidak masuk respons HTTP |
| Error upstream | Dipetakan ulang sebelum dikembalikan ke klien; jangan meneruskan mentah |
| Frontend | Tidak memiliki variabel rahasia sama sekali |
| CI | Tidak memerlukan secret, karena berjalan dengan `FixtureProvider` |

---

## 3. Batasan data yang wajib dipahami sebelum menulis kode

Bagian ini ada karena kesalahan paling mungkin pada proyek ini adalah **mengasumsikan data yang
tidak ada**.

### 3.1 Yang TIDAK tersedia

`/states/all` tidak menyediakan:

- bandara asal atau bandara tujuan;
- nama maskapai;
- nomor penerbangan komersial — yang tersedia hanya *callsign*, dan keduanya tidak sama;
- status seperti "delayed", "on time", atau "landed";
- jadwal keberangkatan atau kedatangan;
- tipe atau model pesawat.

Jangan membuat field ini. Jangan menurunkannya dengan tebakan dari callsign. Jangan menambahkan
tabel pencarian maskapai. Semua tercantum pada OOS-02 charter.

### 3.2 Bentuk respons yang sebenarnya (terverifikasi Fase 0)

Tuple berisi **17 elemen**. Indeks 17 (`category`) hanya muncul bila `extended=1` dikirim, dan
kami tidak mengirimkannya.

| Indeks | Field | Tipe | Catatan dari data nyata |
| --- | --- | --- | --- |
| 0 | `icao24` | string | Identitas unik. Selalu terisi. |
| 1 | `callsign` | string \| null | **Dapat berupa string kosong `""`, bukan null.** Sering berakhiran spasi. |
| 2 | `origin_country` | string | Selalu terisi. |
| 3 | `time_position` | int \| null | Unix time posisi terakhir. |
| 4 | `last_contact` | int | Unix time kontak terakhir. |
| 5 | `longitude` | float \| null | **Longitude lebih dulu, bukan latitude.** |
| 6 | `latitude` | float \| null | |
| 7 | `baro_altitude` | float \| null | Meter. Null pada pesawat di darat. |
| 8 | `on_ground` | boolean | |
| 9 | `velocity` | float \| null | **Meter per detik, bukan knot.** |
| 10 | `true_track` | float \| null | Derajat searah jarum jam dari utara. |
| 11 | `vertical_rate` | float \| null | Meter per detik; positif berarti menanjak. |
| 12 | `sensors` | int[] \| null | **Null pada seluruh 29 record.** |
| 13 | `geo_altitude` | float \| null | Meter. |
| 14 | `squawk` | string \| null | Null pada 5 dari 29 record. |
| 15 | `spi` | boolean | |
| 16 | `position_source` | int | 0=ADS-B, 1=ASTERIX, 2=MLAT, 3=FLARM. |

### 3.3 Empat kesalahan yang paling mungkin terjadi

1. **Indeks 5 adalah longitude, indeks 6 adalah latitude.** Urutannya kebalikan dari kebiasaan
   penulisan koordinat. Kesalahan ini **tidak terdeteksi type-check** karena keduanya `float` —
   aplikasi tetap berjalan, hanya semua pesawat muncul di tempat yang salah. Fault injection
   FI-03 menguji hal ini.
2. **Callsign dapat berupa string kosong.** Memeriksa `null` saja tidak cukup. Normalisasi wajib:
   `callsign?.trim() || null`. Terbukti terjadi pada 2 dari 29 record.
3. **`states` dapat bernilai `null`**, bukan array kosong, ketika tidak ada pesawat.
4. **Satuan adalah SI.** Kecepatan m/s, ketinggian meter. Konversi ke satuan tampilan dilakukan
   di `server/src/domain/`, bukan di komponen, dan setiap konversi memiliki unit test.

| Dari | Ke | Faktor |
| --- | --- | --- |
| `velocity` m/s | km/jam | × 3,6 |
| altitude meter | kaki | × 3,28084 |
| `vertical_rate` m/s | kaki/menit | × 196,850 |

### 3.4 Kepadatan data yang teramati

Satu permintaan mengembalikan 29 state vector, seluruhnya di dalam radius 500 km. Terdekat
37,9 km, terjauh 479,6 km. Negara registrasi: Indonesia 26, Filipina 1, Singapura 1, Arab Saudi 1.
Dua pesawat berstatus `on_ground = true`.

Angka ini menjadi dasar rancangan tampilan. Jangan merancang untuk ratusan pesawat, dan jangan
merancang seolah daftar selalu penuh — kepadatan pada dini hari akan jauh lebih rendah.

---

## 4. Prasyarat

| Kebutuhan | Versi |
| --- | --- |
| Node.js | 20 LTS atau lebih baru |
| npm | 10 atau lebih baru |
| Browser untuk E2E | Chromium (dipasang otomatis oleh Playwright) |

Konektivitas jaringan hanya dibutuhkan untuk mode `opensky`. Seluruh pengembangan, test, dan CI
berjalan dengan `FixtureProvider` tanpa jaringan keluar.

> **Catatan RISK-01:** pengujian awal `node check.js` menghasilkan `ENOTFOUND
> opensky-network.org` sementara browser pada mesin yang sama berhasil. Sebelum menjalankan mode
> `opensky`, verifikasi dari mesin target:
>
> ```bash
> node -e "require('dns').lookup('opensky-network.org', console.log)"
> curl -sS -o /dev/null -w '%{http_code}\n' https://opensky-network.org/api/states/all
> ```

---

## 5. Setup dari checkout bersih

```bash
git clone <url-repo>
cd airadar
npm install
cp server/.env.example server/.env.local   # boleh dibiarkan kosong untuk mode fixture
npm run db:reset                           # migrate + seed dari database kosong
npm run verify                             # seluruh gate, tanpa jaringan, tanpa credit
npm run dev                                # BFF + frontend
```

Setup harus berhasil **tanpa kredensial dan tanpa jaringan**. Mode fixture bukan jalur darurat —
ia adalah mode default, demi menjaga anggaran credit dan determinisme test.

---

## 6. Variabel lingkungan

### 6.1 Backend (`server/.env.local`)

| Variabel | Rahasia | Default | Deskripsi |
| --- | --- | --- | --- |
| `PROVIDER` | Tidak | `fixture` | `fixture` atau `opensky`. |
| `OPENSKY_CLIENT_ID` | **Ya** | — | Hanya diperlukan bila `PROVIDER=opensky`. |
| `OPENSKY_CLIENT_SECRET` | **Ya** | — | Hanya diperlukan bila `PROVIDER=opensky`. |
| `ANCHOR_LAT` | Tidak | `-6.6564` | Lintang titik acuan. |
| `ANCHOR_LON` | Tidak | `106.8760` | Bujur titik acuan. |
| `RADIUS_KM` | Tidak | `500` | Radius penyaringan. |
| `SNAPSHOT_TTL_S` | Tidak | `30` | Umur maksimum snapshot sebelum upstream dipanggil ulang. |
| `MANUAL_REFRESH_MIN_INTERVAL_S` | Tidak | `10` | Jeda minimum antar refresh manual yang benar-benar memanggil upstream. |
| `DAILY_REQUEST_CAP` | Tidak | `1600` | Pagu keras permintaan upstream per hari. |
| `DATABASE_PATH` | Tidak | `./data/airadar.db` | Lokasi berkas SQLite. |

### 6.2 Frontend (`web/.env.local`)

| Variabel | Rahasia | Default | Deskripsi |
| --- | --- | --- | --- |
| `VITE_API_BASE` | Tidak | `/api/v1` | Basis URL BFF. |
| `VITE_POLL_INTERVAL_S` | Tidak | `30` | Interval polling frontend ke BFF. |

Frontend tidak memiliki satu pun variabel rahasia. Bounding box **tidak** dikonfigurasi manual —
diturunkan dari `ANCHOR_LAT`, `ANCHOR_LON`, dan `RADIUS_KM` di `server/src/domain/`.

---

## 7. Perintah standar

| Perintah | Fungsi | Kapan dijalankan |
| --- | --- | --- |
| `npm run dev` | BFF + frontend bersamaan | Pengembangan harian |
| `npm run build` | Build keduanya | Sebelum demo |
| `npm run db:migrate` | Migration sampai versi terbaru | Setelah menambah migration |
| `npm run db:seed` | Memuat seed | Setelah reset |
| `npm run db:reset` | Hapus, migrate, seed | Saat skema berubah |
| `npm run lint` | ESLint + Prettier + aturan batas impor | Sebelum commit |
| `npm run typecheck` | `tsc --noEmit` seluruh workspace | Sebelum commit |
| `npm run openapi:lint` | Validasi kontrak internal | Setelah mengubah `openapi/` |
| `npm run bdd` | Gherkin parse + dry-run | Setelah mengubah `features/` |
| `npm run test:unit` | Unit test | Setelah mengubah domain |
| `npm run test:component` | Component test | Setelah mengubah komponen |
| `npm run test:contract` | Contract test | Setelah mengubah provider atau kontrak |
| `npm run test:integration` | BFF + database | Setelah mengubah http, cache, budget, db |
| `npm run test:e2e` | End-to-end | Sebelum pull request |
| `npm run test:a11y` | Aksesibilitas | Sebelum pull request |
| `npm run test:mutation` | Mutation testing domain | Sebelum menutup Fase 8 |
| `npm run trace:audit` | Audit traceability | Sebelum pull request |
| `npm run verify` | Seluruh gate, sama dengan CI | Sebelum pull request, wajib |

Tidak satu pun memakai credit atau jaringan keluar. Test gagal jika ada panggilan jaringan keluar
selama suite berjalan.

---

## 8. Struktur direktori dan tanggung jawab

| Path | Isi | Aturan |
| --- | --- | --- |
| `server/src/http/` | Route, validasi permintaan, penanganan error, serialisasi kontrak. | Tidak memuat business rule. Tidak meneruskan error upstream mentah. |
| `server/src/domain/` | Haversine, penurunan bounding box, penyaringan radius, pengurutan, konversi satuan. | **Tanpa I/O, tanpa dependensi eksternal, tanpa istilah penyedia.** Seluruh fungsi murni. Satu-satunya target mutation testing. |
| `server/src/providers/provider.ts` | Interface `FlightDataProvider` dan model domain. | Batas abstraksi. Tidak memuat istilah penyedia. |
| `server/src/providers/opensky/` | OAuth, HTTP, skema Zod tuple, mapper. | **Satu-satunya tempat yang boleh mengetahui bentuk data OpenSky.** Tuple tidak boleh diekspor keluar. |
| `server/src/providers/fixture/` | Membaca respons nyata tersimpan. | Harus melewati test yang sama dengan provider lain. |
| `server/src/cache/` | Snapshot cache dengan TTL. | Menyimpan snapshot terakhir yang diketahui baik untuk OUT-07. |
| `server/src/budget/` | Ledger credit, pagu harian, batas refresh manual. | Menolak permintaan sebelum upstream dipanggil, bukan sesudah. |
| `server/src/db/migrations/` | `001_*.sql`, `002_*.sql`, berurutan. | Migration bersifat append-only. Jangan mengedit migration yang sudah di-merge. |
| `server/src/db/seed/` | Seed dari fixture nyata. | Dapat dimuat dari database kosong. |
| `web/src/api/` | Client ke BFF, skema kontrak. | Tidak mengetahui nama penyedia. |
| `web/src/domain/` | Format tampilan saja. | Business rule ada di server, bukan di sini. |
| `web/src/components/` | Komponen presentasional. | Tanpa pemanggilan API langsung. |
| `web/src/app/` | Komposisi halaman, penyediaan data, state seleksi. | Satu-satunya penghubung api dan components. |
| `features/` | Gherkin. | Judul atau tag wajib memuat ID REQ/BR. |
| `openapi/airadar.yaml` | Kontrak internal milik tim. | **Bebas istilah penyedia.** Setiap operasi punya `operationId` ber-ID API-nnn dan contoh. |
| `data/fixtures/` | Respons nyata OpenSky. | **Berasal dari respons nyata, tidak dibuat AI.** Menyertakan waktu pengambilan dan bounding box. |
| `tests/e2e/` | Playwright lintas frontend + BFF. | Setiap test merujuk ID sumber. |
| `reports/` | Keluaran gate. | Dihasilkan, tidak diedit manual. |
| `docs/` | Artefak SDLC. | Diperbarui pada pull request yang sama dengan perubahan kode. |

---

## 9. Kontrak API internal

Dua operasi. Bentuk final ditetapkan pada Fase 4.

| Operasi | Metode | Perilaku |
| --- | --- | --- |
| `getAirspace` | `GET /api/v1/airspace` | Menyajikan snapshot. Jika umur snapshot < TTL, dilayani dari cache tanpa memakai credit. Jika sudah basi dan anggaran mengizinkan, upstream dipanggil. Jika anggaran habis atau upstream gagal, snapshot terakhir dilayani dengan penanda `stale`. |
| `refreshAirspace` | `POST /api/v1/airspace/refresh` | Memaksa pemanggilan upstream. Ditolak dengan penanda `throttled` bila jeda minimum belum terlampaui, atau `budgetExhausted` bila pagu habis. Menyediakan OUT-05 tanpa membuka celah penyalahgunaan anggaran. |

Respons memuat metadata yang dipakai UI: waktu pengamatan dari penyedia, umur data, apakah data
basi, sisa anggaran, dan sumber data (`cache`, `upstream`, `fixture`).

---

## 10. Konvensi kode

1. TypeScript `strict`. Tidak ada `any`, tidak ada `@ts-ignore` tanpa komentar alasan.
2. Business rule hanya di `server/src/domain/`. Tidak ada logika jarak, radius, atau konversi
   satuan yang tersebar di route maupun komponen.
3. Setiap fungsi domain memiliki unit test positif dan negatif sebelum dianggap selesai.
4. Setiap komponen mendukung seluruh state pada `docs/state-matrix.md`.
5. Tidak ada `console.log` pada kode yang di-commit. Gunakan logger yang menyaring rahasia.
6. Nilai literal yang merupakan aturan bisnis (radius 500 km, TTL 30 detik, pagu 1.600) menjadi
   konstanta bernama dengan komentar merujuk ID BR.
7. Koordinat selalu diteruskan sebagai `{ lat, lon }`, tidak pernah sebagai tuple posisional, di
   luar adapter. Mitigasi langsung RISK-05.
8. Migration bersifat append-only.

---

## 11. Alur kerja Git

1. Branch dari `main`: `feat/TASK-nnn-slug`.
2. Kerjakan satu TASK saja.
3. `npm run verify` sampai hijau.
4. Perbarui `docs/traceability-matrix.md` dan `docs/automation-evidence.md`.
5. Buka pull request; tunggu CI hijau.
6. Setelah review manusia (HA-06) disetujui, merge.

Format commit:

```
<type>(<scope>): <deskripsi> [TASK-nnn][REQ-nnn]
```

Contoh:

```
feat(server/domain): turunkan bounding box dari titik acuan dan radius [TASK-003][REQ-002][BR-002]
feat(server/providers): normalisasi callsign kosong menjadi null [TASK-008][DATA-004]
feat(server/budget): tegakkan pagu permintaan harian [TASK-012][BR-005]
fix(server/http): jangan teruskan pesan error upstream ke klien [TASK-015][RISK-03]
test(contract): assertion per indeks tuple state vector [TASK-006][TST-011]
```

---

## 12. Templat pull request

```markdown
## TASK
TASK-nnn — <judul>

## Artefak sumber
- REQ:      - BR:      - UI: 
- API:      - DATA:    - TST: 

## Ringkasan perubahan
<apa yang berubah dan mengapa>

## Quality gate
- [ ] npm run verify hijau secara lokal
- [ ] CI hijau
- [ ] npm run trace:audit

## Batas abstraksi
- [ ] Tidak ada istilah atau tipe penyedia di luar `server/src/providers/<nama>/`
- [ ] Kontrak internal tetap bebas istilah penyedia

## Anggaran
Panggilan upstream nyata selama pengerjaan task ini: ___
(Seharusnya 0. Jika bukan nol, jelaskan alasannya.)

## Koreksi manusia terhadap output AI
| Output AI | Masalah | Koreksi | Alasan |
| --- | --- | --- | --- |

## Output AI yang ditolak
<daftar kandidat yang dibuang beserta alasannya, atau "tidak ada">
```

---

## 13. Instruksi khusus untuk coding agent

Sebelum mengubah kode, agent wajib:

1. Menyatakan pemahamannya terhadap TASK dengan kata-katanya sendiri.
2. Menyebutkan file yang akan diubah.
3. Menyampaikan rencana test.
4. Menandai informasi yang hilang atau bertentangan, dan **berhenti** jika ada.

Setelah mengubah kode, agent wajib:

1. Menjalankan `npm run verify`.
2. Menyampaikan ringkasan diff, risiko, dan traceability.
3. Tidak mengubah cakupan, dependensi, atau versi paket tanpa persetujuan.

Larangan keras:

- **Menambahkan field bandara, maskapai, atau nomor penerbangan.** Data ini tidak ada pada API
  (bagian 3.1). Pelanggaran paling mungkin terjadi dan paling merusak.
- **Mengimpor apa pun dari `server/src/providers/opensky/` di luar direktori itu sendiri.**
- **Menyebut nama penyedia pada kontrak internal, model domain, atau frontend.**
- Membuat fixture sintetis dan menyebutnya sebagai respons API nyata.
- Menuliskan kredensial atau token ke log, pesan error, atau respons.
- Memanggil API nyata dari test atau dari kode yang berjalan saat `npm run verify`.
- Menonaktifkan, melewati (`skip`), atau melonggarkan test agar gate menjadi hijau.
- Mengedit migration yang sudah di-merge.
- Mengubah `docs/specification.md` tanpa persetujuan manusia.

---

## 14. Anggaran API

| Aturan | Nilai |
| --- | --- |
| Mode akses | OAuth2 client credentials (ADR-002) |
| Masa berlaku token | 30 menit; disegarkan proaktif dengan margin 30 detik |
| Anggaran harian | 4.000 credit pada bucket `/states/*` |
| Biaya per permintaan | 2 credit (bounding box 81,43 sq°, kelas 25–100 sq°) |
| Permintaan maksimum teoretis | 2.000/hari |
| Pagu keras yang ditegakkan | **1.600 permintaan/hari** (3.200 credit), cadangan 800 credit |
| TTL snapshot | 30 detik |
| Jeda minimum refresh manual | 10 detik |
| Setara waktu pemakaian aktif | ≈ 13,3 jam/hari pada TTL 30 detik |
| Mode default pengembangan, test, dan CI | `fixture` (0 credit) |

**Mengapa TTL wajib.** Penyegaran 30 detik tanpa cache membutuhkan 2.880 permintaan/hari = 5.760
credit, yaitu 44% di atas kuota. Dengan cache di server, satu snapshot melayani seluruh klien,
sehingga biaya sebanding dengan **waktu pemakaian**, bukan jumlah pengguna. Seratus peserta yang
membuka aplikasi bersamaan berbiaya sama dengan satu orang.

**Mengapa ledger harus persisten.** Jika hitungan pagu hanya berada di memori, restart server
mengembalikannya ke nol dan anggaran dapat terlampaui tanpa terdeteksi. Karena itu ledger
disimpan di SQLite.

Ketika anggaran upstream habis, OpenSky mengembalikan `429` beserta
`X-Rate-Limit-Retry-After-Seconds`. BFF wajib memetakannya menjadi state terdefinisi, menyajikan
snapshot terakhir yang diketahui baik, dan tidak mencoba ulang sebelum waktu tunggu lewat.

Setiap panggilan upstream nyata dicatat pada `docs/automation-evidence.md`.

---

## 15. Riwayat Perubahan

| Versi | Tanggal | Perubahan |
| --- | --- | --- |
| 1.0 | 2026-08-28 | Versi awal berbasis aviationstack, frontend-only. |
| 2.0 | 2026-08-28 | Disesuaikan untuk OpenSky mode anonim; penambahan tabel indeks tuple. |
| 3.0 | 2026-08-28 | Disesuaikan untuk arsitektur BFF: (a) bagian 2 aturan arsitektur yang tidak dapat dinegosiasikan — batas abstraksi provider, model domain provider-neutral, perlindungan kredensial; (b) bagian 3.2 diperbarui dengan temuan nyata 29 record termasuk callsign string kosong dan `sensors` null pada seluruh record; (c) bagian 3.3 empat kesalahan paling mungkin; (d) bagian 9 kontrak API internal dua operasi; (e) bagian 14 ditulis ulang dengan anggaran OAuth2 dan penjelasan mengapa TTL serta ledger persisten diperlukan; (f) penambahan catatan RISK-01 mengenai kegagalan DNS di Node; (g) larangan impor lintas batas dan penyebutan nama penyedia di luar adapter. |
