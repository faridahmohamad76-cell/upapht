# Plan: Sistem Pengurusan Aset Alih — Politeknik Hulu Terengganu

## Top-Level Overview

Membina sebuah aplikasi web full-stack untuk pengurusan aset alih Politeknik Hulu Terengganu dengan dua halaman utama:

1. **Halaman Login (Page 1)** — Single Sign-On (SSO) melalui Microsoft 365 / Azure Active Directory
2. **Halaman Dashboard (Page 2)** — Dashboard aset dengan sidebar navigasi, paparan data, pengesahan aset, muat turun dan muat naik Excel

**Stack teknologi:**
- Frontend: HTML5, CSS3, Vanilla JavaScript (SPA sederhana)
- Backend: Node.js (Express.js)
- Auth: Microsoft Azure AD (MSAL.js untuk frontend, msal-node untuk backend)
- Database: SQLite (ringan, sesuai deployment dalaman)
- Excel: SheetJS (xlsx) untuk baca/tulis fail Excel
- Deployment: Standalone server dalaman (intranet)

**Sumber data asal:**
- `harta_modal (3).xlsx` — 1,548 rekod aset harta modal (12 lajur)
- `Inventori_Gabungan_9_Fail.xlsx` — 8,665 rekod inventori gabungan
- `LAMPIRAN C-UPDATED 7.6.2026.xlsx` — 2,745 rekod kemaskini terkini (25 lajur)

**Dua peranan pengguna:**
- `admin` — Akses penuh: tambah, edit, padam, muat naik, muat turun, lihat semua aset dan audit log
- `pegawai` — Lihat aset sendiri, sahkan aset, mohon perubahan, muat turun senarai aset sendiri

**Tiga soalan utama yang dijawab oleh sistem:**
1. Di manakah aset berada? (Bangunan / Aras / Bilik / Kod Lokasi)
2. Siapakah bertanggungjawab? (Pegawai Ruang vs Pengguna/Penjaga Aset)
3. Bagaimana pengesahan dibuat? (Login individu → Senarai → Sahkan → Audit)

---

## Seni Bina Sistem

```
Browser (Frontend)
    │
    ├── Page 1: login.html        ← Microsoft SSO via MSAL.js
    └── Page 2: dashboard.html    ← Sidebar + Panel konten dinamik
         ├── Panel: Ringkasan KPI
         ├── Panel: Senarai Aset (Carian + Tapis)
         ├── Panel: Aset Saya (Pengesahan)
         ├── Panel: Lokasi
         ├── Panel: Laporan (Muat Turun Excel/PDF)
         └── Panel: Pengurusan (Admin sahaja)
    │
    ▼
Backend (Node.js / Express)
    ├── /api/auth        ← Azure AD token validation
    ├── /api/assets      ← CRUD aset
    ├── /api/verify      ← Pengesahan aset oleh pegawai
    ├── /api/upload      ← Muat naik Excel
    ├── /api/download    ← Muat turun Excel
    └── /api/audit       ← Log audit
    │
    ▼
SQLite Database
    ├── users            ← Profil pengguna dari Azure AD
    ├── assets           ← Data aset (12 lajur dari harta_modal)
    ├── locations        ← Jadual lokasi (kod BGN)
    ├── verifications    ← Rekod pengesahan aset
    └── audit_log        ← Semua perubahan dilog
```

---

## Sub-Tugas

---

### Sub-Tugas 1: Persediaan Projek & Struktur Folder

**Intent:** Sediakan kerangka projek Node.js, pasang kebergantungan, dan cipta struktur folder yang konsisten untuk semua sub-tugas seterusnya.

**Expected Outcomes:**
- Folder projek `asset-system/` dengan struktur lengkap
- `package.json` dengan semua kebergantungan dipasang
- Fail `.env.example` untuk konfigurasi Azure AD
- Fail `README.md` dengan arahan permulaan

**Todo List:**
1. Cipta struktur folder projek:
   ```
   asset-system/
   ├── backend/
   │   ├── server.js
   │   ├── routes/
   │   ├── middleware/
   │   ├── db/
   │   └── utils/
   ├── frontend/
   │   ├── login.html
   │   ├── dashboard.html
   │   ├── css/
   │   └── js/
   ├── data/              ← fail Excel sumber
   ├── .env.example
   ├── package.json
   └── README.md
   ```
