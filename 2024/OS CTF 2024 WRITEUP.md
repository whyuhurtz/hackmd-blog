---
title: OS CTF 2024 WRITEUP
author: ywdh
tags: ["ctf"]
date: 2024-07-13
---

# OS CTF 2024 WRITEUP

[TOC]

# Misc

## Cyber Quiz

### Description

<details>
My teacher assigned me this quiz and told me that I have to
answer each question correctly otherwise I won't be able to
pass the test. Can you help me? Pwease
<br><br>
Author: @5h1kh4r
<br>
nc 34.16.207.52 12345
</details>

### Solve Walkthrough

- Just answer all the questions after making a connection.
- ChatGPT helped a lot. 😂

![Cyber Quiz-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_misc1-01.png)

### Flag

==OSCTF{L33t_Kn0wl3Dg3}==

# OSINT

## The statue is being repaired

### Description

<details>
This statue is typical of an university, can you find out the
name of the statue and the city where it exists?
<br><br>
Format flag: OSCTF{name of statue_city}
<br><br>
If the name of the statue is Statue of Liberty at New York, US,
the flag will be: OSCTF{statueofliberty_newyork}
</details>

The repaired statue image:

![The statue is being repaired-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_osint1-01.png)

### Solve Walkthrough

- Save the picture chall and try to search with Google Lens. You will find 2 posts in Facebook (in Vietnam language).

![The statue is being repaired-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_osint1-02.png)

- The image looks same, and then I explore the tag in the post `#FPTAround`.
- Seems like it refers to the **FPT University in Vietnam**.

![The statue is being repaired-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_osint1-03.png)

- I do more exploration about **fptaround** and luckily I got the community page in Facebook: [_https://www.facebook.com/fptaround_](https://www.facebook.com/fptaround).

![The statue is being repaired-04](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_osint1-04.png)

- I try to translate what is the Intro description

![The statue is being repaired-05](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_osint1-05.png)

- Nice, we got the city name: **Ho Chi Minh**.
- Next, we've to found what is the statue name. I just search FPT university statue in google, and found the useful wiki link: [_https://commons.wikimedia.org/wiki/File:Self-made_Man_statue_at_FPT_University_Ha_Noi.jpg_](https://commons.wikimedia.org/wiki/File:Self-made_Man_statue_at_FPT_University_Ha_Noi.jpg).
- Alhamdulillah, I got the name of statue: **Self-made man**.

![The statue is being repaired-06](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_osint1-06.png)

- Finally, we can crafting the correct flag format: **selfmademan_hochiminh**.

### Flag

==OSCTF{selfmademan_hochiminh}==

# Web

## Introspection

### Description

<details>
Welcome to the Secret Agents Portal. Find the flag hidden in
the secrets of the Universe !!!
<br><br>
Author: @5h1kh4r
<br>
Web Instance: http://34.16.207.52:5134
</details>

### Solve Walkthrough

- Visit the web page, and open the inspect element in web browser.

![Introspection-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_web1-01.png)

- Notice that the web page have JS file embedded. Open the `script.js` file and yeah.. we found the flag.

```javascript
function checkFlag() {
	const flagInput = document.getElementById('flagInput').value;
	const result = document.getElementById('result');
	const flag = "OSCTF{Cr4zY_In5P3c71On}";
	
	if (flagInput === flag) {
		result.textContent = "Congratulations! You found the flag!";
		result.style.color = "green";
	} else {
		result.textContent = "Incorrect flag. Try again.";
		result.style.color = "red";
	}
}
```

- Check the flag for make sure. Sooo... ez dude.

![Introspection-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_web1-02.png)

### Flag

==OSCTF{Cr4zY_In5P3c71On}==

## Heads or Tails

### Description

<details>
pfft .. Listen, I've gained access to this login portal but I'm not
able to log in. The admins are surely hiding something from
the public, but ... I don't understand what. Here take the link
and be quiet, don't share it with anyone
<br><br>
Author: @5h1kh4r
<br>
Web instance: http://34.16.207.52:3635/
</details>

### Solve Walkthrough

- From the title we know that the challenge is the web app vulnerable to **SQL injection attack**.
- Just performing basic SQL injection payload to bypass login: `1' OR 1=1--`.
- Use the payload in username and password and it will make the username and password val0id. 4. In the default page, it will show the **fake flag**. You must be careful okay.

![Heads or Tails-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_web2-01.png)

- But, you can access the admin page directly by changing the endpoint to `/admin` (if you know).

![Heads or Tails-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_web2-02.png)

- If you don't know what all endpoints or subdirectories inside the web page, you can do **directory scanning** using tool like **dirsearch**, **feroxbuster**, **dirb**, or **gobuster**. I recommend to use **dirsearch** cause they have default wordlist.

### Flag

==OSCTF{D1r3ct0RY_BrU7t1nG_4nD_SQL}==

## Indoor WebApp

### Description

<details>
The production of this application has been completely
indoor so that no corona virus spreads, but that's an old talk
right?
<br><br>
Author: @5h1kh4r
<br>
Web Instance: http://34.16.207.52:2546
</details>

### Solve Walkthrough

- Given a web app that we can see personal information, but notice that every person is have unique id: `?user_id` value parameters.

![Indoor WebApp-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_web3-01.png)

- I try to change it to person **2** or `?user_id=2` and I got the flag.

![Indoor WebApp-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_web3-02.png)

- We can perform brute force attack to check if the spesific `user_id` is exist or not by using **Burp Suite** or simply **cURL** (combined with for/while loop).

![Indoor WebApp-03](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_web3-03.png)

- Luckily, we just have 3 available users.

### Flag

==OSCTF{1nd00r_M4dE_n0_5enS3}==

## Action Notes

### Description

<details>
I have created this notes taking app so that I don't forget
what I've studied
<br><br>
Author: @5h1kh4r
<br>
Web Instance: http://34.16.207.52:8965
</details>

### Solve Walkthrough

- I just try to login as **admin** and luckily the password that I guess is **admin123** so I can the flag in the admin page.

![Action Notes-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/os-ctf_web4-01.png)

- Try to guess common user passwords, such as:

| Username | Password  |
| :------- | --------- |
| admin    | admin     |
| admin    | password  |
| admin    | admin123  |
| admin    | admin1234 |

### Flag

==OSCTF{Av0id_S1mpl3_P4ssw0rDs}==
