# Konfigurasi Dasar MikroTik RouterOS (Studi Kasus Jaringan Kantor & Simulasi ISP)

Dokumentasi praktik laboratorium jaringan komputer menggunakan **PNETLab** dan **Winbox** untuk menerapkan konfigurasi dasar **MikroTik RouterOS**. Skenario lab ini mensimulasikan topologi jaringan kantor yang terhubung ke router ISP hingga mendistribusikan koneksi internet ke perangkat klien (PC).

---

## 1. Topologi Jaringan

![Topologi Jaringan](<Topologi Jaringan.png>)

### Skema Koneksi Interface
* **`ISP (Cloud0)`** $\leftrightarrow$ **`eth1` Router-ISP**: Jalur sumber internet asli dan akses remote Winbox `Router-ISP`.
* **`ISP (Cloud0)`** $\leftrightarrow$ **`eth1` Router-Kantor**: Jalur khusus *Out-of-Band Management* untuk remote Winbox `Router-Kantor`.
* **`eth2` Router-ISP** $\leftrightarrow$ **`eth2` Router-Kantor**: Jalur WAN kantor (mendapatkan IP dinamis segmen `192.168.1.0/24`).
* **`eth3` Router-Kantor** $\leftrightarrow$ **`eth0` VPC**: Jalur jaringan lokal (LAN) kantor menuju klien (segmen `192.168.50.0/24`).

---

## 2. Tabel Pengalamatan IP (IP Addressing)

| Perangkat | Interface | Peran / Koneksi | IP Address | Metode Konfigurasi |
| :--- | :--- | :--- | :--- | :--- |
| **Router-ISP** | `eth1` | WAN (ke Cloud0 / Internet) | `192.168.100.x/24` | DHCP Client |
| **Router-ISP** | `eth2` | LAN ISP (ke Router-Kantor) | `192.168.1.1/24` | Static + DHCP Server |
| **Router-Kantor** | `eth1` | Management (Winbox) | `192.168.100.x/24` | DHCP Client (`Distance: 2`) |
| **Router-Kantor** | `eth2` | WAN (ke Router-ISP) | `192.168.1.x/24` | DHCP Client (`Distance: 1`) |
| **Router-Kantor** | `eth3` | LAN Kantor (ke VPC) | `192.168.50.1/24` | Static + DHCP Server |
| **PC-Client (VPC)** | `eth0` | End-User Client | `192.168.50.x/24` | DHCP Client (`ip dhcp`) |

---

## 3. Tahapan Konfigurasi `Router-ISP`

`Router-ISP` berperan sebagai modem/router dari penyedia layanan internet yang membagikan IP publik/WAN (`192.168.1.0/24`) ke router kantor.

### A. Konfigurasi IP Address (`eth2`)
Menambahkan IP Gateway pada interface `eth2` yang mengarah ke `Router-Kantor`:
* **Menu Winbox:** `IP` $\rightarrow$ `Addresses` $\rightarrow$ `+`
* **Address:** `192.168.1.1/24` | **Interface:** `eth2`

![Address List Router ISP](<Address List Router ISP.png>)

### B. Konfigurasi DHCP Server (`eth2`)
Membagikan alamat IP secara otomatis ke `Router-Kantor`:
* **Menu Winbox:** `IP` $\rightarrow$ `DHCP Server` $\rightarrow$ `DHCP Setup`
* **DHCP Server Interface:** `eth2` (ikuti wizard hingga selesai dengan rentang `192.168.1.2-192.168.1.254`).

![DHCP Server Router ISP](<DHCP Server Router ISP.png>)

### C. Konfigurasi Firewall NAT (`Masquerade`)
Menerjemahkan jaringan lokal ISP agar dapat diteruskan ke internet melalui `eth1`:
* **Menu Winbox:** `IP` $\rightarrow$ `Firewall` $\rightarrow$ tab `NAT` $\rightarrow$ `+`
* **Chain:** `srcnat` | **Out. Interface:** `eth1` | **Action:** `masquerade`

