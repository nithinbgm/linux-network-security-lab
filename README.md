# Linux Network Security Lab --- Kali Linux & Ubuntu

> **Hands-on Cybersecurity Portfolio Project**\
> Built in VMware Workstation Pro to practice network reconnaissance,
> SSH security, packet analysis, Linux service enumeration, and
> authentication-log investigation.

------------------------------------------------------------------------

## 📌 Project Overview

This project documents an isolated cybersecurity lab containing:

-   **Kali Linux** --- security testing / analysis workstation
-   **Ubuntu Server** --- Linux server and target
-   **VMware Workstation Pro** --- virtualization platform
-   **Host-only networking** --- isolated communication between the lab
    VMs

The objective was to move beyond theoretical networking concepts and
observe how real services, packets, processes, and security logs relate
to one another.

### Lab Architecture

``` text
                         VMware Workstation Pro
                                  |
                           Host-only Network
                                  |
                    192.168.160.0/24
                         /              \
                        /                \
                       ▼                  ▼
             Kali Linux VM          Ubuntu Server VM
             192.168.160.128        192.168.160.130
             Security workstation   Target server
                       |                  |
                       |---- TCP/22 ----->|
                       |       SSH        |
                       |<--- responses ---|
```

------------------------------------------------------------------------

## 🎯 Objectives

1.  Configure an isolated virtual security lab.
2.  Verify Layer 3 connectivity using ICMP.
3.  Perform TCP port discovery using Nmap.
4.  Identify running services and versions.
5.  Establish an SSH connection from Kali to Ubuntu.
6.  Capture and analyze SSH traffic using Wireshark.
7.  Understand the TCP three-way handshake.
8.  Analyze SSH key exchange and negotiated cryptography.
9.  Inspect Linux listening sockets and associated processes.
10. Review SSH authentication events in Linux logs.
11. Build a foundation for future SIEM, firewall, cloud-security, and
    security-automation labs.

------------------------------------------------------------------------

## 🛠️ Technologies & Tools

  Category                 Technology
  ------------------------ -------------------------------
  Virtualization           VMware Workstation Pro
  Attacker / Security VM   Kali Linux
  Server / Target VM       Ubuntu Server
  Network                  VMware Host-only
  Reconnaissance           Nmap
  Packet Analysis          Wireshark
  Remote Administration    OpenSSH
  Linux Administration     systemctl, ss, ps, journalctl
  Networking               TCP/IP, ICMP, IPv4

------------------------------------------------------------------------

# 1. Lab Environment

## Network Configuration

  System                    IP Address Role
  --------------- -------------------- ----------------------
  Kali Linux         `192.168.160.128` Security workstation
  Ubuntu Server      `192.168.160.130` Target server
  Network           `192.168.160.0/24` Isolated lab network

The two VMs were configured on the same VMware Host-only network.

------------------------------------------------------------------------

# 2. Connectivity Verification

From Kali:

``` bash
ping -c 4 192.168.160.130
```

From Ubuntu:

``` bash
ping -c 4 192.168.160.128
```

Successful replies confirmed two-way network connectivity between the
VMs.

Example result:

``` text
4 packets transmitted, 4 received, 0% packet loss
```

### Key concept

ICMP testing verifies basic Layer 3 reachability, but it does not prove
that an application service is available. That is why the next step was
TCP service discovery.

------------------------------------------------------------------------

# 3. Nmap Reconnaissance

## Basic TCP Scan

From Kali:

``` bash
nmap 192.168.160.130
```

### Result

``` text
PORT   STATE SERVICE
22/tcp open  ssh
```

Nmap identified:

-   Host: `192.168.160.130`
-   Host status: Up
-   TCP port `22`: Open
-   Service: SSH
-   Other default scanned TCP ports: Closed

### Security observation

The Ubuntu server exposed SSH over TCP/22. This became the primary
service for further investigation.

------------------------------------------------------------------------

# 4. Service & Version Detection

Command:

``` bash
nmap -sV 192.168.160.130
```

### Result

``` text
22/tcp open ssh OpenSSH 10.2p1 Ubuntu 2ubuntu3.5
```

This demonstrated the difference between:

-   **Port discovery** --- finding that port 22 is open.
-   **Service detection** --- identifying SSH.
-   **Version detection** --- identifying the OpenSSH version.

### Security lesson

An open port is not automatically a vulnerability. A security assessment
should identify the service, understand its configuration, determine
exposure, and then assess risk.

------------------------------------------------------------------------

# 5. Full TCP Port Scan

Command:

``` bash
nmap -p- 192.168.160.130
```

### Result

``` text
Not shown: 65534 closed tcp ports
22/tcp open ssh
```

This scanned all TCP ports from `1` through `65535`.

### Finding

Only TCP/22 was discovered as open during this scan.

This demonstrated why a security engineer may use both:

``` bash
nmap 192.168.160.130
```

and:

``` bash
nmap -p- 192.168.160.130
```

The first focuses on Nmap's default common ports; the second checks the
entire TCP port range.

------------------------------------------------------------------------

# 6. SSH Connection

From Kali:

``` bash
ssh nithin@192.168.160.130
```

The SSH connection was successfully established.

Network flow:

``` text
Kali 192.168.160.128
        |
        | TCP destination port 22
        ▼
Ubuntu 192.168.160.130
```

SSH provided encrypted remote administration between the two VMs.

------------------------------------------------------------------------

# 7. Wireshark Packet Analysis

A Wireshark capture was performed on Kali while establishing an SSH
connection.

Display filter:

``` text
tcp.port == 22
```

The capture showed the TCP connection beginning with:

``` text
SYN
SYN, ACK
ACK
```

### TCP Three-Way Handshake

``` text
Kali                         Ubuntu
 |                             |
 | -------- SYN -------------> |
 | <------ SYN + ACK --------- |
 | -------- ACK -------------> |
 |                             |
 |     TCP connection ready    |
```

This provided practical confirmation of the TCP three-way handshake.

------------------------------------------------------------------------

## SSH Negotiation

The packet capture also showed SSH protocol negotiation and key-exchange
activity, including:

``` text
Client: SSH-2.0-OpenSSH_9.9p1
Server: SSH-2.0-OpenSSH_10.2p1
```

The traffic later transitioned to encrypted SSH packets.

### Security observation

After encryption was established, the contents of the SSH session were
not visible as plaintext in the packet capture.

------------------------------------------------------------------------

# 8. SSH Server Configuration Assessment

On Ubuntu:

``` bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|maxauthtries|x11forwarding)'
```

### Observed configuration

``` text
port 22
maxauthtries 6
permitrootlogin prohibit-password
pubkeyauthentication yes
passwordauthentication yes
x11forwarding yes
```

### Security assessment

  -----------------------------------------------------------------------
  Setting                 Observed                Assessment
  ----------------------- ----------------------- -----------------------
  SSH port                `22`                    Standard SSH port

  Root login              `prohibit-password`     Password-based root SSH
                                                  login prevented

  Public-key              `yes`                   Supported
  authentication                                  

  Password authentication `yes`                   Enabled

  Max authentication      `6`                     Configured
  attempts                                        

  X11 forwarding          `yes`                   Enabled
  -----------------------------------------------------------------------

This was treated as a **baseline assessment**, not an instruction to
blindly change production settings.

------------------------------------------------------------------------

# 9. SSH Cryptographic Configuration

The effective SSH configuration was inspected with:

``` bash
sudo sshd -T | grep -E '^(ciphers|macs|kexalgorithms|hostkeyalgorithms)'
```

The server supported modern cryptographic options including:

-   ChaCha20-Poly1305
-   AES-GCM
-   AES-CTR
-   Curve25519
-   Diffie-Hellman groups
-   Ed25519 host keys
-   SHA-2 based algorithms

The exercise demonstrated the difference between:

> **Algorithms configured/allowed by the server**

and:

> **Algorithms actually negotiated for one SSH connection**

------------------------------------------------------------------------

# 10. Actual SSH Negotiation

From Kali:

``` bash
ssh -vv nithin@192.168.160.130
```

`-vv` enables detailed SSH debugging output.

The actual connection negotiated:

``` text
Key exchange:
sntrup761x25519-sha512

Host key:
ssh-ed25519

Cipher:
chacha20-poly1305@openssh.com

Authentication:
password
```

### What this means

``` text
SSH Connection
      |
      +-- Key Exchange
      |     └── sntrup761x25519-sha512
      |
      +-- Server Identity
      |     └── ssh-ed25519
      |
      +-- Encryption
      |     └── chacha20-poly1305@openssh.com
      |
      +-- User Authentication
            └── password
```

This demonstrated that SSH uses separate concepts for:

-   Key exchange
-   Server identity / host keys
-   Encryption
-   User authentication

------------------------------------------------------------------------

# 11. Linux Listening-Socket Investigation

On Ubuntu:

``` bash
ss -tulnp
```

With elevated privileges:

``` bash
sudo ss -tulnp
```

The investigation showed SSH listening on:

``` text
0.0.0.0:22
[::]:22
```

and associated it with:

``` text
sshd
PID 2610
```

### Investigation chain

``` text
TCP Port 22
     ↓
Listening socket
     ↓
sshd
     ↓
PID 2610
     ↓
ssh.service
```

This connected external reconnaissance with internal Linux process
information.

------------------------------------------------------------------------

# 12. Process & Service Investigation

Process inspection:

``` bash
ps -p 2610 -f
```

The SSH process was identified as:

``` text
/usr/sbin/sshd -D [listener]
```

Service inspection:

``` bash
sudo systemctl status ssh --no-pager
```

The SSH service was confirmed as:

``` text
Active: active (running)
```

