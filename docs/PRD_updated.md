# Artha Vista – Product Requirements Document (PRD)
Version: 0.1  
Owner: Beni Saprulah  
Last updated: 24 Nov 2025

---

## 1. Ringkasan

Artha Vista adalah **decision support tool** untuk membantu lembaga keuangan (contoh: Amartha) menilai kelayakan kredit UMKM pedesaan dengan lebih cepat, konsisten, dan inklusif.

Produk ini memanfaatkan kombinasi:

- Data profil usaha & peminjam
- Hasil kunjungan lapangan oleh Field Agent (catatan + foto)
- Analisis AI (Gemini + baseline risk model)

Artha Vista **tidak menggantikan core lending system**, tetapi menjadi **lapisan scoring & insight** di atas data yang sudah dimiliki lembaga (borrower, aplikasi, repayment, dll).

---

## 2. Latar Belakang & Masalah

### 2.1 Masalah utama

- Banyak pelaku UMKM pedesaan (mayoritas perempuan) **tidak punya dokumen formal** (slip gaji, rekening koran, agunan) sehingga sulit dinilai dengan metode credit scoring tradisional.  
- Penilaian manual:
  - Mengandalkan intuisi Field Agent & tim kredit.
  - Butuh waktu 3–5 jam per aplikasi.
  - Rentan bias dan tidak konsisten antar wilayah.
- Kapasitas tim kredit dan lapangan terbatas, sementara volume aplikasi tinggi.

Tanpa pendekatan scoring yang lebih data-driven dan inklusif, jutaan pelaku UMKM akan tetap bergantung pada pinjaman informal berbiaya tinggi.

### 2.2 Peluang

- Amartha sudah punya:
  - Aplikasi mobile borrower & proses operasional yang mapan.
  - Jaringan Field Agent yang kuat.
- Yang dibutuhkan: **layer AI** yang:
  - Membaca data, foto, dan catatan kunjungan.
  - Menghasilkan skor risiko, rekomendasi limit, dan ringkasan yang mudah dicerna.

---

## 3. Tujuan Produk & KPI

### 3.1 Tujuan

1. **Mempercepat** proses penilaian kredit tanpa mengorbankan kualitas.
2. **Menstandarkan** penilaian antar Field Agent dan wilayah.
3. **Meningkatkan inklusi** bagi UMKM yang selama ini sulit dinilai secara formal.
4. Menyediakan **penjelasan yang transparan** ke borrower & tim internal.

### 3.2 KPI Sukses (MVP)

- Mengurangi waktu analisis manual per aplikasi (target pilot: dari 3–5 jam → < 1 jam).
- Coverage aplikasi yang melalui Artha Vista (misal: 80% aplikasi di area pilot).
- Akurasi awal:
  - Korelasi skor dengan keputusan final tim kredit.
  - Early indicator NPL (untuk fase lanjut, pakai data repayment).
- Adoption:
  - % Field Agent yang memakai tool saat kunjungan.
  - % aplikasi yang punya skor & ringkasan dari AI.

---

## 4. Ruang Lingkup

### 4.1 In-scope (MVP)

1. **Integrasi alur aplikasi pinjaman**
   - Terima payload aplikasi dari sistem existing (webhook / simulasi).
   - Simpan data ke Firestore.

2. **Dashboard Field Agent**
   - List aplikasi “perlu dikunjungi”.
   - Form input hasil kunjungan (catatan & jawaban survey sederhana).
   - Upload foto usaha/rumah ke Cloud Storage.

3. **Dashboard Agent Officer**
   - List aplikasi yang sudah dikunjungi & sudah/belum di-score.
   - View skor, kategori risiko, summary, rekomendasi limit, dan alasan utama.
   - Tombol keputusan: `Approve` / `Reject` / `Need More Detail` + alasan.

4. **Scoring Engine (AI Layer)**
   - Menggabungkan:
     - Data profil usaha & peminjam.
     - Catatan kunjungan.
     - Foto (URL Cloud Storage).
   - Panggil Gemini API untuk:
     - Ringkasan profil.
     - Kategori risiko (Low/Medium/High).
     - Alasan utama & rekomendasi limit.

