# Modul 01: Inisialisasi Sandbox OS & Jaringan Ganda (Dual-NIC)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Melakukan instalasi dasar Linux Server teroptimasi, mengonfigurasi arsitektur jaringan ganda (Dual-NIC), mengaktifkan fungsi _IP Forwarding_ pada Server 1 agar bertindak sebagai _router/gateway_, menginstal dan mengamankan akses SSH, mengonfigurasi rute statis pada sistem host Windows, serta menyiapkan Server 2 (Slave) di bawah kendali _gateway_ internal.
- **Skenario / Case Study:** Komputer host memiliki keterbatasan RAM (8 GB). Untuk menghemat penggunaan memori, VM Ubuntu dikonfigurasi secara _headless_ (tanpa antarmuka grafis). Dibuat dua server (Master dan Slave) di segmen LAN terisolasi. Server 1 bertindak sebagai router/gateway bagi Server 2 setelah akses internet luar pada Server 2 diputus untuk keperluan pengujian isolasi jaringan internal.
- **Kebutuhan Perangkat / Prasyarat:**
  - VMware Workstation Pro (Personal Use)
  - ISO Ubuntu Desktop 24.04 LTS (Dikonversi ke CLI)
  - Command Prompt Windows (Administrator privileges)
  - **Dua VM Ubuntu 24.04 LTS (ubuntu-server-1-24 dan ubuntu-server-2-24) telah dikloning dari Modul 00.**

## 🗺️ Topologi Jaringan & Arsitektur

![Phase1.01 gambar 1](/assets/phase-1-sandbox/image/Server1-2-network.drawio.png)

> _Topologi menunjukkan laptop host Windows terhubung ke Server 1 melalui dua adapter: Network Adapter 1 (NAT) dan Network Adapter 2 (LAN Segment). Server 2 terhubung ke Server 1 melalui Network Adapter 2 (LAN Segment), di mana Server 1 bertindak sebagai gateway perantara._

### Tabel Pengamatan IP / Interface

| Device Name        | Interface    | IP Address / Subnet    | VLAN / Note                         |
| ------------------ | ------------ | ---------------------- | ----------------------------------- |
| Windows Host       | VMnet8 (NAT) | 192.168.230.1 /24      | Default Gateway VM untuk akses luar |
| ubuntu-server-1-24 | ens33 (NAT)  | DHCP (192.168.230.143) | Jalur pembaruan sistem & paket luar |
| ubuntu-server-1-24 | ens37 (LAN)  | 172.16.2.2 /24         | Segmen lokal internal terisolasi    |
| ubuntu-server-2-24 | ens33 (NAT)  | Disconnected           | Jalur luar dimatikan (Isolated)     |
| ubuntu-server-2-24 | ens37 (LAN)  | 172.16.2.3 /24         | Mengarahkan gateway ke Server 1     |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Alokasi Perangkat Virtual & Optimasi RAM (GUI ke CLI)

- **Penjelasan Singkat:** Mengubah mode boot default dari grafis (GUI) ke teks (CLI) untuk memangkas konsumsi RAM dari ~1.8 GB menjadi ~300 MB. Optimalisasi ini sangat krusial mengingat keterbatasan RAM host (8 GB).
- **Perintah CLI / Konfigurasi:**

```bash
# Mengubah default target systemd ke multi-user (CLI)
ubuntu@ubuntu-server-1-24:~$ sudo systemctl set-default multi-user.target

# Melakukan restart untuk memverifikasi perubahan
ubuntu@ubuntu-server-1-24:~$ sudo reboot
```

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

### Langkah 4: Mengaktifkan IP Forwarding pada Server 1 (Gateway Perantara)

- **Penjelasan Singkat:** Agar Server 1 dapat meneruskan paket data (_routing_) dari Server 2 menuju jaringan luar atau sebaliknya, kernel Linux harus diizinkan untuk meneruskan paket lintas interface (_IP Forwarding_).
- **Perintah CLI / Konfigurasi:**

```bash
# Mengaktifkan IP forwarding secara langsung (*runtime*)
ubuntu@ubuntu-server-1-24:~$ sudo sysctl net.ipv4.ip_forward=1

# Mengaktifkan secara permanen agar tidak hilang saat server reboot
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/sysctl.conf
```

Cari baris berikut, lalu hapus tanda pagar (`#`) di depannya:

```ini
net.ipv4.ip_forward=1
```

