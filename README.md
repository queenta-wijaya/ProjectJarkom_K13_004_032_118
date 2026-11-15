# Smart City Surabaya - ETS Jaringan Komputer (Kelompok K-13)

File proyek: `TOPOLOGI K-13.pkt` (Cisco Packet Tracer)
Network dasar: `10.104.0.0` (K = 13), sesuai perhitungan `VLSM_Subnetting.xlsx`

## 1. Topologi

![Topologi Jaringan K-13](assets/topologi.png)

Struktur topologi:

- R-ISP di puncak, terhubung ke R-PUSAT lewat 203.0.113.0/30 dan menjalankan server publik 8.8.8.8.
- R-PUSAT menjadi pusat: cabang ke R-DC (L1), R-UTARA (L2), R-SELATAN (L3), R-TIMUR (L4), R-BARAT (L5), dan LAN Balai Kota sendiri.
- R-UTARA melanjutkan ke R-PELABUHAN (L6), jadi Pelabuhan tidak terhubung langsung ke Pusat.
- Tiap router daerah melayani 2 LAN lewat satu switch: VLAN 10 untuk perangkat pengguna dan VLAN 20 untuk server. Utara, Barat, Timur, dan Selatan juga punya Access Point untuk HP.
- Link antar-router diberi label L1 sampai L6 di gambar.

Skema pengalamatan memakai VLSM dari network dasar 10.104.0.0. Tiap LAN mendapat mask sesuai kebutuhan (dari /18 sampai /25), dan link antar-router memakai /30. Rincian perhitungan ada di VLSM_Subnetting.xlsx.

VLSM dipilih karena jumlah host berbeda-beda di setiap jaringan. Dengan ukuran subnet yang disesuaikan kebutuhan, alamat IP dapat digunakan lebih efisien dibandingkan jika semua jaringan memakai subnet berukuran sama. Selain itu, menurut kelompok kami, perhitungan VLSM lebih mudah dipahami dibandingkan dengan CIDR.

| LAN | Subnet | Lokasi |
|---|---|---|
| WiFi Utara | 10.104.0.0/18 | R-UTARA |
| WiFi Selatan | 10.104.64.0/19 | R-SELATAN |
| WiFi Timur | 10.104.128.0/19 | R-TIMUR |
| WiFi Barat | 10.104.192.0/19 | R-BARAT |
| CCTV Kota | 10.104.96.0/19 | R-DC |
| Sensor Lalu Lintas | 10.104.236.0/23 | R-UTARA |
| Sensor Banjir | 10.104.232.0/22 | R-TIMUR |
| Sensor SCADA | 10.104.160.0/19 | R-PELABUHAN |
| PJU | 10.104.240.128/25 | R-SELATAN |
| Kantor Pelabuhan | 10.104.240.0/25 | R-PELABUHAN |
| Puskesmas | 10.104.238.0/24 | R-BARAT |
| Pegawai Balai Kota | 10.104.224.0/21 | R-PUSAT |
| Server Storage DC | 10.104.239.0/24 | R-DC |
| Server Balai Kota | 10.104.241.0/27 | R-PUSAT |
| Link L1-L6 | 10.104.241.32/30 sampai /30 | antar-router |
| Ke ISP | 203.0.113.52/30 | R-PUSAT ke R-ISP |

## 2. Penjelasan Topologi

Hierarki jaringan terdiri dari tiga lapis:

1. R-ISP sebagai gerbang ke internet. Router ini tidak mengetahui jaringan privat.
2. R-PUSAT sebagai pusat yang menghubungkan semua router daerah (L1-L5) dan R-UTARA yang melanjutkan ke R-PELABUHAN (L6).
3. Router daerah (R-DC, R-SELATAN, R-TIMUR, R-BARAT, R-UTARA, R-PELABUHAN) yang masing-masing melayani 2 LAN.

Setiap switch memakai dua VLAN, yaitu VLAN 10 untuk perangkat pengguna dan VLAN 20 untuk server. Satu kabel trunk menghubungkan switch ke router, lalu router membaginya menjadi subinterface (contoh G0/1.10 dan G0/1.20) dengan encapsulation dot1Q. Dengan cara ini satu port router cukup untuk dua subnet.

Aturan jalur yang dipakai sesuai ketentuan soal:

- Default route hanya ke router tepat di atasnya: R-DC, R-BARAT, R-TIMUR, dan R-SELATAN menuju R-PUSAT; R-PELABUHAN menuju R-UTARA; R-UTARA menuju R-PUSAT; R-PUSAT menuju R-ISP.
- Hanya R-PUSAT yang memegang static route ke semua LAN.
- R-PELABUHAN tidak terhubung langsung ke R-PUSAT, semua trafficnya lewat R-UTARA.

## 3. Konfigurasi

### 3.1 Soal 3 - Static Routing (VLSM)

Semua IP dipasang manual pada interface, dan interface aktif diberi perintah no shutdown.

Static route disusun berjenjang:

| Router | Route | Keterangan |
|---|---|---|
| R-DC, R-BARAT, R-TIMUR, R-SELATAN | `ip route 0.0.0.0 0.0.0.0 <PUSAT>` | Hanya default route |
| R-PELABUHAN | `ip route 0.0.0.0 0.0.0.0 10.104.241.53` | Default ke R-UTARA |
| R-UTARA | default ke PUSAT + 2 route ke LAN Pelabuhan via .54 | Mengetahui 2 LAN Pelabuhan |
| R-PUSAT | default ke ISP + 12 static route ke semua LAN | Peta lengkap |
| R-ISP | tidak ada route privat | Disengaja, prasyarat NAT |

Contoh static route di R-PUSAT:

```
ip route 10.104.96.0 255.255.224.0 10.104.241.34
ip route 10.104.0.0 255.255.192.0 10.104.241.38
ip route 10.104.160.0 255.255.224.0 10.104.241.38
ip route 0.0.0.0 0.0.0.0 203.0.113.53
```

### 3.2 Soal 4 - DHCP

Interface router dan server memakai IP statis. Perangkat pengguna (8 PC dan 4 HP) memakai DHCP. Khusus subnet CCTV, alamat diberikan oleh server, bukan oleh router.

Contoh pool DHCP di R-PUSAT:

```
ip dhcp excluded-address 10.104.224.1 10.104.224.10
ip dhcp pool LAN-BALAI-KOTA
 network 10.104.224.0 255.255.248.0
 default-router 10.104.224.1
 dns-server 10.104.239.2
```

Keterangan:

- Excluded-address melindungi gateway dan sembilan alamat pertama agar tidak disewakan, sehingga host pertama mendapat alamat .11.
- DNS mengarah ke Server Storage DC (10.104.239.2).
- Total ada 11 pool di 6 router.

Untuk CCTV, R-DC tidak memiliki pool. Subinterface CCTV diberi relay berikut:

```
interface GigabitEthernet0/1.10
 ip helper-address 10.104.239.2
```

Server-STORAGE-DC menjalankan pool bernama CCTV-KOTA (network 10.104.96.0/19, gateway .1, start IP .11). Request DHCP dari PC-CCTV diteruskan lintas subnet oleh helper-address sampai ke server.

### 3.3 Soal 5 - NAT Overload

Seluruh host mengakses server 8.8.8.8 memakai satu IP publik milik R-PUSAT (203.0.113.54):

```
access-list 1 permit 10.104.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet1/0 overload
```

Keterangan:

- ip nat outside ditempatkan di G1/0 yang menghadap ISP, dan ip nat inside di 8 interface sisi dalam.
- ACL 1 mencakup seluruh 10.104.x.x sekaligus.
- Perintah overload membuat banyak host berbagi satu IP publik yang dibedakan lewat nomor port (PAT).
- R-ISP tetap tidak diberi route privat. Paket balik ditujukan ke 203.0.113.54 yang merupakan IP milik R-PUSAT sendiri, sehingga cukup connected route.

### 3.4 Soal 6 - Tunnel GRE
Tunnel GRE buat jalur virtual point to point antara R-PUSAT dan R-PELABUHAN. Lalu lintas antara subnet Balai kota dan subnet Pelabuhan diarahkan ke tunnel ini, sehingga tidak lagi lewat R-UTARA sebagai jalur biasa. 

| Item               | R-PUSAT                    | R-PELABUHAN                |
| ------------------ | -------------------------- | -------------------------- |
| Alamat tunnel      | 192.168.13.1/30            | 192.168.13.2/30            |
| Tunnel source      | G0/2 (10.104.241.37)       | G0/0 (10.104.241.54)       |
| Tunnel destination | 10.104.241.54              | 10.104.241.37              |
| Mode               | GRE over IP                | GRE over IP                |

Konfigurasi di R-PUSAT:
```
interface Tunnel0
 ip address 192.168.13.1 255.255.255.252
 tunnel source GigabitEthernet0/2
 tunnel destination 10.104.241.54
 tunnel mode gre ip
 no shutdown
 
ip route 10.104.241.52 255.255.255.252 10.104.241.38
no ip route 10.104.160.0 255.255.224.0 10.104.241.38
no ip route 10.104.240.0 255.255.255.128 10.104.241.38
ip route 10.104.160.0 255.255.224.0 192.168.13.2
ip route 10.104.240.0 255.255.255.128 192.168.13.2
```

Konfigurasi di R-PELABUHAN: 
```
interface Tunnel0
 ip address 192.168.13.2 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 10.104.241.37
 tunnel mode gre ip
 no shutdown
 
ip route 10.104.224.0 255.255.248.0 192.168.13.1
ip route 10.104.241.0 255.255.255.224 192.168.13.1
```

Keterangan: 

