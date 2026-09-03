# Modul 03: Redundansi DNS Server BIND9 Master-Slave

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Membangun layanan resolusi nama internal menggunakan BIND9, mendesain arsitektur Master-Slave untuk toleransi kesalahan (_fault-tolerance_), serta memahami penulisan rekaman forward dan reverse DNS [8].
- **Skenario / Case Study:** Dalam infrastruktur berskala besar, kegagalan sistem DNS dapat menyebabkan seluruh aplikasi kehilangan koneksi. Skenario ini diatasi dengan merancang dua server DNS (Master dan Slave). Ketika DNS Master tidak aktif, DNS Slave akan secara otomatis melayani permintaan resolusi nama tanpa intervensi manual.
- **Kebutuhan Perangkat / Prasyarat:**
  - 2 Unit VM Ubuntu 24.04 LTS (172.16.1.2 dan 172.16.1.3)
  - Paket BIND9 (`bind9`, `bind9-utils`)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 03 Topology](../assets/03-dns-master-slave-topology.png)

> _[Placeholder Gambar]: Diagram alur komunikasi DNS Zone Transfer menggunakan port TCP 53 dari server Master (172.16.1.2) ke server Slave (172.16.1.3)._

### Tabel Pengamatan IP / Interface

| Device Name        | Peran                 | IP Address | Domain Terdaftar           |
| ------------------ | --------------------- | ---------- | -------------------------- |
| ubuntu-server-1-24 | DNS Master (Primary)  | 172.16.1.2 | milnandy.local             |
| ubuntu-server-2-24 | DNS Slave (Secondary) | 172.16.1.3 | milnandy.local (Replicate) |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Instalasi & Deklarasi Zona pada DNS Master

- **Penjelasan Singkat:** Menginstal paket BIND9 dan mendefinisikan batas lingkup zona baru yang akan dikelola oleh server Master.
- **Perintah CLI / Konfigurasi:**

```bash
# Menginstal paket BIND9
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install bind9 bind9-utils -y

# Menambahkan deklarasi zona kustom ke berkas konfigurasi utama
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/bind/named.conf
```

Tambahkan baris berikut di akhir berkas:

```text
include "/etc/bind/milnandy.local/milnandy.local.zone";
```

### Langkah 2: Konfigurasi File Zona Forward & Reverse pada Master

- **Penjelasan Singkat:** Membuat berkas pemetaan nama domain ke IP (Forward) dan pemetaan IP ke nama domain (Reverse) [8].
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
    allow-transfer { 172.16.1.3; };
    also-notify { 172.16.1.3; };
};

zone "1.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/milnandy.local/db.1.16.172";
    allow-transfer { 172.16.1.3; };
    also-notify { 172.16.1.3; };
};
```

Isi berkas Forward `/etc/bind/milnandy.local/milnandy.local`:

```text
$TTL    604800
@       IN      SOA     milnandy.local. root.milnandy.local. (
                     2026083101         ; Serial (Format: YYYYMMDDNN)
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      milnandy.local.
@       IN      A       172.16.1.2
www     IN      A       172.16.1.2
blog    IN      A       172.16.1.2
games   IN      A       172.16.1.2
game    IN      CNAME   games
```

Isi berkas Reverse `/etc/bind/milnandy.local/db.1.16.172`:

```text
$TTL    604800
@       IN      SOA     milnandy.local. root.milnandy.local. (
                     2026083101         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      milnandy.local.
2       IN      PTR     milnandy.local.
2       IN      PTR     www.milnandy.local.
2       IN      PTR     blog.milnandy.local.
2       IN      PTR     games.milnandy.local.
3       IN      PTR     milnandy.local.
```

### Langkah 3: Konfigurasi DNS Slave (ubuntu-server-2-24)

- **Penjelasan Singkat:** Mengonfigurasi server kedua sebagai Slave agar secara otomatis menduplikasi (_replicate_) seluruh catatan DNS dari server Master.
- **Perintah CLI / Konfigurasi:**

```bash
# Membuka berkas deklarasi zona pada server Slave
ubuntu@ubuntu-server-2-24:~$ sudo nano /etc/bind/milnandy.local/milnandy.local.zone
```

Isi konfigurasi pada server Slave:

```text
zone "milnandy.local" {
    type slave;
    file "milnandy.local";
    masters { 172.16.1.2; };
};

zone "1.16.172.in-addr.arpa" {
    type slave;
    file "db.1.16.172";
    masters { 172.16.1.2; };
};
```

Aktifkan dan restart layanan:

```bash
ubuntu@ubuntu-server-2-24:~$ sudo systemctl restart named
```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Validasi Konfigurasi & Resolusi Domain

```bash
# Validasi sintaks berkas konfigurasi di Master
ubuntu@ubuntu-server-1-24:~$ named-checkconf /etc/bind/milnandy.local/milnandy.local.zone

# Validasi isi file zona forward
ubuntu@ubuntu-server-1-24:~$ named-checkzone milnandy.local /etc/bind/milnandy.local/milnandy.local

# Pengujian resolusi nama (Forward & Reverse) menggunakan nslookup
ubuntu@ubuntu-server-1-24:~$ nslookup www.milnandy.local
ubuntu@ubuntu-server-1-24:~$ nslookup 172.16.1.2
```

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Reverse Lookup Menghasilkan Error NXDOMAIN:**
    - _Gejala:_ Perintah `nslookup 172.16.1.2` tidak menghasilkan domain `www.milnandy.local` melainkan error `NXDOMAIN` [8].
    - _Solusi:_ Pada berkas reverse zona kelas C (`/24`), kolom pertama PTR record tidak boleh diisi dengan alamat IP penuh (`172.16.1.2`), melainkan wajib diisi dengan oktet terakhir saja dari IP host tersebut (angka `2`) [8].
2.  **VM Kehilangan Koneksi Internet Luar Setelah Bypass resolv.conf:**
    - _Gejala:_ Perintah `apt update` gagal karena nama host repositori Ubuntu tidak dapat ditemukan.
    - _Penyebab:_ Bypassing `resolv.conf` langsung mengarah ke DNS lokal yang belum dikonfigurasi dengan _forwarders_ internet [9].
    - _Solusi:_ Edit berkas `/etc/resolv.conf` dan sisipkan alamat DNS server publik (`8.8.8.8`) sebagai resolver sekunder di baris berikutnya [9].
