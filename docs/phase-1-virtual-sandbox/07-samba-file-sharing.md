# Modul 06: File Sharing Enterprise dengan Samba (Security & ACL)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Membangun layanan berbagi berkas lintas platform dengan Samba, mengimplementasikan batasan keamanan unggah berdasarkan ekstensi (_veto files_), dan mengonfigurasi otorisasi berbasis grup sistem (_Group ACLs_) [13].
- **Skenario / Case Study:** Perusahaan membutuhkan folder bersama yang bebas diakses tanpa kata sandi (_public share_) untuk pertukaran dokumen umum, tetapi harus terlindung dari pengunggahan malware/virus bertipe executable. Selain itu, dibuat folder rahasia khusus divisi IT dan HRD yang hanya bisa dibuka oleh anggota masing-masing divisi, serta dapat terpetakan secara simultan pada Windows Host.
- **Kebutuhan Perangkat / Prasyarat:**
  - Samba Server (`samba`)
  - Windows 10 Host Client

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 06 Topology](../assets/06-samba-file-sharing-topology.png)

> _[Placeholder Gambar]: Diagram yang memvisualisasikan Windows Client memetakan dua map drive virtual secara bersamaan menggunakan IP (kredensial divisi IT) dan Hostname (kredensial divisi HRD)._

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Pembuatan Folder Bersama Publik & Proteksi Malware

- **Penjelasan Singkat:** Membuat folder publik tanpa otorisasi kata sandi, namun menerapkan filter _veto_ untuk memblokir berkas eksekusi (_executable_) berbahaya.
- **Perintah CLI / Konfigurasi:**

```bash
# Menginstal paket Samba
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install samba -y

# Membuat folder publik
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /home/share && sudo chmod 777 /home/share

# Mengonfigurasi berkas utama Samba
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/samba/smb.conf
```

Isi konfigurasi blok `[Share]` di akhir berkas:

```ini
[Share]
path = /home/share
writeable = yes
guest = yes
guest only = yes
force create mode = 777
force directory mode = 777
veto files = /*.exe/*.dll/*.bat/*.vbs/*.tmp/
```

### Langkah 2: Pembuatan Hak Akses Berbasis Grup (HRD & IT)

- **Penjelasan Singkat:** Membuat grup sistem operasi, menambahkan user ke grup, dan mengunci izin direktori fisik menggunakan kepemilikan grup.
- **Perintah CLI / Konfigurasi:**

```bash
# Membuat grup divisi
ubuntu@ubuntu-server-1-24:~$ sudo groupadd hrd
ubuntu@ubuntu-server-1-24:~$ sudo groupadd divisi-it

# Membuat pengguna sistem baru
ubuntu@ubuntu-server-1-24:~$ sudo useradd -m -g hrd idn
ubuntu@ubuntu-server-1-24:~$ sudo useradd -m -g divisi-it idn-it

# Mendaftarkan password user ke database Samba
ubuntu@ubuntu-server-1-24:~$ sudo smbpasswd -a idn
ubuntu@ubuntu-server-1-24:~$ sudo smbpasswd -a idn-it
```

Membuat dan mengunci izin folder secara fisik (770):

```bash
# Folder HRD
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /home/hrd
ubuntu@ubuntu-server-1-24:~$ sudo chgrp hrd /home/hrd
ubuntu@ubuntu-server-1-24:~$ sudo chmod 770 /home/hrd

# Folder IT
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /home/office
ubuntu@ubuntu-server-1-24:~$ sudo chgrp divisi-it /home/office
ubuntu@ubuntu-server-1-24:~$ sudo chmod 770 /home/office
```

### Langkah 3: Konfigurasi Keamanan Folder Privat Samba

- **Penjelasan Singkat:** Mengonfigurasi Samba agar memvalidasi kepemilikan grup sebelum mengizinkan penulisan atau pembacaan berkas.
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/samba/smb.conf
```

Tambahkan konfigurasi folder privat di akhir berkas:

```ini
[HRD]
path = /home/hrd
writeable = yes
guest ok = no
valid users = @hrd
force group = hrd
force create mode = 770
force directory mode = 770
inherit permissions = yes

[IT]
path = /home/office
writeable = yes
guest ok = no
valid users = @divisi-it
force group = divisi-it
force create mode = 770
force directory mode = 770
inherit permissions = yes
```

Terapkan perubahan:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart smbd.service
```

---

## 🔍 Verifikasi & Troubleshooting

### Mengatasi Konflik Multi-Session SMB di Windows

- **Penjelasan Singkat:** Menjelaskan solusi cerdik untuk memetakan dua folder dengan kredensial berbeda dari satu Windows Client.
- **Perintah CLI / Konfigurasi:**

```cmd
:: 1. Bersihkan sisa sesi SMB yang menggantung/terkunci pada Windows Host
C:\Windows\system32> net use * /delete /yes

:: 2. Map Folder HRD menggunakan Kredensial User "idn" via alamat IP
C:\Windows\system32> net use Z: \\172.16.1.2\HRD /user:idn idnmantab

:: 3. Map Folder IT menggunakan Kredensial User "idn-it" via alamat Hostname/FQDN
C:\Windows\system32> net use Y: \\ubuntu-server-1-24\IT /user:idn-it idnmantab
```

_Mengapa ini bekerja?_ Windows melihat pemanggilan via IP dan via Hostname sebagai dua entitas jaringan yang berbeda, memungkinkan bypass limitasi default _single-identity_ SMB Windows.

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Samba Share Tidak Bisa Diakses (Access Denied):**
    - _Gejala:_ Pengguna ditolak masuk saat mengakses folder dari Windows Explorer meskipun password sudah benar.
    - _Penyebab:_ Izin direktori di tingkat sistem operasi Linux (_Linux File Permissions_) masih dimiliki oleh `root` dan mengunci akses grup.
    - _Solusi:_ Jalankan perintah `sudo chgrp [nama-grup] /path/folder` dan ubah hak aksesnya ke `770` agar anggota grup dapat membaca dan menulis.
