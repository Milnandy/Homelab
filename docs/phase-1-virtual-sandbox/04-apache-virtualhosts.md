# Modul 04: Apache2 Web Server, Virtual Hosts, & SSL SAN

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Mengonfigurasi web server Apache2 untuk melayani multi-domain (Virtual Hosts), mengonfigurasi Otoritas Sertifikat lokal (_Local Root CA_), serta menerbitkan sertifikat SSL yang mendukung banyak subdomain menggunakan metode _Subject Alternative Name_ (SAN) [10].
- **Skenario / Case Study:** Perusahaan ingin menjalankan dua aplikasi web yang berbeda secara internal: situs portofolio (`www.milnandy.local`) dan game interaktif (`games.milnandy.local`) menggunakan satu IP server tunggal. Selain itu, kedua situs tersebut harus diamankan menggunakan protokol HTTPS tanpa memicu peringatan bahaya (_SSL Warning/Untrusted_) dari browser kerja client.
- **Kebutuhan Perangkat / Prasyarat:**
  - Server Apache2 (`apache2`)
  - OpenSSL Utility
  - Arsip zip situs web (Portofolio & Game 2048)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 04 Topology](../assets/04-apache-virtualhosts-topology.png)

> _[Placeholder Gambar]: Skema akses HTTPS dari browser Windows Client ke domain Virtual Host pada Server 1, yang divalidasi oleh Root CA yang telah terinstal pada Windows Certificate Store._

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Pengiriman Konten & Ekstraksi Web

- **Penjelasan Singkat:** Mentransfer berkas web statis dari komputer host menggunakan SCP custom port, lalu mengekstraknya ke direktori Apache.
- **Perintah CLI / Konfigurasi:**

```cmd
:: Transfer arsip portofolio menggunakan SCP port 2201
C:\Windows\system32> scp -P 2201 startbootstrap-creative-master.zip milnandy@172.16.1.2:~
```

Kembali ke server Ubuntu:

```bash
# Memindahkan dan mengekstrak berkas web ke folder Apache
ubuntu@ubuntu-server-1-24:~$ sudo mv ~/startbootstrap-creative-master.zip /tmp/
ubuntu@ubuntu-server-1-24:~$ cd /tmp && unzip startbootstrap-creative-master.zip
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /var/www/www.milnandy
ubuntu@ubuntu-server-1-24:~$ sudo mv startbootstrap-creative-master /var/www/www.milnandy/
```

### Langkah 2: Konfigurasi Virtual Hosts Apache2

- **Penjelasan Singkat:** Menduplikasi file konfigurasi default dan menyesuaikan parameter `ServerName` serta `DocumentRoot` untuk masing-masing situs.
- **Perintah CLI / Konfigurasi:**

```bash
# Masuk ke direktori konfigurasi situs Apache
ubuntu@ubuntu-server-1-24:~$ cd /etc/apache2/sites-available/
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo cp 000-default.conf www.milnandy.conf
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo nano www.milnandy.conf
```

Konfigurasi Virtual Host `/etc/apache2/sites-available/www.milnandy.conf`:

```apache
<VirtualHost *:80>
    ServerName www.milnandy.local
    DocumentRoot /var/www/www.milnandy/startbootstrap-creative-master/dist/
    ErrorLog ${APACHE_LOG_DIR}/www_error.log
    CustomLog ${APACHE_LOG_DIR}/www_access.log combined
</VirtualHost>
```

Aktifkan konfigurasi situs baru dan muat ulang Apache:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo a2dissite 000-default
ubuntu@ubuntu-server-1-24:~$ sudo a2ensite www.milnandy.conf
ubuntu@ubuntu-server-1-24:~$ sudo systemctl reload apache2
```

### Langkah 3: Pembuatan Otoritas Sertifikat Lokal (Local Root CA)

- **Penjelasan Singkat:** Membuat kunci privat dan sertifikat Root CA yang bertindak sebagai penerbit sertifikat tepercaya bagi domain internal kita.
- **Perintah CLI / Konfigurasi:**

```bash
# Membuat folder penyimpanan sertifikat
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /etc/ssl/milnandy.local && cd /etc/ssl/milnandy.local

# Membuat Private Key Root CA lokal (RSA 4096-bit)
ubuntu@ubuntu-server-1-24:/etc/ssl/milnandy.local$ sudo openssl genrsa -out rootCA.key 4096

