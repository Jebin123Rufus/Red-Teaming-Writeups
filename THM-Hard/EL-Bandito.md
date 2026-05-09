# El Bandito — TryHackMe Write-up

## Room Overview

El Bandito, the new identity of the infamous Jack the Exploiter, has entered the Web3 landscape with a large-scale token scam operation. By abusing the decentralized nature of blockchain technologies, he distributed fraudulent tokens to deceive investors and destabilize trust within the DeFi ecosystem.

The objective of this challenge is to investigate the infrastructure used by El Bandito, uncover hidden vulnerabilities, retrieve the required web flags, and ultimately track down his operations.

Difficulty: **Hard**

---

## Objectives

1. Find the first web flag
2. Find the second web flag

---

## Reconnaissance

### Nmap Scan

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


* Understanding attack surface prioritization
* Combining manual testing with automated enumeration
