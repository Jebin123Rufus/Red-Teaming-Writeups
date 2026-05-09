# El Bandito — TryHackMe Write-up

## Reconnaissance

Started with an Nmap scan to identify open ports and running services on the target.

```bash
nmap -sC -sV <MACHINE_IP>
```

### Output

```text
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
631/tcp open  ipp     CUPS 2.4

http-title: Forbidden - CUPS v2.4.12
```

### Analysis

Two ports were exposed:

* **22/tcp (SSH)**
  Running OpenSSH 8.2p1. This could potentially provide remote access if valid credentials were discovered later during enumeration.

* **631/tcp (CUPS / IPP)**
  The target was hosting a CUPS printing service. The HTTP response returned:

