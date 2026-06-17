# Q-Scada (Qoalca Scada)
A web-based SCADA (Supervisory Control and Data Acquisition) HMI builder with real-time Modbus TCP data acquisition, role-based authentication, historical trending, reporting, alarm management, and automated data archival.

# Link Download di Google Drive
https://drive.google.com/drive/folders/1miobxUWhSVi0Ex355RFXXrj-rUbJ7exo?usp=sharing

# Manual Menjalankan Q-Scada Lite

Panduan menjalankan aplikasi **tanpa Docker** (database SQLite berbasis file).

Aplikasi terdiri dari 2 bagian:

| Bagian          | Folder          | Teknologi                                   | Port    |
| --------------- | --------------- | ------------------------------------------- | ------- |
| **Backend**     | `scada-backend` | Node.js (Express + WebSocket + Modbus + MQTT + SQLite) | `3001`  |
| **Frontend**    | `scada-builder` | React (Vite)                                | `5173` (dev) |

Database tersimpan sebagai **satu file**: `db/scada.db` (dibuat otomatis).

---

## 0. Cara Tercepat (double-click)

Tersedia 2 file pintasan di folder utama:

- **`start.bat`** — otomatis meng-install dependency (jika belum), lalu menjalankan
  backend + frontend di dua jendela, dan membuka browser ke http://localhost:3001.
- **`stop.bat`** — menghentikan backend + frontend (proses di port 3001 & 5173).

Cukup **double-click `start.bat`** untuk menjalankan, dan `stop.bat` untuk berhenti.
Login: **admin / Admin123!**

> Detail langkah manual ada di bagian-bagian berikut bila diperlukan.

---

## 1. Prasyarat

- **Node.js 18 atau lebih baru** (disarankan 20/22). Cek versi:
  ```powershell
  node -v
  npm -v
  ```
  Unduh dari https://nodejs.org bila belum ada.

> Tidak perlu menginstall PostgreSQL maupun Docker. Database SQLite sudah tertanam di aplikasi.

---

## 2. Instalasi (sekali saja)

Buka **PowerShell** / **Command Prompt**, lalu install dependency untuk kedua bagian.

**Backend:**
```powershell
cd "d:\Microplus Product 2026\qscada lite\scada-backend"
npm install
```

**Frontend:**
```powershell
cd "d:\Microplus Product 2026\qscada lite\scada-builder"
npm install
```

---

## 3. Menjalankan (mode pengembangan / harian)

Aplikasi butuh **2 jendela terminal** yang berjalan bersamaan.

### Terminal 1 — Backend
```powershell
cd "d:\Microplus Product 2026\qscada lite\scada-backend"
npm start
```
Berhasil jika muncul:
```
[DB] SQLite ready → ...\db\scada.db
[DB] Tables ready
SCADA backend listening on http://localhost:3001
```
Saat pertama kali, akan otomatis dibuat:
- File database `db/scada.db`
- Akun admin default (lihat bagian Login)

### Terminal 2 — Frontend
```powershell
cd "d:\Microplus Product 2026\qscada lite\scada-builder"
npm run dev
```
Buka alamat yang ditampilkan (biasanya **http://localhost:5173**) di browser.

> Dev server Vite otomatis meneruskan permintaan `/api` dan `/ws` ke backend di port 3001, jadi keduanya cukup dijalankan di komputer yang sama.

---

## 4. Login Default

| Field    | Nilai        |
| -------- | ------------ |
| Username | `admin`      |
| Password | `Admin123!`  |

> **Segera ganti password** setelah login pertama (menu Users/Pengguna).

---

## 5. Menjalankan untuk Produksi

Untuk produksi, frontend di-*build* menjadi file statis lalu disajikan oleh web server.

### Langkah 1 — Build frontend
```powershell
cd "d:\Microplus Product 2026\qscada lite\scada-builder"
npm run build
```
Hasilnya ada di folder `scada-builder/dist`.

### Langkah 2 — Jalankan backend
```powershell
cd "d:\Microplus Product 2026\qscada lite\scada-backend"
npm start
```
Agar backend tetap hidup di server, gunakan process manager seperti **PM2**:
```powershell
npm install -g pm2
cd "d:\Microplus Product 2026\qscada lite\scada-backend"
pm2 start server.js --name scada-backend
pm2 save
```

### Langkah 3 — Sajikan frontend
Pilih salah satu cara menyajikan folder `dist`:

**a) Cara cepat (uji coba):**
```powershell
cd "d:\Microplus Product 2026\qscada lite\scada-builder"
npm run preview
```

