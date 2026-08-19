# Rencana Development SIPRO — Fase 49 (Penutupan Buku, Laporan Owner, Pajak & Kepatuhan)

Problem statement (verbatim):
> "saya ingin anda lanjutkan development dari repo ini https://github.com/luarbinasaaa/sipro …"

Status saat ini (ringkas): lingkungan sudah dipulihkan ke `/app`, baseline hijau (`poc_48.py` PASS, `run_all_gates.sh` 36 gates PASS), dan **Fase 48 sudah ditutup** (E2E + perbaikan).

---

## 1) Objectives
Fokus menutup gap nyata G1–G7 tanpa membangun ulang yang sudah ada:
1. **Year-end closing (G1):** laba/rugi tahun berjalan dipindah ke **Laba Ditahan** via jurnal penutup yang seimbang, idempoten, dan reversible.
2. **Period close bergigi (G2):** penutupan bulan **MENAHAN** bila ada checklist gagal (bank recon, pending approvals, depresiasi, tie-out subledger↔GL, jurnal tidak seimbang, dsb) + override beralasan untuk peran berwenang.
3. **Arus kas per proyek (G3):** laporan cash-flow per proyek dengan tie-out ke konsolidasi.
4. **Paket laporan bulanan owner (G4):** BS/PL/CF + per proyek + rasio + metadata periode (status, siapa menutup, cutoff) + honest-null.
5. **e-Faktur compliance (G5):** ekspor berkas impor e-Faktur per periode + faktur pengganti/batal berjejak.
6. **e-Bupot (G6):** bukti potong bernomor + PDF + ekspor per periode untuk PPh 21/23 (fee mitra) dan PPh final (penjualan bila ada basisnya).
7. **Rekap SPT Masa PPN (G7):** keluaran/masukan/kurang-lebih bayar + status setor + rekonstruksi dari sumber.

---

## 2) Implementation Steps

### FASE 1 — POC Core (WAJIB) 
**Output:** `poc/poc_49.py` hijau (exit 0). Semua fixture dibuat via API resmi (bertanda `poc49`) dan dibersihkan; tidak meninggalkan jurnal/dokumen menggantung.

**User stories (POC):**
1. Sebagai finlead, saat menutup periode, sistem MENAHAN penutupan bila checklist gagal dan menyebut sebab satu per satu.
2. Sebagai finlead, saya bisa override penutupan dengan alasan ≥10 huruf; tercatat audit + melahirkan tugas tinjauan.
3. Sebagai owner, saya menutup TAHUN: laba tahun berjalan pindah ke laba ditahan via jurnal seimbang dan tidak bisa dobel.
4. Sebagai owner, bila periode dibuka kembali, jurnal penutup tahun dibalik (reversible) dengan jejak.
5. Sebagai owner, arus kas per proyek dapat dijumlahkan dan sama dengan arus kas konsolidasi (ada “tanpa proyek” yang jujur).
6. Sebagai finance, ekspor e-Faktur DITOLAK bila ada faktur wajib-NPWP yang belum lengkap dan menyebut faktur mana.
7. Sebagai finance, bukti potong tidak bisa terbit dua kali untuk potongan yang sama; PDF bisa dibuat.

**Langkah POC:**
- P1. Websearch singkat: praktik year-end closing (retain earnings), kontrol period close checklist, format impor e-Faktur, dan praktik e-Bupot (nomor + pembatalan).
- P2. Buat periode uji: jurnal pendapatan/beban + pembayaran kas lintas 2 proyek + 1 transaksi tanpa proyek.
- P3. Uji `POST /gl/periods/close` versi baru: buat kondisi gagal (mis. “bank belum direkonsiliasi” via fixture) → harus HOLD 409/400 dengan daftar `reasons[]`.
- P4. Uji override (finlead/owner saja): close dengan `override=true` + `override_reason` ≥10 → close sukses + audit_log + task review.
- P5. Uji year-end close: panggil `POST /gl/year/close` dua kali → kedua kalinya idempoten (tidak membuat jurnal baru).
- P6. Uji reopen year-end: `POST /gl/year/reopen` → jurnal reversal tercipta, jejak terpaut.
- P7. Uji cashflow per project report + tie-out.
- P8. Uji e-Faktur export guard + bukti potong issuance+PDF.

> Stop point: jangan lanjut Fase 2 sebelum `poc_49.py` PASS.

---

### FASE 2 — V1 App Development (backend + frontend end-to-end)
**Output:** fitur tampil sebagai tab/section pada halaman existing (tanpa pintu sidebar baru):
- `/accounting` (Buku Besar & Jurnal) → tab **Penutupan Buku**
- `/accounting-reports` (Laporan Keuangan) → tab **Paket Laporan Owner** + **Arus Kas per Proyek**
- `/tax` (Perpajakan) → tab **e-Faktur Ekspor** + **Bukti Potong (e-Bupot)** + **Rekap SPT Masa PPN**

**49A — Period Close Checklist + Hold/Override**
- Backend: modul `closing_engine.py` (checklist evaluators + reasons[]), endpoint:
  - `GET /gl/periods/close-check?period=YYYY-MM`
  - upgrade `POST /gl/periods/close` untuk menahan + override beralasan (gl:manage)
- Frontend: panel checklist (status per item, CTA ke halaman sumber), tombol Close (disabled bila gagal), dialog override (alasan ≥10).

**49B — Year-end Closing**
- Backend:
  - `POST /gl/year/close?year=YYYY` (idempoten; jurnal Dr/Cr menutup semua akun P&L ke retained earnings)
  - `POST /gl/year/reopen?year=YYYY` (buat jurnal balik; tidak hapus)
  - metadata tersimpan di `gl_year_closings`
- Frontend: panel “Tutup Tahun” (owner/finlead), riwayat, dan tombol reopen (owner).

