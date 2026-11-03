# Modul 05: Database Server (MariaDB Master-Slave & phpMyAdmin)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Melakukan instalasi dan pengamanan basis data MariaDB, melakukan operasi dasar SQL, mengonfigurasi antarmuka manajemen berbasis web (_phpMyAdmin_), serta membangun arsitektur replikasi data tingkat lanjut (_Master-Slave Database Replication_) menggunakan utilitas `mariadb-backup` tanpa _downtime_ migrasi manual.
- **Skenario / Case Study:** Perusahaan memerlukan server basis data relasional berkecepatan tinggi yang mendukung redundansi data (_High Availability_). Ketika Server 1 (Master) memproses seluruh transaksi tulis (_write/insert_), Server 2 (Slave) secara otomatis menyalin dan menyinkronkan data secara _real-time_ lewat jalur biner log (_binary log_). Selain itu, administrator memasang _phpMyAdmin_ menggunakan _symbolic link_ untuk memudahkan manajemen pemeliharaan sistem modular Apache.
- **Kebutuhan Perangkat / Prasyarat:**
  - 2 Unit VM Ubuntu Server 24.04 (Master: 172.16.2.2, Slave: 172.16.2.3)
  - Paket `mariadb-server`, `mariadb-backup`, `phpmyadmin`

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 05 Topology](../assets/phase-1-sandbox/05-database-server-topology.png)

> _[Placeholder Gambar]: Skema replikasi biner log satu arah dari MariaDB Master (172.16.2.2) menuju MariaDB Slave (172.16.2.3)._

### Tabel Pengamatan Layanan Database

| Peran Server    | Hostname           | IP Address | Port | Keterangan                                   |
| --------------- | ------------------ | ---------- | ---- | -------------------------------------------- |
| Database Master | ubuntu-server-1-24 | 172.16.2.2 | 3306 | Menangani operasi Write & Replikasi Binlog   |
| Database Slave  | ubuntu-server-2-24 | 172.16.2.3 | 3306 | Menerima replikasi data otomatis dari Master |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Instalasi & Pengamanan Dasar MariaDB (Database Hardening)

- **Penjelasan Singkat:** Menginstal MariaDB Server, menjalankan skrip pengamanan awal untuk menutup celah kerentanan default.
- **Perintah CLI / Konfigurasi:**

```bash
# Menginstal MariaDB Server
ubuntu@ubuntu-server-1-24:~$ sudo apt update && sudo apt install mariadb-server -y
ubuntu@ubuntu-server-1-24:~$ sudo systemctl enable --now mariadb

# Menjalankan skrip pengamanan interaktif
ubuntu@ubuntu-server-1-24:~$ sudo mysql_secure_installation
```

_Respons Panduan Interaktif:_

- _Enter current password for root:_ Tekan **Enter**.
- _Switch to unix_socket authentication [Y/n]:_ Ketik **N**.
- _Change the root password? [Y/n]:_ Ketik **N**.
- _Remove anonymous users? [Y/n]:_ Ketik **Y**.
- _Disallow root login remotely? [Y/n]:_ Ketik **Y**.
- _Remove test database and access to it? [Y/n]:_ Ketik **Y**.
- _Reload privilege tables now? [Y/n]:_ Ketik **Y**.

### Langkah 2: Operasi Dasar SQL & Manajemen Pengguna

- **Penjelasan Singkat:** Masuk ke konsol SQL untuk menguji pembuatan tabel, manipulasi data (_CRUD_), serta membuat pengguna administrasi baru (`sa`).
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo mysql -u root -p
```

Di dalam konsol SQL:

```sql
-- Operasi Tabel & Data Dasar
CREATE DATABASE test_database;
CREATE TABLE test_database.test_table (
    No int,
    Nama varchar(50),
    Alamat varchar(50),
    primary key (No)
);