2. Inisialisasi `package.json` dan pasang: `express`, `better-sqlite3`, `xlsx`, `@azure/msal-node`, `dotenv`, `multer`, `cors`
3. Cipta `.env.example` dengan pemboleh ubah: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`, `PORT`, `DB_PATH`
4. Cipta `README.md` dengan arahan setup dan cara jalankan sistem

**Relevant Context:**
- Tiada kod sedia ada — projek baru dari kosong
- Fail Excel sumber ada di `c:\Users\HP\.bob\playground\fileaset\`

**Status:** [ ] pending

---

### Sub-Tugas 2: Database Schema & Import Data Awal

**Intent:** Cipta skema SQLite dan import data dari fail `harta_modal (3).xlsx` sebagai data permulaan sistem.

**Expected Outcomes:**
- Database SQLite dengan 5 jadual dibuat
- Skrip import yang baca `harta_modal (3).xlsx` dan isi jadual `assets`
- Data 1,548 rekod aset berjaya diimport
- Jadual `locations` diisi dengan kod lokasi unik yang diekstrak dari data

**Todo List:**
1. Cipta `backend/db/schema.sql` dengan jadual:
   - `assets` (id, no_siri, jenis, jenama, no_casis, no_vot, no_psn, kos, kod_lokasi, nama_lokasi, pengguna, status, tarikh_diterima, dikemaskini_pada, dikemaskini_oleh)
   - `locations` (kod_lokasi, bangunan, aras, bilik, nama_bilik)
   - `users` (azure_oid, nama, emel, jabatan, peranan)
   - `verifications` (id, asset_id, user_oid, tarikh, status_pengesahan, catatan)
   - `audit_log` (id, asset_id, user_oid, tindakan, nilai_lama, nilai_baru, masa)
2. Cipta `backend/db/init.js` untuk cipta dan migrasi database
3. Cipta `backend/utils/import-excel.js` yang baca `harta_modal (3).xlsx` menggunakan SheetJS
4. Urus baris berselang (baris ganjil = data aset, baris genap = nama bilik) dalam fail Excel sumber
5. Jalankan import dan sahkan 1,548 rekod berjaya masuk

**Relevant Context:**
- Struktur data dalam `harta_modal (3).xlsx`: Lajur A=No, B=No. Siri Pendaftaran, C=Jenis, D=Jenama, E=No. Casis, F=No. VOT, G=No. Psn.Krjn., H=Kos (RM), I=Lokasi (Kod), J=Pengguna, K=Status, L=Tarikh Diterima
- **Perhatian penting:** Baris berselang dalam fail Excel — baris ganjil ada data aset, baris genap (baris berikutnya) ada nama bilik di lajur I/J. Skrip import MESTI tangani corak ini.
- Contoh kod lokasi: `110601/BGN/A/02/064`, `110601/KAW/BE`, `110601/BGN/S/01/002`
- Status yang ada: `Sedang Digunakan` (majoriti)

**Status:** [ ] pending

---

### Sub-Tugas 3: Backend API (Express + Azure AD Auth)

**Intent:** Bina semua endpoint API REST yang diperlukan oleh frontend, dengan pengesahan token Azure AD pada setiap permintaan.

**Expected Outcomes:**
- Server Express berjalan pada port yang dikonfigurasi
- Middleware auth yang sahkan JWT token dari Azure AD
- Endpoint CRUD untuk aset berfungsi
- Endpoint pengesahan aset berfungsi
- Endpoint muat naik dan muat turun Excel berfungsi
- Semua tindakan tulis dilog ke `audit_log`

**Todo List:**
1. Cipta `backend/middleware/auth.js` — sahkan Bearer token Azure AD menggunakan `@azure/msal-node` atau verifikasi JWT manual dengan kunci awam Microsoft
2. Cipta `backend/middleware/roles.js` — middleware semak peranan (`admin` atau `pegawai`)
3. Cipta `backend/routes/assets.js`:
   - `GET /api/assets` — senarai semua aset (admin) atau aset pegawai sendiri (pegawai)
   - `GET /api/assets/:id` — butiran satu aset
   - `POST /api/assets` — tambah aset baru (admin sahaja)
   - `PUT /api/assets/:id` — kemaskini aset (admin sahaja)
   - `DELETE /api/assets/:id` — padam aset (admin sahaja)
4. Cipta `backend/routes/verify.js`:
   - `POST /api/verify/:asset_id` — pegawai sahkan aset masih dalam jagaan
   - `GET /api/verify/pending` — senarai aset belum disahkan oleh pegawai semasa
5. Cipta `backend/routes/excel.js`:
   - `GET /api/download/assets` — jana dan muat turun Excel (sama format harta_modal)
   - `GET /api/download/my-assets` — muat turun aset pegawai sendiri sahaja
   - `GET /api/download/template` — muat turun templat kosong untuk import
   - `POST /api/upload` — terima fail Excel, parsing, dan kemaskini/tambah rekod
6. Cipta `backend/routes/audit.js`:
   - `GET /api/audit` — log audit (admin sahaja), boleh tapis mengikut aset atau pengguna
7. Cipta `backend/server.js` — gabungkan semua route, middleware CORS, dan mulai server

**Relevant Context:**
- Azure AD token mengandungi: `oid` (object ID unik), `name`, `email`, `roles` (dari App Registration)
- Format muat turun Excel mesti sama seperti `harta_modal (3).xlsx` — 12 lajur, termasuk nama bilik di baris genap
- Multer digunakan untuk terima fail Excel yang dimuat naik

**Status:** [ ] pending

---

### Sub-Tugas 4: Halaman Login (Page 1)

**Intent:** Bina halaman login yang menggunakan Microsoft SSO (MSAL.js) dengan reka bentuk rasmi dan mesra pengguna.

**Expected Outcomes:**
- `frontend/login.html` dipaparkan dengan logo dan nama sistem
- Butang "Log Masuk dengan Microsoft 365" berfungsi, buka popup SSO Microsoft
- Selepas berjaya log masuk, token disimpan dan pengguna diubah hala ke `dashboard.html`
- Jika token sudah ada (sesi aktif), auto-redirect ke dashboard
- Mesej ralat dipaparkan jika login gagal

**Todo List:**
1. Cipta `frontend/css/login.css` — gaya halaman login: latar biru gelap, kad putih tengah, logo politeknik
2. Cipta `frontend/login.html` dengan:
   - Logo Politeknik Hulu Terengganu
   - Tajuk sistem: "Sistem Pengurusan Aset Alih"
   - Sub-tajuk: "Politeknik Hulu Terengganu"
   - Butang besar "🔐 Log Masuk dengan Microsoft 365"
   - Footer dengan versi dan nama jabatan
3. Cipta `frontend/js/auth.js` menggunakan MSAL.js (CDN):
   - Konfigurasi dengan `clientId` dan `tenantId` dari pemboleh ubah konfigurasi
   - Fungsi `login()` — panggil `loginPopup()` MSAL
   - Fungsi `getToken()` — dapatkan access token untuk API call
   - Fungsi `logout()` — clear sesi dan redirect ke login
   - Fungsi `checkSession()` — semak token sedia ada semasa muat halaman
4. Pastikan redirect berlaku dengan betul antara `login.html` dan `dashboard.html`

**Relevant Context:**
- MSAL.js v3 boleh dimuatkan dari CDN: `https://alcdn.msauth.net/browser/3.x.x/js/msal-browser.min.js`
- `clientId` dan `tenantId` akan dikonfigurasi dalam fail `frontend/js/config.js` yang berasingan

