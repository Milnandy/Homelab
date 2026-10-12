---

# Modul 03: Redundansi DNS Server BIND9 Master-Slave

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Membangun layanan resolusi nama internal menggunakan BIND9, mendesain arsitektur Master-Slave untuk toleransi kesalahan (*fault-tolerance*), mengatur *forwarders* dan *recursion* keamanan jaringan, serta menerapkan teknik instalasi BIND9 secara *offline* (menggunakan *cache* paket `.deb`) pada Server Slave yang diisolasi akses internetnya.
- **Skenario / Case Study:** Dalam infrastruktur berskala besar, kegagalan sistem DNS dapat menyebabkan seluruh aplikasi kehilangan koneksi. Skenario ini diatasi dengan merancang dua server DNS (Master dan Slave). Karena Server Slave diisolasi akses internetnya, paket instalasi BIND9 ditransfer secara lokal dari Server Master. Ketika DNS Master tidak aktif, DNS Slave akan secara otomatis melayani permintaan resolusi nama tanpa intervensi manual.
- **Kebutuhan Perangkat / Prasyarat:**
  - 2 Unit VM Ubuntu 24.04 LTS (Master: 172.16.2.2, Slave: 172.16.2.3)
  - Paket BIND9 (`bind9`, `bind9-utils`)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 03 Topology](/assets/phase-1-sandbox/image/03-dns-master-slave-topology.png)

> _[Placeholder Gambar]: Diagram alur komunikasi DNS Zone Transfer menggunakan port TCP 53 dari server Master (172.16.2.2) ke server Slave (172.16.2.3)._

### Tabel Pengamatan IP / Interface

| Device Name        | Peran                 | IP Address | Domain Terdaftar           |
| ------------------ | --------------------- | ---------- | -------------------------- |
| ubuntu-server-1-24 | DNS Master (Primary)  | 172.16.2.2 | milnandy.local             |
| ubuntu-server-2-24 | DNS Slave (Secondary) | 172.16.2.3 | milnandy.local (Replicate) |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Instalasi & Deklarasi Zona pada DNS Master

- **Penjelasan Singkat:** Menginstal paket BIND9 dan mendefinisikan direktori zona kustom pada konfigurasi utama BIND.
- **Perintah CLI / Konfigurasi:**

```bash
# Memperbarui paket dan menginstal BIND9 beserta utilitasnya
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install bind9 bind9-utils -y

# Menambahkan deklarasi zona kustom ke berkas konfigurasi utama BIND
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/bind/named.conf
```

Tambahkan baris berikut di bagian akhir berkas:

```text
include "/etc/bind/milnandy.local/milnandy.local.zone";
```

### Langkah 2: Konfigurasi Opsi Global BIND (`named.conf.options`)

- **Penjelasan Singkat:** Mengatur _forwarders_ (penerusan query ke gateway/DNS publik), pembatasan akses kueri hanya untuk subnet internal, serta menonaktifkan validasi DNSSEC yang terkadang bermasalah dengan _gateway_ lokal.
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/bind/named.conf.options
```

Sesuaikan isi berkas menjadi seperti berikut:

```text
options {
        directory "/var/cache/bind";

        forwarders {
                172.16.2.1;
        //      1.1.1.1;
        //      8.8.8.8;
        };

        allow-query { 172.16.2.0/24; 127.0.0.1; localhost; };
        allow-recursion { 172.16.2.0/24; 127.0.0.1; localhost; };
        recursion yes;

        dnssec-validation no;   # Dinonaktifkan karena penyesuaian gateway lokal

        listen-on-v6 { none; };
        listen-on { 127.0.0.1; 172.16.2.2; };
};
```

### Langkah 3: Konfigurasi File Zona Forward & Reverse pada Master

- **Penjelasan Singkat:** Membuat berkas deklarasi zona Master-Slave, pemetaan nama domain ke IP (Forward), dan pemetaan IP ke nama domain (Reverse).
- **Perintah CLI / Konfigurasi:**

```bash
# Membuat folder penyimpanan zona kustom
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /etc/bind/milnandy.local
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/bind/milnandy.local/milnandy.local.zone
```

Isi dari `/etc/bind/milnandy.local/milnandy.local.zone`:

```text
zone "milnandy.local" {
    type master;
    file "/etc/bind/milnandy.local/milnandy.local";
    allow-transfer { 172.16.2.3; };
    also-notify { 172.16.2.3; };
};

