# Modul 04: Apache2 Web Server, Virtual Hosts, & SSL SAN

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Mengonfigurasi web server Apache2 untuk melayani multi-domain (_Virtual Hosts_), membangun infrastruktur PKI (_Public Key Infrastructure_) tingkat enterprise menggunakan Otoritas Sertifikat lokal (_Local Root CA_), menerbitkan sertifikat SSL _wildcard/multi-domain_ berbasis _Subject Alternative Name_ (SAN), melakukan transfer paket _offline_ (Nginx) ke Server 2, menyebarkan aplikasi web statis dan interaktif pada port kustom, serta melakukan penanganan insiden sertifikat kedaluwarsa (_Certificate Expired_).
- **Skenario / Case Study:** Perusahaan ingin menjalankan beberapa aplikasi web secara internal: situs portofolio (`www.milnandy.local`), blog admin terenkripsi SSL (`blog.milnandy.local`), dan game interaktif pada port khusus (`games.milnandy.local:8001`) menggunakan infrastruktur server lokal. Agar tidak memicu peringatan keamanan (_SSL Warning/Untrusted_) dari browser kerja client, administrator membangun _Root CA_ internal dan mendaftarkannya ke dalam _Windows Trusted Root Certificate Store_. Selain itu, Server 2 disiapkan secara _offline_ untuk menjalankan web server Nginx dengan subdomain terpisah (`web2.milnandy.local`).
- **Keterangan Templating / Sumber Proyek:**
  Proyek web statis dalam modul ini dimodifikasi dari templat _open-source_ profesional:
  - [Start Bootstrap Creative](https://github.com/StartBootstrap/startbootstrap-creative) (Digunakan untuk portofolio utama `www.milnandy.local`)
  - [Start Bootstrap Resume](https://github.com/StartBootstrap/startbootstrap-resume) (Digunakan untuk situs blog/resume `blog.milnandy.local`)
  - [Gabriele Cirulli 2048](https://github.com/gabrielecirulli/2048) (Digunakan untuk server game interaktif pada `games.milnandy.local` port `8001`)
- **Kebutuhan Perangkat / Prasyarat:**
  - Server Apache2 (`apache2`) & Nginx (`nginx`)
  - OpenSSL Utility
  - Arsip zip situs web (Bootstrap Template)
  - Akses Administrator pada Windows Host

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 04 Topology](/assets/phase-1-sandbox/image/https-virtualhost-rootca-editable1.drawio.png)

> _Skema akses HTTPS dari browser Windows Client ke domain Virtual Host pada Server 1, yang divalidasi oleh Root CA internal yang terinstal pada Windows Trusted Root Store._

### Tabel Pengamatan IP / Interface & Layanan Web

| Device Name        | Layanan Web     | Port | Dokumen Root Direktori     | Catatan                                     |
| ------------------ | --------------- | ---- | -------------------------- | ------------------------------------------- |
| ubuntu-server-1-24 | Apache2 (HTTP)  | 80   | `/var/www/www.milnandy/`   | Situs Portofolio Utama                      |
| ubuntu-server-1-24 | Apache2 (HTTPS) | 443  | `/var/www/blog.milnandy/`  | Blog/Admin Panel terenkripsi SSL            |
| ubuntu-server-1-24 | Apache2 (HTTP)  | 8001 | `/var/www/games.milnandy/` | Server Game Interaktif 2048                 |
| ubuntu-server-2-24 | Nginx (HTTP)    | 80   | `/var/www/html/`           | Server Web Sekunder (`web2.milnandy.local`) |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Instalasi Apache2 & Penyiapan Struktur Direktori

- **Penjelasan Singkat:** Menginstal web server Apache2 pada Server 1 dan menyiapkan struktur direktori standar enterprise FHS (_Filesystem Hierarchy Standard_).
- **Perintah CLI / Konfigurasi:**

```bash
# Memperbarui dan menginstal Apache2
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install apache2 -y

# Memverifikasi status layanan Apache2
ubuntu@ubuntu-server-1-24:~$ sudo systemctl status apache2
```

Membuat direktori penyimpanan root web:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /var/www/www.milnandy
```

### Langkah 2: Transfer Arsip Web dari Host ke Server

- **Penjelasan Singkat:** Memindahkan file arsip (_zip_) template web dari komputer host ke direktori sementara `/tmp` di Server 1 menggunakan SCP melalui port custom `2201`.
- **Perintah CLI / Konfigurasi:**

```cmd
:: Jalankan di Command Prompt Windows Host
C:\Windows\system32> cd /d D:\KumpulanFolder\Technology
C:\Windows\system32> scp -P 2201 startbootstrap-creative-master.zip milnandy@172.16.2.2:/tmp
```

Kembali ke terminal Server 1, ekstrak file ke folder web:

```bash
ubuntu@ubuntu-server-1-24:~$ cd /var/www/www.milnandy
ubuntu@ubuntu-server-1-24:~$ sudo mv /tmp/startbootstrap-creative-master.zip .
ubuntu@ubuntu-server-1-24:~$ sudo unzip startbootstrap-creative-master.zip
```

### Langkah 3: Konfigurasi Virtual Host HTTP (`www.milnandy.local`)

- **Penjelasan Singkat:** Menyisipkan konfigurasi bawaan Apache, mengubah direktorinya ke folder portofolio, dan mendefinisikan _ServerName_.
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ cd /etc/apache2/sites-available/
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo cp -p 000-default.conf www.milnandy.conf
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo nano www.milnandy.conf
```

Isi konfigurasi `/etc/apache2/sites-available/www.milnandy.conf`:

```apache
<VirtualHost *:80>
        ServerName www.milnandy.local
        DocumentRoot /var/www/www.milnandy/startbootstrap-creative-master/dist/

        ErrorLog ${APACHE_LOG_DIR}/www.error.log
        CustomLog ${APACHE_LOG_DIR}/www.access.log combined
</VirtualHost>
```

Mengaktifkan situs dan memuat ulang Apache:

```bash
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo a2dissite 000-default.conf
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo a2ensite www.milnandy.conf
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo systemctl reload apache2
```

### Langkah 4: Penyesuaian Hosts Windows & Static Routing

- **Penjelasan Singkat:** Mendaftarkan resolusi domain lokal ke file `hosts` Windows serta memastikan komputer host dapat menjangkau subnet _LAN Segment_ VM.
- **Perintah / Konfigurasi (Windows Host):**

1. Salin berkas `C:\Windows\System32\drivers\etc\hosts` ke Desktop.
2. Buka menggunakan Notepad, tambahkan baris berikut di paling bawah:
   ```text
   172.16.2.2 milnandy.local
   ```
3. Simpan, lalu timpa (_replace_) kembali ke direktori asalnya di `C:\Windows\System32\drivers\etc\hosts`.

Menambahkan rute statis agar Windows Host bisa mengakses subnet internal 172.16.2.0/24 (buka **CMD as Administrator**):

```cmd
:: Jika menggunakan NAT Gateway VMware (misal: 192.168.230.143)
C:\Windows\system32> route add 172.16.2.0 mask 255.255.255.0 192.168.230.143 -p

:: Atau jika menggunakan MikroTik CHR (misal: 192.168.230.128)
C:\Windows\system32> route add 172.16.2.0 mask 255.255.255.0 192.168.230.128 -p
```

_Uji coba:_ Buka browser Microsoft Edge / Google Chrome, lalu akses `http://www.milnandy.local/`.

---

### Langkah 5: Pembuatan Infrastruktur PKI & SSL SAN (Enterprise Standard)

- **Penjelasan Singkat:** Membangun _Root CA_ mandiri beserta pembuatan sertifikat web dengan ekstensi SAN (_Subject Alternative Name_) agar browser mempercayai multi-domain/wildcard.
- **Perintah CLI / Konfigurasi:**

Membuat struktur direktori privat FHS:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /etc/ssl/private-ca/{private,certs,requests}
ubuntu@ubuntu-server-1-24:~$ sudo chmod 700 /etc/ssl/private-ca/private
ubuntu@ubuntu-server-1-24:~$ cd /etc/ssl/private-ca
```

**1. Membuat Konfigurasi Root CA (`root-ca.cnf`):**

```bash
sudo tee root-ca.cnf << 'EOF'
[ req ]
default_bits       = 4096
prompt             = no
default_md         = sha256
distinguished_name = dn
x509_extensions    = v3_ca

[ dn ]
C  = ID
ST = Jakarta
L  = Jakarta
O  = Milnandy Lab
OU = IT Infrastructure
CN = Milnandy Root CA

[ v3_ca ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints = critical, CA:TRUE
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
EOF
```

**2. Generate Kunci & Sertifikat Root CA (Berlaku 365 Hari):**

```bash
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo openssl genrsa -out private/root-ca.key 4096
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo chmod 600 private/root-ca.key
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo openssl req -x509 -new -nodes -key private/root-ca.key -sha256 -days 365 -out certs/root-ca.crt -config root-ca.cnf
```

**3. Membuat Berkas Konfigurasi SSL Web Server (`web-server.cnf` dengan SAN):**
_(Catatan: Browser modern mengabaikan CN, mereka wajib membaca SAN [Subject Alternative Name])._

```bash
sudo tee web-server.cnf << 'EOF'
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = req_ext

[ dn ]
C  = ID
ST = Jakarta
L  = Jakarta
O  = Milnandy Lab
OU = Web Services
CN = milnandy.local

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = *.milnandy.local
DNS.2 = milnandy.local
IP.1  = 127.0.0.1
EOF
```

**4. Generate Private Key & CSR Server, lalu Tanda Tangani via Root CA:**

```bash
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo openssl genrsa -out private/web-server.key 2048
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo chmod 600 private/web-server.key

ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo openssl req -new -key private/web-server.key -out requests/web-server.csr -config web-server.cnf

# Tanda tangani menggunakan Root CA (masa aktif 1 hari / perbaruan latihan)
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo openssl x509 -req -in requests/web-server.csr \
  -CA certs/root-ca.crt -CAkey private/root-ca.key -CAcreateserial \
  -out certs/web-server.crt -days 1 -sha256 \
  -extfile web-server.cnf -extensions req_ext
```

Mengatur izin keamanan file sertifikat:

```bash
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo chmod 700 private
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo chmod 600 private/web-server.key
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo chmod 644 certs/web-server.crt certs/root-ca.crt
```

---

### Langkah 6: Konfigurasi Virtual Host HTTPS (`blog.milnandy.local`)

- **Penjelasan Singkat:** Mengunduh template web blog dari [Start Bootstrap Resume](https://github.com/StartBootstrap/startbootstrap-resume), mengonfigurasi Apache VirtualHost port 443 dengan sertifikat SSL buatan sendiri.
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /var/www/blog.milnandy && cd /var/www/blog.milnandy
ubuntu@ubuntu-server-1-24:/var/www/blog.milnandy$ sudo wget https://github.com/StartBootstrap/startbootstrap-resume/archive/refs/heads/master.zip
ubuntu@ubuntu-server-1-24:/var/www/blog.milnandy$ sudo unzip master.zip
```

Membuat file konfigurasi VirtualHost HTTPS (`/etc/apache2/sites-available/blog.milnandy-ssl.conf`):

```apache
<VirtualHost *:443>
    ServerName blog.milnandy.local
    DocumentRoot /var/www/blog.milnandy/startbootstrap-resume-master

    SSLEngine on
    SSLCertificateFile    /etc/ssl/private-ca/certs/web-server.crt
    SSLCertificateKeyFile /etc/ssl/private-ca/private/web-server.key

    <Directory /var/www/blog.milnandy/startbootstrap-resume-master>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Mengaktifkan modul SSL dan situs:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo a2enmod ssl
ubuntu@ubuntu-server-1-24:~$ sudo a2ensite blog.milnandy-ssl.conf
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart apache2
```

---

### Langkah 7: Deploy Web Game 2048 pada Port Kustom (8001)

- **Penjelasan Singkat:** Menambahkan situs web ketiga berupa game interaktif 2048 (bersumber dari [GitHub Gabriele Cirulli 2048](https://github.com/gabrielecirulli/2048)) yang berjalan di atas port alternatif `8001` menggunakan Apache.
- **Perintah CLI / Konfigurasi:**

```bash
# 1. Membuat direktori root web untuk game dan mengunduh repositori github
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /var/www/games.milnandy && cd /var/www/games.milnandy
ubuntu@ubuntu-server-1-24:/var/www/games.milnandy$ sudo wget https://github.com/gabrielecirulli/2048/archive/refs/heads/master.zip
ubuntu@ubuntu-server-1-24:/var/www/games.milnandy$ sudo unzip master.zip

# 2. Menambahkan instruksi agar Apache mendengarkan port 8001
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/apache2/ports.conf
```

Tambahkan baris berikut di bagian direktif _Listen_:

```apache
Listen 8001
```

```bash
# 3. Membuat file konfigurasi VirtualHost untuk port 8001
ubuntu@ubuntu-server-1-24:~$ cd /etc/apache2/sites-available
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo cp -p 000-default.conf games.milnandy.conf
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo nano games.milnandy.conf
```

Isi konfigurasi `/etc/apache2/sites-available/games.milnandy.conf`:

```apache
<VirtualHost *:8001>
        ServerName games.milnandy.local
        DocumentRoot /var/www/games.milnandy/2048-master

        ErrorLog ${APACHE_LOG_DIR}/games.error.log
        CustomLog ${APACHE_LOG_DIR}/games.access.log combined
</VirtualHost>
```

Mengaktifkan situs dan memuat ulang Apache:

```bash
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo a2ensite games.milnandy.conf
ubuntu@ubuntu-server-1-24:/etc/apache2/sites-available$ sudo systemctl reload apache2
```

_Uji Coba:_ Akses melalui browser client ke alamat `http://172.16.2.2:8001` atau `http://milnandy.local:8001`.

---

### Langkah 8: Instalasi Offline Nginx pada Server 2 (Slave)

- **Penjelasan Singkat:** Karena Server 2 diisolasi internetnya (_disconnected_), instalasi Nginx dilakukan dengan mengunduh file `.deb` di Server 1, mentransfernya ke folder `/tmp` via SCP menggunakan user non-root `milnandy`, lalu memindahkannya menggunakan `sudo mv` ke `/var/cache/apt/archives/` sebelum diinstal secara lokal di Server 2 menggunakan `dpkg`.
- **Perintah CLI / Konfigurasi:**

Pada **Server 1**:

```bash
# Unduh paket Nginx beserta dependensinya ke direktori cache lokal
ubuntu@ubuntu-server-1-24:~$ sudo apt update
ubuntu@ubuntu-server-1-24:~$ sudo apt-get install --download-only --reinstall nginx -y

# Kirim file paket .deb ke folder /tmp di Server 2 via port SSH custom (2201) menggunakan user milnandy
ubuntu@ubuntu-server-1-24:~$ sudo scp -P 2201 /var/cache/apt/archives/*nginx*.deb milnandy@172.16.2.3:/tmp/
```

Pada **Server 2 (`ubuntu-server-2-24`)**:

```bash
# Pindahkan paket dari /tmp ke direktori cache apt sistem, lalu instal secara offline menggunakan dpkg
ubuntu@ubuntu-server-2-24:~$ cd /var/cache/apt/archives/
ubuntu@ubuntu-server-2-24:/var/cache/apt/archives$ sudo mv /tmp/*nginx*.deb .
ubuntu@ubuntu-server-2-24:/var/cache/apt/archives$ sudo dpkg -i *nginx*.deb

# Verifikasi status layanan Nginx
ubuntu@ubuntu-server-2-24:/var/cache/apt/archives$ sudo systemctl status nginx.service
```

Menambahkan subdomain kustom (`web2.milnandy.local`) pada Nginx Server 2:

```bash
ubuntu@ubuntu-server-2-24:~$ sudo nano /etc/nginx/sites-available/default
```

Sesuaikan direktif `server_name`:

```nginx
server_name web2.milnandy.local;
```

Muat ulang Nginx: `sudo systemctl restart nginx`.

---

## 🔍 Verifikasi & Troubleshooting

### Instalasi Root CA pada Windows Host (Agar Hijau/Trusted)

1. Salin sertifikat _Root CA_ dari server ke direktori home pengguna:
   ```bash
   ubuntu@ubuntu-server-1-24:~$ cp -p /etc/ssl/private-ca/certs/root-ca.crt /home/milnandy
   ubuntu@ubuntu-server-1-24:~$ sudo chown milnandy:milnandy /home/milnandy/root-ca.crt
   ```
2. Tarik file `root-ca.crt` ke komputer host Windows menggunakan SCP dari CMD:
   ```cmd
   C:\Windows\system32> scp -P 2201 -J abrar@192.168.230.128 milnandy@172.16.2.2:root-ca.crt D:\KumpulanFolder\Technology\
   ```
3. Klik dua kali pada file `root-ca.crt` di Windows -> **Install Certificate...** -> Pilih **Local Machine** -> Pilih **Place all certificates in the following store** -> Browse dan pilih **Trusted Root Certification Authorities** -> Next -> Finish.

![Install Certificate](/assets/phase-1-sandbox/image/install-certificate.PNG)

![Install Certificate](/assets/phase-1-sandbox/image/install-certificate1.PNG)

> _Image Certificate Import Wizard saat memasang sertifikat ke Trusted Root Certification Authorities._

### Mengatasi Pemindaian HTTPS Kaspersky (Opsional jika menggunakan Kaspersky)

Jika Anda menggunakan antivirus Kaspersky dan browser menampilkan peringatan sertifikat meskipun Root CA sudah terpasang:

1. Buka aplikasi Kaspersky di Windows.
2. Masuk ke **Settings** -> **Security Settings** -> **Network Settings**.
3. Di bagian **Encrypted connections scanning**, pilih **Do not scan encrypted connections** atau tambahkan pengecualian (_Manage Exclusions_) untuk domain `*.milnandy.local`.
4. Simpan, lalu lakukan _hard refresh_ browser (`Ctrl + F5`).

![Browser Secure Lock](/assets/phase-1-sandbox/image/kaspersky.PNG)

![Browser Secure Lock](/assets/phase-1-sandbox/image/kaspersky1.PNG)

![Browser Secure Lock](/assets/phase-1-sandbox/image/kaspersky2.PNG)

![Browser Secure Lock](/assets/phase-1-sandbox/image/kaspersky3.PNG)

> _Image browser yang menampilkan ikon gembok aman (Secure Connection) saat membuka https://blog.milnandy.local._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Gagal Restart Apache (AH00557: apr_sockaddr_info_get failed):**
    - _Gejala:_ Muncul pesan peringatan _Could not reliably determine the server's fully qualified domain name_ saat Apache dinyalakan.
    - _Solusi:_ Hal ini bersifat informatif (_warning_) dan tidak menghentikan layanan. Namun, Anda bisa membuangnya secara permanen dengan mendefinisikan `ServerName localhost` atau `ServerName 127.0.0.1` di dalam berkas konfigurasi global `/etc/apache2/apache2.conf`.

### Kasus Simulasi: Sertifikat SSL Kedaluwarsa (_Certificate Expired_)

Pada **Langkah 5**, kita sengaja membuat sertifikat web server dengan masa aktif hanya **1 hari** (`-days 1`) untuk keperluan pembelajaran penanganan insiden (_Incident Response_). Ketika masa berlaku tersebut habis, browser client akan menolak koneksi secara otomatis dan memunculkan peringatan keamanan kritis (_NET::ERR_CERT_DATE_INVALID_).

![Expired Certificate Warning](/assets/phase-1-sandbox/image/certificate-expired.PNG)

![Expired Certificate Warning](/assets/phase-1-sandbox/image/certificate-expired1.PNG)

![Expired Certificate Warning](/assets/phase-1-sandbox/image/certificate-expired2.PNG)
*Image baris perintah terminal saat pengecekan masa aktif menggunakan perintah `openssl x509 -enddate -noout -in certs/web-server.crt` yang mengonfirmasi bahwa sertifikat telah melewati batas waktu (*expired*).*

**Cara Memperbarui (_Renew_) Sertifikat yang Kedaluwarsa:**
Masuk kembali ke direktori privat PKI di Server 1, lalu jalankan ulang perintah penandatanganan CSR menggunakan Root CA yang sama:

```bash
# Pindah ke directory yang ada
ubuntu@ubuntu-server-1-24:~$ cd /etc/ssl/private-ca

# Menandatangani ulang CSR dengan memperpanjang masa aktif (misal: -days 365)
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo openssl x509 -req -in requests/web-server.csr \
  -CA certs/root-ca.crt -CAkey private/root-ca.key -CAcreateserial \
  -out certs/web-server.crt -days 365 -sha256 \
  -extfile web-server.cnf -extensions req_ext

# Muat ulang layanan web server Apache
ubuntu@ubuntu-server-1-24:/etc/ssl/private-ca$ sudo systemctl restart apache2
```

**Cara Memperbarui Otomatis (_Renew_) Sertifikat yang Kedaluwarsa:**

Buat file skripnya menggunakan heredoc yang rapi:

```bash
cat << 'EOF' > /etc/ssl/private-ca/renew-certificate.sh
#!/bin/bash

# Pindah ke direktori kerja CA
cd /etc/ssl/private-ca || exit 1

echo "[*] Memulai pembaruan sertifikat web-server (-days 1)..."

# Jalankan perintah penandatanganan OpenSSL
openssl x509 -req -in requests/web-server.csr \
  -CA certs/root-ca.crt -CAkey private/root-ca.key -CAcreateserial \
  -out certs/web-server.crt -days 1 -sha256 \
  -extfile web-server.cnf -extensions req_ext

if [ $? -eq 0 ]; then
    echo "[+] Sertifikat berhasil diperbarui!"
    # Restart Apache untuk memuat sertifikat baru
    systemctl restart apache2
    echo "[+] Apache2 berhasil direstart."
else
    echo "[-] Gagal memperbarui sertifikat!"
    exit 1
fi
EOF
```

Berikan izin eksekusi (chmod +x):

```bash
chmod +x /etc/ssl/private-ca/renew-certificate.sh
```

Cara Menjalankannya (Cukup 1 Baris Perintah Kapan Pun Kamu Mau):

```bash
sudo /etc/ssl/private-ca/renew-certificate.sh
```
