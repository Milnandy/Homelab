# Modul 08: Manajemen Penyimpanan Dinamis dengan LVM

## 📌 Ringkasan & Tujuan

- **Tujuan Pembelajaran:** Memahami konsep manajemen penyimpanan tingkat lanjut menggunakan Logical Volume Manager (LVM), melakukan deteksi disk baru secara _hot-plug_, serta melakukan ekspansi kapasitas penyimpanan secara langsung (_online resizing_) tanpa menghentikan layanan server [14].
- **Skenario / Case Study:** Server cadangan data (_backup server_) mengalami kehabisan kapasitas ruang penyimpanan. Di lingkungan industri nyata, mematikan server untuk menambah disk fisik sangat dihindari karena akan mengganggu kelangsungan bisnis (_downtime_). Kasus ini diselesaikan dengan menyisipkan disk virtual baru secara _hot-plug_, menyatukannya ke Volume Group, dan melonggarkan ruang penyimpanan tanpa meng-unmount sistem file aktif.
- **Kebutuhan Perangkat / Prasyarat:**
  - VMware Workstation Pro
  - Paket LVM2 (`lvm2`)
  - VM Ubuntu Server aktif

## 🗺️ Topologi Jaringan & Arsitektur

![Modul 08 Topology](../assets/08-lvm-storage-topology.png)

> _[Placeholder Gambar]: Diagram alur transformasi storage dari Physical Volume (PV) -> Volume Group (VG) -> Logical Volume (LV) yang dipetakan ke direktori /media/backup._

### Tabel Pengamatan IP / Interface

| Device Name        | Physical Disk | PV Name  | VG Name           | LV Name                     | Mount Point   |
| ------------------ | ------------- | -------- | ----------------- | --------------------------- | ------------- |
| ubuntu-server-1-24 | /dev/sdb (1G) | /dev/sdb | vg-idn            | logicalvolume-idn           | /media/backup |
| ubuntu-server-1-24 | /dev/sdc (1G) | /dev/sdc | vg-idn (Extended) | logicalvolume-idn (Resized) | /media/backup |

---

## 🚀 Langkah-Langkah Konfigurasi (Step-by-Step)

### Langkah 1: Hot-Plug Disk Baru & Rescan SCSI Bus

- **Penjelasan Singkat:** Menambahkan virtual harddisk baru berukuran 1 GB pada VMware dan mendeteksinya di sistem operasi tanpa melakukan reboot.
- **Perintah CLI / Konfigurasi:**

```bash
# Melakukan pemindaian paksa pada bus SCSI host virtual
ubuntu@ubuntu-server-1-24:~$ for host in /sys/class/scsi_host/host*/scan; do sudo echo "- - -" > "$host"; done

# Memverifikasi disk sdb telah terdeteksi oleh kernel
ubuntu@ubuntu-server-1-24:~$ lsblk
```

### Langkah 2: Inisialisasi LVM (PV, VG, dan LV Creation)

- **Penjelasan Singkat:** Mengubah partisi fisik /dev/sdb menjadi Physical Volume, menyatukannya ke dalam Volume Group, dan memotongnya menjadi Logical Volume.
- **Perintah CLI / Konfigurasi:**

```bash
# Membuat Physical Volume
ubuntu@ubuntu-server-1-24:~$ sudo pvcreate /dev/sdb

# Membuat Volume Group bernama vg-idn
ubuntu@ubuntu-server-1-24:~$ sudo vgcreate vg-idn /dev/sdb

# Membuat Logical Volume sebesar seluruh kapasitas sdb (1020 MB)
ubuntu@ubuntu-server-1-24:~$ sudo lvcreate -L +1020M -n logicalvolume-idn vg-idn
```

### Langkah 3: Formatting & Persistent Mounting

- **Penjelasan Singkat:** Memformat Logical Volume ke tipe sistem file Ext4 dan mengonfigurasinya di `/etc/fstab` agar otomatis terpasang saat boot.
- **Perintah CLI / Konfigurasi:**

