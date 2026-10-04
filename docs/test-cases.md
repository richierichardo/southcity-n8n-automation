# SouthCity Settlement Automation Test Cases

Dokumen ini digunakan untuk memvalidasi matching engine, perhitungan saldo, output Google Sheets, dan integrasi DeepSeek.

## Test case 1 — Baseline dataset

### Input

- `GL - Advances Other - April 2026.xls`
- `Working Paper Advances and Prepayment-Soal.xlsx`

### Expected result

| Metric | Expected |
|---|---:|
| Working Paper rows | 15 |
| Total advance | Rp570.406.879 |
| Total realization | Rp466.602.663 |
| Positive outstanding | Rp107.104.216 |
| Settled rows | 13 |
| Partial rows | 1 |
| Over-settled rows | 1 |
| Matched GL transactions | 24 |
| Unmatched GL transactions | 15 |

### Validation

- Semua 15 baris Working Paper menghasilkan output.
- Kolom realization terisi untuk baris yang memiliki settlement.
- PBB JV 2 Summarecon berstatus `PARTIAL` dengan saldo Rp107.104.216.
- TP01/PO/26010003 berstatus `OVER_SETTLED`.
- Executive summary muncul di `Dashboard`.

## Test case 2 — Multi-voucher settlement

### Purpose

Memastikan satu advance yang diselesaikan oleh beberapa voucher tidak membuat duplicate Working Paper row.

### Expected result

- Satu Working Paper row tetap menjadi satu row output.
- Nomor voucher digabung dengan separator koma.
- Nominal settlement merupakan total seluruh voucher.
- Saldo dihitung dari total realization.
- Rincian per voucher tetap tersedia di `Match_Detail`.

## Test case 3 — PO/code matching

### Purpose

Memastikan transaksi dengan kode PO atau WO dicocokkan berdasarkan kode, bukan hanya berdasarkan deskripsi lengkap.

### Expected result

- Kode seperti `TP01/PO/26010005` dan `HLJC/PO/26030003` menghasilkan match yang benar.
- Perbedaan tambahan teks setelah kode tidak membatalkan matching.
- `matchMethod` mencatat metode matching yang digunakan.

## Test case 4 — Description matching

### Purpose

Memastikan transaksi tanpa kode PO tetap dapat dicocokkan berdasarkan frasa project.

### Expected result

- Deskripsi dinormalisasi menjadi lowercase.
- Spasi dan tanda baca tidak mengganggu matching.
- Hanya deskripsi dengan kesamaan yang cukup kuat yang dicocokkan.
- Transaksi yang tidak memenuhi threshold tetap masuk `Unmatched_GL`.

## Test case 5 — Partial settlement

### Purpose

Memastikan advance yang baru direalisasikan sebagian memiliki saldo positif.

### Expected result

```text
Balance = Advance Amount - Realization Amount
```

Status menjadi `PARTIAL` apabila realization lebih besar dari nol tetapi masih lebih kecil dari advance.

## Test case 6 — Over-settlement

### Purpose

Memastikan settlement yang lebih besar dari advance ditandai sebagai exception.

### Expected result

- Status menjadi `OVER_SETTLED`.
- `balance` signed tetap negatif karena mengikuti rumus akuntansi:

  ```text
  Advance Amount - Realization Amount
  ```

- Nilai kelebihan bayar pada dashboard ditampilkan positif menggunakan nilai absolut atau `overSettlementAmount`.

Contoh:

```text
Advance       : Rp3.300.000
Realization   : Rp6.600.000
Balance       : -Rp3.300.000
Over-settled  : Rp3.300.000
```

## Test case 7 — Unmatched GL

### Purpose

Memastikan transaksi kredit tanpa pasangan tidak hilang dari laporan.

### Expected result

- Transaksi masuk ke `Unmatched_GL`.
- `status` bernilai `UNMATCHED`.
- Nilai `debit` dan `credit` tetap tersedia.
- Dashboard menampilkan jumlah unmatched yang sama dengan jumlah baris di `Unmatched_GL` untuk run yang sama.

## Test case 8 — Missing or invalid source structure

### Purpose

Memastikan workflow tidak menghasilkan laporan seolah-olah berhasil ketika input salah.

### Scenario

- Working Paper tidak memiliki kolom Description.
- GL tidak memiliki kolom KREDIT-IDR.
- File kosong.
- File bukan `.xls` atau `.xlsx` yang didukung.

### Expected result

- Workflow gagal secara jelas atau menghasilkan validation error.
- Tidak menulis hasil parsial ke Google Sheets.
- Error dapat ditemukan pada execution detail n8n.

## Acceptance checklist

- [x] Baseline totals cocok.
- [x] Semua 15 Working Paper rows keluar.
- [x] Multi-voucher tidak duplicate.
- [x] Partial settlement terdeteksi.
- [x] Over-settlement terdeteksi.
- [x] Unmatched GL masuk ke sheet audit.
- [x] Executive summary DeepSeek muncul.
- [x] Nominal di Google Sheets diformat sebagai Rupiah.
- [x] Setiap run memiliki `runId`.
- [x] Tidak ada API key di workflow export atau repository.
