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

![Initial HTTPS Page](../assets/images/El-Bandito-https-recon.png)

Since minimal web applications often hide functionality within client-side resources, the page source was inspected for additional clues.

---

## Source Code Analysis

Reviewing the HTML source code revealed a JavaScript reference pointing to:

```text
static/messages.js
```

This suggested that additional functionality or hidden logic might exist within the application's static resources.

### Page Source Inspection

![Page Source](../assets/images/El-Bandito-https-source.png)

The discovery of the JavaScript file indicated that further web enumeration was required to uncover hidden endpoints and application functionality.

## Directory Enumeration

To identify hidden endpoints and exposed functionality, directory enumeration was performed against the HTTPS service using Gobuster.

```bash id="jlwm8v"
gobuster dir -u https://<MACHINE_IP>:80/ \
-w <WORDLIST_PATH> \
-x txt,js,php,html -k
```

### Output

```text id="jlwm1t"
/static               (Status: 301)
/login                (Status: 405)
/access               (Status: 200)
/ping                 (Status: 200)
/messages             (Status: 302)
/save                 (Status: 405)
/logout               (Status: 302)
/flush                (Status: 200)
```

### Analysis

The enumeration process revealed several interesting endpoints exposed by the application.

Notable findings included:

| Endpoint    | Observation                                    |
| ----------- | ---------------------------------------------- |
| `/access`   | Accessible sign-in page                        |
| `/login`    | Login-related functionality returning HTTP 405 |
| `/messages` | Redirected back to root                        |
| `/flush`    | Accessible endpoint returning HTTP 200         |
| `/ping`     | Active endpoint responding successfully        |

The `/access` endpoint exposed a sign-in interface, indicating that the application implemented an authentication mechanism that could potentially be targeted during further testing.

Multiple endpoints also returned successful responses (`HTTP 200 OK`), suggesting additional application functionality that required deeper investigation.

### Sign-In Page

![Sign-In Page](../assets/images/El-Bandito-https-login.png)

At this stage, the focus shifted toward analyzing the authentication workflow and interacting with the discovered endpoints to identify potential vulnerabilities or logic flaws.