zone "2.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/milnandy.local/db.2.16.172";
    allow-transfer { 172.16.2.3; };
    also-notify { 172.16.2.3; };
};
```

Membuat dan mengisi berkas Forward `/etc/bind/milnandy.local/milnandy.local`:

```text
$TTL    604800
@       IN      SOA     milnandy.local. root.milnandy.local. (
                     2026100810         ; Serial (Format: YYYYMMDDNN)
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      milnandy.local.
@       IN      A       172.16.2.2
@       IN      A       172.16.2.3      ; IP DNS Slave
www     IN      A       172.16.2.2
blog    IN      A       172.16.2.2
games   IN      A       172.16.2.2
game    IN      CNAME   games.milnandy.local.
```

Membuat dan mengisi berkas Reverse `/etc/bind/milnandy.local/db.2.16.172`:

```text
$TTL    604800
@       IN      SOA     milnandy.local. root.milnandy.local. (
                     2026100810         ; Serial (Format: YYYYMMDDNN)
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      milnandy.local.
2       IN      PTR     milnandy.local.
3       IN      PTR     milnandy.local.         ; IP DNS Slave
2       IN      PTR     www.milnandy.local.
2       IN      PTR     blog.milnandy.local.
2       IN      PTR     games.milnandy.local.
```

### Langkah 4: Validasi Sintaks & Pengaturan Resolv.conf pada Master

- **Penjelasan Singkat:** Memastikan seluruh file konfigurasi BIND bebas dari kesalahan sintaks sebelum layanan dijalankan.
- **Perintah CLI / Konfigurasi:**

```bash
# Cek sintaks keseluruhan konfigurasi BIND
ubuntu@ubuntu-server-1-24:~$ sudo named-checkconf

# Cek sintaks file zona forward
ubuntu@ubuntu-server-1-24:~$ sudo named-checkzone milnandy.local /etc/bind/milnandy.local/milnandy.local

# Cek sintaks file zona reverse
ubuntu@ubuntu-server-1-24:~$ sudo named-checkzone 2.16.172.in-addr.arpa /etc/bind/milnandy.local/db.2.16.172
```

Mengonfigurasi `/etc/resolv.conf` pada Master:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo rm /etc/resolv.conf
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/resolv.conf
```

Isi berkas:

```text
nameserver 172.16.2.2
nameserver 8.8.8.8
```

Restart dan aktifkan layanan DNS Master:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart bind9
ubuntu@ubuntu-server-1-24:~$ sudo systemctl enable named
```

---

### Langkah 5: Transfer Paket Instalasi Offline ke Server Slave

- **Penjelasan Singkat:** Karena Server Slave (`ubuntu-server-2-24`) diisolasi akses internetnya (`ens33` didisconnect), paket instalasi `.deb` BIND9 diunduh ulang di cache apt Server Master, lalu ditransfer secara lokal menggunakan `scp`.
- **Perintah CLI / Konfigurasi:**

Pada **Server Master (172.16.2.2)**, unduh paket tanpa menginstalnya (_download-only_):

```bash
# Install ulang paket jika sebelumnya terhapus melalui 'apt autoremove'
ubuntu@ubuntu-server-1-24:~$ sudo apt-get install --download-only --reinstall bind9 bind9-utils -y

# Kirim paket arsip .deb ke folder /tmp di Server Slave via port SSH custom (2201)
ubuntu@ubuntu-server-1-24:~$ sudo scp -P 2201 /var/cache/apt/archives/*bind*.deb milnandy@172.16.2.3:/tmp/
```

Pada **Server Slave (172.16.2.3)**, instal paket secara offline menggunakan `dpkg`:

```bash
ubuntu@ubuntu-server-2-24:~$ cd /tmp
ubuntu@ubuntu-server-2-24:~$ sudo mv *bind*.deb /var/cache/apt/archives/
ubuntu@ubuntu-server-2-24:~$ cd /var/cache/apt/archives/
ubuntu@ubuntu-server-2-24:~$ sudo dpkg -i *.deb
```

### Langkah 6: Konfigurasi DNS Slave (Replikasi Master-Slave)

- **Penjelasan Singkat:** Mengonfigurasi server kedua sebagai Slave agar menduplikasi zona dari Master serta menyesuaikan hak akses direktori _cache_ [8].
- **Perintah CLI / Konfigurasi:**

Tambahkan direktori zona pada `/etc/bind/named.conf` di **Server Slave**:

```bash
ubuntu@ubuntu-server-2-24:~$ sudo nano /etc/bind/named.conf
```

Tambahkan di baris akhir:

```text
include "/etc/bind/milnandy.local/milnandy.local.zone";
```

Buat Berkas:

```bash
# Membuat folder penyimpanan zona kustom
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /etc/bind/milnandy.local
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/bind/milnandy.local/milnandy.local.zone
```

Isi deklarasi zona slave pada `/etc/bind/milnandy.local/milnandy.local.zone`:

```text
zone "milnandy.local" {
    type slave;
    file "/var/cache/bind/milnandy.local";
    masters { 172.16.2.2; };
};

zone "2.16.172.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.2.16.172";
    masters { 172.16.2.2; };
};
```

Atur hak kepemilikan folder _cache_ dan direktori konfigurasi agar BIND9 dapat menulis file salinan zona Master:

```bash
ubuntu@ubuntu-server-2-24:~$ sudo chown -R bind:bind /var/cache/bind
ubuntu@ubuntu-server-2-24:~$ sudo chmod 775 /var/cache/bind

ubuntu@ubuntu-server-2-24:~$ sudo chown -R root:bind /etc/bind/milnandy.local
ubuntu@ubuntu-server-2-24:~$ sudo chmod 755 /etc/bind/milnandy.local

# Restart layanan BIND9 di Server Slave
ubuntu@ubuntu-server-2-24:~$ sudo systemctl restart bind9
```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Resolusi Domain & Replikasi Zona

```bash
# Verifikasi rekaman zona tersalin otomatis di folder cache Slave
ubuntu@ubuntu-server-2-24:~$ ls -la /var/cache/bind/

# Pengujian resolusi nama menggunakan nslookup
ubuntu@ubuntu-server-1-24:~$ nslookup milnandy.local
ubuntu@ubuntu-server-1-24:~$ nslookup 172.16.2.2

# Menguji query langsung ke server lokal menggunakan dig
ubuntu@ubuntu-server-1-24:~$ dig archive.ubuntu.com @127.0.0.1
```

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Reverse Lookup Menghasilkan Error `NXDOMAIN`:**
    - _Gejala:_ Perintah `nslookup 172.16.2.2` tidak menghasilkan domain `www.milnandy.local` melainkan error `NXDOMAIN`.
    - _Solusi:_ Pada berkas reverse zona kelas C (`/24`), kolom pertama PTR record tidak boleh diisi dengan alamat IP penuh (`172.16.2.2`), melainkan wajib diisi dengan oktet terakhir saja dari IP host tersebut (angka `2` dan `3`) [8].
2.  **Zona Slave Gagal Melakukan Transfer (_Zone Transfer Failed_):**
    - _Gejala:_ Berkas salinan zona tidak kunjung muncul di `/var/cache/bind/` pada Server Slave.
    - _Penyebab:_ Direktori `/var/cache/bind` tidak memiliki izin tulis (_write permission_) untuk pengguna daemon `bind`, atau parameter `allow-transfer` di Master belum menunjuk ke IP Slave (`172.16.2.3`).
    - _Solusi:_ Periksa kembali kepemilikan folder dengan `sudo chown -R bind:bind /var/cache/bind` dan pastikan direktif `allow-transfer` serta `also-notify` sudah terpasang dengan benar di sisi Master.
