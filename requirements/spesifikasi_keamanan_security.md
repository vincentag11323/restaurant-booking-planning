# Spesifikasi Keamanan (Security)
**Aplikasi Rumah Sakit**

Keamanan adalah prioritas utama untuk aplikasi kesehatan (HealthTech) untuk melindungi rekam medis yang bersifat sensitif.

## 1. Autentikasi & Otorisasi
* **Role-Based Access Control (RBAC):** Implementasi otorisasi secara ketat. Admin tidak bisa melihat rekam medis detail tanpa izin medis terkait, dan Pasien hanya bisa melihat datanya sendiri.
* **Token Management:** Penggunaan **JWT (JSON Web Tokens)** dengan waktu kedaluwarsa (expiry time) yang singkat (misal: 15 menit) dan *Refresh Token*.
* **MFA:** Multi-Factor Authentication untuk login Admin dan staf medis.

## 2. Enkripsi Data
* **Data in Transit:** Menggunakan protokol TLS 1.2 atau 1.3 (HTTPS) untuk semua komunikasi API.
* **Data at Rest:** Enkripsi *database* menggunakan standar AES-256, terutama untuk kolom NIK, rekam medis, dan password (di-hash dengan algoritma yang kuat seperti bcrypt atau Argon2).

## 3. Kepatuhan Regulasi
* Memenuhi standar **SatuSehat** (Kemenkes RI).
* Mematuhi regulasi Perlindungan Data Pribadi (UU PDP) terkait pengumpulan dan pemrosesan data pasien.