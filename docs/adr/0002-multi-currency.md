# ADR 0002: Klaim multi-mata-uang

- **Status:** Proposed — belum disetujui
- **Tanggal:** 2026-11-25
- **Issue:** [#1 — Epik multi-currency](https://github.com/sandy-demo/expense-app-full/issues/1)
- **Skenario:** S15

## Konteks

Karyawan perlu mengajukan klaim perjalanan dalam IDR, USD, atau SGD. Nilai
klaim harus dapat dibandingkan dan diproses memakai aturan bisnis yang saat ini
dinyatakan dalam rupiah. Baseline menetapkan klaim di bawah Rp1.000.000 perlu
Manager, sedangkan klaim Rp1.000.000 atau lebih perlu Manager lalu Finance
([aturan bisnis](../business-rules.md)). Perubahan ini adalah usulan desain;
baseline dan aturan yang berlaku tidak berubah sebelum Tech Lead menyetujui ADR
dan implementasinya ditinjau.

## Keputusan yang diusulkan

### Kurs untuk lab

Gunakan **fixture kurs tetap dan fiktif**, bukan layanan kurs eksternal:

| Mata uang klaim | Unit | Kurs fixture |
|---|---:|---:|
| IDR | 1 IDR | Rp1 |
| USD | 1 USD | Rp15.000 |
| SGD | 1 SGD | Rp11.000 |

Angka tersebut hanya data ilustrasi untuk lab, bukan kurs aktual, nasihat
keuangan, atau data yang layak dipakai untuk transaksi nyata. Simpan fixture
sebagai satu versi bernama, misalnya `lab-fx-2026-01`, dengan waktu efektif
tetap `2026-01-01T00:00:00Z` dan sumber tercatat `lab-fixed-fixture` (nilai
fiktif yang ditetapkan untuk demo). Jangan mengambil atau menyegarkan kurs dari
jaringan. Perubahan fixture harus menjadi perubahan versi yang dapat ditinjau;
jangan mengubah kurs pada klaim yang sudah dibuat.

Saat klaim diajukan, simpan snapshot kurs yang dipakai bersama klaim:
`currency`, `originalAmountMinor`, `rateToIdr`, `rateSnapshotId`,
`rateEffectiveAt` (UTC ISO 8601), dan `rateSource`. Simpan pula `amountIdr`
hasil konversi sebagai bilangan bulat. Klaim IDR memakai kurs 1:1 dan tetap
mencatat snapshot serta sumber yang sama agar bentuk audit konsisten. Persetujuan
selalu memakai snapshot klaim itu, bukan versi fixture terkini.

### Presisi dan pembulatan

Simpan nominal tanpa floating point. `originalAmountMinor` adalah bilangan
bulat: untuk IDR nilainya rupiah utuh; untuk USD dan SGD nilainya sen. Kurs
`rateToIdr` dinyatakan sebagai rupiah per satu unit mata uang asing (bilangan
bulat pada fixture ini). Konversikan dengan aritmetika integer:

`amountIdr = roundHalfUp(originalAmountMinor × rateToIdr ÷ minorUnitsPerUnit)`

`minorUnitsPerUnit` adalah 1 untuk IDR dan 100 untuk USD/SGD. Pembulatan
`roundHalfUp` dilakukan sekali ke rupiah terdekat; tepat setengah dibulatkan
menjauh dari nol. Nominal harus positif dan memenuhi batas validasi yang
disetujui untuk setiap mata uang. Jumlah dan kurs di luar rentang aman harus
ditolak, bukan diproses dengan presisi yang kehilangan nilai.

### Approval, tampilan, dan laporan

- Terapkan aturan Manager dan Finance yang ada pada `amountIdr`: di bawah
  Rp1.000.000 memerlukan Manager; Rp1.000.000 atau lebih memerlukan Manager
  lalu Finance. Jangan membuat ambang terpisah dalam USD atau SGD.
- API klaim menerima mata uang dan nominal asal, lalu respons mengembalikan
  mata uang, nominal asal, `amountIdr`, dan identitas snapshot kurs yang
  digunakan. Bentuk akhir kontrak, validasi batas nominal, serta perilaku
  endpoint harus disepakati sebelum implementasi dan dicatat di `docs/api.md`.
- UI pengajuan menyediakan pilihan IDR/USD/SGD dan menampilkan estimasi IDR.
  Detail klaim dan antrean approval menampilkan nominal asal serta nilai IDR
  agar dasar keputusan terlihat. Laporan Finance tetap menjumlahkan IDR untuk
  perbandingan lintas mata uang, dengan mata uang asal tersedia untuk penelusuran
  transaksi.

### Audit dan privasi

Simpan mata uang, nominal asal, `amountIdr`, versi/sumber/waktu snapshot, dan
nilai IDR yang dipakai sebagai dasar keputusan pada rekam klaim atau event audit
yang tidak dapat ditimpa. Catat keputusan Manager dan Finance, pelaku, waktu,
serta perubahan status dalam transaksi database yang sama; penolakan atau
persetujuan tidak boleh kehilangan snapshot kurs. Log aplikasi hanya mencatat
metadata non-PII sesuai konvensi repository, bukan isi klaim atau data pribadi.

## Alternatif yang dipertimbangkan

1. **Kurs eksternal saat pengajuan.** Lebih mendekati kurs pasar, tetapi
   menambah dependensi jaringan, kegagalan dan latensi, pengelolaan kredensial,
   perubahan hasil antarwaktu, serta kebutuhan audit atas penyedia dan waktu
   kuotasi. Tidak dipilih untuk lab yang harus deterministik dan aman untuk
   dijalankan tanpa kredensial.
2. **Kurs terkini saat persetujuan atau laporan.** Mudah menyegarkan angka,
   tetapi nilai klaim dan hasil keputusan dapat berubah setelah diajukan.
   Ditolak karena merusak reproduktibilitas serta jejak audit.
3. **Simpan nominal asal saja dan konversi saat laporan.** Mengurangi data
   tersimpan, tetapi tidak memberi dasar approval yang stabil dan dapat
   membuat laporan historis berubah. Tidak dipilih.
4. **Semua klaim dikonversi oleh pengguna ke IDR.** Tidak memerlukan kurs di
   aplikasi, tetapi menghilangkan dukungan mata uang asal dan tidak memenuhi
   kebutuhan epik. Tidak dipilih.

## Rencana kerja dan pemecahan epik

Planner membagi epik menjadi pekerjaan berurutan; setiap item menghasilkan
perubahan terfokus dan tetap di branch/draft PR sampai gate terkait lulus.

1. **Riset dan perencanaan epik:** petakan skema, service, route, validasi,
   tampilan, approval, laporan, audit, serta test yang ada. Catat dependensi,
   risiko batas presisi, dan issue terpisah untuk ADR, API/UI, tes, dan docs.
2. **ADR (pekerjaan ini):** tetapkan sumber kurs, waktu/snapshot, presisi,
   pembulatan, penggunaan IDR untuk approval, audit, alternatif, dan gate.
   **Gate:** Tech Lead meninjau dan menyetujui ADR melalui review sebelum
   implementasi dimulai. Status dokumen ini tetap Proposed sampai persetujuan
   tersebut.
3. **API dan layanan:** setelah ADR disetujui, rancang validasi mata uang dan
   nominal di batas API; konversi serta keputusan bisnis di service; simpan
   snapshot dan audit secara atomik menggunakan prepared statements. Tetapkan
   respons/error dan batas nominal sebelum mengubah kontrak.
4. **UI:** tambahkan pemilihan mata uang dan tampilan nilai asal/IDR pada
   pengajuan, detail, approval, serta laporan. Uji perubahan alur pengguna.
5. **Tes:** tambahkan unit, API, dan e2e sesuai cakupan di bawah. Gunakan clock
   tetap untuk tes waktu agar hasil stabil lintas zona waktu.
6. **Dokumentasi:** perbarui `docs/api.md` setelah kontrak final; selaraskan
   aturan bisnis hanya setelah perubahan disetujui; dokumentasikan keputusan
   implementasi lanjutan bila diperlukan.
7. **Handoff:** coding agent mengirim draft PR pada branch fitur di repo
   turunan, dengan ringkasan keputusan, risiko, dan hasil tes. Minta review
   Tech Lead, review board, dan CodeQL; perbaiki temuan dan tunggu CI, semua
   review yang diwajibkan, serta dokumentasi lengkap. Hanya Tech Lead yang
   boleh menyetujui dan melakukan merge setelah seluruh gate lulus. Agent tidak
   menyetujui atau melakukan merge.

## Kriteria tes penerimaan

- **IDR:** kurs 1:1 mempertahankan nilai bulat; nilai tepat di bawah
  Rp1.000.000 hanya memerlukan Manager, sedangkan tepat Rp1.000.000 dan di
  atasnya memerlukan Manager lalu Finance.
- **USD:** nominal sen memakai fixture USD, menghasilkan `amountIdr` integer
  dengan aturan half-up; kasus pecahan tepat setengah dan nilai di sekitar
  ambang Rp1.000.000 menguji pembulatan serta routing approval.
- **SGD:** nominal sen memakai fixture SGD dan diuji dengan pola konversi,
  pembulatan, serta ambang yang sama.
- **Semua mata uang:** mata uang tak dikenal, nol/negatif, presisi berlebih,
  nilai yang melampaui rentang aman, atau input non-integer ditolak pada batas
  API; tidak ada floating-point drift.
- **Snapshot/audit:** perubahan versi fixture tidak mengubah klaim lama;
  detail klaim dan keputusan approval tetap menunjuk snapshot yang sama;
  kegagalan transaksi tidak meninggalkan klaim tanpa audit atau sebaliknya.
- **API/UI/laporan:** respons menyertakan nilai asal dan IDR; UI menunjukkan
  keduanya; agregat laporan menggunakan IDR dan dapat ditelusuri ke mata uang
  asal.

## Konsekuensi dan risiko

- Fixture statis memberi hasil lokal dan CI yang deterministik serta tidak
  memerlukan kredensial atau koneksi eksternal, tetapi bukan kurs pasar.
  Labeli data sebagai ilustrasi dan batasi penggunaannya pada lab.
- Snapshot menambah data yang disimpan, namun mempertahankan reproducibility
  keputusan dan laporan.
- Pembulatan integer dan validasi batas harus konsisten pada service, API,
  tampilan, dan laporan; keputusan batas nominal maksimal lintas mata uang
  merupakan prasyarat implementasi.
- Penggunaan `amountIdr` menjaga aturan limit yang ada, tetapi memerlukan
  tampilan nilai konversi yang jelas agar pengaju dan approver memahami dasar
  keputusan.
- Sebelum penggunaan nyata, ADR baru perlu membahas penyedia kurs, lisensi dan
  ketersediaan, waktu kuotasi, refresh, fallback, kredensial, rekonsiliasi,
  kontrol audit, serta kepatuhan. Fixture lab tidak boleh dipromosikan menjadi
  layanan produksi.

## Gate handoff

Dokumen ini adalah usulan untuk review, bukan persetujuan desain. Berhenti
setelah draft ADR dan minta Tech Lead meninjau issue #1; jangan mulai perubahan
implementasi sebelum ADR disetujui. Setelah handoff, perubahan tetap berupa
draft PR di repo turunan. Jangan menyetujui ADR/PR, merge, mengubah workflow,
hooks, atau settings sebagai bagian dari pekerjaan ini.
