# Modul 00: Provisioning & Persiapan Virtual Machine (VMware Workstation Pro)

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Melakukan pembuatan (_provisioning_) Virtual Machine dari awal (_scratch_), memahami instalasi OS Linux dengan fitur penyimpanan dinamis LVM [19], serta menerapkan metode efisiensi penyimpanan menggunakan teknik _Linked Clone_ berbasis _Snapshot_ [17, 18].
- **Skenario / Case Study:** Administrator sistem harus menyiapkan dua buah server Linux dengan sistem operasi yang identik. Melakukan proses instalasi satu per satu dari awal akan memakan banyak waktu (~30-45 menit per server). Skenario ini diatasi dengan melakukan instalasi bersih pada Server 1, membuat _Snapshot_ dasar (_Base_), lalu melakukan kloning bertipe _Linked Clone_ untuk Server 2 [17, 18]. Untuk menghemat RAM host (8 GB), kapasitas RAM VM dinaikkan hanya selama instalasi berat dan diturunkan kembali saat berjalan (_runtime_).
- **Kebutuhan Perangkat / Prasyarat:**
  - VMware Workstation Pro (Personal Use)
  - ISO Ubuntu Desktop 24.04 LTS
  - Ruang penyimpanan SSD minimal 30 GB

## 🗺️ Topologi Jaringan & Arsitektur

![Phase1.00 gambar 1](/assets/phase-1-sandbox/skema-visual-create-vm.drawio.png)

### Tabel Alokasi Sumber Daya Virtual (VMware Workstation)

| VM Name                       | vCPU (Cores) | RAM (Instalasi) | RAM (Runtime) | Storage Capacity    | OS / Versi                      |
| ----------------------------- | ------------ | --------------- | ------------- | ------------------- | ------------------------------- |
| Ubuntu_Server_1.2_24 (Master) | 1 Core       | 4 GB            | 2 GB          | 20 GB (Single File) | Ubuntu 24.04 LTS (LVM)          |
| Ubuntu_Server_2.2_24 (Slave)  | 1 Core       | - (Kloning)     | 2 GB          | 20 GB (Linked Disk) | Ubuntu 24.04 LTS (LVM) [17, 18] |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Pembuatan Spesifikasi VM Server 1 (Master)

- **Penjelasan Singkat:** Menyusun wadah perangkat keras virtual (_virtual hardware container_) dengan alokasi khusus untuk menjamin kelancaran instalasi.
- **Urutan Langkah pada VMware GUI:**

1.  Buka VMware Workstation Pro, lalu pilih **Create a New Virtual Machine**.
    ![Phase1.00 gambar 2](/assets/phase-1-sandbox/New-VM1.1.PNG)
2.  Pilih jenis instalasi **Typical (recommended)**, lalu klik _Next_.
    ![Phase1.00 gambar 3](/assets/phase-1-sandbox/New-VM1.2.PNG)
3.  Pilih **Installer disc image file (iso)**, arahkan _path_ ke file `.iso` Ubuntu 24.04 LTS Anda.
    ![Phase1.00 gambar 4](/assets/phase-1-sandbox/New-VM1.3.PNG)
4.  Isi kolom informasi pengguna (Nama, Username, Password) sesuai kebutuhan lab Anda.
    ![Phase1.00 gambar 5](/assets/phase-1-sandbox/New-VM1.4.PNG)
5.  Beri nama VM pertama Anda, misalnya `ubuntu-server-1-24`.
    ![Phase1.00 gambar 6](/assets/phase-1-sandbox/New-VM1.5.PNG)
6.  Pada kapasitas disk, masukkan **20.0 GB** dan pilih opsi **Store virtual disk as a single file** untuk performa I/O yang lebih stabil.
    ![Phase1.00 gambar 7](/assets/phase-1-sandbox/New-VM1.6.PNG)
7.  Klik tombol **Customize Hardware** sebelum menekan _Finish_ untuk mengoptimalkan pengaturan berikut:
    - **Memory:** Atur sementara ke **4096 MB (4 GB)**. Alokasi ini penting untuk mencegah kegagalan dekompresi paket sistem operasi saat proses instalasi.
    - **Processors:** Atur menjadi **1 Core**.
    - **Display:** Hilangkan centang pada opsi **Accelerate 3D graphics**. Menonaktifkan fitur ini sangat krusial untuk mencegah pembebanan CPU host berlebih saat proses render instalatur grafis.
      ![Phase1.00 gambar 8](/assets/phase-1-sandbox/New-VM1.7.PNG)
      ![Phase1.00 gambar 9](/assets/phase-1-sandbox/New-VM1.8.PNG)