5. **Autentikasi & Otorisasi**
   - Firebase Auth untuk Field Agent & Agent Officer.
   - Role-based access di backend (FastAPI).

### 4.2 Out-of-scope (MVP)

- Perubahan core process di aplikasi borrower Amartha (hanya asumsi integrasi).
- Penghitungan interest rate & repayment schedule detail.
- Model risiko berbasis histori repayment “beneran” (di MVP cukup baseline + aturan sederhana).
- Fitur integrasi penuh ke core lending system (hanya demo/pilot level).

---

## 5. Target User & Persona

### 5.1 Field Agent

- Job: survei lapangan, ambil foto, wawancara borrower, input data.
- Need:
  - Tool yang ringan dan mobile-friendly.
  - Guidance saat kunjungan (data apa saja yang wajib diisi).
  - Feedback cepat (skor awal + insight) agar bisa jelaskan ke borrower.

### 5.2 Agent Officer / Credit Team

- Job: review aplikasi & mengambil keputusan akhir.
- Need:
  - Tampilan ringkas: skor, kategori risiko, alasan utama, dan foto kunci.
  - Bisa override AI (approve walau skor medium, reject meski skor high jika ada red flag lainnya).
  - Log keputusan dan alasan (audit trail).

### 5.3 Borrower (indirect user)

- Mendapat manfaat:
  - Proses lebih cepat.
  - Alasan jelas saat approval/rejection.

---

## 6. User Journey (End-to-End)

### 6.1 Borrower

1. Membuat akun & isi profil usaha di app existing.
2. Mengajukan pinjaman (isi jumlah pinjaman, tenor, dll).
3. Menunggu kunjungan Field Agent.
4. Setelah keputusan, menerima notifikasi & limit pinjaman, lalu memilih menerima atau tidak.

### 6.2 Field Agent

1. Login ke Artha Vista dashboard (React + Firebase Auth).
2. Melihat daftar aplikasi **status `new`** di wilayahnya.
3. **Set jadwal kunjungan (schedule visit):**
   - Field Agent memilih salah satu aplikasi dan menentukan tanggal/jam kunjungan.
   - Sistem meng-update status aplikasi menjadi `scheduled` dan mengisi `assigned_field_agent_id` dengan Field Agent tersebut.
   - Aplikasi yang sudah `scheduled` **tidak muncul lagi** di list Field Agent lain (menghindari double visit).
4. Pada hari H, Field Agent melakukan kunjungan ke lokasi borrower:
   - Isi form survey singkat (omzet, lama usaha, dsb).
   - Tulis catatan kunjungan.
   - Upload 3–5 foto (warung, stok, rumah, dll).
5. Submit kunjungan → backend simpan ke Firestore dan kirim permintaan scoring ke AI layer.
6. Setelah skor tersedia, Field Agent bisa melihat highlight utama (opsional di MVP) untuk membantu menjelaskan ke borrower atau ke Officer.
### 6.3 Agent Officer

1. Login ke dashboard officer.
2. Melihat list aplikasi yang sudah punya skor.
3. Membuka detail aplikasi:
   - Data borrower & usaha.
   - Skor & kategori risiko.
   - Ringkasan AI.
   - Foto kunci.
4. Pilih aksi:
   - `Approve` → set limit pinjaman.
   - `Reject` → tulis alasan (wajib).
   - `Need More Detail` → kirim kembali ke Field Agent dengan instruksi lanjutan.

---

## 7. Kebutuhan Fungsional

### 7.1 Integrasi Aplikasi & Webhook

- Endpoint: `POST /api/applications/webhook`
- Input minimal:
  - `borrower_id`
  - `application_id`
  - `loan_amount_requested`
  - Data profil usaha (jenis usaha, lama usaha, omzet, dll)
- Behavior:
  - Validasi signatur / token (untuk Milestone cukup dummy).
  - Simpan ke `applications` collection di Firestore dengan status `new`.

### 7.2 Manajemen Data di Firestore

Collections utama:

- `users` – info user & role.
- `business_profiles` – profil usaha.
- `applications` – aplikasi pinjaman + status.
- `surveys` – hasil kunjungan Field Agent.
- `scores` – hasil scoring AI.

Requirement:

- Setiap `application` harus:
  - Linked ke `borrower_id` dan `business_profile_id`.
  - Punya status lifecycle: `new` → `visited` → `scored` → `approved/rejected/need_more_detail`.

### 7.3 Upload Foto (Cloud Storage)

- Field Agent dapat upload beberapa foto.
- Backend:
  - Terima file atau URL dari frontend.
  - Simpan ke Cloud Storage bucket.
  - Simpan URL publik/bertanda tangan di Firestore pada `surveys`.

### 7.4 Scoring Engine (Gemini + Rule-based)

- Trigger:
  - Saat Field Agent submit survey lengkap.
- Input ke AI:
  - Data profil usaha (numerik & kategori).
  - Catatan kunjungan (teks).
  - URL foto dari Cloud Storage.
- Output (minimal untuk MVP):
  - `risk_score` (0–100) dan `risk_category` (Low / Medium / High).
  - `suggested_loan_limit` + catatan range aman (mis: “disarankan 4.5–5 juta”).
  - `summary` : ringkasan 3–5 kalimat tentang kondisi usaha & peminjam.
  - `top_reasons` : 3–7 alasan utama yang menjelaskan kenapa skor segitu (gabungan faktor positif & negatif).
  - `business_health_summary` : paragraf khusus yang menjelaskan kesehatan usaha (stabilitas omzet, keragaman pelanggan, ketergantungan pada pemasok tunggal, dll).
  - `risk_factors` : list terstruktur berisi kode faktor risiko, label, dan apakah pengaruhnya positif/negatif (misal: `SHORT_BUSINESS_AGE`, `STABLE_REVENUE`, `HIGH_EXISTING_DEBT`).
  - `supporting_evidence` : potongan catatan field agent dan deskripsi foto yang dipakai AI sebagai bukti (misal: “stok di rak hampir habis di semua kategori barang fast moving”).
  - `mitigation_notes` : saran mitigasi yang bisa dipakai Officer (misal: mulai dengan limit lebih kecil, minta dokumen tambahan, atau pantau pembayaran 2 siklus dulu).
  - `visual_observations` : insight yang berasal dari analisis foto (kondisi toko/rumah, kerapihan stok, keberadaan peralatan usaha).
  - `consistency_checks` : hasil cek konsistensi antara data numerik dan observasi lapangan (contoh: omzet vs nilai stok, klaim omzet vs kondisi toko).

Semua output ini disimpan dalam entity/collection `scores` dan ditampilkan sebagai panel **AI Insights** di dashboard Officer, sehingga Officer tidak hanya melihat angka, tapi juga narasi dan bukti pendukung yang mudah dijelaskan ke borrower dan tim audit.
### 7.5 Dashboard Officer

- List view:
  - Filter by status, skor, wilayah.
- Detail view:
  - Data borrower, usaha, skor, summary, foto.
  - Tombol aksi keputusan.
- Post-decision:
  - Simpan keputusan + alasan di aplikasi.
  - Update status di Firestore.

### 7.6 Notifikasi (MVP)

- Minimal:
  - Update status di dashboard Field Agent & Officer.
- Nice-to-have:
  - Web push / email untuk event penting (scoring selesai, aplikasi butuh detail, dll).

---

## 8. Kebutuhan Non-Fungsional

- **Performance**
  - Latensi scoring AI: target < 10 detik per aplikasi (untuk demo).
- **Keamanan**
  - Auth via Firebase.
  - Role based access kontrol di backend.
- **Reliability**
  - Backend di Cloud Run (auto scale).
- **Scalability**
  - Desain schema Firestore untuk jutaan aplikasi jangka panjang.

---

## 9. Teknologi Utama

- Backend: Python, FastAPI di Cloud Run.
- Frontend: React + Firebase Auth, Firebase Hosting.
- Database: Firestore.
- Storage: Cloud Storage.
- AI Stack: Gemini Flash 2.5 untuk multimodal (text + image).

---

## 10. Roadmap Singkat (Hackathon MVP)

- Day 1–2: Finalisasi scope + schema + scaffold API & UI.
- Day 3–4: Implement backend + integrasi Gemini + Firestore.
- Day 5: Build dashboard FA & Officer.
- Day 6–7: Polishing, dummy data, dan demo story.
