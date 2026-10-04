# SouthCity Advances Settlement Automation

Automation workflow untuk membaca transaksi kredit dari General Ledger, mencocokkannya dengan Working Paper Advances, menghitung saldo realisasi, membuat executive summary menggunakan DeepSeek, dan menulis hasil ke Google Sheets.

## Output

Workflow menghasilkan spreadsheet dengan sheet berikut:

- `Working_Paper_Result`: hasil monitoring advance, realisasi, saldo, dan status.
- `Match_Detail`: rincian hubungan antara baris Working Paper dan transaksi kredit GL.
- `Unmatched_GL`: transaksi kredit GL yang tidak berhasil dipadankan.
- `Dashboard`: KPI settlement dan executive summary AI.

## Arsitektur workflow

```text
Manual Trigger
  -> Read Binary GL
  -> Extract GL
  -> Read Binary Working Paper
  -> Extract Working Paper
  -> Merge Source Data
  -> Build Source Contract
  -> Match & Reconcile
      |-> Prepare Working Paper Rows -> Working_Paper_Result
      |-> Prepare Match Detail Rows  -> Match_Detail
      |-> Prepare Unmatched GL Rows  -> Unmatched_GL
      `-> Build AI Summary Payload
          -> Prepare DeepSeek Request
          -> DeepSeek - Executive Summary
          -> Prepare Dashboard Row
          -> Dashboard
```

## Matching logic

1. Workflow membaca seluruh transaksi GL.
2. Hanya transaksi dengan nilai `KREDIT` yang diproses sebagai settlement.
3. Kode PO, WO, atau nomor pengajuan diekstrak dari deskripsi menggunakan pola kode transaksi.
4. Jika kode ditemukan, transaksi dicocokkan berdasarkan kode tersebut.
5. Jika tidak ada kode, workflow menggunakan normalisasi deskripsi dan kesamaan frasa/project.
6. Satu advance yang memiliki beberapa voucher settlement menggunakan pendekatan single row:
   - nomor voucher digabung dengan separator koma;
   - nominal settlement dijumlahkan.
7. Saldo dihitung menggunakan rumus:

   ```text
   Balance = Advance Amount - Total Realization Amount
   ```

   Karena itu saldo over-settled secara matematis bernilai negatif. Nilai kelebihan bayar untuk kebutuhan laporan dapat ditampilkan positif menggunakan:

   ```text
   Over-settlement = max(0, Realization Amount - Advance Amount)
   ```

## AI integration

DeepSeek digunakan untuk membuat executive summary dalam Bahasa Indonesia. AI hanya menerima hasil rekonsiliasi yang sudah dihitung oleh matching engine. Angka utama tidak diserahkan kepada AI untuk dihitung ulang.

Parameter utama:

- Endpoint: `https://api.deepseek.com/chat/completions`
- Model: `deepseek-flash`
- Thinking mode: disabled agar jawaban masuk ke `choices[0].message.content`.

API key disimpan sebagai credential di n8n dan tidak disimpan di repository.

## Setup

### Prerequisites

- n8n self-hosted melalui Docker.
- Google OAuth credential untuk Google Sheets.
- DeepSeek API credential dengan Bearer Auth.
- Dua file input:
  - GL settlement dalam format `.xls`.
  - Working Paper dalam format `.xlsx`.

### File input lokal

Untuk workflow development, file tersedia di dalam container pada:

```text
/files/GL - Advances Other - April 2026.xls
/files/Working Paper Advances and Prepayment-Soal.xlsx
```

Pastikan volume Docker memetakan folder lokal ke `/files` dan credential Google Sheets sudah terhubung.

### Import workflow

1. Buka n8n.
2. Import `workflows/southcity-settlement-workflow.json`.
3. Pilih ulang credential Google Sheets jika credential ID berbeda.
4. Simpan workflow.
5. Jalankan dengan Manual Trigger.

## Google Sheets formatting

Nilai nominal tetap dikirim sebagai angka agar bisa dihitung dan difilter. Format Rupiah diterapkan pada spreadsheet:

| Sheet | Kolom | Format |
|---|---|---|
| `Working_Paper_Result` | `F`, `I`, `J` | `Rp #,##0` |
| `Match_Detail` | `I` | `Rp #,##0` |
| `Unmatched_GL` | `G`, `H` | `Rp #,##0` |
| `Dashboard` | `C`, `D`, `E` | `Rp #,##0` |

Kolom `Dashboard!J:L` menggunakan text wrapping agar executive summary, outstanding items, dan over-settled items mudah dibaca.

## Baseline result

Dengan dataset acuan April 2026, hasil rekonsiliasi yang diharapkan:

```text
Total Advance       : Rp570.406.879
Total Realisasi     : Rp466.602.663
Outstanding Positif : Rp107.104.216
Settled             : 13
Partial             : 1
Over-settled        : 1
Unmatched GL        : 15
```

Item partial utama adalah PBB JV 2 Summarecon. Workflow juga menandai transaksi TP01/PO/26010003 sebagai over-settled.

## Repository structure

```text
.
├── workflows/
│   └── southcity-settlement-workflow.json
├── docs/
│   └── test-cases.md
├── .env.example
├── .gitignore
└── README.md
```

## Security

Jangan commit file atau nilai berikut:

- `.env` asli;
- DeepSeek API key;
- Google OAuth client secret;
- `N8N_ENCRYPTION_KEY`;
- credential secret;
- file transaksi asli jika mengandung data sensitif.

## Pemanfaatan AI Coding Assistant

AI Coding Assistant digunakan sebagai partner pengembangan untuk:

- menyusun struktur workflow n8n;
- membantu membuat dan mereview Code Node JavaScript;
- membantu debugging HTTP request DeepSeek dan JSON body;
- memeriksa edge case multi-voucher, partial settlement, unmatched transaction, dan over-settlement;
- menyusun dokumentasi dan test case.

Seluruh hasil AI tetap diuji menggunakan output aktual workflow, rekonsiliasi angka, dan pengecekan manual di Google Sheets.

## Limitation

- Workflow mengasumsikan struktur kolom input GL dan Working Paper mengikuti template soal.
- File dengan struktur kolom berbeda memerlukan mapping tambahan.
- Hasil `Append` di Google Sheets akan menambahkan baris baru pada setiap run. `runId` digunakan untuk membedakan hasil antar-run.