8.  Klik _Close_, lalu klik _Finish_.
    ![Phase1.00 gambar 10](/assets/phase-1-sandbox/New-VM1.9.PNG)

---

### Langkah 2: Proses Instalasi Interaktif & Aktivasi LVM

- **Penjelasan Singkat:** Menjalankan instalasi Ubuntu dengan memilih opsi penyimpanan dinamis Logical Volume Manager (LVM) [19].
- **Urutan Langkah pada VM Console:**

1.  Nyalakan VM, tunggu hingga layar instalasi bahasa muncul, pilih bahasa sesuai keinginan Anda, lalu klik _Next_.
    ![Phase1.00 gambar 11](/assets/phase-1-sandbox/New-VM1.10.PNG)
2.  Pilih opsi **Interactive Installation** untuk mengontrol penuh konfigurasi sistem.
    ![Phase1.00 gambar 12](/assets/phase-1-sandbox/New-VM1.12.PNG)
3.  Pilih **Default Installation** (opsi ini sudah memadai untuk kebutuhan pembelajaran sandbox kita).
    ![Phase1.00 gambar 13](/assets/phase-1-sandbox/New-VM1.13.PNG)
4.  Pada halaman _Installation Type_, centang **Erase disk and install Ubuntu**, lalu klik tombol **Advanced Features** [19].
    ![Phase1.00 gambar 14](/assets/phase-1-sandbox/New-VM1.15.PNG)
5.  Di dalam menu Advanced Features, pilih **Use LVM (Logical Volume Manager)**, lalu klik _OK_ [19].
    ![Phase1.00 gambar 15](/assets/phase-1-sandbox/New-VM1.16.PNG)
6.  Lanjutkan proses dengan membuat akun administratif utama, menentukan zona waktu (Region), lalu klik **Install**.
    ![Phase1.00 gambar 16](/assets/phase-1-sandbox/New-VM1.17.PNG)
    ![Phase1.00 gambar 17](/assets/phase-1-sandbox/New-VM1.18.PNG)
7.  Tunggu hingga proses instalasi selesai (durasi bergantung pada kecepatan SSD host). Setelah selesai, pilih **Restart Now**.
    ![Phase1.00 gambar 18](/assets/phase-1-sandbox/New-VM1.19.PNG)

---

### Langkah 3: Optimalisasi Pasca Instalasi & Pembuatan Snapshot Base

- **Penjelasan Singkat:** Menurunkan alokasi memori kembali ke kapasitas operasional (2 GB) dan mengambil gambar kondisi awal sistem (_snapshot_) [17, 18].
- **Perintah & Langkah GUI:**

1.  Setelah sistem berhasil masuk ke desktop baru untuk pertama kali, matikan VM secara aman (_Shutdown_).
2.  Buka _Virtual Machine Settings_ untuk VM `Ubuntu_Server_1.2_24`.
3.  Pada bagian **Memory**, turunkan kapasitasnya kembali menjadi **2048 MB (2 GB)** untuk menghemat RAM fisik host Anda. Klik _OK_.
4.  Di menu navigasi atas VMware, pilih menu **VM** -> **Snapshot** -> **Take Snapshot...** [18]
    ![Phase1.00 gambar 19](/assets/phase-1-sandbox/New-VM1.23.PNG)
5.  Beri nama snapshot ini `Fresh Install` atau `Base`, lalu isi deskripsi dengan _"Instalasi bersih Ubuntu 24.04 LTS dengan partisi LVM sebelum optimasi jaringan."_ Klik _Take Snapshot_.
    ![Phase1.00 gambar 20](/assets/phase-1-sandbox/New-VM1.24.PNG)

---

### Langkah 4: Kloning Cepat Server 2 menggunakan Linked Clone