**49C — Cash Flow per Proyek + Tie-out**
- Backend: `GET /gl/reports/cash-flow-projects?date_from&date_to` mengembalikan:
  - per project buckets + `unassigned` + `consolidated` + proof tie-out.
- Frontend: tabel per proyek + baris total, filter periode.

**49D — Paket Laporan Bulanan Owner**
- Backend: `GET /gl/reports/owner-pack?period=YYYY-MM` (BS/PL/CF + ratios + project P/L + cashflow projects + metadata penutupan).
- Frontend: satu layar ringkas + tombol PDF bundle (opsional v1: multi-link PDF yang sudah ada).

**49E — e-Faktur v2 (pengganti/batal + ekspor)**
- Backend:
  - `POST /tax/faktur/{id}/replace` (pengganti)
  - `POST /tax/faktur/{id}/cancel` (batal, alasan wajib)
  - `GET /tax/faktur/export?period=YYYY-MM` → hasilkan berkas CSV sesuai skema impor e-Faktur + menahan bila data wajib kurang.
- Frontend: daftar faktur + aksi pengganti/batal + halaman ekspor (unduh).

**49F — e-Bupot (bukti potong)**
- Backend:
  - koleksi `withholding_docs` (nomor, jenis, pihak, basis, tarif, nilai, ref pembayaran)
  - `POST /tax/withholding/issue` (idempoten per payment/ref)
  - `GET /tax/withholding?period=` + `GET /tax/withholding/{id}/pdf`
  - `GET /tax/withholding/export?period=`
- Frontend: tab bukti potong (list, filter periode, PDF, ekspor).

**49G — Rekap SPT Masa PPN**
- Backend: `GET /tax/vat-return?period=` (keluaran/masukan/kurang-lebih bayar + sumber).
- Frontend: kartu ringkas + link ke daftar faktur/ppn-input.

---

### FASE 3 — SSOT + Seed + RBAC + Gates + Mutasi + Penutupan
**Output:** seed demo `fase49` + guardrail + uji-mutasi + E2E multi-peran.

**User stories (QA/Governance):**
1. Close period MENAHAN bila checklist gagal; override hanya peran berwenang + alasan ≥10.
2. Year-end close idempoten; reopen membuat reversal berjejak.
3. Cash flow per proyek tie-out ke konsolidasi.
4. Ekspor e-Faktur menolak faktur wajib-NPWP yang belum lengkap dengan daftar yang kurang.
5. Bukti potong tidak bisa terbit dobel; nilai = tarif × dasar dan sama dengan yang dipotong.

**Langkah:**
- S1. SSOT reference: grup baru untuk `closing_check_item`, `withholding_kind`, `vat_return_state`.
- S2. Seed idempoten `seed_phase49.py` (`demo_batch="fase49"`):
  - 2 proyek + 1 transaksi tanpa proyek untuk uji arus kas
  - 1 periode dengan kondisi checklist gagal (mis. bank un-reconciled) + 1 yang bersih
  - 1 faktur siap ekspor + 1 faktur “wajib NPWP tapi kosong” untuk uji hold
  - 1 pembayaran fee mitra yang melahirkan bukti potong
- S3. RBAC: aksi baru `gl:close_override`, `gl:year_close`, `tax:export`, `tax:withholding_issue`.
- S4. Gate baru + daftar ke `scripts/run_all_gates.sh`:
  - `scripts/verify_closing.py` (gate 37)
  - `scripts/verify_tax_compliance.py` (gate 38)
- S5. `scripts/mutasi_49.py` (16–24 mutasi) mematikan guard: hold/override, year-close idempoten, tie-out cashflow, ekspor pajak.
- S6. Update dok: `docs/v2/43_CLOSING_TAX_COMPLIANCE_SPEC.md`, `CODEBASE_MAP.md`, `test_result.md`, `memory/test_credentials.md`, `plan.md`.
- S7. Penutupan: testing_agent_v3 E2E multi-peran (owner, finlead, finance, pm, sales).

---

## 3) Next Actions
1. Buat `poc/poc_49.py` + fixture helper + cleanup/settle hingga PASS.
2. Implement backend minimal untuk core: `closing_engine` + year-close + cashflow-projects + tax exports + withholding docs.
3. Pasang UI minimal di tab existing: /accounting, /accounting-reports, /tax.
4. Tambah seed fase49 + 2 gates + mutasi_49 + jalankan `bash scripts/run_all_gates.sh`.
5. Minta testing agent E2E multi-peran untuk menutup Fase 49.

---

## 4) Success Criteria
- `python3 poc/poc_49.py` → **PASS** (tidak meninggalkan jurnal/dokumen menggantung).
- Close period: HOLD + override beralasan (audit + task) berjalan; non-privileged 403.
- Year-end close: jurnal seimbang, idempoten; reopen membuat reversal berjejak.
- Cash flow per proyek tie-out: Σ(project + unassigned) == consolidated.
- e-Faktur: ekspor per periode tersedia; pengganti/batal berjejak; ekspor menolak data wajib yang kurang.
- e-Bupot: bukti potong bernomor + PDF + ekspor; tidak dobel; tie-out ke potongan nyata.
- Rekap SPT PPN periode dapat direkonstruksi.
- Gates: `bash scripts/run_all_gates.sh` → **OVERALL PASS (38 gates)**.
- `python3 scripts/mutasi_49.py` → semua mutasi **TERTANGKAP**.

---

## Fase 50 (disiapkan setelah Fase 49 ditutup)
- **PWA offline terpadu** untuk absensi (Fase 47) + progres + foto dalam satu antrean sinkron.
- **Serah terima unit**: BAST unit, masa garansi, klaim garansi pasca-huni terhubung punch list & komplain CS.
