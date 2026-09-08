# Modul 02: SFTP Jail, Cron Job, & Dashboard Cockpit

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Mengamankan protokol transfer berkas menggunakan mekanisme _Chroot Jail_ pada SFTP, mengotomatiskan tugas sistem menggunakan _Crontab_, dan memantau kesehatan server secara visual melalui dasbor _Cockpit_.
- **Skenario / Case Study:** Di lingkungan produksi, pihak eksternal (seperti pengembang aplikasi/developer) memerlukan akses untuk mengunggah berkas ke server. Memberikan hak akses SSH penuh (_shell login_) sangat berisiko. Skenario ini diselesaikan dengan membuat sebuah akun transfer berkas terisolasi (Chroot) yang hanya bisa melihat foldernya sendiri tanpa kemampuan eksekusi terminal, sementara aktivitas sistem tetap dipantau secara real-time via dasbor Cockpit.
- **Kebutuhan Perangkat / Prasyarat:**
  - VM Ubuntu Server 24.04 (172.16.2.2)
  - Aplikasi FileZilla Client pada Windows Host
  - Paket Cockpit (`cockpit`)

## 🗺️ Topologi Jaringan & Arsitektur

![Phase1.02 gambar 2](../assets/02-sftp-cron-cockpit-topology.png)

> _[Placeholder Gambar]: Gambarlah diagram yang menunjukkan pengguna FileZilla dari Windows Host mengakses port 2201 (SFTP Jail) dan browser mengakses port 9090 (Cockpit) pada VM Server 1._

### Tabel Pengamatan IP / Interface

| Device Name        | Protokol / Layanan    | Port | Jalur Akses             | Keterangan                                    |
| ------------------ | --------------------- | ---- | ----------------------- | --------------------------------------------- |
| ubuntu-server-1-24 | SFTP (SSH Subsystem)  | 2201 | 192.168.230.143         | Isolasi direktori `/var/sftp/sftpuser/upload` |
| ubuntu-server-1-24 | Cockpit Web Dashboard | 9090 | https://172.16.2.2:9090 | Monitor CPU, RAM, Disk, & systemd services    |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Pembuatan User Terisolasi & Struktur Jail SFTP

- **Penjelasan Singkat:** Membuat user pengunggah berkas dan mematikan akses shell interaktifnya, kemudian membuat folder berstruktur khusus.
- **Perintah CLI / Konfigurasi:**

```bash
# Membuat user sftpuser
ubuntu@ubuntu-server-1-24:~$ sudo adduser sftpuser

# Memblokir shell login sftpuser demi keamanan
ubuntu@ubuntu-server-1-24:~$ sudo usermod -s /usr/sbin/nologin sftpuser

# Membuat struktur folder penampung jail
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /var/sftp/sftpuser/upload
```
