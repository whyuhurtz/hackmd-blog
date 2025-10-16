---
title: LITCTF 2024 QUALS WRITEUP
author: ywdh
tags: ["ctf"]
date: 2024-08-10
---

# LITCTF 2024 QUALS WRITEUP

[TOC]

# Misc

## Welcome

### Description

<details>
Please join the Discord for the latest announcements and read the [contest rules](https://lit.lhsmathcs.org/logistics)! Good luck!
</details>

### Solve Walkthrough

- Flag 1 in the announcement channel: `LITCTF{we_4re_happy_1it20`.

![welcome-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/litctf_misc1-01.png)

- Flag 2 at the bottom of [contest rules](https://lit.lhsmathcs.org/logistics) page: `24_is_h4pp3n1ng_and_h0p3_u_r_2}`.

![welcome-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/litctf_misc1-02.png)

### Flag

==LITCTF{we_4re_happy_1it2024_is_h4pp3n1ng_and_h0p3_u_r_2}==

# Web

## anti-inspect

### Description

<details>
can you find the answer? **WARNING: do not open the link your computer will not enjoy it much**. URL: http://litctf.org:31779/
<br><br>
Hint: If your flag does not work, think about how to style the output of console.log
</details>

### Solve Walkthrough

- As you can see at the description, if we open the link,your browser will crashed.
- So, I try another way to open the page, yap.. by using **cURL**.
- Simply, cURL is web browser, but in your terminal. We can perform HTTP request to the target server of course with command-line :).
- Here's the output while i'm trying to request with cURL.

```bash
curl http://litctf.org:31779/
```

![anti inspect-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/litctf_web1-01.png)

- After I see the response, makes sense that we are told not to open the link through a web browser, it does **infinite loop** (look at the _while true_ statement).
- Alright, just ignore it. In the response show me the flag, but, if you read the challenge description carefully, you will notice that the flag is gone wrong !
- So, to fix that, I try to save the response into a HTML file and open it in my web browser to see the correct formatted flag.
- And yeah, we got the flag (just copy the output from your console).

![anti inspect-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/litctf_web1-02.png)

### Flag

==LITCTF{your_fOund_teh_fI@g_94932}==

## jwt-1

### Description

<details>
I just made a website. Since cookies seem to be a thing of the old days, I updated my authentication! With these modern web technologies, I will never have to deal with sessions again. Come try it out at http://litctf.org:31781/.
</details>

### Solve Walkthrough

- As you can know from the challenge title, it shoule be correlation with JWT token, so prepare for https://jwt.io website.
- The website have 3 features/endpoints, that is:
    - Sign up -> **/signup/**
    - Log in -> **/login/**
    - Get the flag -> **/flag**
- Of course we don't know admin password, but after you login (or signup if you don't have an account before), you can see new generated JWT token in the **Storage > Cookies** of your developer tools.
- Copy that JWT token to the jwt.io website, and you will see the payload data contain `name` and `admin`.
- I change the `admin` value to **true**, and then I replace the original JWT token cookies to crafted JWT token payload.

![jwt-1-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/litctf_web2-01.png)

- When I try to visit the **/flag** endpoint, it show me the flag.

![jwt-1-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/litctf_web2-02.png)

### Flag

==LITCTF{o0ps_forg0r_To_v3rify_1re4DV9}==

## jwt-2

### Description

<details>
its like jwt-1 but this one is harder URL: http://litctf.org:31777/
</details>

### Solve Walkthrough

- Given a TypeScript source code like below:

```typescript
import express from "express";
import cookieParser from "cookie-parser";
import path from "path";
import fs from "fs";
import crypto from "crypto";

const accounts: [string, string][] = [];

const jwtSecret = "xook";
const jwtHeader = Buffer.from(
  JSON.stringify({ alg: "HS256", typ: "JWT" }),
  "utf-8"
)
  .toString("base64")
  .replace(/=/g, "");

const sign = (payload: object) => {
  const jwtPayload = Buffer.from(JSON.stringify(payload), "utf-8")
    .toString("base64")
    .replace(/=/g, "");
    const signature = crypto.createHmac('sha256', jwtSecret).update(jwtHeader + '.' + jwtPayload).digest('base64').replace(/=/g, '');
  return jwtHeader + "." + jwtPayload + "." + signature;

}

const app = express();

const port = process.env.PORT || 3000;

app.listen(port, () =>
  console.log("server up on http://localhost:" + port.toString())
);

app.use(cookieParser());
app.use(express.urlencoded({ extended: true }));

app.use(express.static(path.join(__dirname, "site")));

app.get("/flag", (req, res) => {
  if (!req.cookies.token) {
    console.log('no auth')
    return res.status(403).send("Unauthorized");
  }

  try {
    const token = req.cookies.token;
    // split up token
    const [header, payload, signature] = token.split(".");
    if (!header || !payload || !signature) {
      return res.status(403).send("Unauthorized");
    }
    Buffer.from(header, "base64").toString();
    // decode payload
    const decodedPayload = Buffer.from(payload, "base64").toString();
    // parse payload
    const parsedPayload = JSON.parse(decodedPayload);
                // verify signature
                const expectedSignature = crypto.createHmac('sha256', jwtSecret).update(header + '.' + payload).digest('base64').replace(/=/g, '');
                if (signature !== expectedSignature) {
                        return res.status(403).send('Unauthorized ;)');
                }
    // check if user is admin
    if (parsedPayload.admin || !("name" in parsedPayload)) {
      return res.send(
        fs.readFileSync(path.join(__dirname, "flag.txt"), "utf-8")
      );
    } else {
      return res.status(403).send("Unauthorized");
    }
  } catch {
    return res.status(403).send("Unauthorized");
  }
});

app.post("/login", (req, res) => {
  try {
    const { username, password } = req.body;
    if (!username || !password) {
      return res.status(400).send("Bad Request");
    }
    if (
      accounts.find(
        (account) => account[0] === username && account[1] === password
      )
    ) {
      const token = sign({ name: username, admin: false });
      res.cookie("token", token);
      return res.redirect("/");
    } else {
      return res.status(403).send("Account not found");
    }
  } catch {
    return res.status(400).send("Bad Request");
  }
});


app.post('/signup', (req, res) => {
  try {
    const { username, password } = req.body;
    if (!username || !password) {
      return res.status(400).send('Bad Request');
    }
    if (accounts.find(account => account[0] === username)) {
      return res.status(400).send('Bad Request');
    }
    accounts.push([username, password]);
    const token = sign({ name: username, admin: false });
    res.cookie('token', token);
    return res.redirect('/');
  } catch {
    return res.status(400).send('Bad Request');
  }
});
```

- From the source code above, we can see that the secret key is weak (not properly random): `xook`.
- But, if we do the samething like `jwt-1` like previous, we got a response **Unauthorized ;)**.
- Then, I craft this JWT payload (with user **"name": "joni"** and **"admin": true**).

![jwt-2-01](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/litctf_web3-01.png)

- Finally, I replace the original JWT token cookies to crafted JWT token payload with `xook` as secret key.
- And if I visit the **/flag** endpoint, we can see the flag.

![jwt-2-02](https://raw.githubusercontent.com/whyuhurtz/hackmd-blog/main/2024/_assets/litctf_web3-02.png)

### Flag

==LITCTF{v3rifyed_thI3_Tlme_1re4DV9}==
