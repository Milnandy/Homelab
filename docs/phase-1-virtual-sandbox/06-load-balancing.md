# Modul 06: High Availability & Load Balancing (HAProxy vs Nginx)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Merancang dan membandingkan sistem distribusi beban kerja (_load balancing_) menggunakan HAProxy (Layer 7 Proxy) dan Nginx (Upstream Proxy), memanfaatkan fitur ACL (_Access Control List_) untuk perutean berbasis sub-domain/path, mengimplementasikan algoritma _Round-Robin_ serta _Least Connections_, melakukan simulasi uji beban server (_stress testing_ menggunakan ApacheBench), serta migrasi lintas teknologi penyeimbang beban.
- **Skenario / Case Study:** Untuk mengantisipasi kegagalan sistem pada web server utama (Server 1), ditambahkan web server cadangan (Server 2) yang berjalan menggunakan teknologi berbeda (Nginx). Sistem pembagi beban (_Load Balancer_) ditempatkan di depan kedua server tersebut untuk mendistribusikan trafik pengunjung secara adil, membelokkan trafik khusus (seperti phpMyAdmin, Game 2048 di port 8001, dan Blog SSL) ke backend yang sesuai, serta menguji performa server saat menghadapi lonjakan akses konkuren secara simultan.
- **Kebutuhan Perangkat / Prasyarat:**
  - Paket HAProxy (`haproxy`)
  - Paket Nginx (`nginx`)
  - Apache2 Server 1 (Port 8000)
  - Nginx Server 2 (Port 80)
  - Utilitas Uji Beban (`apache2-utils` / `ab`)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 06 Topology](/assets/phase-1-sandbox/image/load-balancer-acl-backend-architecture-editable.drawio.png)

> _Diagram arsitektur yang menggambarkan klien mengirim trafik HTTP ke Load Balancer (Port 80), di mana HAProxy/Nginx memilah request berdasarkan ACL domain/path dan mendistribusikannya ke backend server._

### Tabel Pengamatan IP / Layanan Backend

| Device Name        | Peran                     | IP Address | Port Internal                       | Keterangan Layanan                    |
| ------------------ | ------------------------- | ---------- | ----------------------------------- | ------------------------------------- |
| ubuntu-server-1-24 | Load Balancer & Server 1  | 172.16.2.2 | 80 (LB), 8000 (Apache), 8001 (Game) | Pusat Load Balancer Utama & Backend 1 |
| ubuntu-server-2-24 | Server 2 (Backup/Cluster) | 172.16.2.3 | 80 (Nginx)                          | Node Backend Klaster Kedua            |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Penyesuaian Port Apache pada Server 1

- **Penjelasan Singkat:** Mengubah port default Apache di Server 1 dari port 80 ke port 8000 agar port 80 dapat digunakan secara eksklusif oleh aplikasi _Load Balancer_ (HAProxy/Nginx).
- **Perintah CLI / Konfigurasi:**

```bash
# Mengubah port pendengar Apache
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/apache2/ports.conf
```

Ubah direktif _Listen_ menjadi:

```apache
Listen 8000
```

Menyesuaikan VirtualHost dan memberikan izin direktori eksplisit untuk jalur proxy internal:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/apache2/sites-available/www.milnandy.conf
```

```apache
<VirtualHost *:8000>
        ServerName www.milnandy.local

        DocumentRoot /var/www/www.milnandy/startbootstrap-creative-master/dist/

        # Karena permintaan datang dari proxy internal, berikan izin direktori eksplisit
        <Directory /var/www/www.milnandy/startbootstrap-creative-master/dist/>
            Options Indexes FollowSymLinks
            AllowOverride All
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/www.error.log
        CustomLog ${APACHE_LOG_DIR}/www.access.log combined
</VirtualHost>
```

Muat ulang Apache: `sudo systemctl restart apache2`

### Langkah 2: Konfigurasi HAProxy dengan ACL & Multi-Backend

- **Penjelasan Singkat:** Menginstal HAProxy dan mendesain aturan ACL tingkat lanjut untuk memetakan trafik berdasarkan sub-domain (`www`, `games`, `blog`) dan path (`/phpmyadmin`).
- **Perintah CLI / Konfigurasi:**

```bash
# Menginstal paket HAProxy
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install haproxy -y
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/haproxy/haproxy.cfg
```

Isi konfigurasi utama `/etc/haproxy/haproxy.cfg`:

```ini
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000