- Alamat sumber dan tujuan tunnel memakai alamat fisik link L2 (sisi R-PUSAT) dan link L6 (sisi R-PELABUHAN) MTU tunnel tersimpan 1476 byte (1500 dikurangi 24 byte header GRE dan IP tambahan)
- Route ke 10.104.241.52/30 (link L6) ditambahkan di R-PUSAT karena sebelumnya R-PUSAT tidak punya route ke subnet itu. Tanpa route ini, paket GRE menuju 10.104.241.54 lari ke default route ISP dan tidak sampai (ping bernilai `U`, destination unreachable)
- Route lama ke LAN Pelabuhan lewat R-UTARA (10.104.241.38) dihapus dan diganti next-hop 192.168.13.2 yang terjangkau lewat Tunnel0, sehingga tidak ada jalur pesaing. Jika route lama dibiarkan, dua route dengan tujuan dan jarak administratif sama akan sama-sama aktif dan trafik terbagi dua jalur
-  Di R-PELABUHAN ditambahkan route ke dua subnet Balai Kota (Pegawai 10.104.224.0/21 dan Server 10.104.241.0/27) lewat tunnel. Route ini lebih spesifik daripada default route 0.0.0.0/0 ke R-UTARA, dan router memilih prefix paling spesifik (longest prefix match), sedangkan tujuan lain tetap lewat default route. Kedua route tidak mencakup 10.104.241.37 (ujung tunnel), sehingga ujung tunnel tidak dicari lewat tunnel itu sendiri
- Bagi R-UTARA, paket tunnel hanyalah paket IP biasa dari 10.104.241.37 ke 10.104.241.54, diteruskan tanpa mengetahui isinya. Itu sebabnya R-UTARA tidak tampak lagi sebagai hop pada tracert
- GRE hanya membungkus paket asli dengan header IP dan GRE baru, tanpa mengacak atau mengautentikasi isinya. Siapa pun yang menyadap jalur fisik (misalnya di R-UTARA) tetap bisa membaca paket asli. Kerahasiaan dan integritas data baru diperoleh bila GRE dikombinasikan dengan IPsec

### 3.5 Soal 7 - Dynamic Routing OSPF
Soal 7 dikerjakan di file terpisah, yaitu `TOPOLOGI K13-OSPF.pkt`, yang dibuat sebagai salinan dari file utama `K13_VLSM.pkt`. File utama tetap memakai static route sesuai soal 3.
Link cadangan L7: L7 menghubungkan R-DC langsung ke R-UTARA, sehingga ada jalur cadangan di segitiga R-PUSAT, R-DC, dan R-UTARA
| Item              | Nilai                                              |
| ----------------- | -------------------------------------------------- |
| Subnet L7         | 10.104.241.56/30                                   |
| R-DC              | GigabitEthernet0/2, 10.104.241.57                  |
| R-UTARA           | GigabitEthernet7/0, 10.104.241.58                  |
| Rentang host      | 10.104.241.57 sampai 10.104.241.58                 |
| Broadcast         | 10.104.241.59                                      |

Alasan alamat ini belum terpakai: blok alamat pada tabel VLSM berakhir di 10.104.241.55 (broadcast link L6), sehingga 10.104.241.56/30 adalah blok kosong tepat sesudahnya. Blok ini tidak termasuk LAN mana pun (seluruh LAN berada di bawah 10.104.241.32), dan tidak bentrok dengan link ISP 203.0.113.52/30 maupun tunnel 192.168.13.0/30.
 
Konfigurasi interface L7:
```
R-DC:
interface GigabitEthernet0/2
 description L7-KE-R-UTARA
 ip address 10.104.241.57 255.255.255.252
 ip ospf cost 100
 no shutdown
 
R-UTARA:
interface GigabitEthernet7/0
 description L7-KE-R-DC
 ip address 10.104.241.58 255.255.255.252
 ip ospf cost 100
 no shutdown
```

Konfigurasi OSPF: 
```
router ospf 1
 router-id <ID>
 passive-interface default
 no passive-interface <interface link antar-router>
 network 10.104.0.0 0.0.255.255 area 0
```

| Router      | Router ID | Interface non-passive (link antar-router)                    |
| ----------- | --------- | ------------------------------------------------------------ |
| R-PUSAT     | 1.1.1.1   | G0/2 (L2), Serial0/2/0 (L1), Serial0/2/1 (L3), Serial0/3/0 (L4), Serial0/3/1 (L5) |
| R-DC        | 2.2.2.2   | Serial0/2/0 (L1), G0/2 (L7)                                  |
| R-UTARA     | 3.3.3.3   | G9/0 (L2), G6/0 (L6), G7/0 (L7)                              |
| R-SELATAN   | 4.4.4.4   | Serial0/2/0 (L3)                                             |
| R-TIMUR     | 5.5.5.5   | Serial0/2/0 (L4)                                             |
| R-BARAT     | 6.6.6.6   | Serial0/2/0 (L5)                                             |
| R-PELABUHAN | 7.7.7.7   | G0/0 (L6)                                                    |

Khusus R-PUSAT ditambahkan perintah berikut agar default route ke internet dibagikan ke router lain:
 
```
router ospf 1
 default-information originate
```

Static route yang diganti: Static route antar-router dihapus setelah OSPF aktif dan diuji, satu router pada satu waktu

| Router                            | Dihapus                                                    | Tetap ada                                           |
| --------------------------------- | ---------------------------------------------------------- | --------------------------------------------------- |
| R-DC, R-BARAT, R-TIMUR, R-SELATAN | `ip route 0.0.0.0 0.0.0.0 <PUSAT>`                         | tidak ada                                           |
| R-PELABUHAN                       | `ip route 0.0.0.0 0.0.0.0 10.104.241.53`                   | 2 route ke Balai Kota lewat tunnel (soal 6)         |
| R-UTARA                           | default ke PUSAT dan 2 route ke LAN Pelabuhan              | tidak ada                                           |
| R-PUSAT                           | 11 route ke LAN dan route link L6                          | default ke ISP dan 2 route ke Pelabuhan lewat tunnel |

Keterangan: 

- Syarat tidak mengirim pesan OSPF ke arah LAN Pengguna dan dipenugi dengan `passive-interface default`. Semua interface, termasuk subinterface LAN (contoh G0/1.10 dan G0/1.20), diam dari pesan OSPF, dan hanya interface link antar-router yang diaktifkan dengan `no passive-interface`. Subnet LAN tetap diumumkan ke router lain lewat pernyataan `network`.
- Link ke ISP (G0/0 di R-PUSAT) dan Tunnel0 ikut passive. Alamat ISP 203.0.113.x dan tunnel 192.168.13.x juga berada di luar `network 10.104.0.0/16`, sehingga OSPF tidak bocor ke ISP maupun ke tunnel.
- Default route ke internet hanya dikonfigurasi di R-PUSAT (static default ke 203.0.113.53). Router lain tidak punya default statis, tetapi mempelajarinya dari OSPF lewat `default-information originate` sebagai route `O*E2`.
- `ip ospf cost 100` dipasang di kedua ujung L7. Port serial punya cost bawaan 64, sedangkan Gigabit 1. Tanpa pengaturan ini, jalur memutar lewat L7 bisa tampak lebih murah (1 + 1) daripada link serial langsung (64) dan akan dipilih sebagai jalur utama. Dengan cost 100, link langsung tetap menjadi jalur utama dan L7 baru dipakai saat link lain putus.
- Dua route statis R-PUSAT dan dua route statis R-PELABUHAN yang menuju tunnel sengaja dipertahankan, karena itu bagian soal 6 (lalu lintas Balai Kota dan Pelabuhan harus lewat tunnel), bukan static route antar-router biasa.
- Hanya link L1 dan L2 yang punya jalur cadangan (lewat L7). Link L3, L4, L5, dan L6 menghubungkan satu wilayah saja, sehingga jika putus wilayah itu terisolasi. Itu keterbatasan topologi pada Gambar 1, bukan kesalahan konfigurasi.

### 3.6 Soal 8 - Layanan Aplikasi Smart City

Domain internal memakai `k13.smartcity.lab` (k + nomor kelompok). Layanan disebar di dua server dengan IP statis, dan seluruh perangkat pengguna memperoleh DNS `10.104.239.2` lewat DHCP

| Server             | Alamat IP     | Subnet mask     | Gateway       | DNS           | Layanan                      |
| ------------------ | ------------- | --------------- | ------------- | ------------- | ---------------------------- |
| Server-STORAGE-DC  | 10.104.239.2  | 255.255.255.0   | 10.104.239.1  | 10.104.239.2  | DNS, FTP (juga DHCP CCTV)    |
| Server-BALAI-KOTA  | 10.104.241.2  | 255.255.255.224 | 10.104.241.1  | 10.104.239.2  | Web (HTTP), Email (SMTP, POP3) |

#### 8a. DNS

| Nama                      | Tipe     | Alamat        | Server tujuan     |
| ------------------------- | -------- | ------------- | ----------------- |
| `www.k13.smartcity.lab`   | A Record | 10.104.241.2  | Server-BALAI-KOTA |
| `ftp.k13.smartcity.lab`   | A Record | 10.104.239.2  | Server-STORAGE-DC |
| `mail.k13.smartcity.lab`  | A Record | 10.104.241.2  | Server-BALAI-KOTA |

#### 8b. Web

Web berjalan di Server-BALAI-KOTA (Services, HTTP, HTTP On). Halaman `index.html` diganti dengan isi berikut:
 
```
<html>
<head><title>Portal Smart City Surabaya</title></head>
<body>
<h1>Portal Smart City Surabaya</h1>
<p>Kelompok K-13</p>
</body>
</html>
```
Halaman dibuka dari browser PC dengan alamat `http://www.k13.smartcity.lab`.

#### 8b. FTP

FTP berjalan di Server-STORAGE-DC sebagai penyimpanan rekaman CCTV (Services, FTP, Service On). Pengguna harus login dengan akun yang punya izin penuh:
 
| Item        | Nilai                                                                               |
| ----------- | ----------------------------------------------------------------------------------- |
| Akun        | ISI SETELAH DICEK (di file ada akun `k19` dan akun bawaan `cisco`)                  |
| Izin        | Write, Read, Delete, Rename, List                                                   |
| Klien       | PC-CCTV-KOTA, lewat `ftp ftp.k13.smartcity.lab` di Command Prompt                   |
| File uji    | `rekaman_cctv.txt`, dibuat dengan Text Editor di PC-CCTV-KOTA                       |
 
Perintah yang dipakai di klien:
 
```
ftp ftp.k13.smartcity.lab
(masukkan username dan password)
put rekaman_cctv.txt
dir
quit
```

#### 8d. Email anatar pegawai 

