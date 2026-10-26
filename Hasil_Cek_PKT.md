# Hasil Cek Kecocokan: TOPOLOGI K-13.pkt vs Tabel VLSM vs Konfigurasi No.3

## Peta fisik topologi (hasil dekripsi .pkt)

| Sisi A (port) | Sisi B (port) | Keterangan |
|---|---|---|
| R-PUSAT G1/0 | R-ISP G0/0 | Link WAN ke internet |
| R-PUSAT G2/0 | R-SELATAN G0/0 | L3 |
| R-PUSAT G3/0 | R-TIMUR G0/0 | L4 |
| R-PUSAT G4/0 | R-BARAT G0/0 | L5 |
| R-PUSAT G5/0 | R-DC G0/0 | L1 |
| R-PUSAT G6/0 | Switch-BALAI-KOTA | LAN Balai Kota |
| R-PUSAT G9/0 | R-UTARA G9/0 | L2 |
| R-UTARA G5/0 | Switch-UTARA | LAN Utara (2 LAN di 1 switch!) |
| R-UTARA G6/0 | R-PELABUHAN G0/0 | L6 |

Catatan fisik:
- **PC-PEGAWAI-BALAI-KOTA dan Server-BALAI-KOTA ada di switch yang sama** (Switch-BALAI-KOTA), padahal itu 2 subnet berbeda (67.104.160.0/21 dan 67.104.183.0/24) → butuh VLAN atau 2 kabel.
- **WiFi Publik Utara + Sensor Laut Lintas Utara juga satu switch** (Switch-UTARA, via AP-WIFI-UTARA) → butuh VLAN / router-on-a-stick di R-UTARA G5/0.

## TEMUAN KETIDAKCOCOKAN (di .pkt saat ini)

### 1. Pengalamatan campur aduk (10.13.x.x vs 67.104.x.x)
Sebagian interface masih memakai alamat sementara `10.13.x.x`, sebagian sudah pakai alamat VLSM `67.104.x.x`. Belum ada satu pun LAN yang lengkap sesuai tabel VLSM.

### 2. Kabel dan deskripsi interface tidak sinkron
R-PUSAT dikonfigurasi `G3/0 = L1-KE-R-DC`, tapi kabel fisik R-DC tertancap di **G5/0**. Deskripsi di semua link L1–L5 bergeser dari kabel yang sebenarnya (lihat peta fisik di atas).

### 3. Static route R-PUSAT "mati" (next-hop tidak terhubung)
```
ip route 67.104.128.0 255.255.224.0 67.104.242.34   <- 67.104.242.34 ada di R-DC,
                                                       tapi link PUSAT-DC = 10.13.100.0/30
```
Semua 6 static route R-PUSAT menunjuk next-hop `67.104.242.x` yang **bukan** next-hop di subnet link yang terhubung. Route tidak akan pernah dipakai.

### 4. Default route R-PUSAT ke 203.0.113.53 tapi tidak punya IP di subnet itu
R-ISP sudah `203.0.113.53/30`, tapi **tidak ada interface R-PUSAT di 203.0.113.52/30**. Default route-nya tidak berfungsi. (Sesuai soal: link PUSAT-ISP = 203.0.113.52/30 → R-PUSAT harusnya 203.0.113.54.)

### 5. R-UTARA hampir kosong
- Hanya 1 interface ber-IP: `G6/0 = 67.104.242.53/30` (ke R-PELABUHAN — sisi ini cocok).
- Tidak ada IP di link ke R-PUSAT (G9/0 kosong).
- Default route → `67.104.242.41` = alamat yang **tidak dimiliki siapa pun**.
- Tidak ada IP LAN sama sekali.
- Static route ke LAN Pelabuhan pakai next-hop .54 ✓ tapi ini satu-satunya yang benar.

### 6. Subnet LAN yang salah menurut tabel VLSM
| Interface | Di .pkt | Seharusnya (tabel VLSM) |
|---|---|---|
| R-DC LAN-CCTV-KOTA | 67.104.128.1/19 ⚠️ | **67.104.32.1/19** (128/19 itu milik WiFi Publik Barat) |
| R-DC LAN-SERVER-DC | 10.13.4.1/24 ⚠️ | **67.104.64.1/19** |
| R-BARAT kedua LAN | 10.13.5.1, 10.13.6.1 /24 ⚠️ | **67.104.128.1/19** dan **67.104.182.1/24** |
| R-SELATAN kedua LAN | 10.13.11.1, 10.13.12.1 /24 ⚠️ | **67.104.0.1/19** dan **67.104.184.129/25** |
| R-PELABUHAN LAN-SENSOR-SCADA | 67.104.160.1/19 ⚠️ | **BENTROK dengan Pegawai Balai Kota (67.104.160.0/21)** — subnet SCADA tidak ada di tabel |
| R-PUSAT kedua LAN | 10.13.1.1, 10.13.2.1 /24 ⚠️ | **67.104.160.1/21** dan **67.104.183.1/24** |
| R-TIMUR kedua LAN | 67.104.96.1/19 ✓, 67.104.176.1/22 ✓ | (sudah benar) |
| R-PELABUHAN LAN-KANTOR | 10.13.14.1/24 ⚠️ | **67.104.184.1/25** |

### 7. Link antar-router tidak konsisten kedua ujungnya
| Link | Ujung A | Ujung B | Status |
|---|---|---|---|
| L1 PUSAT–DC | 10.13.100.1/30 | 67.104.242.34/30 | ❌ beda subnet |
| L2 PUSAT–UTARA | 10.13.100.5/30 | (tanpa IP) | ❌ |
| L3 PUSAT–SELATAN | 10.13.100.9/30 | 10.13.100.10/30 | ✓ (tapi pakai alamat sementara) |
| L4 PUSAT–TIMUR | 10.13.100.13/30 | 67.104.242.46/30 | ❌ beda subnet |
| L5 PUSAT–BARAT | 10.13.100.17/30 | 67.104.242.38/30 | ❌ beda subnet |
| L6 UTARA–PELABUHAN | (tanpa IP) | 67.104.242.54/30 (UTARA .53) | ✓ |
| PUSAT–ISP | (tanpa IP) | 203.0.113.53/30 | ❌ R-PUSAT tanpa IP |

### 8. Sisa konfigurasi lain (untuk soal berikutnya)
- R-BARAT punya subinterface `G0/1.30/.31` (dot1Q) tanpa IP dan DHCP pool kosong — bisa dihapus karena kedua LAN sudah punya port fisik.
- Semua LAN (kecuali LAN Server Balai Kota) punya `ip helper-address 10.13.4.2` — rencananya DHCP server di Server-STORAGE-DC. Alamat ini harus diganti saat readdressing (soal 4).
- R-ISP sudah benar: 203.0.113.53/30 + 8.8.8.1/24, tanpa static route ✓ (sesuai syarat soal 5).

## KESIMPULAN
Skrip konfigurasi VLSM yang saya buat (Konfigurasi_No3_StaticRouting_VLSM.md) **sudah benar secara logika routing**, tetapi nomor interface-nya perlu disesuaikan dengan kabel fisik .pkt (lihat peta fisik), dan 2 hal perlu keputusan kalian:
1. Subnet untuk **Sensor dan SCADA Pelabuhan** tidak ada di tabel VLSM (tabel justru menuliskan "WiFi Publik Selatan" di R-PELABUHAN — padahal WiFi Publik Selatan ada di R-SELATAN, dan tabelnya sendiri mencatat hal itu di kolom Catatan).
2. LAN ganda di satu switch (Balai Kota dan Utara) butuh VLAN / tambah kabel.