INSERT INTO test_database.test_table(No, Nama, Alamat) VALUES(1, "user1", "Jakarta"), (2, "user2", "Jakarta");
INSERT INTO test_database.test_table(No, Nama, Alamat) VALUES(3, "user3", "Bali"), (4, "user4", "Bali");

-- Pembuatan User Administratif Khusus
CREATE USER 'sa' IDENTIFIED BY 'db12345678';
GRANT ALL PRIVILEGES ON *.* TO 'sa';
FLUSH PRIVILEGES;
EXIT;
```

### Langkah 3: Instalasi phpMyAdmin & Manajemen Symbolic Link Apache

- **Penjelasan Singkat:** Menginstal phpMyAdmin beserta modul PHP Apache, lalu menghubungkannya menggunakan _symbolic link_ agar pemeliharaan paket (_apt upgrade_) tetap terjaga secara modular.
- **Perintah CLI / Konfigurasi:**

```bash
ubuntu@ubuntu-server-1-24:~$ sudo apt install phpmyadmin libapache2-mod-php8.3 -y
```

**Mengapa Menggunakan _Symbolic Link_ (`ln -s`) Ketimbang Menyalin Berkas (`cp`)?**

- **Pembaruan Otomatis (Maintenance):** Jika ada pembaruan keamanan paket phpMyAdmin via `apt upgrade`, berkas pusat akan diperbarui otomatis tanpa tertinggal.
- **Kebersihan Sistem Modular:** Menghindari duplikasi berkas sampah yang berserakan. Cukup aktifkan atau nonaktifkan modul lewat direktori `conf-available` dan `conf-enabled`.

Membuat _symbolic link_ konfigurasi Apache:

```bash
ubuntu@ubuntu-server-1-24:~$ sudo ln -s /etc/phpmyadmin/apache.conf /etc/apache2/conf-available/phpmyadmin.conf
ubuntu@ubuntu-server-1-24:~$ sudo a2enconf phpmyadmin
ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart apache2
```

_Uji Coba:_ Akses melalui browser host ke alamat `http://172.16.2.2/phpmyadmin` dan login menggunakan user `sa`.

---

### Langkah 4: Konfigurasi Replikasi Master-Slave (Backup & Sinkronisasi Biner)

#### **A. Konfigurasi pada Server 1 (Master - 172.16.2.2)**

1. Sunting berkas konfigurasi `50-server.cnf`:
   ```bash
   ubuntu@ubuntu-server-1-24:~$ sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
   ```
   Tambahkan baris berikut di dalam blok `[mysqld]`:
   ```ini
   server-id       = 1
   log-basename    = master1
   log_bin         = /var/log/mysql/mariadb-bin.log
   binlog_do_db    = test_database
   bind-address    = 0.0.0.0
   ```
2. Buat direktori log dan restart MariaDB:
   ```bash
   ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /var/log/mysql && sudo chown -R mysql:mysql /var/log/mysql
   ubuntu@ubuntu-server-1-24:~$ sudo systemctl restart mariadb
   ```
3. Buat user khusus replikasi dan ambil status posisi log biner (_binlog_):
   ```bash
   ubuntu@ubuntu-server-1-24:~$ sudo mysql -u root -p
   ```
   ```sql
   CREATE USER 'replicator'@'172.16.2.%' IDENTIFIED BY 'db12345678';
   GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'172.16.2.%';
   FLUSH PRIVILEGES;
   SHOW MASTER STATUS;
   ```
   _(Catat posisi File dan Position, contoh: `mariadb-bin.000001` di posisi `330`)_.
4. Lakukan pencadangan fisik (_physical backup_) menggunakan `mariabackup` dan kirim arsipnya ke Server 2 via SCP:
   ```bash
   ubuntu@ubuntu-server-1-24:~$ sudo apt install mariadb-backup -y
   ubuntu@ubuntu-server-1-24:~$ sudo mariabackup --backup --target-dir /backup-db -u root
   ubuntu@ubuntu-server-1-24:~$ cd / && sudo tar cvf db-backup.tar backup-db
   ubuntu@ubuntu-server-1-24:~$ sudo scp -P 2201 db-backup.tar root@172.16.2.3:/
   ```