Email berjalan di Server-BALAI-KOTA (Services, EMAIL, SMTP Service dan POP3 Service On, Domain Name `k13.smartcity.lab`).

| Akun                          | User        | Dipakai di              |
| ----------------------------- | ----------- | ----------------------- |
| `pegawai@k13.smartcity.lab`   | `pegawai`   | PC-PEGAWAI-BALAI-KOTA   |
| `pelabuhan@k13.smartcity.lab` | `pelabuhan` | PC-KANTOR-PELABUHAN     |
 
Pengaturan di klien (Desktop, Email, Configure Mail):
 
| Isian                | PC-PEGAWAI-BALAI-KOTA         | PC-KANTOR-PELABUHAN             |
| -------------------- | ----------------------------- | ------------------------------- |
| Email Address        | pegawai@k13.smartcity.lab     | pelabuhan@k13.smartcity.lab     |
| Incoming Mail Server | mail.k13.smartcity.lab        | mail.k13.smartcity.lab          |
| Outgoing Mail Server | mail.k13.smartcity.lab        | mail.k13.smartcity.lab          |
| User Name            | pegawai                       | pelabuhan                       |
 
Keterangan:
 
- Semua nama layanan diakses lewat DNS, bukan alamat IP. Setiap PC memperoleh DNS `10.104.239.2` dari DHCP, sehingga nama `www`, `ftp`, dan `mail` dapat dipakai dari seluruh wilayah.
- Record `www` dan `mail` menunjuk ke Server-BALAI-KOTA (10.104.241.2), sedangkan `ftp` menunjuk ke Server-STORAGE-DC (10.104.239.2). Server DNS tetap satu, yaitu di Storage DC.
- Dari Pelabuhan, akses ke server Balai Kota (web dan email) melewati tunnel GRE, karena route ke 10.104.241.0/27 di R-PELABUHAN menunjuk ke tunnel (soal 6).
- Perangkat di LAN CCTV Kota mengakses FTP lintas subnet lewat gateway R-DC. Subnet CCTV (10.104.96.0/19) dan subnet server (10.104.239.0/24) berada di VLAN berbeda pada router yang sama, sehingga routing antar-VLAN dilakukan R-DC.
- Alamat email ditulis lengkap dengan domain `k13.smartcity.lab`. Salah ketik domain membuat server menolak surat dan mengirim notifikasi kegagalan ke pengirim.

## 4. Pengerjaan

Alur pengerjaan:

1. Perhitungan VLSM dari 10.104.0.0 menghasilkan 14 LAN dan 6 link /30 (VLSM_Subnetting.xlsx).
2. Pembuatan topologi dan kabel: 8 router, 7 switch, 4 AP, 15 PC/Server/HP.
3. Pengalamatan interface, pembuatan VLAN 10/20, trunk, router-on-a-stick, dan no shutdown.
4. Soal 3: static route berjenjang, lalu pengujian dengan show ip route, ping, dan tracert.
5. Soal 4: 11 pool DHCP dengan excluded-address, helper-address di R-DC, pool CCTV di Server-STORAGE-DC, dan peralihan host ke DHCP.
6. Soal 5: ACL dan NAT overload di R-PUSAT, penandaan ip nat inside dan outside.
7. Verifikasi menyeluruh: 112 pemeriksaan otomatis terhadap file ditambah uji manual di Packet Tracer.
8. Soal 6: mengganti R-PUSAT ke Cisco 2911 dengan kartu HWIC-2T karena router generik tidak mendukung interface tunnel, lalu memindahkan seluruh konfigurasi lama ke nama interface baru. Menambah route ke link L6 di R-PUSAT, membuat interface Tunnel0 di R-PUSAT dan R-PELABUHAN, lalu menguji tunnel dengan ping dan show interface tunnel 0. Mengganti route R-PUSAT ke LAN Pelabuhan dengan route lewat tunnel, menambah route balik ke Balai Kota di R-PELABUHAN, lalu membuktikan jalur dengan tracert sebelum dan sesudah
9. Soal 7: membuat salinan file utama untuk mengeharkan seluruh soal 7, menambahkan link L7 (10.104.241.56/30) anatara R-DC dan R-UTARA dan ngatur `ip ospf cost 100` di kedua ujung, memasang OSPF di 7 routher dengan `passive interface default`, setelah itu memeriksa tetangga OSPF dengan `show ip ospf neighbor` sampai semua `FULL` sementara static route dibiarkan, menghapus static route antar router satu per satu (router daerah,R_UTARA, lalu R-PUSAT) sambil menguji `show ip route` dan ping setiap route, terakhir menguji pemulihan dengan mematikan link L1 (R-PUSAT ke R-DC), lalu memeriksa jalur lewat L7 dan dipulikan kembali
10. Soal 8: mengaktifkan DNS di Server-STORAGE-DC, menambahkan tiga record A (`www`, `ftp`, `mail`) setelah itu mengujinya dengan `nslookup` dari PC di tujuh wilayah, mengaktifkan HTTP di Server-BALAI-KOTA, mengganti isi `index-html` Mmenjadi halaman "Portal Smart City Surabaya", lalu membukannya lewat nama domain dari beberapa wilayah, mengaktifkan FTP di Server-STORAGE-DC lalu mengunggah file dari PC-CCTV-KOTA, mengaktifkan SMTP dan POP3 di Server-BALAI-KOTA, mengisi domain `k13.smartcity.lab`, membuat akun, mengatur configure mail di PC Balai Kota dan PC Pelabuhan lalu menguji mengirim dan terima email.

