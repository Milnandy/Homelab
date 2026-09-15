# Modul 02: SFTP Jail, Cron Job, & Dashboard Cockpit

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Mengamankan protokol transfer berkas menggunakan mekanisme _Chroot Jail_ pada SFTP, mengotomatiskan tugas sistem menggunakan _Crontab_, dan memantau kesehatan server secara visual melalui dasbor _Cockpit_.
- **Skenario / Case Study:** Di lingkungan produksi, pihak eksternal (seperti pengembang aplikasi/developer) memerlukan akses untuk mengunggah berkas ke server. Memberikan hak akses SSH penuh (_shell login_) sangat berisiko. Skenario ini diselesaikan dengan membuat sebuah akun transfer berkas terisolasi (Chroot) yang hanya bisa melihat foldernya sendiri tanpa kemampuan eksekusi terminal, sementara aktivitas sistem tetap dipantau secara real-time via dasbor Cockpit.
- **Kebutuhan Perangkat / Prasyarat:**
  - VM Ubuntu Server 24.04 (172.16.2.2)
  - Aplikasi FileZilla Client pada Windows Host
  - Paket Cockpit (`cockpit`)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 02 Topology](/assets/02-sftp-cron-cockpit-topology.png)

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

### Langkah 2: Mengatur Hak Kepemilikan Jail (Syarat Ketat Chroot)

- **Penjelasan Singkat:** Chroot directory harus sepenuhnya dimiliki oleh `root:root` dan tidak boleh dapat ditulis oleh user biasa, sedangkan subfolder `upload` diserahkan kepemilikannya ke sftpuser [7].
- **Perintah CLI / Konfigurasi:**

```bash
# Mengatur hak milik root jail ke root (Mutlak untuk SSH Chroot)
ubuntu@ubuntu-server-1-24:~$ sudo chown root:root /var/sftp
ubuntu@ubuntu-server-1-24:~$ sudo chown root:root /var/sftp/sftpuser
ubuntu@ubuntu-server-1-24:~$ sudo chmod 755 /var/sftp/sftpuser

# Mengatur hak milik folder upload ke sftpuser agar bisa menulis berkas
ubuntu@ubuntu-server-1-24:~$ sudo chown sftpuser:sftpuser /var/sftp/sftpuser/upload
ubuntu@ubuntu-server-1-24:~$ sudo chmod 755 /var/sftp/sftpuser/upload
```

### Langkah 3: Konfigurasi Aturan Chroot pada SSH Daemon

- **Penjelasan Singkat:** Menyisipkan konfigurasi subsistem internal SFTP pada berkas konfigurasi daemon SSH.
- **Perintah CLI / Konfigurasi:**

```bash
# Membuka konfigurasi SSH daemon
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/ssh/sshd_config
```

Tambahkan konfigurasi _Match User_ di bagian paling bawah berkas:

```ini
Match User sftpuser
    ForceCommand internal-sftp
    ChrootDirectory /var/sftp/%u
    PermitTTY no
    X11Forwarding no
    AllowTcpForwarding no
```

Terapkan perubahan dengan merestart socket dan layanan SSH:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart ssh.socket
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart ssh
```

### Langkah 4: Otomasi Penjadwalan Tugas (Cron Job)

- **Penjelasan Singkat:** Membuat perintah berkala yang berjalan otomatis setiap hari untuk mencatat aktivitas sistem ke file log.
- **Perintah CLI / Konfigurasi:**

```bash
# Membuka editor crontab untuk user root
ubuntu@ubuntu-server-1-24:~$ sudo crontab -e
```

Tambahkan baris berikut untuk dieksekusi setiap pergantian hari (pukul 00:00):

```text
0 0 * * * echo "Testing Crontab $(date)" >> /root/cron.log
```

### Langkah 5: Instalasi Dashboard Cockpit

- **Penjelasan Singkat:** Menginstal antarmuka grafis berbasis web untuk mempermudah administrator memantau kinerja server.
- **Perintah CLI / Konfigurasi:**

```bash
# Menginstal paket Cockpit
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install cockpit -y

# Memulai layanan Cockpit
ubuntu@ubuntu-server-1-24:~$ sudo systemctl start cockpit
```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Akses Layanan

```bash
# Verifikasi Cockpit telah mendengarkan port 9090
ubuntu@ubuntu-server-1-24:~$ ss -tulpu | grep cockpit
```

![SFTP Filezilla Verification](/assets/02-sftp-cron-cockpit-verification.png)

> _[Tugas Dokumentasi]: Masukkan tangkapan layar koneksi sukses aplikasi FileZilla ke port 2201 menggunakan user sftpuser, menampilkan direktori yang terisolasi hanya pada folder /upload._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **SFTP Gagal Terhubung (Fatal: Connection reset by peer):**
    - _Gejala:_ FileZilla menolak koneksi secara instan sesaat setelah memasukkan kredensial sftpuser.
    - _Penyebab:_ Hak akses direktori yang didefinisikan sebagai `ChrootDirectory` tidak bersih dimiliki oleh `root:root`, atau folder tersebut memiliki izin menulis (_writeable_) bagi pengguna/grup lain [7].
    - _Solusi:_ Periksa kembali dengan `ls -ld /var/sftp/sftpuser`. Pastikan mutlak mengembalikan kepemilikan ke root: `sudo chown root:root /var/sftp/sftpuser && sudo chmod 755 /var/sftp/sftpuser`.
