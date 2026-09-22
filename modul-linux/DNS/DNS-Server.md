## Deskripsi Singkat
Pada lingkungan Linux, layanan DNS umumnya ditangani oleh **BIND9** (Berkeley Internet Name Domain). Berbeda dengan Windows Server yang menggunakan GUI, konfigurasi DNS di Debian 13 sepenuhnya berbasis teks (CLI). 

## Persiapan (Prerequisites)
Pastikan Debian 13 sudah dikonfigurasi dengan **IP Address Static**. Pada panduan ini kita menggunakan studi kasus:
* **IP Address Server:** `192.168.10.1`
* **Nama Domain:** `lks.id`

---

## Langkah-Langkah Konfigurasi

### Langkah 1: Install Paket BIND9
Pastikan server terhubung ke repositori lokal/internet (bisa lihat di README.md), lalu jalankan perintah instalasi:
```bash
apt update
apt install bind9 bind9utils bind9-doc dnsutils -y
```

### Langkah 2: Konfigurasi Zona (Zones)
File `named.conf.id` digunakan untuk mendefinisikan *Forward Zone* (Domain ke IP) dan *Reverse Zone* (IP ke Domain).
1. Buka file konfigurasi lokal BIND9:
   ```bash
   nano /etc/bind/named.conf.id
   ```
2. Tambahkan baris berikut di bagian paling bawah:
   ```text
   zone "lks.id" {
       type master;
       file "/etc/bind/db.forward";
   };

   zone "10.168.192.in-addr.arpa" {
       type master;
       file "/etc/bind/db.reverse";
   };
   ```
   *(Catatan: Penulisan reverse zone dibalik dari `192.168.10` menjadi `10.168.192`)*

### Langkah 3: Konfigurasi Forward Zone (Domain ke IP)
Kita akan menyalin file default BIND sebagai *template*.
1. Copy file default ke file *forward*:
   ```bash
   cp /etc/bind/db.id /etc/bind/db.forward
   ```
2. Edit file `db.forward`:
   ```bash
   nano /etc/bind/db.forward
   ```
3. Ubah konfigurasinya menjadi seperti ini (Perhatikan letak **titik** di belakang `lks.id.`):
   ```text
   $TTL    604800
   @       IN      SOA     lks.id. root.lks.id. (
                                 2         ; Serial
                            604800         ; Refresh
                             86400         ; Retry
                           2419200         ; Expire
                            604800 )       ; Negative Cache TTL
   ;
   @       IN      NS      lks.id.
   @       IN      A       192.168.10.1
   www     IN      A       192.168.10.1
   ```

### Langkah 4: Konfigurasi Reverse Zone (IP ke Domain)
1. Copy file default ke file *reverse*:
   ```bash
   cp /etc/bind/db.127 /etc/bind/db.reverse
   ```
2. Edit file `db.reverse`:
   ```bash
   nano /etc/bind/db.reverse
   ```
3. Ubah konfigurasinya menjadi seperti ini:
   ```text
   $TTL    604800
   @       IN      SOA     lks.id. root.lks.id. (
                                 1         ; Serial
                            604800         ; Refresh
                             86400         ; Retry
                           2419200         ; Expire
                            604800 )       ; Negative Cache TTL
   ;
   @       IN      NS      lks.id.
   1       IN      PTR     lks.id.
   1       IN      PTR     www.lks.id.
   ```
   *(Angka `1` di bawah IN PTR merujuk pada oktet terakhir dari IP `192.168.10.1`)*

### Langkah 5: Pengecekan Sintaks dan Restart Layanan
Sebelum me-restart service, pastikan tidak ada kesalahan ketik (typo).
1. Cek konfigurasi utama:
   ```bash
   named-checkconf
   ```
   *(Jika tidak ada output (kosong), berarti sintaks aman)*
2. Cek zona forward:
   ```bash
   named-checkzone lks.id /etc/bind/db.forward
   ```
3. Cek zona reverse:
   ```bash
   named-checkzone 10.168.192.in-addr.arpa /etc/bind/db.reverse
   ```
4. Restart dan cek status BIND9:
   ```bash
   systemctl restart bind9
   systemctl status bind9
   ```
   *Pastikan statusnya **active (running)**.*

---

## Pengujian (Testing)
Pada server Debian atau PC Client (linux/windows), arahkan file `/etc/resolv.conf` atau di network configuration kalau di windows, ke IP DNS Server terlebih dahulu:
LINUX
```bash
nano /etc/resolv.conf
nameserver 192.168.10.1
```

Lakukan pengujian dengan `nslookup` atau `dig`:
```bash
nslookup lks.id
nslookup 192.168.10.1
dig lks.id
```


output:


![alt text](/modul-linux/DNS/image/output1.png)

![alt text](/modul-linux/DNS/image/output2.png)

![alt text](/modul-linux/DNS/image/output3.png)


---

## Troubleshooting
* **Service BIND9 Failed saat di-restart:** 
  Hampir 90% kasus disebabkan oleh *syntax error*. Kurang titik koma (`;`) di `named.conf.id`, atau lupa menaruh titik (`.`) di akhir nama domain pada file forward/reverse. Gunakan perintah `journalctl -xeu bind9.service` untuk melihat baris mana yang error.
* **nslookup memunculkan SERVFAIL atau REFUSED:**
  Periksa permission file zone. Jalankan `chown bind:bind /etc/bind/db.*` untuk memastikan user BIND memiliki akses ke file zona yang baru dibuat.
