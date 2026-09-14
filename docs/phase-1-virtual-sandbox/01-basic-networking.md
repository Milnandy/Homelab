# Modul 01: Inisialisasi Sandbox OS & Jaringan Ganda (Dual-NIC)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Melakukan instalasi dasar Linux Server teroptimasi, mengonfigurasi arsitektur jaringan ganda (Dual-NIC), menginstal dan mengamankan akses SSH, mengonfigurasi rute statis pada sistem host Windows, serta menyiapkan Server 2 (Slave) untuk kebutuhan klaster internal.
- **Skenario / Case Study:** Komputer host memiliki keterbatasan RAM (8 GB). Untuk menghemat penggunaan memori, VM Ubuntu dikonfigurasi secara _headless_ (tanpa antarmuka grafis). Dibuat dua server (Master dan Slave) di segmen LAN terisolasi yang dapat diremote secara aman. Server Slave dinonaktifkan akses internet luarnya untuk kebutuhan pengujian isolasi jaringan internal.
- **Kebutuhan Perangkat / Prasyarat:**
  - VMware Workstation Pro (Personal Use)
  - ISO Ubuntu Desktop 24.04 LTS (Dikonversi ke CLI)
  - Command Prompt Windows (Administrator privileges)
  - **Dua VM Ubuntu 24.04 LTS (ubuntu-server-1-24 dan ubuntu-server-2-24) telah dikloning dari Modul 00.**

## 🗺️ Topologi Jaringan & Arsitektur

![Phase1.01 gambar 1](/assets/phase-1-sandbox/Server1-2-network.drawio.png)

> _Topologi menunjukkan laptop host Windows terhubung ke Server 1 melalui dua adapter: Network Adapter 1 (NAT) dan Network Adapter 2 (LAN Segment) sedangkan Server 2 hanya terhubungk ke Server 1 melalui Network Adapter 2 (LAN Segment). Pastikan IP Address dan Interface sesuai dengan tabel di bawah._

### Tabel Pengamatan IP / Interface

| Device Name        | Interface    | IP Address / Subnet    | VLAN / Note                         |
| ------------------ | ------------ | ---------------------- | ----------------------------------- |
| Windows Host       | VMnet8 (NAT) | 192.168.230.1 /24      | Default Gateway VM untuk akses luar |
| ubuntu-server-1-24 | ens33 (NAT)  | DHCP (192.168.230.143) | Jalur pembaruan sistem & paket      |
| ubuntu-server-1-24 | ens37 (LAN)  | 172.16.2.2 /24         | Segmen lokal internal terisolasi    |
| ubuntu-server-2-24 | ens33 (NAT)  | DHCP (192.168.230.160) | Jalur pembaruan sistem & paket      |
| ubuntu-server-2-24 | ens37 (LAN)  | 172.16.2.3 /24         | Segmen lokal internal terisolasi    |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Alokasi Perangkat Virtual & Optimasi RAM (GUI ke CLI)

- **Penjelasan Singkat:** Mengubah mode boot default dari grafis (GUI) ke teks (CLI) untuk memangkas konsumsi RAM dari ~1.8 GB menjadi ~300 MB. Optimalisasi ini sangat krusial mengingat keterbatasan RAM host (8 GB).
- **Perintah CLI / Konfigurasi:**

````bash
# Mengubah default target systemd ke multi-user (CLI)
ubuntu@ubuntu-server-1-24:~$ sudo systemctl set-default multi-user.target

# Melakukan restart untuk memverifikasi perubahan
ubuntu@ubuntu-server-1-24:~$ sudo reboot```
````

### Langkah 2: Mengubah Nama Identitas Server (Hostname)

- **Penjelasan Singkat:** Penamaan server yang terstruktur memudahkan identifikasi perangkat dalam lingkungan produksi dan jaringan internal.
- **Perintah CLI / Konfigurasi:**

```bash
# Mengubah hostname sistem
ubuntu@ubuntu-server-1-24:~$ sudo hostnamectl set-hostname ubuntu-server-1-24

