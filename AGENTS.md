# AGENTS.md — Event Management & QR Code Check-in System

## 1. Gambaran Umum Proyek

Aplikasi web **Multi-Event Management** yang dirancang ringan dan cepat untuk mengelola pendaftaran berbagai acara secara terpisah melalui **Direct Link (`/e/{slug}`)**, verifikasi pembayaran manual, hingga validasi kehadiran di hari-H tanpa memerlukan manajemen akun untuk panitia lapangan.

### Aktor & Model Akses

1. **Admin (Single Role — Authenticated):** Satu-satunya pengguna yang memiliki akun login (email & password). Mengelola banyak event, menyalin tautan registrasi per event, mengatur kuota, memverifikasi bukti pembayaran peserta (approve/reject), membuat/menonaktifkan _link scanner_, memantau kehadiran, dan mengekspor laporan.
2. **Peserta (Publik — Tanpa Login):** Mengakses halaman registrasi spesifik melalui tautan langsung (`/e/{slug}`) yang dibagikan penyelenggara, mengisi data diri, mengunggah bukti pembayaran, dan menerima email konfirmasi (berisi QR code jika disetujui, atau email pemberitahuan penolakan jika ditolak).
3. **Pengelola / Petugas Lapangan / Panitia (Tokenized Access — Tanpa Login):** Mengakses halaman pemindai kamera (_scanner_) melalui tautan unik ber-hash (`/scan/{token_hash}`) yang dibagikan oleh Admin.

---

## 2. Tech Stack & Lingkungan Pengembangan

- **Backend:** Laravel 13
- **Frontend:** Blade + Tailwind CSS
- **Database:** MySQL
- **QR Code Library:** `simplesoftwareio/simple-qrcode`
- **Scanner Client-Side:** `html5-qrcode`
- **Mail Service:** Mailpit (SMTP lokal untuk pengujian pengiriman email)

---

## 3. Struktur Database (Core Schema)

Gunakan struktur relasi berikut sebagai acuan migrasi dan model Eloquent:

1. **`admins`**
    - `id` (PK)
    - `name` (string)
    - `email` (string, unique)
    - `password` (string, hashed)
    - `created_at`, `updated_at`

2. **`events`**
    - `id` (PK)
    - `title` (string)
    - `slug` (string, unique — URL-friendly identifier unik untuk tautan registrasi publik `/e/{slug}`)
    - `description` (text, nullable)
    - `location` (string)
    - `event_date` (datetime)
    - `quota` (integer, unsigned)
    - `price` (unsignedBigInteger, default: 0 — nominal biaya pendaftaran)
    - `payment_instructions` (text, nullable — informasi nomor rekening / e-wallet / QRIS tujuan transfer)
    - `is_active` (boolean, default: true)
    - `created_at`, `updated_at`

3. **`registrations`**
    - `id` (PK)
    - `event_id` (FK -> `events.id`, cascade on delete)
    - `name` (string)
    - `email` (string)
    - `phone` (string)
    - `payment_proof` (string — path file bukti pembayaran di storage)
    - `payment_status` (enum/string: `pending`, `approved`, `rejected`; default: `pending`)
    - `rejection_reason` (text, nullable — catatan alasan penolakan dari admin)
    - `verified_by` (FK -> `admins.id`, nullable, null on delete)
    - `verified_at` (timestamp, nullable)
    - `ticket_code` (string, unique, nullable — baru di-generate saat `payment_status` menjadi `approved`)
    - `status` (enum/string: `pending`, `registered`, `checked_in`, `cancelled`; default: `pending`)
    - `email_sent_at` (timestamp, nullable)
    - `created_at`, `updated_at`
    - _Constraint:_ 1 email hanya boleh memiliki 1 pendaftaran aktif per event (`payment_status` `pending` atau `approved`). Izinkan pendaftaran ulang dengan email yang sama pada event tersebut jika pendaftaran sebelumnya berstatus `rejected`.

4. **`scanner_links`**
    - `id` (PK)
    - `event_id` (FK -> `events.id`, cascade on delete)
    - `label` (string — contoh: `"Gate A"`, `"Petugas Pintu Utama"`)
    - `token_hash` (string, unique — string kriptografis acak minimal 32 karakter)
    - `expires_at` (datetime, nullable)
    - `is_active` (boolean, default: true)
    - `created_at`, `updated_at`

