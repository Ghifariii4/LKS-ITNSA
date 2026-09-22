# Konfigurasi DNS Master-Slave (Debian 13)

## Deskripsi Singkat
Konfigurasi DNS Master-Slave digunakan untuk *redundancy* (ketersediaan tinggi). Jika server DNS utama (Master) mati atau sibuk, server DNS cadangan (Slave) akan mengambil alih tugas melayani permintaan dari *client*. Zone transfer memungkinkan Slave menyalin data DNS dari Master secara otomatis.

## Topologi & Alokasi IP Address
Berdasarkan skema topologi menggunakan *network adapter* **vmnet8**:
* **Subnet:** `192.168.10.0/24`
* **VM 1 (LKS 1):** `192.168.10.1/24` (DNS MASTER / ns1)
* **VM 2 (LKS 2):** `192.168.10.3/24` (DNS SLAVE / ns2)
* **CLIENT:** `192.168.10.100/24`
* **Domain:** `lks.id`

---

## Konfigurasi VM 1: DNS MASTER (192.168.10.1)

### Langkah 1: Edit `named.conf.local`
Buka file konfigurasi zona pada server Master:
```bash
nano /etc/bind/named.conf.local
```
Tambahkan parameter `allow-transfer` agar Slave diizinkan menyalin data zona:
```text
zone "lks.id" {
    type master;
    file "/etc/bind/db.lks";
    allow-transfer { 192.168.10.3; }; 
};

zone "10.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192";
    allow-transfer { 192.168.10.3; };
};
```

### Langkah 2: Konfigurasi File Forward Zone (`db.lks`)
Buat atau edit file forward zone:
```bash
nano /etc/bind/db.lks
```
Isi dengan konfigurasi berikut sesuai standar kompetensi:
```text
$TTL    604800
@       IN      SOA     ns1.lks.id. admin.lks.id. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expired
                         604800 )       ; Min TTL
;
@       IN      NS      ns1.lks.id.
@       IN      NS      ns2.lks.id.

ns1     IN      A       192.168.10.1
ns2     IN      A       192.168.10.3
www     IN      A       192.168.10.1
```

### Langkah 3: Konfigurasi File Reverse Zone (`db.192`)
Buat atau edit file reverse zone:
```bash
nano /etc/bind/db.192
```
Isi dengan konfigurasi berikut:
```text
$TTL    604800
@       IN      SOA     ns1.lks.id. admin.lks.id. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expired
                         604800 )       ; Min TTL
;
@       IN      NS      ns1.lks.id.
@       IN      NS      ns2.lks.id.

1       IN      PTR     ns1.lks.id.
3       IN      PTR     
```

### Langkah 4: Restart BIND9 di Master
Cek sintaks dan restart layanannya:
```bash
named-checkconf
systemctl restart bind9
systemctl status bind9
```

---

## Konfigurasi VM 2: DNS SLAVE (192.168.10.3)

### Langkah 1: Install BIND9 di Slave
Pastikan BIND9 sudah terinstall di VM 2:
```bash
apt update
apt install bind9 bind9utils dnsutils -y
```

### Langkah 2: Edit `named.conf.local`
Pada server Slave, kita tidak perlu membuat file zona secara manual. File tersebut akan ditarik otomatis dari Master.
```bash
nano /etc/bind/named.conf.local
```
Tambahkan konfigurasi berikut:
```text
zone "lks.id" {
    type slave;
    file "/var/cache/bind/db.lks";
    masters { 192.168.10.1; };
};

zone "10.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.192";
    masters { 192.168.10.1; };
};
```

### Langkah 3: Restart BIND9 di Slave
```bash
systemctl restart bind9
```
Cek log untuk memastikan transfer zona dari Master ke Slave berhasil:
```bash
journalctl -u bind9 -f
```

---

## Pengujian dari Sisi Client (192.168.10.100)
1. Atur IP DNS pada Client agar menggunakan kedua server (Master dan Slave).
   ```text
   IP Address: 192.168.10.100
   Subnet Mask: 255.255.255.0
   Primary DNS: 192.168.10.1
   Secondary DNS: 192.168.10.3
   ```
2. Lakukan `nslookup lks.id` dari CMD/Terminal client.
   output:
   ![alt text](/modul-linux/DNS/image/output4.png)

3. **Uji Redundansi:** 
   - Matikan *service* BIND9 di VM 1 Master (`systemctl stop bind9`).
   - Lakukan `nslookup lks.id` lagi di client. 
   - Jika masih mendapatkan balasan (berarti dibalas oleh VM 2 Slave), maka konfigurasi Master-Slave **BERHASIL!**

   output:
   ![alt text](/modul-linux/DNS/image/output5.png)