- **Penjelasan Singkat:** Membuat replika Server 2 menggunakan metode _Linked Clone_ berbasis titik _snapshot_ Server 1 untuk menghemat kapasitas harddisk fisik host hingga 90% [17, 18].
- **Urutan Langkah pada VMware GUI:**

1.  Klik kanan pada VM `ubuntu-server-1-24` di menu library VMware, pilih **Manage** -> **Clone...** [18]
    ![Phase1.00 gambar 21](/assets/phase-1-sandbox/New-VM1.25.PNG)
2.  Pada jendela Clone Wizard, klik _Next_.
3.  Di bagian _Clone Source_, pilih opsi **An existing snapshot (powered off only)**, lalu pilih nama snapshot `Fresh Install` yang telah Anda buat sebelumnya. Klik _Next_ [18].
    ![Phase1.00 gambar 22](/assets/phase-1-sandbox/New-VM1.26.PNG)
4.  Di bagian _Clone Type_, pilih **Create a linked clone** (Pilihan mutlak untuk efisiensi penyimpanan) [17, 18].
    ![Phase1.00 gambar 23](/assets/phase-1-sandbox/New-VM1.28.PNG)
5.  Beri nama VM baru ini `ubuntu-server-2-24`, tentukan folder penyimpanannya, lalu klik _Finish_.

---

### Langkah 5: Penambahan Adapter Jaringan Ganda untuk Server 2

- **Penjelasan Singkat:** Menyesuaikan kapasitas hardware Server 2 hasil kloning dan menambahkan adapter jaringan segmen LAN internal.
- **Urutan Langkah pada VMware GUI:**

1.  Klik pada VM `Ubuntu_Server_2.2_24` baru hasil kloning, pilih **Edit virtual machine settings**.
    ![Phase1.00 gambar 24](/assets/phase-1-sandbox/Edit-VM-Settings1.1.PNG)
2.  Pastikan kapasitas RAM telah terkonfigurasi ke **2048 MB (2 GB)** dan Prosesor ke **1 Core**.
    ![Phase1.00 gambar 25](/assets/phase-1-sandbox/Edit-VM-Settings1.2.PNG)
3.  Klik tombol **Add...** di bagian bawah, pilih **Network Adapter**, lalu klik _Finish_.
    ![Phase1.00 gambar 26](/assets/phase-1-sandbox/Edit-VM-Settings1.3.PNG)
4.  Pada Network Adapter baru yang muncul (Adapter 2), ubah tipenya menjadi **LAN Segment**.
5.  Klik tombol **LAN Segments...**, buat segmen baru bernama `JALUR_INTERNAL` (jika belum ada), lalu pilih nama segmen tersebut dari menu _dropdown_. Klik _OK_.
    ![Phase1.00 gambar 27](/assets/phase-1-sandbox/Edit-VM-Settings1.4.PNG)

---

## 🔍 Verifikasi & Troubleshooting

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **Proses Instalasi Membeku / Macet di Tengah Jalan:**
    - _Gejala:_ Indikator proses instalasi berhenti bergerak dan sistem tidak merespons (_freeze_) saat melakukan ekstraksi paket.
    - _Penyebab:_ Kapasitas memori RAM VM terlalu kecil (< 2 GB) saat memproses file image sistem operasi yang berukuran besar.
    - _Solusi:_ Matikan paksa VM melalui VMware (_Force Power Off_), buka pengaturan perangkat keras virtual, naikkan alokasi RAM sementara menjadi 4 GB, lalu jalankan ulang proses instalasi dari awal. Jangan lupa untuk menurunkan kembali RAM ke 2 GB setelah instalasi selesai.
2.  **Sistem Sangat Lambat & Kursor Mouse Patah-Patah Saat Instalasi:**
    - _Gejala:_ Pergerakan grafis di dalam konsol VM berjalan sangat lambat dan membebani kinerja CPU komputer host Windows.
    - _Penyebab:_ Fitur akselerasi grafis 3D emulasi bawaan VMware aktif, membebani pemrosesan grafis virtual secara berlebihan pada komputer host yang tidak memiliki GPU diskrit bertenaga tinggi.
    - _Solusi:_ Matikan VM, masuk ke _Virtual Machine Settings_ -> _Display_, hilangkan tanda centang pada opsi **Accelerate 3D graphics**, lalu jalankan kembali VM Anda.