**Status:** [ ] pending

---

### Sub-Tugas 5: Halaman Dashboard — Struktur & Sidebar (Page 2)

**Intent:** Bina kerangka utama `dashboard.html` dengan sidebar navigasi sebelah kiri dan kawasan kandungan utama yang boleh tukar panel secara dinamik.

**Expected Outcomes:**
- `dashboard.html` dengan susun atur dua lajur: sidebar kiri + kandungan kanan
- Sidebar mengandungi semua menu navigasi dengan ikon
- Bahagian atas (topbar) menunjukkan nama pengguna yang log masuk dan butang log keluar
- Panel kandungan bertukar apabila menu diklik (tanpa reload halaman)
- Reka bentuk responsif dan profesional (warna tema: biru kerajaan `#103B75`)

**Todo List:**
1. Cipta `frontend/css/dashboard.css` — susun atur grid, warna tema, gaya sidebar, kad, jadual
2. Cipta `frontend/dashboard.html` dengan struktur:
   ```
   <div class="layout">
     <aside class="sidebar">         ← Sidebar kiri
     <main class="main-content">    ← Kawasan kandungan
       <header class="topbar">      ← Bar atas
       <div id="panel-container">   ← Panel bertukar di sini
   ```
3. Isi sidebar dengan menu item:
   - 🏠 Ringkasan (Dashboard Utama)
   - 📦 Semua Aset
   - ✅ Aset Saya & Pengesahan
   - 📍 Carian Mengikut Lokasi
   - 📊 Laporan & Muat Turun
   - 📋 Log Audit *(admin sahaja)*
   - ⚙️ Pengurusan Aset *(admin sahaja)*
