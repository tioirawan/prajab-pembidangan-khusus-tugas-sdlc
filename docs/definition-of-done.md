# Definition of Done — AiRadar

| Field | Value |
| --- | --- |
| Versi | 3.0 |
| Tanggal | 2026-08-28 |
| Dokumen induk | `project-charter.md` v3.0 |

Berlaku untuk seluruh proyek. Sebuah unit kerja dinyatakan selesai hanya jika seluruh kriteria
yang berlaku terpenuhi. Kriteria bersifat **biner**: terpenuhi atau tidak. Tidak ada status
"sebagian".

---

## 1. DoD tingkat Task

### 1.1 Functional

| ID | Kriteria | Cara memeriksa |
| --- | --- | --- |
| DOD-T01 | Perilaku dapat ditelusuri ke minimal satu REQ atau BR. | `npm run trace:audit` |
| DOD-T02 | Tidak ada perilaku tanpa sumber requirement. | Review diff (HA-06) |
| DOD-T03 | Tidak ada field data yang tidak disediakan penyedia — bandara, maskapai, nomor penerbangan, status keterlambatan. | Review diff terhadap OOS-02 |
| DOD-T04 | Acceptance criteria pada TASK terpenuhi seluruhnya. | Checklist pada pull request |

### 1.2 Arsitektur

| ID | Kriteria | Cara memeriksa |
| --- | --- | --- |
| DOD-T05 | Tidak ada impor dari `server/src/providers/<nama>/` di luar direktori itu sendiri. | Aturan lint batas impor |
| DOD-T06 | Tidak ada istilah khas penyedia pada model domain, kontrak internal, atau frontend. | Skrip pemindaian istilah |
| DOD-T07 | Tipe tuple posisional tidak diekspor keluar adapter. | Review + typecheck |
| DOD-T08 | Business rule berada di `server/src/domain/`, bukan di route maupun komponen. | Review diff |

### 1.3 Kualitas kode

| ID | Kriteria | Cara memeriksa |
| --- | --- | --- |
| DOD-T09 | `npm run lint` = 0 error. | Otomatis |
| DOD-T10 | `npm run typecheck` = 0 error. | Otomatis |
| DOD-T11 | `npm run build` berhasil dari checkout bersih. | Otomatis |
| DOD-T12 | Tidak ada `TODO`, `FIXME`, `console.log`, atau kode dinonaktifkan pada diff. | Lint rule + review |

### 1.4 Test

| ID | Kriteria | Cara memeriksa |
| --- | --- | --- |
| DOD-T13 | Setiap business rule baru memiliki unit test positif dan negatif. | Review + coverage |
| DOD-T14 | Setiap state UI baru memiliki component test. | Review terhadap state matrix |
| DOD-T15 | Setiap indeks tuple baru yang dikonsumsi memiliki assertion tersendiri. | Review `tests/contract` |
| DOD-T16 | Setiap operasi API internal baru memiliki contract test, termasuk jalur kegagalan. | Review |
| DOD-T17 | Perubahan pada skema database disertai migration dan test migrasi dari database kosong. | `npm run db:reset` + test integrasi |
| DOD-T18 | Seluruh test yang terdampak lulus dua kali berturut-turut. | `npm run verify` ×2 |
| DOD-T19 | Coverage pada kode yang diubah ≥ 80%. | Laporan coverage |

### 1.5 Keamanan dan anggaran

| ID | Kriteria | Cara memeriksa |
| --- | --- | --- |
| DOD-T20 | Tidak ada kredensial pada diff, log, respons, maupun riwayat commit. | Uji pemindaian + review |
| DOD-T21 | Error upstream tidak diteruskan mentah ke klien. | Review + contract test |
| DOD-T22 | Panggilan upstream nyata selama pengerjaan task = 0, atau alasannya dijelaskan. | Templat pull request |
| DOD-T23 | Tidak ada panggilan jaringan keluar selama suite test berjalan. | Uji otomatis |

### 1.6 Aksesibilitas

| ID | Kriteria | Cara memeriksa |
| --- | --- | --- |
| DOD-T24 | Temuan `axe-core` critical/serious pada layar terdampak = 0. | `npm run test:a11y` |
| DOD-T25 | Fungsi baru dapat dioperasikan sepenuhnya dengan keyboard, focus terlihat. | Uji manual, dicatat |

### 1.7 Dokumentasi dan proses

| ID | Kriteria | Cara memeriksa |
| --- | --- | --- |
| DOD-T26 | Traceability matrix diperbarui pada pull request yang sama. | `npm run trace:audit` |
| DOD-T27 | Evidence log diisi: input, tool, output, validasi, koreksi manusia. | Review |
| DOD-T28 | Koreksi dan penolakan terhadap output AI dicatat beserta alasan. | Templat pull request |
| DOD-T29 | Pull request mencakup tepat satu TASK. | Review |
| DOD-T30 | CI hijau sebelum merge. | Required status check |
| DOD-T31 | Review manusia (HA-06) disetujui sebelum merge. | Persetujuan pull request |