5. **`checkin_logs`**
    - `id` (PK)
    - `registration_id` (FK -> `registrations.id`, nullable jika tiket tidak valid)
    - `scanner_link_id` (FK -> `scanner_links.id`)
    - `raw_code` (string — menyimpan kode mentah yang dipindai untuk audit)
    - `scanned_at` (timestamp)
    - `status` (enum/string: `success`, `duplicate`, `invalid`)
    - `created_at`, `updated_at`

---

## 4. Alur Bisnis & Aturan Logika Inti

### A. Manajemen Event & Generate Link Scanner (Admin)

- Admin wajib terautentikasi sebelum mengakses endpoint `/admin/*`.
- Saat membuat event, sistem otomatis men-generate `slug` unik dari `title` (tambahkan akhiran acak singkat jika judul sama) agar URL publik tidak memakai `id` database.
- Admin mengisi informasi acara beserta nominal `price` dan `payment_instructions` yang akan tampil di halaman registrasi peserta.
- Pada daftar event di dashboard Admin, sediakan tombol **"Copy Link Registrasi"** untuk menyalin URL lengkap (`{APP_URL}/e/{slug}`) ke clipboard.
- Saat membuat _link scanner_, sistem men-generate `token_hash` acak yang tidak bisa ditebak (gunakan `Str::random(40)`), bukan ID berurutan.
- Admin dapat menonaktifkan (`is_active = false`) atau mengatur `expires_at` pada tautan scanner kapan saja untuk mencabut akses petugas lapangan.

### B. Routing Publik, Registrasi Peserta & Upload Bukti Pembayaran

- **Pola Direct Link (Tanpa Katalog Publik):**
    - Halaman registrasi setiap event diakses melalui `GET /e/{slug}` dan diproses melalui `POST /e/{slug}`.
    - Halaman utama (`GET /`) menampilkan halaman sambutan sederhana.
- Validasi bahwa `event.is_active == true` dan jumlah pendaftar aktif (`payment_status != 'rejected'` dan `status != 'cancelled'`) masih kurang dari `event.quota`.
- Gunakan **database transaction / locking (`lockForUpdate`)** saat pengecekan kuota untuk mencegah _race condition_ (kelebihan kuota saat traffic tinggi).
- Peserta wajib mengunggah file `payment_proof` (validasi format gambar/dokumen: `jpg`, `jpeg`, `png`, `pdf`, maksimal `2MB`).
- Simpan data ke `registrations` dengan `payment_status = 'pending'`, `status = 'pending'`, dan `ticket_code = null`.
- **Jangan generate atau kirim QR code pada tahap ini.** Tampilkan halaman konfirmasi bahwa pendaftaran berhasil dikirim, sedang menunggu verifikasi pembayaran oleh Admin, dan hasil verifikasi akan dikirim melalui email.

### C. Verifikasi Pembayaran & Pengiriman Email (Admin)

Admin memeriksa daftar pendaftar beserta pratinjau file bukti pembayaran (`payment_proof`), lalu memilih salah satu dari dua tindakan berikut:

1. **Approve Pembayaran (`POST /admin/registrations/{registration}/approve`):**
    - Pastikan `payment_status` belum `approved` (cegah duplikasi proses).
    - Dalam satu transaksi database:
        - Generate `ticket_code` unik yang sulit ditebak (misal: `Str::upper(Str::random(12))` atau UUID).
        - Ubah `payment_status = 'approved'`, `status = 'registered'`, `verified_by = auth()->id()`, dan `verified_at = now()`.
    - Generate gambar QR code dari `ticket_code` menggunakan `simplesoftwareio/simple-qrcode`.
    - Kirim email menggunakan Mailable khusus tiket (misal: `RegistrationApprovedMail`) yang berisi detail acara dan lampiran/gambar QR code, lalu isi `email_sent_at = now()`.