frontend http-in
    bind *:80
    option forwardfor

    # Dashboard Monitor Statistik HAProxy (Akses via http://172.16.2.2/lbstats)
    stats enable
    stats uri /lbstats
    stats auth admin:adminpassword
    stats refresh 10s

    # Definisi Aturan ACL (Mendeteksi Subdomain & Path)
    acl is_pma          path_beg -i /phpmyadmin
    acl host_www        hdr_beg(host) -i www.milnandy.local milnandy.local
    acl host_games      hdr_beg(host) -i games.milnandy.local
    acl host_blog       hdr_beg(host) -i blog.milnandy.local

    # Aturan Routing Trafik Berdasarkan ACL
    use_backend backend_phpmyadmin if is_pma
    use_backend backend_www      if host_www
    use_backend backend_games    if host_games
    use_backend backend_blog     if host_blog

    default_backend backend_www

# Backend Klaster Load Balance (Round-Robin / Least Connections)
backend backend_www
    balance roundrobin
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request set-header X-Forwarded-Proto http

    server server1 172.16.2.2:8000 check
    server server2 172.16.2.3:80 check

backend backend_phpmyadmin
    server server1 172.16.2.2:8000 check

backend backend_games
    server server1 172.16.2.2:8001 check

backend backend_blog
    mode http
    http-request redirect location https://blog.milnandy.local%[url]
```

Uji validasi sintaks dan jalankan HAProxy:

```bash
ubuntu@ubuntu-server-1-24:~$ haproxy -c -f /etc/haproxy/haproxy.cfg
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart haproxy
```

### Langkah 3: Pengujian Algoritma _Least Connections_ pada HAProxy

- **Penjelasan Singkat:** Mengubah metode distribusi beban dari _Round-Robin_ menjadi _Least Connections_ (`leastconn`) untuk mengarahkan trafik ke server yang memiliki jumlah koneksi aktif paling sedikit.
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/haproxy/haproxy.cfg
```

Ubah parameter `balance` pada backend:

```ini
backend backend_www
    balance leastconn
    ...
```

Terapkan perubahan: `sudo systemctl reload haproxy`

---

### Langkah 4: Migrasi Load Balancer dari HAProxy ke Nginx

- **Penjelasan Singkat:** Menonaktifkan HAProxy dan mengganti perannya menggunakan Nginx sebagai _Reverse Proxy / Upstream Load Balancer_.
- **Perintah CLI / Konfigurasi:**

```bash
# Mematikan dan mendisable HAProxy
ubuntu@ubuntu-server-1-24:~$ sudo systemctl stop haproxy && sudo systemctl disable haproxy

# Menginstal Nginx
ubuntu@ubuntu-server-1-24:~$ sudo apt install nginx -y

# Membuat konfigurasi load balancer Nginx
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/nginx/conf.d/subdomains.conf
```

Isi konfigurasi `/etc/nginx/conf.d/subdomains.conf`:

```nginx
# 1. Definisi Upstream Klaster dengan Algoritma Least Connections
upstream backend_www {

    # Mengaktifkan algoritma Least Connections
    least_conn;

    server 172.16.2.2:8000; # Apache Server 1
    server 172.16.2.3:80;   # Nginx Server 2
}

# 2. Server Block untuk www & phpMyAdmin
server {
    listen 80;
    server_name www.milnandy.local milnandy.local;

    # Halaman statistik internal Nginx
    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        allow 172.16.2.0/24; # Izinkan subnet jaringan lab lokal Anda
        allow all;
        #deny all;

    location /phpmyadmin {
        proxy_pass http://172.16.2.2:8000/phpmyadmin;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        proxy_pass http://backend_www;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}

# 3. Server Block untuk games.milnandy.local (Port 8001)
server {
    listen 80;
    server_name games.milnandy.local;

    location / {
        proxy_pass http://172.16.2.2:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# 4. Server Block untuk blog.milnandy.local (Redirect HTTPS)
server {
    listen 80;
    server_name blog.milnandy.local;

    location / {
        return 301 https://$host$request_uri;
    }
}
```

