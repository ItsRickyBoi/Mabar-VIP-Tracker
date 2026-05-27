# DonoTrack — Tracker Mabar VIP

Web app untuk mencatat data Mabar VIP/VVIP saat live streaming. Terhubung langsung ke Google Spreadsheet melalui Google Apps Script, dengan fitur berbagi daftar game ke penonton secara live.

---

## Fitur

- Input data donatur (Nama, ID Game, Jumlah Donasi, Tier VIP/VVIP)
- Kalkulasi otomatis jumlah game berdasarkan harga per tier
- Edit jumlah game per donatur langsung dari tabel (untuk bonus/promo)
- Ubah tier donatur + recalculate game otomatis
- Sync dua arah dengan Google Spreadsheet — data lama bisa dimuat kembali saat stream berikutnya
- Bagikan daftar game ke penonton via link `viewer.html` (live, auto-refresh 30 detik)
- Penonton bisa cari nama/ML ID mereka sendiri di halaman viewer
- Filter daftar per tier (VIP/VVIP) dan tanggal
- Export CSV sebagai backup
- Mode Lokal — tanpa spreadsheet, data tersimpan sementara di browser
- Konfigurasi tersimpan di browser, tidak perlu setup ulang setiap buka

---

## File

| File | Keterangan |
|------|-----------|
| `index.html` | Halaman utama (streamer) — input donatur, kelola daftar |
| `viewer.html` | Halaman penonton — cek daftar game, bisa dicari per nama/ML ID |

Kedua file harus berada **dalam satu folder/repository** yang sama.

---

## Cara Deploy ke GitHub Pages

1. Buat repository GitHub baru (boleh publik atau privat)
2. Upload `index.html` dan `viewer.html` ke root repository
3. Pergi ke **Settings → Pages**
4. Set Source: **Deploy from a branch → `main` → `/ (root)`**
5. Klik **Save**, tunggu beberapa menit
6. Akses di:
   - Streamer: `https://username.github.io/nama-repo/`
   - Penonton: `https://username.github.io/nama-repo/viewer.html`

> **Tips:** Jika menggunakan repository privat, GitHub Pages tetap bisa diakses publik — data spreadsheet tetap aman karena tidak ditampilkan langsung.

---

## Setup Google Apps Script (Wajib untuk Sync ke Spreadsheet)

### Langkah 1 — Buat Google Spreadsheet