# Memperbarui sesi bash agar nama baru langsung tampil di prompt terminal
ubuntu@ubuntu-server-1-24:~$ bash
```

### Langkah 3: Konfigurasi Antarmuka Jaringan Ganda (Netplan)

- **Penjelasan Singkat:** Mengonfigurasi interface DHCP pada `ens33` untuk akses internet dan IP statis pada `ens37` untuk jaringan segmen LAN internal.
- **Perintah CLI / Konfigurasi:**

```bash
# Membuka file konfigurasi Netplan
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/netplan/01-network-manager-all.yaml
```

Isi konfigurasi `/etc/netplan/01-network-manager-all.yaml`:

```yaml
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      dhcp4: false
      addresses:
        - 172.16.2.2/24
      nameservers:
        addresses: [4.2.2.2, 8.8.8.8]
      routes:
        - to: default
          via: 192.168.230.143
```

Terapkan konfigurasi:

```bash
# Mengetatkan hak akses file konfigurasi demi keamanan sistem
ubuntu@ubuntu-server-1-24:~$ sudo chmod 600 /etc/netplan/01-network-manager-all.yaml

# Menerapkan konfigurasi jaringan baru
ubuntu@ubuntu-server-1-24:~$ sudo netplan apply
```

### Langkah 4: Penambahan Rute Statis pada Windows Host

- **Penjelasan Singkat:** Agar host Windows dapat menjangkau IP segmen LAN VM (`172.16.2.2`), kita perlu mendaftarkan rute khusus di dalam tabel routing Windows yang mengarah ke gateway IP NAT VM.
- **Perintah CLI / Konfigurasi:**

```cmd
:: Jalankan Command Prompt Windows sebagai Administrator
C:\Windows\system32> route -p add 172.16.2.2 mask 255.255.255.255 192.168.230.143
```

- **Untuk Melihat Hasil Konfigurasi:**

```cmd
C:\Windows\system32> route print
```

![Phase1.01 gambar 2](/assets/phase-1-sandbox/route-print.PNG)

> _Output command "route print" pada Command Prompt Windows dan pastikan rute persisten menuju subnet 172.16.2.2 terdaftar._

### Langkah 5: Konfigurasi DNS Resolver, Instalasi, & Aktivasi Layanan OpenSSH Server

- **Penjelasan Singkat:** Mengonfigurasi penamaan DNS (_resolver_) agar server dapat mengenali alamat repositori internet dengan lancar, mengunduh paket `openssh-server`, serta memastikan layanan SSH aktif dan berjalan otomatis saat sistem menyala.
- **Perintah CLI / Konfigurasi:**

Sebelum mengunduh paket dari internet, pastikan server menggunakan konfigurasi DNS resolver lokal dan publik yang benar agar tidak terjadi kegagalan saat menjalankan `apt update`.

```bash
# Menghapus symlink atau file resolv.conf bawaan yang belum diatur
ubuntu@ubuntu-server-1-24:~$ sudo rm /etc/resolv.conf

# Membuat dan menyimpan konfigurasi DNS baru dengan nameserver lokal dan publik
ubuntu@ubuntu-server-1-24:~$ sudo echo "nameserver 172.16.2.2
nameserver 8.8.8.8" > /etc/resolv.conf
```

### Langkah 6: Instalasi & Aktivasi Layanan OpenSSH Server

- **Penjelasan Singkat:** Mengunduh paket `openssh-server` dari repositori resmi Ubuntu dan memastikan layanan SSH aktif serta berjalan secara otomatis saat sistem pertama kali menyala.
- **Perintah CLI / Konfigurasi:**

```bash
# Memperbarui indeks paket repositori lokal
ubuntu@ubuntu-server-1-24:~$ sudo apt update

# Mengunduh dan menginstal layanan OpenSSH Server
ubuntu@ubuntu-server-1-24:~$ sudo apt install openssh-server -y

# Memastikan layanan SSH aktif dan berjalan otomatis saat booting
ubuntu@ubuntu-server-1-24:~$ sudo systemctl enable --now ssh
```

Verifikasi status service:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo systemctl status ssh
```

Output yang diharapkan seperti dibawah:

![Phase1.01 gambar 3](/assets/phase-1-sandbox/ssh-status.PNG)

### Langkah 7: Migrasi Port Layanan SSH (systemd socket)

