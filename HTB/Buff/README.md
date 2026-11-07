
# Hack The Box — Buff

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![OS](https://img.shields.io/badge/OS-Windows-blue)
![Status](https://img.shields.io/badge/Status-Completed-success)

## Executive Summary

Buff is an Easy-rated Windows machine on Hack The Box that demonstrates how insecure file upload functionality can lead to remote code execution (RCE) and how vulnerable locally hosted software can facilitate privilege escalation.

The initial foothold was obtained by exploiting an unauthenticated file upload vulnerability in Gym Management Software 1.0. Following local enumeration, a vulnerable CloudMe application was identified. Chisel reverse port forwarding was used to access the localhost-only service, and a stack-based buffer overflow was leveraged to obtain an elevated shell.

**Key Techniques:**
- Web application enumeration
- Unrestricted file upload
- Remote code execution
- Windows enumeration
- Reverse port forwarding with Chisel
- Stack-based buffer overflow
- Local privilege escalation

---

## 1. Reconnaissance and Enumeration

### Nmap

I began by enumerating the target using Nmap to identify available services.

```bash
nmap -sC -sV -sC -O --reason <TARGET_IP>
```

The scan identified an Apache web server running on port 8080.

<img width="1351" height="382" alt="image" src="https://github.com/user-attachments/assets/4bd745b4-5fb5-492b-9c2c-b84d99ba858a" />


Further inspection of the HTTP response headers revealed:

| Technology | Version |
|---|---|
| Apache | 2.4.43 (Win64) |
| PHP | 7.4.6 |
| OpenSSL | 1.1.1g |
| Operating System | Windows |

### Web Enumeration

Using Curl, I examined the HTTP response:

```bash
curl -i http://<TARGET_IP>:8080/
```
<img width="642" height="237" alt="image" src="https://github.com/user-attachments/assets/81d08852-d0b7-40ab-bce2-e0823fb9bfcb" />

Initially, I tried to connect without specifying port 8080. This showed the importance of explicitly specifying nonstandard service ports during enumeration.

I then manually navigated through the application's available pages.

The **Contact** page identified the underlying application as:

**Gym Management Software 1.0**

<img width="1557" height="306" alt="image" src="https://github.com/user-attachments/assets/169f2349-1458-4bc4-ab19-5e9d3dc307ae" />

This was an important because I had spent a long time in other pages and source code, all while trying to leverage curl in different ways. Just to finally see that manually looking at every tab on the site is a great core concept.

### Vulnerability Research

Using SearchSploit:

```bash
searchsploit "Gym Management System"
```

Several vulnerabilities were identified, including SQL injection, authentication bypass, and unauthenticated remote code execution.

The most promising result was:

**Gym Management System 1.0 — Unauthenticated Remote Code Execution**

Exploit-DB: **48506**

```bash
searchsploit -x 48506
```

After reviewing the proof of concept, I identified an insecure file upload mechanism within `/upload.php`.

<img width="938" height="465" alt="image" src="https://github.com/user-attachments/assets/ca52a09a-5545-47ac-b462-22f75564a7f8" />

---

## 2. Initial Access — PHP Remote Code Execution

### Vulnerability Analysis

The application contained inconsistent file extension validation.

The upload handler validated the final extension of an uploaded filename:

```php
$extension = @end(explode(".", $_FILES["file"]["name"]));
```

However, when saving the file, the application selected the second component of the filename:

```php
$pic = $_FILES["file"]["name"];
$conv = explode(".", $pic);
$ext = $conv['1'];
```

This created a double-extension vulnerability.

For example:

```text
test.php.png
```

The application validated the final `.png` extension but saved the uploaded content with a `.php` extension.

Additionally, the upload endpoint did not properly enforce authentication.

### Manual Exploitation with Burp Suite

Rather than immediately executing the publicly available exploit, I manually investigated the vulnerability using Burp Suite.

I configured FoxyProxy to route Firefox traffic through Burp Suite and intercepted requests to the vulnerable endpoint.

```http
POST /upload.php?id=test HTTP/1.1
Host: <TARGET_IP>:8080
Content-Type: multipart/form-data
```

The upload handler used the `id` parameter to construct the destination filename.

I initially uploaded a harmless test file and verified that the application saved it successfully.

```bash
curl -i http://<TARGET_IP>:8080/upload/test.png
```

The server returned the contents of the uploaded file, confirming that unauthenticated uploads were possible.

Next, I tested the double-extension vulnerability using a PHP file containing a benign output marker.

The application saved the file as executable PHP, and requesting it confirmed that the server processed the PHP code rather than returning it as plain text.

### Command Execution

After confirming PHP execution, I tested operating-system command execution through PHP's `shell_exec()` function.

```php
<?php echo shell_exec('whoami/prive'); ?>
```

<img width="692" height="365" alt="image" src="https://github.com/user-attachments/assets/fc49a3a9-4b7a-48f4-a538-baed4610a804" />

The server returned the Windows account associated with the executing process.

This confirmed that the vulnerability could be leveraged for operating-system command execution.

<img width="1227" height="362" alt="image" src="https://github.com/user-attachments/assets/249e7886-6aeb-405a-8df1-965db93984b8" />

### Establishing a Reverse Shell

I subsequently used a Windows-compatible PowerShell reverse shell to establish a more interactive command channel.

A Netcat listener was configured on Kali:

```bash
nc -lvnp 4444
```

The PowerShell script was hosted using Python's HTTP server.

```bash
python3 -m http.server 8000
```

An initial attempt failed because the HTTP server was serving the wrong directory, resulting in an HTTP 404 response when the target requested the script.

After correcting the working directory, the target successfully retrieved the script and established a reverse connection.

The initial foothold was obtained with standard user privileges.

---

## 3. Windows Enumeration

Following initial access, I began enumerating the Windows environment.

### User Directory Enumeration

I used PowerShell to recursively enumerate files within the accessible user profile:

```powershell
Get-ChildItem $env:USERPROFILE -Recurse -File -ErrorAction SilentlyContinue | Select-Object FullName
```

This revealed an interesting executable within the Downloads directory:

```text
CloudMe_1112.exe
```

The filename suggested that CloudMe version 1.11.2 was present on the system.

### Process Enumeration

I investigated whether CloudMe was currently running.

```powershell
Get-Process CloudMe -ErrorAction SilentlyContinue
```

The application was running.

Next, I enumerated local listening ports:

```powershell
netstat -ano
```

An interesting TCP listener was identified:

```text
127.0.0.1:8888
```

To correlate the listening port with its owning process, I used:

```powershell
Get-NetTCPConnection -LocalPort 8888 | ForEach-Object { Get-Process -Id $_.OwningProcess }
```

This confirmed that the listener belonged to CloudMe.

Because the service was bound to `127.0.0.1`, it was accessible only from the Windows host itself.

This presented an opportunity to investigate the application for local privilege escalation.

<img width="978" height="287" alt="image" src="https://github.com/user-attachments/assets/a3c34793-9b64-48d0-a3fe-d3657fb10b87" />

---

## 4. Privilege Escalation — CloudMe

### Vulnerability Discovery

I searched for known vulnerabilities affecting CloudMe.

```bash
searchsploit "CloudMe 1.11.2"
```

The results identified a stack-based buffer overflow vulnerability.

**CloudMe 1.11.2 — Buffer Overflow**

Exploit-DB: **48389**

<img width="967" height="331" alt="image" src="https://github.com/user-attachments/assets/062e2bdf-f3b9-437b-a918-fa54ac13a8b0" />


I reviewed the Python proof of concept, which contained several important components:

- Buffer padding
- Instruction pointer (EIP) overwrite
- NOP sled
- Executable shellcode
- Remaining buffer padding

The original proof of concept used a demonstration payload that launched Windows Calculator. Here I failed to utilize -v payload which caused me to have to spend more time troubleshooting and correct the payload of my msfvenom generated reverse shell.

For the environment, the payload was adapted to establish a reverse TCP shell.

<img width="981" height="562" alt="image" src="https://github.com/user-attachments/assets/5ce36933-cca2-47c4-8d54-25ff9fd73e82" />


### Reverse Port Forwarding with Chisel

Because CloudMe was listening only on localhost, the vulnerable service could not be accessed directly from Kali.

For this I tried using a new to me tool Chisel to establish a reverse TCP tunnel.

Chisel was installed on Kali, and the Windows-compatible executable was transferred to the target using PowerShell.

The Chisel server was started on Kali:

```bash
chisel server --reverse --port 9999
```

<img width="960" height="221" alt="image" src="https://github.com/user-attachments/assets/049e9f6b-570e-40f2-80fa-8edcbcea7398" />

The Windows Chisel client connected back to Kali and established a reverse forwarding rule for the CloudMe service.

The resulting network path was:

```text
Kali
127.0.0.1:8888
       |
       v
Chisel Reverse Tunnel
       |
       v
Windows Target
127.0.0.1:8888
       |
       v
CloudMe.exe
```

I verified that the forwarded port was reachable:

```bash
nc -vz 127.0.0.1 8888
```

The connection succeeded.

### Buffer Overflow Exploitation

After establishing the tunnel, I prepared the CloudMe Python proof of concept for testing.

A Windows x86 reverse TCP payload was generated using msfvenom, and the existing demonstration shellcode was replaced.

During modification as I mentioned earlier, I dealt with a Python variable naming issue.

The original exploit referenced a variable named `payload`, while the generated shellcode initially used `buf`.

This mismatch required correction to preserve the script's intended buffer construction.

<img width="645" height="468" alt="image" src="https://github.com/user-attachments/assets/9bb3a465-77b5-42e6-a257-df971e2e1bd7" />

I validated the Python syntax using:

```bash
python3 -m py_compile 48389.py
```

### Troubleshooting

The initial exploitation attempts did not establish a reverse shell.

Investigation of the Chisel logs revealed intermittent connection failures:

```text
dial tcp 127.0.0.1:8888:
No connection could be made because the target machine actively refused it.
```

Looking more into CloudMe I found the process ID changed repeatedly and that the listener on port 8888 was intermittently unavailable.

This made me believe that the destination service was not consistently accepting connections, despite the Chisel tunnel itself remaining established.

After verifying that CloudMe was running and listening, I repeated the test.

The exploit successfully established a reverse TCP connection to Kali.

### Privilege Verification

Once connected, I verified the security context of the new shell:

```cmd
whoami
```

The response confirmed that I was now the local Administrator account.

<img width="630" height="192" alt="image" src="https://github.com/user-attachments/assets/ea7048d6-49fa-41f4-b75b-604afd6838f8" />

---

## 5. Lessons Learned

### Manual Web Enumeration Is Essential

One of the most valuable lessons from Buff was the importance of manually reviewing every accessible webpage.

The exact application name and version were displayed directly on the Contactt page.

Taking time to inspect the website's content can reveal critical information that may not be apparent from automated scans or HTTP headers.

### Understand the Vulnerability Before Exploitation

Manually investigating the insecure upload handler helped me with a better understanding of why the vulnerability existed.

### Validate Each Exploitation Stage

Testing file uploads, PHP execution, and operating-system command execution independently made troubleshooting much easier.

### Understand Localhost-Only Services

CloudMe demonstrated how a vulnerable service can be inaccessible from an external interface while remaining reachable locally.

Chisel provided a practical introduction to reverse port forwarding and TCP tunneling.

### Process IDs Are Not Static

CloudMe's process ID changed repeatedly throughout enumeration.

Correlating process and network information dynamically was more reliable than comparing PIDs collected at different times.

### A Working Tunnel Does Not Guarantee a Working Service

Although Chisel was connected, the destination service intermittently refused connections.

Reviewing the tunnel logs helped distinguish network forwarding problems from application availability problems.

### Troubleshooting Is Part of Exploitation

Several issues arose during the machine, including:

- Incorrect HTTP port usage during initial enumeration
- HTTP 404 errors caused by serving files from the wrong directory
- Python variable naming inconsistencies
- Intermittent CloudMe listener availability
- Differences between successful Python compilation and runtime behavior

Each issue reinforced the importance of understanding the tools and protocols involved rather than blindly following an exploitation sequence.

---

## 6. References

- [Exploit-DB 48506 — Gym Management System Unauthenticated RCE](https://www.exploit-db.com/exploits/48506)
- [Exploit-DB 48389 — CloudMe Buffer Overflow](https://www.exploit-db.com/exploits/48389)
- [Chisel — GitHub](https://github.com/jpillora/chisel)
- [Hack The Box](https://www.hackthebox.com/)

---
