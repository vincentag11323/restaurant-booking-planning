# Sistem Audit (Audit Trail)
**Aplikasi Rumah Sakit**

Setiap tindakan kritis di dalam aplikasi harus dicatat untuk kebutuhan investigasi dan kepatuhan medis.

## 1. Format Log yang Disimpan
Sistem akan menyimpan log aktivitas dengan detail berikut:
* `Timestamp`: Kapan kejadian berlangsung (waktu server).
* `Actor ID`: ID User atau Admin yang melakukan tindakan.
* `Action`: Jenis tindakan (Contoh: `VIEW_MEDICAL_RECORD`, `UPDATE_PATIENT_DATA`, `LOGIN_FAILED`, `CREATE_INVOICE`).
* `Resource ID`: ID data atau entitas yang diubah/diakses.
* `IP Address` & `User Agent`: Informasi perangkat dan jaringan yang digunakan.

## 2. Kebijakan Retensi
* Log audit bersifat *Immutable* (tidak boleh dapat diubah atau dihapus oleh siapapun, termasuk Super Admin).
* Log harus disimpan dengan aman minimal selama **5 tahun** sesuai standar operasional rumah sakit.

## 3. Peringatan Keamanan (Alerting)
* Sistem harus mendeteksi dan mengirim notifikasi otomatis ke *Super Admin* atau tim *Security* jika ada anomali. 
* Contoh anomali: 1 Admin mencoba mengunduh 100 rekam medis dalam waktu 5 menit, atau terjadi 10 kali percobaan login gagal dari IP yang sama.