# Membuat Sertifikat Root CA lokal self-signed (Berlaku 1 tahun)
ubuntu@ubuntu-server-1-24:/etc/ssl/milnandy.local$ sudo openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 365 -out rootCA.crt -subj "/C=ID/ST=Jakarta/L=Jakarta/O=Milnandy Lab/OU=IT/CN=Milnandy Root CA"
```

### Langkah 4: Pembuatan Sertifikat SSL Multi-Domain (SAN)

- **Penjelasan Singkat:** Menyusun konfigurasi DNS perluasan agar satu file sertifikat SSL valid untuk beberapa nama domain sekaligus (Subject Alternative Name) [10].
- **Perintah CLI / Konfigurasi:**

```bash
# Membuat file konfigurasi perluasan SAN
ubuntu@ubuntu-server-1-24:/etc/ssl/milnandy.local$ sudo nano san.cnf
```

Isi dari berkas `san.cnf` [10]:

```ini
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = req_distinguished_name
req_extensions     = req_ext

[req_distinguished_name]
C  = ID
ST = Jakarta
L  = Jakarta
O  = Milnandy Lab
CN = milnandy.local

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = milnandy.local
DNS.2 = www.milnandy.local
DNS.3 = blog.milnandy.local
DNS.4 = games.milnandy.local
```

Tandatangani sertifikat domain menggunakan Root CA lokal [10]:

```bash
# 1. Membuat Key & CSR (Certificate Signing Request) untuk domain
ubuntu@ubuntu-server-1-24:/etc/ssl/milnandy.local$ sudo openssl req -new -nodes -out milnandy.csr -newkey rsa:2048 -keyout milnandy.key -config san.cnf

# 2. Melakukan penandatanganan CSR menggunakan kunci Root CA kita (Masa aktif 825 hari)
ubuntu@ubuntu-server-1-24:/etc/ssl/milnandy.local$ sudo openssl x509 -req -in milnandy.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out milnandy.crt -days 825 -sha256 -extfile san.cnf -extensions req_ext
```

### Langkah 5: Implementasi SSL pada VirtualHost Apache2

- **Penjelasan Singkat:** Mengaktifkan modul SSL Apache dan memetakan kunci serta sertifikat yang telah dibuat ke port HTTPS (443).
- **Perintah CLI / Konfigurasi:**

```bash
# Mengaktifkan modul SSL Apache
ubuntu@ubuntu-server-1-24:~$ sudo a2enmod ssl

# Membuka file konfigurasi VirtualHost SSL baru
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/apache2/sites-available/blog-ssl.conf
```

Isi konfigurasi VirtualHost SSL:

```apache
<VirtualHost *:443>
    ServerName blog.milnandy.local
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/ssl/milnandy.local/milnandy.crt
    SSLCertificateKeyFile /etc/ssl/milnandy.local/milnandy.key

    ErrorLog ${APACHE_LOG_DIR}/blog_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/blog_ssl_access.log combined
</VirtualHost>
```

Aktifkan konfigurasi dan muat ulang layanan:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo a2ensite blog-ssl.conf
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart apache2
```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian HTTPS & Sertifikat Domain

```bash
# Memverifikasi port 443 telah aktif mendengarkan koneksi
ubuntu@ubuntu-server-1-24:~$ ss -tulpn | grep apache2
```

![HTTPS Verification on Browser](../assets/04-apache-virtualhosts-verification.png)

> _[Tugas Dokumentasi]: Pasang tangkapan layar akses HTTPS aman tanpa error keamanan (gembok hijau/aman) ke domain blog.milnandy.local setelah menginstal file rootCA.crt di Windows Trusted Root Certification Authorities._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Gagal Restart Apache (SSL Session Error):**
    - _Gejala:_ Layanan Apache gagal menyala saat di-restart setelah konfigurasi SSL aktif.
    - _Penyebab:_ Lupa mengaktifkan modul SSL di Apache (`a2enmod ssl`) atau jalur penulisan berkas sertifikat salah.
    - _Solusi:_ Periksa kembali sintaks konfigurasi menggunakan perintah `sudo apache2ctl configtest`. Jika muncul peringatan terkait direktif SSL, aktifkan modul menggunakan `sudo a2enmod ssl` lalu periksa kembali penulisan file sertifikat Anda.