Uji sintaks dan restart Nginx:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nginx -t
ubuntu@ubuntu-server-1-24:~$ sudo systemctl reload nginx
```

---

Berikut adalah penyempurnaan pada bagian **Verifikasi & Troubleshooting** di **Modul 06**. Saya telah menambahkan panduan pengujian visual dan langkah verifikasi spesifik untuk memastikan bahwa setiap subdomain (_www_, _games_, _blog_), halaman utilitas (`/phpmyadmin`, `/lbstats`), serta pembuktian algoritma **Round-Robin** dan **Least Connections** pada HAProxy maupun Nginx berjalan dengan sempurna tanpa perlu mencantumkan port eksplisit di URL browser.

Anda dapat mengganti seluruh bagian **🔍 Verifikasi & Troubleshooting** pada berkas `docs/phase-1-sandbox/06-load-balancing.md` dengan teks di bawah ini:

---

## 🔍 Verifikasi & Troubleshooting

### 1. Verifikasi Akses Subdomain Tanpa Port & Path Spesifik

Pastikan rute DNS lokal (`hosts`) di komputer host Windows Anda telah mengarahkan domain utama ke IP Load Balancer (`172.16.2.2`). Akses layanan berikut melalui web browser tanpa menyertakan nomor port:

- **Situs Utama (Load Balancing Round-Robin/LeastConn ke Server 1 & Server 2):**
  - URL: `http://www.milnandy.local` atau `http://milnandy.local`
  - _Hasil:_ Halaman portofolio utama tampil dan merespons secara bergantian dari kedua server backend.
    ![Subdomain & Path Verification](/assets/phase-1-sandbox/image/subdomain-haproxy.PNG)
    ![Subdomain & Path Verification](/assets/phase-1-sandbox/image/subdomain-haproxy1.PNG)
    > _Tampilan web browser dengan penerapan reverse proxy saat membuka www.milnandy.local tanpa port serta penerapan loadbalancer antara server1 dan server2._
- **Aplikasi Game 2048 (Dialihkan ke Backend Port 8001):**
  - URL: `http://games.milnandy.local`
  - _Hasil:_ Permainan 2048 terbuka secara langsung di port standar 80 publik, namun diproses secara transparan oleh _reverse proxy_ di balik layar menuju port `8001`.
    ![Subdomain & Path Verification](/assets/phase-1-sandbox/image/subdomain-haproxy2.PNG)
    > _Tampilan web browser dengan penerapan reverse proxy saat membuka games.milnandy.local tanpa port._
- **Blog / Panel SSL (Dialihkan ke HTTPS):**
  - URL: `http://blog.milnandy.local`
  - _Hasil:_ Browser secara otomatis melakukan _redirect_ 301 menuju protokol aman `https://blog.milnandy.local`.
    ![Subdomain & Path Verification](/assets/phase-1-sandbox/image/subdomain-haproxy3.PNG)
    > _Tampilan web browser dengan penerapan reverse proxy saat membuka blog.milnandy.local tanpa port._
- **Manajemen Database (phpMyAdmin):**
  - URL: `http://www.milnandy.local/phpmyadmin`
  - _Hasil:_ Halaman login phpMyAdmin terbuka dengan mengarahkan trafik secara eksklusif ke Server 1 (`172.16.2.2:8000`).
    ![Subdomain & Path Verification](/assets/phase-1-sandbox/image/subdomain-haproxy4.PNG)
    > _Tampilan web browser dengan penerapan reverse proxy saat membuka /phpmyadmin tanpa port._
- **Dasbor Statistik HAProxy (Khusus saat menggunakan HAProxy):**
  - URL: `http://www.milnandy.local/lbstats` atau `172.16.2.2/lbstats`
  - _Hasil:_ Menampilkan panel pemantauan kesehatan klaster (_health check_) real-time setelah memasukkan autentikasi (`admin:adminpassword`).
    ![Subdomain & Path Verification](/assets/phase-1-sandbox/image/lbstats.PNG)
    > _Image browser saat membuka monitoring bawaan dari haproxy._

---

### 2. Verifikasi Algoritma Load Balancing (Round-Robin & Least Connections)

Untuk membuktikan bahwa mekanisme distribusi beban bekerja dengan baik, kita menggunakan utilitas uji beban **ApacheBench (`ab`)** untuk menembak server secara konkuren.

#### **A. Verifikasi Algoritma Round-Robin (Bergantian Rata)**

Algoritma Round-Robin membagi request secara bergiliran urut dari Server 1 ke Server 2.

1. Pastikan backend dikonfigurasi dengan `balance roundrobin` (di HAProxy dan Nginx) atau tanpa argumen tambahan di _upstream_ Nginx.
2. Jalankan simulasi pengujian beban dengan koneksi konkuren yang padat agar terlihat pengaturannya:
   ```bash
   ubuntu@ubuntu-server-1-24:~$ while true; do ab -n 200 -c 10 http://www.milnandy.local/; sleep 1; done
   ```
3. Cek log akses pada kedua server untuk melihat pencatatan waktu dan giliran IP yang merespons dan lbstats.

![Round-Robin Verification](/assets/phase-1-sandbox/image/lbstats4.PNG)
![Round-Robin Verification](/assets/phase-1-sandbox/image/lbstats5.PNG)
![Round-Robin Verification](/assets/phase-1-sandbox/image/lbstats6.PNG)