4. Cipta `frontend/js/dashboard.js` — logik tukar panel, sembunyikan menu admin dari pegawai biasa
5. Topbar: tunjukkan nama pengguna, peranan badge, dan butang "Log Keluar"

**Relevant Context:**
- Menu admin disembunyikan berdasarkan `role` dalam token Azure AD
- Warna tema: `#103B75` (biru gelap), `#1175C5` (biru cerah), `#DCEEFE` (biru muda)

**Status:** [ ] pending

---

### Sub-Tugas 6: Panel Ringkasan KPI & Carta

**Intent:** Bina panel dashboard utama yang memaparkan statistik ringkasan dan carta visual untuk gambaran pantas keseluruhan aset.

**Expected Outcomes:**
- 4 kad KPI: Jumlah Aset, Nilai Keseluruhan (RM), Aset Aktif, Bilangan Lokasi
- Carta pai/donat: Aset mengikut Kategori/Jenis
- Carta bar: 10 lokasi teratas mengikut bilangan aset
- Carta status: Sedang Digunakan / Rosak / Simpanan / Pelupusan
- Semua nilai diambil secara langsung dari API backend

**Todo List:**
1. Cipta `frontend/js/panels/panel-ringkasan.js` — fetch data dari `/api/assets/summary`
2. Tambah endpoint `GET /api/assets/summary` di backend — kira KPI agregat dari database
3. Render 4 kad KPI dengan nombor besar dan ikon
4. Integrasikan Chart.js (CDN) untuk carta pai dan carta bar
5. Carta pai: kumpulkan jenis aset (contoh: ICT, Perabot, Kenderaan, A/V, Peralatan)
6. Carta bar: Top 10 lokasi, kira dari nama bilik

**Relevant Context:**
- Jenis aset yang ada dalam data: SET SOFA, KOMPUTER RIBA, LCD PROJECTOR, VAN, BAS, MPV, MOTOSIKAL, KAMERA VIDEO, KHEMAH, dll
- Status majoriti: `Sedang Digunakan`
- Nilai keseluruhan inventori gabungan: RM 3,496,287.60

**Status:** [ ] pending

---

### Sub-Tugas 7: Panel Senarai Aset (Carian, Tapis & Butiran)

**Intent:** Bina panel senarai aset yang boleh dicari, ditapis dan dipaparkan secara terperinci dalam jadual dengan fungsi admin untuk edit aset.

**Expected Outcomes:**
- Jadual aset dengan kolum: No. Siri, Jenis, Jenama, Lokasi, Pengguna, Status, Kos, Tindakan
- Carian teks bebas (No. Siri, Nama Pengguna, Jenis)
- Penapis: Jenis, Status, Tahun Perolehan, Lokasi (Bangunan)
- Klik baris → modal popup butiran penuh aset
- Admin nampak butang Edit dan Padam; Pegawai tidak nampak
- Pagination untuk rekod banyak

**Todo List:**
1. Cipta `frontend/js/panels/panel-aset.js` — fetch dan render jadual dari `/api/assets`
2. Tambah query parameter ke `GET /api/assets`: `?search=`, `?jenis=`, `?status=`, `?lokasi=`, `?page=`, `?limit=`
3. Bina elemen carian dan penapis (dropdown dinamik isi dari data unik dalam DB)
4. Render jadual dengan pagination bawah
5. Modal butiran: tunjukkan semua 12 medan + sejarah pengesahan terakhir
6. Butang Edit (admin) buka modal form edit → hantar `PUT /api/assets/:id`
7. Butang Padam (admin) → konfirmasi dialog → hantar `DELETE /api/assets/:id`

