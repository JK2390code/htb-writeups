# Wifinetic | Hack The Box

> Retired Hack The Box machine focused on wireless infrastructure, credential reuse, configuration disclosure, and WPS abuse.

---

# Overview

Wifinetic focused heavily on wireless infrastructure, credential management, and configuration exposure. 


---

# Initial Enumeration

## Nmap Service Discovery

Initial enumeration was performed using Nmap to identify exposed services, versions, and default scripts.

```bash
nmap -sS -sV -sC -O <target-ip>
```
<img width="1013" height="718" alt="image" src="https://github.com/user-attachments/assets/58b884b9-85a2-4f64-9b09-ac909c3b46dc" />

The scan identified several interesting services:

| Port | Service | Notes |
|---|---|---|
| 21 | FTP | Anonymous authentication enabled |
| 22 | SSH | OpenSSH service exposed |
| 53 | DNS | Internal DNS-related functionality |

The FTP service immediately stood out due to anonymous access being permitted.

---

# FTP Enumeration

## Anonymous FTP Access

Authentication to the FTP service was successful using the standard anonymous account convention.

```bash
ftp <target-ip>
```

```text
Username: anonymous
```
<img width="1160" height="329" alt="image" src="https://github.com/user-attachments/assets/537491cf-5927-42f6-adc6-a0c5d57bb9fe" />


Once authenticated, several files and backup archives were exposed:

- `MigrateOpenWrt.txt`
- `ProjectGreatMigration.pdf`
- `ProjectOpenWRT.pdf`
- `backup_OpenWrt-2023-07-26.tar`
- `employees_wellness.pdf`

The backup archive was extremely interesting because infrastructure backups may potentially contain:
- credentials,
- configuration files,
- network information,
- or operational secrets.

All available files were downloaded locally for analysis.

<img width="1114" height="699" alt="image" src="https://github.com/user-attachments/assets/42c48e27-ef65-43c5-a5a5-504a327790ad" />

---

# Backup Analysis

## OpenWRT Configuration Review

The backup archive was extracted and enumerated.

```bash
tar -xvf backup_OpenWrt-2023-07-26.tar
```
<img width="1045" height="706" alt="image" src="https://github.com/user-attachments/assets/1d09f506-2d4a-47ca-b967-28cffcc1678a" />

Wireless configuration files were identified within the extracted OpenWRT configuration.

Reviewing the wireless configuration revealed plaintext WPA credentials:

```text
option ssid 'OpenWrt'
option key 'VeRyUniqWiFiPasswrd1!'
```
<img width="964" height="648" alt="image" src="https://github.com/user-attachments/assets/30d38a08-4dae-4ee5-a08e-1aaef03dc057" />


This demonstrated a common operational practice where sensitive credentials are stored in plaintext within improperly exposed backups.

---

# Initial Access

## Credential Reuse via SSH

Enumeration of `/etc/passwd` identified a likely human-managed account:

```text
netadmin
```
<img width="587" height="260" alt="image" src="https://github.com/user-attachments/assets/4bc94b21-45e3-4a0f-9e36-a2bb0fe9b3b7" />

The previously recovered wireless credential was tested against SSH authentication for the `netadmin` account.

```bash
ssh netadmin@<target-ip>
```
<img width="810" height="723" alt="image" src="https://github.com/user-attachments/assets/5ea19607-892d-4cbb-adac-9a071bb1edd3" />

Authentication was successful due to Netadmin credential reuse.

Initial foothold on the target system was obtained.

---

# Internal Enumeration

## Wireless Interface Analysis

Once authenticated, local enumeration focused on understanding the system’s wireless infrastructure and operational role.

```bash
iwconfig
```
<img width="798" height="536" alt="image" src="https://github.com/user-attachments/assets/991cf5e9-57b7-477b-9a4a-05fd66b6c968" />

Several wireless interfaces were identified:

| Interface | Purpose |
|---|---|
| `wlan0` | Access Point (Master Mode) |
| `wlan1` | Connected Wireless Client |
| `mon0` | Monitor Mode Interface |
| `wlan2` | Additional Wireless Interface |

The presence of:
- monitor mode interfaces,
- `hostapd`,
- and wireless management tooling


<img width="1079" height="278" alt="image" src="https://github.com/user-attachments/assets/20963371-123e-45ce-87b3-ee1933f28cd3" />

---

# Wireless Enumeration & WPS Attack

## Identifying the Access Point

The wireless configuration revealed:
- ESSID: `OpenWrt`
- BSSID: `02:00:00:00:00:00`
- Channel: `1`

Although automated discovery tooling was inconsistent in this environment, sufficient information was gathered manually through native wireless utilities.

## WPS PIN Attack with Reaver

Using the monitor interface (`mon0`), a WPS attack was performed against the target access point using `reaver`.

```bash
reaver -i mon0 -b 02:00:00:00:00:00 -c 1 -vv
```
<img width="891" height="631" alt="image" src="https://github.com/user-attachments/assets/729ab7f9-fbb1-40a5-905f-09feff757d90" />

The attack successfully recovered:
- the WPS PIN,
- and the WPA PSK.

```text
WPA PSK: WhatIsRealAnDWhAtIsNot51121!
```
<img width="528" height="171" alt="image" src="https://github.com/user-attachments/assets/8ced62f6-957b-43f8-941d-afb1984ec239" />

This demonstrates how WPS can significantly weaken otherwise strong WPA credentials.

---

# Privilege Escalation

## Root Credential Reuse

The recovered WPA PSK was tested against the local root account using `su`.

```bash
su root
```
<img width="590" height="133" alt="image" src="https://github.com/user-attachments/assets/4e960c1b-23b0-4123-8572-274e583b623a" />

Authentication succeeded due to password reuse, resulting in full system compromise.

---

# Lessons Learned

Wifinetic displayus several realistic security weaknesses:

- Anonymous FTP exposure
- Sensitive backup disclosure
- Plaintext credential storage
- Password reuse across services
- Weak wireless security posture through WPS
- Reuse of wireless credentials for privileged local accounts


# Skills Practiced

- Network and service enumeration
- FTP enumeration
- Configuration and backup analysis
- Linux account enumeration
- Credential reuse testing
- Wireless interface analysis
- Monitor mode reconnaissance
- WPS attacks with `reaver`
- Privilege escalation through credential reuse