- **Penjelasan Singkat:** Mulai Ubuntu 22.04/24.04, penanganan port SSH dikontrol oleh unit `ssh.socket`. Meskipun modifikasi pada `/etc/ssh/sshd_config` bisa dilakukan, konfigurasi port pada `ssh.socket` adalah metode yang direkomendasikan dan memiliki prioritas untuk sistem yang menggunakan _socket activation_.
- **Perintah CLI / Konfigurasi:**

```bash
# (Opsional) Mengubah port di sshd_config. Ini akan di-override oleh ssh.socket,
# namun beberapa administrator lebih memilih konsistensi konfigurasi.
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/ssh/sshd_config
# Temukan baris "#Port 22", hapus komentar ('#') dan ubah menjadi:
# Port 2201

# Memeriksa sintaks file sshd_config. Jika tidak ada output, sintaks sudah benar.
ubuntu@ubuntu-server-1-24:~$ sudo sshd -t

# (Opsional) Restart layanan SSHD jika sshd_config dimodifikasi.
# Namun, untuk port, systemd socket akan mengambil alih jika diaktifkan.
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart ssh
```

```bash
# Membuka menu pengeditan unit socket SSH (Metode utama untuk mengubah port)
ubuntu@ubuntu-server-1-24:~$ sudo systemctl edit ssh.socket
```

Tambahkan konfigurasi _override_ di dalam blok kosong:

```ini
[Socket]
ListenStream=
ListenStream=2201
```

Terapkan perubahan ke sistem:

```bash
# Muat ulang konfigurasi daemon systemd
ubuntu@ubuntu-server-1-24:~$ sudo systemctl daemon-reload

# Restart layanan socket SSH
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart ssh.socket
```

### Langkah 8: Konfigurasi & Optimalisasi Server 2 (Slave)

- **Penjelasan Singkat:** Menyiapkan VM kedua (ubuntu-server-2-24) dengan konfigurasi dasar serupa Server 1, namun dengan penyesuaian IP dan isolasi jaringan untuk mematikan akses internet luarnya.
- **Perintah CLI / Konfigurasi:**

1.  **Nyalakan VM `ubuntu-server-2-24` dan ulangi `Langkah 1` hingga `Langkah 6` dari Server 1.**
    - **Penyesuaian di `Langkah 2 (Hostname)`:** Ubah menjadi `ubuntu-server-2-24`.
    - **Penyesuaian di `Langkah 3 (Netplan)`:** Ubah IP `ens37` menjadi `172.16.2.3/24` dan `routes via` gateway NAT yang sesuai dengan IP NAT yang diterima oleh Server 2 (misalnya `192.168.230.160`).
    - **Penyesuaian di `Langkah 4 (Windows Route)`:** Tambahkan rute baru untuk `172.16.2.3` melalui gateway NAT Server 2 (misalnya `192.168.230.160`).
2.  **Mengisolasi Akses Internet Luar (Disconnect ens33):**
    - Untuk kebutuhan pengujian isolasi jaringan internal, matikan koneksi internet pada adapter `ens33` di Server 2.

    ```bash
    # Memeriksa status interface jaringan yang dikelola NetworkManager
    ubuntu@ubuntu-server-2-24:~$ nmcli device status

    # Menonaktifkan adapter ens33 (NAT) untuk memutus akses internet luar
    ubuntu@ubuntu-server-2-24:~$ sudo nmcli device disconnect ens33

    # Memverifikasi status dari network device ens33, seharusnya "disconnected"
    ubuntu@ubuntu-server-2-24:~$ nmcli device status
    ```

3.  **Mengonfigurasi `resolv.conf` untuk DNS Internal:**
    - Agar Server 2 hanya menggunakan DNS Server Master (Server 1) untuk resolusi nama, `resolv.conf` perlu diarahkan hanya ke IP `172.16.2.2`.

    ```bash
    # Menghapus symlink atau file resolv.conf yang lama
    ubuntu@ubuntu-server-2-24:~$ sudo rm /etc/resolv.conf

    # Membuat dan menyimpan konfigurasi DNS baru hanya ke DNS Master
    ubuntu@ubuntu-server-2-24:~$ sudo echo "nameserver 172.16.2.2" > /etc/resolv.conf
    ```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Konektivitas & Port

