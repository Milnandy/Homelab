# Modul 09: Mail Server Komprehensif (Postfix, Dovecot, & Roundcube)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Membangun tumpukan surat elektronik lengkap (_Mail Server Stack_) berbasis Postfix (MTA), Dovecot (IMAP), dan Roundcube (Webmail), serta mengimplementasikan penyimpanan berformat _Maildir_ [11, 15].
- **Skenario / Case Study:** Organisasi membutuhkan layanan komunikasi surat elektronik mandiri untuk bertukar pesan secara internal (`mail.milnandy.local`) demi menjaga kerahasiaan data perusahaan. Layanan ini harus mendukung autentikasi yang aman, pembuatan kotak surat otomatis bagi setiap user baru, serta memiliki antarmuka webmail modern yang ramah pengguna.
- **Kebutuhan Perangkat / Prasyarat:**
  - Postfix (`postfix`)
  - Dovecot IMAP (`dovecot-imapd`)
  - MariaDB Server (`mariadb-server`)
  - Roundcube Webmail (`roundcube`, `roundcube-mysql`)

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 09 Topology](../assets/09-mail-server-topology.png)

> _[Placeholder Gambar]: Skema interkoneksi alur surat elektronik: Client Webmail (Port 8080/Roundcube) -> Postfix (Port 25/SMTP) -> Maildir local storage -> Dovecot (Port 143/IMAP) [11, 15]._

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Instalasi Paket Mail Server & Database

- **Penjelasan Singkat:** Mengunduh tumpukan paket pembangun mail server dan mengonfigurasi database untuk aplikasi webmail.
- **Perintah CLI / Konfigurasi:**

```bash
# Menginstal seluruh paket mail server
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install postfix dovecot-imapd mariadb-server roundcube roundcube-mysql -y
```

_Pilihan saat instalasi prompt:_

- _General type of mail configuration:_ Internet Site [15].
- _System mail name:_ `mail.milnandy.local`.

### Langkah 2: Konfigurasi MTA Postfix

- **Penjelasan Singkat:** Menentukan identitas domain email, hak pengiriman lokal, dan mengaktifkan direktori Maildir untuk penyimpanan surat [11, 15].
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/postfix/main.cf
```

Sesuaikan konfigurasi `/etc/postfix/main.cf` menjadi seperti berikut:

```ini
smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no
append_dot_mydomain = no
readme_directory = no
compatibility_level = 3.6

myhostname = mail.milnandy.local
mydomain = milnandy.local
myorigin = /etc/mailname
mydestination = $myhostname, $mydomain, localhost.$mydomain, localhost
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all

# Parameter Utama untuk Aktivasi Format Maildir
home_mailbox = Maildir/
```

Restart Postfix: `sudo systemctl restart postfix`.

### Langkah 3: Konfigurasi MDA Dovecot (IMAP)

- **Penjelasan Singkat:** Menyelaraskan lokasi folder kotak surat Maildir pada Dovecot dan mendefinisikan protokol akses [11].
- **Perintah CLI / Konfigurasi:**

```bash
# 1. Menentukan lokasi Maildir
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/dovecot/conf.d/10-mail.conf
```

Ubah parameter `mail_location` menjadi [11]:

```ini
mail_location = maildir:~/Maildir
```

```bash
# 2. Mengaktifkan protokol IMAP
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/dovecot/dovecot.conf
```

Ubah parameter `protocols`:

```ini
protocols = imap
```

```bash
# 3. Mengonfigurasi skema autentikasi user
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/dovecot/conf.d/10-auth.conf
```

Sesuaikan format username:

```ini
auth_username_format = %n
```

Restart Dovecot: `sudo systemctl restart dovecot`.

### Langkah 4: Pembuatan Template Kotak Surat Otomatis (System Skeleton)

- **Penjelasan Singkat:** Mengonfigurasi direktori `/etc/skel` (Skeleton) agar setiap akun user baru otomatis memiliki struktur folder kotak surat Maildir yang valid saat pertama kali dibuat [11].
- **Perintah CLI / Konfigurasi:**

```bash
# Membuat struktur Maildir pada template sistem (skeleton)
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /etc/skel/Maildir/{cur,new,tmp}
ubuntu@ubuntu-server-1-24:~$ sudo chmod -R 700 /etc/skel/Maildir
```

### Langkah 5: Konfigurasi Webmail Roundcube

- **Penjelasan Singkat:** Mengaktifkan alias direktori webmail di Apache dan menghubungkan Roundcube ke SMTP localhost.
- **Perintah CLI / Konfigurasi:**

```bash
# Mengaktifkan alias /roundcube di Apache
ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/apache2/conf-enabled/roundcube.conf
```

Pastikan baris alias berikut telah diaktifkan (_uncomment_):

```apache
Alias /roundcube /var/lib/roundcube/public_html
```

Konfigurasi parameter koneksi SMTP pada `/etc/roundcube/config.inc.php`:

```php
$config['smtp_host'] = 'localhost:25';
$config['smtp_user'] = '';
$config['smtp_pass'] = '';
```

Restart Apache: `sudo systemctl restart apache2`.

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Pengiriman Surat Elektronik (Mail Flow)

```bash
# Membuat dua akun user baru untuk uji kirim email
ubuntu@ubuntu-server-1-24:~$ sudo adduser user1
ubuntu@ubuntu-server-1-24:~$ sudo adduser user2
```

![Roundcube Mail Verification](../assets/09-mail-server-verification.png)

> _[Tugas Dokumentasi]: Masukkan tangkapan layar dashboard Roundcube Webmail setelah sukses login dan sukses bertukar pesan email antara user1@milnandy.local ke user2@milnandy.local._

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Email Gagal Dikirim (SMTP Error / Log: Permission Denied):**
    - _Gejala:_ Dashboard Roundcube memicu pesan kegagalan pengiriman sesaat setelah menekan tombol "Send".
    - _Penyebab:_ Direktori Maildir tidak dapat ditulis oleh sistem karena kesalahan hak kepemilikan atau parameter Postfix `home_mailbox` belum sinkron [11].
    - _Solusi:_ Pastikan menambahkan `home_mailbox = Maildir/` di `/etc/postfix/main.cf` [11]. Perbaiki hak akses folder Maildir di home direktori pengguna secara manual menggunakan perintah:
      ```bash
      ubuntu@ubuntu-server-1-24:~$ sudo chown -R user1:user1 /home/user1/Maildir
      ubuntu@ubuntu-server-1-24:~$ sudo chmod -R 700 /home/user1/Maildir
      ```