![Firewall NAT Router ISP](<Firewall NAT Router ISP.png>)

---

## 4. Tahapan Konfigurasi `Router-Kantor`

`Router-Kantor` (simulasi perangkat MikroTik RB941-2nD) menerima koneksi internet dari `Router-ISP` dan mendistribusikannya ke jaringan lokal kantor.

### A. Konfigurasi DHCP Client (`eth1` & `eth2`)
* **`eth2` (WAN Utama):** Mengambil IP dari `Router-ISP` (`192.168.1.x`) dengan `Default Route Distance = 1`.
* **`eth1` (Management):** Mengambil IP dari `Cloud0` untuk akses Winbox dengan `Default Route Distance = 2` agar trafik internet tetap melewati `Router-ISP`.

![DHCP Client Router Kantor](<DHCP Client Router Kantor.png>)

### B. Konfigurasi IP Address Lokal (`eth3`)
Menambahkan IP Gateway untuk jaringan lokal kantor pada interface `eth3`:
* **Menu Winbox:** `IP` $\rightarrow$ `Addresses` $\rightarrow$ `+`
* **Address:** `192.168.50.1/24` | **Interface:** `eth3`

![Address List Router Kantor](<Address List Router Kantor.png>)

### C. Konfigurasi DHCP Server Lokal (`eth3`)
Membagikan alamat IP otomatis kepada perangkat klien (`VPC`):
* **Menu Winbox:** `IP` $\rightarrow$ `DHCP Server` $\rightarrow$ `DHCP Setup`
* **DHCP Server Interface:** `eth3` (rentang IP `192.168.50.2-192.168.50.254`).

![DHCP Server Router Kantor](<DHCP Server Router Kantor.png>)

### D. Konfigurasi Firewall NAT (`Masquerade`)
Menerjemahkan segmen IP lokal kantor (`192.168.50.0/24`) menuju interface WAN `eth2`:
* **Menu Winbox:** `IP` $\rightarrow$ `Firewall` $\rightarrow$ tab `NAT` $\rightarrow$ `+`
* **Chain:** `srcnat` | **Out. Interface:** `eth2` | **Action:** `masquerade`

![Firewall NAT Router Kantor](<Firewall NAT Router Kantor.png>)

---

## 5. Pengujian & Verifikasi (`PC-Client / VPC`)

Pengujian dilakukan dari konsol **VPC** dengan meminta IP otomatis (`ip dhcp`) dan melakukan uji koneksi berjenjang (*ping*):
1. `ping 192.168.50.1` $\rightarrow$ Menguji koneksi ke Gateway `Router-Kantor` (**Berhasil**).
2. `ping 192.168.1.1` $\rightarrow$ Menguji koneksi melintasi router menuju Gateway `Router-ISP` (**Berhasil**).
3. `ping google.com` $\rightarrow$ Menguji resolusi DNS dan koneksi ke internet (**Berhasil**).

![Testing Ping Client (VPC)](<Testing Ping Client (VPC).png>)

---

## 6. Catatan Troubleshooting Selama Praktikum

1. **Konflik Subnet antara Management (`eth1`) dan LAN (`eth3`):**
   * *Masalah:* Awalnya `eth3` dikonfigurasi dengan `192.168.100.1/24`, yang ternyata bentrok dengan subnet `Cloud0` pada `eth1` (`192.168.100.0/24`), menyebabkan munculnya flag `DAC+` pada tabel *routing* dan paket tidak melewati `Router-ISP`.
   * *Solusi:* Mengubah segmen LAN `eth3` menjadi `192.168.50.1/24` agar seluruh interface memiliki subnet yang unik sehingga *routing* berjalan normal.
2. **Prioritas Default Route:**
   * *Masalah:* Jalur internet memotong (*bypass*) langsung lewat interface management `eth1`.
   * *Solusi:* Menaikkan nilai `Default Route Distance` pada DHCP Client `eth1` menjadi `2`, sehingga jalur utama tetap melewati `eth2` (`Router-ISP`) dengan `Distance = 1`.