Simpan file, lalu terapkan perubahan permanen:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo sysctl -p
```

### Langkah 5: Penambahan Rute Statis pada Windows Host

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

![Phase1.01 gambar 2](/assets/phase-1-sandbox/image/route-print.PNG)

> _Output command "route print" pada Command Prompt Windows dan pastikan rute persisten menuju subnet 172.16.2.2 terdaftar._

### Langkah 6: Konfigurasi DNS Resolver, Instalasi, & Aktivasi Layanan OpenSSH Server

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

### Langkah 7: Instalasi & Aktivasi Layanan OpenSSH Server

- **Penjelasan Singkat:** Mengunduh paket `openssh-server` dari repositori resmi Ubuntu dan memastikan layanan SSH aktif serta berjalan otomatis saat sistem pertama kali menyala.
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

![Phase1.01 gambar 3](/assets/phase-1-sandbox/image/ssh-status.PNG)

### Langkah 8: Migrasi Port Layanan SSH (systemd socket)

- **Penjelasan Singkat:** Mulai Ubuntu 22.04/24.04, penanganan port SSH dikontrol oleh unit `ssh.socket`. Konfigurasi port pada `ssh.socket` adalah metode yang direkomendasikan dan memiliki prioritas untuk sistem yang menggunakan _socket activation_.
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

### Langkah 9: Konfigurasi & Optimalisasi Server 2 (Slave)

- **Penjelasan Singkat:** Menyiapkan VM kedua (`ubuntu-server-2-24`) dengan konfigurasi serupa Server 1, namun dengan penyesuaian IP, pemutusan akses internet luar (`ens33`), serta mengarahkan _gateway_ internalnya langsung ke Server 1 (`172.16.2.2`).
- **Perintah CLI / Konfigurasi:**

1.  **Nyalakan VM `ubuntu-server-2-24` dan sesuaikan Netplan-nya:**
    - Ubah IP `ens37` menjadi `172.16.2.3/24`.
    - **Penyesuaian Gateway:** Ubah rute _default via_ di Netplan Server 2 agar mengarah ke **Server 1 (`172.16.2.2`)**, bukan lagi ke IP gateway NAT luar.

    Contoh Netplan Server 2 (`/etc/netplan/01-network-manager-all.yaml`):

    ```yaml
    network:
      version: 2
      renderer: NetworkManager
      ethernets:
        ens33:
          dhcp4: true
        ens37:
          dhcp4: false
          addresses:
            - 172.16.2.3/24
          nameservers:
            addresses: [172.16.2.2, 8.8.8.8]
          routes:
            - to: default
              via: 172.16.2.2
    ```

    Terapkan Netplan: `sudo netplan apply`

2.  **Mengisolasi Akses Internet Luar (Disconnect ens33):**
    - Matikan koneksi internet langsung pada adapter `ens33` di Server 2 agar benar-benar terisolasi dan bergantung sepenuhnya pada jalur _IP Forwarding_ Server 1.

    ```bash
    # Memeriksa status interface jaringan yang dikelola NetworkManager
    ubuntu@ubuntu-server-2-24:~$ nmcli device status

    # Menonaktifkan adapter ens33 (NAT) untuk memutus akses internet luar langsung
    ubuntu@ubuntu-server-2-24:~$ sudo nmcli device disconnect ens33

    # Memverifikasi status dari network device ens33, seharusnya "disconnected"
    ubuntu@ubuntu-server-2-24:~$ nmcli device status
    ```

3.  **Mengonfigurasi `resolv.conf` untuk Server 2:**

    ```bash
    # Menghapus symlink atau file resolv.conf yang lama
    ubuntu@ubuntu-server-2-24:~$ sudo rm /etc/resolv.conf

    # Mengarahkan nameserver ke Server 1
    ubuntu@ubuntu-server-2-24:~$ sudo echo "nameserver 172.16.2.2" > /etc/resolv.conf
    ```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Konektivitas & Port

```bash
# Memverifikasi port SSH telah bermigrasi ke port 2201 pada Server 1
ubuntu@ubuntu-server-1-24:~$ ss -tulpn | grep ssh

# Melakukan uji koneksi dari host Windows ke Server 1 menggunakan port custom
C:\Windows\system32> ssh milnandy@172.16.2.2 -p 2201
```

![Phase1.01 gambar 4](/assets/phase-1-sandbox/image/ssh.PNG)

> _Pada Cmd Windows sukses terhubung ke Server 1 (172.16.2.2) lewat SSH pada port 2201._

**Pengujian Koneksi SSH Jumping (Host -> Server 1 -> Server 2)**

```bash
# Melakukan koneksi SSH dari Windows Host, melompat melalui Server 1, ke Server 2
C:\Windows\system32> ssh -J milnandy@172.16.2.2:2201 milnandy@172.16.2.3 -p 2201
```

![Phase1.01 gambar 5](/assets/phase-1-sandbox/image/ssh-jump.PNG)

> _Pada Cmd Windows menampilkan koneksi SSH sukses ke Server 2 (172.16.2.3) melalui Server 1 (172.16.2.2) yang keduanya pada port 2201._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Error `apt update` (Gagal Mengunduh Repositori / Temporary failure resolving):**
    - _Gejala:_ Perintah `sudo apt update` gagal dengan pesan `failed to fetch https://id.archive.ubuntu.com` atau `Temporary failure resolving 'id.archive.ubuntu.com'`.
    - _Penyebab:_ Berkas penunjuk DNS sistem (`/etc/resolv.conf`) belum dikonfigurasi dengan benar setelah penerapan Netplan.
    - _Solusi:_ Hapus berkas `/etc/resolv.conf` yang lama, lalu buat baru yang mengarahkan `nameserver` ke IP lokal dan DNS publik cadangan (`8.8.8.8`) seperti yang telah diuraikan pada **Langkah 6**.
2.  **Server 2 Tidak Bisa Terhubung ke Internet Meskipun ens33 Dimatikan:**
    - _Gejala:_ Server 2 tidak dapat melakukan `ping 8.8.8.8` setelah adapter NAT-nya di-_disconnect_.
    - _Penyebab:_ Fitur penerusan paket (_IP Forwarding_) pada Server 1 belum diaktifkan, atau _gateway_ pada Netplan Server 2 belum mengarah ke IP Server 1 (`172.16.2.2`).
    - _Solusi:_ Pastikan parameter `net.ipv4.ip_forward=1` aktif di `/etc/sysctl.conf` pada Server 1, dan pastikan konfigurasi `routes` di Netplan Server 2 sudah menggunakan _via 172.16.2.2_.
3.  **Koneksi SSH Jumping Gagal ke Server 2:**
    - _Gejala:_ Saat mencoba SSH Jumping ke Server 2, koneksi terputus atau gagal di tengah jalan.
    - _Solusi:_ Verifikasi ulang port SSH Server 2 (`2201`), pastikan _route print_ di Windows Host sudah mencakup rute ke IP `172.16.2.3`, dan uji koneksi internal menggunakan `ping 172.16.2.3` dari Server 1.