## 5. Bukti Pengujian

### Soal 3 - Static Routing

| Uji | Perintah/Lokasi | Hasil |
|---|---|---|
| Ping ke Kantor Pelabuhan (lewat 3 router) | PC-CCTV `ping 10.104.240.2` | Reply 4/4, TTL 125 |
| Ping lintas kota lewat R-SELATAN | PC-CCTV `ping 10.104.240.130` | Reply 4/4, TTL 125 |
| Tabel routing lengkap | R-PUSAT `show ip route` | 12 route S dan default ke ISP |
| Isolasi ISP | R-ISP `show ip route` | tanpa route privat |
| Jalur berjenjang | R-PELABUHAN `show ip route` | default via 10.104.241.53 (R-UTARA) |
| Trunk dan VLAN | Switch `show vlan brief`, `show int trunk` | VLAN 10/20 dan trunk aktif |
| WiFi lewat AP | HP-BARAT `ping 10.104.238.2` | Reply |

#### Bukti ping:

Ping PC-CCTV ke PC-Kantor Pelabuhan (10.104.240.2):

![Ping 10.104.240.2](assets/soal_3-1.png)

Ping PC-CCTV ke PC-PJU Selatan (10.104.240.130):

![Ping 10.104.240.130](assets/soal_3-2.png)

### Soal 4 - DHCP

| Uji | Perintah/Lokasi | Hasil |
|---|---|---|
| Host dapat IP dinamis dari pool router | PC-PEGAWAI, IP Configuration | 10.104.224.11/21, gateway .1 |
| Pool bekerja di router | R-PUSAT `show ip dhcp binding` | 10.104.224.11, Automatic |
| CCTV dapat IP dari server | PC-CCTV, IP Configuration | 10.104.96.11/19, gateway .1 |
| Server DHCP aktif | Server-STORAGE-DC, Services, DHCP | pool CCTV-KOTA |
| Excluded-address bekerja | host pertama tiap pool | selalu .11, tidak pernah .1 sampai .10 |

Bukti:

PC-PEGAWAI mendapat IP dari pool router (10.104.224.11):

![PC-PEGAWAI DHCP](assets/soal_4-1.png)

PC-CCTV mendapat IP dari server lewat helper-address (10.104.96.11):

![PC-CCTV DHCP](assets/soal_4-2.png)

Hasil show ip dhcp binding di R-PUSAT:

![DHCP binding](assets/soal_4-3.png)

Pool CCTV-KOTA di Server-STORAGE-DC:

![Server DHCP](assets/soal_4-4.png)

Config DHCP pool di R-PUSAT:

![Show run R-PUSAT](assets/soal_4-5.png)

### Soal 5 - NAT

| Uji | Perintah/Lokasi | Hasil |
|---|---|---|
| Akses ke server publik | PC-PEGAWAI `ping 8.8.8.8` | Reply 4/4, TTL 126 |
| Terjemahan PAT aktif | R-PUSAT `show ip nat translations` | 10.104.224.11 menjadi 203.0.113.54 |
| Ketentuan ISP | R-ISP `show ip route` | tetap tanpa route privat |

Bukti:

Ping 8.8.8.8 dari PC-PEGAWAI berhasil:

![Ping 8.8.8.8](assets/soal_5-1.png)

Hasil show ip nat translations di R-PUSAT:

![NAT translations](assets/soal_5-2.png)

Sebelum NAT dikonfigurasi, ping 8.8.8.8 dari host yang sama gagal karena R-ISP tidak mengetahui jaringan privat. Setelah NAT, ping berhasil karena R-ISP hanya melihat IP publik 203.0.113.54 milik R-PUSAT.

### Verifikasi otomatis

Selain uji manual di Packet Tracer, konfigurasi diperiksa dengan skrip otomatis terhadap isi file:

| Kelompok cek | Jumlah | Hasil |
|---|---|---|
| Konsistensi subnet kedua ujung link antar-router | 7 | PASS |
| Trunk uplink dan access VLAN sesuai subnet host | 14 | PASS |
| Gateway host sama dengan IP router di subnetnya | 14 | PASS |
| Static route tiap router sesuai aturan berjenjang | 7 | PASS |
| VLAN database, dot1Q, running=startup, device ON, duplikasi IP, SSID | 37 | PASS |
| DHCP (pool, excluded, helper, host dinamis, server statis) | 25 | PASS |
| NAT (ACL, PAT, inside 8 interface, outside 1, posisi baris) | 8 | PASS |
| Total | 112 | 112 PASS, 0 FAIL |

### Soal 6 - Tunnel GRE

