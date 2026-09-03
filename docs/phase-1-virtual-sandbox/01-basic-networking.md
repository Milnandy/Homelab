Langkah Anda untuk menyisipkan proses instalasi `openssh-server` dan peninjauan tabel routing Windows (`route print`) sangat tepat. Tanpa adanya langkah instalasi eksplisit, dokumen ini akan kurang dapat direproduksi (_not fully reproducible_) oleh pembaca yang memulai dari instalasi Ubuntu minimal yang belum menyertakan SSH server bawaan.

Berikut adalah revisi lengkap **Modul 01** yang telah disempurnakan. Saya telah menambahkan **Langkah 5 (Instalasi & Aktivasi)** secara detail dan menggeser langkah migrasi port ke **Langkah 6**, lengkap dengan keselarasan perintah serta verifikasinya sesuai standar penulisan teknis profesional.

---

# Modul 01: Inisialisasi Sandbox OS & Jaringan Ganda (Dual-NIC)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Melakukan instalasi dasar Linux Server teroptimasi, mengonfigurasi arsitektur jaringan ganda (Dual-NIC), mengonfigurasi rute statis pada sistem host Windows, serta menginstal dan bermigrasi port default SSH pada sistem yang menggunakan systemd socket.
- **Skenario / Case Study:** Komputer host memiliki keterbatasan RAM (8 GB). Untuk menghemat penggunaan memori, VM Ubuntu dikonfigurasi secara _headless_ (tanpa antarmuka grafis). Selain itu, untuk mensimulasikan segmen jaringan lokal (_Intranet_) yang aman, dibuat satu segmen LAN khusus yang terisolasi dari internet luar, namun tetap dapat diremote secara aman oleh host Windows menggunakan _static routing_.
- **Kebutuhan Perangkat / Prasyarat:**
  - VMware Workstation Pro (Personal Use)
  - ISO Ubuntu Desktop 24.04 LTS (Dikonversi ke CLI)
  - Command Prompt Windows (Administrator privileges)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 01 Topology](../assets/phase-1-sandbox/01-basic-networking-topology.png)

> _[Placeholder Gambar]: Gambarlah diagram yang menunjukkan laptop host Windows terhubung ke VM melalui dua adapter: Network Adapter 1 (NAT) dan Network Adapter 2 (LAN Segment)._

### Tabel Pengamatan IP / Interface

| Device Name        | Interface    | IP Address / Subnet    | VLAN / Note                         |
| ------------------ | ------------ | ---------------------- | ----------------------------------- |
| Windows Host       | VMnet8 (NAT) | 192.168.230.1 /24      | Default Gateway VM untuk akses luar |
| ubuntu-server-1-24 | ens33 (NAT)  | DHCP (192.168.230.134) | Jalur pembaruan sistem & paket      |
| ubuntu-server-1-24 | ens37 (LAN)  | 172.16.1.2 /24         | Segmen lokal internal terisolasi    |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Alokasi Perangkat Virtual & Optimasi RAM (GUI ke CLI)

- **Penjelasan Singkat:** Mengubah mode boot default dari grafis (GUI) ke teks (CLI) untuk memangkas konsumsi RAM dari ~1.8 GB menjadi ~300 MB.
- **Perintah CLI / Konfigurasi:**

```bash
# Mengubah default target systemd ke multi-user (CLI)
ubuntu@ubuntu-server-1-24:~$ sudo systemctl set-default multi-user.target

# Melakukan restart untuk memverifikasi perubahan
ubuntu@ubuntu-server-1-24:~$ sudo reboot
```

### Langkah 2: Mengubah Nama Identitas Server (Hostname)

- **Penjelasan Singkat:** Penamaan server yang terstruktur memudahkan identifikasi perangkat dalam lingkungan produksi.
- **Perintah CLI / Konfigurasi:**

```bash
# Mengubah hostname sistem
ubuntu@ubuntu-server-1-24:~$ sudo hostnamectl set-hostname ubuntu-server-1-24

# Memperbarui sesi bash agar nama baru langsung tampil di prompt terminal
ubuntu@ubuntu-server-1-24:~$ bash
```

### Langkah 3: Konfigurasi Antarmuka Jaringan Ganda (Netplan)

- **Penjelasan Singkat:** Mengonfigurasi interface DHCP pada ens33 untuk internet dan IP statis pada ens37 untuk jaringan segmen LAN internal.
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
        - 172.16.1.2/24
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

