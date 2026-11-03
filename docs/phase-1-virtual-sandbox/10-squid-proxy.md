# Modul 07: Proxy Service (Squid Forward & Reverse Proxy)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Membangun layanan penengah (_Proxy Server_) menggunakan Squid, mengonfigurasi penyaringan konten (_Content Filtering_) berbasis domain blacklist, serta mendesain _Reverse Proxy/Web Accelerator_.
- **Skenario / Case Study:** Manajemen menginginkan adanya pengontrolan aktivitas berselancar karyawan di internet dengan memblokir situs-situs non-produktif (seperti portal berita detik.com) melalui kebijakan kontrol akses internal. Di sisi lain, server proxy harus bisa diubah perannya untuk bertindak sebagai akselerator trafik web luar (_reverse proxy_) yang mengarahkan trafik dari Server 1 langsung ke Server 2 secara mulus.
- **Kebutuhan Perangkat / Prasyarat:**
  - Squid Proxy Paket (`squid`)
  - Web browser pada Windows Host

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 07 Topology](../assets/07-squid-proxy-topology.png)

> _[Placeholder Gambar]: Skema aliran trafik internet client diarahkan ke Port 3128 Squid Proxy, yang melakukan evaluasi daftar blacklist sebelum melemparkan request ke internet._

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Instalasi & Pembuatan Daftar Hitam Domain Blacklist

- **Penjelasan Singkat:** Menginstal paket Squid dan mendefinisikan daftar nama domain yang akan diblokir aksesnya.
- **Perintah CLI / Konfigurasi:**

```bash
# Menginstal paket Squid Proxy
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install squid -y

# Membuat berkas teks penampung blacklist domain
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/squid/list-domain
```

Masukkan domain yang ingin diblokir di dalam `/etc/squid/list-domain`:

```text
detik.com
www.detik.com
```

### Langkah 2: Konfigurasi Aturan Kontrol Akses (ACL) Squid

- **Penjelasan Singkat:** Mendeklarasikan aturan pemblokiran berkas list-domain dan membuka hak akses jaringan lokal di berkas konfigurasi Squid.
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/squid/squid.conf
```

Sisipkan konfigurasi ACL di bagian atas aturan akses:

```ini
# Mendefinisikan segmen asal ip client (All IP)
acl idnet src 0.0.0.0/0
# Membaca berkas daftar blokir domain
acl blocked dstdomain "/etc/squid/list-domain"

# Aturan Akses (Utamakan Deny sebelum Allow)
http_access deny blocked
http_access allow idnet
```

Terapkan perubahan:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart squid
```

### Langkah 3: Konfigurasi Reverse Proxy (Web Accelerator)

- **Penjelasan Singkat:** Mengonfigurasi Squid untuk bertindak sebagai penerima request port 80 dan mengarahkannya kembali ke Server 2 (172.16.1.3).
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/squid/squid.conf
```

Ganti baris `http_port 3128` menjadi konfigurasi akselerator berikut:

```ini
http_port 80 accel defaultsite=172.16.1.3 vhost
cache_peer 172.16.1.3 parent 80 0 no-query originserver
```

Restart layanan:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart squid
```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Filter Proxy & Monitoring Log

```bash
# Memantau aktivitas akses client secara real-time
ubuntu@ubuntu-server-1-24:~$ tail -f /var/log/squid/access.log
```

![Proxy Verification](../assets/07-squid-proxy-verification.png)

> _[Tugas Dokumentasi]: Masukkan tangkapan layar web browser Windows Client setelah proxy diaktifkan (ke IP 172.16.1.2 Port 3128), menampilkan pesan error Squid "Access Denied" saat membuka detik.com._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Squid Gagal Restart (Sintaks ACL Salah):**
    - _Gejala:_ Layanan Squid tidak aktif setelah perubahan berkas konfigurasi.
    - _Penyebab:_ Kesalahan penulisan tipe direktif ACL atau jalur berkas list-domain tidak dapat ditemukan oleh Squid.
    - _Solusi:_ Periksa kecocokan nama berkas list-domain pada `/etc/squid/list-domain` dan pastikan hak pembacaan berkas tersebut terbuka bagi sistem.
