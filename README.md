# DonTrack — ML Donation Tracker

Web app untuk mencatat data donatur Mobile Legend saat live streaming. Terhubung langsung ke Google Spreadsheet melalui Google Apps Script.

## Fitur
- Input data donatur (Nama, ML ID, Jumlah, Tier VIP/VVIP)
- Kalkulasi otomatis jumlah game berdasarkan harga yang diset
- Edit jumlah game (untuk bonus/promo)
- Kirim data langsung ke Google Spreadsheet
- Export CSV sebagai backup
- Menyimpan konfigurasi di browser (tidak perlu setup ulang)

---

## Cara Deploy ke GitHub Pages

1. Fork atau upload `index.html` ke repository GitHub baru
2. Pergi ke **Settings → Pages**
3. Set Source ke **Deploy from a branch → main → / (root)**
4. Akses di `https://username.github.io/nama-repo`

---

## Setup Google Apps Script (Wajib untuk Sync ke Spreadsheet)

### 1. Buat Google Spreadsheet
- Buat spreadsheet baru di [Google Sheets](https://sheets.google.com)
- Beri nama sheet tab: `Sheet1` (atau sesuai keinginan)

### 2. Buat Apps Script
- Di spreadsheet, klik **Extensions → Apps Script**
- Hapus semua kode yang ada
- Paste kode berikut:

```javascript
function doGet(e) {
  const action = e.parameter.action || '';
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheetName = e.parameter.sheetName || 'Sheet1';
  const sheet = ss.getSheetByName(sheetName) || ss.getSheets()[0];

  if (action === 'ping') {
    return json({status:'ok'});
  }

  if (action === 'getAll') {
    const last = sheet.getLastRow();
    if (last < 2) return json({success:true, rows:[]});
    const data = sheet.getRange(2, 1, last - 1, 8).getValues();
    const rows = data.map((r,i) => ({
      rowIndex: i + 2,
      name: r[1], mlId: r[2], amount: r[3],
      tier: r[4], games: r[5], timestamp: r[6]
    }));
    return json({success:true, rows});
  }

  return json({status:'ok'});
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
        sheet.appendRow(['No','Nama Donatur','ML ID','Jumlah Donasi (Rp)','Tier','Jumlah Game','Waktu']);
      }
      const rowIndex = sheet.getLastRow() + 1;
      sheet.appendRow([sheet.getLastRow(), body.name, body.mlId, body.amount, body.tier, body.games, body.timestamp]);
      return json({success:true, rowIndex});
    }

    if (action === 'updateGames') {
      sheet.getRange(body.rowIndex, 6).setValue(body.games);
      return json({success:true});
    }

    if (action === 'deleteRow') {
      sheet.deleteRow(body.rowIndex);
      return json({success:true});
    }

    return json({success:false, error:'Unknown action'});
  } catch(err) {
    return json({success:false, error:err.toString()});
  }
}

function json(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### 3. Deploy Apps Script
- Klik **Deploy → New Deployment**
- Type: **Web App**
- Execute as: **Me**
- Who has access: **Anyone**
- Klik **Deploy** dan **Authorize**
- Copy URL yang muncul (bentuknya: `https://script.google.com/macros/s/...`)

### 4. Hubungkan ke DonTrack
- Buka DonTrack di browser
- Paste URL Apps Script di kolom URL
- Klik **Hubungkan Spreadsheet**

---

## Mode Lokal (Tanpa Apps Script)
Jika tidak ingin setup Apps Script, klik **Gunakan Mode Lokal**. Data akan disimpan sementara di browser dan bisa di-export sebagai CSV setiap sesi.

---

## Alur Penggunaan

1. **Setup awal** (sekali): Hubungkan spreadsheet + set harga VIP/VVIP
2. **Saat live**: Input nama, ML ID, jumlah donasi, pilih tier
3. **Kalkulasi otomatis**: Jumlah game langsung terhitung
4. **Edit jika ada bonus**: Tambah/kurangi game pakai tombol +/−
5. **Submit**: Data langsung masuk ke Google Spreadsheet

---

## Teknologi
- Pure HTML/CSS/JavaScript (no framework, no build step)
- Google Apps Script sebagai backend proxy ke Google Sheets
- LocalStorage untuk menyimpan konfigurasi