- **Penjelasan Singkat:** Agar host Windows dapat menjangkau IP segmen LAN VM (`172.16.1.2`), kita perlu mendaftarkan rute khusus di dalam tabel routing Windows yang mengarah ke gateway IP NAT VM.
- **Perintah CLI / Konfigurasi:**

```cmd
:: Jalankan Command Prompt Windows sebagai Administrator
C:\Windows\system32> route -p add 172.16.1.2 mask 255.255.255.255 192.168.230.134
```

- **Untuk Melihat Hasil Konfigurasi:**

```cmd
C:\Windows\system32> route print
```

> ![Modul 01 Isi dari "route print"](../assets/phase-1-sandbox/route-print.png)
> _[Instruksi]: Tangkap layar output command "route print" pada Command Prompt Windows dan pastikan rute persisten menuju subnet 172.16.1.2 terdaftar._

### Langkah 5: Instalasi & Aktivasi Layanan OpenSSH Server

- **Penjelasan Singkat:** Mengunduh paket `openssh-server` dari repositori resmi Ubuntu dan memastikan layanan SSH aktif serta berjalan secara otomatis saat sistem pertama kali menyala.
- **Perintah CLI / Konfigurasi:**

```bash
# Memperbarui indeks paket repositori lokal
ubuntu@ubuntu-server-1-24:~$ sudo apt update

# Mengunduh dan menginstal layanan OpenSSH Server
ubuntu@ubuntu-server-1-24:~$ sudo apt install openssh-server -y

# Memastikan layanan SSH aktif dan berjalan otomatis saat booting
ubuntu@ubuntu-server-1-24:~$ sudo systemctl enable ssh
ubuntu@ubuntu-server-1-24:~$ sudo systemctl start ssh
```

### Langkah 6: Migrasi Port Layanan SSH (systemd socket)

- **Penjelasan Singkat:** Mulai Ubuntu 22.04/24.04, penanganan port SSH dikontrol oleh unit `ssh.socket` [6]. Perubahan port pada `/etc/ssh/sshd_config` harus disertai dengan _override_ konfigurasi socket systemd [6].
- **Perintah CLI / Konfigurasi:**

```bash
# Membuka menu pengeditan unit socket SSH
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

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Konektivitas & Port

```bash
# Memverifikasi port SSH telah bermigrasi ke port 2201
ubuntu@ubuntu-server-1-24:~$ ss -tulpn | grep ssh

# Melakukan uji koneksi dari host Windows menggunakan port custom
C:\Windows\system32> ssh milnandy@172.16.1.2 -p 2201
```

> ![Modul 01 Verifikasi SSH](../assets/phase-1-sandbox/01-ssh-verification.png)
> _[Instruksi]: Ambil tangkapan layar terminal Cmd Windows saat sukses terhubung ke server 172.16.1.2 lewat SSH pada port 2201._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Error Netplan Apply (Indentation Warning):**
    - _Gejala:_ Muncul kegagalan _parsing_ YAML saat menjalankan `netplan apply`.
    - _Solusi:_ Periksa spasi pada berkas YAML. Pastikan tidak menggunakan tombol Tab, melainkan murni spasi biasa (indentasi standar adalah 2 spasi per tingkat kedalaman).
2.  **Error Netplan Apply (Netplan configuration should NOT be accessible by others)**
    - _Gejala:_ _Permissions for /etc/netplan/01-network-manager-all.yaml are too open. Netplan configuration should NOT be accessible by others._
    - _Solusi:_ Di Ubuntu 24.04, secara default izin pada berkas `01-network-manager-all.yaml` adalah 644. Ubah izin berkas konfigurasi jaringan tersebut menjadi lebih ketat menggunakan instruksi: `sudo chmod 600 /etc/netplan/01-network-manager-all.yaml`.
3.  **SSH Port Tetap Berjalan di Port 22:**
    - _Gejala:_ Port SSH tidak berubah meskipun file `/etc/ssh/sshd_config` telah dimodifikasi [6].
    - _Solusi:_ Ubuntu 24.04 mengabaikan parameter port di `sshd_config` karena kontrolnya telah diserahkan sepenuhnya ke `ssh.socket` [6]. Konfigurasi wajib dilakukan menggunakan instruksi `systemctl edit ssh.socket` [6].

```

```
