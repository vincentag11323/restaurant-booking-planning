# Alur Kerja (Workflow)
**Aplikasi Rumah Sakit**

## 1. Alur Pendaftaran & Janji Temu (Pasien)
1. Pasien mengunduh aplikasi dan melakukan **Registrasi** (verifikasi via Email/WhatsApp).
2. Pasien melengkapi **Profil Data Diri** (NIK, Nama, Tanggal Lahir).
3. Pasien masuk ke menu **Buat Janji Temu**, memilih Poliklinik & Dokter.
4. Pasien memilih tanggal & slot waktu yang tersedia.
5. Sistem mengonfirmasi *booking* dan menerbitkan **Nomor Antrean / Kode Booking**.
6. H-1, sistem mengirimkan **Notifikasi Pengingat** kepada pasien.

## 2. Alur Pelayanan di Rumah Sakit (Interaksi Pasien & Admin)
1. Pasien tiba di RS dan menunjukkan Kode Booking ke Admin.
2. Admin memasukkan Kode Booking ke sistem untuk **Verifikasi Kehadiran** (Check-in).
3. Status pasien berubah menjadi "Menunggu Konsultasi".
4. Setelah konsultasi selesai, tagihan otomatis masuk ke sistem (dari dokter/farmasi).
5. Admin melakukan konfirmasi **Pembayaran** (Tunai/Non-Tunai/Asuransi).
6. Pasien menerima *digital receipt* di aplikasi.