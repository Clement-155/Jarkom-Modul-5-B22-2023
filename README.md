# Jarkom-Modul-5-B22-2023

## SETUP

### DHCP

Installasi DHCP Relay di Heiter dan Himmel untuk forward request ke Revolte

**Kesulitan :**
- Stuck karena pada relay "Heiter", harus menambah eth0 pada daftar interface padahal tidak perlu di relay "Himmel" & tidak ada client yang dilayani di eth0


### 7 - https://scalingo.com/blog/iptables

##### DNAT

Melakukan DNAT untuk setiap paket ke-2 yang memenuhi syarat soal agar menuju ke server yang lain.

**Heiter**
```
iptables -A PREROUTING -t nat -p tcp -d 192.189.8.2 --dport 80 \
         -m statistic --mode nth --every 2 --packet 0              \
         -j DNAT --to-destination 192.189.14.142:80
```
**Himmel**

```
iptables -A PREROUTING -t nat -p tcp -d 192.189.14.142 --dport 443 \
         -m statistic --mode nth --every 2 --packet 0              \
         -j DNAT --to-destination 192.189.8.2:443
```

##### SNAT

Melakukan SNAT untuk paket yang diredirect agar datang dari interface router yang tepat.

**Heiter**
```
iptables \
  -A POSTROUTING \
  -t nat \
  -p tcp \
  -d 192.189.8.2   \
  --dport 80  \
  -j SNAT     \
  --to-source  192.189.14.141
```
**Himmel**
```
iptables \
  -A POSTROUTING \
  -t nat \
  -p tcp \
  -d 192.189.14.142   \
  --dport 80  \
  -j SNAT   \
  --to-source  192.189.8.1
```

![Alt text](foto/proof-7.JPG)

### 8

Menggunakan modul time untuk DROPPING paket di antara tanggal pemilu presiden sesuai website kpu.

![Alt text](foto/Jadwal_Pemilu.JPG)

**Heiter**
```
iptables -N DROPPING
iptables -A FORWARD -s 192.189.14.150 -d 192.189.8.2 -m time --datestart 2023-11-28 --datestop 2024-03-20 -j DROPPING
iptables -A DROPPING -j LOG --log-prefix "IPTables-Dropped: " --log-level 4
iptables -A DROPPING -j DROP
```
**Himmel**
```
iptables -N DROPPING
iptables -A FORWARD -s 192.189.14.150 -d 192.189.14.142 -m time --datestart 2023-11-28 --datestop 2024-03-20 -j DROPPING 
iptables -A DROPPING -j LOG --log-prefix "IPTables-Dropped: " --log-level 4
iptables -A DROPPING -j DROP
```

![Alt text](foto/proof-8.JPG)

### 9 - https://offkiltersecurity.com/2019/02/07/kalipot-part-2-detecting-nmap-scans-with-iptables/

Menggunakan modul recent untuk mencatat ip yang mengirim 20 paket dalam 10 menit. Untuk ip di blacklist, dilakukan drop.


Daftar ip dan paketnya bisa dilihat di direktori ini : `/proc/net/xt_recent/{{Nama List}}`

**YANG DIPAKAI**
```
iptables -N DROPPING
iptables -A INPUT -m recent --seconds 600 --hitcount 20 --name BLACKLIST --update -j DROPPING
iptables -A DROPPING -j LOG --log-prefix "IPTables-Dropped: " --log-level 4
iptables -A DROPPING -j DROP
iptables -A INPUT -d 192.189.8.2 -m recent --name BLACKLIST --set
```
```
iptables -N DROPPING
iptables -A INPUT -m recent --seconds 600 --hitcount 20 --name BLACKLIST --update -j DROPPING
iptables -A DROPPING -j LOG --log-prefix "IPTables-Dropped: " --log-level 4
iptables -A DROPPING -j DROP
iptables -A INPUT -d 192.189.14.142 -m recent --name BLACKLIST --set
```

- Sebelum nmap (Bisa PING)

![Alt text](foto/proof-9-1.JPG)

- Setelah nmap (Tidak bisa PING)

![Alt text](foto/proof-9-2.JPG)

- Setelah ip dihapus dari daftar BLACKLIST (Bisa PING)

![Alt text](foto/proof-9-3.JPG)

### 10

MASALAH : RSYSLOG tidak menulis log walaupun ada rule LOG di iptables.

PROBLEM : service rsyslog start
chown -v root:adm /var/log/syslog
chmod -v 660 /var/log/syslog

### KESALAHAN + REVISI
- **DHCP - BUG : Setiap client harus `echo nameserver 192.168.122.1 > /etc/resolv.conf` karena filenya menjadi kosong setiap beberapa menit**
- **7,8 - KESALAHAN : Harusnya instruksi di Himmel diletakkan di Frieren.**
- **9 - ERROR : Harusnya `iptables -A INPUT -d 192.189.14.142 -m recent --name BLACKLIST --set`, tidak ada drop**
- *10 - STUCK : Rsyslog tidak menulis log walaupun ada rule LOG di iptables*
