# Just SSH Walkthrough

Insert the box image here

## Overview
### Platform: Tryhackme
### Box Name: Just SSH
### Description: 

---
## Recon and Enumeration (MITRE ATT&CK Outline)

| Techniqe ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1595](https://attack.mitre.org/techniques/T1595/)                       | Active Scanning               | NMAP                        |
| [T1592](https://attack.mitre.org/techniques/T1592/)                       | Gather Victim Host Information | NMAP                       |
| [T1591](https://attack.mitre.org/techniques/T1591/)                       | Gather Vicitm Org Information | ssh -v |
| [T1589](https://attack.mitre.org/techniques/T1589/)                       | Gather Victim Identify Information | Breached Creds, Personnel Roster |

### Enumeration
Run NMAP to discover open ports:
```bash
sudo nmap $ip -Pn -n
```

Port 22 (SSH) was the only open port, and confirmed with the all ports scan:
```bash
sudo nmap $ip -p- -Pn -n -T5
```
---
---
## Initial Access (MITRE ATT&CK Outline)

| Techniqe ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1078](https://attack.mitre.org/techniques/T1078/)    | Valid Accounts    |  Hydra, SSH  |
| [T1110](https://attack.mitre.org/techniques/T1110/004/)   |  Credential Stuffing   |  Hydra   |
| [T1110](https://attack.mitre.org/techniques/T1110/002/)   | Password Cracking   | ssh2john, john/hashcat  |

### Initial Foothold
Executed a credential stuffing attack utilizing the identifed users, and breached passwords.
```bash
hydra -L users.txt -P breachedCreds.txt $ip ssh
```

SSH Password credentials identified: j_moore : somePassword123
---
---
## Lateral Movement (MITRE ATT&CK Outline)

| Techniqe ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1021](https://attack.mitre.org/techniques/T1021/004/)    | Remote Services    | SSH    |
| [T1552](https://attack.mitre.org/techniques/T1552/)    | Unsecured Credentials    | Manual Enumeration / LinPEAS.sh    |
| [T1110](https://attack.mitre.org/techniques/T1110/003/)    | Password Spraying    | NMAP Scripting Engine    |

### j_moore to sftp
j_moore's profile had exisitng id_rsa private and pub keys. The private key was sprayed
against all the users found in the /home direcotory. With the created user list and
the private key, use nmap to conduct a key spray attack.

```bash
ls /home > allUsers.txt
```

```bash
sudo nmap some long argument
```

The nmap scan identifies an additional account that can be accessed via j_moore's id_rsa, the sftp 
account. Confirm the credentials by logging into the sftp service.
```bash
sftp -i id_rsa sftp@$ip
```
enter the passphrase for j_moore's id_rsa, and check your id.
---
---

## Privilege Escalation (MITRE ATT&CK Outline)

| Techniqe ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1548](https://attack.mitre.org/techniques/T1548/003/)    | Abuse Elevation Control Mechanism    | Sudo Privileges
| [T1556](https://attack.mitre.org/techniques/T1556/)    | Modify Authentication Process    | /etc/ssh/sshd_config
| [T1110](https://attack.mitre.org/techniques/T1110/004/)    | Credential Stuffing    | hydra    |


### sftp to net-admin
With valid credentials we can enumerate the sftp service and sift through the documents for pertinent 
information. There are two particular documents that indicate a likely attack vector. blah blah blah

```bash
cd /uploads/.tempKeys
get *
```

Given the information from the support tickets, b_harris is a good starting point to conduct a 
key spray attack with. First, set the permissions of b_harris id_rsa:

```bash
chmod 600 id_rsa_harris
```

```bash
sudo nmap long command
```

The key-spray provides access to an additional account, the net-admin.

### net-admin to sys-admin

Now that we have compromised the net-admin account, we can begin the recon and enumeration of the
newly acquired account. A common check is to see what sudo privs a user has:

```bash
sudo -l
```

The net-admin has the ability to start, stop, restart, check status, and edit the sshd_config file. 
Given our permissions, lets see what the current configuration of the ssh server is:

```bash
sudo /usr/bin/vim /etc/ssh/sshd_config
```

The match blocks at the bottom do not allow password logins, only id_rsa. With the keys we have already,
we have reached a dead end. However, we still have breached credentials we can test agains the net-admin, and
sys-admin accounts. Lets see change the server settings so we can execute more credential stuffing attacks:

```bash
passwordAuthentication yes
```

Using the breached password list, conduct a credential stuffing attack against the net-admin and sys-admin
accounts.

```bash
hydar -L admins.txt -P breachedPWD.txt $ip ssh
```

Now we have more credentials, and they belong to the sys-admin account. Log into the account and check your id
and privileges.

```bash
id
sudo -l
```

Paydirt! Sys-admin has no password for sudo, which means we have completely compromised the account. Switch the
user to root and collect your flag.

```bash
sudo su
id
```
