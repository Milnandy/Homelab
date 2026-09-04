# 🌐 Enterprise Hybrid Home Lab Journey

Selamat datang di repositori dokumentasi infrastruktur sistem dan jaringan saya. Proyek ini dibangun secara metodologis dari nol, bertransformasi dari simulasi mesin virtual hingga pembangunan pusat data mandiri fisik (_physical bare-metal home lab_).

---

## 🗺️ Peta Perjalanan Infrastruktur (Roadmap)

### 🔴 Phase 1: Single-Host Virtual Sandbox (VMware Workstation)

_Fokus:_ Menguasai administrasi sistem Linux, manajemen paket, penyimpanan dinamis, pengamanan akses, dan penyediaan layanan dasar (_core services_) di atas satu host hypervisor tipe 2.

- **[Modul 1.0: Pembuatan VM](./docs/phase-1-virtual-sandbox/00-vm-provisioning-architecture.md)**
- **[Modul 1.1: Jaringan Dasar & SSH Custom Socket](./docs/phase-1-virtual-sandbox/01-basic-networking.md)**
- **[Modul 1.2: SFTP Jail, Cron Job, & Cockpit Dashboard](./docs/phase-1-virtual-sandbox/02-sftp-cron-cockpit.md)**
- **[Modul 1.3: Redundansi DNS Server BIND9 Master-Slave](./docs/phase-1-virtual-sandbox/03-dns-master-slave.md)**
- **[Modul 1.4: Web Server Apache2, Virtual Hosts, & SSL SAN](./docs/phase-1-virtual-sandbox/04-apache-virtualhosts.md)**
- **[Modul 1.5: High Availability & Load Balancing (HAProxy vs Nginx)](./docs/phase-1-virtual-sandbox/05-load-balancing.md)**
- **[Modul 1.6: File Sharing Enterprise dengan Samba (Security & ACL)](./docs/phase-1-virtual-sandbox/06-samba-file-sharing.md)**
- **[Modul 1.7: Proxy Service (Squid Forward & Reverse Proxy)](./docs/phase-1-virtual-sandbox/07-squid-proxy.md)**
- **[Modul 1.8: Manajemen Penyimpanan Dinamis dengan LVM](./docs/phase-1-virtual-sandbox/08-lvm-storage.md)**
- **[Modul 1.9: Mail Server Komprehensif (Postfix, Dovecot, & Roundcube)](./docs/phase-1-virtual-sandbox/09-mail-server.md)**

---

### 🟡 Phase 2: Virtual Networking & RouterOS Integration (VMware & GNS3/EVE-NG)

_Fokus:_ Memindahkan kontrol jaringan dari virtual switch default ke router virtual (MikroTik CHR / Cisco IOSv) untuk mempelajari segmentasi jaringan nyata, firewall, VLAN, dan manajemen bandwidth sebelum diterapkan di perangkat fisik.

- _Status:_ **Perencanaan**
- **[Modul 2.1: Inisialisasi & Lisensi MikroTik CHR di Hypervisor](./docs/phase-2-virtual-networking/01-mikrotik-chr-initial-setup.md)**
- **[Modul 2.2: Implementasi VLAN & Inter-VLAN Routing](./docs/phase-2-virtual-networking/02-vlan-segmentation-routing.md)**
- **[Modul 2.3: Kebijakan Keamanan Firewall & Source-NAT/Dest-NAT](./docs/phase-2-virtual-networking/03-firewall-nat-hardening.md)**

---

### 🟢 Phase 3: Physical Bare-Metal Home Lab (Proxmox VE & Local AI)

_Fokus:_ Migrasi penuh dari virtualisasi desktop ke perangkat keras server fisik (Lenovo ThinkCentre Tiny). Membangun klaster container terdistribusi, mengonfigurasi storage redundan, dan menjalankan Local AI/LLM secara mandiri 24/7.

- _Status:_ **Perencanaan**
- **[Modul 3.1: Instalasi Bare-Metal Proxmox VE & Desain Storage Pool](./docs/phase-3-physical-homelab/01-proxmox-ve-baremetal.md)**
- **[Modul 3.2: Migrasi Layanan ke Docker Swarm & Klaster K3s](./docs/phase-3-physical-homelab/02-docker-swarm-k3s-cluster.md)**
- **[Modul 3.3: Integrasi Model AI Lokal (Ollama & Llama 3) untuk Penggunaan 24/7](./docs/phase-3-physical-homelab/03-local-llm-ollama-deployment.md)**
