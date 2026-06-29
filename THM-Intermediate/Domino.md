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

### Directory Enumeration

#### Command

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -x html,js,php,txt
```

#### Findings

| Endpoint         | Status |
| ---------------- | ------ |
| `/admin/`        | 301    |
| `/support/`      | 301    |
| `/static/`       | 301    |
| `/api/`          | 301    |
| `/backup/`       | 301    |
| `/auth.php`      | 200    |
| `/dashboard.php` | 302    |
| `/team.php`      | 200    |
| `/config.php`    | 200    |
| `/index.php`     | 200    |
| `/reset.php`     | 200    |
| `/logout.php`    | 302    |
| `/javascript/`   | 301    |

The enumeration revealed several interesting endpoints, particularly **`/admin`**, **`/api`**, **`/backup`**, and **`/support`**, which became the primary focus for further investigation.

### User Enumeration

The `team.php` page exposed the names and email addresses of NexusCorp employees, allowing valid usernames to be identified for authentication attempts.

![User Enumeration](../assets/images/Domino-users.png)

### Password Brute Force

Using the enumerated usernames, a password brute-force attack was performed against the login portal.

**Hydra Command**

```bash
hydra -l robert.wilson \
-P /usr/share/wordlists/rockyou.txt \
<TARGET_IP> http-post-form \
"/index.php:username=^USER^&password=^PASS^:Invalid Credentials"
```

The attack successfully recovered the following credentials:

| Username        | Password   |
| --------------- | ---------- |
| `robert.wilson` | `password` |

### Initial Access

The recovered credentials provided access to the NexusCorp Employee Portal as **robert.wilson**. The authenticated dashboard exposed several internal features, including a **My Profile API** endpoint.

![Authenticated Dashboard](../assets/images/Domino-user-dashboard.png)

### Insecure Direct Object Reference (IDOR)

The profile endpoint retrieved user information using a numeric `id` parameter:

```text
/api/users/profile.php?id=<id>
```

By modifying the `id` value from the authenticated user's profile to `1`, it was possible to access the administrator's profile without any authorization checks, confirming an **Insecure Direct Object Reference (IDOR)** vulnerability.

```text
/api/users/profile.php?id=1
```

The response disclosed the administrator's profile along with the first challenge flag stored in the **notes** field.

![Administrator Profile via IDOR](../assets/images/Domino-profile-flag.png)

> **Flag 1:** `THM{1d0r_h0r1z0nt4l_4cc3ss_fl4g1}`
