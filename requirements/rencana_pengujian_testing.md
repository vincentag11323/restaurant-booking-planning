# Rencana Pengujian (Testing)
**Aplikasi Rumah Sakit**

## 1. Fungsional (Functional Testing)
* **Unit Testing:** Menguji fungsi individual di level kode, seperti kalkulasi tagihan, validasi NIK, atau pembuatan kode OTP.
* **Integration Testing:** Memastikan API aplikasi mobile (Pasien) terhubung dengan baik dengan *dashboard* (Admin) dan alur komunikasi *database*.
* **User Acceptance Testing (UAT):** Melibatkan perwakilan pasien dan staf RS untuk mencoba alur *booking* hingga pembayaran di lingkungan *staging* sebelum rilis ke publik.

## 2. Keamanan (Security Testing)
* **Penetration Testing (Pen-Test):** Dilakukan minimal setahun sekali oleh pihak ketiga (*White-box* dan *Black-box* testing) untuk mencari celah keamanan.
* **Vulnerability Scanning:** Pemindaian otomatis secara berkala pada *dependency* atau *library* pihak ketiga untuk mencari *Common Vulnerabilities and Exposures* (CVE).
* **Otorisasi Testing (IDOR):** Memastikan kerentanan *Insecure Direct Object Reference* (IDOR) tidak ada. Contoh pengujian: Memastikan Pasien A tidak bisa mengganti URL parameter (misal: `/record/123` menjadi `/record/124`) untuk melihat rekam medis Pasien B.

## 3. Performa (Performance Testing)
* **Load Testing:** Menguji sistem dengan 1.000+ pengguna secara bersamaan yang melakukan *booking* (mensimulasikan *traffic* tinggi di jam sibuk pendaftaran poli).
* **Stress Testing:** Mengetahui batas maksimal server (CPU/RAM) sebelum sistem mengalami *downtime* atau *timeout*.