```bash
# Memformat LV ke format Ext4
ubuntu@ubuntu-server-1-24:~$ sudo mkfs.ext4 /dev/vg-idn/logicalvolume-idn

# Membuat direktori mount point
ubuntu@ubuntu-server-1-24:~$ sudo mkdir -p /media/backup

# Melakukan mounting
ubuntu@ubuntu-server-1-24:~$ sudo mount /dev/vg-idn/logicalvolume-idn /media/backup/

# Memverifikasi mount point aktif
ubuntu@ubuntu-server-1-24:~$ df -h /media/backup
```

### Langkah 4: Hot-Plug Disk Kedua & Ekspansi Volume Group Online

- **Penjelasan Singkat:** Menyisipkan disk fisik kedua (/dev/sdc - 1 GB), merescan SCSI, dan menggabungkannya ke Volume Group yang telah ada untuk menambah total kapasitas storage pool.
- **Perintah CLI / Konfigurasi:**

```bash
# Memindai SCSI ulang untuk mendeteksi sdc
ubuntu@ubuntu-server-1-24:~$ for host in /sys/class/scsi_host/host*/scan; do sudo echo "- - -" > "$host"; done

# Mengubah sdc menjadi PV baru
ubuntu@ubuntu-server-1-24:~$ sudo pvcreate /dev/sdc

# Memperluas VG vg-idn menggunakan kapasitas sdc
ubuntu@ubuntu-server-1-24:~$ sudo vgextend vg-idn /dev/sdc
```

### Langkah 5: Ekspansi Logical Volume & Sistem File Online

- **Penjelasan Singkat:** Melonggarkan ukuran Logical Volume hingga memanfaatkan seluruh kapasitas bebas baru, lalu menyinkronkannya ke sistem file Ext4 yang sedang ter-mount tanpa unmount.
- **Perintah CLI / Konfigurasi:**

```bash
# Memperluas LV hingga sisa ruang bebas habis (100%FREE)
ubuntu@ubuntu-server-1-24:~$ sudo lvextend -l +100%FREE /dev/vg-idn/logicalvolume-idn

# Melakukan sinkronisasi sistem file Ext4 secara online
ubuntu@ubuntu-server-1-24:~$ sudo resize2fs /dev/vg-idn/logicalvolume-idn
```

---

## 🔍 Verifikasi & Troubleshooting

### Pengujian Validasi Status LVM & Kapasitas Akhir

```bash
# Verifikasi penambahan ruang penyimpanan aktif
ubuntu@ubuntu-server-1-24:~$ df -h /media/backup

# Verifikasi detail alokasi Volume Group fisik
ubuntu@ubuntu-server-1-24:~$ sudo vgs
ubuntu@ubuntu-server-1-24:~$ sudo lvs
```

### Log Masalah Terkenal & Solusi (Troubleshooting Log)

1.  **LVM commands not found:**
    - _Gejala:_ Perintah `pvcreate` atau `vgcreate` mengembalikan error command not found.
    - _Solusi:_ Instal paket pendukung manajemen LVM di Ubuntu dengan mengeksekusi perintah: `sudo apt install lvm2 -y`.
2.  **Kapasitas df -h Tetap 1 GB Setelah lvextend Sukses:**
    - _Gejala:_ Tampilan output `lvs` menunjukkan kapasitas LV telah berubah menjadi 2 GB, namun output perintah `df -h` tetap menunjukkan kapasitas 1 GB.
    - _Penyebab:_ Struktur sistem file Ext4 belum dideklarasikan ulang untuk mengisi blok penyimpanan baru yang ditambahkan oleh LVM.
    - _Solusi:_ Jalankan perintah `sudo resize2fs /dev/vg-idn/logicalvolume-idn` untuk melakukan ekspansi live di tingkat sistem file.