---

## 2. DoD tingkat Fase

Sebuah fase selesai hanya jika seluruh quality gate fase tersebut lulus **dan** seluruh task di
dalamnya memenuhi DoD tingkat task.

| Fase | Metrik | Kriteria lulus |
| --- | --- | --- |
| 0 — Charter | Open question material | 0 sebelum Fase 1 dimulai |
| | Tool coverage | 100% |
| | Verifikasi API nyata | CORS, kepadatan data, bentuk tuple, dan biaya credit dibuktikan dengan panggilan nyata |
| | Reproducibility | Anggota lain berhasil menjalankan setup dari nol tanpa kredensial |
| 1 — Requirement | Completeness | ≥ 90% elemen wajib terisi |
| | Testability | 100% requirement punya hasil teramati |
| | Ambiguity severity tinggi terbuka | 0 |
| | Scope integrity | 0 requirement yang tidak berasal dari charter |
| | Kesesuaian data | 0 requirement yang mengasumsikan data di luar yang disediakan penyedia |
| 2 — Executable spec | REQ/BR coverage | 100% punya scenario |
| | Scenario validity | 100% lolos parser |
| | Observable outcome | 100% Then dapat diverifikasi |
| | Negative coverage | 100% journey punya skenario kegagalan |
| 3 — Prototype & UI | Journey coverage | 100% dapat disimulasikan |
| | State coverage | 100% state wajib tersedia |
| | A11y critical/serious terbuka | 0 |
| | Invented UI behaviour | 0 |
| 4 — Arsitektur & kontrak | Contract validity | 0 error dari linter OpenAPI |
| | Kebersihan abstraksi | 0 istilah penyedia pada kontrak internal |
| | Action coverage | 100% aksi UI punya jalur data |
| | Example coverage | 100% operasi punya contoh request/response |
| | Kesesuaian skema terhadap data nyata | Skema Zod tuple lolos terhadap seluruh 29 record fixture |
| | Schema reproducibility | Migrasi dari database kosong + seed berhasil |
| | ADR | ADR-001 sampai ADR-005 berstatus Accepted |
| 5 — Perencanaan | Task coverage | 100% REQ/API/DATA punya task |
| | Orphan work item | 0 |
| | Dependency cycle | 0 |
| | DoD per task | 100% task punya acceptance + kewajiban test |
| 6 — Implementasi | Build dari checkout bersih | Pass |
| | Lint / type error | 0 |
| | Pelanggaran batas abstraksi | 0 |
| | Traceability komponen | 100% terhubung ke TASK/REQ |
| | Komentar review severity tinggi terbuka | 0 |
| | Jumlah koreksi manusia | Dilaporkan (tidak harus nol) |
| 7 — Verifikasi integrasi | Contract conformance | 100% operasi lolos validasi terhadap OpenAPI |
| | Coverage indeks tuple | 100% indeks yang dikonsumsi punya assertion tersendiri |
| | Penanganan kasus khusus | `states: null`, callsign string kosong, record tanpa koordinat, 429, dan kegagalan upstream masing-masing punya test |
| | Ekuivalensi provider | `FixtureProvider` dan `OpenSkyProvider` menghasilkan keluaran domain identik untuk masukan setara |
| | Budget guard | Pagu harian dan batas refresh manual diuji dan ditegakkan |
| | Migration test | Database kosong → skema terbaru → seed, berhasil |
| | Test isolation | Lulus pada urutan eksekusi acak |
| 8 — Testing | Acceptance coverage | 100% critical scenario terotomasi |
| | Pass rate | 100% pada release candidate |
| | Flaky rate | 0% |
| | Changed-code coverage | ≥ 80% |
| | Mutation score `server/src/domain` | Mencapai target yang ditetapkan dan dijelaskan |
| | Konsumsi credit selama test | 0 |
| | Security / a11y critical terbuka | 0 |
| 9 — CI quality gate | Pipeline hijau pada kode benar | Pass |
| | False green pada fault injection | 0 — FI-01 sampai FI-05 seluruhnya merah |
| | Required checks | 100% gate terdefinisi menjadi required |
| | Kemandirian CI | Pipeline berjalan tanpa satu pun secret |
| | Ketersediaan laporan | Setiap kegagalan menghasilkan artefak diagnostik |
| | Durasi pipeline | Diukur dan dijelaskan |

---

## 3. DoD tingkat Rilis / Demo