| Uji                                  | Perintah/Lokasi                                                  | Hasil                                                                      |
| ------------------------------------ | ---------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Tunnel aktif                         | R-PUSAT `show interface tunnel 0`                                | Tunnel0 up/up, GRE/IP, source 10.104.241.37, destination 10.104.241.54     |
| Ping antar ujung tunnel              | R-PUSAT `ping 192.168.13.2`                                      | Success rate 100 persen (5/5)                                              |
| Ping arah sebaliknya                 | R-PELABUHAN `ping 192.168.13.1`                                  | ISI SETELAH DIUJI                                                          |
| Route ke tunnel di R-PUSAT           | R-PUSAT `show ip route static`                                   | 10.104.160.0/19 dan 10.104.240.0/25 via 192.168.13.2                       |
| Route ke tunnel di R-PELABUHAN       | R-PELABUHAN `show ip route static`                               | 10.104.224.0/21 dan 10.104.241.0/27 via 192.168.13.1                       |
| Jalur Balai Kota ke Pelabuhan        | PC-PEGAWAI-BALAI-KOTA `tracert 10.104.240.11`                    | Hop: 10.104.224.1, 192.168.13.2, 10.104.240.11 (R-UTARA tidak tampak)      |
| Konektivitas melalui tunnel          | PC-PEGAWAI-BALAI-KOTA `ping 10.104.240.11`                       | Reply 4/4, TTL 126                                                         |
| Jalur Pelabuhan ke Balai Kota        | PC-KANTOR-PELABUHAN `tracert 10.104.241.2`                       | ISI SETELAH DIUJI (diharapkan melewati 192.168.13.1)                       |

Bukti: 

Tunnel0 di R-PUSAT aktif dengan protokol GRE/IP:
![interface tunnel 0](assets/soal_6-1.png)

Ping dari R-PUSAT ke ujung tunnel (192.168.13.2):
![Ping tunnel dari R-Pusat](assets/soal_6-2.png)

Ping dari R-PELABUHAN ke ujung tunnel (192.168.13.1):
![Ping tunnel dari R-Pelabuhan](assets/soal_6-3.png)

Route statis ke LAN Pelabuhan di R-PUSAT mengarah ke tunnel:
![Route R-Pusat](assets/soal_6-4.png)

Route statis ke Balai Kota di R-PELABUHAN mengarah ke tunnel:
![Route R-Pelabuhan](assets/soal_6-5.png)

Bukti jalur lewat tunnel (tracert sebelum dan sesudah):

Sebelum tunnel dipakai, trafik lewat R-UTARA (10.104.241.38) dan R-PELABUHAN (10.104.241.54), ping bernilai TTL 125:
![Sebelum](assets/soal_6-6.png)

Sesudah route diarahkan ke tunnel, hop R-UTARA tidak tampak lagi dan jalur melompat ke 192.168.13.2, ping bernilai TTL 126:
![Sesudah](assets/soal_6-7.png)

Ping dari PC-PEGAWAI-BALAI-KOTA ke PC-KANTOR-PELABUHAN setelah tunnel aktif:
![Ping](assets/soal_6-8.png)

Pengujian arah sebaliknya (PC-KANTOR-PELABUHAN ke Server Balai Kota 10.104.241.2), untuk membuktikan route balik lewat tunnel:
![pengujian](assets/soal_6-9.png)

### Soal 7 - OSPF

| Uji                                  | Perintah/Lokasi                                              | Hasil                                                                              |
| ------------------------------------ | ------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| Link L7 aktif                        | R-DC `ping 10.104.241.58`                                    | Reply 4/5 (paket pertama hilang karena ARP)                                        |
| Tetangga OSPF terbentuk              | `show ip ospf neighbor` di tiap router                       | Semua FULL: R-PUSAT 5 tetangga, R-UTARA 3, R-DC 2, router lainnya 1                |
| Tidak ada pesan OSPF ke LAN          | R-PUSAT, R-DC, R-UTARA `show ip protocols`                   | Interface LAN dan subinterface tercantum di Passive Interface(s)                   |
| Default route dari OSPF              | R-BARAT `show ip route`                                      | `O*E2 0.0.0.0/0` via 10.104.241.49, tanpa `S*`                                     |
| Default route dari OSPF di R-UTARA   | R-UTARA `show ip route 0.0.0.0`                              | `Known via "ospf 1"`, dari router-id 1.1.1.1 (R-PUSAT)                             |
| Static route antar-router sudah hilang | R-UTARA `show ip route static`                             | Kosong                                                                             |
| Sisa static di R-PUSAT               | R-PUSAT `show ip route static`                               | Hanya default ke ISP dan 2 route ke Pelabuhan lewat 192.168.13.2 (tunnel)          |
| Konektivitas ke internet             | PC di 6 wilayah `ping 8.8.8.8`                               | Reply 4/4 di CCTV, Pelabuhan, Selatan, Barat, Utara, dan Timur                     |
| Konektivitas antar wilayah           | PC-CCTV-KOTA `ping 10.104.240.11`                            | Reply 4/4                                                                          |
| Tunnel tetap dipakai                 | PC-PEGAWAI-BALAI-KOTA `tracert 10.104.240.11`                | Hop 10.104.224.1, 192.168.13.2, 10.104.240.11                                      |
| Pemulihan saat link putus            | PC-CCTV-KOTA `ping -t 8.8.8.8` saat Serial0/2/0 R-PUSAT dimatikan | 1 paket hilang, lalu pulih sendiri, TTL berubah dari 125 menjadi 124          |
 