#### **B. Konfigurasi pada Server 2 (Slave - 172.16.2.3)**

1. Instal paket MariaDB dan MariaDB-Backup:
   ```bash
   ubuntu@ubuntu-server-2-24:~$ sudo apt update && sudo apt install mariadb-server mariadb-backup -y
   ```
2. Sesuaikan konfigurasi `50-server.cnf`:
   ```bash
   ubuntu@ubuntu-server-2-24:~$ sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
   ```
   Ubah `server-id` menjadi 2:
   ```ini
   server-id       = 2
   relay-log       = /var/log/mysql/mysql-relay-bin.log
   bind-address    = 0.0.0.0
   ```
3. Hentikan layanan MariaDB, bersihkan direktori data bawaan, ekstrak, siapkan (_prepare_), dan pulihkan (_restore_) data dari backup Master:
   ```bash
   ubuntu@ubuntu-server-2-24:~$ sudo systemctl stop mariadb
   ubuntu@ubuntu-server-2-24:~$ sudo rm -rf /var/lib/mysql/*
   ubuntu@ubuntu-server-2-24:~$ cd / && sudo tar xvf db-backup.tar
   ubuntu@ubuntu-server-2-24:~$ sudo mariabackup --prepare --target-dir /backup-db
   ubuntu@ubuntu-server-2-24:~$ sudo mariabackup --copy-back --target-dir /backup-db
   ubuntu@ubuntu-server-2-24:~$ sudo chown -R mysql:mysql /var/lib/mysql /var/log/mysql
   ubuntu@ubuntu-server-2-24:~$ sudo systemctl restart mariadb
   ```
4. Hubungkan Slave ke Master melalui konsol SQL:

   ```bash
   ubuntu@ubuntu-server-2-24:~$ sudo mysql -u root -p
   ```

   ```sql
   STOP SLAVE;

   CHANGE MASTER TO
     MASTER_HOST='172.16.2.2',
     MASTER_USER='replicator',
     MASTER_PASSWORD='db12345678',
     MASTER_LOG_FILE='mariadb-bin.000001',
     MASTER_LOG_POS=330;

   START SLAVE;
   ```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Status Replikasi & Sinkronisasi Data

```bash
# Periksa status replikasi secara mendalam di Server 2 (Slave)
ubuntu@ubuntu-server-2-24:~$ sudo mysql -u root -p -e "SHOW SLAVE STATUS\G"
```

_Pastikan parameter berikut bernilai **Yes**:_

- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`

**Uji Coba Data Masuk (Master ke Slave):**

1. Di **Server 1 (Master)**, masukkan data baru:
   ```sql
   INSERT INTO test_database.test_table(No, Nama, Alamat) VALUES(7, 'user7', 'Tangsel'), (8, 'user8', 'Tangsel');
   ```
2. Di **Server 2 (Slave)**, verifikasi apakah data tersebut otomatis tersalin:
   ```sql
   SELECT * FROM test_database.test_table;
   ```

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Status Replikasi `Slave_IO_Running: Connecting` atau `No`:**
    - _Gejala:_ Proses sinkronisasi terputus dan gagal terhubung ke server Master.
    - _Penyebab:_ Kesalahan alamat IP Master, port 3306 terblokir oleh firewall lokal, atau kredensial `replicator` tidak memiliki hak akses dari subnet _LAN Segment_.
    - _Solusi:_ Uji koneksi dari Slave ke Master menggunakan `nc -zv 172.16.2.2 3306`. Pastikan direktif _bind-address_ di Master diatur ke `0.0.0.0` dan _grant privileges_ menggunakan wildcard subnet yang sesuai (`'172.16.2.%'`).
