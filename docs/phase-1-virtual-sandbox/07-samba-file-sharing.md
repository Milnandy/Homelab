# Modul 07: File Sharing Enterprise dengan Samba (Security & ACL)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Membangun layanan berbagi berkas (_file sharing_) lintas platform menggunakan Samba, mengonfigurasi multi-tipe _share_ (Public, Department Private, Personal Home Directories, dan Read-Only Archive), menerapkan hak akses berbasis grup sistem dengan **Setgid bit**, membersihkan peringatan NetBIOS melalui penyesuaian _hostname/hosts_, serta mengelola pemetaan _network drive_ pada sistem operasi Windows [13].
- **Skenario / Case Study:** Perusahaan memerlukan infrastruktur penyimpanan bersama yang terstruktur dan aman. Administrator membuat folder publik untuk pengumuman umum, folder privat khusus tim _developer_ yang diatur menggunakan hak akses grup dengan pewarisan otomatis (_Setgid_), direktori arsip perusahaan yang bersifat baca-saja (_Read-Only_), serta direktori privat personal untuk setiap _user_. Seluruhnya diuji coba dan dipetakan secara simultan dari klien Windows Host.
- **Kebutuhan Perangkat / Prasyarat:**
  - Paket Samba (`samba`)
  - Windows 10/11 Host Client (File Explorer & CMD)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 07 Topology](../assets/phase-1-sandbox/07-samba-file-sharing-topology.png)

> _[Placeholder Gambar]: Diagram yang memvisualisasikan Windows Client memetakan berbagai jenis share Samba (`PublicGeneral`, `DeveloperTeam`, `ArsipSOP`) dari server `172.16.2.2` / `server1-24`._

### Tabel Pengamatan Struktur Direktori Samba

| Nama Share (`smb.conf`) | Path Fisik Direktori   | Tipe Akses             | Target Pengguna                           |
| ----------------------- | ---------------------- | ---------------------- | ----------------------------------------- |
| `[PublicGeneral]`       | `/srv/samba/public`    | Publik (Guest OK)      | Seluruh Karyawan (Tanpa Login)            |
| `[DeveloperTeam]`       | `/srv/samba/developer` | Privat (Berbasis Grup) | Anggota Grup `developer` (`dev1`, `dev2`) |
| `[homes]`               | `/home/%U`             | Privat per User        | Setiap User Sistem Aktif                  |
| `[ArsipSOP]`            | `/srv/samba/arsip`     | Baca Saja (Read-Only)  | Anggota Grup `karyawan`                   |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Pembersihan Dependensi & Instalasi Samba

- **Penjelasan Singkat:** Memastikan paket instalasi bersih dari korupsi cache apt, lalu menginstal layanan Samba.
- **Perintah CLI / Konfigurasi:**

```bash
# Membersihkan cache dan memperbaiki potensi kerusakan dependensi
ubuntu@ubuntu-server-1-24:~$ sudo apt clean && sudo apt autoclean
ubuntu@ubuntu-server-1-24:~$ sudo apt --fix-broken install -y
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install samba -y

# Memverifikasi status layanan Samba (smbd)
ubuntu@ubuntu-server-1-24:~$ sudo systemctl status smbd
```

### Langkah 2: Pembuatan Struktur Direktori, Grup, dan Pengguna Sistem

- **Penjelasan Singkat:** Membuat struktur folder standar di `/srv/samba/`, membuat grup sistem (`developer`, `karyawan`), mendaftarkan user baru, serta mengonfigurasi **Setgid bit (`2770`)** agar file baru di dalam folder otomatis mewarisi kepemilikan grup [13].
- **Perintah CLI / Konfigurasi:**

```bash
# Membuat grup sistem dan pengguna
ubuntu@ubuntu-server-1-24:~$ sudo groupadd developer
ubuntu@ubuntu-server-1-24:~$ sudo groupadd karyawan
ubuntu@ubuntu-server-1-24:~$ sudo useradd -m -G developer dev1
ubuntu@ubuntu-server-1-24:~$ sudo useradd -m -G developer dev2
ubuntu@ubuntu-server-1-24:~$ sudo useradd -m -G karyawan dev1
ubuntu@ubuntu-server-1-24:~$ sudo useradd -m -G karyawan dev2

# Mendaftarkan password user ke database Samba
ubuntu@ubuntu-server-1-24:~$ sudo smbpasswd -a dev1
ubuntu@ubuntu-server-1-24:~$ sudo smbpasswd -a dev2
```

Membuat struktur direktori fisik dan menerapkan _Setgid bit_:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /srv/samba/{public,developer,arsip}
ubuntu@ubuntu-server-1-24:~$ sudo chown -R root:developer /srv/samba/developer
ubuntu@ubuntu-server-1-24:~$ sudo chmod -R 2770 /srv/samba/developer
```

_(Catatan: Angka `2` di depan `2770` adalah **Setgid bit**, memastikan setiap file/folder baru di dalam direktori ini otomatis mewarisi grup `developer`)._

### Langkah 3: Konfigurasi Komprehensif Berkas Samba (`smb.conf`)

- **Penjelasan Singkat:** Mengonfigurasi empat jenis _share_ berbeda di dalam berkas konfigurasi utama Samba.
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/samba/smb.conf
```

