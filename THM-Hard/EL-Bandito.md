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
PORT     STATE SERVICE VERSION

22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13

631/tcp  open  ipp     CUPS 2.4
- http-title: Forbidden - CUPS v2.4.12

80/tcp   open  ssl/http El Bandito Server
8080/tcp open  http    Nginx

```

## Web Enumeration

After the initial reconnaissance phase, the HTTPS service hosted on the target machine was manually inspected through a web browser.

Browsing to the HTTPS application revealed a minimal page displaying the message:

```text
nothing to see
```

At first glance, the application appeared intentionally minimal and did not expose any obvious functionality.

### Initial HTTPS Page

![Initial HTTPS Page](assets/images/https-homepage.png)

Since minimal web applications often hide functionality within client-side resources, the page source was inspected for additional clues.

---

## Source Code Analysis

Reviewing the HTML source code revealed a JavaScript reference pointing to:

```text
static/messages.js
```

This suggested that additional functionality or hidden logic might exist within the application's static resources.

### Page Source Inspection

![Page Source](assets/images/page-source.png)

The discovery of the JavaScript file indicated that further web enumeration was required to uncover hidden endpoints and application functionality.
