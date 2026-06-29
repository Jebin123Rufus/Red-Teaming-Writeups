# Domino

## Room Description

The **NexusCorp Employee Portal** appears to be a typical internal application with authentication controls and role-based access in place. However, multiple small weaknesses, ranging from misconfigurations to logic flaws, can be combined to fully compromise the system.

As an attacker, the objective is to observe how the application behaves, interact with its endpoints, and identify weak trust boundaries. By analysing requests, modifying parameters, and chaining vulnerabilities together, access can be progressively escalated to move deeper into the system.

A single misstep can trigger a chain reaction—exploit each weakness in sequence and watch the system fall, one domino at a time.

---

## Objectives

Throughout this assessment, the following objectives must be achieved:

* Enumerate the target application and identify its exposed functionality.
* Analyse the application's behaviour and discover weaknesses in its trust boundaries.
* Chain multiple vulnerabilities together to progressively escalate privileges.
* Retrieve the flag from the **admin user's profile notes**.
* Gain **administrator access** and retrieve the flag displayed on the **Admin Panel**.
* Achieve **Remote Code Execution (RCE)** on the target server and obtain the flag stored in `/opt/flag3.txt`.
* Pivot to the **devops** user and retrieve the flag from the user's home directory.
* Escalate privileges to **root** and capture the final flag.

## Reconnaissance

### Port Scan

#### Command

```bash
nmap -sC -sV -p- <TARGET_IP>
```

#### Findings

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 22   | SSH     | OpenSSH 9.6p1 |
| 80   | HTTP    | Apache 2.4.58 |

The scan identified **SSH** and an **Apache web server** hosting the **NexusCorp Portal**. With only these two services exposed, the web application became the primary focus for further enumeration.
