# Modul 02: SFTP Jail, Cron Job, & Dashboard Cockpit

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Mengamankan protokol transfer berkas menggunakan mekanisme _Chroot Jail_ berbasis grup (_Group-based_), mengotomatiskan tugas sistem menggunakan _Crontab_, serta memantau kesehatan klaster server secara terpusat melalui dasbor web _Cockpit_.
- **Skenario / Case Study:** Di lingkungan produksi, pihak eksternal atau tim developer membutuhkan akses unggah berkas ke server tanpa memberikan hak akses terminal (_shell login_) yang berisiko. Skenario ini diselesaikan dengan membuat sebuah grup penampung (_sftpgroup_) dengan beberapa akun pengguna di dalamnya. Seluruh akun tersebut diisolasi (_chroot_) ke dalam satu folder direktori bersama (_shared folder_). Selain itu, administrator memantau metrik performa seluruh server secara terpusat melalui satu panel dasbor web Cockpit.
- **Kebutuhan Perangkat / Prasyarat:**
  - VM Ubuntu Server 24.04 Master (172.16.2.2) dan Slave (172.16.2.3)
  - Aplikasi Client SFTP (FileZilla / WinSCP) pada Windows Host
  - Paket Cockpit (`cockpit`)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 02 Topology](/assets/phase-1-sandbox/image/diagram-akses-monitoring-editable.drawio.png)

> _Diagram pengguna FileZilla/WinSCP dari Windows Host mengakses port 2201 (SFTP Jail) dan browser mengakses port 9090 (Cockpit) pada VM Server 1 yang memantau Server 2._

### Tabel Pengamatan IP / Layanan

| Device Name        | Protokol / Layanan        | Port | Jalur Akses             | Keterangan                                          |
| ------------------ | ------------------------- | ---- | ----------------------- | --------------------------------------------------- |
| ubuntu-server-1-24 | SFTP (SSH Subsystem)      | 2201 | 192.168.230.143         | Isolasi direktori bersama `/var/sftp/shared/upload` |
| ubuntu-server-1-24 | Cockpit Central Dashboard | 9090 | https://172.16.2.2:9090 | Memantau kesehatan Server 1 & Server 2              |
| ubuntu-server-2-24 | Cockpit Agent             | 9090 | 172.16.2.3:9090         | Terhubung ke Cockpit Master                         |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Pembuatan Grup Terisolasi & Struktur Jail SFTP Bersama

- **Penjelasan Singkat:** Membuat grup sistem khusus SFTP, membuat akun pengguna tanpa direktori _home_ dan tanpa akses _shell_, serta menyiapkan struktur direktori _chroot_.
- **Perintah CLI / Konfigurasi:**

```bash
# 1. Membuat folder bersama (shared folder) untuk direktori jail
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /var/sftp/shared/upload

# 2. Mengatur kepemilikan direktori root chroot ke root:root (Syarat mutlak chroot)
ubuntu@ubuntu-server-1-24:~$ sudo chown root:root /var/sftp/shared
ubuntu@ubuntu-server-1-24:~$ sudo chmod 755 /var/sftp/shared

# 3. Mengatur kepemilikan subfolder upload agar bisa diakses oleh grup sftp
ubuntu@ubuntu-server-1-24:~$ sudo chown root:sftpgroup /var/sftp/shared/upload
ubuntu@ubuntu-server-1-24:~$ sudo chmod 755 /var/sftp/shared/upload

# 4. Mengatur kepemilikan subfolder sharing-folder agar bisa diakses oleh grup sftp dengan full permission
ubuntu@ubuntu-server-1-24:~$ sudo chown root:sftpgroup /var/sftp/shared/upload/sharing-folder
ubuntu@ubuntu-server-1-24:~$ sudo chmod 775 /var/sftp/shared/upload/sharing-folder
```

Membuat grup, user tanpa _home_ (`-M`), dan memblokir shell (`-s /usr/sbin/nologin`):

```bash
# Membuat grup sftp
ubuntu@ubuntu-server-1-24:~$ sudo groupadd sftpgroup

# Membuat user pertama dan kedua yang masuk ke dalam sftpgroup
ubuntu@ubuntu-server-1-24:~$ sudo useradd -G sftpgroup -M -s /usr/sbin/nologin sftpuser1
ubuntu@ubuntu-server-1-24:~$ sudo useradd -G sftpgroup -M -s /usr/sbin/nologin sftpuser2

# Memberikan kata sandi untuk masing-masing user
ubuntu@ubuntu-server-1-24:~$ sudo passwd sftpuser1
ubuntu@ubuntu-server-1-24:~$ sudo passwd sftpuser2
```

Verifikasi pembuatan user dan grup:

```bash
ubuntu@ubuntu-server-1-24:~$ getent group | grep sftpgroup
ubuntu@ubuntu-server-1-24:~$ tail -n 5 /etc/passwd
```

### Langkah 2: Konfigurasi Aturan Chroot Berbasis Grup pada SSH Daemon

- **Penjelasan Singkat:** Menerapkan aturan _Match Group_ pada konfigurasi SSH agar seluruh anggota `sftpgroup` otomatis dibatasi ruang geraknya ke dalam direktori jail yang ditentukan.
- **Perintah CLI / Konfigurasi:**

```bash
# Membuka konfigurasi SSH daemon
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/ssh/sshd_config
```

Tambahkan konfigurasi _Match Group_ di bagian paling bawah berkas:

```ini
Match Group sftpgroup
    ForceCommand internal-sftp
    ChrootDirectory /var/sftp/shared/upload
    PermitTTY no
    X11Forwarding no
    AllowTcpForwarding no
```

Terapkan perubahan dengan merestart layanan SSH socket:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart ssh.socket
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart ssh
```

### Langkah 3: Otomasi Penjadwalan Tugas (Cron Job)

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

### Langkah 4: Instalasi Dashboard Cockpit (Master & Slave)

- **Penjelasan Singkat:** Menginstal antarmuka grafis berbasis web pada kedua server (`ubuntu-server-1-24` dan `ubuntu-server-2-24`) agar administrator dapat memantau kesehatan klaster secara terpusat.
- **Perintah CLI / Konfigurasi:**

Jalankan perintah ini pada **kedua server** (Server 1 dan Server 2):

```bash
# Menginstal paket Cockpit
ubuntu@ubuntu-server-x-24:~$ sudo apt update && sudo apt install cockpit -y

# Memulai dan mengaktifkan layanan Cockpit
ubuntu@ubuntu-server-x-24:~$ sudo systemctl enable --now cockpit
```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Akses SFTP (FileZilla / WinSCP)

Buka aplikasi **FileZilla** atau **WinSCP** pada Windows Host, lalu masukkan parameter berikut:

- **Protocol:** SFTP - SSH File Transfer Protocol
- **Host:** `172.16.2.2` (IP NAT Server 1)
- **Port:** `2201` (Port custom SSH yang telah diatur di Modul 01)
- **User / Pass:** `sftpuser1` (atau `sftpuser2`) / _password yang telah dibuat_

![SFTP Filezilla Site Manager Config](/assets/phase-1-sandbox/image/FileZilla.PNG)

> _Contoh konfigurasi Site Manager pada FileZilla._

![SFTP WinSCP Config](/assets/phase-1-sandbox/image/WinSCP.PNG)

> _Contoh konfigurasi Session WinSCP._

![SFTP Connection Success Verification](/assets/phase-1-sandbox/image/FileZilla1.PNG)

> _Tampilan success connection pada FileZilla._

![SFTP Connection Success Verification](/assets/phase-1-sandbox/image/WinSCP1.PNG)

> _Tampilan success connection pada WinSCP._

### Pengujian Dasbor Terpusat Cockpit

1. Buka browser pada Windows Host, akses alamat dasbor Server 1: `https://172.16.2.2:9090`
2. Masuk menggunakan kredensial _root_ atau _sudo user_ Server 1.
3. Untuk memantau Server 2 dari satu layar, pilih menu **Dashboard** atau **Server List**, lalu tambahkan koneksi ke Server 2 (`172.16.2.3`).

![Cockpit Web Access Verification](/assets/phase-1-sandbox/image/Cockpit.PNG)

> _[Instruksi]: Masukkan tangkapan layar sukses membuka halaman login Cockpit di web browser._

![Cockpit Multi-Server Integration Verification](/assets/phase-1-sandbox/image/Cockpit2.PNG)

> _[Instruksi]: Masukkan tangkapan layar panel Cockpit Server 1 yang berhasil mendeteksi dan menampilkan status metrik Server 2 di dalam satu dasbor._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **SFTP Gagal Terhubung (Fatal: Connection reset by peer):**
    - _Gejala:_ FileZilla/WinSCP menolak koneksi secara instan sesaat setelah memasukkan kredensial `sftpuser1`.
    - _Penyebab:_ Hak akses direktori yang didefinisikan sebagai `ChrootDirectory` (`/var/sftp/shared/upload`) tidak bersih dimiliki oleh `root` atau struktur kepemilikan grupnya salah [7].
    - _Solusi:_ Periksa kembali izin direktori menggunakan `namei -l /var/sftp/shared/upload`. Pastikan tingkat direktori root mutlak milik `root:root` dengan izin `755`, dan subfolder _upload_ diatur ke `root:sftpgroup` dengan izin `755`.