Tambahkan konfigurasi berikut di bagian paling bawah berkas:

```ini
# 1. Jenis Public Share (Bebas / Tanpa Login)
[PublicGeneral]
path = /srv/samba/public
browsable = yes
writable = yes
guest ok = yes
read only = no

# 2. Jenis Department/Team Private Share (Berbasis Grup)
[DeveloperTeam]
path = /srv/samba/developer
browsable = yes
writable = yes
valid users = @developer
force group = developer
create mask = 0660
directory mask = 0770
read only = no

# 3. Jenis Personal / Home Directory Share (Kerahasiaan Tinggi per User)
[homes]
comment = Home Directories
browseable = no
writable = yes
valid users = %S
create mask = 0700
directory mask = 0700

# 4. Jenis Read-Only Archive / Distribution Share (Arsip Perusahaan)
[ArsipSOP]
path = /srv/samba/arsip
browsable = yes
writable = no
read only = yes
valid users = @karyawan
```

### Langkah 4: Pembersihan Peringatan NetBIOS (_NetBIOS Warning Cleanup_)

- **Penjelasan Singkat:** Menyesuaikan _hostname_ sistem agar sesuai dengan standar resolusi lokal, menghilangkan peringatan NetBIOS saat _testparm_ dijalankan.
- **Perintah CLI / Konfigurasi:**

```bash
# Mengubah hostname server
ubuntu@ubuntu-server-1-24:~$ sudo hostnamectl set-hostname server1-24

# Memetakan hostname ke localhost pada /etc/hosts
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/hosts
```

Tambahkan/sesuaikan baris berikut:

```text
127.0.1.1 server1-24
```

Uji validasi konfigurasi Samba tanpa error/warning:

```bash
ubuntu@ubuntu-server-1-24:~$ testparm
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart smbd
```

### Langkah 5: Mengisi Data Percobaan pada Direktori Share

- **Penjelasan Singkat:** Menyiapkan file uji coba di masing-masing direktori share untuk memvalidasi hak akses.
- **Perintah CLI / Konfigurasi:**

```bash
# Isi folder Public
sudo echo "Halo! Ini adalah file pengumuman umum untuk semua orang di jaringan kantor." | sudo tee /srv/samba/public/pengumuman.txt

# Isi folder Developer
sudo tee /srv/samba/developer/deploy-test.sh << 'EOF'
#!/bin/bash
echo 'Source code backend v1.0 berjalan normal.'
EOF
sudo chmod 660 /srv/samba/developer/deploy-test.sh

# Isi folder Arsip SOP
sudo echo "SOP-01: Prosedur Keamanan Jaringan dan Server Perusahaan." | sudo tee /srv/samba/arsip/sop-keamanan.txt
sudo chown -R root:root /srv/samba/arsip && sudo chmod -R 0755 /srv/samba/arsip
```

---

## 🔍 Verifikasi & Troubleshooting

### Pemetaan _Network Drive_ pada Klien Windows

Untuk menghindari konflik sesi SMB tunggal di Windows, gunakan perintah berikut melalui **CMD (Command Prompt)**:

```cmd
:: 1. Bersihkan sisa sesi SMB yang menggantung pada Windows Host
C:\Windows\system32> net use * /delete /yes

:: 2. Map Folder DeveloperTeam menggunakan kredensial user dev1 via alamat IP
C:\Windows\system32> net use Z: \\172.16.2.2\DeveloperTeam /user:dev1

:: 3. Map Folder ArsipSOP menggunakan kredensial user dev1 via Hostname/FQDN
C:\Windows\system32> net use Y: \\server1-24\ArsipSOP /user:dev1
```

> ![Samba Windows Mapping Verification](../assets/phase-1-sandbox/07-samba-mapping-verification.png)
> _[Instruksi]: Lampirkan tangkapan layar File Explorer Windows yang menampilkan Drive Z: (DeveloperTeam) dan Drive Y: (ArsipSOP) berhasil terhubung secara bersamaan._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Konflik Akses Kredensial Berbeda pada IP yang Sama di Windows:**
    - _Gejala:_ Saat mencoba memetakan dua _share_ berbeda ke IP server yang sama dengan user berbeda, Windows menolak dengan pesan _Multiple connections to a server... are not allowed_.
    - _Solusi:_ Bersihkan cache sesi menggunakan `net use * /delete /yes`. Akses salah satu share menggunakan alamat IP (`\\172.16.2.2\...`) dan share lainnya menggunakan nama hostname (`\\server1-24\...`) [13].
2.  **Anggota Grup Tidak Bisa Menulis File di Folder `DeveloperTeam`:**
    - _Gejala:_ User `dev2` bisa masuk, tetapi gagal membuat atau menyimpan file baru.
    - _Penyebab:_ Direktori fisik belum mengaktifkan _Setgid bit_, sehingga file baru tidak mewarisi grup `developer`.
    - _Solusi:_ Pastikan direktori diatur dengan perintah `sudo chmod 2770 /srv/samba/developer` dan kepemilikan grup diatur dengan `sudo chown -R root:developer /srv/samba/developer`.