**Relevant Context:**
- Kod lokasi format: `110601/BGN/A/02/064` — bahagian ke-3 ialah bangunan (A, E, G, J, L, S), ke-4 ialah aras, ke-5 ialah bilik
- Nama bilik disimpan di baris genap dalam Excel asal, mesti disimpan dalam DB semasa import

**Status:** [ ] pending

---

### Sub-Tugas 8: Panel Aset Saya & Pengesahan

**Intent:** Bina panel khusus untuk pegawai melihat senarai aset di bawah jagaan mereka dan melakukan pengesahan (acknowledgment) bahawa aset masih dalam jagaan.

**Expected Outcomes:**
- Pegawai login → nampak senarai aset atas nama mereka sahaja
- Setiap aset ada badge status pengesahan: ✅ Disahkan / ⚠️ Belum Disahkan / 🔴 Tertunggak
- Butang "Sahkan Aset" pada setiap baris — klik → modal konfirmasi → hantar POST ke API
- Butang "Sahkan Semua" untuk sahkan keseluruhan senarai sekali gus
- Selepas pengesahan, timestamp dan nama pegawai direkod dalam `verifications`
- Ruangan notifikasi: "X aset belum disahkan tahun ini"

**Todo List:**
1. Cipta `frontend/js/panels/panel-aset-saya.js` — fetch dari `/api/assets?pengguna=self`
2. Render jadual dengan kolum tambahan: Tarikh Pengesahan Terakhir, Status Pengesahan
3. Logik warna badge: hijau (disahkan dalam tahun semasa), kuning (>6 bulan), merah (>12 bulan atau belum pernah)
4. Butang "Sahkan" → modal: "Saya mengesahkan aset [No. Siri] masih dalam jagaan saya pada [tarikh hari ini]" + butang Sahkan
5. Butang "Mohon Perubahan" → modal borang: pilih jenis perubahan (Tukar Lokasi / Tukar Pengguna / Laporkan Rosak), isi catatan → hantar notifikasi ke admin
6. Endpoint `POST /api/verify/:asset_id` simpan rekod ke jadual `verifications`

**Relevant Context:**
- Padanan pegawai berdasarkan nama dalam lajur `pengguna` di database vs nama dalam token Azure AD
- Admin perlu boleh set padanan manual jika nama tidak tepat sama

**Status:** [ ] pending

---

### Sub-Tugas 9: Panel Muat Turun & Muat Naik Excel

**Intent:** Bina panel laporan dengan butang muat turun pelbagai format dan fungsi muat naik fail Excel untuk kemaskini data.

**Expected Outcomes:**
- Butang muat turun menghasilkan fail Excel dengan format sama seperti `harta_modal (3).xlsx`
- Muat naik Excel memproses fail dan kemaskini rekod dalam database
- Pratonton data sebelum import dikonfirmasi
- Ralat dalam fail Excel (kolum salah, data tidak lengkap) dipaparkan dengan jelas

**Todo List:**
1. Cipta `frontend/js/panels/panel-laporan.js` — render butang-butang muat turun
2. **Butang Muat Turun:**
   - "⬇️ Muat Turun Semua Aset" → `GET /api/download/assets` (admin)
   - "⬇️ Muat Turun Aset Saya" → `GET /api/download/my-assets` (semua pengguna)
   - "⬇️ Muat Turun Mengikut Lokasi" → dropdown pilih lokasi → `GET /api/download/assets?lokasi=X`
   - "⬇️ Muat Turun Template" → `GET /api/download/template`
3. **Butang Muat Naik (admin sahaja):**
   - Input file pilih → preview nama fail
   - Butang "📤 Muat Naik & Kemaskini" → POST ke `/api/upload`
4. Backend `POST /api/upload`:
   - Parse Excel menggunakan SheetJS
   - Sahkan lajur: mesti ada B (No. Siri), C (Jenis), H (Kos), I (Lokasi), J (Pengguna), K (Status)
   - Untuk setiap baris: jika No. Siri wujud → UPDATE; jika tidak → INSERT
   - Log semua perubahan ke `audit_log`
   - Kembalikan ringkasan: X rekod dikemaskini, Y rekod baru ditambah, Z ralat
5. Frontend paparkan hasil import dalam modal (jadual ringkasan)
6. Backend `GET /api/download/assets`:
   - Jana Excel dengan format 12 lajur + baris genap nama bilik
   - Set header `Content-Disposition: attachment; filename=Senarai_Aset_[tarikh].xlsx`

