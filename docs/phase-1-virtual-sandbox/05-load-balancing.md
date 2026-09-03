# Modul 05: High Availability & Load Balancing (HAProxy vs Nginx)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Merancang dan membandingkan sistem distribusi beban kerja (_load balancing_) menggunakan HAProxy (Layer 7 Proxy) dan Nginx (Upstream Proxy), serta memanfaatkan fitur ACL (Access Control List).
- **Skenario / Case Study:** Untuk mengantisipasi kegagalan sistem pada web server utama (Server 1), ditambahkan web server cadangan (Server 2) yang berjalan menggunakan teknologi yang berbeda (Nginx). Sistem pembagi beban (_Load Balancer_) ditempatkan di depan kedua server tersebut untuk membagi rata request pengunjung dan membelokkan trafik khusus (seperti database administrator / phpMyAdmin) hanya ke server yang memiliki paket tersebut.
- **Kebutuhan Perangkat / Prasyarat:**
  - HAProxy Paket (`haproxy`)
  - Nginx Paket (`nginx`)
  - Server 1 (Apache2 - Port 8080)
  - Server 2 (Nginx - Port 80)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 05 Topology](../assets/05-load-balancing-topology.png)

> _[Placeholder Gambar]: Diagram yang menggambarkan klien mengirim trafik HTTP ke Load Balancer (Port 80) dan diteruskan ke backend server menggunakan algoritma Round-Robin._

### Tabel Pengamatan IP / Interface

| Device Name        | Peran                 | IP Address | Port Internal                      |
| ------------------ | --------------------- | ---------- | ---------------------------------- |
| ubuntu-server-1-24 | Load Balancer / Web 1 | 172.16.1.2 | HAProxy/Nginx (80), Apache2 (8080) |
| ubuntu-server-2-24 | Web 2                 | 172.16.1.3 | Nginx (80)                         |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Instalasi & Konfigurasi HAProxy (Layer 7 Routing)

- **Penjelasan Singkat:** Menginstal HAProxy dan mendesain aturan ACL untuk memilah trafik berdasarkan domain atau path URI.
- **Perintah CLI / Konfigurasi:**

```bash
# Menginstal paket HAProxy
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install haproxy -y

# Mengedit berkas konfigurasi utama HAProxy
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/haproxy/haproxy.cfg
```

Isi blok konfigurasi `/etc/haproxy/haproxy.cfg`:

```ini
frontend http-in
    bind *:80
    option forwardfor

    # Dashboard Monitor Statistik HAProxy
    stats enable
    stats uri /lbstats
    stats auth admin:adminpassword
    stats refresh 10s

    # Definisi Aturan ACL
    acl is_pma   path_beg -i /phpmyadmin
    acl is_games hdr_beg(host) -i games.milnandy.local

    # Aturan Routing Trafik
    use_backend apache_server1 if is_pma || is_games
    default_backend backend_servers

backend apache_server1
    server server1 172.16.1.2:8080 check

backend backend_servers
    balance roundrobin
    server server1 172.16.1.2:8080 check
    server server2 172.16.1.3:80 check
```

Restart layanan:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart haproxy.service
```

### Langkah 2: Konfigurasi Nginx sebagai Load Balancer Alternatif

- **Penjelasan Singkat:** Menghentikan HAProxy dan mengganti perannya menggunakan Nginx dengan skema konfigurasi _upstream_.
- **Perintah CLI / Konfigurasi:**

```bash
# Menghentikan dan mendisable HAProxy agar tidak berbenturan port
ubuntu@ubuntu-server-1-24:~$ sudo systemctl stop haproxy && sudo systemctl disable haproxy

# Membuat file konfigurasi load balancer di Nginx
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/nginx/conf.d/loadbalancer.conf
```

Isi berkas `/etc/nginx/conf.d/loadbalancer.conf`:

```nginx
upstream backend_servers {
    server 172.16.1.2:8080; # Apache2
    server 172.16.1.3:80;   # Nginx
}

server {
    listen 80;
    server_name www.milnandy.local milnandy.local;

    # Kunci akses phpMyAdmin ke Server 1 saja
    location /phpmyadmin {
        proxy_pass http://172.16.1.2:8080/phpmyadmin;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Distribusi load balance utama
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Uji dan restart Nginx:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nginx -t
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart nginx
```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Mekanisme Pembagian Beban (Round-Robin)

```bash
# Lakukan request berulang kali menggunakan curl dari komputer host
C:\Windows\system32> curl -I http://www.milnandy.local
C:\Windows\system32> curl -I http://www.milnandy.local

# Verifikasi respons header harus bergantian antara Apache2 (Server 1) dan Nginx (Server 2)
```

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **HAProxy Gagal Start (Port 80 Already in Use):**
    - _Gejala:_ Error saat melakukan restart layanan HAProxy.
    - _Penyebab:_ Terdapat aplikasi web server lain (seperti Apache2 default atau Nginx) yang sedang berjalan di port 80 pada server yang sama.
    - _Solusi:_ Periksa proses yang menggunakan port tersebut menggunakan `sudo ss -tulpn | grep :80`. Pastikan menghentikan atau mengubah konfigurasi port aplikasi yang berbenturan tersebut sebelum menyalakan kembali HAProxy.