```bash
# Memverifikasi port SSH telah bermigrasi ke port 2201
ubuntu@ubuntu-server-1-24:~$ ss -tulpn | grep ssh

# Melakukan uji koneksi dari host Windows ke Server 1 menggunakan port custom
C:\Windows\system32> ssh milnandy@172.16.2.2 -p 2201
```

![Phase1.01 gambar 4](/assets/phase-1-sandbox/ssh.PNG)

> _SSH dari Laptop Host Windows menggunakan CMD ke Server 1 (172.16.2.2) pada port 2201._

**Pengujian Koneksi SSH Jumping (Host -> Server 1 -> Server 2)**

```bash
# Melakukan koneksi SSH dari Windows Host, melompat melalui Server 1, ke Server 2
C:\Windows\system32> ssh -J milnandy@172.16.2.2:2201 milnandy@172.16.2.3 -p 2201
```

![Phase1.01 gambar 5](/assets/phase-1-sandbox/ssh-jump.PNG)

> _SSH Jump dari Laptop Host Windows ke Server 2 (172.16.2.3) melalui Server 1 (172.16.2.2) yang keduanya pada port 2201._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Error `apt update` (Gagal Mengunduh Repositori / Temporary failure resolving):**
    - _Gejala:_ Perintah `sudo apt update` gagal dengan pesan `failed to fetch https://id.archive.ubuntu.com` atau `Temporary failure resolving 'id.archive.ubuntu.com'`.
    - _Penyebab:_ Berkas penunjuk DNS sistem (`/etc/resolv.conf`) belum dikonfigurasi dengan benar setelah penerapan Netplan, sehingga server tidak tahu ke mana harus bertanya saat menerjemahkan nama domain repositori Ubuntu menjadi alamat IP.
    - _Solusi:_ Hapus berkas `/etc/resolv.conf` yang lama, lalu buat baru yang mengarahkan `nameserver` ke IP lokal dan DNS publik cadangan (`8.8.8.8`) seperti yang telah diuraikan pada **Langkah 5**.
2.  **Error Netplan Apply (Indentation Warning):**
    - _Gejala:_ Muncul kegagalan _parsing_ YAML saat menjalankan `netplan apply`.
    - _Solusi:_ Periksa spasi pada berkas YAML. Pastikan tidak menggunakan tombol Tab, melainkan murni spasi biasa (indentasi standar adalah 2 spasi per tingkat kedalaman).
3.  **Error Netplan Apply (Netplan configuration should NOT be accessible by others)**
    - _Gejala:_ _Permissions for /etc/netplan/01-network-manager-all.yaml are too open. Netplan configuration should NOT be accessible by others._
    - _Solusi:_ Di Ubuntu 24.04, secara default izin pada berkas `01-network-manager-all.yaml` adalah 644. Ubah izin berkas konfigurasi jaringan tersebut menjadi lebih ketat menggunakan instruksi: `sudo chmod 600 /etc/netplan/01-network-manager-all.yaml`.
4.  **SSH Port Tetap Berjalan di Port 22:**
    - _Gejala:_ Port SSH tidak berubah meskipun file `/etc/ssh/sshd_config` telah dimodifikasi.
    - _Solusi:_ Ubuntu 24.04 mengabaikan parameter port di `sshd_config` karena kontrolnya telah diserahkan sepenuhnya ke `ssh.socket`. Konfigurasi wajib dilakukan menggunakan instruksi `systemctl edit ssh.socket`.
5.  **Koneksi SSH Jumping Gagal ke Server 2:**
    - _Gejala:_ Saat mencoba SSH Jumping ke Server 2, koneksi terputus atau gagal di tengah jalan.
    - _Penyebab:_
      1.  Port SSH pada Server 2 belum disesuaikan ke `2201`.
      2.  Server 1 tidak memiliki rute untuk menjangkau Server 2 (masalah internal LAN segment).
      3.  Gateway NAT Server 1 (192.168.230.143) tidak bisa dijangkau oleh Windows Host.
    - _Solusi:_
      1.  Verifikasi ulang langkah 6 pada Server 2 untuk memastikan port SSH sudah 2201.
      2.  Uji ping dari Server 1 ke Server 2 (`ping 172.16.2.3`) untuk memastikan konektivitas dalam segmen LAN internal.
      3.  Pastikan `route -p add` pada Windows Host sudah mencakup rute ke kedua server.

---