**b) Pakai Nginx (disarankan untuk produksi):**
File contoh sudah tersedia di `scada-builder/nginx.conf`. File ini:
- menyajikan isi folder `dist` sebagai SPA, dan
- mem-proxy `/api` & `/ws` ke backend (`http://localhost:3001`).

Salin `dist` ke folder root nginx Anda dan pasang `nginx.conf` tersebut.
Jika backend berada di host/port lain, ubah `http://localhost:3001` di dalam `nginx.conf`.

---

## 6. Konfigurasi (opsional)

Buat file `.env` di dalam folder `scada-backend` (lihat contoh `scada-backend/.env.example`):

```ini
# Port backend (default 3001)
PORT=3001

# Lokasi file database SQLite.
# Kosongkan untuk memakai default: <project>/db/scada.db
# SQLITE_PATH=D:/data/scada.db

# Kunci rahasia JWT (WAJIB diganti di produksi)
# JWT_SECRET=ganti_dengan_kunci_acak_panjang
```

---

## 7. Database & Backup

- **Lokasi:** `db/scada.db` (plus file pendamping `scada.db-wal` dan `scada.db-shm` saat berjalan).
- **Backup:** hentikan backend, lalu salin file `db/scada.db`. Itu sudah memuat **seluruh** data (layout, koneksi, log, alarm, pengguna, dll).
- **Reset total:** hentikan backend, hapus `db/scada.db` (beserta `-wal`/`-shm`). Database baru + admin default akan dibuat ulang saat backend dijalankan lagi.
- **Arsip CSV** (fitur archival) tersimpan di `scada-backend/archives/`.

---

## 8. Troubleshooting

| Masalah | Penyebab / Solusi |
| --- | --- |
| `Error: Cannot find module 'better-sqlite3'` | Dependency belum terpasang → jalankan `npm install` di `scada-backend`. |
| `EADDRINUSE: ...:3001` | Port 3001 sudah dipakai proses lain. Tutup proses itu, atau set `PORT` lain di `.env`. |
| Frontend tampil tapi data kosong / error koneksi | Backend belum jalan. Pastikan Terminal 1 (`npm start`) aktif dan menampilkan "listening on http://localhost:3001". |
| Lupa password admin | Hentikan backend, hapus file `db/scada.db`, jalankan lagi → admin default (`admin` / `Admin123!`) dibuat ulang. (Catatan: ini menghapus seluruh data.) |
| `better-sqlite3` gagal di-install (butuh compiler) | Pastikan Node.js versi LTS terbaru. Di Windows, biasanya tersedia binary siap pakai sehingga tidak perlu compiler. |
| Build frontend gagal kehabisan memori | Jalankan: `set NODE_OPTIONS=--max-old-space-size=2048` lalu `npm run build`. |

---

## 9. Ringkasan Perintah Cepat

```powershell
# --- Sekali saja: install ---
cd "d:\Microplus Product 2026\qscada lite\scada-backend"; npm install
cd "d:\Microplus Product 2026\qscada lite\scada-builder"; npm install

# --- Setiap kali menjalankan (2 terminal) ---
# Terminal 1:
cd "d:\Microplus Product 2026\qscada lite\scada-backend"; npm start
# Terminal 2:
cd "d:\Microplus Product 2026\qscada lite\scada-builder"; npm run dev
# Lalu buka http://localhost:5173  (login: admin / Admin123!)
```