| ID | Kriteria | Bukti |
| --- | --- | --- |
| DOD-R01 | Setup berhasil dari checkout bersih pada mesin yang belum pernah menjalankan proyek, tanpa kredensial dan tanpa jaringan. | Rekaman / log setup |
| DOD-R02 | `npm run db:reset` membangun database dari nol melalui migration dan seed. | Demo langsung |
| DOD-R03 | Satu requirement dapat ditelusuri langsung sampai ke test yang menjalankannya. | Traceability matrix |
| DOD-R04 | Critical user journey berjalan pada aplikasi. | Demo langsung |
| DOD-R05 | Kontrak API internal dan record database yang berubah dapat ditunjukkan. | OpenAPI + inspector + query SQLite |
| DOD-R06 | Seluruh test suite dijalankan dan lulus di depan penguji. | Eksekusi langsung |
| DOD-R07 | Pipeline CI ditunjukkan hijau, lalu merah pada fault injection, lalu hijau kembali. | Riwayat run GitHub Actions |
| DOD-R08 | **Pergantian provider didemonstrasikan** dengan mengubah satu variabel lingkungan, tanpa mengubah kode frontend maupun domain. | Demo langsung — bukti ADR-003 |
| DOD-R09 | Tombol refresh manual ditunjukkan, termasuk perilaku ketika ditekan berulang dan dibatasi server. | Demo langsung |
| DOD-R10 | Perilaku ketika upstream gagal ditunjukkan: snapshot terakhir disajikan beserta penanda usia data. | Demo langsung |
| DOD-R11 | Minimal satu contoh output AI yang salah ditunjukkan, beserta cara mendeteksinya dan koreksi manusianya. | Evidence log |
| DOD-R12 | Aplikasi dapat didemonstrasikan penuh dari fixture, tanpa memakai satu credit pun. | Mode fixture |
| DOD-R13 | Mode `opensky` ditunjukkan setidaknya sekali, dengan konsumsi credit yang terhitung dan ditampilkan. | Demo langsung + panel anggaran |
| DOD-R14 | Seluruh open question pada charter berstatus CLOSED dengan bukti keputusan. | Charter v≥3.1 |
| DOD-R15 | `final-review.md` menyatakan READY beserta alasan berbasis bukti. | Dokumen ditandatangani |

---

## 4. Yang secara eksplisit **bukan** kriteria selesai

Dicantumkan agar tidak menjadi target yang keliru:

1. **Jumlah baris kode.** Assignment menyatakan nilai ditentukan oleh bukti dan traceability,
   bukan volume kode.
2. **Nol koreksi manusia terhadap output AI.** Koreksi wajib dilaporkan, dan jumlah nol justru
   mencurigakan.
3. **Coverage 100%.** Target adalah coverage bermakna pada kode yang diubah, dibuktikan efektif
   melalui mutation testing — bukan angka yang dikejar dengan test kosong.
4. **Banyaknya fitur.** Proyek ini memiliki satu fitur, dan itu disengaja.
5. **Test yang hijau.** Test hijau yang tidak dapat mendeteksi kesalahan adalah kegagalan gate,
   bukan keberhasilan. Itulah sebabnya fault injection bersifat wajib.
6. **Peta yang penuh pesawat.** Kepadatan ditentukan oleh cakupan receiver ADS-B sukarelawan,
   bukan oleh kualitas aplikasi. Peta yang jarang dengan empty state yang jujur lebih baik
   daripada peta yang penuh dengan data yang dikarang.
7. **Backend yang canggih.** BFF ada untuk empat alasan spesifik: menembus CORS, menyimpan
   kredensial, menyatukan biaya, dan mengisolasi penyedia. Kemampuan di luar itu adalah scope
   creep.

---

## 5. Riwayat Perubahan

| Versi | Tanggal | Perubahan |
| --- | --- | --- |
| 1.0 | 2026-08-28 | Versi awal berbasis aviationstack, frontend-only. |
| 2.0 | 2026-08-28 | Disesuaikan untuk OpenSky mode anonim. |
| 3.0 | 2026-08-28 | Disesuaikan untuk arsitektur BFF: (a) bagian 1.2 kriteria arsitektur baru — batas abstraksi, kebersihan istilah penyedia, tuple tidak bocor; (b) DOD-T17 migration wajib disertai test dari database kosong; (c) DOD-T21 error upstream tidak diteruskan mentah; (d) DOD-T23 tidak ada panggilan jaringan keluar selama test; (e) DOD-T30 CI hijau sebelum merge; (f) gate Fase 7 diperluas dengan ekuivalensi provider dan budget guard; (g) gate Fase 9 ditulis ulang untuk CI, termasuk syarat pipeline berjalan tanpa secret; (h) DOD-R02, DOD-R08, DOD-R09, DOD-R10, DOD-R13 ditambahkan pada DoD demo; (i) butir 7 ditambahkan pada bagian 4. |
