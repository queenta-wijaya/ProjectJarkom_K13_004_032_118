# Konfigurasi K-13 — Soal 3 (Static Routing VLSM), Soal 4 (DHCP) & Soal 5 (NAT)

> File: **TOPOLOGI K-13.pkt** — network dasar `10.104.0.0` (K=13), sesuai `VLSM_Subnetting.xlsx`.

## Soal 3 — Static Routing (SELESAI, teruji)

- **Alamat link (L1–L6 /30, dari tabel):**
  - L1 PUSAT–DC: .33/.34 | L2 PUSAT–UTARA: .37/.38 | L3 PUSAT–SELATAN: .41/.42
  - L4 PUSAT–TIMUR: .45/.46 | L5 PUSAT–BARAT: .49/.50 | L6 UTARA–PELABUHAN: .53/.54
  - PUSAT–ISP: 203.0.113.52/30 (PUSAT .54, ISP .53)
- **Routing:**
  - R-DC/BARAT/TIMUR/SELATAN/PELABUHAN: hanya default route ke router di atasnya
  - R-UTARA: default ke R-PUSAT + 2 static route ke LAN Pelabuhan via .54
  - R-PUSAT: default ke ISP + 12 static route ke semua LAN
  - R-ISP: tanpa route privat (persiapan NAT soal 5)
- **VLAN 10/20 + router-on-a-stick** di 7 switch; interface router semua `no shutdown`.
- Hasil uji: ping lintas LAN berhasil (CCTV→PJU TTL 125, 3 hop).

## Soal 4 — DHCP (SELESAI, ter-audit 25/25 + teruji di PT)

- **Statis (tetap):** semua interface router, Server-BALAI-KOTA (10.104.241.2), Server-STORAGE-DC (10.104.239.2), Server-8.8.8.8.
- **Dinamis (DHCP):** 12 perangkat pengguna (8 PC + 4 HP).
- **Pool DHCP di router** (gateway + DNS lokal 10.104.239.2, 10 alamat pertama di-exclude):

| Router | Pool | Network | Gateway |
|---|---|---|---|
| R-PUSAT | LAN-BALAI-KOTA | 10.104.224.0/21 | 10.104.224.1 |
| R-UTARA | WIFI-UTARA / SENSOR-LL | 10.104.0.0/18, 10.104.236.0/23 | .0.1 / .236.1 |
| R-BARAT | WIFI-BARAT / PUSKESMAS | 10.104.192.0/19, 10.104.238.0/24 | .192.1 / .238.1 |
| R-TIMUR | WIFI-TIMUR / SENSOR-BANJIR | 10.104.128.0/19, 10.104.232.0/22 | .128.1 / .232.1 |
| R-SELATAN | WIFI-SELATAN / PJU | 10.104.64.0/19, 10.104.240.128/25 | .64.1 / .240.129 |
| R-PELABUHAN | SCADA / KANTOR | 10.104.160.0/19, 10.104.240.0/25 | .160.1 / .240.1 |

- **CCTV Kota (R-DC):** TANPA pool di router — DHCP dari **Server-STORAGE-DC**:
  - Pool `CCTV-KOTA`: 10.104.96.0/19, gw 10.104.96.1, DNS 10.104.239.2, start IP 10.104.96.11
  - Relay: `ip helper-address 10.104.239.2` **hanya** di R-DC G0/1.10
- Contoh config (R-PUSAT):
```
ip dhcp excluded-address 10.104.224.1 10.104.224.10
ip dhcp pool LAN-BALAI-KOTA
 network 10.104.224.0 255.255.248.0
 default-router 10.104.224.1
 dns-server 10.104.239.2
```
- Teruji: PC-PEGAWAI dapat 10.104.224.11, PC-CCTV dapat 10.104.96.11 (dari server via helper), `show ip dhcp binding` di R-PUSAT menunjukkan lease Automatic.

## Soal 5 — NAT Overload di R-PUSAT (SELESAI, ter-audit 8/8)

Ketentuan: seluruh host mengakses server 8.8.8.8 memakai **satu IP publik milik R-PUSAT** (203.0.113.54), dan R-ISP **tetap tanpa** route ke jaringan privat.

Config di R-PUSAT (running + startup):
```
access-list 1 permit 10.104.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet1/0 overload
```
- `ip nat outside` → GigabitEthernet1/0 (menghadap ISP)
- `ip nat inside` → G2/0, G3/0, G4/0, G5/0, G6/0 (+.10/.20), G9/0 (semua sisi dalam)

Cara kerja: host privat 10.104.x.x diterjemahkan menjadi 203.0.113.54 dengan port unik (PAT). R-ISP cukup tahu jaringan publik saja — traffic balik diteruskan ke R-PUSAT karena tujuannya IP milik R-PUSAT sendiri.

Cara uji:
1. PC mana pun → `ping 8.8.8.8` → sekarang **Reply** (sebelum NAT: timed out)
2. R-PUSAT → `show ip nat translations` → entri icmp 10.104.x.x → 203.0.113.54
3. R-ISP → `show ip route` → tetap tanpa route privat