### Key Linux concepts practiced

-   Process ID (PID)
-   systemd services
-   Listening sockets
-   Network ports
-   Processes owning sockets

------------------------------------------------------------------------

# 13. SSH Authentication Log Investigation

SSH-related events were searched in the system journal:

``` bash
sudo journalctl --since "5 minutes ago" --no-pager | grep -Ei 'sshd|ssh'
```

The lab generated successful authentication events such as:

``` text
Accepted password for nithin from 192.168.160.128
```

and:

``` text
pam_unix(sshd:session): session opened for user nithin
```

### Security interpretation

``` text
Source IP:
192.168.160.128

Username:
nithin

Service:
SSH

Authentication:
Password

Result:
Successful
```

This demonstrates how a Linux server records authentication activity
that could later be collected by a SIEM such as Splunk or Microsoft
Sentinel.

------------------------------------------------------------------------

# 14. Security Investigation Workflow

The complete workflow practiced in this lab was:

``` text
                DISCOVER
                   ↓
             Ping / Nmap
                   ↓
                IDENTIFY
                   ↓
          Ports + Services
                   ↓
              ENUMERATE
                   ↓
          Version Detection
                   ↓
               ANALYZE
                   ↓
     Wireshark + SSH Negotiation
                   ↓
              INVESTIGATE
                   ↓
      Sockets + Processes + Logs
                   ↓
                ASSESS
                   ↓
       Configuration & Exposure
                   ↓
              HARDEN / VERIFY
                   ↓
          Future Lab Activities
```

------------------------------------------------------------------------

# 15. Key Skills Demonstrated

### Networking

-   IPv4 addressing
-   `/24` subnetting
-   ICMP
-   TCP
-   TCP three-way handshake
-   TCP ports
-   Client/server communication
-   MAC vs IP addressing

### Linux

-   `ip`
-   `ping`
-   `ss`
-   `ps`
-   `systemctl`
-   `journalctl`
-   Linux processes and services
-   SSH administration

### Security

-   Network reconnaissance
-   Service enumeration
-   Attack-surface identification
-   SSH security assessment
-   Cryptographic negotiation
-   Packet analysis
-   Authentication-log investigation

### Tools

-   Kali Linux
-   Nmap
-   Wireshark
-   OpenSSH
-   VMware Workstation Pro

------------------------------------------------------------------------

# 16. Lessons Learned

1.  An IP address alone does not tell us which services are exposed.
2.  Nmap can identify reachable ports and services.
3.  `-sV` performs service/version detection.
4.  `-p-` scans the complete TCP port range.
5.  `ss` provides the server-side view of listening sockets.
6.  `ps` can associate a service with a process.
7.  `systemctl` provides the service-management view.
8.  `journalctl` provides an important source of security events.
9.  Wireshark makes TCP and SSH behavior visible at packet level.
10. SSH separates authentication from encryption and key exchange.
11. Security assessment should be based on evidence rather than blindly
    changing configuration.

------------------------------------------------------------------------

# 17. Future Improvements

Planned extensions to this lab:

-   SSH key-based authentication
-   SSH hardening
-   UFW firewall configuration
-   Controlled failed-login detection
-   Linux permissions and privilege management
-   DNS traffic analysis
-   HTTP/HTTPS traffic analysis
-   More advanced Nmap enumeration
-   Log-based detection
-   Splunk/Sentinel integration
-   Security automation with Python

------------------------------------------------------------------------

# 📸 Screenshots

Recommended screenshots to add to this repository:

``` text
screenshots/
├── vmware-network.png
├── kali-ip.png
├── ubuntu-ip.png
├── nmap-basic.png
├── nmap-service-detection.png
├── nmap-full-scan.png
├── wireshark-ssh.png
├── ssh-debug.png
├── ss-listening-ports.png
└── ssh-authentication-logs.png
```

Add screenshots only after removing any personal information you do not
want publicly visible.

------------------------------------------------------------------------

# ⚠️ Ethical Scope

All reconnaissance and testing documented here was performed against
virtual machines owned and controlled by the lab operator in an isolated
environment.

Do not scan systems or networks without explicit authorization.

------------------------------------------------------------------------

# 🚀 Career Relevance

This project is part of a broader roadmap toward **Security Engineering
/ Cloud Security / Product Security** roles.

The purpose is to demonstrate practical understanding rather than simply
listing tools on a resume.

Future projects will extend this foundation into:

``` text
Linux & Networking
       ↓
Network Security
       ↓
Cloud Security
       ↓
IAM / Zero Trust
       ↓
SIEM & Detection
       ↓
Python Security Automation
       ↓
Web / API Security
       ↓
DevSecOps
       ↓
Security Engineering
```

------------------------------------------------------------------------

## Author

**Nithin Goud Mamidi**

Cybersecurity Engineer \| Network Security \| Palo Alto \| Zscaler \|
Linux \| Cloud Security
