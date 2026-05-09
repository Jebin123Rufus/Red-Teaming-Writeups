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

## Further Enumeration

Despite identifying multiple endpoints and an exposed sign-in interface, further manual testing and walkthrough analysis of the web application on port 80 did not immediately reveal any exploitable vulnerabilities or useful information.

The discovered functionality appeared intentionally minimal, and the exposed endpoints largely resulted in redirects, restricted methods, or dead ends during initial testing.

At this stage, the attack surface on port 80 appeared exhausted for the moment.

To continue the assessment, attention shifted toward another exposed service running on port `8080`, which became the next target for enumeration and analysis.

## Port 8080 Enumeration

After exhausting the initial attack surface on port 80, attention shifted toward another exposed service running on port `8080`.

Browsing to the application revealed a cryptocurrency-themed website called **Bandit-Coin**.

The application appeared to simulate a Web3 cryptocurrency platform and exposed a dashboard-style interface.

### Bandit-Coin Dashboard

![Bandit-Coin Dashboard](../assets/images/El-Bandito-8080-dashboard.png)

Initial walkthrough and manual testing of the application did not immediately reveal any sensitive information or obvious vulnerabilities.

Since the web application appeared larger and more feature-rich than the previous service, directory enumeration was performed to identify hidden endpoints and administrative functionality.

### Gobuster Enumeration

```bash
gobuster dir -u http://<MACHINE_IP>:8080 \
-w <WORDLIST_PATH> \
-b 404
```

### Interesting Endpoints

```text
/admin               (Status: 403)
/assets              (Status: 200)
/health              (Status: 200)
/traceroute          (Status: 403)
/trace               (Status: 403)
/environment         (Status: 403)
/administration      (Status: 403)
/error               (Status: 500)
/administrator       (Status: 403)
/metrics             (Status: 403)
/env                 (Status: 403)
/dump                (Status: 403)
```

### Analysis

The enumeration process exposed multiple potentially sensitive endpoints commonly associated with:

* administration panels
* debugging functionality
* environment configurations
* application health monitoring
* tracing utilities
* metrics collection

Several endpoints returning `403 Forbidden` were especially interesting because they confirmed the existence of restricted resources rather than nonexistent paths.

Notable findings included:

| Endpoint                 | Observation                                       |
| ------------------------ | ------------------------------------------------- |
| `/admin`                 | Restricted administrative functionality           |
| `/environment`           | Potential environment configuration exposure      |
| `/metrics`               | Possible monitoring or observability endpoint     |
| `/dump`                  | Potential debug or memory dump functionality      |
| `/trace` & `/traceroute` | Possible tracing or internal diagnostic utilities |
| `/error`                 | Returned HTTP 500 Internal Server Error           |

The `/error` endpoint returning a `500 Internal Server Error` strongly suggested backend processing issues and indicated that the application might expose additional information during malformed requests or forced error conditions.

At this stage, the assessment shifted toward probing these endpoints further for misconfigurations, information disclosure, or access control weaknesses.

## JavaScript Analysis

While analyzing the exposed `static/messages.js` file discovered earlier during source code inspection, additional application functionality was identified within the client-side JavaScript logic.

Reviewing the script revealed two interesting endpoints used by the application:

```text id="7r9x1m"
/getMessages
/send_message
```

### Discovered Endpoints

![Discovered Endpoints](../assets/images/messages-js-endpoints.png)

The `fetchMessages()` function performed a request to:

```javascript id="8vt6mz"
fetch("/getMessages")
```

Attempting to access this endpoint directly through the browser resulted in a redirect back to the login page, strongly suggesting that the endpoint required authentication or session validation.

### `/getMessages` Redirect

![getMessages Redirect](../assets/images/El-Bandito-getmessages-function.png)

Further analysis of the JavaScript source revealed another function named `sendMessage()` which issued a POST request to:

```javascript id="jlwm5k"
fetch("/send_message", {
	method: "POST"
})
```

Direct browser access to `/send_message` returned:

```text id="jlwm3r"
Method Not Allowed
```

### `/send_message` Response

![send\_message Response](../assets/images/El-Bandito-sendmessage-function.png)

This behavior indicated that the endpoint likely expected specifically crafted POST requests rather than standard browser GET requests.

The exposed messaging functionality appeared highly interesting because:

* authenticated message retrieval was implemented
* client-side message handling logic was exposed
* custom POST requests were used for sending data
* authentication checks appeared inconsistently enforced

At this stage, the focus shifted toward testing the exposed endpoints directly and analyzing how the backend processed user-controlled input.
