# 🌐 Enterprise Hybrid Home Lab Journey

Repositori ini berisi dokumentasi proses belajar saya dalam membangun homelab berbasis Linux dan jaringan virtual. Fokus utama dokumentasi ini adalah melatih kemampuan operasional dan troubleshooting di lingkungan lab, sambil menyiapkan artefak yang bisa dipakai sebagai portofolio teknis untuk menunjukkan pemahaman saya terhadap infrastrukturnya.

---

## Status progres

Dokumen yang sudah saya ulang dan validasi secara praktik pada tahap awal ini adalah:

- [Modul 00: Provisioning & Persiapan VM](./docs/phase-1-virtual-sandbox/00-vm-provisioning-architecture.md) — pembuatan VM, snapshot base, dan linked clone
- [Modul 01: Inisialisasi Sandbox OS & Jaringan Ganda](./docs/phase-1-virtual-sandbox/01-basic-networking.md) — dual-NIC, IP forwarding, SSH custom port, dan gateway internal
- [Modul 02: SFTP Jail, Cron Job, & Cockpit Dashboard](./docs/phase-1-virtual-sandbox/02-sftp-cron-cockpit.md) — chroot SFTP, cron automation, dan monitoring terpusat

### Rencana pengembangan selanjutnya

- melanjutkan modul berikutnya untuk DNS, web server, reverse proxy, file sharing, storage, dan mail server
- menata kembali aset visual dan penamaan file agar lebih konsisten dan siap ditampilkan ke publik
- memperbaiki gaya dokumentasi agar lebih formal, ringkas, dan terlihat profesional sebagai portofolio teknis

---

## Peta perjalanan belajar

### Phase 1: Single-host virtual sandbox

Pada fase ini saya fokus mempelajari dasar-dasar operasional server dan jaringan melalui lingkungan VM yang dibangun secara bertahap:

- administrasi Linux
- konfigurasi jaringan virtual
- keamanan akses dan SSH
- hardening berbasis grup dan chroot
- penjadwalan tugas otomatis
- monitoring layanan
- troubleshooting sistem dan jaringan

Daftar modul:

- [Modul 00: Provisioning & Persiapan VM](./docs/phase-1-virtual-sandbox/00-vm-provisioning-architecture.md) — menyiapkan VM master, snapshot base, dan clone Server 2
- [Modul 01: Inisialisasi Sandbox OS & Jaringan Ganda](./docs/phase-1-virtual-sandbox/01-basic-networking.md) — konfigurasi dual-NIC, gateway, dan SSH custom port
- [Modul 02: SFTP Jail, Cron Job, & Cockpit Dashboard](./docs/phase-1-virtual-sandbox/02-sftp-cron-cockpit.md) — isolasi akses SFTP dan monitoring multi-server

### Phase 2: Networking dan routing lanjutan

Tahap berikutnya akan fokus pada routing, VLAN, firewall, dan integrasi router virtual untuk memperluas pemahaman jaringan di tingkat yang lebih realistis.

### Phase 3: Bare-metal dan platform lokal

Tahap ini akan membahas migrasi dari lingkungan virtual ke infrastruktur fisik, termasuk Proxmox, container, serta layanan lokal yang lebih siap untuk produksi skala kecil.

---

## Catatan penting

Dokumen ini masih menampilkan konteks pembelajaran dan eksperimen lab, bukan konfigurasi produksi siap pakai. Tujuan utamanya adalah menunjukkan proses belajar yang konsisten, dokumentasi yang dapat ditelusuri, dan kemampuan troubleshooting dalam infrastruktur berbasis Linux.
