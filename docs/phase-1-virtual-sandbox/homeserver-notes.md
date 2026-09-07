# Video 1: NanangMrk (Server Mini , RT RW NET, Hosting , Nas , HomeLab. Murah Untuk Pemula)

**Hardware:**

1. Mikrotik E50UG
2. switch managed ruijie RGS208GC
3. pc server
4. HTB distribution
5. dll

![Modul x gambar 1](/assets/phase-1-sandbox/module-x-topologi.PNG)

> mikrotik hanya untuk manajemen bandwith, all interface inbound outbound dimanage oleh **_"ruijie"_**

![Modul x gambar 2](/assets/phase-1-sandbox/module-x-topologi1.1.PNG)

---

# Video 2: NanangMrk (Cara Membuat Server dengan Proxmox di Mini PC Murah & Mudah)

1. Download ISO Proxmox
   - Buka Browser "proxmox.com" pilih menu downloads dan download "Proxmox VE 9.2 ISO Installer"
     ![Module x gambar 3](/assets/phase-1-sandbox/module-x-proxmox.PNG)
2. Burning ke Flashdisk
   - Buka software untuk burning rufus, link download "rufus.ie" pilih menu downloads dan sesuaikan pilihan download "rufus-4.15.exe"
     ![Module x gambar 4](/assets/phase-1-sandbox/module-x-rufus.PNG)
3. Proses Installasi Proxmox
   - Hubungkan flashdisk ke thinkcentre, keyboard dan jika tidak ada monitor bisa menggunakan hdmi capture untuk via laptop
   - Masuk ke BIOS, Enable _"Smart Power On"_ pada menu Power, untuk nyala otomatis ketika tiba2 mati karena listrik padam dsb
   - Pada menu startup masuk ke _"Primary Boot Sequence"_ pastikan fashdisk yang berisi proxmox berada di paling atas(primary) bisa menekan tombol "+" / "shift + =" untuk memindakannya ke paling atas
   - Pada Installasi pilih saja yang "Graphical" sisanya sesuaikan saja
4. Post Installation
   - Setup Proxmox, bisa manual bisa pake script, untuk script bisa cari https://community-scripts.org/scripts/post-pve-install?from=scripts

     ![Module x gambar 5](/assets/phase-1-sandbox/module-x-proxmox1.1.PNG)

   - Setelah Install semua distro Linux yang gratis sudah tersedia, tinggal install saja. jika ingin install **CasaOS** bisa cari lagi di "community-scripts sebelumnya"