2. **Reject Pembayaran (`POST /admin/registrations/{registration}/reject`):**
    - Ubah `payment_status = 'rejected'`, `status = 'cancelled'`, `rejection_reason` (opsional dari input admin), `verified_by = auth()->id()`, dan `verified_at = now()`.
    - Kuota otomatis kembali tersedia dan email peserta tersebut dapat digunakan untuk mendaftar ulang pada event yang sama.
    - Kirim email menggunakan Mailable berbeda (misal: `RegistrationRejectedMail`) **tanpa QR code**, berisi pesan pemberitahuan bahwa bukti pembayaran tidak dapat diverifikasi (beserta alasan penolakan jika diisi) dan instruksi tindak lanjut, lalu isi `email_sent_at = now()`.

### D. Validasi & Scan QR Code Hari-H (Pengelola)

- **Validasi Akses Halaman Scanner (`/scan/{token_hash}`):**
    - Cek apakah `token_hash` ada di `scanner_links`, `is_active == true`, dan belum melewati `expires_at`.
    - Jika tidak valid/kedaluwarsa, tolak akses (HTTP 403/404) dan tampilkan pesan akses ditolak.
- **Validasi Tiket Saat Pemindaian (`POST /scan/{token_hash}/verify`):**
    1. Validasi ulang keabsahan `token_hash` di setiap request API scan.
    2. Cari `ticket_code` pada tabel `registrations` yang sesuai dengan `event_id` milik `scanner_link` dan memiliki `payment_status == 'approved'`:
        - **Tidak ditemukan / belum di-approve:** Catat ke `checkin_logs` dengan `status = 'invalid'`. Kembalikan respons **Merah** (_Tiket Tidak Terdaftar / Tidak Valid_).
        - **Sudah berstatus `checked_in`:** Catat ke `checkin_logs` dengan `status = 'duplicate'`. Kembalikan respons **Kuning** (_Tiket Sudah Digunakan_) beserta waktu check-in sebelumnya.
        - **Valid (`status == 'registered'`) & belum dipakai:** Dalam satu transaksi atomik, ubah `registrations.status = 'checked_in'` dan simpan log ke `checkin_logs` dengan `status = 'success'`. Kembalikan respons **Hijau** (_Check-in Berhasil_) beserta nama peserta.

### E. Monitoring & Laporan (Admin)

- Tampilkan ringkasan per event: total kuota, pendaftar menunggu verifikasi (`pending`), pembayaran disetujui (`approved`), pembayaran ditolak (`rejected`), total sudah _check-in_, dan sisa belum hadir.
- Sediakan fitur ekspor data pendaftar & kehadiran (nama, email, no HP, `payment_status`, `ticket_code`, status kehadiran, waktu scan, dan label petugas scanner) ke format `.xlsx` atau `.csv`.

---

## 5. Aturan Keamanan & Konvensi Kode

- **Arsitektur Laravel 13:**
    - Gunakan **Form Request** (`app/Http/Requests`) untuk validasi input dan upload file.
    - Pisahkan logika bisnis kompleks (registrasi + kuota locking, approval + QR generation + mail dispatch, verifikasi scan) ke dalam **Action / Service classes** (`app/Actions` atau `app/Services`), jaga agar Controller tetap tipis.
    - Buat dua class Mailable terpisah: `RegistrationApprovedMail` (dengan QR code) dan `RegistrationRejectedMail` (tanpa QR code).
- **Keamanan File & Endpoint:**
    - Simpan file `payment_proof` pada disk private (`storage/app/private/payment_proofs`) dengan nama file ter-hash, dan tampilkan kepada Admin melalui route controller terproteksi (`/admin/registrations/{registration}/proof`) agar tidak bisa diakses publik secara acak.
    - Terapkan _rate limiting_ pada endpoint registrasi publik dan endpoint API pemindaian QR code.
    - Jangan pernah mengekspos `id` internal database pada URL publik (`/e/{slug}`) maupun isi QR code.
- **UI/UX Scanner:** Halaman `/scan/{token_hash}` wajib _mobile-first_, mendukung akses kamera belakang perangkat secara langsung, serta memberikan umpan balik visual yang kontras (Hijau / Kuning / Merah) dan notifikasi suara/getar agar cepat dibaca petugas di lapangan.