Bukti:

Tetangga OSPF di R-PUSAT (5 tetangga, semua FULL):
![Neighbor R-PUSAT](assets/soal_7-1.png)

Tetangga OSPF di R-UTARA (R-PELABUHAN, R-PUSAT, R-DC lewat L7):
![Neighbor R-UTARA](assets/soal_7-2.png)

Tetangga OSPF di R-DC (R-UTARA lewat L7 dan R-PUSAT):
![Neighbor R-DC](assets/soal_7-3.png)

Interface LAN bersifat passive, sehingga tidak ada pesan OSPF ke LAN pengguna. R-PUSAT (termasuk link ISP dan Tunnel0 yang juga passive):
![R-pusat](assets/soal_7-4.png)

R-DC:
![R-DC](assets/soal_7-5.png)

R-UTARA:
![R-UTARA](assets/soal_7-6.png)

Router daerah mempelajari default route dari OSPF (`O*E2`), bukan dari static. Contoh di R-BARAT:
![Deafult](assets/soal_7-7.png)


R-UTARA tidak punya static route lagi, dan default route-nya berasal dari OSPF:
![Route R_UTARA](assets/soal_7-8.png)


Static route yang tersisa di R-PUSAT hanya default ke ISP dan dua route tunnel:
![Static R-PUSAT](assets/soal_7-9.png)


pemulihan otomatis saat link bacbone putus: 
![Ping -t saat link putus](assets/soal_7-10.png)

Log OSPF di R-PUSAT: tetangga R-DC (2.2.2.2) turun saat `shutdown`, lalu kembali FULL setelah `no shutdown`:
![R-PUSAT](assets/soal_7-11.png)

### Soal 8 - Layanan Aplikasi

| Uji                                  | Perintah/Lokasi                                                           | Hasil                                                                                  |
| ------------------------------------ | ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Record DNS terdaftar                 | Server-STORAGE-DC, Services, DNS                                          | Tiga record A: ftp, mail, www                                                          |
| DNS dari seluruh wilayah             | PC tiap wilayah `nslookup www.k13.smartcity.lab`, `ftp...`, `mail...`     | Dijawab 10.104.239.2: www 10.104.241.2, ftp 10.104.239.2, mail 10.104.241.2            |
| Web dari beberapa wilayah            | Web Browser di PC-CCTV-KOTA, PC-KANTOR-PELABUHAN, PC-PUSKESMAS-BARAT      | Halaman "Portal Smart City Surabaya" dan "Kelompok K-13" terbuka                       |
| FTP upload dari LAN CCTV             | PC-CCTV-KOTA `ftp ftp.k13.smartcity.lab`, `put rekaman_cctv.txt`, `dir`   | `Transfer complete - 22 bytes`, `rekaman_cctv.txt` tampil di daftar file server        |
| Email Balai Kota ke Pelabuhan        | PC-PEGAWAI-BALAI-KOTA Compose, PC-KANTOR-PELABUHAN Receive                | Surat "Tes dari Balai Kota" masuk di kotak masuk Pelabuhan                             |
| Email Pelabuhan ke Balai Kota        | PC-KANTOR-PELABUHAN Compose, PC-PEGAWAI-BALAI-KOTA Receive                | Surat "Balasan dari Pelabuhan" masuk di kotak masuk Balai Kota                         |
 
#### 8a - DNS
 
Tiga record A di Server-STORAGE-DC:
![Record DNS](assets/soal_8a-1.png)

Hasil `nslookup` dari tujuh wilayah. Semua memakai server DNS 10.104.239.2 dan memberi jawaban yang sama:
![Hasil DNS](assets/soal_8a-2.png)
![Hasil DNS](assets/soal_8a-3.png)

#### 8b - Web 

Portal dibuka lewat nama domain dari PC-CCTV-KOTA:
![Hasil Web CCTV](assets/soal_8b-1.png)

Dari PC-KANTOR-PELABUHAN (lewat tunnel GRE):
![Hasil Web Pelabuhan](assets/soal_8b-2.png)


Dari PC-PUSKESMAS-BARAT:
![Hasil dari R-Barat](assets/soal_8b-3.png)

#### 8c - FTP

Akun FTP dan izinnya di Server-STORAGE-DC (Services, FTP):
![Akun FTP](assets/soal_8c-1.png)


Sesi FTP dari PC-CCTV-KOTA (login, `put`, dan `dir`). Hasil `put` menampilkan `Transfer complete - 22 bytes`, dan `rekaman_cctv.txt` muncul di daftar file server (nomor 35):
![Hasil FTP](assets/soal_8c-2.png)

#### 8d - Email antar pegawai

Pengaturan email di Server-BALAI-KOTA (domain dan dua akun):
![Pengaturan Email](assets/soal_8d-1.png)

Surat dari Balai Kota diterima di PC-KANTOR-PELABUHAN:
![Surat diterima](assets/soal_8d-3.png)


Balasan dari Pelabuhan diterima di PC-PEGAWAI-BALAI-KOTA:
![Balasan](assets/soal-8d-2.png)