- Buka [Google Sheets](https://sheets.google.com) dan buat spreadsheet baru
- Nama tab sheet bisa dibiarkan **Sheet1** (default) atau ubah sesuai keinginan, catat namanya

### Langkah 2 — Buat Apps Script

1. Di spreadsheet, klik **Extensions → Apps Script**
2. Hapus **semua** kode yang ada di editor
3. Paste seluruh kode berikut:

```javascript
function doGet(e) {
  const action = e.parameter.action || '';
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheetName = e.parameter.sheetName || 'Sheet1';
  const sheet = ss.getSheetByName(sheetName) || ss.getSheets()[0];

  if (action === 'ping') {
    return json({ status: 'ok' });
  }

  if (action === 'getAll') {
    const last = sheet.getLastRow();
    if (last < 2) return json({ success: true, rows: [] });
    const data = sheet.getRange(2, 1, last - 1, 8).getValues();
    const rows = data.map((r, i) => ({
      rowIndex: i + 2,
      name: r[1], mlId: r[2], amount: r[3],
      tier: r[4], games: r[5], timestamp: r[6]
    }));
    return json({ success: true, rows });
  }

  if (action === 'getPublic') {
    const token = e.parameter.token || '';
    const configSheet = ss.getSheetByName('_dontrack_config');
    if (!configSheet) return json({ tokenInvalid: true, sharing: false });

    const storedToken = configSheet.getRange('A1').getValue();
    if (!storedToken || storedToken !== token) {
      return json({ tokenInvalid: true, sharing: false });
    }

    const last = sheet.getLastRow();
    if (last < 2) return json({ success: true, rows: [] });
    const data = sheet.getRange(2, 1, last - 1, 8).getValues();
    const rows = data.map((r, i) => ({
      rowIndex: i + 2,
      name: r[1], mlId: r[2],
      tier: r[4], games: r[5], timestamp: r[6]
    }));
    return json({ success: true, rows });
  }

  return json({ status: 'ok' });
}

function doPost(e) {
  try {
    const body = JSON.parse(e.postData.contents);
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sheetName = body.sheetName || 'Sheet1';
    const sheet = ss.getSheetByName(sheetName) || ss.getSheets()[0];
    const action = body.action || 'append';

    if (action === 'append') {
      if (sheet.getLastRow() === 0) {
        sheet.appendRow(['No', 'Nama Donatur', 'ML ID', 'Jumlah Donasi (Rp)', 'Tier', 'Jumlah Game', 'Waktu']);
      }
      const rowIndex = sheet.getLastRow() + 1;
      sheet.appendRow([sheet.getLastRow(), body.name, body.mlId, body.amount, body.tier, body.games, body.timestamp]);
      return json({ success: true, rowIndex });
    }

    if (action === 'updateGames') {
      sheet.getRange(body.rowIndex, 6).setValue(body.games);
      return json({ success: true });
    }

    if (action === 'deleteRow') {
      sheet.deleteRow(body.rowIndex);
      return json({ success: true });
    }

    if (action === 'updateTier') {
      sheet.getRange(body.rowIndex, 5).setValue(body.tier);
      sheet.getRange(body.rowIndex, 6).setValue(body.games);
      return json({ success: true });
    }

    if (action === 'setShareToken') {
      let configSheet = ss.getSheetByName('_dontrack_config');
      if (!configSheet) {
        configSheet = ss.insertSheet('_dontrack_config');
        configSheet.hideSheet();
      }
      configSheet.getRange('A1').setValue(body.token || '');
      return json({ success: true });
    }

    return json({ success: false, error: 'Unknown action' });
  } catch(err) {
    return json({ success: false, error: err.toString() });
  }
}

function json(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### Langkah 3 — Deploy Apps Script

1. Klik **Deploy → New Deployment**
2. Klik ikon ⚙ di sebelah "Type" → pilih **Web App**
3. Isi pengaturan:
   - **Description:** DonoTracker (bebas)
   - **Execute as:** Me
   - **Who has access:** Anyone
4. Klik **Deploy**
5. Klik **Authorize access** → pilih akun Google → klik **Allow**
6. **Copy URL** yang muncul — bentuknya seperti:
   ```
   https://script.google.com/macros/s/XXXXXXXXXXXXXX/exec
   ```
7. Simpan URL ini, akan dipakai di DonoTracker

> **Penting:** Setiap kali kode Apps Script diubah, harus di-deploy ulang dengan **New Deployment** (bukan "Manage deployments"). URL baru akan berbeda dari sebelumnya.

### Langkah 4 — Hubungkan ke DonoTracker

1. Buka `index.html` di browser
2. Paste URL Apps Script di kolom **Google Apps Script Web App URL**
3. Isi nama sheet jika berbeda dari `Sheet1`
4. Klik **Hubungkan & Muat Data**
5. Jika berhasil, status header akan berubah hijau: **Terhubung · Apps Script**

---

## Cara Pakai (Alur Harian)

### Setup Awal (sekali saja)

1. Hubungkan spreadsheet → set harga per game VIP dan VVIP
2. Konfigurasi tersimpan otomatis di browser — tidak perlu diulang sesi berikutnya

### Saat Live Stream

1. Buka `index.html` — data dari sesi sebelumnya otomatis dimuat dari spreadsheet
2. Saat ada donatur masuk, isi:
   - **Nama Donatur** — nama yang ingin ditampilkan
   - **ID Game** — format `ID game`
   - **Jumlah Donasi** — nominal dalam Rupiah
   - **Tier** — VIP atau VVIP
3. Jumlah game terhitung otomatis berdasarkan harga yang diset
4. Jika ada bonus, sesuaikan jumlah game dengan tombol **+** / **−**
5. Klik **Tambah Donatur** — data langsung masuk ke spreadsheet

### Mengelola Daftar

- **Edit jumlah game:** Klik **+** / **−** pada kolom Game di baris donatur — tersimpan otomatis ke spreadsheet setelah 0.8 detik
- **Ubah tier:** Klik badge **VIP ✎** atau **VVIP ✎** di kolom Tier — game dihitung ulang otomatis
- **Hapus baris:** Klik tombol **✕** di kolom Aksi
- **Filter:** Gunakan tombol Semua / VIP / VVIP dan dropdown tanggal
- **Cari:** Ketik nama atau ML ID di kolom pencarian
- **Refresh:** Klik **↻ Refresh** untuk memuat ulang data terbaru dari spreadsheet

### Setelah Stream

Data tetap tersimpan di spreadsheet. Saat stream berikutnya dibuka, semua data lama muncul kembali secara otomatis.

---

## Fitur Berbagi ke Penonton (Viewer)

Penonton bisa cek sendiri berapa game yang mereka punya tanpa harus bertanya ke streamer.

### Cara Mengaktifkan

1. Di halaman utama (Step 3), klik **Mulai Bagikan** pada panel **Bagikan Daftar ke Penonton**
2. Link viewer akan muncul — copy dan bagikan ke chat stream / bio
3. Penonton buka link tersebut di browser masing-masing

### Yang Bisa Dilakukan Penonton di Viewer

- Lihat seluruh daftar donatur beserta jumlah game
- Cari nama sendiri atau ML ID menggunakan kolom pencarian
- Filter tampilan per tier VIP / VVIP
- Data diperbarui otomatis setiap **30 detik** tanpa perlu refresh manual

### Menghentikan Berbagi

Klik **⬛ Stop Berbagi** — link lama langsung tidak bisa diakses. Saat berbagi dimulai lagi, link baru akan dibuat.

> Fitur berbagi membutuhkan koneksi Apps Script (tidak tersedia di Mode Lokal).

---

## Mode Lokal (Tanpa Spreadsheet)

Cocok untuk penggunaan sederhana tanpa ingin setup Apps Script, atau sebagai cadangan saat koneksi bermasalah.

### Cara Mengaktifkan

Di halaman Step 1, klik **Mode Lokal** — tidak perlu memasukkan URL apapun.

### Cara Kerja

- Data donatur tersimpan **sementara di memori browser** selama tab masih terbuka
- Saat tab ditutup atau browser di-refresh, **data hilang** — tidak ada persistensi
- Konfigurasi harga (VIP/VVIP) tetap tersimpan di localStorage dan akan muncul kembali saat dibuka lagi

### Backup Data di Mode Lokal

Sebelum menutup browser, klik **⬇ CSV** untuk mengunduh seluruh data sesi sebagai file CSV. File ini bisa dibuka di Excel atau Google Sheets.

### Keterbatasan Mode Lokal

| Fitur | Mode Lokal | Mode Apps Script |
|-------|-----------|-----------------|
| Input donatur | ✅ | ✅ |
| Edit jumlah game | ✅ (lokal) | ✅ (sync ke sheet) |
| Data tersimpan permanen | ❌ | ✅ |
| Muat data sesi sebelumnya | ❌ | ✅ |
| Berbagi ke penonton | ❌ | ✅ |
| Export CSV | ✅ | ✅ |

---

## Struktur Data Spreadsheet

Data disimpan di sheet dengan kolom berikut (baris pertama = header otomatis):

| Kolom | Isi |
|-------|-----|
| A | Nomor urut |
| B | Nama Donatur |
| C | ML ID |
| D | Jumlah Donasi (Rp) |
| E | Tier (VIP / VVIP) |
| F | Jumlah Game |
| G | Waktu input |

Sheet tersembunyi `_dontrack_config` digunakan secara internal untuk menyimpan token berbagi — jangan dihapus saat fitur berbagi aktif.

---

## Troubleshooting

**Tombol "Hubungkan" gagal / tidak bisa terhubung**
- Pastikan Apps Script sudah di-deploy dengan setting **Who has access: Anyone**
- Pastikan URL yang dipaste adalah URL deployment (`/exec`), bukan URL editor script
- Coba deploy ulang dengan **New Deployment**

**Data tidak muncul setelah refresh**
- Klik **↻ Refresh** secara manual
- Pastikan nama sheet di DonoTracker sama persis dengan nama tab di spreadsheet (huruf besar/kecil berpengaruh)

**Donatur berhasil ditambah tapi tidak muncul di spreadsheet**
- Cek apakah Apps Script sudah di-authorize dengan benar
- Buka URL Apps Script langsung di browser — harus muncul `{"status":"ok"}`
- Jika ada perubahan kode script, deploy ulang dengan **New Deployment**

**Link viewer tidak bisa diakses penonton**
- Pastikan **Mulai Berbagi** sudah diklik dan status panel hijau
- Link hanya valid selama sesi berbagi aktif — jika streamer klik Stop, link mati
- Pastikan `viewer.html` ada di folder/repository yang sama dengan `index.html`

---

## Teknologi

- Pure HTML / CSS / JavaScript — tanpa framework, tanpa build step
- Google Apps Script sebagai backend proxy ke Google Sheets
- `localStorage` untuk menyimpan konfigurasi antar sesi
- Tidak ada server eksternal — semua berjalan di browser dan Google