> _Image menunjukkan hasil access log haproxy di Server 1 dan Server 2 menerima request secara bergantian (selang-seling) dan monitoring haproxy (bisa dilihat dibagian "backend_www" header "Session" di kolom "Cur" dan "Total")._

![Round-Robin Verification](/assets/phase-1-sandbox/image/nginx-status4.PNG)
![Round-Robin Verification](/assets/phase-1-sandbox/image/nginx-status5.PNG)
![Round-Robin Verification](/assets/phase-1-sandbox/image/nginx-status6.PNG)

> _Image menunjukkan hasil access log nginx di Server 1 dan Server 2 menerima request secara leastconn (koneksi paling sedikit) dan monitoring nginx (bisa dilihat dibagian "server accepts handled requests") tidak sedetail pada monitoring haproxy._

#### **B. Verifikasi Algoritma Least Connections (`leastconn` / `least_conn`)**

Algoritma ini mengarahkan trafik baru ke server yang saat itu sedang memegang jumlah koneksi aktif paling sedikit.

1. Aktifkan konfigurasi `leastconn` (HAProxy) atau `least_conn;` (Nginx di dalam blok _upstream_).
2. Jalankan simulasi pengujian beban dengan koneksi konkuren yang padat agar terlihat pengaturannya:
   ```bash
   ubuntu@ubuntu-server-1-24:~$ while true; do ab -n 200 -c 10 http://www.milnandy.local/; sleep 1; done
   ```
3. Amati perubahan metrik jumlah sesi aktif pada dasbor `/lbstats` (jika menggunakan HAProxy) atau amati lonjakan trafik pada _access.log_ dan `/nginx_status` (jika menggunakan Nginx).

![Least Connections Verification](/assets/phase-1-sandbox/image/lbstats1.PNG)
![Least Connections Verification](/assets/phase-1-sandbox/image/lbstats2.PNG)
![Least Connections Verification](/assets/phase-1-sandbox/image/lbstats3.PNG)

> _Image menunjukkan hasil access log haproxy di Server 1 dan Server 2 menerima request secara leastconn (koneksi paling sedikit) dan monitoring haproxy (bisa dilihat dibagian "backend_www" header "Session" di kolom "Cur" dan "Total")._

![Least Connections Verification](/assets/phase-1-sandbox/image/nginx-status1.PNG)
![Least Connections Verification](/assets/phase-1-sandbox/image/nginx-status2.PNG)
![Least Connections Verification](/assets/phase-1-sandbox/image/nginx-status3.PNG)

> _Image menunjukkan hasil access log nginx di Server 1 dan Server 2 menerima request secara leastconn (koneksi paling sedikit) dan monitoring nginx (bisa dilihat dibagian "server accepts handled requests") tidak sedetail pada monitoring haproxy._

---

### 🛠️ Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **HAProxy / Nginx Gagal Start (Port 80 Terpakai):**
    - _Gejala:_ Layanan load balancer gagal melakukan _bind_ ke port 80.
    - _Penyebab:_ Proses web server lama (seperti Apache default sebelum dipindah ke port 8000) masih menduduki port 80.
    - _Solusi:_ Periksa proses aktif menggunakan `sudo ss -tulpn | grep :80`, lalu matikan proses yang berbenturan atau sesuaikan kembali file `ports.conf`.
2.  **Asset CSS/JS Berantakan di Web Portofolio:**
    - _Gejala:_ Situs web merespons, namun tata letak CSS dan gambar gagal dimuat (_404 Not Found_ pada aset).
    - _Penyebab:_ Path relatif pada direktori web tidak diteruskan dengan benar oleh _reverse proxy_ ke port backend Apache (8000).
    - _Solusi:_ Tambahkan direktif header tambahan pada konfigurasi proxy (`proxy_set_header Host $host;` dan `proxy_set_header X-Forwarded-For ...`) serta pastikan blok `<Directory>` di dalam konfigurasi Apache mengizinkan akses penuh (`Require all granted`).
3.  **Halaman `/phpmyadmin` Mengembalikan Error 404 pada Nginx:**
    - _Gejala:_ Saat mengakses `http://www.milnandy.local/phpmyadmin`, browser menampilkan halaman _Not Found_.
    - _Penyebab:_ Blok `location /phpmyadmin` pada konfigurasi Nginx belum dipetakan secara benar ke _backend_ Apache port 8000 tempat phpMyAdmin di-host.
    - _Solusi:_ Pastikan direktif `proxy_pass http://172.16.2.2:8000/phpmyadmin;` terpasang dengan tepat di dalam blok _server block_ Nginx.