**Relevant Context:**
- Format Excel muat turun: MESTI ikut format `harta_modal (3).xlsx` — baris ganjil data, baris genap nama bilik
- SheetJS (`xlsx` npm package) digunakan di backend untuk jana dan baca Excel

**Status:** [ ] pending

---

### Sub-Tugas 10: Panel Log Audit (Admin)

**Intent:** Bina panel log audit untuk admin melihat semua perubahan yang dibuat pada data aset — siapa, bila, apa yang diubah.

**Expected Outcomes:**
- Jadual log audit dengan kolum: Masa, Pengguna, Aset (No. Siri), Tindakan, Nilai Lama, Nilai Baru
- Boleh tapis mengikut: julat tarikh, nama pengguna, jenis tindakan (TAMBAH / KEMASKINI / PADAM / SAHKAN / IMPORT)
- Panel ini hanya kelihatan kepada admin

**Todo List:**
1. Cipta `frontend/js/panels/panel-audit.js` — fetch dari `/api/audit`
2. Tambah tapis tarikh (dari-hingga), tapis pengguna, tapis jenis tindakan
3. Render jadual dengan warna kod: hijau=TAMBAH, biru=KEMASKINI, merah=PADAM, kuning=IMPORT
4. Pastikan setiap tindakan tulis di backend (CRUD, upload, verify) menulis rekod ke `audit_log`

**Relevant Context:**
- Jadual `audit_log`: `(id, asset_id, user_oid, user_name, tindakan, medan, nilai_lama, nilai_baru, masa)`
- Ini panel paparan sahaja — tiada operasi tulis dari panel ini

**Status:** [ ] pending

---

### Sub-Tugas 11: Panel Pengurusan Aset — Tambah Aset Baru (Admin)

**Intent:** Bina borang terstruktur untuk admin menambah rekod aset baru secara manual, satu demi satu.

**Expected Outcomes:**
- Borang dengan semua 12 medan aset dengan validasi
- Dropdown dinamik untuk Jenis (dari nilai unik dalam DB) dan Lokasi
- Selepas submit berjaya, rekod baru muncul dalam senarai aset
- Log audit direkod secara automatik

**Todo List:**
1. Cipta `frontend/js/panels/panel-pengurusan.js` — render borang tambah aset
2. Medan borang: No. Siri (required, unik), Jenis (dropdown+input), Jenama, No. Casis, No. VOT, No. Psn.Krjn., Kos (nombor), Lokasi (dropdown kod + input nama bilik), Pengguna (carian nama), Status (dropdown), Tarikh Diterima (date picker)
3. Validasi: No. Siri tidak boleh duplikasi, Kos mesti nombor positif, Tarikh format betul
4. Submit → `POST /api/assets` → papar mesej berjaya / ralat

**Status:** [ ] pending

---

## Susunan Pembangunan yang Disyorkan

```
Sub-Tugas 1 → Sub-Tugas 2 → Sub-Tugas 3 → Sub-Tugas 4 → Sub-Tugas 5
     ↓               ↓               ↓               ↓               ↓
  [Projek]      [Database]       [API]           [Login]       [Dashboard]
                                                                     ↓
                           Sub-Tugas 6 → Sub-Tugas 7 → Sub-Tugas 8
                                ↓               ↓               ↓
                           [KPI Panel]    [Senarai Aset] [Aset Saya]
                                                                     ↓
                           Sub-Tugas 9 → Sub-Tugas 10 → Sub-Tugas 11
                                ↓               ↓               ↓
                           [Excel I/O]    [Audit Log]   [Tambah Aset]
```

---

## Konfigurasi Azure AD yang Diperlukan (Perlu Dibuat Dahulu)

Sebelum memulakan pembangunan, admin IT perlu:
1. Daftar aplikasi dalam Azure AD Portal (`portal.azure.com`)
2. Dapatkan `Client ID`, `Tenant ID`, dan cipta `Client Secret`
3. Set Redirect URI: `http://localhost:3000/dashboard.html` (development) dan URL sebenar (production)
4. Cipta dua App Role dalam manifest: `Admin` dan `Pegawai`
5. Assign pengguna ke roles yang sesuai

Nilai-nilai ini diisi dalam fail `.env` (backend) dan `frontend/js/config.js` (frontend).
